# Synthesizer Task

Synthesize all explorer outputs into a unified L0/L1/L2 documentation of the QMD codebase. Read every explorer's output and produce the final documentation.

## Inputs to Read

Read ALL of these files before writing anything:

1. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/cli-explorer/output/findings.md`
2. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/cli-explorer/output/functions.md`
3. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/cli-explorer/output/diagrams.md`
4. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/store-explorer/output/findings.md`
5. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/store-explorer/output/functions.md`
6. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/store-explorer/output/diagrams.md`
7. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/db-coll-explorer/output/findings.md`
8. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/db-coll-explorer/output/functions.md`
9. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/db-coll-explorer/output/diagrams.md`
10. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/llm-ast-explorer/output/findings.md`
11. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/llm-ast-explorer/output/functions.md`
12. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/llm-ast-explorer/output/diagrams.md`
13. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/sdk-mcp-explorer/output/findings.md`
14. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/sdk-mcp-explorer/output/functions.md`
15. `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/sdk-mcp-explorer/output/diagrams.md`

## Scope

DO NOT modify any source files. DO NOT re-read source files — trust the explorer outputs. Read-only synthesis.

## Deliverables

Save ALL output files to `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/synthesizer/output/` — use absolute paths.

### 1. `L0-overview.md`

One-page system overview for someone who has never seen QMD:
- What QMD is (one paragraph)
- What problem it solves
- Key capabilities (bullet list)
- System architecture — single ASCII diagram showing all layers and data flow
- Tech stack summary

### 2. `L1-modules.md`

Module architecture map:
- Table of all modules with: name, file(s), responsibility, lines of code (approximate), dependencies
- Layer diagram (ASCII): CLI → SDK → Store → LLM → DB, with sidecars (Collections, AST, MCP, Maintenance)
- Interface boundaries: what each module exposes, what it consumes
- Data flow between modules (ASCII arrows)

### 3. `L2-implementation.md`

Deep implementation details organized by subsystem:
- ### Search Pipeline — step-by-step from query string to results, including all algorithms
- ### Chunking — regex + AST, with the actual scoring math explained
- ### Indexing & Storage — content-addressable design, document lifecycle
- ### Query Expansion — the LLM prompt and output format
- ### Hybrid Fusion — RRF formula, score normalization, candidate limits
- ### Configuration System — YAML → SQLite sync, local vs global
- ### Cross-Runtime Compatibility — Bun vs Node, macOS extensions

### 4. `architecture.asc`

Master ASCII architecture diagram — the one diagram to rule them all. Must include:
- All major components (CLI, SDK, MCP, Store, LLM, DB, Collections, AST, Maintenance)
- All data stores (YAML config, SQLite DB, GGUF models, markdown files)
- All external interfaces (stdio, HTTP, filesystem)
- Data flow arrows showing how a search query and an indexing job flow through the system

Use box-drawing characters: `+`, `-`, `|`, `>`, `^`, `v`, `/`, `\`, `=`.
Make it wide (80-120 chars) and tall enough to show all relationships clearly.

## Quality Standards

- All diagrams MUST be ASCII (no mermaid)
- Every claim must trace back to an explorer output
- Cross-reference explorer outputs: "(see store-explorer: Chunking section)"
- Write for a senior engineer who needs to understand QMD to contribute
- Include file paths for every mentioned function/constant
