# DB + Collections Explorer Task

Explore the QMD database compatibility layer and collections configuration layer. Read both source files and understand the cross-runtime abstractions and YAML config system.

## Files to Read

- `/home/sagar/temp/qmd/src/db.ts` — SQLite compatibility (Bun + Node), extension loading
- `/home/sagar/temp/qmd/src/collections.ts` — YAML config I/O, collection/context management

## Scope

DO NOT modify any source files. Read-only analysis.

## Deliverables

Save ALL output files to `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/db-coll-explorer/output/` — use absolute paths.

### 1. `findings.md`

Structure:
- ## DB Layer Overview — why a cross-runtime abstraction exists
- ## Bun vs Node Path — dynamic import of `bun:sqlite` vs `better-sqlite3`, API differences handled
- ## macOS SQLite Extension Loading — setCustomSQLite for Homebrew SQLite, fallback behavior
- ## sqlite-vec Loading — how the extension is loaded on both runtimes, failure modes
- ## Database Interface — the common subset (exec, prepare, transaction, close)
- ## Collections Layer Overview — YAML as source of truth, why it mirrors into SQLite
- ## Config Resolution — XDG dirs, local `.qmd/index.yaml` discovery, index naming
- ## Config Sync — syncConfigToDb: hash-based optimization, collections upsert/delete, global context
- ## Context System — per-path context within collections, global context, YAML persistence
- ## Gotchas / Design Decisions — why YAML not pure SQLite, config hash caching, inline config mode for SDK

### 2. `functions.md`

Inventory of all exported functions in both files with:
- Signature (params + return type)
- One-line purpose

### 3. `diagrams.md`

ASCII diagrams (NO mermaid):
- DB initialization flow: detect runtime → load driver → test extension → open database
- Config load flow: check inline → check custom path → check local → check global → parse YAML
- Config sync flow: load YAML → hash → compare stored hash → upsert/delete collections → update store_config
- Collections + contexts data model (YAML structure visualized)

Use box-drawing characters: `+`, `-`, `|`, `>`, `^`, `v`.
