# PostgreSQL Database Administration — Course Outline

**Day 5**

---

## Automating Maintenance: Autovacuum

The operations covered earlier — `ANALYZE`, `VACUUM`, `VACUUM FULL`, `REINDEX` — are typically run **manually**.

PostgreSQL also provides a way to **automate** two of these: `VACUUM` and `ANALYZE`.

- **`VACUUM`** — removes dead tuples, freeing up storage for new records.
- **`ANALYZE`** — updates the query planner's statistics (information about table contents).

PostgreSQL runs a background process called the **autovacuum launcher process** to handle this automatically. It periodically checks tables against configured thresholds and runs `VACUUM`/`ANALYZE` on the tables that need it.

### `autovacuum_work_mem`
- Settings are configured in `postgresql.conf`.
- **Default value:** `-1` *(meaning it falls back to using the value configured for `maintenance_work_mem`)*.
- If the default isn't desired, a specific value can be set.
- Drawn from the remaining RAM (outside `shared_buffers`).
- Scope: **session-specific**.

---

## 6. `effective_cache_size`

This is best understood as a **"hint" parameter** rather than a real memory allocation — PostgreSQL doesn't actually reserve this memory.

- For new incoming connections, it reports **free memory + `shared_buffers`** as the "available memory" estimate the query planner can consider.
- When the planner sees that more memory is effectively available, it becomes more likely to favor **index-based access** over a full table scan — boosting performance for queries that use indexes.
- In short: `effective_cache_size` improves the *effectiveness* of indexes by helping the query planner make better decisions.

- **Default value:** 128 MB.
- **Recommended value:** ~75% of total RAM.
- Configured in `postgresql.conf`.

**Important distinction:**
- 25% of RAM → allocated to `shared_buffers` (actual dedicated memory).
- `work_mem`, `temp_buffers`, `maintenance_work_mem`, and `autovacuum_work_mem` → all drawn from the *remaining* RAM (actual consumption, on demand).
- `effective_cache_size` → **not actual consumption** — it's simply a planner-facing estimate of `shared_buffers` + free RAM.

---

## Memory Parameter Summary

| Parameter | Purpose |
|---|---|
| `shared_buffers` | ~25% of total RAM. Used for internal operations (authentication, authorization, data processing, transaction log management, checkpointing, data flush). |
| `work_mem` | Default 4 MB. Used for external operations (hashing/sorting); increased to speed up DDL/DML operations. |
| `temp_buffers` | Used for temp operations (temp tables/views, subquery intermediate results). |
| `maintenance_work_mem` | Used for `ANALYZE` / `VACUUM` / `VACUUM FULL` / `REINDEX`. |
| `autovacuum_work_mem` | Increased to speed up automated vacuum/analyze operations. |
| `effective_cache_size` | Increased to improve performance for index-heavy queries (planner hint, not real allocation). |

All memory parameters above are configured in `postgresql.conf`.
- Some are **static** (require a full PostgreSQL server **restart**).
- Some are **dynamic** (a simple **reload** of the PostgreSQL service is enough).

---

## Transaction Log Internals

- A `SELECT` is a **read-only** operation and is **not tracked** in transaction logs.
- **Real transactions** are ones that make changes to data or structure.
- **Committed (completed) transaction:** query execution has fully finished.
- **Uncommitted transaction:** still in progress / not yet finished.

### Flow of a Transaction

1. All transactions, by default, are first written to **`wal_buffers`**.
2. Once execution completes, the transaction is marked as completed — this status is stored in **`clog_buffers`**.
3. Committed transactions are then **flushed from `wal_buffers` to disk** (permanent storage).

### On-Disk WAL Structure

- The **data directory** contains a subdirectory called **`pg_wal`**.
- Inside `pg_wal`, **WAL (Write-Ahead Log)** files are created.
- All committed/completed transactions are recorded in WAL files.
- **Default WAL file size:** 16 MB.
- A **new WAL file** is created for every 16 MB of log data generated.
- Each WAL file name contains a hash-like value plus timestamp information, so WAL file names are always **unique**.

### WAL Retention

- **Default `pg_wal` directory capacity:** ~1 GB (1024 MB).
- **Default number of WAL files retained:** **64** (64 × 16 MB ≈ 1 GB).
- When the **65th** WAL file is generated, the **oldest** WAL file is removed/recycled.
- Removing old WAL files has **no impact** on the currently running service, since those transactions have already completed and been applied.
- **However**, if you have a **replication setup**, or need old WAL files for **Point-in-Time Recovery (PITR)**, those old files become important — and losing them prematurely is a problem.

**Solutions to retain WAL files longer:**

**A. Enable Archive Mode**
Continuously copies WAL files to a separate archive directory before they're recycled.

**B. Increase `pg_wal` capacity**
- `wal_keep_size` — e.g., 1024 MB / 2048 MB / 3072 MB *(corrected from "3096mb" — likely meant 3072 MB, i.e., 3 GB, since WAL-related sizing is typically in powers of 2)*. Introduced in **PostgreSQL 13**.
- `wal_keep_segments` — e.g., 64 (number of files). Used up through **PostgreSQL 12**; replaced by `wal_keep_size` from version 13 onward.

### WAL File Size

- WAL file size can be increased beyond the 16 MB default, but **only at initialization time** — using the `wal_segment_size` parameter (e.g., `wal_segment_size = 1GB`).
- This is generally **not recommended** and can only be set **once**, during initial cluster setup — it can't be changed afterward without reinitializing.

### Naming History

- The `pg_wal` directory name was introduced in **PostgreSQL 9.6**; before that, it was called **`pg_xlog`**.
- WAL files are **not human-readable**. They can only be inspected using the dedicated utility **`pg_waldump`**.

---

### General Notes on This Session
- `autovacuum_work_mem` defaulting to `-1` means it inherits from `maintenance_work_mem` rather than having its own independent default — worth double-checking against the specific PostgreSQL version being taught, since defaults can shift slightly across releases.
