# CLI Explorer Task

Explore the QMD CLI layer. Read the source files, understand every command, output format, and internal mechanism. Produce structured documentation.

## Files to Read

- `/home/sagar/temp/qmd/src/cli/qmd.ts` — main CLI entry (all commands, arg parsing, progress bars, colors)
- `/home/sagar/temp/qmd/src/cli/formatter.ts` — output formatters (json, csv, md, xml, files, cli)

## Scope

DO NOT modify any source files. DO NOT explore files outside `src/cli/`. Read-only analysis.

## Deliverables

Save ALL output files to `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/cli-explorer/output/` — use absolute paths.

### 1. `findings.md`

Structure:
- ## Overview — what the CLI layer does, its role in the system
- ## Commands — one section per command (collection, context, get, multi-get, search, query, vsearch, update, embed, status, mcp, ls). For each: purpose, key flags, implementation notes
- ## Key Mechanisms — progress bars, terminal colors, cursor control, ETA formatting, virtual path resolution, full-path rendering
- ## Data Flow — how a command flows from argv → parseArgs → store function → output
- ## Gotchas / Design Decisions — anything non-obvious (e.g., why process.exitCode = 0 instead of process.exit(0))

### 2. `functions.md`

Inventory of all exported/non-trivial functions with:
- Signature (params + return type)
- One-line purpose
- Which command(s) use it

### 3. `diagrams.md`

ASCII diagrams (NO mermaid):
- CLI command dispatch table (command → handler function)
- Output format pipeline (SearchResult[] → formatter → stdout)
- Lifecycle diagram (getStore → getDb → command → closeDb)

Use box-drawing characters: `+`, `-`, `|`, `>`, `^`, `v`.
