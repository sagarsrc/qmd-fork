## LLM Lifecycle

```mermaid
flowchart TD
    A["getDefaultLlamaCpp()"] --> B["lazy init:<br/>ensureLlama + model + context"]
    B --> C["run operation<br/>embed / generate / rank"]
    C --> D["touchActivity()"]
    D --> E["inactivity timeout"]
    E --> F{"canUnloadLLM?"}
    F -->|yes| G["dispose idle contexts"]
    F -->|no| H["reschedule timer"]
    G --> I["keep models loaded<br/>unless opt-in dispose"]
    H --> I
```

## Embedding Pipeline

```mermaid
flowchart TD
    A["input text"] --> B["format for model<br/>query or document"]
    B --> C["ensure embed model<br/>and context pool"]
    C --> D["tokenize<br/>model tokenizer"]
    D --> E["truncate if needed<br/>context-safe text"]
    E --> F["getEmbeddingFor()<br/>native vector"]
    F --> G["Array.from(vector)<br/>public number[]"]
```

## Query Expansion Flow

```mermaid
flowchart TD
    A["user query"] --> B["optional intent<br/>build /no_think<br/>prompt"]
    B --> C["LLM generation<br/>grammar constrained"]
    C --> D["lines of output<br/>type: content"]
    D --> E["parse and filter<br/>lex/vec/hyde only"]
    E --> F["queryables"]
    F --> G["lex subqueries"]
    F --> H["vec subqueries"]
    F --> I["hyde subqueries"]
```

## AST Chunking Flow

```mermaid
flowchart TD
    A["file content + path"] --> B{"detectLanguage()"}
    B -->|unsupported| C["[] AST breakpoints"]
    B -->|supported| D["init + loadGrammar +<br/>parse source"]
    D -->|fail| C
    D --> E["query captures +<br/>extractBreakpoints"]
    E --> F["AST breakpoints"]
    F --> G["merge with regex<br/>chunkDocumentWithBreakPoints"]
```

## Model Resolution

```mermaid
flowchart TD
    A["model URI or path"] --> B["resolveModelFile()<br/>node-llama-cpp"]
    B --> C["cache directory<br/>~/.cache/qmd/models"]
    C --> D{"cache hit?"}
    D -->|"yes"| E["local file"]
    D -->|"no"| F["download from HF<br/>into model cache"]
    E --> G["inspect GGUF magic"]
    F --> G
    G --> H{"valid GGUF?"}
    H -->|"yes"| I["use model<br/>path"]
    H -->|"no"| J["remove bad cache<br/>throw clear error"]
```
