# How to Use QMD

## How do I install QMD?
Clone or download the repo, then install dependencies and link the CLI globally with Bun:

```sh
bun install
bun link
```

This makes the `qmd` command available in your shell (see CLAUDE.md: Development).

## How do I index my first folder?
1. Add a collection:
   ```sh
   qmd collection add ~/Documents/notes --name notes
   ```
2. Index the files:
   ```sh
   qmd update
   ```
3. Generate embeddings for semantic search:
   ```sh
   qmd embed
   ```

`qmd update` scans the collection, hashes contents, builds the FTS index, and deactivates missing files. `qmd embed` creates vector embeddings for each chunk (see cli-explorer: `collection add`, `update`, `embed`; store-explorer: Indexing).

## How do I search?
- **Keyword (BM25):**
  ```sh
  qmd search "database migration"
  ```
- **Vector (semantic only):**
  ```sh
  qmd vsearch "how to handle concurrency"
  ```
- **Hybrid (recommended):**
  ```sh
  qmd query "refactoring large react components"
  ```

Hybrid search automatically expands your query, runs both lexical and vector searches, fuses the results, and reranks them (see cli-explorer: `search`, `vsearch`, `query`; store-explorer: Hybrid Search).

## How do I retrieve a document?
By virtual path:
```sh
qmd get notes/ideas.md
```

By docid (shown in search results as `#abc123`):
```sh
qmd get "#abc123"
```

By line range:
```sh
qmd get notes/ideas.md:10:20
```

Multiple documents by glob or comma-separated list:
```sh
qmd multi-get "notes/*.md"
qmd multi-get "#abc123, #def456"
```

Oversized files are skipped rather than failing the command (see cli-explorer: `get`, `multi-get`; store-explorer: Docids).

## How do I add context to improve search?
Global context (applies to all collections):
```sh
qmd context add / "Always prefer recent entries."
```

Collection context:
```sh
qmd context add notes "These are personal brainstorming notes."
```

Virtual path context:
```sh
qmd context add qmd://notes/2024 "Journal entries from 2024."
```

Context is inherited by longest-prefix match and falls back to global (see cli-explorer: `context`; db-coll-explorer: Context System).

## How do I update the index when files change?
Re-index everything:
```sh
qmd update
```

Then embed any new or changed content:
```sh
qmd embed
```

If a collection has a configured `update` shell command, `qmd update` runs it before re-indexing (see cli-explorer: `update`; store-explorer: Indexing).

## How do I use the SDK in my code?
```ts
import { createStore } from 'qmd';

const store = await createStore({
  dbPath: '/path/to/index.sqlite',
  configPath: '/path/to/.qmd/index.yaml',
});

const results = await store.search({ query: 'error handling patterns' });
for (const r of results) {
  console.log(r.file, r.score);
}

await store.close();
```

The SDK supports DB-only mode (no config source), inline config, and write-through mutations (see sdk-mcp-explorer: SDK Overview).

## How do I use the MCP server?
- **Stdio** (for clients like Claude Desktop):
  ```sh
  qmd mcp
  ```
- **HTTP foreground:**
  ```sh
  qmd mcp --http --port 8181
  ```
- **HTTP daemon:**
  ```sh
  qmd mcp --http --daemon
  qmd mcp stop
  ```

The MCP server exposes `query`, `get`, `multi_get`, and `status` tools, plus `qmd://{+path}` resources (see cli-explorer: `mcp`; sdk-mcp-explorer: MCP Overview).

## What output formats are supported?
`json`, `csv`, `md`, `xml`, and `files`. Use `--format`:

```sh
qmd search "deploy" --format json
qmd query "auth flow" --format md
qmd multi-get "notes/*.md" --format files
```

Legacy boolean flags `--json`, `--csv`, `--md`, `--xml`, `--files` still work but are undocumented (see cli-explorer: Output Format Pipeline).

## What are virtual paths?
Virtual paths are canonical identifiers of the form `qmd://collection/path/to/file.md`. They appear in search results, `get` output, and MCP resources. You can use them anywhere a file path is accepted (see cli-explorer: Virtual path resolution; store-explorer: Virtual Paths).
