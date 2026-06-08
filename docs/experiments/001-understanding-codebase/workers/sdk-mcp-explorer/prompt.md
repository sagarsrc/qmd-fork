# SDK + MCP + Support Explorer Task

Explore the QMD SDK API, MCP server, and support utilities. Read all source files and understand the programmatic interface and protocol server.

## Files to Read

- `/home/sagar/temp/qmd/src/index.ts` — SDK API: createStore(), QMDStore interface
- `/home/sagar/temp/qmd/src/mcp/server.ts` — MCP server (stdio + HTTP transports)
- `/home/sagar/temp/qmd/src/paths.ts` — path resolution utilities
- `/home/sagar/temp/qmd/src/maintenance.ts` — housekeeping operations

## Scope

DO NOT modify any source files. Read-only analysis.

## Deliverables

Save ALL output files to `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/sdk-mcp-explorer/output/` — use absolute paths.

### 1. `findings.md`

Structure:
- ## SDK Overview — design goals, StoreOptions, QMDStore interface, per-store LlamaCpp instance
- ## SDK Search API — search(), searchLex(), searchVector(), expandQuery() — how they map to store internals
- ## SDK Collection Management — addCollection, removeCollection, renameCollection, listCollections — SQLite + YAML write-through
- ## SDK Context Management — addContext, removeContext, setGlobalContext — SQLite + YAML write-through
- ## SDK Indexing — update(), embed() — collection filtering, progress callbacks
- ## SDK Lifecycle — close() — model disposal + DB close + config source reset
- ## MCP Overview — server initialization, stdio vs HTTP transports, tool/resource/prompt registration
- ## MCP Tools — search, get, multi_get, status — parameters, return types, error handling
- ## MCP Resources — qmd:// document access, read-only, no list() by design
- ## MCP Instructions — buildInstructions() — dynamic system prompt from index state
- ## Paths — qmdHomedir(), cross-platform home dir resolution
- ## Maintenance — Maintenance class: vacuum, orphaned content cleanup, orphaned vector cleanup, LLM cache deletion
- ## Gotchas / Design Decisions — why SDK has its own LlamaCpp per store, why MCP instructions are dynamic, write-through pattern for SDK mutations

### 2. `functions.md`

Inventory of all exported functions/types in all four files with:
- Signature (params + return type)
- One-line purpose

### 3. `diagrams.md`

ASCII diagrams (NO mermaid):
- SDK usage flow: createStore(config) → search/update/embed → close
- SDK internal architecture: QMDStore → InternalStore + LlamaCpp + DB
- MCP architecture: McpServer → tools + resources + prompts → stdio/HTTP transport
- MCP tool call flow: client request → tool handler → store.search → format results → response
- Maintenance operations flow

Use box-drawing characters: `+`, `-`, `|`, `>`, `^`, `v`.
