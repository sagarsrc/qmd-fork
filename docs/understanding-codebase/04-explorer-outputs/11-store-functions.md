# Exported Function Inventory

This inventory covers exported functions from `src/store.ts`. Exported constants and types are not listed unless they are directly involved in an exported function signature.

| Function | Signature | Purpose | Subsystem |
|---|---|---|---|
| `getEmbeddingFingerprint` | `(model: string = DEFAULT_EMBED_MODEL) => string` | Builds a 6-char fingerprint for embedding model, formatting, and chunk config. | Embeddings |
| `scanBreakPoints` | `(text: string) => BreakPoint[]` | Finds scored markdown break positions with best score per char offset. | Chunking |
| `findCodeFences` | `(text: string) => CodeFenceRegion[]` | Detects fenced code regions so chunks do not split inside them. | Chunking |
| `isInsideCodeFence` | `(pos: number, fences: CodeFenceRegion[]) => boolean` | Tests whether a candidate cut lies inside a code fence. | Chunking |
| `findBestCutoff` | `(breakPoints: BreakPoint[], targetCharPos: number, windowChars = CHUNK_WINDOW_CHARS, decayFactor = 0.7, codeFences: CodeFenceRegion[] = []) => number` | Chooses the highest scoring cutoff near a target using squared distance decay. | Chunking |
| `mergeBreakPoints` | `(a: BreakPoint[], b: BreakPoint[]) => BreakPoint[]` | Combines regex and AST breakpoints, keeping highest score per position. | Chunking |
| `chunkDocumentWithBreakPoints` | `(content: string, breakPoints: BreakPoint[], codeFences: CodeFenceRegion[], maxChars = CHUNK_SIZE_CHARS, overlapChars = CHUNK_OVERLAP_CHARS, windowChars = CHUNK_WINDOW_CHARS) => { text: string; pos: number }[]` | Shared character-space chunking implementation. | Chunking |
| `homedir` | `() => string` | Returns QMD home directory. | Paths |
| `isAbsolutePath` | `(path: string) => boolean` | Recognizes Unix, Windows, and Git Bash absolute paths. | Paths |
| `normalizePathSeparators` | `(path: string) => string` | Converts backslashes to forward slashes. | Paths |
| `getRelativePathFromPrefix` | `(path: string, prefix: string) => string \| null` | Returns path relative to prefix, or null if not under prefix. | Paths |
| `resolve` | `(...paths: string[]) => string` | Cross-platform path resolver with slash normalization and Windows drive handling. | Paths |
| `enableProductionMode` | `() => void` | Allows default DB path resolution for production CLI startup. | Store lifecycle |
| `_resetProductionModeForTesting` | `() => void` | Resets production flag for tests. | Store lifecycle |
| `getDefaultDbPath` | `(indexName: string = "index") => string` | Resolves default SQLite path, respecting `INDEX_PATH`. | Store lifecycle |
| `getPwd` | `() => string` | Returns `PWD` or `process.cwd()`. | Paths |
| `getRealPath` | `(path: string) => string` | Returns realpath, falling back to resolved path if missing. | Paths |
| `normalizeVirtualPath` | `(input: string) => string` | Normalizes explicit `qmd:` or `//` virtual path forms. | Virtual paths |
| `parseVirtualPath` | `(virtualPath: string) => VirtualPath \| null` | Parses `qmd://collection/path?index=name`. | Virtual paths |
| `buildVirtualPath` | `(collectionName: string, path: string, indexName?: string) => string` | Builds a `qmd://` path. | Virtual paths |
| `isVirtualPath` | `(path: string) => boolean` | Checks whether a path is explicitly virtual. | Virtual paths |
| `resolveVirtualPath` | `(db: Database, virtualPath: string) => string \| null` | Resolves virtual path to filesystem path via collection root. | Virtual paths |
| `toVirtualPath` | `(db: Database, absolutePath: string) => string \| null` | Converts an indexed absolute path to `qmd://`. | Virtual paths |
| `verifySqliteVecLoaded` | `(db: Database) => void` | Probes sqlite-vec availability with `vec_version()`. | Database/vector |
| `normalizeCjkForFTS` | `(text: string) => string` | Spaces CJK runs for FTS5 unicode61 indexing/search. | BM25/FTS |
| `getStoreCollections` | `(db: Database) => NamedCollection[]` | Reads all DB-backed collection configs. | Collections |
| `getStoreCollection` | `(db: Database, name: string) => NamedCollection \| null` | Reads one collection config. | Collections |
| `getStoreGlobalContext` | `(db: Database) => string \| undefined` | Reads global context from `store_config`. | Context |
| `getStoreContexts` | `(db: Database) => Array<{ collection: string; path: string; context: string }>` | Lists global and collection path contexts. | Context |
| `upsertStoreCollection` | `(db: Database, name: string, collection: Omit<Collection, "pattern"> & { pattern?: string }) => void` | Inserts or updates `store_collections`. | Collections |
| `deleteStoreCollection` | `(db: Database, name: string) => boolean` | Deletes a collection config row. | Collections |
| `renameStoreCollection` | `(db: Database, oldName: string, newName: string) => boolean` | Renames a collection config, rejecting target collisions. | Collections |
| `updateStoreContext` | `(db: Database, collectionName: string, path: string, text: string) => boolean` | Adds/updates path context JSON for a collection. | Context |
| `removeStoreContext` | `(db: Database, collectionName: string, path: string) => boolean` | Removes a path context from a collection. | Context |
| `setStoreGlobalContext` | `(db: Database, value: string \| undefined) => void` | Sets or deletes global context. | Context |
| `syncConfigToDb` | `(db: Database, config: CollectionConfig) => void` | Syncs external collection config into SQLite using a config hash. | Collections |
| `isSqliteVecAvailable` | `() => boolean` | Reports whether sqlite-vec loaded at initialization. | Database/vector |
| `reindexCollection` | `(store: Store, collectionPath: string, globPattern: string, collectionName: string, options?: { ignorePatterns?: string[]; onProgress?: (info: ReindexProgress) => void }) => Promise<ReindexResult>` | Scans files and updates content/documents/FTS lifecycle. | Indexing |
| `generateEmbeddings` | `(store: Store, options?: EmbedOptions) => Promise<EmbedResult>` | Chunks pending content and stores embeddings with retry handling. | Embeddings |
| `createStore` | `(dbPath?: string) => Store` | Opens and initializes DB, returns bound Store facade. | Store lifecycle |
| `getDocid` | `(hash: string) => string` | Returns first 6 characters of content hash. | Docids |
| `handelize` | `(path: string) => string` | Converts path text to token-friendly slug form for display/legacy matching. | Paths/documents |
| `getHashesNeedingEmbedding` | `(db: Database, collection?: string, model: string = DEFAULT_EMBED_MODEL) => number` | Counts active hashes missing complete current-fingerprint embeddings. | Index health |
| `maybeAdoptLegacyEmbeddingFingerprint` | `(store: Store, model: string = DEFAULT_EMBED_MODEL) => Promise<LegacyFingerprintAdoptionResult>` | Verifies and adopts empty-fingerprint legacy embeddings. | Embeddings |
| `getIndexHealth` | `(db: Database, model: string = DEFAULT_EMBED_MODEL) => IndexHealthInfo` | Reports pending embeddings, active docs, and staleness days. | Index health |
| `getCacheKey` | `(url: string, body: object) => string` | Hashes cache namespace and request body. | LLM cache |
| `getCachedResult` | `(db: Database, cacheKey: string) => string \| null` | Reads cached LLM result. | LLM cache |
| `setCachedResult` | `(db: Database, cacheKey: string, result: string) => void` | Writes cached LLM result and occasionally trims to 1000 newest. | LLM cache |
| `clearCache` | `(db: Database) => void` | Deletes all LLM cache rows. | LLM cache |
| `deleteLLMCache` | `(db: Database) => number` | Deletes cache and returns changed row count. | Maintenance |
| `deleteInactiveDocuments` | `(db: Database) => number` | Hard-deletes inactive document tombstones. | Maintenance |
| `cleanupOrphanedContent` | `(db: Database) => number` | Deletes content hashes unused by any document row. | Maintenance |
| `cleanupOrphanedVectors` | `(db: Database) => number` | Deletes vector chunks not referenced by active documents. | Maintenance/vector |
| `vacuumDatabase` | `(db: Database) => void` | Runs SQLite `VACUUM`. | Maintenance |
| `hashContent` | `(content: string) => Promise<string>` | Computes SHA-256 hex hash for content. | Indexing |
| `extractTitle` | `(content: string, filename: string) => string` | Extracts markdown/org title, falling back to filename. | Indexing |
| `insertContent` | `(db: Database, hash: string, content: string, createdAt: string) => void` | Inserts content row if hash is new. | Indexing |
| `insertDocument` | `(db: Database, collectionName: string, path: string, title: string, hash: string, createdAt: string, modifiedAt: string) => void` | Upserts active document row and rebuilds FTS. | Indexing |
| `findActiveDocument` | `(db: Database, collectionName: string, path: string) => { id: number; hash: string; title: string } \| null` | Finds active document by collection/path. | Indexing |
| `findOrMigrateLegacyDocument` | `(db: Database, collectionName: string, path: string) => { id: number; hash: string; title: string } \| null` | Finds active doc or migrates legacy case/slug path to literal path. | Indexing |
| `updateDocumentTitle` | `(db: Database, documentId: number, title: string, modifiedAt: string) => void` | Updates title/mtime and rebuilds FTS. | Indexing |
| `updateDocument` | `(db: Database, documentId: number, title: string, hash: string, modifiedAt: string) => void` | Updates hash/title/mtime and rebuilds FTS. | Indexing |
| `deactivateDocument` | `(db: Database, collectionName: string, path: string) => void` | Soft-deactivates a missing document. | Indexing |
| `getActiveDocumentPaths` | `(db: Database, collectionName: string) => string[]` | Lists active paths for a collection. | Indexing |
| `chunkDocument` | `(content: string, maxChars = CHUNK_SIZE_CHARS, overlapChars = CHUNK_OVERLAP_CHARS, windowChars = CHUNK_WINDOW_CHARS) => { text: string; pos: number }[]` | Synchronous regex-only document chunking. | Chunking |
| `chunkDocumentAsync` | `(content: string, maxChars = CHUNK_SIZE_CHARS, overlapChars = CHUNK_OVERLAP_CHARS, windowChars = CHUNK_WINDOW_CHARS, filepath?: string, chunkStrategy: ChunkStrategy = "regex") => Promise<{ text: string; pos: number }[]>` | Async chunking with optional AST breakpoints. | Chunking |
| `chunkDocumentByTokens` | `(content: string, maxTokens = CHUNK_SIZE_TOKENS, overlapTokens = CHUNK_OVERLAP_TOKENS, windowTokens = CHUNK_WINDOW_TOKENS, filepath?: string, chunkStrategy: ChunkStrategy = "regex", signal?: AbortSignal) => Promise<{ text: string; pos: number; tokens: number }[]>` | Token-limited chunking using LLM tokenizer and recursive splitting. | Chunking/embeddings |
| `normalizeDocid` | `(docid: string) => string` | Strips quotes and leading `#` from docid input. | Docids |
| `isDocid` | `(input: string) => boolean` | Checks if input is a valid 6+ char hex docid reference. | Docids |
| `findDocumentByDocid` | `(db: Database, docid: string) => { filepath: string; hash: string } \| null` | Finds first active document whose hash starts with the docid. | Docids |
| `findSimilarFiles` | `(db: Database, query: string, maxDistance: number = 3, limit: number = 5) => string[]` | Levenshtein fuzzy path suggestions. | Retrieval |
| `matchFilesByGlob` | `(db: Database, pattern: string) => { filepath: string; displayPath: string; bodyLength: number }[]` | Matches active virtual/display paths with picomatch. | Retrieval |
| `getContextForPath` | `(db: Database, collectionName: string, path: string) => string \| null` | Applies global and hierarchical collection path contexts. | Context |
| `getContextForFile` | `(db: Database, filepath: string) => string \| null` | Resolves virtual/absolute path and returns inherited context. | Context |
| `getCollectionByName` | `(db: Database, name: string) => { name: string; pwd: string; glob_pattern: string } \| null` | Compatibility wrapper around `store_collections`. | Collections |
| `listCollections` | `(db: Database) => { name: string; pwd: string; glob_pattern: string; doc_count: number; active_count: number; last_modified: string \| null; includeByDefault: boolean }[]` | Lists collection configs plus document stats. | Collections/status |
| `removeCollection` | `(db: Database, collectionName: string) => { deletedDocs: number; cleanedHashes: number }` | Deletes collection docs/config and orphaned active content. | Collections |
| `renameCollection` | `(db: Database, oldName: string, newName: string) => void` | Renames documents and collection config. | Collections |
| `insertContext` | `(db: Database, collectionId: number, pathPrefix: string, context: string) => void` | Legacy-id context insert wrapper; currently depends on dropped `collections` table. | Context/legacy |
| `deleteContext` | `(db: Database, collectionName: string, pathPrefix: string) => number` | Removes one collection path context. | Context |
| `deleteGlobalContexts` | `(db: Database) => number` | Deletes global context and root collection contexts. | Context |
| `listPathContexts` | `(db: Database) => { collection_name: string; path_prefix: string; context: string }[]` | Lists contexts sorted by collection and specificity. | Context |
| `getAllCollections` | `(db: Database) => { name: string }[]` | Lists collection names. | Collections |
| `getCollectionsWithoutContext` | `(db: Database) => { name: string; pwd: string; doc_count: number }[]` | Finds collections without context entries. | Context |
| `getTopLevelPathsWithoutContext` | `(db: Database, collectionName: string) => string[]` | Finds top-level directories without context coverage. | Context |
| `sanitizeFTS5Term` | `(term: string) => string` | Sanitizes one FTS token. | BM25/FTS |
| `validateSemanticQuery` | `(query: string) => string \| null` | Rejects lex-only negation syntax in vec/hyde queries. | Search validation |
| `validateLexQuery` | `(query: string) => string \| null` | Rejects multiline or unmatched-quote lex queries. | Search validation |
| `searchFTS` | `(db: Database, query: string, limit: number = 20, collectionName?: string) => SearchResult[]` | Runs BM25 FTS search and returns document results. | BM25/FTS |
| `searchVec` | `(db: Database, query: string, model: string, limit: number = 20, collectionName?: string, session?: ILLMSession, precomputedEmbedding?: number[]) => Promise<SearchResult[]>` | Runs sqlite-vec cosine search and maps chunks to documents. | Vector search |
| `getHashesForEmbedding` | `(db: Database, model: string = DEFAULT_EMBED_MODEL) => { hash: string; body: string; path: string }[]` | Lists active content hashes needing embeddings. | Embeddings |
| `clearAllEmbeddings` | `(db: Database, collection?: string) => void` | Clears all or collection-exclusive embeddings. | Embeddings |
| `insertEmbedding` | `(db: Database, hash: string, seq: number, pos: number, embedding: Float32Array, model: string, embeddedAt: string, totalChunks?: number, fingerprint?: string) => void` | Stores vector metadata and sqlite-vec row. | Embeddings |
| `expandQuery` | `(query: string, model: string = DEFAULT_QUERY_MODEL, db: Database, intent?: string, llmOverride?: LlamaCpp) => Promise<ExpandedQuery[]>` | Cached LLM expansion to typed lex/vec/hyde query variants. | Hybrid search |
| `rerank` | `(query: string, documents: { file: string; text: string }[], model: string = DEFAULT_RERANK_MODEL, db: Database, intent?: string, llmOverride?: LlamaCpp) => Promise<{ file: string; score: number }[]>` | Cached LLM reranking of selected chunks. | Hybrid search |
| `reciprocalRankFusion` | `(resultLists: RankedResult[][], weights: number[] = [], k: number = 60) => RankedResult[]` | Fuses ranked lists using weighted RRF plus top-rank bonus. | Hybrid search |
| `buildRrfTrace` | `(resultLists: RankedResult[][], weights: number[] = [], listMeta: RankedListMeta[] = [], k: number = 60) => Map<string, RRFScoreTrace>` | Builds explain traces for RRF contributions. | Hybrid search/explain |
| `findDocument` | `(db: Database, filename: string, options: { includeBody?: boolean } = {}) => DocumentResult \| DocumentNotFound` | Resolves virtual/absolute/relative/docid input to document metadata. | Retrieval |
| `getDocumentBody` | `(db: Database, doc: DocumentResult \| { filepath: string }, fromLine?: number, maxLines?: number) => string \| null` | Loads full or line-sliced document body. | Retrieval |
| `findDocuments` | `(db: Database, pattern: string, options: { includeBody?: boolean; maxBytes?: number } = {}) => { docs: MultiGetResult[]; errors: string[] }` | Multi-get by glob or comma-separated names with size skipping. | Retrieval |
| `getStatus` | `(db: Database, model: string = DEFAULT_EMBED_MODEL) => IndexStatus` | Reports document counts, embedding backlog, vector presence, and collections. | Status |
| `extractIntentTerms` | `(intent: string) => string[]` | Tokenizes meaningful non-stopword intent terms. | Search/snippets |
| `extractSnippet` | `(body: string, query: string, maxLen = 500, chunkPos?: number, chunkLen?: number, intent?: string) => SnippetResult` | Selects a line-window snippet with query/intent scoring. | Snippets |
| `addLineNumbers` | `(text: string, startLine: number = 1) => string` | Prefixes each line with a line number. | Retrieval helpers |
| `getHybridRrfWeights` | `(rankedListMeta: RankedListMeta[]) => number[]` | Assigns 2x weight to original-query lists, 1x to expansions. | Hybrid search |
| `hybridQuery` | `(store: Store, query: string, options?: HybridQueryOptions) => Promise<HybridQueryResult[]>` | Full BM25/vector/expansion/RRF/rerank search pipeline. | Hybrid search |
| `vectorSearchQuery` | `(store: Store, query: string, options?: VectorSearchOptions) => Promise<VectorSearchResult[]>` | Vector-only semantic search with vec/hyde expansion. | Vector search |
| `structuredSearch` | `(store: Store, searches: ExpandedQuery[], options?: StructuredSearchOptions) => Promise<HybridQueryResult[]>` | Executes caller-supplied typed searches through fusion/rerank pipeline. | Hybrid search |

## Re-exported LLM Formatting Functions

| Function | Signature source | Purpose | Subsystem |
|---|---|---|---|
| `formatQueryForEmbedding` | imported from `./llm.js` | Applies query-side embedding formatting. | Embeddings/search |
| `formatDocForEmbedding` | imported from `./llm.js` | Applies document-side embedding formatting. | Embeddings/search |
