## LLM Layer Overview

`src/llm.ts` is the local model abstraction for QMD. It wraps `node-llama-cpp` behind an `LLM` interface and a concrete `LlamaCpp` class that supports embeddings, generation, query expansion, reranking, model diagnostics, and disposal.

The lifecycle is lazy:

- `getDefaultLlamaCpp()` constructs a `LlamaCpp` singleton only when needed.
- The native `Llama` runtime is loaded by `ensureLlama()`.
- Embedding, generation, and rerank models are independently loaded on first use.
- Embedding and rerank contexts are pooled and cached for parallel work.
- Generation contexts are created per call and disposed immediately after the call.

The class uses load/create promise guards (`embedModelLoadPromise`, `embedContextsCreatePromise`, etc.) so concurrent requests do not allocate duplicate native models or contexts. Activity calls reset an inactivity timer. By default, idle timeout is 5 minutes and unloads contexts only; model disposal on inactivity is opt-in via `disposeModelsOnInactivity`.

Session helpers (`withLLMSession`, `withLLMSessionForLlm`) add reference counting and operation tracking. `canUnloadLLM()` returns false while sessions or operations are active, preventing the idle timer from disposing contexts mid-operation.

## Model Loading

Model URIs default to HuggingFace GGUF references:

- Embedding: `hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf`
- Rerank: `hf:ggml-org/Qwen3-Reranker-0.6B-Q8_0-GGUF/qwen3-reranker-0.6b-q8_0.gguf`
- Generation: `hf:tobil/qmd-query-expansion-1.7B-gguf/qmd-query-expansion-1.7B-q4_k_m.gguf`

They can be overridden with constructor config or env vars:

- `QMD_EMBED_MODEL`
- `QMD_GENERATE_MODEL`
- `QMD_RERANK_MODEL`

The cache directory is:

- `$XDG_CACHE_HOME/qmd/models` when `XDG_CACHE_HOME` is set
- otherwise `~/.cache/qmd/models`

`resolveModel()` ensures the cache directory exists, delegates URI resolution/download to `node-llama-cpp`'s `resolveModelFile(modelUri, cacheDir)`, then validates the local file with `validateGgufFile()`.

`pullModels()` performs eager model download/refresh. For `hf:` URIs it parses `hf:<owner>/<repo>/<file>`, checks the remote ETag with a HuggingFace `HEAD` request, compares it with a local `.etag` file, deletes stale cached candidates, calls `resolveModelFile`, validates the GGUF, then stores the latest ETag.

GGUF inspection reads the first 512 bytes and checks the `GGUF` magic. It distinguishes missing files, valid GGUF files, HTML responses, and other invalid files. HTML detection exists because failed downloads can produce proxy/firewall/captive portal pages. Invalid cached files are removed so the next run can retry cleanly.

## Embedding Formatting

There are two embedding prompt formats.

Default/embeddinggemma/nomic-style:

- Query: `task: search result | query: <query>`
- Document: `title: <title-or-none> | text: <text>`

Qwen3-Embedding format is selected by `isQwen3EmbeddingModel()`, which matches model URIs containing `qwen` and `embed` in either order.

Qwen3 query format:

- `Instruct: Retrieve relevant documents for the given query`
- `Query: <query>`

Qwen3 document format:

- raw text
- or `<title>\n<text>` when a title exists

`formatQueryForEmbedding()` and `formatDocForEmbedding()` default to the resolved embedding model URI when no explicit model URI is supplied. This matters because indexed documents and search queries must use the same formatting convention for comparable vectors.

## Embedding Generation

Single embedding flow:

1. `embed()` touches activity.
2. `ensureEmbedContext()` loads the runtime, embedding model, and first embedding context if needed.
3. `truncateToContextSize()` tokenizes with the embedding model tokenizer and truncates to the smaller of `EMBED_CONTEXT_SIZE` and the model's trained context size.
4. `context.getEmbeddingFor(safeText)` returns a native vector.
5. The vector is converted to a JS `number[]` with `Array.from(embedding.vector)`.

The exported `EmbeddingResult` type stores `embedding: number[]`, not `Float32Array`. The underlying native vector is array-like and is normally backed by floating point data, but the public API serializes it as a plain numeric array.

Batch embedding uses `embedBatch()`. It creates a pool of embedding contexts and splits texts across them. With one context it embeds sequentially; with multiple contexts it chunks the input and uses `Promise.all`. Each text is truncated independently. Failed individual embeddings return `null`; a broad batch failure returns an all-null array.

Parallelism is computed from hardware:

- GPU: use 25% of free VRAM, estimate 150 MB per embedding context, cap at 8.
- CPU: split math cores, target at least 4 threads per context, cap at 4.
- Env override: `QMD_EMBED_PARALLELISM`, integer 1-8.
- Windows CUDA is serialized by default because concurrent contexts are known unstable there.

Embedding dimensions are not hard-coded in `llm.ts`. They are inferred by consumers from returned vector length. The code also uses model tokenization for context-window safety rather than estimating length by characters.

## Query Expansion

`expandQuery()` turns a user query into typed sub-queries for the hybrid search stack:

- `lex`: lexical/BM25-style search terms
- `vec`: semantic vector search query
- `hyde`: hypothetical-answer style query/document text

The method:

1. Ensures the native runtime and generation model are loaded.
2. Builds a constrained grammar:
   - each line must be `lex: ...`, `vec: ...`, or `hyde: ...`
3. Builds the prompt:
   - `/no_think Expand this search query: <query>`
   - plus `Query intent: <intent>` when an intent option exists
4. Creates a bounded generation context using `expandContextSize` default 2048.
5. Runs `LlamaChatSession.prompt()` with grammar, `maxTokens: 600`, `temperature: 0.7`, `topK: 20`, `topP: 0.8`, and presence penalty.
6. Parses lines into `{ type, text }`.
7. Filters out malformed lines and generated lines that contain none of the original query terms.
8. Optionally removes `lex` entries when `includeLexical` is false.

If the structured output is empty, the fallback is:

- `hyde: Information about <query>`
- `lex: <query>`
- `vec: <query>`

If generation fails, the fallback is just `vec: <query>` plus `lex: <query>` when lexical queries are enabled.

`options.context` exists in the type but is not used in the current prompt. `options.intent` is accepted by the concrete method and is passed from `store.ts`.

## Reranking

`RerankDocument` is:

- `file: string`
- `text: string`
- optional `title?: string`

The rerank path uses the Qwen3 reranker through `node-llama-cpp` ranking contexts:

1. `ensureRerankContexts()` lazily creates ranking contexts.
2. `rerank()` tokenizes the query and computes document token budget:
   - `RERANK_CONTEXT_SIZE` default 4096
   - minus `RERANK_TEMPLATE_OVERHEAD` 512
   - minus query tokens
3. Documents are token-truncated with the rerank model tokenizer.
4. Effective duplicate document texts are deduplicated before scoring.
5. Unique texts are split across active ranking contexts.
6. Each context runs `rankAll(query, chunk)`.
7. Scores are flattened, sorted descending, and mapped back to original documents.

The result preserves all original duplicate documents by expanding deduplicated scores back to their original `{ file, index }` entries. The returned score is whatever `node-llama-cpp`'s ranking context returns; higher means more relevant.

## GPU/CPU Management

GPU mode is controlled by `resolveLlamaGpuMode()`:

- `QMD_FORCE_CPU` with any truthy non-disable value forces CPU mode.
- `QMD_LLAMA_GPU=false/off/none/disable/disabled/0` disables GPU.
- `QMD_LLAMA_GPU=metal|vulkan|cuda` selects a backend.
- unset or invalid values fall back to `"auto"`.

`ensureLlama()` loads `node-llama-cpp` with `gpu` mode and `build: "auto"` by default. If auto selects CPU and `getLlamaGpuTypes("allValid")` is available, it probes OS-valid GPU backends without building from source (`build: "never"`). If a candidate successfully reports GPU support, it replaces the CPU runtime.

Failed GPU modes are remembered in-process in `failedGpuInitModes`, preventing repeated expensive probes. GPU init failures fall back to CPU. If CPU-only prebuilt bindings are unavailable, the code loads the packaged auto backend and disables model GPU offloading with `gpuLayers: 0` for model loads.

Mac Metal mitigation is handled by environment, not by JS disposal. `isDarwinMetalMitigationActive()` reports whether `GGML_METAL_NO_RESIDENCY=1` is active on Darwin and not overridden by `QMD_METAL_KEEP_RESIDENCY=1`. Comments explain this avoids a libggml-metal static-destructor assertion at process exit. `installDarwinExitGuard()` is now a compatibility no-op.

## stdout Redirect

`withNativeStdoutRedirectedToStderr()` temporarily replaces `process.stdout.write` so native `node-llama-cpp` initialization/build/probe noise goes to stderr. It uses a depth counter so nested calls restore stdout only after the outermost operation finishes.

This exists because QMD has JSON-producing API/CLI paths where stdout must contain only machine-readable data. Native library logs on stdout would corrupt JSON payloads; stderr is safe for diagnostics.

The dynamic import of `node-llama-cpp` is also wrapped in this redirect, because import-time native probing can write output.

## AST Chunking Overview

`src/ast.ts` implements best-effort AST breakpoints for code files using `web-tree-sitter`.

Supported languages:

- TypeScript: `.ts`, `.mts`, `.cts`
- TSX: `.tsx`, `.jsx`
- JavaScript: `.js`, `.mjs`, `.cjs`
- Python: `.py`
- Go: `.go`
- Rust: `.rs`

`detectLanguage(filepath)` maps file extensions to a `SupportedLanguage` or returns `null`. Markdown is intentionally unsupported by AST and continues to use regex breakpoints.

AST breakpoints are returned as `BreakPoint` objects with byte positions, scores, and `ast:<capture>` types. The scores align with markdown breakpoint scores, so the existing `findBestCutoff()` distance-decay algorithm can choose among regex and AST boundaries without special cases.

## Parser Initialization

`web-tree-sitter` is dynamically imported by `ensureInit()` to avoid top-level WASM setup. Initialization is cached in `initPromise`, and the module classes are stored in `ParserClass`, `LanguageClass`, and `QueryClass`.

Grammar resolution uses `createRequire(import.meta.url)` and `require.resolve("<pkg>/<wasm>")`. Grammars are cached by WASM filename in `grammarCache`; this lets TypeScript and JavaScript share the same TypeScript WASM file in the current mapping. Compiled queries are cached per language in `queryCache`.

Grammar load failures are graceful:

- failed languages are recorded in `failedLanguages`
- detailed messages are stored in `grammarLoadErrors`
- a warning is printed once per language
- future loads for that language immediately return `null`

`getASTStatus()` initializes tree-sitter, attempts to load each grammar, compiles each query, and reports availability per language.

## Query-Based Breakpoints

Each supported language has a tree-sitter S-expression query. Captures include declarations and import/use statements:

- TypeScript/TSX: exports, classes, functions, methods, interfaces, type aliases, enums, imports, arrow/function-expression declarations
- JavaScript: exports, classes, functions, methods, imports, arrow/function-expression declarations
- Python: classes, functions, decorated definitions, imports
- Go: type declarations, functions, methods, imports
- Rust: structs, impls, functions, traits, enums, use declarations, type items, modules

`SCORE_MAP` aligns AST captures with markdown structure scores:

- 100: class/interface/struct/trait/impl/module-level structures
- 90: exports/functions/methods/decorated definitions
- 80: types/enums
- 60: imports
- default 20 for unknown captures

At each byte position, `getASTBreakPoints()` keeps only the highest-scoring capture. This handles overlapping captures such as an `export_statement` wrapping a class declaration.

## Graceful Degradation

`getASTBreakPoints()` never throws to callers. Unsupported extension, tree-sitter init failure, grammar load failure, parse failure, query failure, or any other exception returns `[]`.

The regex fallback is implemented in `store.ts`:

- `chunkDocumentAsync()` always scans regex breakpoints first.
- When `chunkStrategy === "auto"` and a filepath exists, it imports `getASTBreakPoints()`.
- If AST breakpoints are non-empty, it merges them with regex breakpoints using `mergeBreakPoints()`.
- If AST returns empty, regex breakpoints are used unchanged.

This design keeps AST chunking an enhancement rather than a dependency for indexing.

## Gotchas / Design Decisions

- `node-llama-cpp` is used instead of raw llama.cpp so QMD can work from TypeScript with managed model loading, tokenization, embeddings, chat sessions, ranking contexts, GPU backend selection, and packaged native bindings.
- stdout redirection is required because native llama initialization can emit stdout noise, and QMD uses stdout for JSON payloads.
- Model/context lifecycle follows node-llama-cpp ownership rules: contexts are disposed before models, models before the runtime. Generation contexts are always short-lived.
- Idle cleanup defaults to disposing contexts but keeping models loaded, avoiding repeated model reload and VRAM churn during typical search sessions.
- CI mode blocks expensive LLM operations in most paths by throwing when `CI=true`.
- Embedding API results are plain `number[]`; downstream code should infer vector dimension from array length.
- `formatQueryForEmbedding()` and `formatDocForEmbedding()` must stay consistent with the active embedding model, or stored vectors and query vectors become semantically mismatched.
- `options.context` in query expansion is currently unused.
- AST grammar versions in `GRAMMAR_MAP` are documentation/error-message versions, but `package.json` currently lists `tree-sitter-go` and `tree-sitter-python` at `0.25.0` while `ast.ts` error messages mention `0.23.4` for those two packages. That version drift is worth checking before calling the grammar versions truly pinned.
- The JavaScript grammar mapping uses `tree-sitter-typescript.wasm`, not a separate `tree-sitter-javascript` WASM. This is an implementation choice to watch if JavaScript query compatibility changes.
- `ast.ts` comments mention optional dependencies, but the current `package.json` lists tree-sitter grammar packages under `dependencies`.
