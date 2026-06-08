# CLI Function Inventory

## `src/cli/qmd.ts`

| Function | Signature | Purpose | Used by command(s) |
|---|---|---|---|
| `getStore` | `(): ReturnType<typeof createStore>` | Lazily create/sync the store and configure default LLM models. | All DB/store commands |
| `getDb` | `(): Database` | Return the active store DB handle. | Most commands |
| `resyncConfig` | `(): void` | Force YAML config back into SQLite after config mutations. | `collection`, `context`, `init` |
| `closeDb` | `(): void` | Close active store and reset singleton. | Most commands |
| `getDbPath` | `(): string` | Resolve current DB path for status/help/MCP/bench. | `status`, `mcp`, `bench`, help |
| `getActiveIndexName` | `(): string` | Return selected index name. | search output path rendering |
| `setIndexName` | `(name: string | null): void` | Normalize and select a named index DB path. | global `--index`, virtual-path get |
| `ensureVecTable` | `(_db: Database, dimensions: number): void` | Ensure vector table through active store. | Compatibility/internal |
| `flushWritable` | `(stream: CliLifecycleWritable): Promise<void>` | Flush a writable stream via empty write callback. | lifecycle |
| `finishSuccessfulCliCommand` | `(options: FinishSuccessfulCliCommandOptions): Promise<void>` | Flush output, cleanup llama.cpp, set successful exit code without `process.exit(0)`. | all non-MCP successful commands |
| `formatETA` | `(seconds: number): string` | Human-readable duration/ETA. | `update`, `embed`, indexing |
| `checkIndexHealth` | `(db: Database, model?: string): void` | Print pending embedding/stale index tips. | `query`, `vsearch` |
| `computeDisplayPath` | `(filepath: string, collectionPath: string, existingPaths: Set<string>): string` | Build shortest unique display path. | Legacy/helper, no direct dispatch use in read scope |
| `formatTimeAgo` | `(date: Date): string` | Format relative age. | `status`, `collection list` |
| `formatMs` | `(ms: number): string` | Format milliseconds/seconds. | `query` progress hooks |
| `formatBytes` | `(bytes: number): string` | Format bytes as B/KB/MB/GB. | `status`, `ls`, `embed`, `pull` |
| `sameDirectory` | `(a: string, b: string): boolean` | Compare directories by realpath with fallback. | `init` |
| `initLocalIndex` | `(): void` | Create project-local `.qmd` config and SQLite index. | `init` |
| `isForceCpuEnabled` | `(): boolean` | Detect CPU-forced mode from env. | `doctor` |
| `configuredGpuModeLabel` | `(): string` | Render configured GPU mode. | `doctor` |
| `summarizeDeviceNames` | `(names: string[]): string` | Collapse repeated device names. | `doctor` |
| `sanitizeDiagnosticMessage` | `(message: string): string` | Trim/redact diagnostic errors. | `doctor` |
| `showStatus` | `(): Promise<void>` | Print status, health, collections, contexts, models, tips. | `status` |
| `updateCollections` | `(): Promise<void>` | Run collection update commands and reindex all collections. | `update` |
| `detectCollectionFromPath` | `(db: Database, fsPath: string): { collectionName: string; relativePath: string } | null` | Map filesystem path to containing YAML collection by longest prefix. | `context`, `get` |
| `contextAdd` | `(pathArg: string | undefined, contextText: string): Promise<void>` | Add global, virtual, or filesystem-derived context. | `context add` |
| `contextList` | `(): void` | Print all configured contexts grouped by collection. | `context list` |
| `contextRemove` | `(pathArg: string): void` | Remove global, virtual, or filesystem-derived context. | `context rm/remove` |
| `renderFullPath` | `(absolutePath: string, cwd?: string): string` | Render realpath as `./relative` under cwd or absolute otherwise. | `get`, `multi-get`, `search`, `query`, `vsearch` |
| `getDocument` | `(filename: string, fromLine?: number, maxLines?: number, lineNumbers?: boolean, fullPath?: boolean): void` | Resolve and print one document. | `get` |
| `multiGet` | `(pattern: string, maxLines?: number, maxBytes?: number, format?: OutputFormat, lineNumbers?: boolean, fullPath?: boolean): void` | Resolve and print multiple documents. | `multi-get` |
| `listFiles` | `(pathArg?: string): void` | List collections or indexed files under a virtual path. | `ls` |
| `formatLsTime` | `(date: Date): string` | Format timestamp like `ls -l`. | `ls` |
| `collectionList` | `(): void` | Print collections and metadata. | `collection list` |
| `collectionAdd` | `(pwd: string, globPattern: string, name?: string): Promise<void>` | Add YAML collection and index it. | `collection add` |
| `collectionRemove` | `(name: string): void` | Remove collection from DB and YAML. | `collection remove/rm` |
| `collectionRename` | `(oldName: string, newName: string): void` | Rename collection in DB and YAML. | `collection rename/mv` |
| `indexFiles` | `(pwd?: string, globPattern?: string, collectionName?: string, suppressEmbedNotice?: boolean, ignorePatterns?: string[]): Promise<void>` | Glob, hash, insert/update/deactivate docs, and cleanup content. | `collection add` |
| `renderProgressBar` | `(percent: number, width?: number): string` | Create text progress bar. | `embed` |
| `parseEmbedBatchOption` | `(name: string, value: unknown): number | undefined` | Validate positive integer embedding options. | `embed` |
| `parseChunkStrategy` | `(value: unknown): ChunkStrategy | undefined` | Validate `auto`/`regex`. | `parseCLI`, `embed` |
| `ensureModelsConfiguredForCli` | `(): { embed: string; generate: string; rerank: string }` | Resolve models and persist defaults into config if missing. | store init, model helpers |
| `resolveEmbedModelForCli` | `(): string` | Return active embedding model. | `embed`, `status`, health |
| `resolveGenerateModelForCli` | `(): string` | Return active generation model. | exported/helper |
| `resolveRerankModelForCli` | `(): string` | Return active rerank model. | exported/helper |
| `resolveModelsForCli` | `(): { embed: string; generate: string; rerank: string }` | Return all active models. | `status`, `pull` |
| `vectorIndex` | `(model?: string, force?: boolean, batchOptions?: {...}): Promise<void>` | Generate embeddings with progress and failure reporting. | `embed` |
| `sanitizeFTS5Term` | `(term: string): string` | Strip punctuation for FTS query construction. | `buildFTS5Query` |
| `buildFTS5Query` | `(query: string): string` | Build phrase/proximity/term FTS5 expression. | Internal/legacy helper |
| `normalizeBM25` | `(score: number): number` | Convert SQLite BM25 negative score to 0-1. | Internal/legacy helper |
| `highlightTerms` | `(text: string, query: string): string` | ANSI-highlight query terms in CLI snippets. | `search`, `query`, `vsearch` via `outputResults` |
| `formatScore` | `(score: number): string` | Percent score with color thresholds. | CLI search output |
| `formatExplainNumber` | `(value: number): string` | Fixed-width explain metric formatting. | `query --explain` |
| `shortPath` | `(dirpath: string): string` | Replace home prefix with `~`. | Doctor/context helpers |
| `printEmptySearchResults` | `(format: OutputFormat, reason?: EmptySearchReason): void` | Emit format-safe empty search output. | `search`, `query`, `vsearch` |
| `encodePathForEditorUri` | `(absolutePath: string): string` | URI-encode filesystem paths for editor links. | `buildEditorUri` |
| `getEditorUriTemplate` | `(): string` | Resolve editor URI template from env/config/default. | CLI search output |
| `buildEditorUri` | `(template: string, absolutePath: string, line: number, col: number): string` | Fill editor URI placeholders. | CLI search output, exported tests |
| `termLink` | `(text: string, url: string, isTTY?: boolean): string` | Wrap text in OSC 8 hyperlink when TTY. | CLI search output, exported tests |
| `outputResults` | `(results: OutputRow[], query: string, opts: OutputOptions): void` | Central renderer for search-like results in all output formats. | `search`, `query`, `vsearch` |
| `resolveCollectionFilter` | `(raw: string | string[] | undefined, useDefaults?: boolean): string[]` | Validate collection filters and optionally use defaults. | `search`, `query`, `vsearch`, `embed` |
| `filterByCollections` | `<T extends { filepath?: string; file?: string }>(results: T[], collectionNames: string[]): T[]` | Post-filter by `qmd://collection/` prefixes. | `search` |
| `parseStructuredQuery` | `(query: string): ParsedStructuredQuery | null` | Parse `lex:`/`vec:`/`hyde:`/`intent:` query documents. | `query` |
| `search` | `(query: string, opts: OutputOptions): void` | Run lexical FTS search and render results. | `search` |
| `logExpansionTree` | `(originalQuery: string, expanded: ExpandedQuery[]): void` | Print query expansion tree to stderr. | `query`, `vsearch` |
| `vectorSearch` | `(query: string, opts: OutputOptions, _model?: string): Promise<void>` | Run vector-only search and render results. | `vsearch`, `vector-search` |
| `querySearch` | `(query: string, opts: OutputOptions, _embedModel?: string, _rerankModel?: string): Promise<void>` | Run hybrid or structured search and render results. | `query`, `deep-search` |
| `parseCLI` | `(): { command: string; args: string[]; query: string; opts: OutputOptions; values: parseArgs values }` | Parse argv, select index/config, resolve output/search options. | main dispatch |
| `getSkillInstallDir` | `(globalInstall: boolean): string` | Compute QMD skill install path. | `skill install` |
| `getClaudeSkillLinkPath` | `(globalInstall: boolean): string` | Compute Claude symlink path. | `skill install` |
| `pathExists` | `(path: string): boolean` | Safe lstat existence check. | skill install |
| `removePath` | `(path: string): void` | Remove file/symlink/directory. | skill install |
| `findPackageRoot` | `(): string | null` | Find package root containing bundled skills. | skills |
| `getSkillSearchDirs` | `(_runtimeOnly?: boolean): string[]` | Resolve bundled skill search dirs. | `skills`, `skill` |
| `parseSkillFrontmatter` | `(content: string): { name: string; description: string; hidden: boolean } | null` | Parse skill metadata from frontmatter. | `skills` |
| `discoverSkills` | `(runtimeOnly?: boolean): SkillInfo[]` | List bundled runtime skills. | `skills` |
| `findSkill` | `(name: string, runtimeOnly?: boolean): SkillInfo | null` | Locate one skill by name. | `skills`, `skill` |
| `readSkillContent` | `(skill: SkillInfo): string` | Read a skill's `SKILL.md`. | `skills`, `skill show` |
| `collectSkillFiles` | `(skill: SkillInfo): { relativePath: string; content: string }[]` | Read supplementary skill files. | `skills get --full` |
| `showSkill` | `(): void` | Print bundled QMD skill. | `--skill`, `skill show` |
| `copyDirectoryContents` | `(sourceDir: string, targetDir: string): void` | Recursive file copy. | `skill install` |
| `installedSkillStubContent` | `(): string` | Generate installed bootstrap skill content. | `skill install` |
| `writeSkillInstall` | `(targetDir: string, force: boolean): void` | Install bundled QMD skill stub/files. | `skill install` |
| `outputSkillsJson` | `(payload: unknown): void` | Print compact skills JSON. | `skills --json` |
| `runSkillsCommand` | `(args: string[], jsonMode: boolean, fullOption?: boolean, allOption?: boolean): void` | Dispatch `skills list|get|path`. | `skills` |
| `showSkillsHelp` | `(): void` | Print skills help. | `skills help` |
| `ensureClaudeSymlink` | `(linkPath: string, targetDir: string, force: boolean): boolean` | Create/update Claude skill symlink. | `skill install` |
| `shouldCreateClaudeSymlink` | `(linkPath: string, autoYes: boolean): Promise<boolean>` | Prompt or auto-confirm Claude symlink creation. | `skill install` |
| `installSkill` | `(globalInstall: boolean, force: boolean, autoYes: boolean): Promise<void>` | Install QMD agent skill and optional Claude link. | `skill install` |
| `showHelp` | `(): void` | Print top-level help. | `--help`, missing command |
| `doctorCheck` | `(label: string, ok: boolean, details: string): void` | Print diagnostic check line. | `doctor` |
| `formatCount` | `(n: number): string` | Locale-format counts. | `embed`, `doctor` |
| `shortModelName` | `(model: string): string` | Shorten model URI/name. | `embed`, `doctor` |
| `normalizedDoctorNextSteps` | `(steps: string[]): string[]` | De-duplicate and normalize doctor next steps. | `doctor` |
| `shortHashSeq` | `(hashSeq: string): string` | Shorten vector chunk identifiers. | `doctor` |
| `decodeStoredEmbedding` | `(bytes: Uint8Array): Float32Array` | Decode stored vector bytes. | `doctor` |
| `cosineDistance` | `(a: ArrayLike<number>, b: ArrayLike<number>): number` | Compute vector distance. | `doctor` |
| `formatModelDiagnosticPath` | `(path: string): string` | Display model cache path compactly. | `doctor` |
| `findCachedModelInspection` | `(model: string): CachedModelInspection` | Inspect cached GGUF model metadata. | `doctor` |
| `envValueForDisplay` | `(value: string): string` | Redact/shorten env values. | `doctor` |
| `collectEnvironmentOverrides` | `(activeModels: {...}, configModels?: ModelsConfig): EnvOverride[]` | Find model/GPU env overrides. | `doctor` |
| `checkDoctorIndexConfig` | `(nextSteps: string[]): DoctorConfigCheck` | Diagnose index/config presence. | `doctor` |
| `checkEnvironmentOverrides` | `(activeModels: {...}, configModels?: ModelsConfig): void` | Print env override diagnostics. | `doctor` |
| `checkModelDefaults` | `(activeModels: {...}, configModels?: ModelsConfig): void` | Print model default/config diagnostics. | `doctor` |
| `checkModelCache` | `(activeModels: {...}, nextSteps: string[]): void` | Check local model cache state. | `doctor` |
| `checkEmbeddingVectorSamples` | `(db: Database, model: string, fingerprint: string, sampleSize?: number): Promise<DoctorVectorSampleResult>` | Validate sampled stored embeddings. | `doctor` |
| `hasLibraryInDirs` | `(libraryBaseName: string, dirs: string[]): boolean` | Detect native library files. | `doctor` |
| `linuxCudaRuntimeDiagnostic` | `(): string | null` | Diagnose Linux CUDA runtime availability. | `doctor` |
| `runDoctorDeviceChecks` | `(nextSteps: string[]): Promise<void>` | Run llama.cpp/device diagnostics. | `doctor` |
| `showDoctor` | `(): Promise<void>` | Print full diagnostic report. | `doctor` |
| `printDoctorHint` | `(): void` | Print suggestion to run `qmd doctor`. | error paths |
| `exitWithError` | `(error: unknown, code?: number): never` | Print error plus doctor hint and exit. | `init`, `embed`, `skill install` |
| `readPackageJson` | `(): PackageJson` | Read package metadata. | `version` |
| `showVersion` | `(): Promise<void>` | Print package version and git commit when available. | `--version` |

## `src/cli/formatter.ts`

| Function | Signature | Purpose | Used by command(s) |
|---|---|---|---|
| `addLineNumbers` | `(text: string, startLine?: number): string` | Prefix each line with `line: content`. | Formatter outputs; CLI imports store version instead |
| `getDocid` | `(hash: string): string` | Return first six hash chars. | Formatter helper |
| `escapeCSV` | `(value: string | null | number): string` | CSV-quote strings with comma, quote, or newline. | Search/document CSV; CLI output |
| `escapeXml` | `(str: string): string` | XML-escape special characters. | XML outputs; CLI multi-get |
| `searchResultsToJson` | `(results: SearchResult[], opts?: FormatOptions): string` | Format search rows as pretty JSON. | Generic formatter |
| `searchResultsToCsv` | `(results: SearchResult[], opts?: FormatOptions): string` | Format search rows as CSV with snippets. | Generic formatter |
| `searchResultsToFiles` | `(results: SearchResult[]): string` | Format search rows as `#docid,score,path[,context]`. | Generic formatter |
| `searchResultsToMarkdown` | `(results: SearchResult[], opts?: FormatOptions): string` | Format search rows as Markdown sections. | Generic formatter |
| `searchResultsToXml` | `(results: SearchResult[], opts?: FormatOptions): string` | Format search rows as `<file>` blocks. | Generic formatter |
| `searchResultsToMcpCsv` | `(results: { docid; file; title; score; context; snippet }[]): string` | Format MCP search rows with precomputed snippets. | MCP integrations |
| `documentsToJson` | `(results: MultiGetFile[]): string` | Format multi-document results as JSON. | Generic formatter |
| `documentsToCsv` | `(results: MultiGetFile[]): string` | Format multi-document results as CSV. | Generic formatter |
| `documentsToFiles` | `(results: MultiGetFile[]): string` | Format multi-document results as path list with skip markers. | Generic formatter |
| `documentsToMarkdown` | `(results: MultiGetFile[]): string` | Format multi-document results as Markdown sections/code blocks. | Generic formatter |
| `documentsToXml` | `(results: MultiGetFile[]): string` | Format multi-document results as XML document list. | Generic formatter |
| `documentToJson` | `(doc: DocumentResult): string` | Format one document as JSON metadata/body. | Generic formatter |
| `documentToMarkdown` | `(doc: DocumentResult): string` | Format one document as Markdown. | Generic formatter |
| `documentToXml` | `(doc: DocumentResult): string` | Format one document as XML. | Generic formatter |
| `formatDocument` | `(doc: DocumentResult, format: OutputFormat): string` | Dispatch single-document formatter; defaults CLI to Markdown. | Generic formatter |
| `formatSearchResults` | `(results: SearchResult[], format: OutputFormat, opts?: FormatOptions): string` | Dispatch search formatter by output kind. | Imported by CLI; available to callers |
| `formatDocuments` | `(results: MultiGetFile[], format: OutputFormat): string` | Dispatch multi-document formatter by output kind. | Imported by CLI; available to callers |
