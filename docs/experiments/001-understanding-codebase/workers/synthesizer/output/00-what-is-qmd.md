# What is QMD?

## What is QMD?
QMD is a local-first search engine for your markdown files. It indexes folders you configure, stores content by SHA-256 hash, and lets you search by keyword (BM25) or meaning (vector similarity) using local GGUF models (see store-explorer: Overview; cli-explorer: Overview).

## What problem does it solve?
It bridges the gap between `grep`-style literal lookup and cloud-hosted semantic search. If you have thousands of markdown notes and want to ask "how do I handle errors in React?" without remembering filenames or exact phrases, QMD gives you relevant results locally (see L0-overview: Problem It Solves).

## What can it do?
- Index one or more collections of markdown (and optionally code) files.
- Search with fast BM25 keyword search.
- Search with vector semantic similarity.
- Combine both with automatic query expansion, reciprocal-rank fusion, and LLM reranking.
- Retrieve documents by virtual path, filesystem path, glob, or short docid.
- Attach human-written context to collections or paths to improve relevance.
- Expose the same index through CLI, TypeScript SDK, MCP stdio, and MCP HTTP.
- Run entirely offline with downloaded GGUF models (see L0-overview: Key Capabilities).

## Who is it for?
Developers, writers, and researchers who maintain large local markdown corpora and want fast, private search without shipping data to a cloud service.

## What does "local-first" mean here?
Your documents stay on disk. The index lives in a local SQLite file. Models are downloaded to `~/.cache/qmd/models`. Search does not call external APIs (see cli-explorer: Commands; llm-ast-explorer: Model Loading).

## What tech does it use?
- TypeScript on Node or Bun.
- SQLite with FTS5 (BM25) and optional sqlite-vec (cosine vector search).
- node-llama-cpp for embeddings, query expansion, and reranking.
- YAML for collection configuration.
- Optional web-tree-sitter for AST-aware chunking of code files (see L0-overview: Tech Stack Summary).

## How is it different from grep / ripgrep?
`grep` matches literal substrings. QMD understands relevance scoring across title, path, and body; it can rank by semantic meaning even when no keyword matches exactly; and it supports docids, snippets, and context-aware retrieval (see store-explorer: BM25 Search; cli-explorer: `get`).

## How is it different from Elasticsearch / cloud search?
No server cluster, no network round-trips, no subscription. The index is a single SQLite file on your machine, and the embedding model runs locally. Setup is one command, not a deployment (see L0-overview: System Architecture).
