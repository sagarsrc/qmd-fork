---
title: "codebase exploration"
experiment: 001-understanding-codebase
created: "2026-06-08 06:19 UTC"
---

# QMD Codebase Exploration Plan

## Goal
Produce L0 (system overview), L1 (module architecture), and L2 (implementation detail) documentation of the QMD codebase. Use ASCII diagrams primarily. Parallel exploration via dag-fleet.

## L0 — System Overview (What QMD is)

QMD = Query Markup Documents. On-device hybrid search over markdown files.

```
+------------------+     +------------------+     +------------------+
|   User Input     |     |   Markdown Files |     |   GGUF Models    |
|  (CLI / MCP /    |     |  (Collections)   |     |  (Local LLMs)    |
|   SDK)           |     |                  |     |                  |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         v                        v                        v
+--------+---------------------------------------------------------+
|                           QMD Engine                             |
|  +----------------+  +----------------+  +---------------------+ |
|  |  CLI (qmd.ts)  |  |  SDK (index.ts)|  |  MCP (server.ts)    | |
|  +--------+-------+  +--------+-------+  +----------+----------+ |
|           |                   |                    |              |
|           v                   v                    v              |
|  +-----------------------------------------------------------+   |
|  |                     Store (store.ts)                       |   |
|  |  - Chunking (regex + AST)                                  |   |
|  |  - BM25 search (FTS5)                                      |   |
|  |  - Vector search (sqlite-vec)                              |   |
|  |  - Hybrid fusion (RRF)                                     |   |
|  |  - LLM reranking                                           |   |
|  +---------------------------+--------------------------------+   |
|                              |                                    |
|  +---------------------------v--------------------------------+   |
|  |                     LLM Layer (llm.ts)                     |   |
|  |  - Embeddings (node-llama-cpp)                             |   |
|  |  - Query expansion                                         |   |
|  |  - Reranking                                               |   |
|  +---------------------------+--------------------------------+   |
|                              |                                    |
|  +---------------------------v--------------------------------+   |
|  |                     DB Layer (db.ts)                       |   |
|  |  - SQLite (bun:sqlite / better-sqlite3)                    |   |
|  |  - sqlite-vec extension                                    |   |
|  +-----------------------------------------------------------+   |
|                                                                   |
+-------------------------------------------------------------------+
         |
         v
+------------------+
|  YAML Config     |
|  (collections.ts)|
|  ~/.config/qmd/  |
+------------------+
```

## L1 — Module Breakdown

| Module | File(s) | Responsibility |
|--------|---------|----------------|
| CLI | `src/cli/qmd.ts`, `src/cli/formatter.ts` | Argument parsing, all commands, output formatting |
| Store | `src/store.ts` | Core engine: chunking, search, indexing, embeddings |
| DB | `src/db.ts` | SQLite compatibility (Bun + Node), extension loading |
| Collections | `src/collections.ts` | YAML config I/O, collection/context management |
| LLM | `src/llm.ts` | Model lifecycle, embeddings, generation, reranking |
| AST | `src/ast.ts` | Tree-sitter chunking for code files (.ts/.js/.py/.go/.rs) |
| SDK | `src/index.ts` | Programmatic API: `createStore()` |
| MCP | `src/mcp/server.ts` | Model Context Protocol server (stdio + HTTP) |
| Maintenance | `src/maintenance.ts` | Housekeeping: vacuum, orphaned cleanup |
| Paths | `src/paths.ts` | Path resolution utilities |

## L2 — Implementation Detail Strategy

Each module gets a dedicated worker that reads its source files and produces:
1. **Function inventory** — all exported/public functions with signatures
2. **Data structures** — key types, interfaces, schemas
3. **Algorithms** — how chunking works, how search fusion works, how embeddings work
4. **State flow** — how data moves through the module
5. **ASCII diagrams** — architecture of that module

## Parallel Exploration Fleet

```
                    +------------------+
                    |   dag-fleet      |
                    +--------+---------+
                             |
         +-------------------+-------------------+
         |                   |                   |
         v                   v                   v
  +-------------+     +-------------+     +-------------+
  | cli-explorer|     |store-explorer|    |db-coll-explorer
  |  (write)    |     |  (write)    |     |  (write)    |
  +------+------+     +------+------+     +------+------+
         |                   |                   |
         v                   v                   v
  +-------------+     +-------------+     +-------------+
  |llm-ast-explorer     |sdk-mcp-explorer    | (test-explorer)
  |  (write)    |     |  (write)    |     |  (write)    |
  +------+------+     +------+------+     +------+------+
         |                   |                   |
         +-------------------+-------------------+
                             |
                             v
                    +------------------+
                    |   synthesizer    |
                    |    (write)       |
                    |  depends_on: all |
                    +------------------+
```

### Worker Assignments

| Worker | Files | Focus |
|--------|-------|-------|
| `cli-explorer` | `src/cli/qmd.ts`, `src/cli/formatter.ts` | All CLI commands, arg parsing, progress bars, color output, formatters (json/csv/md/xml/files) |
| `store-explorer` | `src/store.ts` | Chunking (regex + smart breakpoints), BM25/vector/hybrid search, RRF fusion, reindexing, embeddings pipeline, virtual paths, docids |
| `db-coll-explorer` | `src/db.ts`, `src/collections.ts` | Cross-runtime SQLite, extension loading, schema init, YAML config, store_collections sync |
| `llm-ast-explorer` | `src/llm.ts`, `src/ast.ts` | Model loading, GPU/CPU fallback, embedding formatting, query expansion, reranking, tree-sitter grammar loading, AST breakpoints |
| `sdk-mcp-explorer` | `src/index.ts`, `src/mcp/server.ts`, `src/paths.ts`, `src/maintenance.ts` | SDK API surface, MCP tools/resources/prompts, path utils, maintenance ops |
| `synthesizer` | All worker outputs | L0 overview, L1 module map, L2 deep dives, ASCII architecture diagrams, cross-module data flow |

### Worker Types

All explorers: `write` type (they need to write findings to output files)
Synthesizer: `write` type (writes final documentation)

### Models

- Explorers: `sonnet` (good balance of speed and depth for code analysis)
- Synthesizer: `opus` (needs largest context to synthesize all outputs)

### Budgets

- Explorers: $2.00 each (code analysis of large files)
- Synthesizer: $5.00 (needs to read all outputs + produce final docs)

## Output Format

Each explorer produces:
- `findings.md` — module-specific findings
- `functions.md` — function inventory with signatures
- `diagrams.md` — ASCII diagrams

Synthesizer produces:
- `L0-overview.md` — 10,000 ft view
- `L1-modules.md` — Module architecture map
- `L2-implementation.md` — Deep implementation details
- `architecture.asc` — Master ASCII architecture diagram

## Launch Commands

```bash
# Generate fleet
# (fleet.json + prompts auto-generated)

# Launch
bash /home/sagar/.pi/agent/skills/dag-fleet/scripts/launch.sh <fleet-root>

# Monitor
bash /home/sagar/.pi/agent/skills/dag-fleet/scripts/status.sh <fleet-root>

# Report when done
bash /home/sagar/.pi/agent/skills/dag-fleet/scripts/report.sh <fleet-root>
```
