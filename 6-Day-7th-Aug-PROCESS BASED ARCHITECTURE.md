# PostgreSQL Database Administration — Course Outline

**Day 6**

---

## Process-Based Architecture

We refer to people based on their work or profession. Similarly, at the operating system / database level, we refer to a "worker" as a **process**.

Based on the type of work performed, PostgreSQL processes are categorized as below.

### Mandatory Processes

**1. Postmaster**
- This is the **only** process we start directly — it then starts several mandatory child processes.
- The main process responsible for starting, stopping, and restarting PostgreSQL server services.
- Performs host-based authentication.
- Referred to as the **supervisor process**.

**2. WAL Writer (Writer Process)**
- Flushes all committed transaction log entries from `wal_buffers` to the WAL file, immediately after commit.

**3. Background Writer (`bgwriter`)**
- Flushes dirty buffers from `shared_buffers` to data files in the `base` folder.
- Default frequency: **every 200 ms**.

**4. Logical Replication Launcher**
- Only relevant when a **logical replication** setup is in use.
- No need to create it manually as an extension — it's a built-in background process.

**5. Stats Collector Process**
- *Deprecated as of PostgreSQL 15* (its functionality was folded into the shared-memory statistics system).
- Updated statistics for system-created tables.

**6. Autovacuum Launcher Process**
- Updates statistics for **both** system-created tables **and** user-created tables (and triggers autovacuum/autoanalyze as needed).

**7–9. I/O Workers 1, 2, and 3**
- Introduced in **PostgreSQL 18** as part of a performance improvement initiative (asynchronous I/O handling).

**10. Checkpointer**
- Works alongside the Background Writer as a safety mechanism.
- Every **5 minutes** (by default), it flushes all dirty buffers from `shared_buffers` to data files on disk.

### Optional Processes
*(can be enabled or disabled as needed)*

**11. Logger Process**
- Captures database logging activity.

**12. Archiver Process**
- Copies WAL files from the `pg_wal` directory to the archive directory (used when archive mode is enabled).

**13. WAL Sender Process**
- Sends transaction logs from the primary server to a replica (used in replication setups).

**14. WAL Receiver Process**
- Receives transaction logs from the primary server (runs on the replica/standby side).

*Note: Only committed transaction data is ever flushed to permanent storage — uncommitted changes are never written to the data files.*

---

## Data Files vs. WAL Files

**Data Files (`base` folder)** — contain the actual data.

Example — table `emp(eno, ename)`:
```
(1, ram)
(2, sam)
(3, bheem)
(4, jan)
```

**WAL Files (`pg_wal` folder)** — contain transaction logs; simple transaction information stored in **binary format**:
```
INSERT ...
UPDATE ...
DELETE ...
```

---

## Crash Recovery Example (Durability in Action)

Consider this sequence of events:

1. Data is **modified in `shared_buffers`** (not yet flushed to the data files on disk).
2. The corresponding transaction is **already recorded in the WAL file**.
3. The **server crashes** at this point.

Because of the **Durability** property in **ACID**, PostgreSQL tracks all pending changes that require recovery.

- When the crashed server restarts, it first enters a **recovery state** — replaying the WAL entries that were committed but not yet reflected in the data files.
- Only after recovery completes does the server return to a **normal operating state**.
- As a result, **there is no data loss even if the server crashes** — as long as the transaction was committed (and thus recorded in the WAL) before the crash.

---

### General Notes on This Session
- The "I/O Worker" processes (introduced in PostgreSQL 18) relate to PostgreSQL's move toward **asynchronous I/O** for performance — worth cross-referencing the official release notes for exact configuration parameters (e.g., `io_method`, `io_workers`) if covering this in more depth.
- The Stats Collector Process is noted as deprecated in PostgreSQL 15 — its responsibilities were absorbed into an in-memory statistics mechanism; useful to mention this distinction if students encounter older documentation referencing it.
- The crash-recovery walkthrough at the end nicely illustrates *why* WAL exists — this could be a good spot to explicitly link back to the WAL internals covered on Day 4.
