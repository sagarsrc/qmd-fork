## SDK Overview

`src/index.ts` is the public programmatic SDK facade. Its main goal is to let consumers open a QMD SQLite index, optionally seed or synchronize collection config, perform search/retrieval/indexing operations, mutate collections and contexts, and then close all resources cleanly.

`StoreOptions` requires `dbPath` and accepts either `configPath` or inline `config`, but not both. If neither config source is provided, the SDK opens the DB in DB-only mode and reads existing `store_collections` state.

`QMDStore` exposes the underlying `internal` store for advanced consumers plus high-level async methods for search, document retrieval, collection management, context management, indexing, health, and lifecycle. The SDK creates one `InternalStore` with `createStoreInternal(dbPath)` and one per-store `LlamaCpp` instance. That LLM instance is configured from optional `config.models`, lazy-loads models, auto-unloads after five minutes of inactivity, and is attached to `internal.llm`.

## SDK Search API

`search(options)` is the unified high-level search method. It requires either `query` or `queries`. It normalizes `collection` plus `collections` into a collection array and maps `rerank: false` to `skipRerank`.

If callers pass pre-expanded `queries`, `search()` calls `structuredSearch(internal, queries, ...)`. That path executes caller-provided typed searches (`lex`, `vec`, `hyde`) and can filter across multiple collections.

If callers pass a simple `query`, `search()` calls `hybridQuery(internal, query, ...)`. That path performs initial FTS probing, optional query expansion, multi-signal retrieval, reciprocal-rank-style fusion, optional reranking, score filtering, and result limiting. This path only forwards the first normalized collection as `collection`.

`searchLex(query, options)` maps directly to `internal.searchFTS(query, limit, collection)`. It is fast BM25 keyword search with no LLM use.

`searchVector(query, options)` maps to `internal.searchVec(query, llm.embedModelName, limit, collection)`. It uses the store LLM embedding model name and vector index, but does not rerank.

`expandQuery(query, options)` maps to `internal.expandQuery(query, undefined, intent)`. Internally the store uses the attached per-store LLM when present and caches expansion results in SQLite.

## SDK Collection Management

Collection mutations write to SQLite first and then write through to YAML or inline config if the SDK was opened with `configPath` or `config`.

`addCollection(name, opts)` calls `upsertStoreCollection(db, name, ...)` and then `collectionsAddCollection(...)` when a config source exists. SQLite stores path, pattern, ignore, and default-include metadata.

`removeCollection(name)` calls `deleteStoreCollection(db, name)`, then `collectionsRemoveCollection(name)` when configured, and returns the SQLite deletion result.

`renameCollection(oldName, newName)` calls `renameStoreCollection(db, oldName, newName)`, then `collectionsRenameCollection(...)` when configured, and returns whether SQLite changed a row.

`listCollections()` calls `storeListCollections(db)` and returns collection metadata plus document stats. `getDefaultCollectionNames()` filters that same list to collections with `includeByDefault`.

## SDK Context Management

Context mutations use the same SQLite-first, optional config write-through pattern.

`addContext(collectionName, pathPrefix, contextText)` calls `updateStoreContext(db, ...)`, then `collectionsAddContext(...)` if a config source exists.

`removeContext(collectionName, pathPrefix)` calls `removeStoreContext(db, ...)`, then `collectionsRemoveContext(...)` if a config source exists.

`setGlobalContext(context)` calls `setStoreGlobalContext(db, context)`, then `collectionsSetGlobalContext(context)` if a config source exists.

`getGlobalContext()` reads `getStoreGlobalContext(db)`. `listContexts()` reads `getStoreContexts(db)`.

## SDK Indexing

`update(options)` reads collections from SQLite with `getStoreCollections(db)`, optionally filters to `options.collections`, clears the internal search/cache state, and reindexes each selected collection with `reindexCollection(internal, path, pattern || "**/*.md", name, ...)`.

Progress callbacks are adapted from per-file reindex progress into SDK `UpdateProgress` by adding `collection: col.name`. The final `UpdateResult` aggregates `indexed`, `updated`, `unchanged`, and `removed` across collections, reports the number of collections processed, and computes `needsEmbedding` with `internal.getHashesNeedingEmbedding()`.

`embed(options)` forwards to `generateEmbeddings(internal, ...)`. Options include `force`, model override, single-collection restriction, batch limits, `chunkStrategy`, and progress callback. Collection filtering is handled by the internal embedding generator.

## SDK Lifecycle

`close()` disposes the per-store `LlamaCpp` instance, closes the internal SQLite store, and resets the process-level config source with `setConfigSource(undefined)` when the SDK had injected a YAML or inline config source. This prevents a configured SDK instance from leaking config state into later callers.

## MCP Overview

`src/mcp/server.ts` exposes QMD through MCP. `createMcpServer(store)` creates an `McpServer` named `qmd`, injects dynamic instructions from `buildInstructions(store)`, pre-fetches default collection names, and registers one document resource plus `query`, `get`, `multi_get`, and `status` tools.

`startMcpServer()` starts the default stdio transport. It enables production mode, resolves the config path with `getConfigPath()`, opens a SDK store using `options.dbPath ?? getDefaultDbPath()`, conditionally attaches `configPath`, creates the MCP server, and connects it to `StdioServerTransport`.

`startMcpHttpServer(port, options)` starts a localhost Streamable HTTP server. It shares one SDK store across sessions, but creates one `McpServer` and one `WebStandardStreamableHTTPServerTransport` per MCP session. It also provides `/health` and REST `/query` or `/search` endpoints outside the MCP protocol.

Tool, resource, and prompt-like instruction registration is centralized in `createMcpServer()`. The dynamic instructions are passed in the MCP server constructor rather than registered as a separate callable prompt.

## MCP Tools

`query` is the primary search tool. Parameters are `searches`, `limit`, `minScore`, `candidateLimit`, `collections`, `intent`, and `rerank`. `searches` is an array of typed sub-queries with `type: "lex" | "vec" | "hyde"` and `query: string`. The handler maps these to `ExpandedQuery[]`, uses default collections when none are supplied, calls `store.search({ queries, ... })`, extracts snippets with `extractSnippet`, adds line numbers, and returns both text summary content and structured results. Errors from validation or store search propagate through MCP handler error handling.

`get` retrieves one document by file path or docid. Parameters are `file`, `fromLine`, `maxLines`, and `lineNumbers`. The `file` parameter may include `:line` or `:line:count` suffixes unless explicit arguments override them. Missing documents return `isError: true` with similar-file suggestions. Successful results return a `resource` content item with a `qmd://` URI and markdown text, optionally line-numbered and prefixed with HTML context metadata.

`multi_get` retrieves multiple documents by glob or comma-separated list. Parameters are `pattern`, `maxLines`, `maxBytes`, and `lineNumbers`. It calls `store.multiGet(pattern, { includeBody: true, maxBytes })`. No matches with no errors returns `isError: true`. Individual oversized/skipped docs become text entries; retrieved docs become resource entries.

`status` has no parameters. It calls `store.getStatus()` and returns a human-readable summary plus structured status containing document totals, embedding state, vector-index availability, and collection stats.

## MCP Resources

The server registers one read-only resource template: `qmd://{+path}` named `document`. It has no `list()` implementation by design, so clients discover documents through search rather than enumerating the whole index.

Resource reads URL-decode the path, call `store.get(decodedPath, { includeBody: true })`, and return line-numbered markdown. Missing documents return a text body saying the document was not found rather than throwing.

## MCP Instructions

`buildInstructions(store)` constructs the MCP server instructions dynamically from `store.getStatus()` and `store.getGlobalContext()`. It includes total searchable markdown documents, optional global context, collection names, vector-index or stale-embedding notices, guidance for `lex`, `vec`, and `hyde` searches, retrieval workflow hints, and practical tips.

The design gives an MCP client immediate system-prompt context about what is searchable and whether semantic search is available without requiring a status call first.

## Paths

`qmdHomedir()` resolves QMD's home directory in a cross-platform way: `process.env.HOME`, then `process.env.USERPROFILE`, then Node's `os.homedir()`, then `"/tmp"` as a final fallback.

## Maintenance

`src/maintenance.ts` defines `Maintenance`, a thin wrapper around low-level store housekeeping functions. It takes an internal `Store` in the constructor and accesses `store.db` directly.

`vacuum()` runs SQLite `VACUUM` through `vacuumDatabase`.

`cleanupOrphanedContent()` deletes content rows no longer referenced by documents and returns the number removed.

`cleanupOrphanedVectors()` deletes vector rows for content hashes that no longer exist and returns the number removed.

`clearLLMCache()` deletes cached LLM responses for query expansion and reranking and returns the number removed.

`deleteInactiveDocs()` deletes documents marked inactive after disappearing from the filesystem and returns the number removed.

`clearEmbeddings()` deletes all embeddings, forcing future re-embedding.

## Gotchas / Design Decisions

The SDK uses its own `LlamaCpp` per store to avoid global singleton state, allow model names from each store config, and make lifecycle ownership explicit. Closing a store disposes that store's models.

MCP instructions are dynamic because index state matters to tool use. Total docs, collection names, global context, and embedding availability change over time, and exposing that at initialization helps clients choose the right search strategy.

Collection and context mutations follow a write-through pattern: SQLite is the operational source used by indexing/search, while YAML or inline config is updated when the store was created with a config source. DB-only mode intentionally skips external config writes.

`search({ query })` and `search({ queries })` are not identical. The simple-query path uses `hybridQuery` and currently forwards only one collection, while the structured-query path uses `structuredSearch` and supports multiple collections.

The HTTP transport shares one SDK store across MCP sessions but creates separate MCP server/transport pairs per session to satisfy MCP session requirements.
