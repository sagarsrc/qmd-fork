## `src/db.ts`

### `openDatabase(path: string): Database`

Open a SQLite database using the runtime-selected driver (`bun:sqlite` on Bun, `better-sqlite3` on Node).

### `loadSqliteVec(db: Database): void`

Load the sqlite-vec extension into an already-open database, or throw with platform-specific guidance if sqlite-vec is unavailable.

## `src/collections.ts`

### `setConfigSource(source?: { configPath?: string; config?: CollectionConfig }): void`

Select default file config, a custom YAML file path, or an inline in-memory SDK config.

### `setConfigIndexName(name: string): void`

Set the active index name used to derive the default config filename.

### `findLocalConfigPath(startDir: string = process.cwd()): string | undefined`

Walk upward from a start directory and return the first `.qmd/index.yaml` or `.qmd/index.yml` found.

### `getLocalDbPath(configPath: string): string`

Return the sibling `.qmd/index.sqlite` path for a project-local config file.

### `loadConfig(): CollectionConfig`

Load the active config from inline memory or YAML, returning an empty `{ collections: {} }` config when the file does not exist.

### `saveConfig(config: CollectionConfig): void`

Persist config to the active inline source or YAML file.

### `getCollection(name: string): NamedCollection | null`

Return one named collection from config, or null if it is not configured.

### `listCollections(): NamedCollection[]`

Return all configured collections with their names included.

### `getDefaultCollections(): NamedCollection[]`

Return collections whose `includeByDefault` field is not explicitly false.

### `getDefaultCollectionNames(): string[]`

Return only the names of collections included by default.

### `updateCollectionSettings(name: string, settings: { update?: string | null; includeByDefault?: boolean }): boolean`

Update optional collection settings, preserving default behavior by deleting fields where appropriate.

### `addCollection(name: string, path: string, pattern: string = "**/*.md"): void`

Add or replace a collection definition while preserving any existing context map for that name.

### `removeCollection(name: string): boolean`

Delete a collection from config if present.

### `renameCollection(oldName: string, newName: string): boolean`

Rename a configured collection, throwing if the destination name already exists.

### `getGlobalContext(): string | undefined`

Return the global context text from config, if present.

### `setGlobalContext(context: string | undefined): void`

Set or clear the global context and persist the config.

### `getContexts(collectionName: string): ContextMap | undefined`

Return the path-prefix context map for a collection, if present.

### `addContext(collectionName: string, pathPrefix: string, contextText: string): boolean`

Add or update one context entry for a path prefix inside a collection.

### `removeContext(collectionName: string, pathPrefix: string): boolean`

Remove one context entry and delete the collection context object when it becomes empty.

### `listAllContexts(): Array<{ collection: string; path: string; context: string }>`

Return global and per-collection contexts as one flattened list.

### `findContextForPath(collectionName: string, filePath: string): string | undefined`

Return the most specific matching collection context by longest prefix, falling back to global context.

### `getConfigPath(): string`

Return the active config path for diagnostics, or `<inline>` in inline config mode.

### `configExists(): boolean`

Report whether the active file config exists, or true for inline config mode.

### `isValidCollectionName(name: string): boolean`

Validate that a collection name contains only letters, numbers, underscores, and hyphens.

## Exported non-function API

`db.ts` also exports `isBun`, `SQLiteValue`, `SQLiteParams`, `Database`, and `Statement`.

`collections.ts` also exports `ContextMap`, `Collection`, `ModelsConfig`, `CollectionConfig`, and `NamedCollection`.
