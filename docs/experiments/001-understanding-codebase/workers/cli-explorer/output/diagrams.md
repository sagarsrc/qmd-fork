# CLI Diagrams

## CLI Command Dispatch Table

```mermaid
graph TD
    subgraph "Help / Info"
        H1["--help / empty"] --> H2["showHelp"]
        H3["--version"] --> H4["showVersion"]
        H5["--skill"] --> H6["showSkill"]
    end

    subgraph "Context"
        C1["context add"] --> C2["contextAdd"]
        C3["context list"] --> C4["contextList"]
        C5["context rm/remove"] --> C6["contextRemove"]
    end

    subgraph "Document Retrieval"
        R1["get"] --> R2["getDocument"]
        R3["multi-get"] --> R4["multiGet"]
        R5["ls"] --> R6["listFiles"]
    end

    subgraph "Collection Management"
        M1["collection list"] --> M2["collectionList"]
        M3["collection add"] --> M4["collectionAdd"] --> M5["indexFiles"]
        M6["collection rm"] --> M7["collectionRemove"]
        M8["collection rename"] --> M9["collectionRename"]
        M10["collection update"] --> M11["updateCollectionSettings"]
        M12["collection include"] --> M11
        M13["collection show"] --> M14["getCollection"] --> M15["print"]
    end

    subgraph "Other Commands"
        O1["init"] --> O2["initLocalIndex"]
        O3["status"] --> O4["showStatus"]
        O5["doctor"] --> O6["showDoctor"]
        O7["update"] --> O8["updateCollections"]
        O9["embed"] --> O10["vectorIndex"]
        O11["pull"] --> O12["pullModels"]
        O13["bench"] --> O14["runBenchmark"]
    end

    subgraph "Search"
        S1["search"] --> S2["search"]
        S3["vsearch"] --> S4["vectorSearch"]
        S5["vector-search"] --> S6["vectorSearch"]
        S7["query"] --> S8["querySearch"]
        S9["deep-search"] --> S10["querySearch"]
    end

    subgraph "MCP & Skills"
        P1["mcp"] --> P2["startMcpServer/startMcpHttp..."]
        P3["mcp stop"] --> P4["PID kill/cleanup"]
        P5["skills"] --> P6["runSkillsCommand"]
        P7["skill show"] --> P8["showSkill"]
        P9["skill install"] --> P10["installSkill"]
        P11["cleanup"] --> P12["cache/vector/doc cleanup"]
    end
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
    B --> C["getStore()<br/>- createStore<br/>- loadConfig<br/>- syncConfigToDb<br/>- setDefaultLlamaCpp"]
    C --> D["getDb()<br/>active SQLite handle"]
    D --> E["command work<br/>store/collections/LLM"]
    E --> F["print output<br/>stdout/stderr"]
    F --> G["closeDb()<br/>store.close(); null"]
    G --> H["finishSuccessful...<br/>flush stdout<br/>dispose llama.cpp<br/>flush stderr<br/>process.exitCode = 0"]
```
