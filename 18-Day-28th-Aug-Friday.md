# PostgreSQL 18: Database Maintenance Activities

**Class Date:** 28th August 2026 – Friday

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

### Attribute

An **attribute** represents a table column.

### Dead Tuple

PostgreSQL uses Multi-Version Concurrency Control (**MVCC**).

When a row is updated or deleted, the old row version does not necessarily disappear physically from the table file immediately.

That old row version can eventually become a **dead tuple** when it is no longer visible to any transaction.

Example:

* Table contains 100 rows
* Delete 50 rows
* 50 rows remain logically
* Old row versions can remain physically until VACUUM processes them

VACUUM can then make the space occupied by removable dead tuples available for reuse.

---

# 3. What is Bloat?

**Bloat** generally refers to space in tables or indexes that is no longer efficiently used.

Example:

* Table size before deletion: `10 GB`
* Large amount of data deleted
* Table file may still remain close to `10 GB`

Normal VACUUM can make dead-tuple space reusable internally, but it normally does not return that space to the operating system.

Indexes can also become bloated.

---

# 4. Table Size and Index Size

When checking PostgreSQL storage, distinguish between the table itself and the total relation size.

### Table Only

`pg_table_size()`

### Indexes Only

`pg_indexes_size()`

### Table + Indexes

`pg_total_relation_size()`

---

# 5. ANALYZE

`ANALYZE` collects statistics about the contents of tables.

The PostgreSQL query planner uses these statistics when generating execution plans.

### Main Purpose

**Update optimizer statistics.**

If statistics become outdated, PostgreSQL may make inaccurate row-count estimates.

Command:

```sql
ANALYZE emp;
```

Verbose form:

```sql
ANALYZE VERBOSE emp;
```

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

---

# 7. ANALYZE Lab Test

Create the test table.

```sql
CREATE TABLE emp
(
    eno int,
    ename varchar,
    esal int,
    eaddr varchar
);
```

Load 1,000,000 rows.

```sql
INSERT INTO emp
VALUES
(
    generate_series(1,1000000),
    'emp'||generate_series(1,1000000),
    generate_series(1,1000000),
    'addr'||generate_series(1,1000000)
);
```

The same insert was executed five times during the class.

```sql
INSERT INTO emp
VALUES
(
    generate_series(1,1000000),
    'emp'||generate_series(1,1000000),
    generate_series(1,1000000),
    'addr'||generate_series(1,1000000)
);
```

```sql
INSERT INTO emp
VALUES
(
    generate_series(1,1000000),
    'emp'||generate_series(1,1000000),
    generate_series(1,1000000),
    'addr'||generate_series(1,1000000)
);
```

```sql
INSERT INTO emp
VALUES
(
    generate_series(1,1000000),
    'emp'||generate_series(1,1000000),
    generate_series(1,1000000),
    'addr'||generate_series(1,1000000)
);
```

```sql
INSERT INTO emp
VALUES
(
    generate_series(1,1000000),
    'emp'||generate_series(1,1000000),
    generate_series(1,1000000),
    'addr'||generate_series(1,1000000)
);
```

Verify the row count.

```sql
SELECT count(*) FROM emp;
```

Result:

`5000000`

---

# 8. Check Table Statistics

Query:

```sql
SELECT
    relname,
    last_vacuum,
    last_analyze,
    n_dead_tup
FROM pg_stat_user_tables;
```

Initially:

* `last_vacuum` – empty
* `last_analyze` – empty
* `n_dead_tup` – `0`

---

# 9. Execution Plan Before ANALYZE

Command:

```sql
EXPLAIN SELECT * FROM emp;
```

Before updated statistics, PostgreSQL estimated approximately:

`2977560 rows`

Actual rows:

`5000000`

---

# 10. Run ANALYZE

Command:

```sql
ANALYZE VERBOSE emp;
```

Check:

```sql
SHOW default_statistics_target;
```

Result:

`100`

Change it:

```sql
SET default_statistics_target TO 200;
```

Run ANALYZE again:

```sql
ANALYZE VERBOSE emp;
```

After this, PostgreSQL estimated approximately:

`5000000 rows`

---

# 11. default_statistics_target

Commands:

```sql
SHOW default_statistics_target;
```

```sql
SET default_statistics_target TO 200;
```

```sql
ANALYZE VERBOSE emp;
```

---

# 12. Execution Plan After ANALYZE

Command:

```sql
EXPLAIN SELECT * FROM emp;
```

After ANALYZE:

`rows=5000000`

---

# 13. PostgreSQL Relation File

Command:

```sql
SELECT pg_relation_filepath('emp');
```

Result:

`base/5/16932`

Calculate approximate table pages:

```sql
SELECT 36760*8;
```

Result:

`294080`

Then:

```sql
SELECT 294080/1024;
```

Result:

`287`

This matched the approximate table size of `287 MB`.

---

# 14. Create Dead Tuples

Verify the current rows:

```sql
SELECT count(*) FROM emp;
```

Result:

`5000000`

Delete rows:

```sql
DELETE FROM emp
WHERE eno <= 500000;
```

Result:

`DELETE 2500000`

Verify:

```sql
SELECT count(*) FROM emp;
```

Result:

`2500000`

Check table size:

```text
\dt+ emp
```

Result:

Approximately `287 MB`

Check live and dead tuples:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables;
```

Class result:

* `n_live_tup = 2500000`
* `n_dead_tup = 2500000`

---

# 15. Does ANALYZE Remove Dead Tuples?

No.

Command:

```sql
ANALYZE emp;
```

Check again:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables;
```

The dead tuples were still present.

---

# 16. VACUUM

Normal `VACUUM` processes dead tuples that are safe to remove and makes their space available for reuse inside the table.

Command:

```sql
VACUUM VERBOSE emp;
```

---

# 17. What VACUUM Does

VACUUM:

* Processes removable dead tuples
* Makes space reusable
* Updates Free Space Map
* Updates Visibility Map
* Helps transaction ID maintenance
* Supports index-only scans

---

# 18. Free Space Map

The **Free Space Map (FSM)** tracks available space inside PostgreSQL pages.

VACUUM makes freed locations available for future reuse.

---

# 19. Visibility Map

The **Visibility Map (VM)** tracks pages containing tuples that are visible to all transactions and pages that are fully frozen.

VACUUM maintains this information.

---

# 20. VACUUM Lab Result

Command:

```sql
VACUUM VERBOSE emp;
```

Check table statistics:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables;
```

After VACUUM:

* Live tuples: `2500000`
* Dead tuples: `0`

Check size:

```text
\dt+ emp
```

The table still showed approximately:

`287 MB`

Check physical file:

```sql
SELECT pg_relation_filepath('emp');
```

Result remained:

`base/5/16932`

---

# 21. Normal VACUUM Summary

Before VACUUM:

* Table size: `287 MB`
* Live tuples: `2.5 million`
* Dead tuples: `2.5 million`

After VACUUM:

* Table size: `287 MB`
* Live tuples: `2.5 million`
* Dead tuples: `0`

Normal VACUUM makes the space reusable internally.

---

# 22. Locking Behavior of VACUUM

Normal VACUUM is intended to operate with normal workload activity.

Class observation:

`At any point of time we can run vacuum operation.`

---

# 23. VACUUM FULL

`VACUUM FULL` is used to physically rewrite a table and reclaim unused disk space.

Command:

```sql
VACUUM FULL emp;
```

---

# 24. How VACUUM FULL Works

Conceptually:

**Existing Table**

↓

**Rewrite Live Data**

↓

**Create New Physical Relation**

↓

**Replace Old Relation**

↓

**Release Unused Storage**

---

# 25. VACUUM FULL Locking

VACUUM FULL requires an exclusive table lock.

It should be planned during a maintenance window.

---

# 26. VACUUM FULL Storage Requirement

VACUUM FULL requires additional working disk space because the table is rewritten.

---

# 27. maintenance_work_mem

Check:

```sql
SHOW maintenance_work_mem;
```

Result:

`64MB`

Change it:

```sql
SET maintenance_work_mem TO '256MB';
```

Verify:

```sql
SHOW maintenance_work_mem;
```

Result:

`256MB`

---

# 28. VACUUM FULL Lab Result

Before VACUUM FULL:

```text
\dt+ emp
```

Approximate size:

`287 MB`

Run:

```sql
VACUUM FULL emp;
```

Check again:

```text
\dt+ emp
```

Approximate size:

`144 MB`

---

# 29. Relation File Changed After VACUUM FULL

Before:

```sql
SELECT pg_relation_filepath('emp');
```

Result:

`base/5/16932`

After VACUUM FULL:

```sql
SELECT pg_relation_filepath('emp');
```

Result:

`base/5/16937`

This demonstrated that the table was rewritten.

Then run:

```sql
VACUUM VERBOSE emp;
```

---

# 30. VACUUM vs VACUUM FULL

| Feature               | VACUUM      | VACUUM FULL |
| --------------------- | ----------- | ----------- |
| Processes dead tuples | Yes         | Yes         |
| Makes space reusable  | Yes         | Yes         |
| Returns space to OS   | Normally No | Yes         |
| Rewrites table        | No          | Yes         |
| Strong table lock     | No          | Yes         |
| Additional disk space | Limited     | Required    |
| Routine maintenance   | Yes         | No          |
| Maintenance window    | Usually No  | Recommended |

---

# 31. Table Bloat Detection

Use the following bloat-estimation query:

```sql
WITH constants AS
(
    SELECT
        current_setting('block_size')::numeric AS bs,
        23 AS hdr,
        8 AS ma
),
no_stats AS
(
    SELECT
        table_schema,
        table_name,
        n_live_tup::numeric AS est_rows,
        pg_table_size(relid)::numeric AS table_size
    FROM information_schema.columns
    JOIN pg_stat_user_tables AS psut
      ON table_schema = psut.schemaname
     AND table_name = psut.relname
    LEFT OUTER JOIN pg_stats
      ON table_schema = pg_stats.schemaname
     AND table_name = pg_stats.tablename
     AND column_name = attname
    WHERE attname IS NULL
      AND table_schema NOT IN
          ('pg_catalog','information_schema')
    GROUP BY
        table_schema,
        table_name,
        relid,
        n_live_tup
),
null_headers AS
(
    SELECT
        hdr + 1 +
        (
            SUM(
                CASE
                    WHEN null_frac <> 0
                    THEN 1
                    ELSE 0
                END
            ) / 8
        ) AS nullhdr,

        SUM(
            (1-null_frac) * avg_width
        ) AS datawidth,

        MAX(null_frac) AS maxfracsum,

        schemaname,
        tablename,
        hdr,
        ma,
        bs

    FROM pg_stats
    CROSS JOIN constants

    LEFT OUTER JOIN no_stats
      ON schemaname = no_stats.table_schema
     AND tablename = no_stats.table_name

    WHERE schemaname NOT IN
          ('pg_catalog','information_schema')

      AND no_stats.table_name IS NULL

      AND EXISTS
      (
          SELECT 1
          FROM information_schema.columns
          WHERE schemaname =
                columns.table_schema
            AND tablename =
                columns.table_name
      )

    GROUP BY
        schemaname,
        tablename,
        hdr,
        ma,
        bs
),
data_headers AS
(
    SELECT
        ma,
        bs,
        hdr,
        schemaname,
        tablename,

        (
            datawidth +
            (
                hdr + ma -
                (
                    CASE
                        WHEN hdr % ma = 0
                        THEN ma
                        ELSE hdr % ma
                    END
                )
            )
        )::numeric AS datahdr,

        (
            maxfracsum *
            (
                nullhdr + ma -
                (
                    CASE
                        WHEN nullhdr % ma = 0
                        THEN ma
                        ELSE nullhdr % ma
                    END
                )
            )
        ) AS nullhdr2

    FROM null_headers
),
table_estimates AS
(
    SELECT
        schemaname,
        tablename,
        bs,

        reltuples::numeric AS est_rows,

        relpages * bs AS table_bytes,

        CEIL
        (
            (
                reltuples *
                (
                    datahdr +
                    nullhdr2 +
                    4 +
                    ma -
                    (
                        CASE
                            WHEN datahdr % ma = 0
                            THEN ma
                            ELSE datahdr % ma
                        END
                    )
                )
                /
                (bs - 20)
            )
        ) * bs AS expected_bytes,

        reltoastrelid

    FROM data_headers

    JOIN pg_class
      ON tablename = relname

    JOIN pg_namespace
      ON relnamespace = pg_namespace.oid
     AND schemaname = nspname

    WHERE pg_class.relkind = 'r'
),
estimates_with_toast AS
(
    SELECT
        schemaname,
        tablename,
        TRUE AS can_estimate,
        est_rows,

        table_bytes +
        (
            COALESCE(toast.relpages,0) * bs
        ) AS table_bytes,

        expected_bytes +
        (
            CEIL(
                COALESCE(toast.reltuples,0) / 4
            ) * bs
        ) AS expected_bytes

    FROM table_estimates

    LEFT OUTER JOIN pg_class AS toast
      ON table_estimates.reltoastrelid =
         toast.oid
     AND toast.relkind = 't'
),
table_estimates_plus AS
(
    SELECT
        current_database() AS databasename,
        schemaname,
        tablename,
        can_estimate,
        est_rows,

        CASE
            WHEN table_bytes > 0
            THEN table_bytes::numeric
            ELSE NULL::numeric
        END AS table_bytes,

        CASE
            WHEN expected_bytes > 0
            THEN expected_bytes::numeric
            ELSE NULL::numeric
        END AS expected_bytes,

        CASE
            WHEN expected_bytes > 0
             AND table_bytes > 0
             AND expected_bytes <= table_bytes
            THEN
                (
                    table_bytes -
                    expected_bytes
                )::numeric
            ELSE
                0::numeric
        END AS bloat_bytes

    FROM estimates_with_toast

    UNION ALL

    SELECT
        current_database() AS databasename,
        table_schema,
        table_name,
        FALSE,
        est_rows,
        table_size,
        NULL::numeric,
        NULL::numeric

    FROM no_stats
),
bloat_data AS
(
    SELECT
        current_database() AS databasename,
        schemaname,
        tablename,
        can_estimate,
        table_bytes,

        ROUND(
            table_bytes /
            (1024^2)::numeric,
            3
        ) AS table_mb,

        expected_bytes,

        ROUND(
            expected_bytes /
            (1024^2)::numeric,
            3
        ) AS expected_mb,

        ROUND(
            bloat_bytes *
            100 /
            table_bytes
        ) AS pct_bloat,

        ROUND(
            bloat_bytes /
            (1024::numeric^2),
            2
        ) AS mb_bloat,

        est_rows

    FROM table_estimates_plus
)

SELECT
    databasename,
    schemaname,
    tablename,
    can_estimate,
    est_rows,
    pct_bloat,
    mb_bloat,
    table_mb
FROM bloat_data;
```

Class result for `emp` was approximately:

* Estimated rows: `2500000`
* Bloat percentage: `50%`
* Bloat: `143.78 MB`
* Table size: `287.188 MB`

---

# 32. Deciding When to Run VACUUM FULL

Class operational examples:

For smaller tables:

* Table size: `40 MB`
* Bloat: `50%`
* Bloat space: approximately `20 MB`

For larger tables:

* Minimum bloat: approximately `1 GB`
* Bloat percentage: approximately `25%`

Example:

* Table: `4 GB`
* Bloat: `1 GB`
* Bloat percentage: `25%`

These are operational examples for identifying possible VACUUM FULL candidates.

---

# 33. Online Alternative for Table Reorganization

The third-party utility mentioned in class was:

`pg_repack`

It can be used as an alternative where lower blocking is required compared with VACUUM FULL.

---

# 34. REINDEX

REINDEX is used when an index requires rebuilding.

Check for invalid indexes:

```sql
SELECT
    indexrelid,
    indrelid,
    indisvalid
FROM pg_index
WHERE indisvalid = 'f';
```

Class result:

`0 rows`

---

# 35. REINDEX a Table

Command:

```sql
REINDEX TABLE emp;
```

Concurrent version:

```sql
REINDEX TABLE CONCURRENTLY emp;
```

---

# 36. REINDEX a Database

Command:

```sql
REINDEX DATABASE postgres;
```

---

# 37. REINDEX a Specific Index

Create index:

```sql
CREATE INDEX emp_idx
ON emp(eno);
```

Normal rebuild:

```sql
REINDEX INDEX emp_idx;
```

Concurrent rebuild:

```sql
REINDEX INDEX CONCURRENTLY emp_idx;
```

---

# 38. VACUUM FULL and Indexes

Question:

**If EMP has an index and VACUUM FULL is performed, do we still need REINDEX?**

Answer:

Normally no.

VACUUM FULL rewrites the table and rebuilds its indexes.

---

# 39. ANALYZE Entire Database

Command:

```sql
ANALYZE;
```

---

# 40. VACUUM Entire Database

Command:

```sql
VACUUM;
```

---

# 41. VACUUM FULL Entire Database

Command:

```sql
VACUUM FULL;
```

---

# 42. Create Additional Tables for Maintenance Testing

Check tables:

```text
\dt
```

Create `dept`:

```sql
CREATE TABLE dept AS
SELECT * FROM emp;
```

Create `hr`:

```sql
CREATE TABLE hr AS
SELECT * FROM emp;
```

Create `news`:

```sql
CREATE TABLE news AS
SELECT * FROM emp;
```

Verify:

```text
\dt
```

Tables created:

* `emp`
* `dept`
* `hr`
* `news`

---

# 43. PostgreSQL Maintenance Utilities

Move to:

```bash
cd /usr/pgsql-18/bin/
```

List utilities:

```bash
ls -ltr
```

Important utilities include:

* `vacuumdb`
* `reindexdb`
* `psql`
* `pg_ctl`
* `pg_dump`
* `pg_restore`
* `pg_basebackup`

---

# 44. vacuumdb

`vacuumdb` provides command-line access to VACUUM and ANALYZE maintenance.

---

# 45. VACUUM All Databases

Command:

```bash
vacuumdb -a
```

It processed databases such as:

* `inc1`
* `inc2`
* `inc3`
* `mydb`
* `newdb`
* `postgres`
* `rajdb`
* `saidb`
* `template0`
* `template1`
* `testdb`

---

# 46. VACUUM a Single Database

Command:

```bash
vacuumdb -d postgres
```

---

# 47. VACUUM Specific Tables

Command:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept
```

---

# 48. VACUUM + ANALYZE

Command:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-z
```

`-z` means VACUUM followed by ANALYZE.

---

# 49. ANALYZE Only

Command:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-Z
```

Capital `-Z` means ANALYZE only.

---

# 50. VACUUM FULL Using vacuumdb

Command:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-f
```

`-f` means VACUUM FULL.

---

# 51. REINDEX Command-Line Utility

The utility is:

`/usr/pgsql-18/bin/reindexdb`

This is used for command-line index maintenance.

---

# 52. AUTOVACUUM

PostgreSQL provides autovacuum for automatic VACUUM and ANALYZE maintenance.

Check:

```sql
SHOW autovacuum;
```

Default class value:

`on`

---

# 53. Autovacuum Architecture

**PostgreSQL Server**

↓

**Autovacuum Launcher**

↓

**Autovacuum Workers**

↓

**VACUUM / ANALYZE Eligible Tables**

---

# 54. Check Autovacuum Settings

Query:

```sql
SELECT
    name,
    setting,
    short_desc
FROM pg_settings
WHERE name LIKE '%autovacuum%';
```

Important settings reviewed:

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

Class values included:

* Analyze scale factor: `0.1`
* Analyze threshold: `50`
* Vacuum scale factor: `0.2`
* Vacuum threshold: `50`
* Freeze max age: `200000000`
* Max workers: `3`
* Naptime: `60`

---

# 55. Autovacuum VACUUM Threshold

Class explanation:

`50 updates/deletes + 20% growth`

Simplified formula:

`autovacuum_vacuum_threshold`

*

`autovacuum_vacuum_scale_factor × reltuples`

---

# 56. Auto-Analyze Threshold

Class explanation:

`50 changes + 10% growth`

Simplified formula:

`autovacuum_analyze_threshold`

*

`autovacuum_analyze_scale_factor × reltuples`

---

# 57. autovacuum_naptime

Class setting:

`autovacuum_naptime = 60`

---

# 58. autovacuum_max_workers

Class setting:

`autovacuum_max_workers = 3`

---

# 59. autovacuum_freeze_max_age

Class setting:

`autovacuum_freeze_max_age = 200000000`

---

# 60. Temporarily Disable Autovacuum for Lab Testing

Check:

```sql
SHOW autovacuum;
```

Move to data directory:

```bash
cd /var/lib/pgsql/18/data/
```

Edit configuration:

```bash
vi postgresql.conf
```

Set:

```text
autovacuum = off
```

Reload:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data/ reload
```

Verify processes:

```bash
ps -ef | grep postgres
```

Verify in SQL:

```sql
SHOW autovacuum;
```

Result:

`off`

---

# 61. Enable Autovacuum Again

Move to:

```bash
cd /var/lib/pgsql/18/data/
```

Edit:

```bash
vi postgresql.conf
```

Set:

```text
autovacuum = on
```

Reload:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data/ reload
```

Verify:

```bash
ps -ef | grep postgres
```

The process list later showed:

`postgres: autovacuum launcher`

---

# 62. Monitor Autovacuum Activity

Query:

```sql
SELECT
    relname,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables;
```

---

# 63. Autovacuum Test

Check time:

```sql
SELECT now();
```

Check EMP row count:

```sql
SELECT count(*) FROM emp;
```

Result:

`2500000`

Update all rows:

```sql
UPDATE emp
SET eno = 100000;
```

Result:

`UPDATE 2500000`

Check maintenance timestamps:

```sql
SELECT
    relname,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables;
```

Check again later:

```sql
SELECT
    relname,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables;
```

For `emp`, both values were populated.

Check current time:

```sql
SELECT now();
```

---

# 64. Autovacuum on Busy Tables

Busy tables need careful monitoring and may require manual maintenance planning when automatic maintenance cannot keep up.

---

# 65. Should We Rely on Autovacuum?

Autovacuum should be monitored, and manual VACUUM and ANALYZE can be planned where required.

---

# 66. Manual Maintenance Strategy

Class scheduling examples:

* Tables below `5 GB` – daily VACUUM + ANALYZE
* Tables above `5 GB` and below `100 GB` – weekly
* Tables above `100 GB` – monthly

These were operational scheduling examples from the class.

---

# 67. Scheduling Maintenance Using Cron

Check existing cron entries:

```bash
crontab -l
```

Initial result:

`no crontab for postgres`

Edit:

```bash
crontab -e
```

Schedule:

```text
0 1 * * * vacuumdb -d postgres -t emp -t hr -t dept -z
```

Verify:

```bash
crontab -l
```

Result:

```text
0 1 * * * vacuumdb -d postgres -t emp -t hr -t dept -z
```

---

# 68. Cron Format

Cron format:

```text
Minute  Hour  Day  Month  Weekday
```

Ranges:

```text
Minute   : 0-59
Hour     : 0-23
Day      : 1-31
Month    : 1-12
Weekday  : 0-6
```

Example:

```text
0 1 * * *
```

Means:

**Every day at 1:00 AM**

---

# 69. Every Monday Example

Example:

```text
0 0 * * MON
```

Means:

**Every Monday at midnight**

---

# 70. Maintenance Cron Command

The maintenance command used was:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-z
```

---

# 71. Maintenance Decision Guide

### Need Updated Statistics?

Use:

```sql
ANALYZE;
```

### Need Dead Tuple Cleanup?

Use:

```sql
VACUUM;
```

### Need Physical Space Reclamation?

Use:

```sql
VACUUM FULL;
```

### Need to Rebuild an Index?

Use:

```sql
REINDEX;
```

### Need Automatic Routine Maintenance?

Use:

`AUTOVACUUM`

---

# 72. Quick Comparison

| Activity    | Main Purpose          | Shrinks Table | Main Usage         |
| ----------- | --------------------- | ------------- | ------------------ |
| ANALYZE     | Update statistics     | No            | Routine            |
| VACUUM      | Clean dead tuples     | Normally No   | Routine            |
| VACUUM FULL | Rewrite table         | Yes           | Maintenance window |
| REINDEX     | Rebuild indexes       | Index only    | As required        |
| AUTOVACUUM  | Automatic maintenance | Normally No   | Continuous         |

---

# 73. Useful Monitoring Queries

### Dead Tuples

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables;
```

### Manual Maintenance History

```sql
SELECT
    relname,
    last_vacuum,
    last_analyze,
    n_dead_tup
FROM pg_stat_user_tables;
```

### Autovacuum History

```sql
SELECT
    relname,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables;
```

### Autovacuum Settings

```sql
SELECT
    name,
    setting,
    short_desc
FROM pg_settings
WHERE name LIKE '%autovacuum%';
```

### Relation File

```sql
SELECT pg_relation_filepath('emp');
```

### Table Size

```text
\dt+ emp
```

---

# 74. Important Production Points

* Keep autovacuum enabled in normal environments.
* Monitor dead tuples.
* Keep statistics current.
* Use normal VACUUM for regular dead-tuple cleanup.
* Use VACUUM FULL when physical table shrinkage is required.
* Plan VACUUM FULL during an appropriate maintenance period.
* Check available storage before rewriting large tables.
* Use REINDEX for index rebuilding.
* Use concurrent REINDEX where reduced blocking is required.
* Use `vacuumdb` for shell-based maintenance.
* Use cron when scheduled manual maintenance is required.

---

# 75. Complete Maintenance Flow

**Monitor Database**

↓

**Check Statistics**

↓

`ANALYZE`

↓

**Check Dead Tuples**

↓

`VACUUM`

↓

**Check Bloat**

↓

`VACUUM FULL` if required

↓

**Check Indexes**

↓

`REINDEX` if required

↓

**Monitor AUTOVACUUM**

↓

**Schedule Additional Maintenance if Required**

---

# 76. Final Class Summary

### ANALYZE

Update optimizer statistics.

```sql
ANALYZE emp;
```

### VACUUM

Clean dead tuples and make their space reusable.

```sql
VACUUM VERBOSE emp;
```

### VACUUM FULL

Rewrite the table and reclaim physical storage.

```sql
VACUUM FULL emp;
```

### REINDEX

Rebuild indexes.

```sql
REINDEX TABLE emp;
```

```sql
REINDEX TABLE CONCURRENTLY emp;
```

```sql
REINDEX DATABASE postgres;
```

```sql
REINDEX INDEX emp_idx;
```

```sql
REINDEX INDEX CONCURRENTLY emp_idx;
```

### AUTOVACUUM

Check:

```sql
SHOW autovacuum;
```

Monitor:

```sql
SELECT
    relname,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables;
```

### vacuumdb

All databases:

```bash
vacuumdb -a
```

Single database:

```bash
vacuumdb -d postgres
```

Specific tables:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept
```

VACUUM + ANALYZE:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-z
```

ANALYZE only:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-Z
```

VACUUM FULL:

```bash
vacuumdb -d postgres \
-t emp \
-t hr \
-t dept \
-f
```

Cron example:

```text
0 1 * * * vacuumdb -d postgres -t emp -t hr -t dept -z
```

This version keeps the same class-note structure and content, with all references made neutral and without mentioning any trainer. 
