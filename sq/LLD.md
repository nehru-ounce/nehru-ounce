# docs/LLD_Backend.md

## SQL Developer-like Backend — LLD (Spring Boot)

### 1. Base Package Structure (Final)

Root package: **`com.gc.smarthub.sql`**

```text
com.gc.smarthub.sql
  ├─ config
  │   ├─ AppConfig.java
  │   ├─ CacheConfig.java
  │   └─ SecurityConfig.java
  ├─ security
  │   ├─ auth
  │   ├─ jwt
  │   └─ rbac
  ├─ common
  │   ├─ dto
  │   ├─ errors
  │   ├─ util
  │   └─ constants
  ├─ connection
  │   ├─ api
  │   │   └─ ConnectionController.java
  │   ├─ service
  │   │   ├─ ConnectionService.java
  │   │   ├─ ConnectionValidator.java
  │   │   ├─ PropertyService.java
  │   │   └─ DataSourceRegistry.java
  │   ├─ persistence
  │   │   ├─ ConnectionRepository.java
  │   │   └─ ConnectionPropertyRepository.java
  │   └─ model
  │       ├─ ConnectionEntity.java
  │       ├─ ConnectionPropertyEntity.java
  │       └─ DbType.java
  ├─ crypto
  │   ├─ CryptoService.java
  │   ├─ AesGcmCryptoService.java
  │   └─ KeyProvider.java
  ├─ dialect
  │   ├─ DbDialect.java
  │   ├─ DialectResolver.java
  │   ├─ oracle
  │   ├─ postgres
  │   ├─ mysql
  │   └─ sqlserver
  ├─ metadata
  │   ├─ api
  │   │   └─ MetadataController.java
  │   ├─ service
  │   │   └─ MetadataService.java
  │   └─ cache
  │       └─ MetadataCache.java
  ├─ query
  │   ├─ api
  │   │   └─ QueryController.java
  │   ├─ model
  │   │   ├─ ExecuteRequest.java
  │   │   ├─ QueryResultDto.java
  │   │   └─ ColumnMetaDto.java
  │   ├─ execution
  │   │   ├─ ExecutionRegistry.java
  │   │   ├─ ExecutionHandle.java
  │   │   └─ SqlSplitter.java
  │   ├─ transaction
  │   │   ├─ WorksheetContext.java
  │   │   └─ WorksheetContextStore.java
  │   └─ service
  │       └─ QueryService.java
  ├─ history
  │   ├─ api
  │   ├─ model
  │   ├─ persistence
  │   └─ service
  └─ audit
      ├─ api
      ├─ model
      ├─ persistence
      └─ service
```

---

### 2. Database Schema (Final)

## 2.1 `connections`

Purpose: stable identity, ownership, db type.

**Columns**

* `id` (UUID/BIGINT, PK)
* `name` (varchar, not null)
* `db_type` (varchar, not null)
* `created_by` (varchar/uuid, not null)
* `team_id` (varchar/uuid, nullable)
* `is_active` (boolean, default true)
* `created_at`, `updated_at`

**Constraints**

* Unique: `(created_by, name)` OR `(team_id, name)` based on your org model

---

## 2.2 `connection_properties`

Purpose: store all connection configuration as key/value.

**Columns**

* `id` (UUID/BIGINT, PK)
* `connection_id` (FK → connections.id, not null)
* `prop_key` (varchar, not null)
* `prop_value` (text, nullable)
* `is_secret` (boolean, default false)
* `created_at`, `updated_at`

**Constraints**

* Unique: `(connection_id, prop_key)`

**Important**

* `password` is stored as encrypted blob (Base64 encoded ciphertext) in `prop_value`
* `password_iv` stored as another property key OR pack both into a single value (recommended pack)

✅ Recommended approach: store a single packed value:

* `password_enc = base64(iv) + ":" + base64(ciphertext) + ":" + keyVersion`

Example:

* prop_key = `password_enc`
* prop_value = `b64iv:b64cipher:v1`
* is_secret = true

---

### 3. Standard Property Keys

Common:

* `host`, `port`, `username`
* `database` (MySQL/Postgres/SQLServer)
* `schema` (optional default schema)
* `jdbc_url_override` (optional)
* `password_enc` (**required for all**)

Oracle-specific:

* `service_name` (recommended)
* `sid` (optional)

SQL Server-specific:

* `instance_name` (optional)
* `encrypt` (true/false)
* `trust_server_certificate` (true/false)

MySQL/MariaDB:

* `use_ssl` (true/false)
* `allow_public_key_retrieval` (true/false)

Postgres:

* `current_schema` (optional)
* `application_name` (optional)

---

### 4. Encryption Design (Password at Rest)

#### 4.1 CryptoService Contract

* `encrypt(plainText) -> packedString`
* `decrypt(packedString) -> plainText`

#### 4.2 Algorithm

* AES-GCM
* 12-byte IV recommended
* Store packed format: `iv:cipher:keyVersion`

#### 4.3 KeyProvider & Rotation

* `KeyProvider.getKey(keyVersion)` returns SecretKey
* Active key version configured via env:

  * `APP_CRYPTO_ACTIVE_KEY_VERSION=v1`
  * `APP_CRYPTO_KEYS_V1=<base64key>`
  * `APP_CRYPTO_KEYS_V2=<base64key>` (for future rotation)

Rotation strategy:

* New connections use active version
* Old values decrypted with their stored keyVersion
* Optional migration job later to re-encrypt

#### 4.4 API Masking

When returning properties:

* If `is_secret=true` → return `"***"`

---

### 5. Connection Runtime Build

#### 5.1 ConnectionConfig (runtime DTO)

Built from:

* ConnectionEntity + its properties map + decrypted password

Fields:

* dbType
* jdbcUrl
* username
* password (decrypted)
* driverClassName (optional)
* additional properties

#### 5.2 Validation

`ConnectionValidator` checks required keys per dbType.

Examples:

* MySQL/Postgres/SQLServer require: `host, port, database, username, password_enc`
* Oracle requires: `host, port, username, password_enc` + (`service_name` OR `sid`)

---

### 6. DataSourceRegistry (Hikari Pools)

* One pool per `connectionId`
* Created lazily on first use/test/query/metadata

Methods:

* `DataSource getOrCreate(connectionId)`
* `void evict(connectionId)` (close and remove)
* `Connection getConnection(connectionId)`

Pool defaults (configurable):

* maxPoolSize=10
* connectionTimeout=10s
* validationTimeout=5s
* idleTimeout=3m
* maxLifetime=30m

---

### 7. Dialect Layer (Multi-DB)

`DbDialect` responsibilities:

* `buildJdbcUrl(properties)`
* `applyPagination(sql, limit, offset)`
* optional: `supportsExplain()`, `buildExplainSql(sql)`
* `sqlSplitStrategy()` (script execution)
* `quoteIdentifier(name)`

Pagination:

* Postgres/MySQL/MariaDB: `LIMIT x OFFSET y`
* SQL Server: `OFFSET y ROWS FETCH NEXT x ROWS ONLY` (needs ORDER BY policy)
* Oracle: 12c+ `OFFSET/FETCH` (recommended; older Oracle optional)

---

### 8. Metadata Service

Uses JDBC `DatabaseMetaData`:

* schemas: `getSchemas()`
* tables: `getTables()`
* columns: `getColumns()`
* PK/FK: `getPrimaryKeys()`, `getImportedKeys()`
* indexes: `getIndexInfo()`

Caching:

* TTL default 10 minutes
* Key includes: connectionId + schema + object
* Invalidate on connection update/delete

---

### 9. Query Engine

#### 9.1 WorksheetContext (Transaction Pinning)

`worksheetId` represents a UI tab.

Fields:

* worksheetId, userId, connectionId
* autoCommit flag
* pinnedConnectionLease (only when autocommit OFF)
* lastAccess

Rules:

* autocommit ON: borrow connection per query
* autocommit OFF: pin one connection until commit/rollback/timeout

Cleanup:

* background sweeper releases pinned connections for idle worksheets

---

#### 9.2 ExecutionRegistry (Cancel Support)

Keeps RUNNING statements:

* executionId → ExecutionHandle
  ExecutionHandle contains:
* statement reference
* connection lease reference (if non-pinned)
* timings, status

Cancel:

* find statement → call `statement.cancel()`

---

#### 9.3 Execution Modes

**SINGLE**

* execute one statement

**SCRIPT**

* split into statements using SqlSplitter
* execute sequentially
* return array of statement results

---

#### 9.4 Result Handling

* Preview rows: default 500 (config)
* Return:

  * `columns[]`
  * `rows[]` (list of list)
  * or `updateCount`

Pagination endpoint re-executes original SQL using dialect paging.

Limits:

* page limit max 1000
* query timeout default 60s

---

### 10. Security (LLD)

RBAC:

* ADMIN: all connections + audits
* DEVELOPER: owns/has access connections + full query rights
* READONLY: SELECT only (policy enforced)

SQL policy (initial heuristic):

* allow starting keywords: `select`, `with`, `show`, `describe`, `explain` (configurable)
* block others

Audit:

* store event type + connectionId + userId + duration + status
* store SQL as:

  * full SQL or hashed+preview (config via `app.security.storeFullSql`)

---

### 11. Error Response Standard

```json
{
  "timestamp": "2026-05-27T12:00:00Z",
  "errorCode": "DB_QUERY_FAILED",
  "message": "ORA-00942: table or view does not exist",
  "details": { "executionId": "exec-1" }
}
```

---

### 12. Config Keys (application.yml)

* `app.query.timeoutSeconds=60`
* `app.query.previewMaxRows=500`
* `app.query.pageMaxLimit=1000`
* `app.worksheet.idleTtlMinutes=30`
* `app.metadata.cacheTtlMinutes=10`
* `app.pool.maxPerConnection=10`
* `app.security.storeFullSql=false`
* `app.crypto.activeKeyVersion=v1`
