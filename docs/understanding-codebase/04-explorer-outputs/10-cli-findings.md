# CLI Layer Findings

## Overview

The QMD CLI layer is the command-line front door for indexing, searching, retrieving, maintaining, and serving QMD's markdown document index. `src/cli/qmd.ts` owns argument parsing, command dispatch, terminal UX, lifecycle cleanup, YAML-to-SQLite config synchronization, and command-specific orchestration. It delegates durable index behavior to `store.ts`, collection config behavior to `collections.ts`, LLM/model behavior to `llm.ts`, and MCP serving to `mcp/server.ts`.

`src/cli/formatter.ts` is a focused formatting utility for converting search results and document results into `json`, `csv`, `md`, `xml`, `files`, or fallback CLI-friendly text. The main CLI uses its escape helpers and imports universal formatters, but search output is mostly rendered by `outputResults()` in `qmd.ts` because it needs terminal colors, docids, clickable editor links, `--full-path`, line-number handling, and explain traces.

The CLI is intentionally lazy about opening the store. `getStore()` creates the store only when a command needs it, syncs YAML config into SQLite, configures default llama.cpp models, and reuses the handle until `closeDb()` is called. The module does not call `enableProductionMode()` at import time; production mode is enabled only under the main-module guard so tests can import exported helpers without changing global path behavior.

## Commands

### collection

Purpose: Manage indexed folders/collections and their YAML-backed settings.

Key flags/subcommands:
- `collection list`: print configured/indexed collections with pattern, ignore patterns, active file count, update age, and `[excluded]` marker when `includeByDefault === false`.
- `collection add <path> [--name NAME] [--mask GLOB]`: add a YAML collection, sync config to DB, then index files.
- `collection remove|rm <name>`: remove DB documents and YAML config.
- `collection rename|mv <old> <new>`: rename in DB and YAML; virtual path prefix changes from `qmd://old/` to `qmd://new/`.
- `collection update-cmd|set-update <name> [command]`: set or clear pre-update shell command.
- `collection include|exclude <name>`: toggle whether default queries include the collection.
- `collection show|info <name>`: print YAML collection details.

Implementation notes: collection config is canonical in YAML. Mutating commands call YAML helpers and, where needed, `resyncConfig()` so SQLite `store_collections` follows the YAML config. `collection add` rejects duplicate names and duplicate path+glob pairs before indexing.

### context

Purpose: Attach human-written context strings to global scope, collection roots, or collection subpaths.

Key flags/subcommands:
- `context add [path] "text"`: add context; with one argument, the path defaults to current directory.
- `context add / "text"`: set global context.
- `context add qmd://collection/path "text"`: set context for a virtual path.
- `context list`: print contexts grouped by collection.
- `context rm|remove <path>`: remove global, virtual-path, or filesystem-detected context.

Implementation notes: filesystem paths are normalized with `getPwd()`, `homedir()`, and `resolve()`. The CLI maps filesystem paths back to collections with `detectCollectionFromPath()`, using longest-prefix matching over YAML collections. Virtual paths use `parseVirtualPath()` and YAML collection lookup before mutation.

### get

Purpose: Fetch and print a single indexed document.

Key flags:
- `--from <line>`: start line.
- `-l <lines>`: max lines.
- `--no-line-numbers`: disables default line-numbered output.
- `--full-path`: show on-disk path rather than `qmd://... #docid` when resolvable.
- Inline suffixes: `<file>:100` and `<file>:100:40` set start line and line count unless explicit flags override.

Implementation notes: lookup accepts docids, virtual paths, `collection/path` shorthand, filesystem paths under a configured collection, and fuzzy filename suffixes. Output header is either canonical `qmd://collection/path  #docid` or a rendered filesystem path. Context is printed as `Folder Context: ...`, followed by `---` and the document body. Line numbers are on by default.

### multi-get

Purpose: Fetch multiple indexed documents by glob or comma-separated list.

Key flags:
- `-l <lines>`: max lines per file.
- `--max-bytes <bytes>`: skip files larger than the threshold; default is `DEFAULT_MULTI_GET_MAX_BYTES`.
- `--no-line-numbers`: disables default line-numbered output.
- `--full-path`: use filesystem paths where resolvable and omit docids.
- `--format cli|json|csv|md|xml|files`: choose structured output.

Implementation notes: comma-separated input is used when the pattern contains commas and no glob metacharacters. Each name can be a virtual path or a path/suffix. Glob input is resolved by `matchFilesByGlob()`, which returns virtual paths. Oversized files are represented as skipped entries rather than failing the command. Unlike `formatter.ts`'s generic `formatDocuments()`, this command has custom output so it can include docids and full-path policy.

### search

Purpose: Fast lexical/full-text BM25 search without LLM calls.

Key flags:
- `-n <num>`, `--all`, `--min-score <num>`.
- `--full`, `--line-numbers`, `--full-path`.
- `--format cli|json|csv|md|xml|files`.
- `-c, --collection <name>` multiple allowed; defaults to configured default collections.

Implementation notes: `search()` validates collection filters, calls `searchFTS()`, adds folder context with `getContextForFile()`, closes the DB, then calls `outputResults()`. For multiple collection filters, the CLI post-filters by `qmd://<collection>/` prefixes. Empty output is format-safe: JSON emits `[]`, CSV emits a header, XML emits `<results></results>`, and `md/files` emit nothing.

### query

Purpose: Recommended hybrid search with automatic query expansion, vector search, lexical search, reciprocal-rank style blending, and optional reranking.

Key flags:
- Search/output flags from `search`.
- `-C, --candidate-limit <n>`: cap candidates to rerank.
- `--no-rerank`: skip LLM reranking.
- `--no-gpu`: sets `QMD_FORCE_CPU=1`.
- `--intent <text>`: disambiguation hint.
- `--chunk-strategy <auto|regex>`.
- `--explain`: include score traces in CLI and JSON.

Implementation notes: `parseStructuredQuery()` detects query documents. A plain single-line query uses automatic expansion via `hybridQuery()`. A structured document with `lex:`, `vec:`, `hyde:`, and optional `intent:` calls `structuredSearch()` directly. Progress hooks write expansion, embedding, and reranking status to stderr; results go through `outputResults()`.

### vsearch

Purpose: Vector-only similarity search.

Key flags: same output/filter flags as `search`; default `--min-score` is set to `0.3` if the user does not provide one.

Implementation notes: dispatch aliases `vector-search` to `vsearch`. The command checks index health, runs inside `withLLMSession()`, calls `vectorSearchQuery()`, post-filters multi-collection output, and renders through `outputResults()`.

### update

Purpose: Re-index all configured collections and clear cached LLM/query results.

Key flags: help says `--pull`, but implementation reads collection-specific YAML update commands instead of using the parsed global `pull` flag in the shown code.

Implementation notes: `updateCollections()` clears cache, lists collections, runs any per-collection `update` shell command via `bash -c` in the collection working directory, then calls `reindexCollection()` with progress hooks. It prints per-collection counts for indexed, updated, unchanged, removed, and orphan cleanup. At the end it suggests `qmd embed` if hashes need vectors.

### embed

Purpose: Generate or refresh embeddings for content hashes.

Key flags:
- `-f, --force`: clear/regenerate vectors.
- `-c, --collection <name>`: embed only one validated collection; only the first value is used.
- `--max-docs-per-batch <n>`.
- `--max-batch-mb <n>`.
- `--chunk-strategy <auto|regex>`.

Implementation notes: dispatch parses and validates numeric batch options, then calls `vectorIndex()`. `vectorIndex()` checks pending hashes, hides the cursor, uses OSC progress and a text progress bar, tracks bytes processed rather than discovered chunks, and prints chunk/document counts plus retry failures.

### status

Purpose: Print index health and configuration summary.

Key output:
- DB path and size.
- MCP daemon liveness from cache PID file.
- document count, vector count, pending embeddings, latest update age.
- AST chunking availability and language status.
- collections with virtual path examples, patterns, file counts, contexts.
- active embed/rerank/generation model links.
- tips for missing collection contexts or update commands.

Implementation notes: status reads both SQLite stats and YAML contexts/settings. It silently removes stale MCP PID files. Model strings using `hf:org/repo/file.gguf` are rendered as Hugging Face repository links.

### mcp

Purpose: Start, daemonize, or stop the MCP server.

Key flags/subcommands:
- `qmd mcp`: start stdio MCP server.
- `qmd mcp --http [--port <port>]`: start foreground HTTP server, default port `8181`.
- `qmd mcp --http --daemon [--port <port>]`: spawn detached server, write PID/log files.
- `qmd mcp stop`: terminate daemon by PID file or clean stale PID.

Implementation notes: PID and log files live under `${XDG_CACHE_HOME}/qmd` or `~/.cache/qmd`. HTTP foreground mode removes the CLI's top-level SIGINT/SIGTERM handlers so MCP server cleanup handlers can run. Daemon mode re-invokes the current CLI file, using `tsx` import args for `.ts` entrypoints.

### ls

Purpose: Inspect the virtual indexed file tree.

Key forms:
- `qmd ls`: list collections as `qmd://collection/` with file counts.
- `qmd ls collection`.
- `qmd ls collection/path`.
- `qmd ls qmd://collection/path`.
- Raw absolute paths and absolute-path collection names are handled with longest-prefix matching.

Implementation notes: `listFiles()` parses virtual paths carefully, including `qmd:///absolute/path` forms so leading slashes are preserved for absolute-path collection names. Output resembles `ls -l`: formatted size, date/time or year, and a colored `qmd://collection/path`.

## Key Mechanisms

### Progress bars

Two progress channels are used. `progress` writes OSC 9;4 escape sequences to stderr when stderr is a TTY: set percent, clear, indeterminate, and error. Human-readable inline progress also writes carriage-return lines to stderr. `update` and `indexFiles()` show `Indexing: current/total ETA`, while `embed` shows a fixed-width text bar plus byte throughput, chunk count, errors, and ETA.

### Terminal colors

Color is enabled only when `process.stdout.isTTY` and `NO_COLOR` is not set. The `c` object centralizes ANSI reset, dim, bold, cyan, yellow, green, magenta, and blue. Search highlighting is disabled entirely when color is off.

### Cursor control

`cursor.hide()` and `cursor.show()` write ANSI private mode sequences to stderr. The CLI restores the cursor on SIGINT and SIGTERM. `embed` hides the cursor during long vector generation and shows it afterward.

### ETA formatting

`formatETA()` emits seconds as `Ns`, minutes as `Nm Ns`, and hours as `Nh Nm`. It is used by indexing and embedding progress. `formatTimeAgo()` is separate and emits `s/m/h/d ago` for status/list displays.

### Virtual path resolution

The CLI treats `qmd://collection/path` as the canonical document identifier. `parseVirtualPath()`, `buildVirtualPath()`, `isVirtualPath()`, `resolveVirtualPath()`, and `toVirtualPath()` come from `store.ts`. The CLI also accepts docid aliases, `collection/path` shorthand, filesystem paths, and suffix matches in retrieval commands. `--index` and virtual paths with index metadata can switch the active DB/config index.

### Full-path rendering

`renderFullPath()` resolves symlinks where possible and compares the file against the current working directory. If the file is under `$PWD`, output is `./relative/path`; if it is the cwd, output is `./`; otherwise it emits the absolute realpath. `--full-path` uses this policy in `get`, `multi-get`, `search`, `query`, and `vsearch`; docids are usually omitted because the filesystem path becomes the visible identifier.

### Clickable editor links

CLI search output can wrap visible paths in OSC 8 hyperlinks when stdout is a TTY and the file resolves. `QMD_EDITOR_URI` or config keys such as `editor_uri` can override the default `vscode://file/{path}:{line}:{col}` template. `buildEditorUri()` percent-encodes paths and fills `{path}`, `{line}`, `{col}`, and `{column}`.

## Data Flow

1. Process starts under the main-module guard and calls `enableProductionMode()`.
2. `parseCLI()` calls `util.parseArgs()` on `process.argv.slice(2)`.
3. Global index selection is applied: explicit `--index`, project-local `.qmd/index.yaml`, or default global config.
4. Output format is resolved from `--format` or legacy boolean aliases.
5. Dispatch switch selects the command and extracts command-specific flags.
6. Command opens the store lazily through `getStore()` or `getDb()`.
7. `getStore()` creates `createStore()`, loads config, syncs YAML into SQLite, and configures llama.cpp defaults.
8. Command calls store/collection/LLM/MCP functions.
9. Result rows are decorated with context, docid, snippets, full-path labels, or explain traces.
10. Output is printed directly or through formatter helpers.
11. Most commands call `closeDb()` directly or before rendering; the main block then calls `finishSuccessfulCliCommand()` for non-MCP commands.

## Gotchas / Design Decisions

- Successful command completion deliberately avoids `process.exit(0)`. `finishSuccessfulCliCommand()` flushes stdout, disposes llama.cpp, flushes stderr, and sets `process.exitCode = 0` so Node's `beforeExit` hooks can run. This avoids a macOS ggml-metal destructor assertion documented in comments.
- Error paths still use `process.exit()` in many command handlers, so the no-direct-exit policy is specifically for successful post-output lifecycle.
- `enableProductionMode()` is not run at import time. Tests can import exported helpers without changing global DB path behavior.
- YAML config is treated as canonical for collections and contexts, with SQLite sync used so store queries can read collection metadata from DB.
- `--format` is the preferred output selector; `--csv`, `--md`, `--xml`, `--files`, and `--json` remain as undocumented compatibility aliases.
- Search line numbers default off unless `--line-numbers` is passed, but `get` and `multi-get` line numbers default on and require `--no-line-numbers` to disable.
- `--full-path` changes identifier semantics: docids are omitted in most structured/search outputs because the on-disk path is considered the identifier.
- `qmd ls` has special handling for absolute-path collection names and `qmd:///...` so path normalization does not strip meaningful leading slashes.
- `query` structured syntax rejects ambiguous multi-line plain input. Every line in a query document must be typed with `lex:`, `vec:`, `hyde:`, or `intent:`.
- `vsearch` has an implicit default min score of `0.3`; `query` and `search` default to `0` unless specified.
- Some imported/general formatter functions are not used for the richer CLI render path because `qmd.ts` has to account for docids, hyperlinks, explain traces, full-path output, and terminal color.
