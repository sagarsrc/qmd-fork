# QMD Master Doc

Reference script for presenting QMD to a technical audience.

Use this as the talk track:

1. What QMD is.
2. Why it exists.
3. What it can do.
4. How retrieval works.
5. Why the architecture matters.

---

## What is QMD?

QMD is a local-first search engine for markdown files.

It indexes folders you configure, stores document content by SHA-256 hash, and lets you search by:

- **keywords** using BM25 / SQLite FTS5
- **meaning** using vector similarity
- **hybrid retrieval** using query expansion, RRF fusion, and reranking

In one sentence: **QMD turns a folder of markdown into a local, agent-friendly retrieval system.**

```mermaid
flowchart TD
    A["Markdown folders"] --> B["QMD index"]
    B --> C["BM25 keyword search"]
    B --> D["Vector semantic search"]
    B --> E["Hybrid query + rerank"]
    C --> F["Useful document context"]
    D --> F
    E --> F
```

## What problem does it solve?

QMD sits between `grep` and a full cloud search stack.

- `grep` is fast but only finds exact text.
- Cloud search is powerful but requires infra, network calls, and data leaving your machine.
- QMD gives local search with both lexical and semantic retrieval.

Good example: if your notes contain “token expiration handling,” but you search for “auth timeout bug,” QMD can still find the relevant document.

## Who is it for?

QMD is useful for people with large local text corpora:

- developers searching docs, ADRs, notes, code-adjacent markdown
- researchers searching paper summaries or literature notes
- writers maintaining knowledge bases
- AI agents that need reliable local document retrieval

The core user is someone who wants **private, local, scriptable retrieval** over markdown.

## What can QMD do?

Core capabilities:

1. **Index collections** — named folders of markdown files.
2. **Search lexically** — BM25 over path, title, and body.
3. **Search semantically** — embeddings + sqlite-vec cosine similarity.
4. **Do hybrid search** — query expansion + BM25 + vector + RRF + reranking.
5. **Retrieve evidence** — full docs, snippets, line ranges, glob batches.
6. **Expose retrieval everywhere** — CLI, TypeScript SDK, MCP stdio, MCP HTTP.
7. **Stay local** — SQLite index + local GGUF models.

## What does local-first mean here?

Local-first means:

- documents stay on disk
- index lives in a local SQLite DB
- models are downloaded to `~/.cache/qmd/models`
- no external API is required for normal search

This is useful when the corpus is private, large, or frequently queried by local agents.

## What tech does QMD use?

Important pieces:

| Area | Technology |
|---|---|
| Runtime | TypeScript on Node or Bun |
| Keyword search | SQLite FTS5 / BM25 |
| Vector search | sqlite-vec |
| Local ML | node-llama-cpp + GGUF models |
| Config | YAML collection config |
| Code chunking | optional web-tree-sitter AST breakpoints |
| Interfaces | CLI, SDK, MCP |

## How is QMD different from grep?

`grep` answers: “Which files contain this exact string?”

QMD answers: “Which documents are probably relevant to this question?”

Differences:

- ranks results by relevance
- searches title/path/body with different weights
- supports semantic matches through embeddings
- returns snippets, docids, line ranges, and context
- can retrieve batches for agents

## How is QMD different from Elasticsearch or cloud search?

QMD is much smaller and local.

- no server cluster
- no cloud dependency
- no hosted vector DB
- no data leaving the machine
- simple SQLite-backed index

It is not trying to replace enterprise search. It is trying to make local markdown corpora searchable and agent-friendly.

---

## How is QMD structured?

QMD has four main layers: interfaces, core store, subsystems, and storage.

```mermaid
flowchart TD
    A["Interfaces<br/>CLI / SDK / MCP"] --> B["Core store<br/>src/store.ts"]
    B --> C["Subsystems<br/>DB / LLM / Config / AST"]
    C --> D["Storage<br/>SQLite / YAML / GGUF"]
```

**What:** User-facing interfaces stay thin. The store owns the real indexing, retrieval, chunking, and search orchestration.

**Why:** This keeps behavior consistent across CLI, SDK, and MCP. If search changes in `src/store.ts`, all interfaces benefit.

## How does search flow internally?

```mermaid
flowchart TD
    A["CLI / SDK / MCP query"] --> B["BM25 probe"]
    B --> C{"strong?"}
    C -->|yes| D["skip expansion"]
    C -->|no| E["LLM expandQuery"]
    D --> F["BM25 + vector search"]
    E --> F
    F --> G["RRF fusion + chunk"]
    G --> H["rerank + blend"]
    H --> I["results"]
```

**What:** Search starts with a cheap BM25 probe, then optionally expands into lexical, semantic, and HyDE queries.

**Why:** Cheap exact matching handles obvious queries quickly; hybrid retrieval helps when wording differs from document text.

## How does indexing flow internally?

```mermaid
flowchart TD
    A["fast-glob scan"] --> B["read + hash + title"]
    B --> C{"exists?"}
    C -->|new| D["insert content + doc"]
    C -->|changed| E["update doc + FTS"]
    C -->|same| F["skip"]
    D --> G["deactivate missing<br/>cleanup orphans"]
    E --> G
    F --> G
```

**What:** Indexing scans configured collections, hashes files, updates changed docs, and keeps FTS rows current.

**Why:** Hash-based indexing avoids unnecessary writes and lets QMD deduplicate content while detecting real file changes.

---

## Retrieval in QMD

Retrieval in QMD means: turn a user question or document identifier into useful markdown content with enough context for a human or agent to act on it.

```mermaid
flowchart TD
    A["User asks/searches"] --> B["1. Find candidates<br/>BM25 / vector / hybrid"]
    B --> C["2. Resolve documents<br/>collection + path + docid"]
    C --> D["3. Return useful context<br/>snippet / full body / line numbers"]
    D --> E["CLI / SDK / MCP output"]
```

### 1. Candidate discovery

QMD first finds likely-relevant documents.

```mermaid
flowchart TD
    A["query"] --> B["keyword signal<br/>BM25 / FTS5"]
    A --> C["semantic signal<br/>embedding search"]
    B --> D["candidate list"]
    C --> D
    D --> E["optional fusion<br/>RRF + rerank"]
```

**What:** QMD builds a candidate set using lexical search, semantic search, or both.

**Why:** Different questions need different signals: exact names, semantic meaning, or a blend of both.

- `qmd search` uses **BM25 / FTS5** for exact keyword-style matching.
- `qmd vsearch` uses **vector similarity** for semantic matching.
- `qmd query` combines both, optionally expands the query, fuses rankings with RRF, and reranks best chunks.

### 2. Document resolution

Once candidates exist, QMD maps them back to stable document identities.

```mermaid
flowchart TD
    A["candidate result"] --> B["collection name"]
    A --> C["relative path"]
    A --> D["content hash"]
    B --> E["virtual path<br/>qmd://collection/file.md"]
    C --> E
    D --> F["docid<br/>#abc123"]
```

**What:** QMD turns search hits into stable identifiers: collection, relative path, virtual path, and docid.

**Why:** Agents need stable references they can cite and retrieve later, not fragile absolute filesystem paths.

- Documents live inside named **collections**.
- Results use virtual paths like `qmd://docs/api/auth.md`.
- Docids like `#abc123` point to content hashes, so they are easy to cite and retrieve later.

### 3. Context packaging

QMD then returns content in a form useful for the caller.

```mermaid
flowchart TD
    A["resolved document"] --> B{"caller needs"}
    B -->|search| C["snippet + score + docid"]
    B -->|get| D["full body or line range"]
    B -->|multi-get| E["batch of documents"]
    C --> F["CLI / SDK / MCP"]
    D --> F
    E --> F
```

**What:** QMD packages the resolved document as a snippet, full body, line range, or batch.

**Why:** Retrieval is not just “find a file”; it is “return the right evidence in the right shape.”

- Search returns ranked files with snippets, scores, docids, and context.
- `qmd get` returns a full document or a line range.
- `qmd multi-get` batches documents for agent workflows.
- MCP and SDK expose the same retrieval path programmatically.

## Mental Model

Think of QMD retrieval as a three-step pipeline:

1. **Find** relevant candidates.
2. **Resolve** candidates into stable document identities.
3. **Package** content for humans, scripts, SDK callers, or MCP agents.

## Closing Framing

If you are presenting QMD, the simplest framing is:

> QMD is a local retrieval layer for markdown. It gives agents and humans a way to search, cite, and retrieve private knowledge using both classic IR and local ML.
