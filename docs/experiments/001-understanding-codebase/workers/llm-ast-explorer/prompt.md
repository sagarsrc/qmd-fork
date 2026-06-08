# LLM + AST Explorer Task

Explore the QMD LLM abstraction layer and AST-aware chunking system. Read both source files and understand model lifecycle, embedding formats, and tree-sitter integration.

## Files to Read

- `/home/sagar/temp/qmd/src/llm.ts` — model lifecycle, embeddings, generation, reranking, GPU/CPU fallback
- `/home/sagar/temp/qmd/src/ast.ts` — tree-sitter chunking for code files

## Scope

DO NOT modify any source files. Read-only analysis. `llm.ts` is ~1800+ lines — read it thoroughly.

## Deliverables

Save ALL output files to `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/llm-ast-explorer/output/` — use absolute paths.

### 1. `findings.md`

Structure:
- ## LLM Layer Overview — LlamaCpp class, lazy loading, auto-dispose after inactivity
- ## Model Loading — resolveModelFile, pullModels, model cache directory, GGUF inspection
- ## Embedding Formatting — nomic-style vs Qwen3-Embedding formats, formatQueryForEmbedding, formatDocForEmbedding
- ## Embedding Generation — batching, tokenization, Float32Array output, dimension detection
- ## Query Expansion — how a user query becomes lex/vec/hyde sub-queries, prompt templates
- ## Reranking — RerankDocument interface, scoring logic, batch processing
- ## GPU/CPU Management — getLlamaGpuTypes, force CPU mode, Metal mitigation on macOS
- ## stdout Redirect — withNativeStdoutRedirectedToStderr, why it exists (JSON API safety)
- ## AST Chunking Overview — supported languages, extension map, grammar resolution
- ## Parser Initialization — web-tree-sitter lazy init, WASM loading, grammar caching
- ## Query-Based Breakpoints — tree-sitter S-expression queries per language, SCORE_MAP alignment with markdown breakpoints
- ## Graceful Degradation — parse failure → empty breakpoints → regex fallback
- ## Gotchas / Design Decisions — why node-llama-cpp not raw llama.cpp, why stdout redirect matters, grammar version pinning

### 2. `functions.md`

Inventory of all exported functions in both files with:
- Signature (params + return type)
- One-line purpose

### 3. `diagrams.md`

ASCII diagrams (NO mermaid):
- LLM lifecycle: lazy load → use → inactivity timer → auto-dispose
- Embedding pipeline: text → format → tokenize → embed → Float32Array
- Query expansion flow: user query → LLM → [lex, vec, hyde] sub-queries
- AST chunking flow: file → detectLanguage → loadGrammar → parse → query → extractBreakpoints → merge with regex
- Model resolution: URI → resolveModelFile → cache check → download → GGUF validate

Use box-drawing characters: `+`, `-`, `|`, `>`, `^`, `v`.
