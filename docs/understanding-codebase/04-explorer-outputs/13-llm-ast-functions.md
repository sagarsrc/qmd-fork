## src/llm.ts

### Exported Functions

`setNodeLlamaCppModuleForTest(module: NodeLlamaCppModule | null): void`

Installs or clears a test double for the `node-llama-cpp` module and resets GPU warning/probe state.

`withNativeStdoutRedirectedToStderr<T>(fn: () => Promise<T>): Promise<T>`

Runs an async native llama operation while temporarily routing stdout writes to stderr.

`isQwen3EmbeddingModel(modelUri: string): boolean`

Detects whether a model URI should use Qwen3-Embedding prompt formatting.

`formatQueryForEmbedding(query: string, modelUri?: string): string`

Formats a search query for the active embedding model family.

`formatDocForEmbedding(text: string, title?: string, modelUri?: string): string`

Formats document text and optional title for the active embedding model family.

`resolveEmbedModel(config?: ModelResolutionConfig): string`

Resolves the embedding model URI from config, env, or default.

`resolveGenerateModel(config?: ModelResolutionConfig): string`

Resolves the generation model URI from config, env, or default.

`resolveRerankModel(config?: ModelResolutionConfig): string`

Resolves the rerank model URI from config, env, or default.

`resolveModels(config?: ModelResolutionConfig): Required<ModelResolutionConfig>`

Resolves embed, generate, and rerank model URIs together.

`inspectGgufFile(filePath: string): GgufFileInspection`

Inspects a local file and reports whether it exists and has valid GGUF magic.

`pullModels(models: string[], options?: { refresh?: boolean; cacheDir?: string }): Promise<PullResult[]>`

Downloads or refreshes model files into the cache and validates them as GGUF files.

`resolveParallelismOverride(envValue?: string): number | undefined`

Parses `QMD_EMBED_PARALLELISM`, returning a capped explicit parallelism value or `undefined`.

`resolveSafeParallelism(options: ParallelismOptions): number`

Applies hardware/env/platform rules to choose safe embedding/rerank context parallelism.

`resolveLlamaGpuMode(envValue?: string, forceCpuValue?: string): LlamaGpuMode`

Resolves GPU mode from `QMD_LLAMA_GPU` and `QMD_FORCE_CPU`.

`withLLMSession<T>(fn: (session: ILLMSession) => Promise<T>, options?: LLMSessionOptions): Promise<T>`

Runs work with a scoped default LLM session and releases it automatically.

`withLLMSessionForLlm<T>(llm: LlamaCpp, fn: (session: ILLMSession) => Promise<T>, options?: LLMSessionOptions): Promise<T>`

Runs work with a scoped session around a specific `LlamaCpp` instance.

`canUnloadLLM(): boolean`

Reports whether the default LLM has no active sessions or in-flight operations.

`isDarwinMetalMitigationActive(): boolean`

Reports whether the macOS Metal residency-set mitigation is active.

`installDarwinExitGuard(): void`

Compatibility no-op retained for older code paths.

`isDarwinExitGuardInstalled(): boolean`

Deprecated alias for `isDarwinMetalMitigationActive()`.

`getDefaultLlamaCpp(): LlamaCpp`

Returns the singleton `LlamaCpp`, creating it if necessary.

`setDefaultLlamaCpp(llm: LlamaCpp | null): void`

Installs or clears the default `LlamaCpp` singleton, primarily for tests.

`hasDefaultLlamaCpp(): boolean`

Checks whether the default singleton already exists without creating it.

`disposeDefaultLlamaCpp(): Promise<void>`

Disposes and clears the default singleton if present.

### Exported Class Methods

`new LlamaCpp(config?: LlamaCppConfig)`

Creates a lazy local LLM wrapper with model URIs, cache directory, context size, and idle cleanup config.

`get embedModelName(): string`

Returns the resolved embedding model URI.

`get generateModelName(): string`

Returns the resolved generation model URI.

`get rerankModelName(): string`

Returns the resolved rerank model URI.

`unloadIdleResources(): Promise<void>`

Disposes cached embedding/rerank contexts and optionally models while keeping the instance reusable.

`tokenize(text: string): Promise<readonly LlamaToken[]>`

Tokenizes text with the embedding model tokenizer.

`countTokens(text: string): Promise<number>`

Counts embedding-model tokens for text.

`detokenize(tokens: readonly LlamaToken[]): Promise<string>`

Converts embedding-model tokens back into text.

`embed(text: string, options?: EmbedOptions): Promise<EmbeddingResult | null>`

Returns one embedding result or `null` on failure.

`embedBatch(texts: string[], options?: EmbedOptions): Promise<(EmbeddingResult | null)[]>`

Embeds many texts using one or more embedding contexts.

`generate(prompt: string, options?: GenerateOptions): Promise<GenerateResult | null>`

Generates text from a prompt using a fresh generation context.

`modelExists(modelUri: string): Promise<ModelInfo>`

Checks local-path existence or assumes HuggingFace model URIs are available.

`expandQuery(query: string, options?: { context?: string; includeLexical?: boolean; intent?: string }): Promise<Queryable[]>`

Uses the generation model to produce typed `lex`, `vec`, and `hyde` query variants.

`rerank(query: string, documents: RerankDocument[], options?: RerankOptions): Promise<RerankResult>`

Scores documents by relevance to a query using ranking contexts.

`getDeviceInfo(options?: { allowBuild?: boolean }): Promise<{ gpu: string | false; gpuOffloading: boolean; gpuDevices: string[]; vram?: { total: number; used: number; free: number }; cpuCores: number }>`

Returns runtime GPU/CPU and optional VRAM diagnostics.

`dispose(): Promise<void>`

Disposes contexts, models, runtime, timers, and load promises.

### Exported Error Class

`new SessionReleasedError(message?: string)`

Creates the error thrown when a released or aborted scoped LLM session is used.

## src/ast.ts

### Exported Functions

`detectLanguage(filepath: string): SupportedLanguage | null`

Maps a filepath extension to a supported tree-sitter language.

`formatGrammarLoadError(language: SupportedLanguage, err: unknown): string`

Builds a user-facing grammar load failure message with a repair command.

`getASTBreakPoints(content: string, filepath: string): Promise<BreakPoint[]>`

Parses a supported code file and returns scored AST boundary breakpoints, or `[]` on any failure.

`getASTStatus(): Promise<{ available: boolean; languages: { language: SupportedLanguage; available: boolean; error?: string }[] }>`

Checks tree-sitter initialization, grammar loading, and query compilation status for all supported languages.

`extractSymbols(_content: string, _language: string, _startPos: number, _endPos: number): SymbolInfo[]`

Phase 2 stub for symbol extraction; currently always returns `[]`.
