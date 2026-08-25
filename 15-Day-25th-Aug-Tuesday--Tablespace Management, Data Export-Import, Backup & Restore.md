Note: Below PostgreSQL technical points are verified at PostgreSQL 18 official documentation.

Cover tablespaces, `COPY`, logical backup/restore, physical backup, and incremental backups.   

# PostgreSQL Class Notes

## Tablespace Management, Data Export/Import, Backup & Restore

**Date: 25 August 2026**

**PostgreSQL Version Used in Examples: PostgreSQL 18**

---

# PART 1 — TABLESPACE MANAGEMENT

## 1. What is a Tablespace?

A **tablespace** in PostgreSQL is a named location in the file system where PostgreSQL can store the physical files belonging to database objects.

In simple terms:

**Tablespace Name → File-System Directory → Disk / Mount Point / Storage Volume**

Example:

```text
ram_tbs
   |
   +-- /var/lib/pgsql/tbs
             |
             +-- EBS / SAN / Local Disk / Mounted Volume
```

Once the tablespace is created, DBAs and users normally refer to the **tablespace name**, rather than repeatedly specifying the physical path.

Example:

```sql
CREATE TABLE emp (
    eno   int,
    ename varchar(50)
) TABLESPACE ram_tbs;
```

PostgreSQL supports placing objects such as:

* Databases
* Tables
* Indexes
* Materialized views
* Some constraint indexes

in different tablespaces.

### Important

A PostgreSQL tablespace is **not an independent storage unit** that can be detached and attached to another PostgreSQL cluster.

It is an integral part of the PostgreSQL cluster.

---

# 2. Why Do We Use Tablespaces?

## Advantage 1 — Storage Management

Suppose:

```text
PostgreSQL DATA disk = 100 GB
Used                  = 90 GB
Free                  = 10 GB
```

The storage team says:

```text
Existing disk cannot be extended.
New 500 GB disk can be provided.
```

Instead of moving the complete PostgreSQL cluster:

1. Mount the new 500 GB disk.
2. Create an empty directory owned by `postgres`.
3. Create a PostgreSQL tablespace using that directory.
4. Move selected large tables or indexes to the new tablespace.

Example:

```text
Before:

100 GB DATA disk
|
+-- 90 GB used


After:

100 GB DATA disk          New 500 GB disk
|                         |
+-- PostgreSQL DATA       +-- large_tbs
    ~10 GB                    |
                              +-- 80/90 GB large table
```

This is one of the major practical uses of tablespaces.

---

# 3. Tablespaces and Performance

Tablespaces can also help with I/O distribution.

For example:

```text
Frequently accessed index
        |
        +----> Fast SSD tablespace

Archive table
        |
        +----> Slower / cheaper disk tablespace
```

This can improve performance **when the underlying storage devices have different performance characteristics or independent I/O paths**.

Simply creating multiple directories on the **same disk** does not automatically improve performance.

---

# 4. Tablespaces and Security

Different storage locations can have different operating-system-level controls, mount options, encryption, backup policies, and storage policies.

However, a PostgreSQL tablespace itself should not be treated as a security boundary.

PostgreSQL permissions and operating-system permissions must still be configured correctly.

---

# 5. Important Correction — Tablespaces Do NOT Remove Single Point of Failure

It is incorrect to say:

```text
Multiple tablespaces avoid a single point of failure.
```

A tablespace is still part of the same PostgreSQL cluster.

If a required tablespace disk is lost, PostgreSQL objects stored there become unavailable, and the cluster can potentially become unusable.

Therefore:

**Tablespace ≠ High Availability**

High availability should be implemented using mechanisms such as:

* Streaming replication
* Standby servers
* Storage redundancy
* Backups
* WAL archiving
* Appropriate HA architecture

---

# 6. Default PostgreSQL Tablespaces

PostgreSQL creates two tablespaces automatically:

```text
pg_default
pg_global
```

Check them using:

```sql
\db
```

Typical output:

```text
    Name    | Owner    | Location
------------+----------+----------
 pg_default | postgres |
 pg_global  | postgres |
```

## pg_default

`pg_default` is the default tablespace for normal database objects unless another tablespace is selected.

## pg_global

`pg_global` is used for shared system catalogs.

It is not used for normal application tables.

---

# 7. Creating a User Tablespace

First create the directory at OS level.

Example:

```bash
mkdir /var/lib/pgsql/tbs
chown postgres:postgres /var/lib/pgsql/tbs
chmod 700 /var/lib/pgsql/tbs
```

The directory should:

* Already exist
* Normally be empty
* Be owned by the PostgreSQL operating-system user
* Use an absolute path
* Be on reliable persistent storage

Then create the tablespace:

```sql
CREATE TABLESPACE ram_tbs
LOCATION '/var/lib/pgsql/tbs';
```

Verify:

```sql
\db
```

or:

```sql
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

---

# 8. Who Can Create a Tablespace?

Creating a tablespace requires a PostgreSQL **superuser**.

Example:

```sql
CREATE TABLESPACE ram_tbs
LOCATION '/var/lib/pgsql/tbs';
```

A superuser can later grant another user permission to create objects in it:

```sql
GRANT CREATE ON TABLESPACE ram_tbs TO raj;
```

---

# 9. pg_tblspc Directory

Custom tablespaces are represented inside:

```text
$PGDATA/pg_tblspc/
```

For example:

```bash
cd /var/lib/pgsql/18/data/pg_tblspc
ls -ltr
```

You might see:

```text
16495 -> /var/lib/pgsql/tbs
```

This is a **symbolic link**.

Conceptually:

```text
$PGDATA/pg_tblspc/16495
             |
             +----> /var/lib/pgsql/tbs
```

`16495` is the tablespace OID in this example.

We can check it from PostgreSQL:

```sql
SELECT oid, spcname
FROM pg_tablespace;
```

---

# 10. Version-Specific Directory Inside a Tablespace

Inside the tablespace directory PostgreSQL creates a version-specific directory.

Example:

```text
/var/lib/pgsql/tbs/
    |
    +-- PG_18_202506291/
```

Under that, PostgreSQL creates directories for databases that have objects stored in the tablespace.

These directories and file names are PostgreSQL internal implementation details.

**DBAs should not manually create, rename, move, or delete PostgreSQL relation files.**

---

# 11. Creating a Database in a Tablespace

Example:

```sql
CREATE DATABASE testdb
TABLESPACE ram_tbs;
```

Check:

```sql
\l+ testdb
```

The database's tablespace becomes its default storage location.

Conceptually:

```text
testdb
   |
   +-- Default tablespace = ram_tbs
```

---

# 12. Database Tablespace Does Not Force Every Object Into It

Suppose:

```text
testdb default tablespace = ram_tbs
```

Create:

```sql
CREATE TABLE emp (
    eno int
);
```

Because no tablespace was explicitly specified, PostgreSQL normally uses the database's default tablespace.

Now create:

```sql
CREATE TABLE dept (
    dno int
) TABLESPACE pg_default;
```

Now:

```text
testdb
|
+-- emp  ---> ram_tbs
|
+-- dept ---> pg_default
```

Therefore:

> A database can have objects stored in multiple tablespaces.

---

# 13. Finding the Physical File of a Table

Use:

```sql
SELECT pg_relation_filepath('emp');
```

Example:

```text
pg_tblspc/16495/PG_18_202506291/16496/16497
```

For an object in `pg_default`:

```sql
SELECT pg_relation_filepath('dept');
```

Example:

```text
base/16496/16503
```

This is useful for understanding PostgreSQL physical storage.

However, DBAs should not directly manipulate these files.

---

# 14. Moving a Table to Another Tablespace

Syntax:

```sql
ALTER TABLE table_name
SET TABLESPACE tablespace_name;
```

Example:

```sql
ALTER TABLE dept
SET TABLESPACE ram_tbs;
```

PostgreSQL physically rewrites/moves the table to the destination tablespace.

---

# 15. Locking While Moving a Table

`ALTER TABLE ... SET TABLESPACE` requires a strong lock on the table.

Therefore, applications using that table can be blocked while the move is taking place.

For a very large table, the operation can take significant time.

Example:

```text
500 GB table
     |
     +--> ALTER TABLE ... SET TABLESPACE
                     |
                     +--> Data physically copied
                     +--> Strong table lock
                     +--> Additional I/O
```

### Production Recommendation

Before moving a large table:

* Estimate its size.
* Check free destination space.
* Estimate maintenance window.
* Check dependent indexes.
* Monitor disk I/O.
* Plan for application impact.
* Test the operation in a lower environment.

Do not simply assume every tablespace change requires a complete database outage. The actual impact depends on the object and command, but moving a busy large table can effectively require a maintenance window.

---

# 16. Moving an Entire Database

Example:

```sql
ALTER DATABASE rajdb
SET TABLESPACE ram_tbs;
```

You cannot perform this while connected to the database being moved.

For example:

```text
testdb=# ALTER DATABASE testdb
         SET TABLESPACE pg_default;

ERROR:
cannot change the tablespace of the currently open database
```

Connect to another database:

```sql
\c postgres
```

Then:

```sql
ALTER DATABASE testdb
SET TABLESPACE pg_default;
```

The target database must not have conflicting active sessions during this operation.

---

# 17. Important Database Tablespace Scenario

Suppose:

```text
testdb default tablespace = ram_tbs
```

But:

```text
dept table = pg_default
```

Now we try:

```sql
ALTER DATABASE testdb
SET TABLESPACE pg_default;
```

PostgreSQL can reject the operation because relations belonging to the database are already explicitly stored in the destination tablespace.

Typical error:

```text
ERROR: some relations of database "testdb"
       are already in tablespace "pg_default"

HINT: You must move them back to the database's
      default tablespace before using this command.
```

Solution:

First connect to `testdb`:

```sql
\c testdb
```

Move the table back to the current database default:

```sql
ALTER TABLE dept
SET TABLESPACE ram_tbs;
```

Then connect elsewhere:

```sql
\c postgres
```

Move the database:

```sql
ALTER DATABASE testdb
SET TABLESPACE pg_default;
```

---

# 18. Dropping a Tablespace

A tablespace must be empty before it can be dropped.

Example:

```sql
DROP TABLESPACE ram_tbs;
```

If objects still exist in it, PostgreSQL will reject the operation.

Remember that a tablespace is cluster-wide and can contain objects belonging to multiple databases.

Therefore, check all databases before dropping it.

---

# 19. Tablespace Limitations and Important Rules

1. A single relation is stored in one tablespace at a time.

2. A PostgreSQL tablespace maps to one directory location.

3. One tablespace can contain many database objects.

4. The same tablespace can be used by multiple databases in the cluster.

5. PostgreSQL does not use Oracle-style tablespace datafiles.

6. PostgreSQL does not define a tablespace `MAXSIZE` in the Oracle sense.

7. Available capacity is mainly controlled by the underlying file system/storage.

8. A tablespace must be empty before `DROP TABLESPACE`.

9. Moving large relations can cause significant locking and I/O.

10. Tablespaces are part of the PostgreSQL cluster and must be included in the backup/recovery design.

11. Do not manually copy a tablespace directory and attempt to attach it to another cluster.

---

# 20. PostgreSQL vs Oracle Tablespace

| Oracle                                       | PostgreSQL                                                      |
| -------------------------------------------- | --------------------------------------------------------------- |
| Tablespace consists of one or more datafiles | Tablespace points to a filesystem directory                     |
| Datafiles can be added to a tablespace       | No Oracle-style `ADD DATAFILE` concept                          |
| Datafile sizes/max sizes can be managed      | Capacity mainly comes from underlying filesystem/storage        |
| Oracle manages tablespace datafiles          | PostgreSQL stores relation files under the tablespace directory |

A simple way to remember:

```text
Oracle:
Tablespace
   |
   +-- Datafile 1
   +-- Datafile 2
   +-- Datafile 3


PostgreSQL:
Tablespace name
   |
   +-- Filesystem directory
             |
             +-- PostgreSQL relation files
```

---

# 21. Correction — PostgreSQL Upgrade and Tablespaces

It is **not a general PostgreSQL rule** that all databases must be moved back to `pg_default` before every upgrade.

Major-version upgrade procedures must account for custom tablespaces, and the exact method depends on the upgrade strategy.

For example:

* `pg_upgrade`
* Logical migration
* Dump and restore
* Replication-based migration

Therefore, the correct statement is:

> Custom tablespaces make upgrade planning more important, but PostgreSQL does not generally require every database to be moved to `pg_default` before an upgrade.

---

# 22. Correction — Backup Automation with Tablespaces

Tablespaces do **not** make automated full backups impossible.

PostgreSQL backup tools understand tablespaces.

However, custom tablespaces make the physical storage layout more complex, so backup, restore, disaster recovery, monitoring, and migration procedures must account for them.

---

# PART 2 — DATA EXPORT AND IMPORT USING COPY

# 23. What is COPY?

`COPY` transfers **table data** between PostgreSQL and files/streams.

It is mainly a bulk data movement mechanism.

It does not create the destination table structure for you.

Main operations:

```text
Table ---> File
       COPY TO

File ---> Table
       COPY FROM
```

---

# 24. Create Sample Table

```sql
CREATE TABLE emp (
    eno   int,
    ename varchar
);
```

Insert data:

```sql
INSERT INTO emp
VALUES
    (1,'ram'),
    (2,'sam');
```

Check:

```sql
SELECT * FROM emp;
```

Output:

```text
 eno | ename
-----+-------
 1   | ram
 2   | sam
```

---

# 25. Export Table Data

```sql
COPY emp
TO '/tmp/emp.txt';
```

Output:

```text
COPY 2
```

OS:

```bash
cat /tmp/emp.txt
```

Output:

```text
1    ram
2    sam
```

---

# 26. Import Table Data

```sql
COPY emp
FROM '/tmp/emp.txt';
```

Important:

`COPY FROM` **appends rows** to the table.

It does not automatically truncate existing data.

Therefore, if two rows already exist:

```text
Before COPY = 2 rows
Imported    = 2 rows
After COPY  = 4 rows
```

---

# 27. COPY with Delimiter

Export:

```sql
COPY emp
TO '/tmp/emp.txt'
WITH (DELIMITER '|');
```

File:

```text
1|ram
2|sam
```

Import:

```sql
COPY emp
FROM '/tmp/emp.txt'
WITH (DELIMITER '|');
```

The delimiter used during import must match the file format.

---

# 28. COPY with Header

A cleaner approach is to use CSV format:

```sql
COPY emp
TO '/tmp/emp.csv'
WITH (
    FORMAT CSV,
    HEADER,
    DELIMITER '|'
);
```

Output:

```text
eno|ename
1|ram
2|sam
```

Import:

```sql
COPY emp
FROM '/tmp/emp.csv'
WITH (
    FORMAT CSV,
    HEADER,
    DELIMITER '|'
);
```

### Important Correction

If the file contains:

```text
eno|ename
1|ram
2|sam
```

and you import it without telling PostgreSQL that the first row is a header, PostgreSQL can try to insert `eno` into the integer column and fail.

Therefore, the import options must match the exported format.

---

# 29. Exporting Query Results

`COPY TO` can export the result of a query.

Example:

```sql
COPY (
    SELECT *
    FROM emp
    WHERE eno <= 1
)
TO '/tmp/emp.csv'
WITH (
    FORMAT CSV,
    HEADER,
    DELIMITER '|'
);
```

This is useful when developers request only selected data.

---

# 30. Can EMP Data Be Loaded Into DEPT?

Suppose:

```text
EMP

eno INTEGER
ename VARCHAR
```

and:

```text
DEPT

dno INTEGER
dname VARCHAR
```

The column names do not have to be identical.

The incoming values must be compatible with the destination columns, and the column order/count must match the `COPY FROM` target column list.

Example:

```sql
COPY dept(dno,dname)
FROM '/tmp/emp.csv'
WITH (
    FORMAT CSV,
    HEADER,
    DELIMITER '|'
);
```

This can work because:

```text
eno   -> dno
ename -> dname
```

have compatible values/data types.

---

# 31. COPY vs \copy

This is an important DBA interview topic.

## COPY

```sql
COPY emp
TO '/tmp/emp.csv'
WITH (FORMAT CSV, HEADER);
```

The file is handled by the **PostgreSQL server**.

Therefore:

```text
PostgreSQL Server
      |
      +--> /tmp/emp.csv
```

The PostgreSQL server process must have permission to access the file.

Server-side file `COPY` also requires appropriate PostgreSQL privileges.

## \copy

From `psql`:

```sql
\copy emp TO '/home/raj/emp.csv' WITH (FORMAT CSV, HEADER)
```

Here the **psql client** reads/writes the file.

```text
PostgreSQL Server
       |
       | network
       |
psql Client
       |
       +--> /home/raj/emp.csv
```

This is especially useful when the database is remote.

---

# 32. COPY Additional Points

PostgreSQL 18 supports options such as:

```text
FORMAT
DELIMITER
NULL
HEADER
QUOTE
ESCAPE
ENCODING
ON_ERROR
REJECT_LIMIT
```

`COPY` progress can also be monitored using:

```sql
SELECT *
FROM pg_stat_progress_copy;
```

---

# PART 3 — BACKUP AND RESTORE

# 33. What is a Backup?

A backup is a copy of database information that can be used for recovery, migration, refresh, or other administrative purposes.

Common purposes:

* Data protection
* Disaster recovery
* Migration
* Database refresh
* Recovery from accidental object deletion
* Recovery from corruption
* Testing
* Development environment refresh

---

# 34. What is Restore?

Restore means rebuilding or recovering database information from a previously created backup.

Examples:

```text
Production backup
       |
       +--> Restore into DEV

Production backup
       |
       +--> Restore after accidental table deletion
```

Point-in-time recovery is different from simply replaying a logical SQL dump; PITR normally uses a physical base backup plus WAL.

---

# 35. PostgreSQL Backup Categories

At a high level:

```text
PostgreSQL Backup
       |
       +--------------------+
       |                    |
 Logical Backup       Physical Backup
       |                    |
 pg_dump              pg_basebackup
 pg_dumpall           Base backup + WAL
```

---

# PART 4 — LOGICAL BACKUP

# 36. What is a Logical Backup?

A logical backup represents database objects and/or their data logically rather than copying PostgreSQL's physical relation files.

Logical backups are generally more portable than physical backups.

They can be used for:

* Individual tables
* Multiple tables
* Schemas
* Multiple schemas
* Individual databases
* Migration between supported PostgreSQL versions
* Selective restore when using archive formats

---

# 37. pg_dump Output Formats

`pg_dump` supports:

```text
Plain
Custom
Directory
Tar
```

Common format options:

```bash
-F p    Plain
-F c    Custom
-F d    Directory
-F t    Tar
```

---

# 38. Plain SQL Format

Example:

```bash
pg_dump -d testdb -F p -f testdb.sql
```

The output is a text SQL script.

It can be viewed using:

```bash
less testdb.sql
```

Restore:

```bash
psql -d testdb_restore -f testdb.sql
```

---

# 39. Custom Format

Example:

```bash
pg_dump -d testdb -F c -f testdb.dump
```

Restore:

```bash
pg_restore -d testdb_restore testdb.dump
```

Custom format is an archive format understood by `pg_restore`.

Benefits include:

* Selective restore
* Reordering restore items
* Compression support
* Parallel restore support where applicable
* Flexible object selection

### Important Correction

Custom format should **not automatically be described as encrypted or more secure**.

If backup encryption is required, it must be implemented separately using appropriate security/storage mechanisms.

---

# 40. Directory Format

Example:

```bash
pg_dump -d testdb -F d -f /backup/testdb_dump
```

Directory format is useful for parallel dump and restore.

Example:

```bash
pg_dump -Fd -j 4 -d testdb -f /backup/testdb_dump
```

Restore:

```bash
pg_restore -j 4 \
    -d testdb_restore \
    /backup/testdb_dump
```

---

# 41. Tar Format

Example:

```bash
pg_dump -Ft -d testdb -f testdb.tar
```

Restore:

```bash
pg_restore -d testdb_restore testdb.tar
```

---

# 42. pg_dump

`pg_dump` works on **one database at a time**.

It can back up:

* Entire database
* One table
* Multiple tables
* One schema
* Multiple schemas
* Schema only
* Data only

Example database backup:

```bash
pg_dump -Fc -d testdb -f testdb.dump
```

Single table:

```bash
pg_dump -Fc \
    -d testdb \
    -t public.emp \
    -f emp.dump
```

Multiple tables:

```bash
pg_dump -Fc \
    -d testdb \
    -t public.emp \
    -t public.dept \
    -f emp_dept.dump
```

Schema:

```bash
pg_dump -Fc \
    -d testdb \
    -n sales \
    -f sales.dump
```

---

# 43. pg_dumpall

`pg_dumpall` creates a logical script representing all databases in a PostgreSQL cluster, together with cluster-wide objects such as roles and tablespaces.

Example:

```bash
pg_dumpall > cluster.sql
```

Restore:

```bash
psql -f cluster.sql postgres
```

`pg_dumpall` produces a **plain-text SQL script**, not a custom `pg_restore` archive.

---

# 44. pg_dump vs pg_dumpall

| Feature                                   | pg_dump | pg_dumpall                              |
| ----------------------------------------- | ------- | --------------------------------------- |
| One database                              | Yes     | Included as part of cluster dump        |
| Single table                              | Yes     | Not its normal selective-backup purpose |
| Multiple tables                           | Yes     | Not its normal selective-backup purpose |
| Single schema                             | Yes     | Not its normal selective-backup purpose |
| Multiple schemas                          | Yes     | Not its normal selective-backup purpose |
| Multiple databases in one pg_dump command | No      | Yes                                     |
| Cluster-wide roles                        | No      | Yes                                     |
| Tablespace definitions                    | No      | Yes                                     |
| Plain format                              | Yes     | Yes                                     |
| Custom format                             | Yes     | No                                      |
| Directory format                          | Yes     | No                                      |
| Tar format                                | Yes     | No                                      |

---

# 45. Locking During pg_dump

A common statement is:

```text
pg_dump does not lock tables.
```

This needs clarification.

`pg_dump` takes locks that prevent conflicting schema changes while the dump is running.

However, it does **not normally block ordinary SELECT/INSERT/UPDATE/DELETE operations**.

Therefore, a better statement is:

> `pg_dump` creates a transactionally consistent backup while normal database activity can usually continue, but it does take lightweight/access-share locks to protect the objects being dumped from conflicting DDL.

---

# PART 5 — LOGICAL RESTORE

# 46. psql

Use `psql` for plain SQL dumps.

Example:

```bash
psql -d testdb_restore -f testdb.sql
```

---

# 47. pg_restore

Use `pg_restore` for archive formats created by `pg_dump`, including:

```text
Custom
Directory
Tar
```

Example:

```bash
pg_restore \
    -d testdb_restore \
    testdb.dump
```

---

# 48. Selective Restore

Suppose a custom backup contains:

```text
emp
dept
orders
customers
...
```

Restore only `emp`:

```bash
pg_restore \
    -d testdb_restore \
    -t emp \
    testdb.dump
```

Restore schema only:

```bash
pg_restore \
    --schema-only \
    -d testdb_restore \
    testdb.dump
```

Restore data only:

```bash
pg_restore \
    --data-only \
    -d testdb_restore \
    testdb.dump
```

---

# 49. Correction — "pg_restore is 10 Times Faster"

There is no PostgreSQL rule that:

```text
pg_restore is 10 times faster than psql
```

Performance depends on:

* Backup format
* Database size
* Number of objects
* Number of indexes
* Constraints
* Hardware
* CPU
* Storage performance
* Parallel jobs
* Configuration

A major advantage of archive formats is that `pg_restore` can perform **parallel restore** for supported formats.

Example:

```bash
pg_restore \
    -j 8 \
    -d testdb_restore \
    testdb.dump
```

So the correct statement is:

> `pg_restore` can provide much more flexible restoration and can improve performance using parallel restore, but there is no fixed "10x faster" guarantee.

---

# 50. Plain vs Custom/Archive Backup

## Plain

```text
pg_dump
   |
   +--> testdb.sql
             |
             +--> psql
```

Advantages:

* Human-readable
* Easy to inspect
* Easy to edit as SQL

## Custom

```text
pg_dump -Fc
      |
      +--> testdb.dump
                |
                +--> pg_restore
```

Advantages:

* Selective restore
* Flexible restore
* Compression
* Parallel restore support

For many production logical-backup requirements, custom or directory format is preferred.

But the correct format should be selected according to the restore requirement.

---

# PART 6 — PHYSICAL BACKUP

# 51. What is a Physical Backup?

A physical backup copies PostgreSQL cluster files at the physical level.

It contains the files needed to reconstruct the PostgreSQL cluster rather than SQL statements representing individual objects.

The standard PostgreSQL utility for taking an online base backup is:

```text
pg_basebackup
```

---

# 52. pg_basebackup

Example:

```bash
pg_basebackup \
    -D /backup/full \
    -Fp \
    -Xs \
    -P
```

`pg_basebackup` takes a base backup of a **running PostgreSQL cluster**.

Normal client activity can continue while the backup is running.

---

# 53. Physical Backup Scope

Physical backup operates at the **cluster level**.

It is not intended for backing up one individual table.

Example:

```text
PostgreSQL Cluster
      |
      +-- database1
      +-- database2
      +-- database3
      +-- system catalogs
      +-- tablespaces
      |
      +----> Physical Base Backup
```

For object-level backup, use logical tools such as `pg_dump`.

---

# 54. Physical Backup and PITR

Physical backups can be combined with archived WAL for:

**Point-In-Time Recovery (PITR)**.

Conceptually:

```text
Base Backup
    +
Archived WAL
    |
    v
Restore PostgreSQL
to a selected point in time
```

Example:

```text
10:00 AM ---- Base Backup
10:30 AM ---- Transactions
11:00 AM ---- Transactions
11:20 AM ---- User accidentally drops table
```

With a suitable backup and WAL archive, we might recover to:

```text
11:19 AM
```

before the unwanted operation.

---

# 55. Physical Backup Compatibility

Physical backups are tightly coupled to the PostgreSQL server's physical storage format.

A physical base backup should not be treated like a logical dump that can simply be restored into a different major PostgreSQL version.

For major-version migration, normally use:

* `pg_upgrade`
* Logical dump/restore
* Logical replication
* Other supported migration strategies

depending on the requirement.

---

# PART 7 — FULL AND INCREMENTAL PHYSICAL BACKUPS

# 56. PostgreSQL Incremental Backup

PostgreSQL 17 introduced built-in support for incremental base backups.

Therefore:

```text
PostgreSQL <= 16
Built-in pg_basebackup incremental backup: No

PostgreSQL >= 17
Built-in pg_basebackup incremental backup: Yes
```

---

# 57. Full Backup

Example:

```text
Sunday

Full Backup
    |
    +--> 1 TB
```

A full backup contains the complete required cluster backup contents.

---

# 58. Incremental Backup Concept

After a full backup:

```text
Sunday
FULL
  |
  +--> Monday Incremental
  |
  +--> Tuesday Incremental
  |
  +--> Wednesday Incremental
```

Incremental backups avoid storing every unchanged block again.

They contain information representing blocks changed relative to a reference backup.

---

# 59. Important Correction — Incremental Backup Does Not Simply Compare Transaction Logs

The statement:

```text
Incremental backup compares transaction logs
and backs up the transactions.
```

is not technically accurate.

PostgreSQL incremental backup uses **WAL summaries** to identify changed blocks.

Therefore, think of it as:

```text
WAL activity
     |
     v
WAL summaries
     |
     v
Identify changed data blocks
     |
     v
Incremental base backup
```

It is a physical/block-level backup mechanism, not a logical backup of individual transactions.

---

# 60. Incremental Backup Example

Assume:

```text
Sunday Full Backup = 1 TB
```

After Sunday, only a relatively small part of the database changes.

Instead of copying the complete 1 TB again, an incremental backup can store the required changed blocks relative to the reference backup.

Conceptually:

```text
Sunday
FULL BACKUP
1 TB
   |
   +---- Monday Incremental
   |
   +---- Tuesday Incremental
   |
   +---- Wednesday Incremental
```

Actual incremental backup size depends on the blocks that changed, not simply on the amount of logical data inserted.

For example:

```text
1 GB INSERT
```

does **not guarantee**:

```text
Incremental backup = exactly 1 GB
```

because PostgreSQL works with physical blocks and other database changes may also occur.

---

# 61. pg_combinebackup

An incremental backup cannot simply be used by itself as though it were an independent full backup.

PostgreSQL provides:

```text
pg_combinebackup
```

to reconstruct a synthetic full backup from the incremental backup chain.

Conceptually:

```text
Full Backup
    +
Incremental 1
    +
Incremental 2
    +
Incremental 3
    |
    v
pg_combinebackup
    |
    v
Combined Full Backup
```

This combined backup can then be used as part of the recovery process.

---

# 62. Important Production Backup Tools

Remember these PostgreSQL utilities:

```text
pg_dump
    |
    +--> Logical backup of one database / selected objects

pg_dumpall
    |
    +--> Logical SQL dump of the complete cluster

psql
    |
    +--> Restore plain SQL dump

pg_restore
    |
    +--> Restore pg_dump archive formats

pg_basebackup
    |
    +--> Physical full/incremental base backup

pg_combinebackup
    |
    +--> Combine incremental backup chain

pg_verifybackup
    |
    +--> Verify a base backup
```

---

# 63. Logical Backup vs Physical Backup

| Feature                       | Logical                           | Physical                                         |
| ----------------------------- | --------------------------------- | ------------------------------------------------ |
| Main tools                    | pg_dump / pg_dumpall              | pg_basebackup                                    |
| SQL/object level              | Yes                               | No                                               |
| Physical files                | No                                | Yes                                              |
| Single table backup           | Yes with pg_dump                  | No                                               |
| Single schema backup          | Yes                               | No                                               |
| Single database backup        | Yes                               | Physical backup is cluster-level                 |
| Cluster backup                | pg_dumpall logically              | Yes                                              |
| Selective restore             | Excellent with pg_restore archive | Not object-level                                 |
| Cross-major-version migration | Common use case                   | No direct physical restore across major versions |
| PITR                          | Not by itself                     | Yes, with WAL                                    |
| Incremental built-in          | Not applicable in same sense      | PostgreSQL 17+                                   |
| Human-readable                | Plain dump can be                 | No                                               |

---

# 64. Important Corrections From the Original Class Notes

### Correction 1

Incorrect:

```text
Tablespaces avoid single point of failure.
```

Correct:

```text
Tablespaces control physical storage placement.
They do not provide high availability or remove
single points of failure.
```

### Correction 2

Incorrect:

```text
Before PostgreSQL upgrade all databases must
be moved to pg_default.
```

Correct:

```text
Not a general PostgreSQL requirement.
Custom tablespaces must be handled correctly
according to the chosen upgrade method.
```

### Correction 3

Incorrect:

```text
Full backup automation is not possible
with tablespaces.
```

Correct:

```text
Backup automation is possible.
Custom tablespaces simply need to be included
correctly in backup and recovery planning.
```

### Correction 4

Incorrect:

```text
pg_dump/pg_dumpall apply no locks.
```

Correct:

```text
Logical dumps are designed to allow normal database
activity, but pg_dump takes locks such as ACCESS SHARE
to protect dumped objects from conflicting DDL.
```

### Correction 5

Incorrect:

```text
pg_restore is always 10 times faster than psql.
```

Correct:

```text
There is no fixed performance ratio.
pg_restore supports features such as parallel restoration
that can make archive restores significantly faster.
```

### Correction 6

Incorrect:

```text
Custom format is preferred because it provides security.
```

Correct:

```text
Custom format provides flexible/selective restore,
compression and parallel restore capabilities.
It does not automatically provide encryption.
```

### Correction 7

Incorrect:

```text
Incremental backup simply backs up transactions
by comparing transaction logs.
```

Correct:

```text
PostgreSQL 17+ incremental backup uses WAL summaries
to identify changed blocks.
```

### Correction 8

Incorrect COPY example:

```sql
COPY dept
FROM '/tmp/emp.txt'
WITH (DELIMITER '|');
```

when the file contains:

```text
eno|ename
```

as the first line.

Correct:

```sql
COPY dept
FROM '/tmp/emp.txt'
WITH (
    FORMAT CSV,
    HEADER,
    DELIMITER '|'
);
```

---

# 65. Interview Quick Revision

## What is a tablespace?

A named PostgreSQL storage location that maps database objects to a filesystem directory.

## Where are custom tablespace links stored?

```text
$PGDATA/pg_tblspc
```

## Default tablespaces?

```text
pg_default
pg_global
```

## Can one tablespace contain multiple objects?

**Yes.**

## Can one database contain objects from multiple tablespaces?

**Yes.**

## Can a non-empty tablespace be dropped?

**No.**

## Does a tablespace provide HA?

**No.**

## Can a large table be moved to another tablespace?

**Yes.**

```sql
ALTER TABLE emp
SET TABLESPACE fast_tbs;
```

## How do we check a relation's physical path?

```sql
SELECT pg_relation_filepath('emp');
```

## Data export/import utility?

```text
COPY / \copy
```

## COPY vs \copy?

```text
COPY  = server-side file
\copy = psql/client-side file
```

## Logical backup utilities?

```text
pg_dump
pg_dumpall
```

## Plain SQL restore?

```text
psql
```

## Custom/directory/tar restore?

```text
pg_restore
```

## Physical backup?

```text
pg_basebackup
```

## PITR requires?

```text
Physical base backup + WAL
```

## PostgreSQL built-in incremental backup introduced?

```text
PostgreSQL 17
```

## Utility used with incremental backup chain?

```text
pg_combinebackup
```

## Backup verification utility?

```text
pg_verifybackup
```

---

# 66. Final Architecture View

```text
                   POSTGRESQL STORAGE
                          |
           +--------------+--------------+
           |                             |
       pg_default                  Custom Tablespace
                                         |
                                  /mnt/postgres/tbs
                                         |
                                  Separate Storage


                   POSTGRESQL BACKUP
                          |
             +------------+------------+
             |                         |
          Logical                    Physical
             |                         |
       +-----+------+             pg_basebackup
       |            |                   |
    pg_dump     pg_dumpall        +------+------+
       |            |             |             |
   pg_restore      psql          Full      Incremental
                                           PostgreSQL 17+
                                                |
                                         pg_combinebackup
```

## One-Line Memory Rule

**Tablespace = where PostgreSQL stores data.**

**COPY = move table data between PostgreSQL and files.**

**pg_dump = logical backup of a database or selected objects.**

**pg_dumpall = logical SQL dump of the cluster.**

**pg_restore = restore pg_dump archive backups.**

**pg_basebackup = physical cluster backup.**

**Base backup + WAL = PITR.**

**PostgreSQL 17+ = built-in incremental physical backup support.**

The key official-documentation corrections are important: PostgreSQL describes tablespaces as filesystem locations and explicitly warns that losing one can make the cluster unreadable, so tablespaces should not be presented as a way to eliminate a single point of failure. ([PostgreSQL][1])

For `COPY`, PostgreSQL confirms that `COPY FROM` appends to existing rows, server-side `COPY` accesses files from the database server, while psql `\copy` accesses files from the client side. ([PostgreSQL][2])

For backup/restore, `pg_restore` is specifically for non-plain `pg_dump` archives and supports selective restoration; there is no official "10 times faster" rule. ([PostgreSQL][3]) PostgreSQL 17 introduced incremental base backups, and PostgreSQL 18 continues to support full and incremental `pg_basebackup` backups, with incremental backups needing to be combined with their dependency chain using `pg_combinebackup` before they can be used as a full backup. ([PostgreSQL][4])

[1]: https://www.postgresql.org/docs/current/manage-ag-tablespaces.html "PostgreSQL: Documentation: 18: 22.6. Tablespaces"
[2]: https://www.postgresql.org/docs/18/sql-copy.html "PostgreSQL: Documentation: 18: COPY"
[3]: https://www.postgresql.org/docs/18/app-pgrestore.html "PostgreSQL: Documentation: 18: pg_restore"
[4]: https://www.postgresql.org/files/documentation/pdf/18/postgresql-18-US.pdf "PostgreSQL 18.1 Documentation"
