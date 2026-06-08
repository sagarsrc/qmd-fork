# CLI Diagrams

## CLI Command Dispatch Table

```mermaid
graph TD
    CMD["argv command"] --> CAT{category}

    CAT --> META["help / version / status / doctor"]
    CAT --> CFG["context add/list/rm<br/>collection add/rm/rename/list/update<br/>init"]
    CAT --> DOC["get / multi-get / ls"]
    CAT --> IDX["update / embed / pull"]
    CAT --> SRCH["search / vsearch / query / deep-search"]
    CAT --> SRV["mcp / bench"]

    SRCH --> SE["searchFTS / searchVec<br/>expandQuery + hybridSearch"]
    IDX --> IE["reindexCollection<br/>generateEmbeddings<br/>pullModels"]
```

## Output Format Pipeline

```mermaid
graph TD
    A["Store results<br/>SearchResult[]"] --> B["CLI decoration<br/>context / docid / displayPath<br/>full-path policy / chunk/snippet info"]
    B --> C["outputResults()<br/>or formatter.ts dispatcher"]
    C --> D["cli text"]
    C --> E["json []"]
    C --> F["csv hdr"]
    C --> G["md ---"]
    C --> H["xml &lt;file&gt;"]
    D --> I[stdout]
    E --> I
    F --> I
    G --> I
    H --> I
```

## Lifecycle Diagram

```mermaid
graph TD
    A["parseCLI()"] --> B["command dispatch"]
    B --> C["getStore() + getDb()<br/>create / load / sync / open"]
    C --> D["command work<br/>store / collections / LLM"]
    D --> E["print output + closeDb"]
    E --> F["finishSuccessful:<br/>flush + dispose + exitCode = 0"]
```
