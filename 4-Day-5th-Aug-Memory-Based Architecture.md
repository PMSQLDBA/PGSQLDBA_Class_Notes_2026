# 4th Class – 5th August 2026

## 2nd Layer: Memory-Based Architecture

### PostgreSQL Memory-Based Architecture

PostgreSQL is an operating-system-based database management system, so **memory (RAM) plays a very important role in PostgreSQL performance**.

PostgreSQL uses different memory areas for different purposes. The important memory-related parameters are:

1. **`shared_buffers`**

   * WAL-related buffers (`wal_buffers`)
   * Transaction-status buffers (`transaction_buffers`, historically called CLOG buffers)
2. **`work_mem`**
3. **`temp_buffers`**
4. **`maintenance_work_mem`**
5. **`autovacuum_work_mem`**
6. **`effective_cache_size`**

> **Important:** These parameters do not all represent the same kind of memory allocation. Some allocate PostgreSQL memory, while `effective_cache_size` is only a planner estimate and does **not** allocate RAM.

---

# Database Internal and External Operations

### Database Internal Operations

Examples of internal PostgreSQL operations include:

* Buffer/cache management
* Authentication
* Authorization
* Query processing
* Transaction management
* WAL management
* Checkpoints
* Archiving
* Crash recovery and recovery processing
* Background writer activity
* Autovacuum and other maintenance activities

### Database External Operations

Examples of operations initiated by users/applications include:

**DML:**

* `SELECT`
* `INSERT`
* `UPDATE`
* `DELETE`

**DDL:**

* `CREATE`
* `ALTER`
* `DROP`
* `TRUNCATE`
* `RENAME`

The performance of external operations is influenced by PostgreSQL's internal memory management, I/O, CPU, storage, query plans, and concurrency.

---

# Example Server Configuration

Assume:

**PostgreSQL server RAM = 16 GB**

A reasonable starting point for `shared_buffers` on a dedicated database server is approximately **25% of system RAM**.

Therefore:

**16 GB × 25% = 4 GB**

So:

```text
shared_buffers = 4GB
```

This leaves approximately:

```text
16 GB - 4 GB = 12 GB
```

for the operating system, PostgreSQL processes, per-operation memory, OS page cache, and other applications/services.

The 25% value is a **starting point, not a universal rule**. PostgreSQL documentation notes that some workloads benefit from larger values, but allocating more than about 40% of system RAM to `shared_buffers` is unlikely to be beneficial because PostgreSQL also relies on the operating system cache. ([PostgreSQL][1])

---

# 1. SHARED_BUFFERS

`shared_buffers` is the amount of memory PostgreSQL uses for its **shared memory buffer area**.

It is primarily used to cache data pages and other shared PostgreSQL data structures.

### Default Value

The default is typically:

```text
shared_buffers = 128MB
```

The minimum is:

```text
128kB
```

The actual default can depend on the system and initialization environment. ([PostgreSQL][1])

### Recommended Starting Point

For a dedicated PostgreSQL server with 1 GB or more of RAM:

```text
shared_buffers ≈ 25% of total RAM
```

For our example:

```text
RAM              = 16GB
shared_buffers   = 4GB
Remaining        ≈ 12GB
```

### Configuration

`shared_buffers` can be configured in `postgresql.conf`.

It is a **server-start parameter**, so changing it requires a PostgreSQL server restart. ([PostgreSQL][1])

You can verify the current value with:

```sql
SHOW shared_buffers;
```

### Important Correction

Do not think of `shared_buffers` as memory that is used **only for internal operations**.

It is the PostgreSQL shared buffer area used heavily for caching database pages and other shared structures. User SQL operations such as `SELECT`, `INSERT`, `UPDATE`, and `DELETE` interact with these buffers.

Also, PostgreSQL continues to rely on the **operating system's filesystem cache**, so `shared_buffers` is not the only cache involved in query execution. ([PostgreSQL][1])

---

# 1A. WAL_BUFFERS

`wal_buffers` specifies the amount of shared memory used for **WAL data that has not yet been written to disk**.

The default is:

```text
-1
```

With the automatic setting, PostgreSQL chooses approximately:

```text
shared_buffers / 32
```

subject to a minimum of 64 kB and a maximum of one WAL segment, which is typically 16 MB. ([PostgreSQL][2])

### Example

If:

```text
shared_buffers = 4GB
```

then:

```text
4GB / 32 = 128MB
```

However, **the current PostgreSQL documentation limits the automatic value to the size of one WAL segment**, typically 16 MB.

Therefore, the statement:

> "With 4 GB shared_buffers, WAL buffers will automatically be 128 MB"

is **not correct for current PostgreSQL**.

The automatic calculation reaches 128 MB, but the maximum automatic value is normally one WAL segment, typically 16 MB. ([PostgreSQL][2])

WAL buffers hold WAL records before they are written/flushed to storage. WAL is essential for crash recovery and data durability. ([PostgreSQL][3])

---

# 1B. TRANSACTION STATUS BUFFERS

## Historically called CLOG Buffers

Your original notes call this:

```text
CLOG_BUFFERS
```

A more accurate current PostgreSQL terminology is:

```text
transaction_buffers
```

These buffers cache the contents of:

```text
$PGDATA/pg_xact
```

`pg_xact` stores transaction commit-status information. ([PostgreSQL][4])

### Important Correction

The statement:

> "CLOG_BUFFERS consume 1/64 of shared_buffers."

should **not** be used as a current PostgreSQL rule.

In current PostgreSQL 18, `transaction_buffers` defaults to:

```text
0
```

which means PostgreSQL automatically chooses a value based on:

```text
shared_buffers / 512
```

with limits of up to 1024 blocks and not fewer than 16 blocks. ([PostgreSQL][1])

For an 8 kB block size:

```text
4GB / 512 = 8MB
```

So for a 4 GB `shared_buffers` configuration, the automatic transaction-status buffer calculation is approximately **8 MB**, subject to the documented limits.

### Terminology

The term **CLOG** is still encountered in PostgreSQL internals and historical documentation, but `pg_xact` and transaction-status buffers are better terminology for current notes.

---

# PostgreSQL and Lightweight Software

PostgreSQL has relatively flexible resource requirements and can run on systems with different amounts of RAM and operating-system environments.

However, saying:

> "PostgreSQL can use only 128 KB shared_buffers, therefore it is lightweight software"

is misleading.

`128 kB` is simply the **minimum allowed value** for `shared_buffers`; it is not a recommended production setting. PostgreSQL documentation explicitly notes that settings significantly higher than the minimum are usually needed for good performance. ([PostgreSQL][1])

---

# 2. WORK_MEM

`work_mem` is memory available to an individual **query operation**, such as a sort or hash operation, before PostgreSQL starts writing temporary data to disk.

Default:

```text
work_mem = 4MB
```

The minimum is:

```text
64kB
```

([PostgreSQL][1])

### Operations that can use `work_mem`

Examples include:

* `ORDER BY`
* `DISTINCT`
* Merge joins
* Hash joins
* Hash-based aggregation
* Memoize operations
* Hash processing for `IN` subqueries

([PostgreSQL][1])

### Important Correction

Do **not** say:

> "For every SQL query, PostgreSQL consumes work_mem."

A better statement is:

> `work_mem` is available to individual query execution operations that require memory, such as sorting and hashing.

A query may use multiple memory-consuming operations simultaneously.

For example:

```text
One query
   |
   +-- Sort operation       → work_mem
   |
   +-- Hash Join            → work_mem × hash_mem_multiplier
   |
   +-- Another Sort         → work_mem
```

Therefore, memory consumption can be **many times larger than the configured `work_mem` value**. ([PostgreSQL][1])

---

## Hash Memory

PostgreSQL 18 has:

```text
hash_mem_multiplier = 2.0
```

by default.

The memory limit for a hash-based operation is calculated using:

```text
work_mem × hash_mem_multiplier
```

Therefore, with:

```text
work_mem = 4MB
hash_mem_multiplier = 2
```

a hash operation can use approximately:

```text
4MB × 2 = 8MB
```

before spilling according to the applicable memory rules. ([PostgreSQL][1])

---

# WORK_MEM and Connections

`work_mem` is **not simply allocated once per connection**.

This is a very important point.

For example:

```text
100 connections
work_mem = 1GB
```

does **not** mean PostgreSQL immediately allocates:

```text
100 × 1GB = 100GB
```

Instead, `work_mem` is a limit/base amount available to individual query operations, and multiple operations and sessions can use it concurrently.

Therefore, setting `work_mem` very high can still create serious memory pressure.

PostgreSQL itself warns that several operations in one query and several concurrent sessions can cause total memory consumption to become many times the configured `work_mem`. ([PostgreSQL][1])

---

## Where Can WORK_MEM Be Configured?

`work_mem` can be configured at different scopes, including:

### Cluster/server level

In `postgresql.conf`:

```text
work_mem = 4MB
```

### Database level

```sql
ALTER DATABASE mydb SET work_mem = '10MB';
```

### Role/user level

```sql
ALTER ROLE myuser SET work_mem = '10MB';
```

### Session level

```sql
SET work_mem = '100MB';
```

A session-specific setting is useful when a particular operation requires more memory.

### Practical Principle

Do not blindly increase `work_mem` globally.

A better approach is:

> Start with a reasonable global value and increase it selectively for workloads that genuinely benefit from additional memory.

---

# 3. TEMP_BUFFERS

`temp_buffers` specifies the maximum amount of memory used for **temporary-table buffers within each database session**.

Default:

```text
temp_buffers = 8MB
```

([PostgreSQL][1])

### Important Correction

Your notes state that `temp_buffers` is used for:

> "temporary result sets generated by subqueries."

This is **not correct**.

`temp_buffers` is specifically for access to **temporary tables**. PostgreSQL documentation describes these as session-local buffers used only for temporary tables. ([PostgreSQL][1])

For example:

```sql
CREATE TEMP TABLE temp_emp AS
SELECT * FROM emp;
```

The temporary table can use `temp_buffers`.

---

## Subqueries and Temporary Results

Consider:

```sql
SELECT *
FROM emp
WHERE eno IN
(
    SELECT eno
    FROM dept
    WHERE dno IN
    (
        SELECT dno
        FROM hr
    )
);
```

The execution of subqueries does **not automatically mean their intermediate results are stored in `temp_buffers`**.

The PostgreSQL planner/executor determines how the query is executed. Intermediate results may be processed in memory or may be written to temporary files when operations such as sorting or hashing exceed their memory limits.

Those execution-memory operations are associated with `work_mem`, not `temp_buffers`.

---

## TEMP_BUFFERS Allocation

A session can allocate temporary buffers **as needed**, up to the configured limit.

For example:

```text
temp_buffers = 8MB
```

does not necessarily mean every session immediately consumes 8 MB.

PostgreSQL allocates temporary buffers as they are needed. ([PostgreSQL][1])

---

# 4. MAINTENANCE_WORK_MEM

`maintenance_work_mem` is memory available for PostgreSQL maintenance operations.

Examples include:

* `VACUUM`
* `CREATE INDEX`
* `ALTER TABLE ADD FOREIGN KEY`
* Other maintenance operations

The default is:

```text
maintenance_work_mem = 64MB
```

([PostgreSQL][1])

It can generally be set significantly higher than `work_mem` because a database session normally does not execute many maintenance operations simultaneously.

Higher values can improve the performance of operations such as vacuuming and database restoration. ([PostgreSQL][1])

---

## Example

Suppose:

```text
RAM = 16GB
shared_buffers = 4GB
```

You may temporarily increase:

```sql
SET maintenance_work_mem = '1GB';

VACUUM ANALYZE my_table;
```

The idea is to provide additional memory for the maintenance operation when appropriate.

However, do not assume that every `VACUUM` or maintenance operation will consume the entire configured value.

The setting represents a **maximum amount available for the applicable operation**, not an amount that is always immediately allocated.

---

# AUTOVACUUM_WORK_MEM

This is an important parameter missing from the detailed notes.

`autovacuum_work_mem` specifies the maximum memory available to **each autovacuum worker**.

Default:

```text
autovacuum_work_mem = -1
```

`-1` means:

> Use the value of `maintenance_work_mem`.

([PostgreSQL][1])

This is important because several autovacuum workers can run concurrently.

For example, if:

```text
autovacuum_max_workers = 3
maintenance_work_mem = 1GB
autovacuum_work_mem = -1
```

then, in principle, autovacuum workers can collectively require approximately:

```text
3 × 1GB = 3GB
```

of maintenance memory.

Therefore, setting `maintenance_work_mem` very high without considering autovacuum concurrency can create memory pressure. PostgreSQL specifically recommends considering `autovacuum_work_mem` separately when necessary. ([PostgreSQL][1])

---

# 5. EFFECTIVE_CACHE_SIZE

This parameter is **very different from the other memory parameters**.

`effective_cache_size` does **not allocate RAM**.

It is a value used by the PostgreSQL **query planner** to estimate how much data caching may be available for PostgreSQL data.

It considers the combined caching available from PostgreSQL's shared buffers and the operating-system cache. ([PostgreSQL][5])

### Very Important

Do not say:

> "effective_cache_size reserves this amount of RAM."

Instead:

> "`effective_cache_size` is a planner estimate. It does not allocate memory and does not reserve RAM."

A higher value makes index scans more attractive to the planner; a lower value makes sequential scans relatively more attractive. ([PostgreSQL][6])

---

# Memory Architecture – Simplified View

For a server with:

```text
Total RAM = 16GB
```

we can conceptually think of memory like this:

```text
                    16 GB RAM
                        |
        +---------------+----------------+
        |                                |
 PostgreSQL shared                    Remaining
 memory / shared_buffers               system RAM
        |                                |
      4 GB                       OS + PostgreSQL processes
        |                         + work_mem
        |                         + temp_buffers
        |                         + maintenance_work_mem
        |                         + autovacuum_work_mem
        |                         + OS filesystem cache
        |                         + other applications
        |
        +-- WAL-related shared memory
        |
        +-- Transaction-status buffers
        |
        +-- Other PostgreSQL shared structures
```

This is a **conceptual model**, not a strict formula in which all remaining RAM is one PostgreSQL-managed pool.

PostgreSQL uses both its own memory areas and operating-system facilities. ([PostgreSQL][1])

---

# Important Memory Parameters – Quick Revision

| Parameter              | Main Purpose                           |           Default | Scope / Nature                    |
| ---------------------- | -------------------------------------- | ----------------: | --------------------------------- |
| `shared_buffers`       | PostgreSQL shared buffer area          |   Typically 128MB | Server-start                      |
| `wal_buffers`          | WAL data waiting to be written         |       Auto (`-1`) | Server-start                      |
| `transaction_buffers`  | Cache for `pg_xact` transaction status |        Auto (`0`) | Server-start                      |
| `work_mem`             | Sort/hash/query-operation memory       |               4MB | Per operation                     |
| `temp_buffers`         | Temporary-table buffers                |               8MB | Per session                       |
| `maintenance_work_mem` | Maintenance operations                 |              64MB | Per maintenance operation/session |
| `autovacuum_work_mem`  | Memory per autovacuum worker           |              `-1` | Per autovacuum worker             |
| `effective_cache_size` | Planner's cache-size estimate          | Version dependent | **Does not allocate RAM**         |

The defaults and behavior above are based on the current PostgreSQL 18 documentation. ([PostgreSQL][1])

---

# Key Interview Points

### 1. Is `shared_buffers` 25% mandatory?

**No.**

25% is a reasonable **starting point** for a dedicated PostgreSQL server with sufficient RAM. Actual tuning depends on workload, operating system, storage, concurrency, and other applications. PostgreSQL documentation says allocations above roughly 40% are unlikely to be better in many cases because PostgreSQL also relies on the OS cache. ([PostgreSQL][1])

### 2. Does `work_mem` mean memory per connection?

**Not exactly.**

It is primarily a memory limit/base amount for an **individual query operation**, such as a sort or hash. A single query can have several such operations, and many sessions can run concurrently. ([PostgreSQL][1])

### 3. Does `temp_buffers` store subquery results?

**No.**

`temp_buffers` is for **temporary tables** in a session. Query-operation memory for sorting/hashing is governed by `work_mem` and related settings. ([PostgreSQL][1])

### 4. Does `effective_cache_size` allocate memory?

**No.**

It is only a **query planner estimate**. ([PostgreSQL][5])

### 5. What is CLOG?

CLOG is the historical/common term associated with PostgreSQL's transaction-status information.

In current PostgreSQL terminology, the relevant on-disk area is:

```text
$PGDATA/pg_xact
```

and current PostgreSQL provides the `transaction_buffers` parameter to control the shared-memory cache for it. ([PostgreSQL][4])

### 6. Does increasing memory always improve performance?

**No.**

Increasing a memory parameter without understanding workload and concurrency can cause memory pressure and potentially operating-system-level out-of-memory conditions.

PostgreSQL documentation specifically recommends considering parameters such as `shared_buffers`, `work_mem`, and connection counts when diagnosing memory exhaustion. ([PostgreSQL][7])

---

# Useful Commands

Check current memory settings:

```sql
SHOW shared_buffers;
SHOW wal_buffers;
SHOW transaction_buffers;
SHOW work_mem;
SHOW temp_buffers;
SHOW maintenance_work_mem;
SHOW autovacuum_work_mem;
SHOW effective_cache_size;
```

Check the configuration source of a parameter:

```sql
SELECT name, setting, unit, source, context
FROM pg_settings
WHERE name IN (
    'shared_buffers',
    'wal_buffers',
    'transaction_buffers',
    'work_mem',
    'temp_buffers',
    'maintenance_work_mem',
    'autovacuum_work_mem',
    'effective_cache_size'
);
```

On Linux, system memory can be checked with:

```bash
free -m
```

---

# Final Concept

Remember the memory architecture using this simple classification:

```text
POSTGRESQL MEMORY
│
├── SHARED / SERVER-START MEMORY
│   ├── shared_buffers
│   ├── wal_buffers
│   └── transaction_buffers
│
├── SESSION / QUERY MEMORY
│   ├── work_mem
│   └── temp_buffers
│
├── MAINTENANCE MEMORY
│   ├── maintenance_work_mem
│   └── autovacuum_work_mem
│
└── PLANNER ESTIMATE
    └── effective_cache_size
       → Does NOT allocate RAM
```

The most important distinction to remember is:

**`shared_buffers` = PostgreSQL shared buffer memory**

**`work_mem` = memory available per query operation**

**`temp_buffers` = temporary-table buffers per session**

**`maintenance_work_mem` = maintenance-operation memory**

**`autovacuum_work_mem` = memory per autovacuum worker**

**`effective_cache_size` = planner estimate, NOT an allocation**

Official PostgreSQL documentation: 

[PostgreSQL 18 – Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html) 

[PostgreSQL 18 – WAL Configuration](https://www.postgresql.org/docs/18/wal-configuration.html)

[1]: https://www.postgresql.org/docs/current/runtime-config-resource.html "PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[2]: https://www.postgresql.org/docs/18/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[3]: https://www.postgresql.org/docs/18/wal-configuration.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 28.5. WAL Configuration"
[4]: https://www.postgresql.org/docs/18/storage-file-layout.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 66.1. Database File Layout"
[5]: https://www.postgresql.org/message-id/20181105170112.dtxazkxcyuoszfi2%40alvherre.pgsql?utm_source=chatgpt.com "PostgreSQL: Re: effective_cache_size"
[6]: https://www.postgresql.org/docs/8.1/runtime-config-query.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 8.1: Query Planning"
[7]: https://www.postgresql.org/docs/18/kernel-resources.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.4. Managing Kernel Resources"
