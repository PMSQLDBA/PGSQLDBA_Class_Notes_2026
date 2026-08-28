# PostgreSQL 18: Database Maintenance Activities

**Class Date:** 28th August 2026 – Friday

These class notes cover PostgreSQL database maintenance activities demonstrated in the supplied session, including **ANALYZE, VACUUM, VACUUM FULL, REINDEX, AUTOVACUUM, bloat identification, vacuumdb utilities, and maintenance scheduling**. 

---

## 1. Objective

Database maintenance is performed to keep PostgreSQL databases healthy, stable, and performing efficiently.

The main goals are:

* Maintain accurate optimizer statistics
* Manage dead tuples
* Make reusable space available
* Control table and index bloat
* Maintain indexes
* Improve query-planning accuracy
* Reduce unnecessary storage consumption
* Prevent transaction ID wraparound problems
* Automate routine maintenance where appropriate

The main PostgreSQL maintenance operations covered in this class are:

1. `ANALYZE`
2. `VACUUM`
3. `VACUUM FULL`
4. `REINDEX`
5. `AUTOVACUUM`

---

# 2. Important PostgreSQL Terms

Before understanding maintenance activities, we need to understand a few basic terms.

### Statistics

Statistics contain information about the distribution and characteristics of table data.

The PostgreSQL query planner uses these statistics to estimate:

* Number of rows
* Selectivity of conditions
* Data distribution
* Suitable join methods
* Suitable scan methods

Accurate statistics help PostgreSQL choose better execution plans.

### Tuple

In PostgreSQL terminology, a **tuple** generally represents a table row.

For example, if a table contains 100 records, it contains approximately 100 logical rows or tuples.

### Attribute

An **attribute** represents a table column.

For example:

`emp(eno, ename, esal, eaddr)`

contains four attributes.

### Dead Tuple

PostgreSQL uses Multi-Version Concurrency Control (**MVCC**).

When a row is updated or deleted, the old row version does not necessarily disappear physically from the table file immediately.

That old row version can eventually become a **dead tuple** when it is no longer visible to any transaction.

For example:

* Table contains 100 rows
* Delete 50 rows
* 50 rows remain logically
* Old row versions can remain physically until VACUUM processes them

VACUUM can then make the space occupied by removable dead tuples available for reuse.

---

# 3. What is Bloat?

**Bloat** generally refers to space in tables or indexes that is no longer efficiently used because of PostgreSQL's MVCC behavior, updates, deletes, page layout, and other storage effects.

Example:

* Table size before deletion: `10 GB`
* Large amount of data deleted
* Table file may still remain close to `10 GB`

Normal VACUUM can make dead-tuple space reusable internally, but it normally does not return that space to the operating system.

This unused or inefficiently used space is commonly referred to as **table bloat**.

Indexes can also become bloated.

---

# 4. Table Size and Index Size

When checking PostgreSQL storage, distinguish between the table itself and the total relation size.

### Table Only

Use:

`pg_table_size()`

This measures the table and related storage such as TOAST, but excludes indexes.

### Indexes Only

Use:

`pg_indexes_size()`

### Table + Indexes

Use:

`pg_total_relation_size()`

Example:

`SELECT pg_size_pretty(pg_total_relation_size('emp'));`

This provides the total disk space associated with the table, including its indexes.

---

# 5. ANALYZE

`ANALYZE` collects statistics about the contents of tables.

The PostgreSQL query planner uses these statistics when generating execution plans.

### Main Purpose

**Update optimizer statistics.**

If statistics become outdated, PostgreSQL may make inaccurate row-count estimates and potentially choose inefficient execution plans.

Run ANALYZE for a table using:

`ANALYZE emp;`

For additional information during execution:

`ANALYZE VERBOSE emp;`

---

# 6. Important Characteristics of ANALYZE

ANALYZE:

* Collects table statistics
* Updates planner statistics
* Reads a statistical sample of table data in normal cases
* Updates PostgreSQL system catalog statistics
* Does not rebuild the table
* Does not remove table bloat
* Does not remove dead tuples
* Normally requires much less intrusive locking than operations such as VACUUM FULL

ANALYZE does take a lightweight table lock, so it is better to say that it generally **does not block normal reads and writes**, rather than saying that it takes no lock at all.

---

# 7. ANALYZE Lab Test

Create the test table.

**SQL:**

`CREATE TABLE emp`

`(eno int, ename varchar, esal int, eaddr varchar);`

Load data into the table.

The class loaded 1,000,000 rows five times.

After loading, verify the row count.

**SQL:**

`SELECT count(*) FROM emp;`

Result:

`5000000`

The table now contains **5 million rows**.

---

# 8. Check Table Statistics

Use:

`SELECT relname, last_vacuum, last_analyze, n_dead_tup`

`FROM pg_stat_user_tables;`

Initially, the class showed:

* `last_vacuum` – empty
* `last_analyze` – empty
* `n_dead_tup` – `0`

This means no manual VACUUM or ANALYZE had yet been recorded for the test table.

---

# 9. Execution Plan Before ANALYZE

Run:

`EXPLAIN SELECT * FROM emp;`

Before accurate statistics were available, PostgreSQL estimated approximately:

`2977560 rows`

However, the table actually contained:

`5000000 rows`

This demonstrates why current statistics are important.

---

# 10. Run ANALYZE

Run:

`ANALYZE VERBOSE emp;`

PostgreSQL collected statistics from the table.

The class output estimated approximately:

`5000016 rows`

This was much closer to the actual 5 million rows.

---

# 11. default_statistics_target

Check:

`SHOW default_statistics_target;`

The class value was:

`100`

The session changed it temporarily to:

`200`

**SQL:**

`SET default_statistics_target TO 200;`

Then:

`ANALYZE VERBOSE emp;`

A higher statistics target can cause PostgreSQL to collect more detailed statistics, potentially improving estimates for some data distributions.

However, it can also increase:

* ANALYZE time
* Statistics storage
* Planning overhead

Therefore, increasing it globally should be based on an actual requirement rather than done automatically for every database.

---

# 12. Execution Plan After ANALYZE

Run:

`EXPLAIN SELECT * FROM emp;`

After ANALYZE, the class showed:

`rows=5000000`

This matched the actual row count much more closely.

This demonstrates the relationship:

**Accurate Statistics**

↓

**Better Cardinality Estimates**

↓

**Better Information for Query Planner**

↓

**Potentially Better Execution Plans**

---

# 13. PostgreSQL Relation File

To find the physical relation path:

`SELECT pg_relation_filepath('emp');`

The class initially showed:

`base/5/16932`

This represents the relation file path relative to the PostgreSQL data directory.

---

# 14. Create Dead Tuples

The table initially contained:

`5000000 rows`

The class deleted:

`2500000 rows`

Verify:

`SELECT count(*) FROM emp;`

Result:

`2500000`

Check the table size:

`\dt+ emp`

The table still showed approximately:

`287 MB`

Check table statistics:

`SELECT relname, n_live_tup, n_dead_tup,`

`last_vacuum, last_analyze`

`FROM pg_stat_user_tables;`

The result showed approximately:

* Live tuples: `2500000`
* Dead tuples: `2500000`

This demonstrates PostgreSQL's MVCC behavior.

---

# 15. Does ANALYZE Remove Dead Tuples?

No.

The class executed:

`ANALYZE emp;`

After ANALYZE:

* `n_live_tup` remained around `2500000`
* `n_dead_tup` remained around `2500000`

ANALYZE updates statistics.

It does **not** perform the normal dead-tuple cleanup work of VACUUM.

---

# 16. VACUUM

Normal `VACUUM` processes dead tuples that are safe to remove and makes their space available for reuse inside the table.

Run:

`VACUUM emp;`

For detailed output:

`VACUUM VERBOSE emp;`

### Main Purpose

**Clean up removable dead tuples and make their space reusable.**

---

# 17. What VACUUM Does

VACUUM performs several important functions:

* Processes dead tuples that are safe to remove
* Makes reusable space available through the Free Space Map
* Updates the Visibility Map where appropriate
* Maintains visibility information
* Helps support index-only scans
* Advances tuple freezing information
* Helps prevent transaction ID wraparound
* Updates maintenance statistics

Normal VACUUM does not generally shrink the physical table file to its minimum possible size.

---

# 18. Free Space Map

The **Free Space Map (FSM)** tracks available space inside table and index pages.

PostgreSQL can use this information when inserting or updating rows.

After VACUUM processes dead tuples, reusable space can be recorded so PostgreSQL can use that space for future data.

---

# 19. Visibility Map

The **Visibility Map (VM)** tracks pages where tuples are known to be visible to all transactions and pages where tuples are frozen.

It is important for:

* Vacuum optimization
* Index-only scans
* Transaction ID maintenance

VACUUM updates visibility information as appropriate.

---

# 20. VACUUM Lab Result

The class executed:

`VACUUM VERBOSE emp;`

After VACUUM:

* Live tuples: approximately `2500000`
* Dead tuples: `0`
* `last_vacuum` was updated

However:

`\dt+ emp`

still showed approximately:

`287 MB`

This is expected.

Normal VACUUM made dead-tuple space reusable but did not shrink the table file significantly.

---

# 21. Normal VACUUM Summary

Before VACUUM:

* Table size: approximately `287 MB`
* Live tuples: `2.5 million`
* Dead tuples: `2.5 million`

After VACUUM:

* Table size: approximately `287 MB`
* Live tuples: `2.5 million`
* Dead tuples: `0`
* Free space available for reuse

Therefore:

**VACUUM = Reuse Space Internally**

It does not normally mean:

**VACUUM = Return All Free Space to OS**

---

# 22. Locking Behavior of VACUUM

Normal VACUUM takes a lightweight table lock.

It generally allows normal operations such as:

* SELECT
* INSERT
* UPDATE
* DELETE

However, it can conflict with certain DDL operations that require stronger locks.

Therefore, normal VACUUM is designed to work alongside normal database activity.

---

# 23. VACUUM FULL

`VACUUM FULL` is different from normal VACUUM.

Its main purpose is to rewrite a table into a compact physical representation and return unused space to the operating system.

Run:

`VACUUM FULL emp;`

---

# 24. How VACUUM FULL Works

Conceptually:

**Existing Bloated Table**

↓

**Rewrite Live Rows**

↓

**Create Compact Physical Representation**

↓

**Replace Old Storage**

↓

**Release Unused Space**

This requires significantly stronger locking than normal VACUUM.

---

# 25. VACUUM FULL Locking

VACUUM FULL requires an:

**ACCESS EXCLUSIVE lock**

This blocks other access to the table while the operation is running.

Therefore, VACUUM FULL should normally be planned carefully.

For production systems, it is commonly performed during an approved maintenance window when the required lock is acceptable.

---

# 26. VACUUM FULL Storage Requirement

VACUUM FULL rewrites the table.

Therefore, additional disk space is required while the operation is running.

Before running it on a large table, verify:

* Current table size
* Index size
* Available filesystem space
* Expected maintenance duration
* Application impact
* Locking impact

---

# 27. maintenance_work_mem

The class checked:

`SHOW maintenance_work_mem;`

Result:

`64MB`

It was temporarily changed to:

`256MB`

**SQL:**

`SET maintenance_work_mem TO '256MB';`

Verify:

`SHOW maintenance_work_mem;`

`maintenance_work_mem` controls memory available to certain maintenance operations.

Its value should be selected based on available system memory and maintenance workload.

---

# 28. VACUUM FULL Lab Result

Before VACUUM FULL:

* Table size: approximately `287 MB`

Run:

`VACUUM FULL emp;`

After VACUUM FULL:

* Table size: approximately `144 MB`

Therefore, approximately half of the previously allocated table storage was eliminated by the rewrite in this particular demonstration.

---

# 29. Relation File Changed After VACUUM FULL

Before VACUUM FULL:

`base/5/16932`

After VACUUM FULL:

`base/5/16937`

The class checked this using:

`SELECT pg_relation_filepath('emp');`

The changed relation file demonstrates that VACUUM FULL rewrote the table storage.

---

# 30. VACUUM vs VACUUM FULL

| Feature                          | VACUUM               | VACUUM FULL          |
| -------------------------------- | -------------------- | -------------------- |
| Processes dead tuples            | Yes                  | Yes                  |
| Makes space reusable             | Yes                  | Yes                  |
| Returns space to OS              | Usually No           | Yes                  |
| Rewrites table                   | No                   | Yes                  |
| Blocks normal table access       | Generally No         | Yes                  |
| ACCESS EXCLUSIVE lock            | No                   | Yes                  |
| Extra disk space                 | Limited              | Required for rewrite |
| Suitable for routine maintenance | Yes                  | No                   |
| Maintenance window recommended   | Usually not required | Usually yes          |

---

# 31. Table Bloat Detection

The class used a SQL query to estimate table bloat.

The query compared:

* Estimated rows
* Actual table size
* Expected table size
* Estimated bloat
* Percentage of bloat

For the `emp` table, the output showed approximately:

* Estimated rows: `2500000`
* Bloat: `50%`
* Bloat size: `143.78 MB`
* Table size: `287.188 MB`

This aligned closely with the later VACUUM FULL result, where the table reduced to approximately `144 MB`.

### Important Note

Bloat queries based on PostgreSQL statistics are **estimates**.

Treat the result as an indicator for investigation rather than an exact measurement in every environment.

---

# 32. Deciding When to Run VACUUM FULL

The class discussed example criteria such as:

* Smaller table with high percentage bloat
* Larger table with at least `1 GB` of wasted space
* Larger table with approximately `25%` or more bloat

For example:

* Table size: `4 GB`
* Estimated bloat: `1 GB`
* Bloat percentage: `25%`

This may justify further investigation.

These should be treated as **local operational guidelines**, not PostgreSQL-required thresholds.

Before VACUUM FULL, also consider:

* Available disk space
* Business impact
* Lock duration
* Table growth
* Actual storage pressure
* Frequency of updates/deletes
* Whether normal VACUUM can manage future reuse

---

# 33. Online Alternative for Table Reorganization

The class mentioned the third-party tool:

`pg_repack`

`pg_repack` is commonly used to reorganize PostgreSQL tables and indexes with reduced blocking compared with VACUUM FULL.

It is not part of PostgreSQL core.

Production use requires separate installation, testing, compatibility checks, operational planning, and understanding of its locking requirements.

---

# 34. REINDEX

Indexes can also become bloated or require rebuilding.

PostgreSQL provides `REINDEX` to rebuild indexes.

Typical reasons include:

* Index bloat
* Index corruption concerns
* Invalid indexes in appropriate recovery scenarios
* Rebuilding an index after certain operational problems

---

# 35. Check for Invalid Indexes

The class used:

`SELECT indexrelid, indrelid, indisvalid`

`FROM pg_index`

`WHERE indisvalid = false;`

The result returned:

`0 rows`

This means no invalid indexes were found at that time.

---

# 36. REINDEX a Table

Rebuild indexes belonging to a table:

`REINDEX TABLE emp;`

For reduced blocking:

`REINDEX TABLE CONCURRENTLY emp;`

The concurrent form is useful when maintaining availability is important, although it takes longer and has additional restrictions and processing overhead.

---

# 37. REINDEX a Database

Run:

`REINDEX DATABASE postgres;`

This rebuilds indexes in the specified database according to PostgreSQL's REINDEX DATABASE behavior.

This can be a substantial maintenance operation and should be planned carefully.

---

# 38. REINDEX a Specific Index

Create the example index:

`CREATE INDEX emp_idx ON emp(eno);`

Rebuild it:

`REINDEX INDEX emp_idx;`

Concurrent rebuild:

`REINDEX INDEX CONCURRENTLY emp_idx;`

---

# 39. VACUUM FULL and Indexes

The class asked:

> If the EMP table has indexes and VACUUM FULL is performed, do we need to immediately REINDEX those indexes separately?

Normally, **no**.

VACUUM FULL rewrites the table and rebuilds its indexes as part of the operation.

Therefore, an additional REINDEX solely to remove the previous index bloat is normally unnecessary immediately after a successful VACUUM FULL.

---

# 40. ANALYZE Entire Database

When connected to a database:

`ANALYZE;`

With no table specified, PostgreSQL analyzes tables in the current database that the executing user has permission to analyze.

---

# 41. VACUUM Entire Database

Run:

`VACUUM;`

This performs normal VACUUM processing on eligible tables in the current database.

---

# 42. VACUUM FULL Entire Database

Run:

`VACUUM FULL;`

This applies VACUUM FULL to tables in the current database that can be processed.

Because VACUUM FULL takes strong locks and rewrites tables, running it broadly against a production database requires careful planning.

---

# 43. PostgreSQL Maintenance Utilities

PostgreSQL provides command-line maintenance utilities under:

`/usr/pgsql-18/bin/`

Important utilities include:

* `vacuumdb`
* `reindexdb`

These are useful for:

* Shell scripts
* Cron jobs
* DBA maintenance jobs
* Automation

---

# 44. vacuumdb

`vacuumdb` is a command-line wrapper around PostgreSQL VACUUM and ANALYZE functionality.

It is useful when maintenance needs to be performed from the operating-system command line.

---

# 45. VACUUM All Databases

The class used:

`vacuumdb -a`

`-a` means:

**All databases**

The output showed maintenance running against multiple databases.

---

# 46. VACUUM a Single Database

Run:

`vacuumdb -d postgres`

Here:

* `-d postgres` specifies the database

---

# 47. VACUUM Specific Tables

Run:

`vacuumdb -d postgres -t emp -t hr -t dept`

Here:

* Database: `postgres`
* Tables: `emp`, `hr`, `dept`

This is useful when only selected tables require maintenance.

---

# 48. VACUUM + ANALYZE

Run:

`vacuumdb -d postgres -t emp -t hr -t dept -z`

The `-z` option means:

**VACUUM and ANALYZE**

This performs both operations for the selected tables.

---

# 49. ANALYZE Only

Run:

`vacuumdb -d postgres -t emp -t hr -t dept -Z`

Capital `-Z` means:

**ANALYZE only**

This updates statistics without requesting VACUUM.

---

# 50. VACUUM FULL Using vacuumdb

Run:

`vacuumdb -d postgres -t emp -t hr -t dept -f`

The `-f` option requests:

**VACUUM FULL**

Remember that this can acquire strong locks on the selected tables.

---

# 51. REINDEX Command-Line Utility

PostgreSQL also provides:

`reindexdb`

Location:

`/usr/pgsql-18/bin/reindexdb`

This can be used for command-line index maintenance and is useful in shell-based DBA procedures.

---

# 52. AUTOVACUUM

PostgreSQL provides **autovacuum** to automate routine VACUUM and ANALYZE maintenance.

Check its status:

`SHOW autovacuum;`

The normal default is:

`on`

Autovacuum is an important PostgreSQL subsystem and should generally remain enabled.

---

# 53. Autovacuum Architecture

At a high level:

**PostgreSQL Server**

↓

**Autovacuum Launcher**

↓

**Checks Databases**

↓

**Autovacuum Workers**

↓

**Process Tables Requiring Maintenance**

↓

**VACUUM and/or ANALYZE**

The autovacuum launcher coordinates autovacuum workers.

---

# 54. Check Autovacuum Settings

The class used:

`SELECT name, setting, short_desc`

`FROM pg_settings`

`WHERE name LIKE '%autovacuum%';`

Important settings include:

* `autovacuum`
* `autovacuum_analyze_scale_factor`
* `autovacuum_analyze_threshold`
* `autovacuum_freeze_max_age`
* `autovacuum_max_workers`
* `autovacuum_naptime`
* `autovacuum_vacuum_cost_delay`
* `autovacuum_vacuum_insert_scale_factor`
* `autovacuum_vacuum_insert_threshold`
* `autovacuum_vacuum_max_threshold`
* `autovacuum_vacuum_scale_factor`
* `autovacuum_vacuum_threshold`
* `autovacuum_work_mem`
* `autovacuum_worker_slots`
* `log_autovacuum_min_duration`

The exact defaults should always be checked using `SHOW` or `pg_settings` on the PostgreSQL server because settings can vary by PostgreSQL version and configuration.

---

# 55. Autovacuum VACUUM Threshold

A simplified threshold calculation is based on:

**VACUUM Threshold**

`autovacuum_vacuum_threshold`

*

`autovacuum_vacuum_scale_factor × table size estimate`

The class values showed:

* Threshold: `50`
* Scale factor: `0.2`

For a table with 1,000,000 tuples, a simplified example would be approximately:

`50 + (0.2 × 1,000,000)`

=

`200,050`

This helps determine when autovacuum considers VACUUM work necessary, subject to PostgreSQL's complete autovacuum rules and configuration.

---

# 56. Auto-Analyze Threshold

A simplified ANALYZE threshold is:

`autovacuum_analyze_threshold`

*

`autovacuum_analyze_scale_factor × table size estimate`

The class values showed:

* Analyze threshold: `50`
* Analyze scale factor: `0.1`

For 1,000,000 tuples:

`50 + (0.1 × 1,000,000)`

=

`100,050`

Again, this is a simplified explanation of the threshold calculation.

---

# 57. autovacuum_naptime

The class showed:

`autovacuum_naptime = 60`

This setting controls the minimum delay between autovacuum launcher activity on individual databases.

The displayed unit should be checked through PostgreSQL configuration; the common setting is expressed as a time value such as seconds.

---

# 58. autovacuum_max_workers

The class showed:

`autovacuum_max_workers = 3`

This controls the maximum number of autovacuum worker processes that can run concurrently, subject to PostgreSQL's worker-slot configuration.

For large or heavily updated systems, autovacuum capacity should be monitored and tuned based on actual workload.

---

# 59. autovacuum_freeze_max_age

The class showed:

`autovacuum_freeze_max_age = 200000000`

This setting is important for transaction ID wraparound protection.

PostgreSQL may perform autovacuum work to ensure old transaction IDs are frozen before reaching dangerous age limits.

This is one of the reasons autovacuum should not simply be disabled in a production database.

---

# 60. Temporarily Disable Autovacuum for Lab Testing

The class disabled autovacuum to make the manual maintenance behavior easier to demonstrate.

Check:

`SHOW autovacuum;`

Edit:

`postgresql.conf`

Set:

`autovacuum = off`

Reload PostgreSQL:

`/usr/pgsql-18/bin/pg_ctl -D /var/lib/pgsql/18/data/ reload`

Verify:

`SHOW autovacuum;`

Result:

`off`

### Important

Disabling autovacuum is appropriate only for carefully controlled testing scenarios.

It is **not a general production recommendation**.

---

# 61. Enable Autovacuum Again

Edit:

`postgresql.conf`

Set:

`autovacuum = on`

Reload:

`/usr/pgsql-18/bin/pg_ctl -D /var/lib/pgsql/18/data/ reload`

Check PostgreSQL processes.

**Command:**

`ps -ef | grep postgres`

The class later showed:

`postgres: autovacuum launcher`

This confirms that the autovacuum launcher was running.

---

# 62. Monitor Autovacuum Activity

Use:

`SELECT relname, last_autovacuum, last_autoanalyze`

`FROM pg_stat_user_tables;`

This helps determine when PostgreSQL last automatically vacuumed or analyzed individual tables.

Additional useful columns include:

* `n_live_tup`
* `n_dead_tup`
* `last_vacuum`
* `last_autovacuum`
* `last_analyze`
* `last_autoanalyze`
* `vacuum_count`
* `autovacuum_count`
* `analyze_count`
* `autoanalyze_count`

---

# 63. Autovacuum Test

The class updated all rows in the `emp` table:

`UPDATE emp SET eno = 100000;`

This generated a large amount of row-version churn.

Initially, the autovacuum timestamps were empty.

Later, PostgreSQL showed:

* `last_autovacuum` populated for `emp`
* `last_autoanalyze` populated for `emp`

This demonstrated automatic maintenance being triggered after sufficient table changes.

---

# 64. Autovacuum on Busy Tables

A busy table does **not simply cause autovacuum to be skipped because it has traffic**.

Autovacuum is specifically designed to operate concurrently with normal database workloads.

However, autovacuum can be delayed or prevented from making progress because of conditions such as:

* Conflicting locks
* Long-running transactions
* Insufficient autovacuum workers
* Heavy I/O pressure
* Cost-based throttling
* Poor autovacuum configuration
* Continuous high-volume table changes

Therefore, heavily updated tables should be **monitored and tuned**, rather than assuming autovacuum will always be skipped.

---

# 65. Should We Rely on Autovacuum?

Autovacuum is a core PostgreSQL maintenance feature and should normally be enabled.

A good production approach is:

**Keep autovacuum enabled**

*

**Monitor whether it keeps up**

*

**Tune per table when necessary**

*

**Use manual maintenance for exceptional cases**

Manual VACUUM and ANALYZE can supplement autovacuum, but routinely disabling or replacing autovacuum with cron jobs is generally not recommended.

---

# 66. Manual Maintenance Strategy

The class discussed scheduling maintenance based on table size.

Example operational categories were:

* Tables smaller than `5 GB` – potentially daily
* Tables between `5 GB` and `100 GB` – potentially weekly
* Tables larger than `100 GB` – potentially monthly

These are **example local scheduling guidelines**, not PostgreSQL-required rules.

Actual scheduling should depend on:

* Update/delete frequency
* Dead-tuple generation
* Table size
* Autovacuum effectiveness
* Query workload
* Maintenance window
* Storage pressure
* Transaction age

---

# 67. Scheduling Maintenance Using Cron

Linux `cron` can be used to schedule command-line maintenance jobs.

Check the postgres user's crontab:

`crontab -l`

Edit it:

`crontab -e`

Example from the class:

`0 1 * * * vacuumdb -d postgres -t emp -t hr -t dept -z`

This means the command runs every day at:

**1:00 AM**

---

# 68. Cron Format

Cron uses five scheduling fields:

`Minute Hour Day Month Weekday`

Typical ranges are:

* Minute: `0–59`
* Hour: `0–23`
* Day of month: `1–31`
* Month: `1–12`
* Day of week: commonly `0–7`, with Sunday represented by `0` or `7` on many cron implementations

Example:

`0 1 * * *`

means:

**Every day at 1:00 AM**

---

# 69. Every Monday Example

The class example was:

`0 0 * * MON`

This means:

**Every Monday at midnight**

Cron syntax can vary slightly between implementations, so validate production schedules on the target operating system.

---

# 70. Production Cron Recommendation

When scheduling PostgreSQL utilities, it is safer to use the full executable path.

For example:

`0 1 * * * /usr/pgsql-18/bin/vacuumdb -d postgres -t emp -t hr -t dept -z`

Also consider:

* Logging stdout and stderr
* Alerting on failure
* Authentication configuration
* Preventing overlapping jobs
* Runtime monitoring
* Maintenance windows

---

# 71. Maintenance Decision Guide

### Need Updated Statistics?

Use:

`ANALYZE`

### Need Dead-Tuple Cleanup and Reusable Space?

Use:

`VACUUM`

### Need to Physically Shrink a Bloated Table?

Consider:

`VACUUM FULL`

### Need to Rebuild an Index?

Use:

`REINDEX`

### Need Reduced Blocking for Index Rebuild?

Consider:

`REINDEX ... CONCURRENTLY`

### Need Automatic Routine Maintenance?

Use:

`AUTOVACUUM`

### Need Shell-Based Maintenance?

Use:

`vacuumdb` and `reindexdb`

---

# 72. Quick Comparison

| Activity             | Main Purpose                  | Normal DML Blocking               | Shrinks Table File | Typical Usage  |
| -------------------- | ----------------------------- | --------------------------------- | ------------------ | -------------- |
| ANALYZE              | Update statistics             | No                                | No                 | Routine        |
| VACUUM               | Reclaim reusable tuple space  | No                                | Usually No         | Routine        |
| VACUUM FULL          | Rewrite and compact table     | Yes                               | Yes                | Exceptional    |
| REINDEX              | Rebuild indexes               | Depends on mode                   | Index only         | As needed      |
| REINDEX CONCURRENTLY | Rebuild with reduced blocking | Reduced                           | Index only         | Online systems |
| AUTOVACUUM           | Automatic VACUUM/ANALYZE      | Designed for concurrent workloads | Usually No         | Continuous     |

---

# 73. Useful Monitoring Queries

### Dead Tuples

`SELECT schemaname, relname, n_live_tup, n_dead_tup`

`FROM pg_stat_user_tables`

`ORDER BY n_dead_tup DESC;`

### Maintenance History

`SELECT relname, last_vacuum, last_autovacuum,`

`last_analyze, last_autoanalyze`

`FROM pg_stat_user_tables;`

### Autovacuum Configuration

`SELECT name, setting, unit`

`FROM pg_settings`

`WHERE name LIKE 'autovacuum%';`

### Table Size

`SELECT pg_size_pretty(pg_table_size('emp'));`

### Index Size

`SELECT pg_size_pretty(pg_indexes_size('emp'));`

### Total Relation Size

`SELECT pg_size_pretty(pg_total_relation_size('emp'));`

### Relation File

`SELECT pg_relation_filepath('emp');`

---

# 74. Important Production Points

* Keep **autovacuum enabled** in normal production environments.
* Monitor whether autovacuum is keeping up with table changes.
* Tune heavily modified tables individually when required.
* Use ANALYZE when planner statistics need refreshing.
* Use normal VACUUM as routine maintenance.
* Do not expect normal VACUUM to significantly reduce the physical table file.
* Use VACUUM FULL only when a table rewrite and disk-space reclamation justify the locking impact.
* Check free disk space before VACUUM FULL.
* Remember that VACUUM FULL requires an ACCESS EXCLUSIVE lock.
* Use REINDEX when an index needs rebuilding.
* Consider `REINDEX CONCURRENTLY` when availability is important.
* Treat bloat-estimation queries as estimates.
* Do not use fixed table-size schedules as a replacement for monitoring.
* Monitor dead tuples, transaction age, maintenance timestamps, storage growth, and autovacuum performance.

---

# 75. Complete Maintenance Flow

A practical maintenance workflow is:

**Monitor Tables**

↓

Check:

* Table size
* Dead tuples
* Statistics
* Autovacuum history
* Transaction age
* Bloat indicators

↓

**Statistics Outdated?**

Run:

`ANALYZE`

↓

**Dead Tuples Accumulating?**

Run or tune:

`VACUUM / AUTOVACUUM`

↓

**Severe Table Bloat and Space Must Be Returned?**

Evaluate:

`VACUUM FULL`

↓

**Index Requires Rebuild?**

Evaluate:

`REINDEX`

or

`REINDEX CONCURRENTLY`

↓

**Continue Monitoring**

---

# 76. Final Class Summary

PostgreSQL database maintenance can be understood using four core operations:

### ANALYZE

**Purpose:** Update statistics.

`ANALYZE emp;`

### VACUUM

**Purpose:** Process dead tuples and make space reusable.

`VACUUM emp;`

### VACUUM FULL

**Purpose:** Rewrite the table and return unused physical space.

`VACUUM FULL emp;`

### REINDEX

**Purpose:** Rebuild indexes.

`REINDEX INDEX emp_idx;`

And PostgreSQL automates routine VACUUM and ANALYZE work using:

### AUTOVACUUM

`autovacuum = on`

The preferred production approach is:

**Autovacuum Enabled**

↓

**Continuous Monitoring**

↓

**Tune Where Needed**

↓

**Manual VACUUM / ANALYZE for Exceptional Workloads**

↓

**VACUUM FULL / REINDEX Only When Justified**

This combination helps maintain **accurate statistics, manageable dead tuples, efficient storage usage, healthy indexes, and stable PostgreSQL performance**.
