# QMD Command Journeys

## collection add

```mermaid
flowchart TD
    A["qmd collection add /path<br/>--name notes --mask '**/*.md'"] --> B["resolve absolute path"]
    B --> C{"name already exists?<br/>or path+glob duplicate?"}
    C -->|yes| D["exit 1 with error"]
    C -->|no| E["addCollection() → YAML"]
    E --> F["resyncConfig()<br/>syncConfigToDb()"]
    F --> G["indexFiles()<br/>fast-glob → chunk → insert"]
    G --> H["done"]
```

- **What it does:** Registers a new collection in YAML config, syncs to SQLite, and indexes files immediately
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`, `src/store.ts`
- **Side effects:** Writes `~/.config/qmd/index.yml`, creates `documents`/`content` rows, rebuilds FTS, clears `llm_cache`

---

## collection list

```mermaid
flowchart TD
    A["qmd collection list"] --> B["getDb() → listCollections(db)"]
    B --> C["yamlListCollections()"]
    C --> D["for each: count active docs<br/>get YAML includeByDefault"]
    D --> E["print name + virtual path<br/>+ pattern + file count + updated ago<br/>+ [excluded] tag"]
    E --> F["done"]
```

- **What it does:** Prints configured collections from YAML with live DB stats
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`
- **Side effects:** None (read-only)

---

## collection remove

```mermaid
flowchart TD
    A["qmd collection remove notes"] --> B{"exists in YAML?"}
    B -->|no| C["exit 1"]
    B -->|yes| D["removeCollection(db, name)<br/>delete docs + deactivate content"]
    D --> E["yamlRemoveCollectionFn(name)"]
    E --> F["closeDb()"]
    F --> G["print deleted doc count<br/>+ orphaned hash cleanup"]
    G --> H["done"]
```

- **What it does:** Removes collection from DB, YAML, and cleans orphaned content hashes
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`, `src/store.ts`
- **Side effects:** Deletes `documents` rows, updates `store_collections`, writes YAML

---

## collection rename

```mermaid
flowchart TD
    A["qmd collection rename old new"] --> B{"old exists?<br/>new taken?"}
    B -->|fail| C["exit 1"]
    B -->|ok| D["renameCollection(db, old, new)<br/>UPDATE documents.collection"]
    D --> E["yamlRenameCollectionFn(old, new)"]
    E --> F["closeDb()"]
    F --> G["print virtual path change"]
    G --> H["done"]
```

- **What it does:** Renames collection in DB and YAML; virtual paths change from `qmd://old/` to `qmd://new/`
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`, `src/store.ts`
- **Side effects:** Updates `documents.collection`, `store_collections.name`, writes YAML

---

## ls

```mermaid
flowchart TD
    A["qmd ls [collection[/path]]"] --> B{"arg provided?"}
    B -->|no| C["yamlListCollections()<br/>print qmd://name/ + file count"]
    B -->|yes| D["parse virtual/fs/absolute path"]
    D --> E["find collection + prefix"]
    E --> F["SELECT active docs<br/>WHERE collection = ?<br/>AND path LIKE prefix%"]
    F --> G["print ls -l style<br/>size + date + virtual path"]
    G --> H["done"]
```

- **What it does:** Lists collections or files inside a collection virtual tree
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`
- **Side effects:** None (read-only)

---

## context add

```mermaid
flowchart TD
    A["qmd context add [path] 'text'"] --> B{"path == '/'?"}
    B -->|yes| C["setGlobalContext(text)<br/>resyncConfig()"]
    B -->|no| D["resolve fs or virtual path"]
    D --> E{"virtual path?"}
    E -->|yes| F["yamlAddContext(coll, path, text)<br/>resyncConfig()"]
    E -->|no| G["detectCollectionFromPath(db, fsPath)"]
    G --> H{"found?"}
    H -->|no| I["exit 1"]
    H -->|yes| F
    F --> J["closeDb() + print ok"]
    J --> K["done"]
```

- **What it does:** Attaches context text to global scope, collection root, or subpath
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`, `src/store.ts`
- **Side effects:** Writes YAML, syncs to `store_collections.context` or `store_config.global_context`

---

## context list

```mermaid
flowchart TD
    A["qmd context list"] --> B["getDb()"]
    B --> C["listAllContexts()<br/>global + per-collection"]
    C --> D["group by collection name"]
    D --> E["print collection<br/>→ path prefix + context preview"]
    E --> F["done"]
```

- **What it does:** Prints all contexts grouped by collection
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`
- **Side effects:** None (read-only)

---

## context rm

```mermaid
flowchart TD
    A["qmd context rm <path>"] --> B{"path == '/'?"}
    B -->|yes| C["setGlobalContext(undefined)<br/>resyncConfig()"]
    B -->|no| D{"virtual path?"}
    D -->|yes| E["yamlRemoveContext(coll, path)"]
    D -->|no| F["detectCollectionFromPath(db, fsPath)"]
    F --> G["yamlRemoveContext(coll, relPath)"]
    E --> H["print ok"]
    G --> H
    H --> I["done"]
```

- **What it does:** Removes context entry from YAML and syncs to DB
- **Key files:** `src/cli/qmd.ts`, `src/collections.ts`
- **Side effects:** Writes YAML, updates SQLite synced config

---

## get

```mermaid
flowchart TD
    A["qmd get <arg> [--from N] [-l N]"] --> B["parse :line:count suffix"]
    B --> C{"docid? (#abc123)"}
    C -->|yes| D["findDocumentByDocid(db)<br/>→ virtual path"]
    C -->|no| E{"virtual path qmd://...?"}
    E -->|yes| F["SELECT exact match<br/>OR fuzzy path LIKE"]
    E -->|no| G{"collection/path shorthand?"}
    G -->|yes| F
    G -->|no| H["resolve filesystem path<br/>→ toVirtualPath()"]
    H --> I{"found in collection?"}
    I -->|no| J["exit 1"]
    I -->|yes| F
    F --> K["getContextForFile()"]
    K --> L["print header + context + body<br/>with optional line numbers"]
    L --> M["done"]
```

- **What it does:** Fetches a single document by docid, virtual path, collection shorthand, or filesystem path
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`
- **Side effects:** None (read-only)

---

## multi-get

```mermaid
flowchart TD
    A["qmd multi-get <pattern>"] --> B{"comma-separated?<br/>no glob chars"}
    B -->|yes| C["split by comma<br/>for each: virtual/suffix lookup"]
    B -->|no| D["matchFilesByGlob(db)<br/>→ virtual paths"]
    C --> E["for each file:<br/>resolve collection+path<br/>get context + docid<br/>optionally skip >maxBytes"]
    D --> E
    E --> F["format: cli/json/csv/md/xml/files"]
    F --> G["done"]
```

- **What it does:** Retrieves multiple documents by comma list or glob pattern
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`
- **Side effects:** None (read-only)

---

## status

```mermaid
flowchart TD
    A["qmd status"] --> B["getDb() + getDbPath()"]
    B --> C["DB size + total docs<br/>vector count + pending embeds<br/>latest update age"]
    C --> D["MCP PID file check<br/>kill(pid, 0) → running/stale"]
    D --> E["AST chunking status<br/>language availability"]
    E --> F["for each YAML collection:<br/>print stats + contexts<br/>+ model links + tips"]
    F --> G["done"]
```

- **What it does:** Prints index health, collections, models, contexts, and actionable tips
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`, `src/ast.ts`
- **Side effects:** Silently deletes stale MCP PID file if found

---

## update

```mermaid
flowchart TD
    A["qmd update [--pull]"] --> B["getDb() + clearCache(db)"]
    B --> C["listCollections(db)"]
    C --> D{"collections == 0?"}
    D -->|yes| E["print hint + return"]
    D -->|no| F["for each collection:"]
    F --> G["run YAML update_command<br/>via bash -c in cwd"]
    G --> H["reindexCollection()<br/>fast-glob → per file:<br/>hash → compare → insert/update/<br/>deactivate missing<br/>cleanupOrphanedContent()"]
    H --> I["progress via OSC 9;4"]
    I --> J["print indexed/updated/<br/>unchanged/removed counts"]
    J --> K{"more collections?"}
    K -->|yes| F
    K -->|no| L["check hashesNeedingEmbedding<br/>suggest embed if >0"]
    L --> M["done"]
```

- **What it does:** Re-indexes all collections, runs per-collection update commands, cleans orphans
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`, `src/collections.ts`
- **Side effects:** Writes `documents`, `content`, `documents_fts`, deactivates missing files, deletes orphaned content

---

## embed

```mermaid
flowchart TD
    A["qmd embed [-f] [-c coll]<br/>[--chunk-strategy auto|regex]"] --> B["getStore()"]
    B --> C{"force?"}
    C -->|yes| D["clearAllEmbeddings()"]
    C -->|no| E["count hashesNeedingEmbedding"]
    E --> F{"pending == 0?"}
    F -->|yes| G["print already embedded + return"]
    F -->|no| H["generateEmbeddings()"]
    H --> I["chunkDocumentByTokens()<br/>regex + optional AST breakpoints"]
    I --> J["probe first chunk →<br/>ensureVecTable(dimensions)"]
    J --> K["embedBatch()<br/>LLM pooled contexts"]
    K --> L["insertEmbedding()<br/>content_vectors + vectors_vec"]
    L --> M["retry failures<br/>remove incomplete"]
    M --> N["print chunk/doc count<br/>+ errors + duration"]
    N --> O["done"]
```

- **What it does:** Generates or refreshes vector embeddings for all pending content hashes
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`, `src/llm.ts`
- **Side effects:** Writes `content_vectors`, creates/updates `vectors_vec` virtual table, progress on stderr

---

## search

```mermaid
flowchart TD
    A["qmd search 'query' [-c coll]<br/>[-n N] [--min-score N]"] --> B["resolveCollectionFilter()<br/>default collections if none"]
    B --> C["searchFTS(db, query)<br/>buildFTS5Query() →<br/>bm25(documents_fts, 1.5, 4.0, 1.0)"]
    C --> D["post-filter by collection<br/>prefix qmd://coll/"]
    D --> E["getContextForFile()<br/>+ docid from hash"]
    E --> F["closeDb()"]
    F --> G["outputResults()<br/>format: cli/json/csv/md/xml/files"]
    G --> H["done"]
```

- **What it does:** Fast lexical BM25 search without LLM; weighted title > filepath > body
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`
- **Side effects:** None (read-only)

---

## vsearch

```mermaid
flowchart TD
    A["qmd vsearch 'query'<br/>[-c coll] [--min-score 0.3]"] --> B["getStore() + checkIndexHealth"]
    B --> C["withLLMSession()"]
    C --> D["vectorSearchQuery()"]
    D --> E["expandQuery()<br/>cached in llm_cache"]
    E --> F["embed query texts"]
    F --> G["searchVec()<br/>sqlite-vec MATCH + k=N<br/>→ dedupe by filepath<br/>score = 1 - distance"]
    G --> H["post-filter collections"]
    H --> I["closeDb()"]
    I --> J["outputResults()"]
    J --> K["done"]
```

- **What it does:** Vector-only similarity search using sqlite-vec cosine distance
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`, `src/llm.ts`
- **Side effects:** May cache query expansion in `llm_cache`

---

## query

```mermaid
flowchart TD
    A["qmd query 'query'<br/>[--no-rerank] [--explain]<br/>[--intent 'hint']"] --> B["parseStructuredQuery()<br/>lex:/vec:/hyde:/intent: ?"]
    B -->|structured| C["structuredSearch()"]
    B -->|plain| D["hybridQuery()"]
    D --> E["BM25 probe<br/>→ strong signal? skip expansion"]
    E -->|weak| F["expandQuery()<br/>→ lex, vec, hyde"]
    F --> G["searchFTS() + searchVec()<br/>multiple query variants"]
    G --> H["RRF fusion<br/>weight 2.0 orig / 1.0 expanded"]
    H --> I{"skipRerank?"}
    I -->|no| J["chunk candidates<br/>pick best by keyword overlap<br/>rerank() with LLM"]
    J --> K["blend RRF + rerank score"]
    I -->|yes| L["use RRF score only"]
    K --> M["dedupe + filter + limit"]
    L --> M
    C --> M
    M --> N["closeDb() + outputResults()<br/>with optional explain traces"]
    N --> O["done"]
```

- **What it does:** Recommended hybrid search with query expansion, RRF fusion, and optional LLM reranking
- **Key files:** `src/cli/qmd.ts`, `src/store.ts`, `src/llm.ts`
- **Side effects:** May write `llm_cache` entries for expansions/reranking

---

## mcp

```mermaid
flowchart TD
    A["qmd mcp"] --> B["getDbPath()"]
    B --> C["startMcpServer()<br/>enableProductionMode()<br/>createStore() + StdioServerTransport"]
    C --> D["register tools:<br/>query, get, multi_get, status"]
    D --> E["register resource:<br/>qmd://{+path}"]
    E --> F["register dynamic<br/>instructions from status"]
    F --> G["stdio loop until<br/>transport closes"]
    G --> H["done"]
```

- **What it does:** Starts MCP server over stdio exposing search and document retrieval tools
- **Key files:** `src/mcp/server.ts`, `src/index.ts`
- **Side effects:** Opens DB + LLM; blocks until transport closes

---

## mcp --http --daemon

```mermaid
flowchart TD
    A["qmd mcp --http --daemon<br/>[--port 8181]"] --> B{"PID file exists +<br/>process alive?"}
    B -->|yes| C["exit 1: already running"]
    B -->|no| D["mkdir cache dir"]
    D --> E["spawn child:<br/>node qmd.ts mcp --http --port N"]
    E --> F["write PID file"]
    F --> G["print URL + log path<br/>exit 0"]
    G --> H["done"]
```

- **What it does:** Detaches an HTTP MCP server as a background daemon
- **Key files:** `src/cli/qmd.ts`, `src/mcp/server.ts`
- **Side effects:** Writes `~/.cache/qmd/mcp.pid` and `mcp.log`, spawns detached child process

---

## mcp stop

```mermaid
flowchart TD
    A["qmd mcp stop"] --> B{"PID file exists?"}
    B -->|no| C["print not running + exit 0"]
    B -->|yes| D["read PID"]
    D --> E{"kill(pid, 0) alive?"}
    E -->|yes| F["kill(pid, SIGTERM)<br/>unlink PID file"]
    F --> G["print stopped"]
    E -->|no| H["unlink stale PID"]
    H --> I["print cleaned up"]
    G --> J["exit 0"]
    I --> J
```

- **What it does:** Terminates the background MCP daemon by PID file
- **Key files:** `src/cli/qmd.ts`
- **Side effects:** Sends SIGTERM, deletes `~/.cache/qmd/mcp.pid`
