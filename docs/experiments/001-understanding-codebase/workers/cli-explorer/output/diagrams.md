# CLI Diagrams

## CLI Command Dispatch Table

```
+-------------------+-------------------------------+
| argv command      | handler/function               |
+-------------------+-------------------------------+
| --help / empty    | showHelp                       |
| --version         | showVersion                    |
| --skill           | showSkill                      |
| context add       | contextAdd                     |
| context list      | contextList                    |
| context rm/remove | contextRemove                  |
| get               | getDocument                    |
| multi-get         | multiGet                       |
| ls                | listFiles                      |
| collection list   | collectionList                 |
| collection add    | collectionAdd > indexFiles     |
| collection rm     | collectionRemove               |
| collection rename | collectionRename               |
| collection update | updateCollectionSettings       |
| collection include| updateCollectionSettings       |
| collection show   | getCollection + print          |
| init              | initLocalIndex                 |
| status            | showStatus                     |
| doctor            | showDoctor                     |
| update            | updateCollections              |
| embed             | vectorIndex                    |
| pull              | pullModels                     |
| search            | search                         |
| vsearch           | vectorSearch                   |
| vector-search     | vectorSearch                   |
| query             | querySearch                    |
| deep-search       | querySearch                    |
| bench             | runBenchmark                   |
| mcp               | startMcpServer/startMcpHttp... |
| mcp stop          | PID kill/cleanup               |
| skills            | runSkillsCommand               |
| skill show        | showSkill                      |
| skill install     | installSkill                   |
| cleanup           | cache/vector/doc cleanup       |
+-------------------+-------------------------------+
```

## Output Format Pipeline

```
+----------------+
| Store results  |
| SearchResult[] |
+-------+--------+
        |
        v
+--------------------+
| CLI decoration     |
| context            |
| docid              |
| displayPath        |
| full-path policy   |
| chunk/snippet info |
+-------+------------+
        |
        v
+-----------------------------+
| outputResults()             |
| or formatter.ts dispatcher  |
+---+-----+-----+-----+---+---+
    |     |     |     |   |
    v     v     v     v   v
+------+ +-----+ +----+ +-----+ +-------+
| cli  | |json | |csv | | md  | | xml   |
| text | | []  | |hdr | | --- | | <file>|
+--+---+ +--+--+ +--+-+ +--+--+ +---+---+
   |        |       |      |        |
   +--------+-------+------+--------+
                    |
                    v
             +-------------+
             | stdout      |
             +-------------+
```

## Lifecycle Diagram

```
+-------------+
| parseCLI()  |
+------+------+ 
       |
       v
+------------------------+
| command dispatch       |
+-----------+------------+
            |
            v
+------------------------+
| getStore()             |
| - createStore          |
| - loadConfig           |
| - syncConfigToDb       |
| - setDefaultLlamaCpp   |
+-----------+------------+
            |
            v
+------------------------+
| getDb()                |
| active SQLite handle   |
+-----------+------------+
            |
            v
+------------------------+
| command work           |
| store/collections/LLM  |
+-----------+------------+
            |
            v
+------------------------+
| print output           |
| stdout/stderr          |
+-----------+------------+
            |
            v
+------------------------+
| closeDb()              |
| store.close(); null    |
+-----------+------------+
            |
            v
+------------------------+
| finishSuccessful...    |
| flush stdout           |
| dispose llama.cpp      |
| flush stderr           |
| process.exitCode = 0   |
+------------------------+
```
