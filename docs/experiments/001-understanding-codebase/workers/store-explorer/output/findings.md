# Store Layer Findings

## Overview

`src/store.ts` is the QMD core engine. It owns the durable SQLite index, collection metadata, content-addressed document storage, FTS search, vector search, hybrid retrieval orchestration, query expansion/reranking cache, chunking, embeddings, virtual path resolution, docid lookup, context inheritance, cleanup, and index health reporting.

The layer exposes both low-level database primitives and high-level search functions. `createStore()` opens the database, initializes schema and triggers, loads sqlite-vec when available, and returns a bound `Store` facade. Higher-level functions such as `hybridQuery()`, `vectorSearchQuery()`, and `structuredSearch()` accept a `Store` and orchestrate retrieval without owning database lifecycle.

The central storage model is content-addressable: file records in `documents` point to immutable body rows in `content` by SHA-256 hash. Identical content across paths or collections can share one `content` row and one set of embeddings.

## Chunking

Chunking has three layers:

1. Regex break-point detection with `BREAK_PATTERNS`.
2. Shared character-space chunking with `chunkDocumentWithBreakPoints()`.
3. Optional token-aware post-processing in `chunkDocumentByTokens()`.

`BREAK_PATTERNS` assigns wide scores to markdown boundaries:

| Pattern | Score | Type |
|---|---:|---|
| `\n#{1}(?!#)` | 100 | `h1` |
| `\n#{2}(?!#)` | 90 | `h2` |
| `\n#{3}(?!#)` | 80 | `h3` |
| `\n#{4}(?!#)` | 70 | `h4` |
| `\n#{5}(?!#)` | 60 | `h5` |
| `\n#{6}(?!#)` | 50 | `h6` |
| `\n```` | 80 | `codeblock` |
| horizontal rule | 60 | `hr` |
| blank paragraph boundary | 20 | `blank` |
| unordered list item | 5 | `list` |
| ordered list item | 5 | `numlist` |
| newline | 1 | `newline` |

`scanBreakPoints(text)` runs every regex with `matchAll()`, stores the best score per character position in a `Map`, then returns sorted `BreakPoint[]`. This means pattern order is less important than score when two patterns hit the same index.

`findCodeFences(text)` scans only newline-prefixed triple backticks (`\n````). It toggles `inFence`; pairs become `{ start, end }`, and an unclosed fence runs to document end. `isInsideCodeFence()` returns true for `pos > start && pos < end`, so chunk cuts are allowed at the fence boundary but not inside the fenced body.

`findBestCutoff()` searches backward from the target character position within a window. Defaults:

| Token config | Char approximation |
|---|---:|
| `CHUNK_SIZE_TOKENS = 900` | `CHUNK_SIZE_CHARS = 3600` at 4 chars/token |
| `CHUNK_OVERLAP_TOKENS = 135` | `CHUNK_OVERLAP_CHARS = 540` |
| `CHUNK_WINDOW_TOKENS = 200` | `CHUNK_WINDOW_CHARS = 800` |

Each candidate gets squared distance decay:

```text
distance = targetCharPos - bp.pos
normalizedDist = distance / windowChars
multiplier = 1.0 - (normalizedDist * normalizedDist) * decayFactor
finalScore = bp.score * multiplier
```

With the default `decayFactor = 0.7`, a break at the target keeps 100% score, a break halfway back keeps 82.5%, and a break at the window edge keeps 30%. The squared curve lets a strong heading farther back beat a weak newline near the target.

`chunkDocumentWithBreakPoints()` slices from `charPos` to `targetEndPos = charPos + maxChars`, replaces `endPos` with the best cutoff if it is within the current chunk, then advances to `endPos - overlapChars`. A guard prevents overlap from moving backward or repeating the same chunk start.

`chunkDocument()` is synchronous regex-only. `chunkDocumentAsync()` can merge AST breakpoints from `./ast.js` when `chunkStrategy === "auto"` and `filepath` is provided, otherwise it is regex-only. `chunkDocumentByTokens()` starts with a conservative 3 chars/token estimate, tokenizes each candidate chunk with the LLM tokenizer, recursively splits over-limit chunks using measured chars/token, and finally falls back to detokenized truncation if pathological text cannot be split safely.

## BM25 Search

FTS5 is initialized as:

```sql
CREATE VIRTUAL TABLE documents_fts USING fts5(
  filepath, title, body,
  tokenize='porter unicode61'
)
```

The BM25 call uses weighted columns:

```sql
bm25(documents_fts, 1.5, 4.0, 1.0)
```

So title matches are weighted highest, filepath next, body lowest. FTS scores are lower-is-better and often negative; `searchFTS()` maps them to a stable positive score:

```text
score = abs(bm25_score) / (1 + abs(bm25_score))
```

Query construction is handled by private `buildFTS5Query()`:

| Input syntax | FTS behavior |
|---|---|
| `term` | `"term"*` prefix search |
| `"exact phrase"` | exact phrase, no prefix |
| `-term` | binary `NOT`, only valid with positive terms |
| `multi-agent` | phrase `"multi agent"` to match tokenizer split |
| `2026.4.10` | `"2026"* AND "4"* AND "10"*` |
| CJK run | spaced characters inside a phrase |

`validateLexQuery()` rejects newlines and unmatched double quotes. `sanitizeFTS5Term()` keeps Unicode letters, numbers, apostrophes, and underscore, then lowercases.

CJK normalization exists because `unicode61` does not segment CJK words. `normalizeCjkForFTS()` spaces each Han/Hiragana/Katakana/Hangul character. Startup calls `rebuildFTSForCjkNormalization()` if `store_config.fts_cjk_normalized_version` is not current. Insert/update paths also call `rebuildDocumentFTS()` to normalize `filepath`, `title`, and `body` explicitly. Triggers still exist for direct writes, but the production TypeScript paths rebuild FTS entries so CJK text is normalized before indexing.

`searchFTS()` uses a CTE to force FTS first, then joins to `documents` and `content`. When filtering by collection, it requests `limit * 10` FTS candidates before applying the collection filter.

Snippet extraction is separate from FTS. `extractSnippet()` scores lines by literal query terms plus intent terms, optionally searches around a selected chunk with +/-100 chars, returns a diff-style header, and falls back to full-body snippets when chunk-local matching is unhelpful.

## Vector Search

Vector support is optional. `initializeDatabase()` attempts `loadSqliteVec(db)` and probes `vec_version()`. If unavailable, FTS continues to work and vector cleanup/search degrade gracefully.

Embeddings are represented in two tables:

| Table | Role |
|---|---|
| `content_vectors` | metadata for each embedded content chunk |
| `vectors_vec` | sqlite-vec virtual table storing the actual float vector |

`vectors_vec` is created lazily by `ensureVecTableInternal(dimensions)`:

```sql
CREATE VIRTUAL TABLE vectors_vec USING vec0(
  hash_seq TEXT PRIMARY KEY,
  embedding float[DIM] distance_metric=cosine
)
```

Dimension handling is strict. If `vectors_vec` already exists with the same dimension, `hash_seq`, and cosine metric, it is reused. If it exists with a different dimension, the store throws and instructs the user to force re-embed. If the schema is legacy but dimension-compatible, it drops/recreates the virtual table.

`insertEmbedding()` writes `content_vectors` first, then deletes and inserts into `vectors_vec`. This ordering is intentionally crash-aware: the metadata table is what pending-embedding queries inspect. sqlite-vec does not honor `INSERT OR REPLACE`, so the virtual table uses explicit `DELETE` then `INSERT`.

`searchVec()` embeds the query unless a precomputed embedding is supplied. It queries sqlite-vec without joins:

```sql
SELECT hash_seq, distance
FROM vectors_vec
WHERE embedding MATCH ? AND k = ?
```

Then it does a second SQL query against `content_vectors`, `documents`, and `content`. The two-step design avoids sqlite-vec hangs observed when combining virtual vector search with joins. Results are deduped by filepath, keeping the lowest cosine distance. Score is cosine similarity:

```text
score = 1 - bestDistance
```

## Hybrid Search

`hybridQuery()` is the main retrieval pipeline:

1. Run original BM25 probe.
2. Skip LLM query expansion if BM25 has a strong signal.
3. Expand query to typed `lex`, `vec`, and `hyde` variants otherwise.
4. Run lex queries through FTS.
5. Batch embed original plus vec/hyde queries.
6. Run vector lookups using precomputed embeddings.
7. Fuse ranked lists with weighted RRF.
8. Chunk candidate documents and select the best lexical chunk.
9. Rerank selected chunks.
10. Blend RRF position score with reranker score.
11. Deduplicate, filter, and limit.

Strong BM25 bypass requires no intent and:

```text
topScore >= 0.85
topScore - secondScore >= 0.15
```

Expansion uses `expandQuery()`, cached in `llm_cache`. `lex` routes to FTS only. `vec` and `hyde` route to vector only. `hyde` is treated as a hypothetical document text for embedding.

RRF formula:

```text
contribution = weight / (k + rank)
```

In `reciprocalRankFusion()`, internal rank is 0-based and contribution is `weight / (k + rank + 1)` with default `k = 60`. A top-rank bonus is added: +0.05 for rank 1, +0.02 for ranks 2-3. `buildRrfTrace()` reports the same calculation with 1-indexed ranks for explain output.

`getHybridRrfWeights()` gives original-query retrieval lists weight 2.0 and expansion-derived lists weight 1.0. This protects original FTS and original vector evidence regardless of list insertion order.

Reranking is deliberately chunk-based. Candidate documents are chunked, a best chunk is selected by keyword overlap (`queryTerms` weight 1.0, `intentTerms` weight 0.5), and only that chunk is sent to `rerank()`. Rerank results are blended with RRF position:

```text
rrfScore = 1 / rrfRank
rrfWeight = 0.75 if rank <= 3
          = 0.60 if rank <= 10
          = 0.40 otherwise
blendedScore = rrfWeight * rrfScore + (1 - rrfWeight) * rerankScore
```

`structuredSearch()` skips internal expansion and accepts caller-supplied typed searches. Its first ranked list gets 2x weight, assuming caller ordering reflects importance. `vectorSearchQuery()` is vector-only, expands the query, ignores lex variants, searches original plus vec/hyde variants sequentially, and keeps the max score per file.

## Indexing

`reindexCollection(store, collectionPath, globPattern, collectionName, options)` scans a single collection with `fast-glob`.

Default ignored directories:

```text
node_modules, .git, .cache, vendor, dist, build
```

User ignore patterns are appended. Hidden files/folders are filtered after globbing. Each file path is stored as the literal normalized relative path; `handelize()` is explicitly not applied at index time.

For each non-empty readable file:

1. Resolve real filesystem path.
2. Read UTF-8 content.
3. Compute SHA-256 content hash with `hashContent()`.
4. Extract title with `extractTitle()`.
5. Find existing active document via `findOrMigrateLegacyDocument()`.
6. If hash and title unchanged: count unchanged.
7. If title changed only: update title and FTS.
8. If content changed: insert content row and update document hash/title/mtime.
9. If new: insert content row and document row.

After scanning, `getActiveDocumentPaths()` is compared to seen paths. Missing files are soft-deactivated with `active = 0`. Finally `cleanupOrphanedContent()` deletes content hashes no longer referenced by any document row.

Legacy document migration handles case-insensitive path matches and old handalized paths. On a successful rename it rebuilds the FTS row; embeddings remain valid because they are keyed by content hash, not path.

## Embeddings Pipeline

`generateEmbeddings(store, options)` embeds pending active content hashes. Pending means no vector metadata for the current embedding fingerprint, or fewer stored chunks than expected.

The embedding fingerprint is a 6-char SHA-256 prefix over:

```text
model name
query embedding formatter output
document embedding formatter output
chunk size tokens
chunk overlap tokens
```

This invalidates embeddings when model, formatting, chunk size, or overlap changes.

Batch controls default to:

```text
maxDocsPerBatch = 64
maxBatchBytes = 64MB
chunk batch size for embedBatch = 32
max session duration = 30 minutes
```

Pipeline:

1. Optionally clear embeddings when `force` is true.
2. Load pending docs grouped by hash.
3. Build doc batches by count and byte limit.
4. Fetch bodies for each batch.
5. Chunk each body with `chunkDocumentByTokens()`.
6. Probe first chunk to determine vector dimensions and initialize `vectors_vec`.
7. Embed chunks in batches of 32.
8. Store each embedding with `(hash, seq, pos, model, fingerprint, total_chunks, embedded_at)`.
9. Retry failed chunks after 64 successful chunks, and force-retry at batch end.
10. Remove incomplete embeddings for any hash whose chunk set is partial.
11. Report progress and final failures.

Failure handling is per chunk. Failed chunks are tracked by `hash:seq`, attempts are capped at 3, reasons are truncated to 180 chars, and visible error count reflects active unrecovered failures. If the LLM session expires, remaining chunks are marked failed. If active error rate exceeds 80% after at least one batch, the batch is aborted.

`maybeAdoptLegacyEmbeddingFingerprint()` can adopt old rows with empty fingerprints. It embeds one legacy sample under the current fingerprint, searches nearest vector, and if the expected `hash_seq` matches within distance `0.0001`, updates all empty-fingerprint rows for that model.

## Virtual Paths

Virtual paths use the `qmd://` protocol:

```text
qmd://collection-name/path/to/file.md
qmd://collection-name/path.md?index=name
```

`normalizeVirtualPath()` normalizes explicit virtual path forms only:

| Input | Output |
|---|---|
| `qmd://collection/path` | unchanged |
| `qmd:////collection/path` | `qmd://collection/path` |
| `//collection/path` | `qmd://collection/path` |

Bare `collection/path.md` is not considered explicitly virtual.

`parseVirtualPath()` extracts `{ collectionName, path, indexName? }`. Collection root is supported with empty `path`. `buildVirtualPath()` constructs the protocol string and encodes optional `index`.

`resolveVirtualPath()` looks up the collection path from `store_collections` and joins it with the relative path. `toVirtualPath()` maps an absolute path back to a virtual path by finding a collection whose root prefixes the absolute path and verifying an active document exists for the relative path.

## Docids

Docids are short content hash prefixes. `getDocid(hash)` returns the first 6 hex chars. `normalizeDocid()` strips surrounding single/double quotes and a leading `#`. `isDocid()` accepts any normalized hex string of length 6 or more.

`findDocumentByDocid()` performs a prefix lookup:

```sql
WHERE d.hash LIKE ? AND d.active = 1
LIMIT 1
```

If multiple documents share a hash prefix, the first active match is returned. Because docids are content-derived, identical bodies across collections share the same docid. Search and document results include `docid` for citation and quick lookup.

## Database Schema

```text
+-------------------+          +-------------------+
| content           |          | documents         |
|-------------------|          |-------------------|
| hash PK           |<---------| hash FK           |
| doc               |          | id PK             |
| created_at        |          | collection        |
+-------------------+          | path              |
        ^                      | title             |
        |                      | created_at        |
        |                      | modified_at       |
        |                      | active            |
        |                      | UNIQUE(collection,path)
        |                      +-------------------+
        |                               |
        |                               v triggers/rebuild
        |                      +-------------------+
        |                      | documents_fts     |
        |                      |-------------------|
        |                      | rowid = documents.id
        |                      | filepath          |
        |                      | title             |
        |                      | body              |
        |                      +-------------------+
        |
        v
+-------------------+          +-------------------+
| content_vectors   |          | vectors_vec       |
|-------------------|          |-------------------|
| hash PK part      |--------->| hash_seq PK       |
| seq PK part       |          | embedding float[N]|
| pos               |          | cosine distance   |
| model             |          +-------------------+
| embed_fingerprint |
| total_chunks      |
| embedded_at       |
+-------------------+

+-------------------+          +-------------------+
| store_collections |          | store_config      |
|-------------------|          |-------------------|
| name PK           |          | key PK            |
| path              |          | value             |
| pattern           |          +-------------------+
| ignore_patterns   |
| include_by_default|
| update_command    |
| context JSON      |
+-------------------+

+-------------------+
| llm_cache         |
|-------------------|
| hash PK           |
| result            |
| created_at        |
+-------------------+
```

Tables:

| Table | Columns / keys |
|---|---|
| `content` | `hash TEXT PRIMARY KEY`, `doc TEXT NOT NULL`, `created_at TEXT NOT NULL` |
| `documents` | `id INTEGER PRIMARY KEY AUTOINCREMENT`, `collection`, `path`, `title`, `hash`, `created_at`, `modified_at`, `active DEFAULT 1`, FK `hash -> content(hash) ON DELETE CASCADE`, `UNIQUE(collection,path)` |
| `documents_fts` | FTS5 virtual table: `filepath`, `title`, `body`, tokenizer `porter unicode61` |
| `content_vectors` | `hash`, `seq DEFAULT 0`, `pos DEFAULT 0`, `model`, `embed_fingerprint DEFAULT ''`, `total_chunks DEFAULT 1`, `embedded_at`, `PRIMARY KEY(hash,seq)` |
| `vectors_vec` | sqlite-vec virtual table: `hash_seq TEXT PRIMARY KEY`, `embedding float[N] distance_metric=cosine` |
| `store_collections` | `name TEXT PRIMARY KEY`, `path`, `pattern DEFAULT '**/*.md'`, `ignore_patterns`, `include_by_default DEFAULT 1`, `update_command`, `context` |
| `llm_cache` | `hash TEXT PRIMARY KEY`, `result`, `created_at` |
| `store_config` | `key TEXT PRIMARY KEY`, `value` |

Indexes:

```text
idx_documents_collection ON documents(collection, active)
idx_documents_hash       ON documents(hash)
idx_documents_path       ON documents(path, active)
```

FTS triggers:

| Trigger | Event | Behavior |
|---|---|---|
| `documents_ai` | after insert | inserts active document into FTS |
| `documents_ad` | after delete | deletes FTS row by old id |
| `documents_au` | after update | deletes inactive row and upserts active row |

## Gotchas / Design Decisions

- Content-addressable storage means content and embeddings are keyed by hash, while paths are a filesystem mapping layer.
- WAL mode is enabled at startup with `PRAGMA journal_mode = WAL`; foreign keys are enabled.
- sqlite-vec is optional. The DB can still open and FTS can still function when vector extension loading fails.
- `vectors_vec` must not be joined directly in search queries; vector lookup is intentionally two-step to avoid hangs.
- FTS sync has triggers, but indexing code explicitly rebuilds FTS rows to apply CJK normalization.
- Embedding invalidation is fingerprint-based, covering model, prompt formatting, chunk size, and overlap.
- `content_vectors` schema repairs are lazy and idempotent; missing legacy columns are added only when vector operations need them.
- Scoped embedding clears preserve hashes shared with other active collections.
- Reranking uses chunks, not full documents, to avoid token-cost blowups.
- Strong BM25 shortcut is disabled when `intent` is present because intent can change the desired interpretation.
- `handelize()` is display/legacy migration logic, not current index path storage.
- `insertContext()` still queries a dropped legacy `collections` table and appears incompatible with the current `store_collections` schema.
