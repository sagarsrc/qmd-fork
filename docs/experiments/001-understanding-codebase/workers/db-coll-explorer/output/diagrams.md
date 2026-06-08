## DB initialization flow

```
+----------------------+
| module load: db.ts   |
+----------------------+
          |
          v
+----------------------+
| is "Bun" in global?  |
+----------------------+
     | yes        | no
     v            v
+----------------+  +----------------------+
| import dynamic |  | import               |
| "bun:sqlite"   |  | better-sqlite3       |
+----------------+  +----------------------+
     |                    |
     v                    v
+----------------+  +----------------------+
| darwin?        |  | import sqlite-vec    |
+----------------+  | package loader       |
     | yes/no       +----------------------+
     v                    |
+----------------+        |
| try Homebrew   |        |
| setCustomSQLite|        |
+----------------+        |
     |                    |
     v                    |
+----------------+        |
| test sqlite-vec|        |
| on :memory: DB |        |
+----------------+        |
     | ok/fail            |
     v                    v
+-------------------------------+
| set _Database and             |
| set _sqliteVecLoad or null    |
+-------------------------------+
          |
          v
+-------------------------------+
| openDatabase(path)            |
| > new _Database(path)         |
+-------------------------------+
          |
          v
+-------------------------------+
| store.createStore(path)       |
| > open database               |
| > loadSqliteVec(db)           |
| > verify vec if available     |
| > create schema               |
+-------------------------------+
```

## Config load flow

```
+------------------------------+
| loadConfig()                 |
+------------------------------+
          |
          v
+------------------------------+
| configSource.type == inline? |
+------------------------------+
     | yes              | no
     v                  v
+-------------------+  +-------------------------------+
| return in-memory  |  | configSource.path exists?     |
| CollectionConfig  |  +-------------------------------+
+-------------------+       | yes              | no
                            v                  v
                  +-------------------+  +-----------------------+
                  | use custom path   |  | use default path      |
                  +-------------------+  | configDir/index.yml   |
                            |            +-----------------------+
                            v                  |
                       +-----------------------+
                       | does file exist?      |
                       +-----------------------+
                            | yes        | no
                            v            v
                  +----------------+  +----------------------+
                  | read YAML      |  | return               |
                  | parse YAML     |  | { collections: {} }  |
                  +----------------+  +----------------------+
                            |
                            v
                  +-----------------------------+
                  | ensure config.collections   |
                  | return parsed config        |
                  +-----------------------------+
```

Project-local discovery used by the CLI before default config:

```
+-------------------------------+
| no explicit --index           |
+-------------------------------+
          |
          v
+-------------------------------+
| findLocalConfigPath(cwd)      |
+-------------------------------+
          |
          v
+-------------------------------+
| check dir/.qmd/index.yaml     |
| check dir/.qmd/index.yml      |
+-------------------------------+
          |
          v
+-------------------------------+
| found?                        |
+-------------------------------+
     | yes              | no
     v                  v
+------------------+  +------------------------------+
| setConfigSource  |  | move to parent dir           |
| local YAML path  |  | until filesystem root        |
| set DB override  |  +------------------------------+
| to index.sqlite  |
+------------------+
```

## Config sync flow

```
+------------------------------+
| CLI/store loads config       |
| loadConfig()                 |
+------------------------------+
          |
          v
+------------------------------+
| syncConfigToDb(db, config)   |
+------------------------------+
          |
          v
+------------------------------+
| JSON.stringify(config)       |
| SHA-256 hash                 |
+------------------------------+
          |
          v
+------------------------------+
| read store_config            |
| key = config_hash            |
+------------------------------+
          |
          v
+------------------------------+
| hash matches?                |
+------------------------------+
     | yes              | no
     v                  v
+----------------+  +------------------------------+
| return early   |  | for each config collection   |
+----------------+  | > upsert store_collections   |
                    +------------------------------+
                                  |
                                  v
                    +------------------------------+
                    | read DB collection names     |
                    | delete names missing config  |
                    +------------------------------+
                                  |
                                  v
                    +------------------------------+
                    | sync global_context          |
                    | into store_config            |
                    +------------------------------+
                                  |
                                  v
                    +------------------------------+
                    | write store_config           |
                    | key = config_hash            |
                    +------------------------------+
```

## Collections + contexts data model

```
+--------------------------------------------------+
| CollectionConfig YAML                            |
+--------------------------------------------------+
| global_context: "...applies to all..."           |
| editor_uri: "..."                                |
| editor_uri_template: "..."                       |
| models:                                          |
|   embed: "..."                                   |
|   rerank: "..."                                  |
|   generate: "..."                                |
| collections:                                     |
|   notes:                                         |
|     path: "/abs/path/to/notes"                   |
|     pattern: "**/*.md"                           |
|     ignore:                                      |
|       - "Archive/**"                             |
|     update: "git pull"                           |
|     includeByDefault: false                      |
|     context:                                     |
|       "/": "root collection context"             |
|       "/2024": "more specific context"           |
|       "/2024/board": "most specific context"     |
|   work:                                          |
|     path: "/abs/path/to/work"                    |
|     pattern: "**/*.md"                           |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
| SQLite mirror                                    |
+--------------------------------------------------+
| store_collections                                |
|   name: notes                                    |
|   path: /abs/path/to/notes                       |
|   pattern: **/*.md                               |
|   ignore_patterns: JSON string or NULL           |
|   include_by_default: 0 or 1                     |
|   update_command: string or NULL                 |
|   context: JSON ContextMap or NULL               |
|                                                  |
| store_config                                     |
|   global_context: string or absent               |
|   config_hash: sha256(JSON config)               |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
| Context lookup                                   |
+--------------------------------------------------+
| input: collectionName + filePath                 |
| normalize to leading slash                       |
| collect matching context prefixes                |
| sort by longest prefix                           |
| return most specific match                       |
| otherwise return global_context                  |
+--------------------------------------------------+
```
