# docs/HLD_Backend.md

## SQL Developer-like Backend — HLD (Spring Boot)

### 1. Purpose

Build a backend that works like a SQL client (SQL Developer style):

* Manage multiple DB connections (save/test/update/delete)
* Browse metadata (schemas → tables → columns → keys/indexes)
* Execute queries/scripts
* Show results (preview rows) + pagination
* Manage transactions (autocommit toggle, commit, rollback) per worksheet/tab
* Cancel running queries
* Store history + audit

✅ Supports multiple database types via **JDBC + Dialect layer**:
Oracle, PostgreSQL, MySQL, MariaDB, SQL Server (extensible).

---

### 2. Scope

#### MVP (Backend)

1. Connection CRUD + Test
2. Metadata explorer (schemas/tables/columns + basic keys/indexes)
3. Query execution (single + script)
4. Preview rows + pagination
5. Worksheet transaction handling (autocommit OFF pins a connection)
6. Cancel query
7. History + audit logs

#### Phase-2 (Later)

* Explain plan
* Export as job (CSV/XLSX/JSON)
* Advanced metadata (procedures/packages/triggers/grants)
* Background jobs/worker

---

### 3. Non-Functional Requirements

**Security**

* Password stored encrypted at rest (AES-GCM)
* RBAC: ADMIN / DEVELOPER / READONLY
* Audit for: connection ops, query execute/cancel, commit/rollback

**Reliability**

* Query timeout enforced
* Proper cleanup for statements/resultsets/connections
* Safe cancellation (best-effort across JDBC drivers)

**Performance**

* Pool per connection definition (HikariCP)
* Metadata caching (TTL)
* Preview rows limited + paging supported

**Scalability**

* Stateless nodes possible
* For cluster cancellation routing: Redis optional (future)

---

### 4. High-Level Architecture

**Spring Boot Backend**

* Connection Management (two-table model)
* DataSource Registry (Hikari pools per saved connection)
* Dialect Resolver (DB-specific SQL behaviors)
* Metadata Service (DatabaseMetaData + overrides)
* Query Engine

  * WorksheetContext store (transaction pinning)
  * ExecutionRegistry (for cancellation)
  * Pagination via dialect query rewrite
* History + Audit

**App Metadata DB**

* Stores users (optional), connections, properties, history, audit

**Target DBs**

* Accessed via JDBC drivers (Oracle/MySQL/Postgres/SQL Server/etc.)

---

### 5. Connection Storage Model (Two Tables)

#### 5.1 `connections` (stable identity + access control)

* id, name, db_type, created_by, team_id(optional), is_active, timestamps

#### 5.2 `connection_properties` (key/value config)

All connection settings are stored here:

* host, port, db/service name, username
* encrypted password (**required**)
* vendor-specific options (SSL, instance_name, etc.)

**Benefits**

* Add new DB type / property without schema changes
* Keeps `connections` stable and clean

---

### 6. Password Encryption (Final Decision)

Passwords are stored **encrypted in `connection_properties`**.

**Algorithm**

* AES-GCM (authenticated encryption)
* Store:

  * ciphertext (Base64)
  * iv/nonce (Base64)
  * key version (for rotation)

**Key Management**

* Master key injected via environment/secret in deployment
* Support key rotation by `key_version` and multiple active keys

**API Masking**

* Never return password value in any response
* All secret properties masked: `"***"`

---

### 7. Multi-DB Support Strategy

Dialect layer controls:

* JDBC URL creation
* Pagination syntax
* SQL script splitting rules
* Identifier quoting rules
* Explain-plan support (later)

Start with:

* OracleDialect
* PostgresDialect
* MySqlDialect (covers MariaDB with minor tweaks)
* SqlServerDialect

---

### 8. Core Backend Flows (HLD)

#### 8.1 Create / Update Connection

1. Validate required properties based on dbType
2. Encrypt password and store in `connection_properties`
3. Save/Upsert properties
4. Evict pool if updated

#### 8.2 Test Connection

1. Resolve properties → build JDBC URL via dialect
2. Decrypt password
3. Attempt connection with small timeouts
4. Return DB product/version if success

#### 8.3 Metadata Browse

1. Borrow connection from pool
2. Fetch via `DatabaseMetaData`
3. Cache responses with TTL
4. Return DTOs

#### 8.4 Execute SQL

1. Check access + role policy
2. Acquire connection:

   * autocommit ON → short-lived
   * autocommit OFF → pinned to worksheet
3. Execute with timeout + preview limit
4. Return:

   * columns + preview rows OR updateCount
5. Record history + audit
6. Enable cancellation using ExecutionRegistry
7. Pagination endpoint re-executes query with dialect paging

#### 8.5 Transaction Control

* autocommit OFF pins one connection to worksheet
* commit/rollback uses pinned connection
* autocommit ON releases pinned connection

---

### 9. API Summary (HLD)

**Connections**

* `POST /api/connections`
* `GET /api/connections`
* `GET /api/connections/{id}`
* `PUT /api/connections/{id}`
* `DELETE /api/connections/{id}`
* `POST /api/connections/{id}/test`

**Metadata**

* `GET /api/metadata/{connectionId}/schemas`
* `GET /api/metadata/{connectionId}/schemas/{schema}/tables`
* `GET /api/metadata/{connectionId}/schemas/{schema}/tables/{table}`

**Query**

* `POST /api/query/execute`
* `GET /api/query/executions/{executionId}/status`
* `POST /api/query/executions/{executionId}/cancel`
* `GET /api/query/executions/{executionId}/page?offset=&limit=`

**Transaction**

* `POST /api/query/worksheets/{worksheetId}/autocommit/on`
* `POST /api/query/worksheets/{worksheetId}/autocommit/off`
* `POST /api/query/worksheets/{worksheetId}/commit`
* `POST /api/query/worksheets/{worksheetId}/rollback`

**History/Audit**

* `GET /api/history?connectionId=&limit=`
* `GET /api/audit?from=&to=` (ADMIN)

---

### 10. Acceptance Criteria

* Can create + test Oracle/MySQL/Postgres/SQLServer connections
* Password is stored encrypted and never exposed
* Can browse schemas/tables/columns
* Can execute SELECT with preview + pagination
* Can execute DML/DDL and get update count
* Can disable autocommit, run multiple statements, commit/rollback
* Can cancel a long-running query
* History + audit records created properly
