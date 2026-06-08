## src/index.ts

### Re-exported SDK Types

`DocumentResult`: document metadata and optional body returned by retrieval/search internals.

`DocumentNotFound`: not-found result with error data and similar-file suggestions.

`SearchResult`: lexical or vector search result returned by low-level search methods.

`HybridQueryResult`: fused/reranked search result returned by `search()` and internal hybrid search.

`HybridQueryOptions`: options accepted by internal `hybridQuery`.

`HybridQueryExplain`: explain trace metadata for hybrid results.

`ExpandedQuery`: typed query item, typically `{ type: "lex" | "vec" | "hyde"; query: string }`.

`StructuredSearchOptions`: options accepted by internal `structuredSearch`.

`MultiGetResult`: result item for multi-document retrieval, including skip state.

`IndexStatus`: document, embedding, vector-index, and collection status summary.

`IndexHealthInfo`: index health details such as stale or missing embeddings.

`SearchHooks`: callbacks for search phases such as expansion, embedding, and reranking.

`ReindexProgress`: low-level progress event emitted while indexing files.

`ReindexResult`: per-collection reindex counts.

`EmbedProgress`: progress event emitted during embedding.

`EmbedResult`: embedding run summary.

`Collection`: single collection configuration.

`CollectionConfig`: top-level config containing collections, models, and context.

`NamedCollection`: collection config plus its persisted name.

`ContextMap`: path-to-context mapping type.

`InternalStore`: alias re-export of the low-level `Store` from `store.ts`.

`ChunkStrategy`: chunking mode used by search/index/embed paths.

### Re-exported SDK Values

`extractSnippet(body: string, query: string, ...): { line: number; snippet: string }`: Extract the best text snippet around a query or chunk position.

`addLineNumbers(text: string, startLine?: number): string`: Prefix markdown/text content with 1-indexed line numbers.

`DEFAULT_MULTI_GET_MAX_BYTES: number`: Default byte cap for multi-document retrieval.

`getDefaultDbPath(): string`: Return QMD's default SQLite database path.

`Maintenance`: Housekeeping class re-exported from `maintenance.ts`.

### Local Types

`UpdateProgress = { collection: string; file: string; current: number; total: number }`: Progress info emitted during `QMDStore.update()`.

`UpdateResult = { collections: number; indexed: number; updated: number; unchanged: number; removed: number; needsEmbedding: number }`: Aggregated `update()` result across selected collections.

`SearchOptions`: `{ query?: string; queries?: ExpandedQuery[]; intent?: string; rerank?: boolean; collection?: string; collections?: string[]; limit?: number; candidateLimit?: number; minScore?: number; explain?: boolean; chunkStrategy?: ChunkStrategy }`. Options for unified full search.

`LexSearchOptions`: `{ limit?: number; collection?: string }`. Options for BM25 keyword search.

`VectorSearchOptions`: `{ limit?: number; collection?: string }`. Options for vector similarity search.

`ExpandQueryOptions`: `{ intent?: string }`. Options for manual query expansion.

`StoreOptions`: `{ dbPath: string; configPath?: string; config?: CollectionConfig }`. Options for opening a SDK store.

`QMDStore`: public async SDK interface. Shape: `{ readonly internal: InternalStore; readonly dbPath: string; search(...); searchLex(...); searchVector(...); expandQuery(...); get(...); getDocumentBody(...); multiGet(...); addCollection(...); removeCollection(...); renameCollection(...); listCollections(...); getDefaultCollectionNames(...); addContext(...); removeContext(...); setGlobalContext(...); getGlobalContext(...); listContexts(...); update(...); embed(...); getStatus(...); getIndexHealth(...); close(...) }`.

### Local Functions

`createStore(options: StoreOptions): Promise<QMDStore>`: Open a SQLite-backed SDK store, optionally synchronize config, attach a per-store `LlamaCpp`, and return the high-level SDK facade.

## src/mcp/server.ts

### Exported Types

`McpStartupOptions = { dbPath?: string }`: Startup options shared by stdio and HTTP MCP servers.

`HttpServerHandle = { httpServer: import("http").Server; port: number; stop: () => Promise<void> }`: Handle returned by the HTTP server for port discovery and shutdown.

### Exported Functions

`startMcpServer(options: McpStartupOptions = {}): Promise<void>`: Start the QMD MCP server over stdio using the default DB path unless overridden.

`startMcpHttpServer(port: number, options: ({ quiet?: boolean } & McpStartupOptions) = {}): Promise<HttpServerHandle>`: Start the QMD MCP server over localhost Streamable HTTP with `/mcp`, `/health`, and REST `/query` or `/search` endpoints.

### Internal Types

`SearchResultItem = { docid: string; file: string; title: string; score: number; context: string | null; line: number; snippet: string }`: Structured result returned by MCP search.

`StatusResult = { totalDocuments: number; needsEmbedding: number; hasVectorIndex: boolean; collections: { name: string; path: string | null; pattern: string | null; documents: number; lastUpdated: string }[] }`: Structured status payload for MCP status.

`JsonRpcLikeBody = { method?: unknown; params?: { name?: unknown; arguments?: Record<string, unknown> } }`: Minimal JSON-RPC shape used for HTTP logging.

`RestSearchInput = { type?: unknown; query?: unknown }`: Minimal REST search input shape.

### Internal Functions

`encodeQmdPath(path: string): string`: Encode a path for `qmd://` URIs while preserving forward slashes.

`formatSearchSummary(results: SearchResultItem[], query: string): string`: Build human-readable MCP text content for search results.

`getPackageVersion(): string`: Read `package.json` beside the built module and return the package version or `"unknown"`.

`buildInstructions(store: QMDStore): Promise<string>`: Build dynamic MCP server instructions from index status and global context.

`createMcpServer(store: QMDStore): Promise<McpServer>`: Create an MCP server and register QMD resource, tools, and dynamic instructions.

`createSession(): Promise<WebStandardStreamableHTTPServerTransport>`: HTTP-only helper that creates one MCP server and Streamable HTTP transport for a new session.

`ts(): string`: HTTP-only helper that formats a timestamp as `HH:mm:ss.SSS`.

`describeRequest(body: JsonRpcLikeBody): string`: HTTP-only helper that creates a short request label for logging.

`log(msg: string): void`: HTTP-only helper that writes to stderr unless quiet mode is enabled.

`collectBody(req: IncomingMessage): Promise<string>`: HTTP-only helper that reads a Node request body into a string.

`stop(): Promise<void>`: HTTP server shutdown helper that closes transports, clears sessions, closes the HTTP server, and closes the shared store.

## src/paths.ts

`qmdHomedir(): string`: Resolve the user home directory from `HOME`, `USERPROFILE`, `os.homedir()`, or `"/tmp"`.

## src/maintenance.ts

`Maintenance`: `class Maintenance { constructor(store: Store); vacuum(): void; cleanupOrphanedContent(): number; cleanupOrphanedVectors(): number; clearLLMCache(): number; deleteInactiveDocs(): number; clearEmbeddings(): void }`. Thin wrapper around database maintenance operations.

`constructor(store: Store)`: Store the internal QMD store used for direct DB maintenance.

`vacuum(): void`: Run SQLite `VACUUM` to reclaim disk space.

`cleanupOrphanedContent(): number`: Delete unreferenced content rows.

`cleanupOrphanedVectors(): number`: Delete vector embeddings for missing content.

`clearLLMCache(): number`: Delete cached LLM expansion/rerank responses.

`deleteInactiveDocs(): number`: Delete documents marked inactive.

`clearEmbeddings(): void`: Delete all vector embeddings.
