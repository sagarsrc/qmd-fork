# Store Explorer Task

Explore the QMD Store layer — the core engine. Read the source file and understand every algorithm, data structure, and search mechanism. Produce structured documentation.

## Files to Read

- `/home/sagar/temp/qmd/src/store.ts` — core engine (chunking, search, indexing, embeddings, virtual paths, docids)

## Scope

DO NOT modify any source files. DO NOT explore files outside `src/store.ts`. Read-only analysis. This file is ~5200 lines — read it in full, every function matters.

## Deliverables

Save ALL output files to `/home/sagar/temp/qmd/docs/experiments/001-understanding-codebase/workers/store-explorer/output/` — use absolute paths.

### 1. `findings.md`

Structure:
- ## Overview — what the Store layer does, its role as the system engine
- ## Chunking — regex chunking algorithm: BREAK_PATTERNS, scanBreakPoints, findBestCutoff, distance decay math. How chunk size (900 tokens), overlap (15%), window (200 tokens) interact. How code fences are avoided.
- ## BM25 Search — FTS5 setup, query construction, CJK normalization, snippet extraction
- ## Vector Search — sqlite-vec table, embedding storage, cosine similarity, dimension handling
- ## Hybrid Search — query expansion (lex/vec/hyde), parallel retrieval, RRF fusion formula, reranking pipeline
- ## Indexing — reindexCollection: file scan, content hashing, deduplication, document lifecycle (active/inactive)
- ## Embeddings Pipeline — generateEmbeddings: batching, retry logic, chunk → embed → store, failure handling
- ## Virtual Paths — qmd:// protocol, parseVirtualPath, buildVirtualPath, collection/path resolution
- ## Docids — hash → short ID, lookup, citation system
- ## Database Schema — all tables: content, documents, documents_fts, content_vectors, vectors_vec, store_collections, llm_cache, store_config. Include ASCII schema diagram.
- ## Gotchas / Design Decisions — content-addressable storage, WAL mode, trigger-based FTS sync, fingerprint-based embedding invalidation

### 2. `functions.md`

Inventory of all exported functions with:
- Signature (params + return type)
- One-line purpose
- Which subsystem uses it

### 3. `diagrams.md`

ASCII diagrams (NO mermaid):
- Search pipeline: query → expand → BM25 + vector → RRF → rerank → results
- Chunking pipeline: document → scanBreakPoints → findBestCutoff → chunks
- Indexing pipeline: file scan → hash → insertContent → insertDocument → FTS trigger
- Database schema (all tables with columns and keys)
- Virtual path resolution flow

Use box-drawing characters: `+`, `-`, `|`, `>`, `^`, `v`.
