# QMD Gotchas

## Why does `process.exitCode = 0` instead of `process.exit(0)`?
Successful CLI commands set `process.exitCode = 0` and finish cleanly so Node `beforeExit` hooks can run. Direct `process.exit(0)` triggers a macOS libggml-metal static-destructor assertion. Error paths may still call `process.exit()`.

## Why is stdout redirected to stderr during LLM init?
Native `node-llama-cpp` initialization and GPU probing can print diagnostic noise to stdout. QMD temporarily swaps `process.stdout.write` with stderr so JSON-producing commands do not emit corrupted output.

## Why is YAML the source of truth but SQLite mirrors it?
YAML is user-editable and friendly for version control. SQLite mirrors it so the store can query collection metadata quickly without parsing YAML on every search. A config hash in `store_config` avoids redundant sync work when YAML has not changed.

## What happens if sqlite-vec fails to load?
Vector search degrades gracefully. FTS5/BM25 remains fully functional. On macOS Bun, the failure often means Apple's system SQLite lacks extension loading; installing Homebrew SQLite may fix it.

## What happens if my embedding model changes?
Embeddings are invalidated by an embedding fingerprint that covers model name, query/document formatter strings, chunk size, and overlap. Changing any of these marks existing vectors stale, and you must re-run `qmd embed`.

## Why are chunks 900 tokens with 15% overlap?
Defaults approximate 900 tokens as 3600 characters and 15% overlap as 540 characters. This targets a practical context size for local embedding models while preserving continuity across chunk boundaries.

## What is the strong signal threshold and why does it skip expansion?
If the top BM25 score is `>= 0.85` and the gap to the second score is `>= 0.15`, the query is considered already well answered by keyword search. Skipping expansion saves an LLM call and latency. The threshold is disabled when an explicit `--intent` is provided.

## What is a docid and why is it useful?
A docid is the first 6 characters of a document's SHA-256 content hash. It is stable across collections and renames, short enough to cite in chat or notes, and resolvable with `qmd get #abc123`.

## Why does the SDK create its own LlamaCpp per store?
Each `QMDStore` instantiates its own `LlamaCpp` to avoid global singleton state, allow per-store model overrides, and make lifecycle ownership explicit. Closing the store disposes its models.

## What is the macOS Metal mitigation?
On Darwin, `GGML_METAL_NO_RESIDENCY=1` is active by default to avoid a libggml-metal static-destructor assertion at process exit. You can override it with `QMD_METAL_KEEP_RESIDENCY=1`.

## What files does AST chunking support?
TypeScript (`.ts`, `.mts`, `.cts`), TSX/JSX (`.tsx`, `.jsx`), JavaScript (`.js`, `.mjs`, `.cjs`), Python (`.py`), Go (`.go`), and Rust (`.rs`). Markdown is intentionally unsupported and always uses regex breakpoints.

## What is the difference between `search`, `query`, and `vsearch`?
- `search` — BM25 keyword search only. No LLM calls.
- `query` — Hybrid search with automatic expansion, RRF fusion, and optional reranking. Recommended default.
- `vsearch` — Vector-only similarity search. Defaults to `--min-score 0.3`.

## What happens to inactive documents?
Files that disappear from the filesystem during `qmd update` are soft-deactivated (`active = 0`). Their content rows remain until orphan cleanup removes unreferenced hashes. You can delete inactive documents explicitly via maintenance.
