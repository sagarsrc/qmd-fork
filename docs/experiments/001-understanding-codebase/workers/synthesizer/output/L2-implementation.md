# QMD L2 Implementation

## Search Pipeline

The primary search path is `hybridQuery(store, query, options)` in `src/store.ts`, invoked by CLI `querySearch()` in `src/cli/qmd.ts`, SDK `QMDStore.search({ query })` in `src/index.ts`, and MCP through SDK search calls in `src/mcp/server.ts` (see cli-explorer: `query`; sdk-mcp-explorer: SDK Search API; store-explorer: Hybrid Search).

Step by step:

1. Validate and probe the raw query with BM25 using `searchFTS(db, query, limit, collectionName?)` in `src/store.ts` (see store-explorer: BM25 Search).
2. If there is no intent and the BM25 top score is strong, expansion is skipped: `topScore >= 0.85` and `topScore - secondScore >= 0.15` (see store-explorer: Hybrid Search).
3. Otherwise `expandQuery(query, model, db, intent, llmOverride?)` in `src/store.ts` calls the LLM layer to produce typed `lex`, `vec`, and `hyde` variants, cached in `llm_cache` (see store-explorer: Hybrid Search; llm-ast-explorer: Query Expansion).
4. Lexical lists are routed to FTS: original query plus `lex` expansions call `searchFTS()` (see store-explorer: Hybrid Search).
5. Semantic lists are routed to vector search: original query plus `vec` and `hyde` expansions are embedded in batch and passed to `searchVec()` with precomputed embeddings (see store-explorer: Vector Search and Hybrid Search).
6. Ranked result lists are fused with `reciprocalRankFusion(resultLists, weights, k = 60)` in `src/store.ts`; original-query lists receive 2.0 weight via `getHybridRrfWeights()` and expansion lists receive 1.0 (see store-explorer: Hybrid Search; functions).
7. The top `candidateLimit` documents, default 40, are chunked and each candidate gets a best chunk selected by keyword overlap. Query terms have weight 1.0 and intent terms have weight 0.5 (see store-explorer: Hybrid Search).
8. Unless reranking is disabled, `rerank(query, documents, model, db, intent, llmOverride?)` in `src/store.ts` calls `LlamaCpp.rerank()` in `src/llm.ts` over the selected chunks (see store-explorer: Hybrid Search; llm-ast-explorer: Reranking).
9. Final score blends RRF position with rerank score: `rrfScore = 1 / rrfRank`; RRF weight is 0.75 for ranks 1-3, 0.60 for ranks 4-10, and 0.40 afterward; `blendedScore = rrfWeight * rrfScore + (1 - rrfWeight) * rerankScore` (see store-explorer: Hybrid Search).
10. Results are deduped by file, filtered by min score, sliced to limit, decorated with snippets/context, and rendered by CLI `outputResults()` or returned through SDK/MCP (see cli-explorer: Output Format Pipeline; sdk-mcp-explorer: MCP Tool Call Flow).

`structuredSearch(store, searches, options)` in `src/store.ts` skips internal expansion and uses caller-provided typed searches; the first ranked list gets 2x weight. CLI structured query documents are parsed by `parseStructuredQuery()` in `src/cli/qmd.ts`; MCP `query` requires typed searches directly (see cli-explorer: `query`; sdk-mcp-explorer: MCP Tools; store-explorer: Hybrid Search).

## Chunking

Chunking is centered in `src/store.ts`: `scanBreakPoints`, `findCodeFences`, `findBestCutoff`, `mergeBreakPoints`, `chunkDocumentWithBreakPoints`, `chunkDocument`, `chunkDocumentAsync`, and `chunkDocumentByTokens` (see store-explorer: Exported Function Inventory).

Regex breakpoints come from `BREAK_PATTERNS` in `src/store.ts`: h1=100, h2=90, h3=80, h4=70, h5=60, h6=50, code fence=80, horizontal rule=60, blank paragraph=20, list item=5, ordered list item=5, newline=1. `scanBreakPoints(text)` runs every regex, keeps the highest score per character position, and returns sorted breakpoints (see store-explorer: Chunking).

Code fences are detected by `findCodeFences(text)` in `src/store.ts`, which toggles on newline-prefixed triple backticks and treats an unclosed fence as running to document end. `isInsideCodeFence(pos, fences)` blocks cuts where `pos > start && pos < end`, allowing cuts at fence boundaries but not inside fenced bodies (see store-explorer: Chunking).

The actual cutoff score in `findBestCutoff()` is:

```text
distance = targetCharPos - bp.pos
normalizedDist = distance / windowChars
multiplier = 1.0 - (normalizedDist * normalizedDist) * decayFactor
finalScore = bp.score * multiplier
```

With default `decayFactor = 0.7`, a breakpoint exactly at target keeps full score, one halfway back keeps 82.5%, and one at the search-window edge keeps 30%. Defaults approximate 900 tokens as 3600 chars, 135 overlap tokens as 540 chars, and 200 window tokens as 800 chars (see store-explorer: Chunking).

`chunkDocumentWithBreakPoints()` slices from `charPos` to `targetEndPos`, replaces `endPos` with the best cutoff when available, and advances by `endPos - overlapChars` with a guard against repeated starts. `chunkDocumentAsync()` adds optional AST breakpoints from `src/ast.ts` when `chunkStrategy === "auto"` and a filepath is provided. `chunkDocumentByTokens()` uses the LLM tokenizer, starts with a conservative 3 chars/token estimate, recursively splits over-limit chunks, and detokenizes truncation only as a fallback (see store-explorer: Chunking; llm-ast-explorer: Graceful Degradation).

AST breakpoints are implemented by `getASTBreakPoints(content, filepath)` in `src/ast.ts`. It supports TypeScript, TSX/JSX, JavaScript, Python, Go, and Rust; unsupported files return `[]`. Captures score class/interface/struct/trait/impl/module-level structures at 100, exports/functions/methods/decorated definitions at 90, types/enums at 80, imports at 60, and unknown captures at 20 (see llm-ast-explorer: AST Chunking Overview and Query-Based Breakpoints).

## Indexing & Storage

The durable model in `src/store.ts` is content-addressable. `documents` rows store collection/path/title/hash lifecycle metadata, while `content` rows store immutable bodies by SHA-256 hash. Multiple document paths can share one content row and embedding set (see store-explorer: Overview and Database Schema).

`reindexCollection(store, collectionPath, globPattern, collectionName, options)` in `src/store.ts` scans a collection using fast-glob. Default ignores are `node_modules`, `.git`, `.cache`, `vendor`, `dist`, and `build`; user ignore patterns are appended, and hidden path parts are filtered after globbing (see store-explorer: Indexing).

For each readable non-empty UTF-8 file, the indexer resolves the real path, stores the normalized relative path literally, computes `hashContent(content)`, extracts a title with `extractTitle(content, filename)`, and calls `findOrMigrateLegacyDocument(db, collectionName, path)` (see store-explorer: Indexing; functions).

Lifecycle behavior:

- New file: `insertContent()` then `insertDocument()` insert content/document rows and rebuild FTS (see store-explorer: Indexing).
- Same hash and title: counted unchanged (see store-explorer: Indexing).
- Same hash but changed title: `updateDocumentTitle()` updates title/mtime and FTS (see store-explorer: functions).
- Changed content: `insertContent()` then `updateDocument()` update hash/title/mtime and FTS (see store-explorer: functions).
- Missing active path after scan: `deactivateDocument()` soft-deactivates the document (see store-explorer: Indexing).
- Orphan cleanup: `cleanupOrphanedContent()` removes content hashes no longer referenced by documents (see store-explorer: Indexing and Maintenance functions).

FTS storage uses `documents_fts` with columns `filepath`, `title`, and `body`, tokenized as `porter unicode61`. BM25 weights are `bm25(documents_fts, 1.5, 4.0, 1.0)`, so title matches weigh highest, filepath next, and body lowest. Raw lower-is-better BM25 values are normalized as `abs(bm25_score) / (1 + abs(bm25_score))` (see store-explorer: BM25 Search).

Vectors are split between `content_vectors` metadata and `vectors_vec` sqlite-vec rows. `ensureVecTableInternal(dimensions)` creates `vectors_vec` as `vec0(hash_seq TEXT PRIMARY KEY, embedding float[DIM] distance_metric=cosine)`. Dimension mismatches throw and instruct re-embedding; compatible legacy schema may be dropped/recreated (see store-explorer: Vector Search).

## Query Expansion

The LLM expansion implementation is `LlamaCpp.expandQuery(query, options)` in `src/llm.ts`, wrapped and cached by `expandQuery(query, model, db, intent, llmOverride?)` in `src/store.ts` (see llm-ast-explorer: Query Expansion; store-explorer: Exported Function Inventory).

Prompt shape:

```text
/no_think Expand this search query: <query>
Query intent: <intent>        # only when intent is supplied
```

Generation is grammar constrained so every line must be one of:

```text
lex: ...
vec: ...
hyde: ...
```

The generation call uses a bounded context, default `expandContextSize = 2048`, `maxTokens: 600`, `temperature: 0.7`, `topK: 20`, `topP: 0.8`, and a presence penalty. Parsed lines become `{ type, text }`, malformed lines are discarded, and generated lines that contain none of the original query terms are filtered. If output is empty, fallback is `hyde: Information about <query>`, `lex: <query>`, and `vec: <query>`; if generation fails, fallback is `vec: <query>` plus `lex: <query>` when lexical expansion is enabled (see llm-ast-explorer: Query Expansion).

`options.context` exists but is not currently included in the prompt. `intent` is used and is passed from CLI `--intent`, SDK `SearchOptions.intent`, and MCP query parameters (see llm-ast-explorer: Gotchas / Design Decisions; cli-explorer: `query`; sdk-mcp-explorer: MCP Tools).

## Hybrid Fusion

Fusion is `reciprocalRankFusion(resultLists, weights = [], k = 60)` in `src/store.ts`. Internal ranks are 0-based, so contribution is:

```text
contribution = weight / (k + rank + 1)
```

A top-rank bonus is added: `+0.05` for rank 1 and `+0.02` for ranks 2-3. `buildRrfTrace(resultLists, weights, listMeta, k)` reports explain traces with 1-indexed ranks for `--explain` output (see store-explorer: Hybrid Search; functions).

`getHybridRrfWeights(rankedListMeta)` in `src/store.ts` assigns weight 2.0 to original-query lists and 1.0 to expansion-derived lists. `structuredSearch()` gives the first caller-supplied ranked list 2x weight because caller ordering is treated as importance (see store-explorer: Hybrid Search).

Candidate and score limits:

- CLI `query` supports `-C, --candidate-limit <n>` and `--min-score`; `--no-rerank` bypasses LLM reranking (see cli-explorer: `query`).
- Hybrid default candidate limit is 40 before chunk selection and reranking (see store-explorer: Hybrid Search).
- `vsearch` defaults min score to 0.3 when not provided; `query` and `search` default to 0 (see cli-explorer: Gotchas / Design Decisions).
- `searchVec()` scores vector hits as cosine similarity: `score = 1 - bestDistance`, after deduping by filepath and keeping the lowest cosine distance (see store-explorer: Vector Search).

## Configuration System

External config lives in `src/collections.ts` as `CollectionConfig`. It can come from default YAML, a custom YAML path, or inline SDK memory via `setConfigSource(source?)` (see db-coll-explorer: Config Resolution; sdk-mcp-explorer: SDK Overview).

Default file resolution:

- `QMD_CONFIG_DIR` wins.
- Otherwise `XDG_CONFIG_HOME/qmd` is used.
- Otherwise fallback is `qmdHomedir()/.config/qmd` (see db-coll-explorer: Config Resolution).

The active index name defaults to `index`; `setConfigIndexName(name)` maps path-like names to safe config filenames. CLI local config discovery calls `findLocalConfigPath(startDir)`, walking upward for `.qmd/index.yaml` then `.qmd/index.yml`. A found local config pairs with `.qmd/index.sqlite` from `getLocalDbPath(configPath)` (see db-coll-explorer: Config Resolution and diagrams).

YAML is canonical for CLI config. SQLite mirrors it through `syncConfigToDb(db, config)` in `src/store.ts`, which hashes `JSON.stringify(config)` and compares it to `store_config.config_hash`. If the hash changed, it upserts configured collections into `store_collections`, deletes DB collections missing from YAML, syncs `global_context` into `store_config`, and writes the new hash (see db-coll-explorer: Config Sync).

Collections contain path, pattern, ignore list, update command, include-by-default flag, and context map. Context lookup normalizes file paths and prefixes to leading-slash paths, returns the longest matching collection prefix, and falls back to global context (see db-coll-explorer: Context System).

SDK mutations are SQLite-first and optionally write through to YAML/inline config when the SDK was opened with a config source. DB-only SDK mode intentionally skips external config writes (see sdk-mcp-explorer: SDK Collection Management, SDK Context Management, Gotchas).

## Cross-Runtime Compatibility

`src/db.ts` provides the runtime boundary. `isBun = "Bun" in globalThis`; Bun dynamically imports `"bun:" + "sqlite"` to avoid static resolution by Node-oriented builds, while Node imports `better-sqlite3` (see db-coll-explorer: Bun vs Node Path).

The common `Database` surface is deliberately narrow: `exec`, `prepare`, `loadExtension`, `transaction`, and `close`. `Statement` exposes `run`, `get`, and `all`. Store code depends on this subset instead of runtime-specific driver APIs (see db-coll-explorer: Database Interface).

sqlite-vec loading differs by runtime. On Bun, `sqlite-vec` provides a native loadable path, tested against an in-memory DB, and real DBs call `db.loadExtension(vecPath)`. On Node, the `sqlite-vec` package loader calls `sqliteVec.load(db)` for better-sqlite3. If loading fails, `store.ts` records sqlite-vec unavailable and FTS/BM25 still works (see db-coll-explorer: sqlite-vec Loading; store-explorer: Vector Search).

macOS Bun has extra handling because Apple's system SQLite may lack extension loading. The Bun path tries Homebrew SQLite at `/opt/homebrew/opt/sqlite/lib/libsqlite3.dylib` and `/usr/local/opt/sqlite/lib/libsqlite3.dylib` via `BunDatabase.setCustomSQLite()`, then still probes sqlite-vec. Failure leads to vector unavailability guidance, not total DB failure (see db-coll-explorer: macOS SQLite Extension Loading).

The LLM layer also handles platform compatibility. `resolveLlamaGpuMode()` honors `QMD_FORCE_CPU` and `QMD_LLAMA_GPU`; GPU initialization can fall back to CPU; failed GPU modes are remembered in-process. On Darwin, `GGML_METAL_NO_RESIDENCY=1` mitigation avoids a libggml-metal process-exit assertion unless `QMD_METAL_KEEP_RESIDENCY=1` overrides it (see llm-ast-explorer: GPU/CPU Management and Gotchas).
