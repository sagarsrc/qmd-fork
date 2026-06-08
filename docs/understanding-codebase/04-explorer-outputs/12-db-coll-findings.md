## DB Layer Overview — why a cross-runtime abstraction exists

`src/db.ts` provides a small compatibility wrapper over two SQLite drivers: Bun's built-in `bun:sqlite` and Node's `better-sqlite3`. QMD wants the rest of the codebase to call one `openDatabase()` function and use one `Database`/`Statement` shape, regardless of whether the runtime is Bun or Node.

The abstraction is deliberately narrow. It only exposes the operations QMD needs: execute SQL, prepare statements, load native extensions, run transactions, and close the database. This avoids leaking runtime-specific driver details into indexing, search, and store code.

## Bun vs Node Path — dynamic import of `bun:sqlite` vs `better-sqlite3`, API differences handled

Runtime detection is `export const isBun = "Bun" in globalThis`.

On Bun, the module builds the import string as `"bun:" + "sqlite"` before dynamic import. That prevents TypeScript and Node-oriented builds from trying to resolve `bun:sqlite` statically. The imported `Database` constructor becomes the private `_Database`.

On Node, `_Database` is assigned from `(await import("better-sqlite3")).default`. The code casts it into the common `DatabaseConstructor` type.

The API differences handled here are mostly import-path and sqlite-vec loading differences. Once `_Database` is set, `openDatabase(path)` can return `new _Database(path)` in both runtimes. The shared driver surface is captured by `Database` and `Statement`.

## macOS SQLite Extension Loading — setCustomSQLite for Homebrew SQLite, fallback behavior

Bun on macOS may use Apple's system SQLite, which is commonly built without extension loading support. Native extensions such as sqlite-vec need loadable extension support, so the Bun path attempts `BunDatabase.setCustomSQLite()` before any database instance is created.

The code tries two Homebrew locations:

- `/opt/homebrew/opt/sqlite/lib/libsqlite3.dylib` for Apple Silicon
- `/usr/local/opt/sqlite/lib/libsqlite3.dylib` for Intel Macs

Each attempt is wrapped in `try/catch`; failures are ignored and the loop continues. Afterward the code still tests sqlite-vec loading against an in-memory database, because `setCustomSQLite()` may not have succeeded. If the test fails, sqlite-vec is marked unavailable for this process, but the DB layer still opens SQLite databases and non-vector features can continue.

## sqlite-vec Loading — how the extension is loaded on both runtimes, failure modes

The sqlite-vec loader is stored as `_sqliteVecLoad`.

On Bun, the code imports `getLoadablePath()` from `sqlite-vec`, gets the native extension path, opens a `:memory:` database, calls `testDb.loadExtension(vecPath)`, and closes the test DB. If that succeeds, `_sqliteVecLoad` becomes a function that calls `db.loadExtension(vecPath)` on real databases.

On Node, the code imports the `sqlite-vec` package and sets `_sqliteVecLoad` to call `sqliteVec.load(db)`, using the package's better-sqlite3-compatible loader.

Failure modes:

- Bun/macOS may not have a loadable SQLite library available.
- sqlite-vec native files may be missing or incompatible.
- Extension loading can fail even if a SQLite database can be opened.

`loadSqliteVec(db)` throws if `_sqliteVecLoad` is null. On Bun/macOS the error suggests installing Homebrew SQLite or installing QMD with npm; otherwise it suggests checking sqlite-vec installation. In `store.ts`, database initialization catches this error, records sqlite-vec as unavailable, and allows FTS/BM25 behavior to continue.

## Database Interface — the common subset (exec, prepare, transaction, close)

`Database` includes:

- `exec(sql: string): void`
- `prepare(sql: string): Statement`
- `loadExtension(path: string): void`
- `transaction<T extends (...args: SQLiteValue[]) => unknown>(fn: T): T`
- `close(): void`

`Statement` includes:

- `run(...params): { changes; lastInsertRowid }`
- `get<T>(...params): T | undefined`
- `all<T>(...params): T[]`

This is enough for schema creation, reads, writes, extension loading, and transactional mutations without committing QMD to one runtime's full SQLite API.

## Collections Layer Overview — YAML as source of truth, why it mirrors into SQLite

`src/collections.ts` owns the external collection configuration model. It reads, writes, and mutates a YAML-backed `CollectionConfig` containing `global_context`, editor URI settings, collection definitions, and optional model overrides.

The collection config mirrors into SQLite so store and query code can operate from the database. The `store_collections` table makes an index DB self-contained and gives search/indexing code a stable relational source for collection metadata. YAML remains the user-editable source of truth; SQLite receives a synchronized copy for runtime operations.

The actual mirror function, `syncConfigToDb`, is implemented in `src/store.ts`, not `src/collections.ts`. `collections.ts` provides the config types and YAML I/O that feed that sync.

## Config Resolution — XDG dirs, local `.qmd/index.yaml` discovery, index naming

The default config file is `{configDir}/{currentIndexName}.yml`, where `currentIndexName` defaults to `index`.

Config directory resolution:

- `QMD_CONFIG_DIR` wins when set.
- Otherwise `XDG_CONFIG_HOME/qmd` is used when `XDG_CONFIG_HOME` exists.
- Otherwise the fallback is `qmdHomedir()/.config/qmd`.

`setConfigIndexName(name)` changes the file stem. If the name contains `/`, it is resolved against `process.cwd()`, then path separators are replaced with underscores and a leading underscore is removed. This lets path-like index names become valid config filenames.

`findLocalConfigPath(startDir)` walks upward from `startDir`, looking for `.qmd/index.yaml` first and `.qmd/index.yml` second. The CLI uses this when no explicit `--index` is passed. When found, it sets that YAML file as the config source and pairs it with `.qmd/index.sqlite` via `getLocalDbPath(configPath)`.

`setConfigSource()` supports three modes:

- default file config
- custom file config path
- inline in-memory config for SDK usage

## Config Sync — syncConfigToDb: hash-based optimization, collections upsert/delete, global context

`syncConfigToDb(db, config)` in `store.ts` serializes the config with `JSON.stringify(config)`, hashes it with SHA-256, and compares it to `store_config.config_hash`. If the hash matches, sync returns early.

When the hash differs, external config wins:

- Each configured collection is upserted into `store_collections`.
- `ignore` and `context` maps are JSON-encoded into text columns.
- `includeByDefault === false` becomes `include_by_default = 0`; anything else becomes `1`.
- `update` is stored as `update_command`.
- Any DB collection whose name no longer appears in YAML is deleted.
- `global_context` is inserted/updated in `store_config`, or deleted if absent.
- The new config hash is saved under `store_config.config_hash`.

The CLI calls this on store creation and forces a re-sync after config mutations by deleting the stored hash first.

## Context System — per-path context within collections, global context, YAML persistence

Global context is stored at `config.global_context`. Per-collection context is stored as `collection.context`, a `Record<string, string>` keyed by path prefix.

`addContext(collectionName, pathPrefix, contextText)` creates the collection context map if needed, writes/overwrites the prefix entry, and saves YAML. `removeContext()` deletes a prefix and removes the whole `context` object when it becomes empty.

`listAllContexts()` emits global context as `{ collection: "*", path: "/", context }`, then flattens all collection context entries.

`findContextForPath(collectionName, filePath)` normalizes both file path and context prefixes to leading-slash paths, collects prefix matches, returns the longest prefix match, and falls back to global context when no collection context applies.

During DB sync, the collection context map is JSON-encoded into `store_collections.context`, while global context is stored separately in `store_config`.

## Gotchas / Design Decisions — why YAML not pure SQLite, config hash caching, inline config mode for SDK

YAML is the editable source of truth. It is friendlier for users, config files, project-local `.qmd` directories, and SDK-provided inline config than direct SQL mutation would be.

SQLite still stores a mirrored copy because runtime store/search code needs fast, self-contained metadata and because an index DB should carry collection settings with it.

The hash optimization is simple and effective, but it depends on the JSON serialization of the config object. Mutations that semantically preserve config but reorder keys may still change the hash; that only causes extra sync work, not incorrect state.

Inline config mode is important for SDK usage. `loadConfig()` returns the in-memory object directly, and `saveConfig()` replaces the in-memory object rather than touching disk. In inline mode, `getConfigPath()` returns `<inline>` and `configExists()` returns true.

One small code smell: `ensureConfigDir()` exists in `collections.ts` but is not used; `saveConfig()` creates the target directory directly from the active config path.
