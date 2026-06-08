# Store Layer Diagrams

ASCII only. No Mermaid.

## Search Pipeline

```mermaid
flowchart TD
    A([user query]) --> B[initial BM25 / FTS<br/>searchFTS query]
    B --> C{strong signal?<br/>top >= 0.85 and gap >= 0.15<br/>and no intent}
    C -->|yes| D[skip expansion<br/>expanded = []]
    C -->|no| E[expandQuery<br/>-> lex / vec / hyde]
    D --> F[route retrieval lists]
    E --> F
    F --> G[FTS lists<br/>original + lex<br/>searchFTS]
    F --> H[vector lists<br/>original + vec + hyde<br/>embedBatch + searchVec]
    G --> I[weighted RRF fusion<br/>original lists: 2.0<br/>expansion lists: 1.0]
    H --> I
    I --> J[top candidateLimit docs<br/>default 40]
    J --> K[chunk each document<br/>choose best text chunk]
    K --> L{skipRerank}
    L -->|true| M[score = 1 / rank]
    L -->|false| N[rerank selected chunks<br/>cached LLM reranker]
    N --> O[position-aware blend<br/>RRF rank + rerank score]
    O --> P
    M --> P[dedupe by file<br/>filter minScore<br/>slice limit]
    P --> Q([results])
```

## Chunking Pipeline

```mermaid
flowchart TD
    A([document text]) --> B[scanBreakPoints<br/>regex markdown signals]
    B --> C[findCodeFences<br/>start/end fenced zones]
    C --> D[optional AST breakpoints<br/>only chunkStrategy = auto<br/>mergeBreakPoints regex, ast]
    D --> E[chunkDocumentWithBreakPoints<br/>charPos -> targetEndPos]
    E --> F[findBestCutoff<br/>search backward 200 tokens<br/>avoid code fences<br/>score * squared distance decay]
    F --> G[slice chunk<br/>advance by end - 15% overlap]
    G --> H[token check if chunkByTokens<br/>split recursively if > 900 toks]
    H --> I([chunks])
```

## Indexing Pipeline

```mermaid
flowchart TD
    A([collection cfg<br/>path + glob]) --> B[fast-glob file scan<br/>ignore default + user dirs<br/>filter hidden path parts]
    B --> C[for each readable non-empty file]
    C --> D[normalize relative path<br/>hashContent body<br/>extractTitle body, path]
    D --> E[findOrMigrateLegacyDocument<br/>exact / case / handelized]
    E -->|new| F[insertContent<br/>insertDocument]
    E -->|existing| G{same hash?<br/>title changed?}
    E -->|changed hash| H[insertContent<br/>updateDocument]
    G -->|unchanged or updateTitle| I
    F --> I[rebuildDocumentFTS<br/>normalized CJK]
    H --> I
    I --> J[documents_fts row<br/>also protected by<br/>insert/update/delete<br/>triggers]
    J --> K[deactivate missing<br/>cleanup orphans]
```

## Database Schema

```mermaid
erDiagram
    content {
        TEXT hash PK
        TEXT doc "NOT NULL"
        TEXT created_at "NOT NULL"
    }

    documents {
        INTEGER id PK "AUTOINCREMENT"
        TEXT collection "NOT NULL"
        TEXT path "NOT NULL"
        TEXT title "NOT NULL"
        TEXT hash "NOT NULL"
        TEXT created_at "NOT NULL"
        TEXT modified_at "NOT NULL"
        INTEGER active "DEFAULT 1"
    }

    documents_fts {
        INTEGER rowid "documents.id"
        TEXT filepath
        TEXT title
        TEXT body
    }

    content_vectors {
        TEXT hash PK "part"
        INTEGER seq PK "part"
        INTEGER pos
        TEXT model
        TEXT embed_fingerprint
        INTEGER total_chunks
        TEXT embedded_at
    }

    vectors_vec {
        TEXT hash_seq PK
        float embedding "N"
    }

    store_collections {
        TEXT name PK
        TEXT path
        TEXT pattern
        TEXT ignore_patterns "JSON"
        INTEGER include_by_default
        TEXT update_command
        TEXT context "JSON"
    }

    store_config {
        TEXT key PK
        TEXT value
    }

    llm_cache {
        TEXT hash PK
        TEXT result
        TEXT created_at
    }

    documents ||--|| content : "hash FK"
    documents ||--|| documents_fts : "triggers (rowid=id)"
    content ||--|| content_vectors : "hash"
    content_vectors ||--|| vectors_vec : "hash || '_' || seq"

    idx_documents_collection["idx_documents_collection(collection, active)"]
    idx_documents_hash["idx_documents_hash(hash)"]
    idx_documents_path["idx_documents_path(path, active)"]
```

## Virtual Path Resolution Flow

```mermaid
flowchart TD
    A([input path]) --> B{is explicitly virtual?<br/>qmd:... or //collection/...}
    B -->|yes| C[normalizeVirtualPath<br/>qmd:////x -> qmd://x<br/>//x -> qmd://x]
    B -->|no| D[filesystem / relative path]
    C --> E[parseVirtualPath<br/>collectionName<br/>path<br/>optional index query]
    D --> F[find collection whose root<br/>prefixes absolute path, or<br/>try relative path in each collection]
    E --> G[getCollectionByName<br/>from store_collections]
    F --> H[query active document by<br/>collection + relative path]
    G --> I[resolve collection.pwd, path<br/>absolute filesystem path]
    H --> J[build virtual result<br/>qmd://collection/path]
```
