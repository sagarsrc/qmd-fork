## LLM Lifecycle

```text
+----------------------+
| getDefaultLlamaCpp() |
+----------+-----------+
           |
           v
+----------------------+
| LlamaCpp instance    |
| no native load yet   |
+----------+-----------+
           |
           v
+----------------------+
| first operation      |
| embed/generate/rank  |
+----------+-----------+
           |
           v
+----------------------+
| ensureLlama()        |
| load native runtime  |
+----------+-----------+
           |
           v
+----------------------+
| ensure model         |
| embed/gen/rerank     |
+----------+-----------+
           |
           v
+----------------------+
| create context(s)    |
| run operation        |
+----------+-----------+
           |
           v
+----------------------+
| touchActivity()      |
| reset idle timer     |
+----------+-----------+
           |
           v
+----------------------+
| inactivity timeout   |
+----------+-----------+
           |
           v
+----------------------+
| canUnloadLLM()?      |
+----+------------+----+
     | yes        | no
     v            v
+----------------+  +----------------------+
| dispose idle   |  | reschedule timer     |
| contexts       |  | active session/op     |
+-------+--------+  +----------+-----------+
        |                      |
        v                      |
+----------------------+       |
| keep models loaded   | <-----+
| unless opt-in dispose|
+----------------------+
```

## Embedding Pipeline

```text
+----------------------+
| input text           |
+----------+-----------+
           |
           v
+----------------------+
| format for model     |
| query or document    |
+----------+-----------+
           |
           v
+----------------------+
| ensure embed model   |
| and context pool     |
+----------+-----------+
           |
           v
+----------------------+
| tokenize             |
| model tokenizer      |
+----------+-----------+
           |
           v
+----------------------+
| truncate if needed   |
| context-safe text    |
+----------+-----------+
           |
           v
+----------------------+
| getEmbeddingFor()    |
| native vector        |
+----------+-----------+
           |
           v
+----------------------+
| Array.from(vector)   |
| public number[]      |
+----------------------+
```

## Query Expansion Flow

```text
+----------------------+
| user query           |
+----------+-----------+
           |
           v
+----------------------+
| optional intent      |
| build /no_think      |
| prompt               |
+----------+-----------+
           |
           v
+----------------------+
| LLM generation       |
| grammar constrained  |
+----------+-----------+
           |
           v
+----------------------+
| lines of output      |
| type: content        |
+----------+-----------+
           |
           v
+----------------------+
| parse and filter     |
| lex/vec/hyde only    |
+----------+-----------+
           |
           v
+----------------------+
| queryables           |
+----------+-----------+
           |
           +------------------+------------------+
           |                  |                  |
           v                  v                  v
+----------------+  +----------------+  +----------------+
| lex subqueries |  | vec subqueries |  | hyde subqueries|
+----------------+  +----------------+  +----------------+
```

## AST Chunking Flow

```text
+----------------------+
| file content + path  |
+----------+-----------+
           |
           v
+----------------------+
| detectLanguage()     |
+----------+-----------+
           |
           | unsupported
           +--------------------+
           |                    v
           |          +----------------------+
           |          | [] AST breakpoints   |
           |          +----------+-----------+
           |                     |
           v supported          |
+----------------------+         |
| ensureInit()         |         |
| web-tree-sitter      |         |
+----------+-----------+         |
           |                     |
           v                     |
+----------------------+         |
| loadGrammar()        |         |
| cached WASM grammar  |         |
+----------+-----------+         |
           | fail                |
           +-------------------->+
           |
           v
+----------------------+
| parse source         |
+----------+-----------+
           | fail
           +-------------------->+
           |
           v
+----------------------+
| getQuery()           |
| cached S-expression |
+----------+-----------+
           |
           v
+----------------------+
| query captures       |
+----------+-----------+
           |
           v
+----------------------+
| extractBreakpoints   |
| score + dedupe       |
+----------+-----------+
           |
           v
+----------------------+
| AST breakpoints      |
+----------+-----------+
           |
           v
+----------------------+
| store.ts             |
| merge with regex     |
+----------+-----------+
           |
           v
+----------------------+
| chunkDocumentWith    |
| BreakPoints()        |
+----------------------+
```

## Model Resolution

```text
+----------------------+
| model URI or path    |
+----------+-----------+
           |
           v
+----------------------+
| resolveModelFile()   |
| node-llama-cpp       |
+----------+-----------+
           |
           v
+----------------------+
| cache directory      |
| ~/.cache/qmd/models  |
+----------+-----------+
           |
           v
+----------------------+
| cache hit?           |
+-----+-----------+----+
      | yes       | no
      v           v
+-----------+  +----------------------+
| local file|  | download from HF     |
+-----+-----+  | into model cache     |
      |        +----------+-----------+
      |                   |
      +---------+---------+
                |
                v
+----------------------+
| inspect GGUF magic   |
+----------+-----------+
           |
           v
+----------------------+
| valid GGUF?          |
+-----+-----------+----+
      | yes       | no
      v           v
+-----------+  +----------------------+
| use model |  | remove bad cache    |
| path      |  | throw clear error   |
+-----------+  +----------------------+
```
