# PostgreSQL Database Administration — Course Outline

**Day 3 — 5th August 2026**

---

## Teams Involved With a Database

Two teams typically handle a database:

1. **Developers** — build the code and manage the complete set of database objects.
2. **DBA Team** — maintains the database (installation, configuration, user access management, backup management, tuning).

- As a developer, knowing how to connect to the database is generally sufficient.
- As a DBA, simply knowing how to connect is **not** enough — a deeper understanding of the internals is required.

---

## What Is "Architecture"?

Any task's complete flow — from start to finish — is referred to as its **architecture**.

With respect to databases, the complete backend flow — from the start of query execution until the result set is returned — is what we call **architecture**.

Based on query execution, memory utilization, and the number of processes involved, architecture is divided into **3 layers**:

1. General Architecture
2. Memory-Based Architecture
3. Process-Based Architecture

---

## 1. General Architecture

This is generic and common across most databases — only terminology differs between systems, but the overall flow is similar.

*Definition: From the start of query execution to the point the result set is returned, the complete backend flow is called the general architecture.*

**Example:**
```sql
SELECT * FROM emp;
```
...and the output is returned.

### Request Flow

```
User
 → Executes queries via a client (GUI / CLI / app server)
   → Query reaches the PostgreSQL server operating system
     → The postmaster process receives it
       → Performs host-based authentication by reading pg_hba.conf
         → If authentication succeeds, a new PostgreSQL backend process is created for the query
           → The backend process performs its execution steps:
              (Syntax check → Security check → Resolver → Parser → Optimizer → Dispatcher)
```

*Note: "plastic steps" in the original notes has been corrected to "execution steps" — likely a transcription/autocorrect error.*

### Key Components

| Component | Role |
|---|---|
| **postmaster process** | Handles host-based authentication; manages service start/stop/restart |
| **postgres backend process** | Performs the actual work of executing the query |
| **`pg_hba.conf`** | Controls host-based authentication |
| **`postgresql.conf`** | Controls and manages overall resource configuration |

---

## 2. Memory-Based Architecture

PostgreSQL's architecture is deeply tied to the operating system, so **memory** plays a key role in performance.

Memory is categorized as follows, with respect to PostgreSQL operations:

1. `shared_buffers`
   - 1a. `wal_buffers`
   - 1b. `clog_buffers` *(commonly referred to as commit log / `pg_xact` buffers)*
2. `work_mem`
3. `temp_buffers`
4. `maintenance_work_mem`
5. `autovacuum_work_mem`
6. `effective_cache_size`

### Database Internal vs. External Operations

- **Internal operations:** caching, authentication, authorization, data processing, transaction log management, checkpointing, archive file generation, recovery.
- **External operations:** DML (`SELECT`, `UPDATE`, `DELETE`, `INSERT`) and DDL (`DROP`, `RENAME`, `ALTER`, `CREATE`, `TRUNCATE`).

*Note: The performance of external operations is heavily dependent on how well internal operations are configured and tuned.*

**Example server for this discussion: PostgreSQL server with 16 GB of RAM.**

---

### 1. `shared_buffers`

- A piece of RAM used **exclusively** for PostgreSQL's internal operations.
- **Default value:** 128 MB.
- **Recommended value:** roughly 25% of total RAM (don't leave it at the default in production).
  - Example: 25% of 16 GB = **4 GB** assigned to `shared_buffers`.
  - Remaining RAM: **12 GB**.
- If the assigned `shared_buffers` isn't sufficient for internal processing, PostgreSQL will **not** silently exceed the configured limit — instead, it raises a "shared memory limit exceeded" type error.
- `shared_buffers` can be increased further (e.g., to ~30%) *only* if you're confident the remaining RAM is genuinely unused by other workloads. Increasing it blindly risks server instability or crashes.
- Configured in `postgresql.conf`, located in the **data directory**.
- This is a **static parameter** — changing it requires a **PostgreSQL service restart** to take effect.
- Scope: **instance level**.
- `wal_buffers` and `clog_buffers` don't need to be configured explicitly — they are automatically carved out of `shared_buffers`.

**1a. `wal_buffers`**
- Used by every transaction; internally consumed from `shared_buffers`.
- Default: **1/32 of `shared_buffers`**.
- Example: with `shared_buffers` = 4 GB, `wal_buffers` ≈ 128 MB, assigned automatically.

**1b. `clog_buffers`**
- Tracks only **committed transaction** details (commit log).
- Default: **1/64 of `shared_buffers`**.
- Example: ≈ 64 MB, assigned automatically.

Both `wal_buffers` and `clog_buffers` are drawn from `shared_buffers`.

*Note: `shared_buffers` can be set as low as 128 KB — this flexibility is part of why PostgreSQL is often described as lightweight and portable.*

---

### 2. `work_mem`

- Drawn from the **remaining RAM** (in our example, 12 GB).
- Used **only for external operations** — specifically for **hashing** and **sorting** during query execution.
  - **Hashing:** a hashing algorithm generates the address (pointer) of where a value sits on disk.
  - **Sorting:** used when `SELECT` results need to be returned in ascending/descending order.
- Consumed **per query**.
- **Default:** 4 MB, drawn from the remaining RAM.
- Assigned **per session** — e.g., 10 connections × 4 MB = 40 MB consumed; 100 connections × 4 MB = 400 MB consumed.

**Caution with large `work_mem` values:**
- Suppose `work_mem` is set to 1 GB, and `max_connections = 100` (i.e., a maximum of 100 concurrent connections; a 101st connection attempt triggers a "max connections limit exceeded" error).
- If all 100 connections are active simultaneously, total `work_mem` demand = 100 × 1 GB = **100 GB** — but only 12 GB of remaining RAM is available.
- In practice, only the first ~11–12 connections would get sufficient `work_mem`; the 12th connection onward would start hitting **out-of-memory** errors.
- **Recommendation:** keep `work_mem` modest — typically **4 MB / 8 MB / 12 MB / 16 MB** at most. Raising it at the **cluster (instance-wide) level** is generally not advisable.

**Workload-based scoping:** PostgreSQL allows `work_mem` to be set at different levels for workload prioritization:
- **Database-specific:** e.g., set 10 MB for one database — all connections to that database get 10 MB `work_mem`.
- **User-specific:** e.g., set 10 MB for one user — all connections from that user get 10 MB `work_mem`.
- **Session-specific:** a larger `work_mem` can be set for an individual session as needed.

- The default is set in `postgresql.conf` (a **static parameter**).
- Non-cluster-level scopes (database, user, session) can be adjusted using the `ALTER` command.

---

### 3. `temp_buffers`

- Also drawn from the remaining RAM.
- Consumed when performing **TEMP operations** or when a query generates temporary result sets — e.g., creating temp objects (temp tables, temp views).
- **Default value:** 8 MB.
- Example: 10 connections generating temp results/objects → 10 × 8 MB consumed.

**Subqueries and temp buffers:**
- Subqueries (a query inside another query) generate temporary result sets.

```sql
SELECT * FROM emp
WHERE eno IN (
  SELECT eno FROM dept
  WHERE dno IN (
    SELECT dno FROM hr
  )
);
```

- The innermost (child) query always executes first; its result set is then passed as input to the parent query.
- In the gap between the child query's execution and the parent query's use of that result, the intermediate result set is held in `temp_buffers`.

- `temp_buffers` is a **dynamic parameter** — it can be changed at any point without a restart.
- Configured in `postgresql.conf`, located in the data directory.
- **Recommended max:** 16–32 MB.

---

### 4. `maintenance_work_mem`

- Also drawn from the remaining RAM.
- Consumed during PostgreSQL **maintenance activities**: `ANALYZE`, `VACUUM`, `VACUUM FULL`, `REINDEX`.
- These activities are run to keep the database safe, consistent, and performant.
- **Default value:** 64 MB, consumed **per session**.

**Temporarily increasing it for a maintenance run:**
```sql
SET maintenance_work_mem = '1GB';
ANALYZE; -- or VACUUM, etc.
```
- This is a **dynamic, session-level** change — once you exit the session and log back in, the value reverts to the configured default (64 MB).

**Illustrative example (16 GB server):**
- 25% reserved for `shared_buffers` → 4 GB.
- Check free memory with `free -m` → e.g., ~10–12 GB free.
- Temporarily bump `maintenance_work_mem` to 4–6 GB for a maintenance activity, then revert once done.

---

### General Notes on This Session
- "plastic steps" corrected to "execution steps" in the general architecture flow — likely a mishearing/autocorrect of a term like "the following steps."
- `clog_buffers` is noted here as commonly called the commit log (or `pg_xact`) buffer, for cross-reference with official PostgreSQL documentation, which may use slightly different terminology across versions.
- Numeric examples (25% of RAM, 1/32 and 1/64 fractions, specific MB/GB figures) are retained as taught — useful as rules of thumb, though real-world tuning should also account for workload testing and official documentation recommendations.
