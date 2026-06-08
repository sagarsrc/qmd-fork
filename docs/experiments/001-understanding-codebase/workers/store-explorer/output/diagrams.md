# Store Layer Diagrams

ASCII only. No Mermaid.

## Search Pipeline

```text
+-------------+
| user query  |
+-------------+
       |
       v
+----------------------+
| initial BM25 / FTS   |
| searchFTS(query)     |
+----------------------+
       |
       v
+------------------------------+
| strong signal?               |
| top >= 0.85 and gap >= 0.15 |
| and no intent                |
+------------------------------+
       | yes                         | no
       v                             v
+----------------+          +----------------------+
| skip expansion |          | expandQuery          |
| expanded = []  |          | -> lex / vec / hyde  |
+----------------+          +----------------------+
       |                             |
       +-------------+---------------+
                     |
                     v
        +----------------------------+
        | route retrieval lists      |
        +----------------------------+
             |                  |
             v                  v
 +---------------------+  +------------------------+
 | FTS lists           |  | vector lists           |
 | original + lex      |  | original + vec + hyde  |
 | searchFTS           |  | embedBatch + searchVec |
 +---------------------+  +------------------------+
             |                  |
             +---------+--------+
                       |
                       v
          +-------------------------+
          | weighted RRF fusion     |
          | original lists: 2.0     |
          | expansion lists: 1.0    |
          +-------------------------+
                       |
                       v
          +-------------------------+
          | top candidateLimit docs |
          | default 40              |
          +-------------------------+
                       |
                       v
          +-------------------------+
          | chunk each document     |
          | choose best text chunk  |
          +-------------------------+
                       |
          +------------+------------+
          |                         |
          v                         v
+-------------------+     +------------------------+
| skipRerank = true |     | rerank selected chunks |
| score = 1 / rank  |     | cached LLM reranker    |
+-------------------+     +------------------------+
          |                         |
          |                         v
          |              +-------------------------+
          |              | position-aware blend    |
          |              | RRF rank + rerank score |
          |              +-------------------------+
          |                         |
          +------------+------------+
                       |
                       v
          +-------------------------+
          | dedupe by file          |
          | filter minScore         |
          | slice limit             |
          +-------------------------+
                       |
                       v
                 +-----------+
                 | results   |
                 +-----------+
```

## Chunking Pipeline

```text
+----------------+
| document text  |
+----------------+
        |
        v
+-------------------------+
| scanBreakPoints         |
| regex markdown signals  |
+-------------------------+
        |
        v
+-------------------------+
| findCodeFences          |
| start/end fenced zones  |
+-------------------------+
        |
        v
+----------------------------------+
| optional AST breakpoints         |
| only chunkStrategy = auto        |
| mergeBreakPoints(regex, ast)     |
+----------------------------------+
        |
        v
+----------------------------------+
| chunkDocumentWithBreakPoints     |
| charPos -> targetEndPos          |
+----------------------------------+
        |
        v
+----------------------------------+
| findBestCutoff                   |
| search backward 200 tokens       |
| avoid code fences                |
| score * squared distance decay   |
+----------------------------------+
        |
        v
+----------------------------------+
| slice chunk                      |
| advance by end - 15% overlap     |
+----------------------------------+
        |
        v
+----------------------------------+
| token check if chunkByTokens     |
| split recursively if > 900 toks  |
+----------------------------------+
        |
        v
    +----------+
    | chunks   |
    +----------+
```

## Indexing Pipeline

```text
+----------------+
| collection cfg |
| path + glob    |
+----------------+
        |
        v
+-----------------------------+
| fast-glob file scan         |
| ignore default + user dirs  |
| filter hidden path parts    |
+-----------------------------+
        |
        v
+-----------------------------+
| for each readable non-empty |
| file                        |
+-----------------------------+
        |
        v
+-----------------------------+
| normalize relative path     |
| hashContent(body)           |
| extractTitle(body, path)    |
+-----------------------------+
        |
        v
+-----------------------------+
| findOrMigrateLegacyDocument |
| exact / case / handelized   |
+-----------------------------+
        |
        +--------------------+--------------------+
        | new                | existing           |
        v                    v                    v
+----------------+   +----------------+   +----------------+
| insertContent  |   | same hash?     |   | changed hash?  |
| insertDocument |   | title changed? |   | insertContent  |
+----------------+   +----------------+   | updateDocument |
        |                    |            +----------------+
        |                    v                    |
        |           +----------------+            |
        |           | unchanged or   |            |
        |           | updateTitle    |            |
        |           +----------------+            |
        +--------------------+--------------------+
                             |
                             v
                  +----------------------+
                  | rebuildDocumentFTS   |
                  | normalized CJK       |
                  +----------------------+
                             |
                             v
                  +----------------------+
                  | documents_fts row    |
                  | also protected by    |
                  | insert/update/delete |
                  | triggers             |
                  +----------------------+
                             |
                             v
                  +----------------------+
                  | deactivate missing   |
                  | cleanup orphans      |
                  +----------------------+
```

## Database Schema

```text
+-----------------------------+
| content                     |
|-----------------------------|
| hash TEXT PK                |
| doc TEXT NOT NULL           |
| created_at TEXT NOT NULL    |
+-----------------------------+
              ^
              |
              | documents.hash FK
              |
+-----------------------------+
| documents                   |
|-----------------------------|
| id INTEGER PK AUTOINCREMENT |
| collection TEXT NOT NULL    |
| path TEXT NOT NULL          |
| title TEXT NOT NULL         |
| hash TEXT NOT NULL          |
| created_at TEXT NOT NULL    |
| modified_at TEXT NOT NULL   |
| active INTEGER DEFAULT 1    |
| UNIQUE(collection, path)    |
+-----------------------------+
       |              |
       | triggers     | joins by hash
       v              v
+-----------------------------+      +-----------------------------+
| documents_fts               |      | content_vectors             |
|-----------------------------|      |-----------------------------|
| rowid = documents.id        |      | hash TEXT PK part           |
| filepath                    |      | seq INTEGER PK part         |
| title                       |      | pos INTEGER                 |
| body                        |      | model TEXT                  |
| tokenizer porter unicode61  |      | embed_fingerprint TEXT      |
+-----------------------------+      | total_chunks INTEGER        |
                                     | embedded_at TEXT            |
                                     +-----------------------------+
                                                   |
                                                   | hash || "_" || seq
                                                   v
                                     +-----------------------------+
                                     | vectors_vec                 |
                                     |-----------------------------|
                                     | hash_seq TEXT PK            |
                                     | embedding float[N]          |
                                     | distance_metric = cosine    |
                                     +-----------------------------+

+-----------------------------+      +-----------------------------+
| store_collections           |      | store_config                |
|-----------------------------|      |-----------------------------|
| name TEXT PK                |      | key TEXT PK                 |
| path TEXT                   |      | value TEXT                  |
| pattern TEXT                |      +-----------------------------+
| ignore_patterns TEXT JSON   |
| include_by_default INTEGER  |
| update_command TEXT         |
| context TEXT JSON           |
+-----------------------------+

+-----------------------------+
| llm_cache                   |
|-----------------------------|
| hash TEXT PK                |
| result TEXT                 |
| created_at TEXT             |
+-----------------------------+

+-----------------------------------------------+
| indexes                                       |
|-----------------------------------------------|
| idx_documents_collection(collection, active)  |
| idx_documents_hash(hash)                      |
| idx_documents_path(path, active)              |
+-----------------------------------------------+
```

## Virtual Path Resolution Flow

```text
+------------------------------+
| input path                    |
+------------------------------+
        |
        v
+------------------------------+
| is explicitly virtual?       |
| qmd:... or //collection/...  |
+------------------------------+
        | yes                         | no
        v                             v
+------------------------------+   +------------------------------+
| normalizeVirtualPath         |   | filesystem / relative path   |
| qmd:////x -> qmd://x         |   +------------------------------+
| //x -> qmd://x               |                 |
+------------------------------+                 v
        |                         +------------------------------+
        v                         | find collection whose root   |
+------------------------------+  | prefixes absolute path, or   |
| parseVirtualPath             |  | try relative path in each    |
| collectionName               |  | collection                   |
| path                         |  +------------------------------+
| optional index query         |                 |
+------------------------------+                 v
        |                         +------------------------------+
        v                         | query active document by     |
+------------------------------+  | collection + relative path   |
| getCollectionByName          |  +------------------------------+
| from store_collections       |                 |
+------------------------------+                 v
        |                         +------------------------------+
        v                         | build virtual result         |
+------------------------------+  | qmd://collection/path       |
| resolve(collection.pwd,path) |  +------------------------------+
| absolute filesystem path     |
+------------------------------+
```
