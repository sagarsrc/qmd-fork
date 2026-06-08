## DB initialization flow

```mermaid
graph TD
    A["module load: db.ts"] --> B["is 'Bun' in global?"]
    B -->|yes| C["import dynamic<br/>'bun:sqlite'"]
    B -->|no| D["import<br/>better-sqlite3"]
    C --> E[darwin?]
    D --> F["import sqlite-vec<br/>package loader"]
    E -->|yes/no| G["try Homebrew<br/>setCustomSQLite"]
    G --> H["test sqlite-vec<br/>on :memory: DB"]
    H -->|ok/fail| I["set _Database and<br/>set _sqliteVecLoad or null"]
    F --> I
    I --> J["openDatabase(path)<br/>> new _Database(path)"]
    J --> K["store.createStore(path)<br/>> open database<br/>> loadSqliteVec(db)<br/>> verify vec if available<br/>> create schema"]
```

## Config load flow

```mermaid
graph TD
    A[loadConfig] --> B["configSource.type == inline?"]
    B -->|yes| C["return in-memory<br/>CollectionConfig"]
    B -->|no| D["configSource.path exists?"]
    D -->|yes| E["use custom path"]
    D -->|no| F["use default path<br/>configDir/index.yml"]
    E --> G["does file exist?"]
    F --> G
    G -->|yes| H["read YAML<br/>parse YAML"]
    G -->|no| I["return<br/>{ collections: {} }"]
    H --> J["ensure config.collections<br/>return parsed config"]
```

Project-local discovery used by the CLI before default config:

```mermaid
graph TD
    A["no explicit --index"] --> B["findLocalConfigPath(cwd)"]
    B --> C["check dir/.qmd/index.yaml<br/>check dir/.qmd/index.yml"]
    C --> D["found?"]
    D -->|yes| E["setConfigSource<br/>local YAML path<br/>set DB override<br/>to index.sqlite"]
    D -->|no| F["move to parent dir<br/>until filesystem root"]
```

## Config sync flow

```mermaid
graph TD
    A["CLI/store loads config<br/>loadConfig()"] --> B["syncConfigToDb(db, config)"]
    B --> C["JSON.stringify(config)<br/>SHA-256 hash"]
    C --> D["read store_config<br/>key = config_hash"]
    D --> E["hash matches?"]
    E -->|yes| F[return early]
    E -->|no| G["for each config collection<br/>> upsert store_collections"]
    G --> H["read DB collection names<br/>delete names missing config"]
    H --> I["sync global_context<br/>into store_config"]
    I --> J["write store_config<br/>key = config_hash"]
```

## Collections + contexts data model

```mermaid
graph TD
    subgraph "CollectionConfig YAML"
        Y1["global_context: &quot;...applies to all...&quot;<br/>editor_uri: &quot;...&quot;<br/>editor_uri_template: &quot;...&quot;<br/>models:<br/>  embed: &quot;...&quot;<br/>  rerank: &quot;...&quot;<br/>  generate: &quot;...&quot;<br/>collections:<br/>  notes:<br/>    path: &quot;/abs/path/to/notes&quot;<br/>    pattern: &quot;**/*.md&quot;<br/>    ignore:<br/>      - &quot;Archive/**&quot;<br/>    update: &quot;git pull&quot;<br/>    includeByDefault: false<br/>    context:<br/>      &quot;/&quot;: &quot;root collection context&quot;<br/>      &quot;/2024&quot;: &quot;more specific context&quot;<br/>      &quot;/2024/board&quot;: &quot;most specific context&quot;<br/>  work:<br/>    path: &quot;/abs/path/to/work&quot;<br/>    pattern: &quot;**/*.md&quot;"]
    end

    Y1 --> S1

    subgraph "SQLite mirror"
        S1["store_collections<br/>  name: notes<br/>  path: /abs/path/to/notes<br/>  pattern: **/*.md<br/>  ignore_patterns: JSON string or NULL<br/>  include_by_default: 0 or 1<br/>  update_command: string or NULL<br/>  context: JSON ContextMap or NULL<br/><br/>store_config<br/>  global_context: string or absent<br/>  config_hash: sha256(JSON config)"]
    end

    S1 --> L1

    subgraph "Context lookup"
        L1["input: collectionName + filePath<br/>normalize to leading slash<br/>collect matching context prefixes<br/>sort by longest prefix<br/>return most specific match<br/>otherwise return global_context"]
    end
```
