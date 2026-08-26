# PostgreSQL 18 – Backup and Restore (16-Day-26th-Aug-Wednesday)

## Class Notes with Real-Time Practice Scenarios

**Topic:** PostgreSQL Logical and Physical Backup & Restore
**Version:** PostgreSQL 18
**Level:** Beginner → Intermediate → Production DBA
**Focus:** `pg_dump`, `pg_restore`, `pg_dumpall`, `pg_basebackup`, `pg_verifybackup`, `pg_combinebackup`, WAL Archiving and Incremental Backup

In particular: 
tar dumps do not support compression; 
parallel `pg_dump` requires directory format and uses `jobs + 1` connections; 
parallel `pg_restore` works with custom/directory archives; 
`summarize_wal` is required for incremental physical backups; 
and dependent incremental backups must be combined oldest-to-newest. ([PostgreSQL][1])

---

# 1. What Is a Database Backup?

A database backup is a copy of database information that can be used to recover data when something goes wrong.

Typical reasons for taking backups:

* User accidentally deletes data.
* Table is accidentally dropped.
* Database is accidentally dropped.
* Server crashes.
* Storage fails.
* Database becomes corrupted.
* Application deployment damages data.
* Database must be migrated to another server.
* Database needs to be refreshed into QA/UAT.
* Disaster Recovery server must be built.
* Point-in-Time Recovery is required.

A DBA should always remember:

> Taking a backup is only half of the job. A backup is useful only when it can be restored successfully.

---

# 2. PostgreSQL Backup Types

PostgreSQL backup methods can mainly be divided into:

```text
PostgreSQL Backup
        |
        +--------------------+
        |                    |
   Logical Backup       Physical Backup
        |                    |
   pg_dump             pg_basebackup
   pg_dumpall          WAL Archiving
   pg_restore          Incremental Backup
        |                    |
   Object Level        Cluster Level
```

---

# 3. Logical Backup

Logical backup exports database objects and/or their data.

Main tools:

```text
pg_dump
pg_restore
pg_dumpall
psql
```

Logical backups are useful when we need:

* One table
* Multiple tables
* One schema
* Multiple schemas
* One database
* Selected database objects
* Migration between PostgreSQL servers
* Development/QA database refresh
* Cross-version logical migration

---

# 4. Physical Backup

Physical backup copies PostgreSQL cluster files.

Main utilities:

```text
pg_basebackup
pg_verifybackup
pg_combinebackup
```

Physical backup is mainly used for:

* Complete PostgreSQL cluster backup
* Disaster Recovery
* Physical standby creation
* Point-in-Time Recovery
* Full physical backup
* Incremental physical backup

Physical backup does **not** provide normal table-level or schema-level selection.

---

# 5. Important PostgreSQL Backup Utilities

| Requirement                    | Utility            |
| ------------------------------ | ------------------ |
| One table                      | `pg_dump`          |
| Multiple tables                | `pg_dump`          |
| One schema                     | `pg_dump`          |
| Multiple schemas               | `pg_dump`          |
| One database                   | `pg_dump`          |
| All databases logically        | `pg_dumpall`       |
| Roles and tablespaces          | `pg_dumpall -g`    |
| Restore plain SQL              | `psql`             |
| Restore custom archive         | `pg_restore`       |
| Restore directory archive      | `pg_restore`       |
| Restore tar archive            | `pg_restore`       |
| Physical cluster backup        | `pg_basebackup`    |
| Verify physical backup         | `pg_verifybackup`  |
| Reconstruct incremental backup | `pg_combinebackup` |

---

# 6. Lab Setup

Create a database:

```sql
CREATE DATABASE mydb;
```

Connect:

```text
\c mydb
```

Create schemas:

```sql
CREATE SCHEMA raj1;
CREATE SCHEMA raj2;
CREATE SCHEMA raj3;
```

---

# 7. Create Test Tables

Create tables under `raj1`:

```sql
CREATE TABLE raj1.emp
(
    eno   int,
    ename varchar
);

CREATE TABLE raj1.dept
(
    eno   int,
    ename varchar
);

CREATE TABLE raj1.hr
(
    eno   int,
    ename varchar
);
```

Create the same tables under `raj2`:

```sql
CREATE TABLE raj2.emp
(
    eno   int,
    ename varchar
);

CREATE TABLE raj2.dept
(
    eno   int,
    ename varchar
);

CREATE TABLE raj2.hr
(
    eno   int,
    ename varchar
);
```

Create the same tables under `raj3`:

```sql
CREATE TABLE raj3.emp
(
    eno   int,
    ename varchar
);

CREATE TABLE raj3.dept
(
    eno   int,
    ename varchar
);

CREATE TABLE raj3.hr
(
    eno   int,
    ename varchar
);
```

---

# 8. Verify Tables

```text
\dt raj1.*
\dt raj2.*
\dt raj3.*
```

Expected:

```text
raj1.dept
raj1.emp
raj1.hr

raj2.dept
raj2.emp
raj2.hr

raj3.dept
raj3.emp
raj3.hr
```

---

# 9. Insert Sample Data

```sql
INSERT INTO raj1.emp
VALUES
(1,'ram'),
(2,'sam');
```

Run again for testing:

```sql
INSERT INTO raj1.emp
VALUES
(1,'ram'),
(2,'sam');
```

Check:

```sql
SELECT count(*)
FROM raj1.emp;
```

Expected:

```text
4
```

Load data into other tables:

```sql
INSERT INTO raj1.dept
SELECT * FROM raj1.emp;

INSERT INTO raj1.hr
SELECT * FROM raj1.emp;

INSERT INTO raj2.dept
SELECT * FROM raj1.emp;

INSERT INTO raj2.hr
SELECT * FROM raj1.emp;

INSERT INTO raj3.dept
SELECT * FROM raj1.emp;

INSERT INTO raj3.hr
SELECT * FROM raj1.emp;
```

---

# 10. Create Backup Directory

As the `postgres` OS user:

```bash
mkdir -p /var/lib/pgsql/backup
```

Check:

```bash
ls -ld /var/lib/pgsql/backup
```

For a private backup directory:

```bash
chmod 0700 /var/lib/pgsql/backup
```

Move into it:

```bash
cd /var/lib/pgsql/backup
```

---

# 11. Scenario 1 – Single Table Backup in Plain Format

Requirement:

> Take complete backup of `raj1.emp`, including structure and data.

Command:

```bash
pg_dump \
  -d mydb \
  -U postgres \
  -t raj1.emp \
  -Fp \
  -f emp_backup.sql
```

Short version:

```bash
pg_dump -d mydb -t raj1.emp > emp_backup.sql
```

`-Fp` means:

```text
Plain Format
```

Plain is the default format.

---

# 12. Check Backup

```bash
ls -lh emp_backup.sql
```

View:

```bash
cat emp_backup.sql
```

The dump can contain:

```sql
CREATE TABLE raj1.emp
(
    eno integer,
    ename character varying
);
```

and:

```text
COPY raj1.emp (eno, ename) FROM stdin;
1    ram
2    sam
1    ram
2    sam
\.
```

Therefore, the dump contains:

```text
Structure
+
Data
```

---

# 13. Security of Plain SQL Backup

Plain SQL dumps are human-readable.

Anyone who can read the file may be able to see:

* Customer information
* Employee information
* Application data
* Business data
* Object definitions

Do not say:

> Plain backup is always a security violation.

That statement is too broad.

The correct statement is:

> Plain backups can expose sensitive information if backup files are not properly protected.

Production controls should include:

* Restricted file permissions
* Restricted backup directory
* Encryption at rest
* Encryption in transit
* Controlled backup accounts
* Backup retention policy
* Audit access
* Secure off-site copies

---

# 14. Scenario 2 – Restore Plain Table Backup

Plain dumps are restored using:

```text
psql
```

Restore:

```bash
psql \
  -d mydb \
  -U postgres \
  -f emp_backup.sql
```

---

# 15. Better Production Restore for Plain Dumps

By default, `psql` can continue after SQL errors.

For a controlled restore:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d mydb \
  -f emp_backup.sql
```

This is safer because the restore stops when an SQL error occurs.

For an all-or-nothing restore, where appropriate:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  --single-transaction \
  -d mydb \
  -f emp_backup.sql
```

Be careful with very large restores because a single transaction may run for a long time.

---

# 16. Existing Table Problem

Suppose:

```text
raj1.emp
```

already exists.

The plain dump contains:

```sql
CREATE TABLE raj1.emp ...
```

The restore can fail with:

```text
relation "emp" already exists
```

One solution:

```sql
DROP TABLE raj1.emp;
```

Then restore:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d mydb \
  -f emp_backup.sql
```

Another option is to create the dump with cleanup commands when appropriate.

---

# 17. Scenario 3 – Structure-Only Backup

Requirement:

> Backup only the table definition.

Command:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  --schema-only \
  -f emp_struct.sql
```

Short option:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -s \
  -f emp_struct.sql
```

This contains:

```text
Table definition
Indexes belonging to selected objects
Constraints belonging to selected objects
Other selected object definitions
```

but not table rows.

---

# 18. Scenario 4 – Data-Only Backup

Requirement:

> Backup only the rows.

Command:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  --data-only \
  -f emp_data.sql
```

Short option:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -a \
  -f emp_data.sql
```

The target table must already exist before a normal data-only restore.

---

# 19. Scenario 5 – Restore Structure and Data Separately

Restore structure first:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d mydb \
  -f emp_struct.sql
```

Then restore data:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d mydb \
  -f emp_data.sql
```

Verify:

```sql
SELECT count(*)
FROM raj1.emp;
```

---

# 20. pg_dump Backup Formats

PostgreSQL supports four important formats.

| Format    | Option | Human Readable |    Compression | Restore      | Parallel Dump | Parallel Restore |
| --------- | ------ | -------------: | -------------: | ------------ | ------------: | ---------------: |
| Plain     | `-Fp`  |            Yes |  No by default | `psql`       |            No |               No |
| Custom    | `-Fc`  |             No | Yes by default | `pg_restore` |            No |              Yes |
| Tar       | `-Ft`  |             No |         **No** | `pg_restore` |            No |               No |
| Directory | `-Fd`  |             No | Yes by default | `pg_restore` |           Yes |              Yes |

## Important Correction

Tar format does **not** support compression.

Therefore, do not say:

> Tar format is used to reduce backup size.

For large databases where compression and parallel dump are important, directory format is often more useful.

---

# 21. Custom Format

Custom format is:

```text
-Fc
```

The correct PostgreSQL terminology is:

> Custom archive format.

Advantages:

* Compressed by default
* Selective restore
* `pg_restore` support
* Parallel restore support
* Table/schema selection during restore

---

# 22. Scenario 6A – Custom Format Backup

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -Fc \
  -f emp.dump \
  -v \
  2>emp_backup.log
```

Check:

```bash
ls -lh emp.dump
```

The dump is not meant to be read with `cat`.

Inspect it with:

```bash
pg_restore --list emp.dump
```

---

# 23. Restore Custom Backup

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t emp \
  --clean \
  --if-exists \
  --exit-on-error \
  -v \
  emp.dump \
  2>restore.log
```

Important options:

```text
--clean
```

Drops objects before restoring them.

```text
--if-exists
```

Avoids unnecessary "object does not exist" errors during cleanup.

```text
--exit-on-error
```

Stops when a restore SQL error occurs.

```text
-v
```

Verbose logging.

---

# 24. pg_restore `-d` and `-f`

Understand this clearly.

## `-d`

```bash
pg_restore -d mydb emp.dump
```

means:

> Connect to `mydb` and perform the restore.

## `-f`

```bash
pg_restore -f restore.sql emp.dump
```

means:

> Generate SQL output into `restore.sql`.

Do not combine them expecting both direct restore and SQL-file generation.

If SQL needs to be generated:

```bash
pg_restore \
  -f restore.sql \
  emp.dump
```

Then:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d mydb \
  -f restore.sql
```

---

# 25. Scenario 6B – Tar Backup

```bash
pg_dump \
  -d mydb \
  -Ft \
  -t raj1.emp \
  -f emp.tar \
  -v \
  2>backup.log
```

Restore:

```bash
pg_restore \
  -d mydb \
  --clean \
  --if-exists \
  --exit-on-error \
  emp.tar
```

`pg_restore` normally detects the archive format automatically.

Therefore:

```text
-Ft
```

does not normally need to be supplied during restore.

---

# 26. Tar Parallel Backup Test

Try:

```bash
pg_dump \
  -d mydb \
  -Ft \
  -j 2 \
  -f emp.tar
```

Expected:

```text
parallel backup only supported by the directory format
```

Reason:

> Parallel `pg_dump` requires directory format.

---

# 27. Scenario 6C – Directory Format

Backup:

```bash
pg_dump \
  -d mydb \
  -Fd \
  -t raj1.emp \
  -f empdir \
  -v \
  2>backup.log
```

Output:

```text
empdir/
```

This is a directory rather than one dump file.

---

# 28. Parallel Directory Backup

```bash
pg_dump \
  -d mydb \
  -Fd \
  -j 2 \
  -t raj1.emp \
  -f empdir \
  -v \
  2>backup.log
```

---

# 29. Parallel Restore

```bash
pg_restore \
  -d mydb \
  --clean \
  --if-exists \
  --exit-on-error \
  -j 2 \
  -v \
  empdir \
  2>restore.log
```

Parallel restore can significantly reduce restore time for a database with enough independent work.

---

# 30. How Many Parallel Jobs?

Do not use this rule blindly:

```text
jobs = number of CPUs
```

CPU count is only one consideration.

Check:

```bash
lscpu
```

Also consider:

* CPU
* Storage IOPS
* Storage throughput
* Network throughput
* Database size
* Number of tables
* Number of indexes
* Current production workload
* Available database connections

Important:

```text
pg_dump -j N
```

uses approximately:

```text
N + 1 database connections
```

Example:

```text
-j 4
```

can use approximately:

```text
5 connections
```

So check:

```sql
SHOW max_connections;
```

before aggressive parallel dumping.

---

# 31. Scenario 7 – Backup Multiple Tables

Requirement:

> Backup `emp`, `dept` and `hr`.

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -t raj1.dept \
  -t raj1.hr \
  -Fc \
  -f multi.dump \
  -v \
  2>backup.log
```

---

# 32. Inspect Multiple-Table Backup

```bash
pg_restore --list multi.dump
```

You may see:

```text
TABLE raj1 dept
TABLE raj1 emp
TABLE raj1 hr

TABLE DATA raj1 dept
TABLE DATA raj1 emp
TABLE DATA raj1 hr
```

This is better than depending only on backup logs to understand archive contents.

---

# 33. Restore All Three Tables

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t emp \
  -t dept \
  -t hr \
  --clean \
  --if-exists \
  --exit-on-error \
  multi.dump
```

---

# 34. Scenario 8 – Restore Only One Table

Requirement:

> Multi-table backup exists, but only `hr` was accidentally dropped.

Restore:

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t hr \
  --clean \
  --if-exists \
  --exit-on-error \
  multi.dump
```

---

# 35. Important Selective-Restore Warning

A table can depend on other database objects.

Examples:

```text
Custom data types
Functions
Sequences
Extensions
Objects in another schema
Referenced tables
```

When selecting only a table during backup or restore, PostgreSQL does not guarantee that every external dependency will automatically be included.

Therefore:

> Always understand object dependencies before performing a selective production restore.

---

# 36. Scenario 9 – Complete Schema Backup

Backup `raj1`:

```bash
pg_dump \
  -d mydb \
  -n raj1 \
  -Fc \
  -f raj1.dump \
  -v \
  2>backup.log
```

---

# 37. Restore Schema

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  --clean \
  --if-exists \
  --exit-on-error \
  raj1.dump
```

---

# 38. Schema Dependency Warning

This command:

```bash
pg_dump -d mydb -n raj1 ...
```

selects objects belonging to `raj1`.

It does not automatically guarantee inclusion of every external object on which `raj1` depends.

Example:

```text
raj1 table
     |
     +---- uses a custom type from common_schema
```

A dump containing only:

```text
raj1
```

may not contain that external type.

Therefore, schema-only selection should be tested before production use.

---

# 39. Scenario 10 – Exclude One Table

Requirement:

> Backup schema `raj1`, but do not include `raj1.emp`.

```bash
pg_dump \
  -d mydb \
  -n raj1 \
  -T raj1.emp \
  -Fc \
  -f raj1_without_emp.dump \
  -v \
  2>backup.log
```

Verify:

```bash
pg_restore --list raj1_without_emp.dump
```

---

# 40. Scenario 11 – Multiple Schema Backup

```bash
pg_dump \
  -d mydb \
  -n raj1 \
  -n raj2 \
  -n raj3 \
  -Fc \
  -f multischema.dump \
  -v \
  2>backup.log
```

---

# 41. Restore Only raj2

```bash
pg_restore \
  -d mydb \
  -n raj2 \
  --clean \
  --if-exists \
  --exit-on-error \
  multischema.dump
```

---

# 42. Scenario 12 – Complete Database Backup

Custom format:

```bash
pg_dump \
  -d mydb \
  -Fc \
  -f mydb.dump \
  -v \
  2>mydb_backup.log
```

Verify archive:

```bash
pg_restore --list mydb.dump
```

---

# 43. Complete Database Restore

For a controlled lab restore:

```bash
pg_restore \
  -d mydb \
  --clean \
  --if-exists \
  --exit-on-error \
  -v \
  mydb.dump \
  2>restore.log
```

---

# 44. Scenario 13 – Restore Database Under Another Database Name

Requirement:

```text
Source database : mydb
Target database : saidb
```

Create a clean database:

```bash
createdb -T template0 saidb
```

Restore:

```bash
pg_restore \
  -d saidb \
  --exit-on-error \
  -v \
  mydb.dump \
  2>restore.log
```

Verify:

```bash
psql -d saidb
```

Then:

```text
\dn
\dt raj1.*
\dt raj2.*
\dt raj3.*
```

---

# 45. Database Remapping – Correct Terminology

Do not call this:

```text
Database Remapping
```

The better terminology is:

> Restore the dump into a different target database.

PostgreSQL does not provide Oracle Data Pump-style:

```text
REMAP_SCHEMA
REMAP_TABLE
```

options in `pg_restore`.

---

# 46. Scenario 14 – Complete Logical Cluster Backup

Use:

```bash
pg_dumpall \
  -f fullbackup.sql
```

This produces a plain SQL script.

`pg_dumpall` works across the PostgreSQL cluster and handles cluster-level objects that normal `pg_dump` does not.

---

# 47. Restore pg_dumpall Backup

Use:

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d postgres \
  -f fullbackup.sql
```

A clean target cluster is often preferred for migration testing.

---

# 48. pg_dumpall with Cleanup Commands

If the restore is intended to overwrite existing cluster objects, the dump can be generated with:

```bash
pg_dumpall \
  --clean \
  --if-exists \
  -f fullbackup.sql
```

This generates appropriate cleanup commands for dumped:

* Databases
* Roles
* Tablespaces

Manual cleanup is therefore not always mandatory.

---

# 49. Scenario 15 – Global Objects Backup

Use:

```bash
pg_dumpall \
  --globals-only \
  -f globals.sql
```

Short form:

```bash
pg_dumpall -g > globals.sql
```

This mainly contains:

* Roles
* Role memberships
* Tablespaces
* Cluster-level grants/settings supported by the dump

---

# 50. Restore Global Objects

```bash
psql \
  -X \
  --set ON_ERROR_STOP=on \
  -d postgres \
  -f globals.sql
```

---

# 51. Password Security in Global Dumps

A global dump can contain PostgreSQL password verifiers.

For example:

```text
SCRAM-SHA-256$...
```

This is not the user's plaintext password.

However, it is still sensitive authentication material.

Protect:

```text
globals.sql
```

carefully.

When password verifiers should not be dumped:

```bash
pg_dumpall \
  --globals-only \
  --no-role-passwords \
  -f globals.sql
```

---

# 52. Scenario 16 – Five Large Databases

Assume:

```text
db1 = 100 GB
db2 = 100 GB
db3 = 100 GB
db4 = 100 GB
db5 = 100 GB
```

If each database takes 20 minutes and everything runs serially:

```text
db1 → db2 → db3 → db4 → db5
```

the total backup window can become large.

Instead, independent database dumps can be executed concurrently.

Example:

```bash
pg_dump -d db1 -Fd -j 2 -f /backup/db1
```

```bash
pg_dump -d db2 -Fd -j 2 -f /backup/db2
```

```bash
pg_dump -d db3 -Fd -j 2 -f /backup/db3
```

```bash
pg_dump -d db4 -Fd -j 2 -f /backup/db4
```

```bash
pg_dump -d db5 -Fd -j 2 -f /backup/db5
```

Also:

```bash
pg_dumpall \
  -g \
  -f /backup/globals.sql
```

---

# 53. Important Production Warning About Parallel Databases

Do not promise:

```text
5 databases × 5 minutes
= everything completes in 5 minutes
```

That is not guaranteed.

When multiple dumps run simultaneously, they compete for:

* CPU
* Disk
* Network
* Memory/cache
* Database connections
* Storage throughput

Always test the workload.

---

# 54. Scenario 17 – Remote Database Backup

Database server:

```text
192.168.56.201
```

Backup from another machine:

```bash
pg_dump \
  -h 192.168.56.201 \
  -U postgres \
  -d db1 \
  -Fd \
  -j 2 \
  -f /tmp/backup/db1 \
  -v \
  2>db1_backup.log
```

Important:

> The backup files are created on the machine where `pg_dump` is running.

They are not automatically stored on the database server.

---

# 55. Scenario 18 – Direct Server-to-Server Logical Migration

Suppose:

```text
Old Server → New Server
```

A simple logical pipeline can be:

```bash
pg_dump \
  -h old-db-server \
  -U postgres \
  mydb \
| psql \
  -X \
  --set ON_ERROR_STOP=on \
  -h new-db-server \
  -U postgres \
  mydb
```

This can be useful for migration.

For production migrations, consider:

* Network reliability
* Database size
* Downtime
* Authentication
* Encryption
* Error handling
* Rollback plan

---

# 56. Physical Backup Introduction

Logical backup:

```text
Database objects/data
```

Physical backup:

```text
Actual PostgreSQL cluster files
```

Main tool:

```text
pg_basebackup
```

---

# 57. Scenario 19 – Full Physical Backup

Recommended general example:

```bash
/usr/pgsql-18/bin/pg_basebackup \
  -D /var/lib/pgsql/fullbackup \
  -X stream \
  -P
```

Options:

```text
-D
```

Destination directory.

```text
-X stream
```

Stream required WAL while the backup is running.

```text
-P
```

Display progress.

---

# 58. `-X fetch` vs `-X stream`

The original lab uses:

```bash
pg_basebackup \
  -D /var/lib/pgsql/backup \
  -X fetch \
  -P
```

`fetch` collects required WAL at the end.

A commonly safer approach is:

```bash
pg_basebackup \
  -D /var/lib/pgsql/backup \
  -X stream \
  -P
```

because WAL is streamed while the backup is running.

---

# 59. Fast Checkpoint

The original lab used:

```text
-cfast
```

A clearer form is:

```text
--checkpoint=fast
```

Example:

```bash
pg_basebackup \
  -D /var/lib/pgsql/fullbackup \
  -X stream \
  -P \
  --checkpoint=fast
```

Be careful:

> Fast checkpoint can increase I/O load on a busy production database.

Do not use it automatically without understanding the impact.

---

# 60. Physical Backup Contents

A plain physical backup can contain:

```text
PG_VERSION
base/
global/
pg_wal/
pg_xact/
pg_multixact/
pg_tblspc/
pg_logical/
postgresql.conf
pg_hba.conf
backup_label
backup_manifest
```

It represents a physical PostgreSQL cluster backup.

---

# 61. Scenario 20 – Verify Physical Backup

After backup:

```bash
pg_verifybackup \
  /var/lib/pgsql/fullbackup
```

Do not stop here.

`pg_verifybackup` can detect many problems such as:

* Missing files
* Modified files
* Checksum mismatches
* Manifest inconsistencies
* Required WAL verification issues

But it cannot prove every possible recovery scenario will work.

Therefore:

```text
Backup
   ↓
pg_verifybackup
   ↓
Restore Test
   ↓
Start PostgreSQL
   ↓
Database Validation
   ↓
Application Validation
```

---

# 62. WAL Archiving

WAL means:

```text
Write-Ahead Log
```

WAL is critical for:

* Crash recovery
* Physical replication
* Point-in-Time Recovery
* Continuous archiving
* Incremental backup block tracking through WAL summaries

Archived WAL must be protected according to recovery requirements.

---

# 63. Scenario 21 – Archive Cleanup

Example:

```bash
pg_archivecleanup \
  /var/lib/pgsql/archive \
  0000000100000000000000CA.00000028.backup
```

Before real cleanup, perform a dry run where supported:

```bash
pg_archivecleanup \
  --dry-run \
  /var/lib/pgsql/archive \
  0000000100000000000000CA.00000028.backup
```

Review the files that would be removed.

Then execute the real cleanup.

---

# 64. Critical WAL Cleanup Warning

Never delete old WAL only because:

> A new full backup completed.

Old WAL might still be needed by:

* Older retained backups
* PITR
* Standby servers
* DR environment
* Recovery policy
* Other archive consumers

Production retention must consider the **oldest recovery point still required**.

---

# 65. Scenario 22 – Restore Physical Backup as Separate Instance

Assume production:

```text
Data directory : /var/lib/pgsql/18/data
Port           : 5432
```

Restore copy:

```text
Data directory : /var/lib/pgsql/restoretest
Port           : 5433
```

---

# 66. Prepare Restore-Test Instance

Before starting the copied cluster, review its configuration carefully.

Set:

```text
port = 5433
cluster_name = 'restoretest'
```

Permissions:

```bash
chown -R postgres:postgres \
  /var/lib/pgsql/restoretest
```

```bash
chmod 0700 \
  /var/lib/pgsql/restoretest
```

---

# 67. Critical Restore Isolation Checks

Changing only the port is not enough.

Before starting the restored physical copy, inspect:

```text
port
unix_socket_directories
archive_mode
archive_command
restore_command
primary_conninfo
primary_slot_name
tablespace paths
shared_preload_libraries
external monitoring configuration
```

Why?

Imagine the copied configuration still contains:

```text
archive_command = 'cp %p /production/archive/%f'
```

Starting the test instance could cause the restored cluster to interact with the production archive environment.

Therefore:

> Never start a cloned physical PostgreSQL cluster until external integration points have been isolated.

---

# 68. Start Separate Restore Instance

```bash
/usr/pgsql-18/bin/pg_ctl \
  -D /var/lib/pgsql/restoretest \
  start
```

Check:

```bash
ps -ef | grep postgres
```

Connect:

```bash
psql -p 5433
```

Verify:

```sql
SHOW port;
```

```sql
SHOW data_directory;
```

---

# 69. Scenario 23 – Restore Over Existing Instance

This is more dangerous.

Stop PostgreSQL:

```bash
systemctl stop postgresql-18
```

Preserve existing data:

```bash
mv \
  /var/lib/pgsql/18/data \
  /var/lib/pgsql/18/data_old
```

Move restored data:

```bash
mv \
  /var/lib/pgsql/restore \
  /var/lib/pgsql/18/data
```

Correct ownership:

```bash
chown -R postgres:postgres \
  /var/lib/pgsql/18/data
```

Permissions:

```bash
chmod 0700 \
  /var/lib/pgsql/18/data
```

Start:

```bash
systemctl start postgresql-18
```

---

# 70. Which Restore Method Is Safer?

For testing:

> Restore as a separate instance whenever possible.

Advantages:

* Existing database remains untouched.
* Easier rollback.
* Easy data comparison.
* Safer restore testing.
* Application testing can happen before cutover.

Disadvantage:

* Requires additional storage and resources.

---

# 71. PostgreSQL 18 Incremental Physical Backup

PostgreSQL supports incremental physical backups using:

```text
pg_basebackup --incremental
```

along with:

```text
WAL Summarization
+
backup_manifest
+
pg_combinebackup
```

---

# 72. WAL Summarization

Incremental backup requires:

```text
summarize_wal = on
```

Check:

```sql
SHOW summarize_wal;
```

Also check:

```sql
SHOW wal_level;
```

`summarize_wal` cannot operate correctly for incremental-backup purposes with WAL generated at:

```text
wal_level = minimal
```

Normal production systems commonly use:

```text
replica
```

or:

```text
logical
```

depending on requirements.

---

# 73. Enable WAL Summarization

Edit:

```text
postgresql.conf
```

Configure:

```text
summarize_wal = on
```

Reload:

```bash
/usr/pgsql-18/bin/pg_ctl \
  -D /var/lib/pgsql/18/data \
  reload
```

Verify:

```sql
SHOW summarize_wal;
```

Expected:

```text
on
```

Check process:

```bash
ps -ef | grep walsummarizer
```

---

# 74. Verify Available WAL Summaries

PostgreSQL provides information about available WAL summaries.

Example:

```sql
SELECT *
FROM pg_available_wal_summaries();
```

WAL summary files are stored under:

```text
pg_wal/summaries
```

They support incremental physical backup.

---

# 75. wal_summary_keep_time

Important parameter:

```text
wal_summary_keep_time
```

Check:

```sql
SHOW wal_summary_keep_time;
```

The default is:

```text
10 days
```

The retention should be comfortably longer than the possible time between a backup and a later incremental backup that depends on it.

If required WAL summaries are removed:

> Incremental backup can fail.

---

# 76. Scenario 24 – Full Backup for Incremental Chain

Take full backup:

```bash
pg_basebackup \
  -D /var/lib/pgsql/fullbackup \
  -X stream \
  -P
```

After completion:

```bash
ls -l \
  /var/lib/pgsql/fullbackup/backup_manifest
```

The manifest is important for future incremental backups.

---

# 77. Generate Database Changes

Create databases:

```sql
CREATE DATABASE inc1;
CREATE DATABASE inc2;
CREATE DATABASE inc3;
```

Connect:

```text
\c inc1
```

Create schema:

```sql
CREATE SCHEMA inc;
```

Create tables:

```sql
CREATE TABLE inc.emp
(
    eno int
);

CREATE TABLE inc.dept
(
    eno int
);
```

Insert/change data to generate WAL.

---

# 78. Scenario 25 – Incremental Backup

Use the previous backup manifest:

```bash
pg_basebackup \
  -D /var/lib/pgsql/incremental \
  -X stream \
  -P \
  --incremental=/var/lib/pgsql/fullbackup/backup_manifest
```

The earlier manifest tells PostgreSQL which earlier backup is the reference.

---

# 79. Incremental Backup Behavior

An incremental backup does not simply mean:

```text
Copy every changed file completely.
```

PostgreSQL can store changed relation blocks based on WAL summary information.

Some non-relation files are still copied completely.

Therefore incremental backups can reduce:

* Backup size
* Backup I/O
* Backup time

especially when the database is large and only part of it changes.

---

# 80. When Incremental Backup Helps

Good case:

```text
Database size : 5 TB
Daily changes : 100 GB
```

Incremental backup can provide significant savings.

Less useful case:

```text
Database size : 50 GB
Daily changes : 40 GB
```

If most blocks change, the incremental backup may not provide large savings.

---

# 81. Incremental Backup Is Not Standalone

You cannot normally use the incremental backup directory directly as a complete PostgreSQL cluster.

It must first be reconstructed using:

```text
pg_combinebackup
```

---

# 82. Scenario 26 – Combine Full + Incremental

```bash
pg_combinebackup \
  /var/lib/pgsql/fullbackup \
  /var/lib/pgsql/incremental \
  -o /var/lib/pgsql/finalbackup
```

Output:

```text
finalbackup
```

is a reconstructed:

> Synthetic Full Backup.

---

# 83. Verify Combined Backup

```bash
pg_verifybackup \
  /var/lib/pgsql/finalbackup
```

Then perform an actual restore test.

---

# 84. Scenario 27 – Chained Incremental Backups

Assume:

```text
Sunday     Full
Monday     Inc1
Tuesday    Inc2
Wednesday  Inc3
```

If:

```text
Inc1 depends on Full
Inc2 depends on Inc1
Inc3 depends on Inc2
```

then:

```text
Full
 ↓
Inc1
 ↓
Inc2
 ↓
Inc3
```

To reconstruct Wednesday:

```bash
pg_combinebackup \
  /backup/full \
  /backup/inc1 \
  /backup/inc2 \
  /backup/inc3 \
  -o /backup/wednesday_full
```

Required backups are supplied:

```text
Oldest → Newest
```

---

# 85. Independent Incrementals from Full

Another possible design:

```text
        +--> Inc1
        |
Full ---+--> Inc2
        |
        +--> Inc3
```

This happens when each incremental references the same full backup manifest.

Therefore:

> Incremental dependency is determined by the backup manifest supplied to `--incremental`.

Do not blindly describe PostgreSQL incremental backup as traditional differential or chained incremental backup without checking the actual manifest relationship.

---

# 86. Protect the Incremental Chain

PostgreSQL does not maintain your backup dependency inventory for you.

The DBA must track:

```text
Full Backup
    |
    +--- Which incrementals depend on it?
            |
            +--- Which later incrementals depend on those?
```

If a required parent backup is deleted, a dependent incremental backup may become unusable.

---

# 87. Full vs Incremental Physical Backup

| Feature                   | Full   | Incremental           |
| ------------------------- | ------ | --------------------- |
| `pg_basebackup`           | Yes    | Yes                   |
| `--incremental`           | No     | Yes                   |
| Standalone backup         | Yes    | No                    |
| WAL summaries needed      | No     | Yes                   |
| Backup manifest           | Yes    | Required as reference |
| Combine before normal use | No     | Yes                   |
| Backup size               | Larger | Usually smaller       |
| Management complexity     | Lower  | Higher                |

---

# 88. Logical vs Physical Backup

| Requirement                     | Logical   | Physical                                    |
| ------------------------------- | --------- | ------------------------------------------- |
| Single table                    | Yes       | No                                          |
| Schema                          | Yes       | No                                          |
| Single database                 | Yes       | No                                          |
| Selective restore               | Excellent | Limited                                     |
| Cluster files                   | No        | Yes                                         |
| Standby creation                | No        | Yes                                         |
| PITR foundation                 | No        | Yes                                         |
| Incremental physical backup     | No        | Yes                                         |
| Human-readable option           | Yes       | No                                          |
| Cross-version logical migration | Useful    | Physical restore has stricter compatibility |

---

# 89. Real-Time Scenario 28 – Developer Dropped One Table

Incident:

```text
10:15 AM

Developer accidentally runs:

DROP TABLE payroll.employee;
```

Available:

```text
Nightly custom-format logical dump
```

Requirement:

> Restore only `payroll.employee`.

First inspect:

```bash
pg_restore \
  --list \
  payroll.dump
```

Restore:

```bash
pg_restore \
  -d payrolldb \
  -n payroll \
  -t employee \
  --exit-on-error \
  payroll.dump
```

Then validate:

```sql
SELECT count(*)
FROM payroll.employee;
```

Also check dependent:

* Sequences
* Indexes
* Constraints
* Triggers
* Foreign keys
* Permissions

This is why selective logical backups/restores are useful.

---

# 90. Real-Time Scenario 29 – QA Refresh from Production

Requirement:

```text
Production database : salesprod
QA database         : salesqa
```

Take backup:

```bash
pg_dump \
  -h prod-db \
  -U backup_user \
  -d salesprod \
  -Fc \
  -f salesprod.dump
```

Create clean QA database:

```bash
createdb \
  -T template0 \
  salesqa
```

Restore:

```bash
pg_restore \
  -d salesqa \
  --exit-on-error \
  -v \
  salesprod.dump
```

Production concern:

> Sensitive production data should normally be masked or sanitized before being made available to QA users.

Examples:

```text
Customer names
Emails
Phone numbers
Addresses
Payment information
Employee information
```

---

# 91. Real-Time Scenario 30 – Database Migration

Requirement:

```text
Old PostgreSQL server
        ↓
New PostgreSQL server
```

Steps:

```text
1. Check versions.
2. Check extensions.
3. Check database sizes.
4. Check roles.
5. Check tablespaces.
6. Take logical backup.
7. Backup globals.
8. Transfer backups.
9. Restore globals.
10. Restore databases.
11. Analyze database.
12. Validate row counts.
13. Validate application.
14. Cut over application.
```

Backup globals:

```bash
pg_dumpall \
  --globals-only \
  -f globals.sql
```

Database:

```bash
pg_dump \
  -d mydb \
  -Fd \
  -j 4 \
  -f mydb_dump
```

Restore:

```bash
pg_restore \
  -d mydb \
  -j 4 \
  --exit-on-error \
  mydb_dump
```

---

# 92. Real-Time Scenario 31 – 2 TB Database Backup Window Too Long

Problem:

```text
Database size : 2 TB
Backup window : 8 hours
Allowed window: 4 hours
```

Possible investigation:

```text
Current format?
Current compression?
Storage throughput?
Network throughput?
Number of large tables?
Parallel jobs?
CPU usage?
Database load?
```

Possible test:

```bash
pg_dump \
  -Fd \
  -j 8 \
  -d largeprod \
  -f /backup/largeprod
```

Do not immediately increase:

```text
-j 8
```

to:

```text
-j 32
```

without testing.

More workers can overload storage and make the backup slower.

---

# 93. Real-Time Scenario 32 – Backup Completed but Restore Failed

Problem:

```text
Nightly backup job = SUCCESS

During DR test:
Restore = FAILED
```

Possible reasons:

* Backup file damaged
* Missing archive file
* Missing external dependency
* Role missing
* Tablespace missing
* Extension missing
* Incorrect permissions
* Backup incomplete
* Required WAL missing
* Incremental parent backup deleted

Lesson:

> Backup-job success does not equal recovery success.

Required practice:

```text
Backup
+
Verification
+
Restore Testing
```

---

# 94. Real-Time Scenario 33 – Restore into Isolated Validation Server

Production:

```text
prod-db:5432
```

Restore server:

```text
restore-db:5433
```

Process:

```text
1. Copy backup.
2. Verify backup.
3. Restore separately.
4. Start isolated PostgreSQL.
5. Check logs.
6. Check databases.
7. Check schemas.
8. Check row counts.
9. Check roles.
10. Run application validation.
11. Record restore duration.
```

This gives the DBA a real measured:

```text
RTO
```

instead of an assumed RTO.

---

# 95. Real-Time Scenario 34 – Archived WAL Storage Is Full

Problem:

```text
/archive = 100% full
```

Wrong action:

```bash
rm -rf /archive/*
```

Never do this blindly.

Correct investigation:

```text
Which base backups are retained?
What PITR window is required?
Which standbys consume archive?
What is the oldest required WAL?
Is archiving healthy?
Is archive cleanup configured?
```

Use:

```bash
pg_archivecleanup --dry-run ...
```

only after determining the correct retention boundary.

---

# 96. Real-Time Scenario 35 – Incremental Backup Suddenly Fails

Error:

```text
incremental backups cannot be taken unless
WAL summarization is enabled
```

Check:

```sql
SHOW summarize_wal;
```

Check:

```sql
SHOW wal_level;
```

Check summaries:

```sql
SELECT *
FROM pg_available_wal_summaries();
```

Check process:

```bash
ps -ef | grep walsummarizer
```

Check retention:

```sql
SHOW wal_summary_keep_time;
```

Possible causes:

* `summarize_wal` disabled
* Required summaries missing
* WAL summarizer behind
* `wal_level` problem
* Incorrect reference manifest
* Parent backup from wrong server lineage

---

# 97. Real-Time Scenario 36 – Backup User Instead of postgres

Production best practice:

> Avoid running every logical backup as the `postgres` superuser when a properly designed backup role can satisfy the requirement.

Example concept:

```text
backup_user
```

The backup account needs enough privileges to read the required objects.

Exact privilege design depends on:

* PostgreSQL version
* Objects being dumped
* Row-level security
* Security policy
* Whether global objects are required

Test the backup role before production implementation.

---

# 98. Real-Time Scenario 37 – Restore Without Original Owner

Problem:

Backup contains:

```text
Owner: app_owner
```

Target environment does not have:

```text
app_owner
```

Possible migration approach:

```bash
pg_restore \
  --no-owner \
  -d targetdb \
  backup.dump
```

You may also use ownership reassignment strategies depending on the migration design.

This is common when moving:

```text
Production → Development
```

or:

```text
On-premises → Cloud
```

---

# 99. Real-Time Scenario 38 – Restore Only Data

Suppose table structures are already deployed by application migration scripts.

Requirement:

> Load only backup data.

Use:

```bash
pg_restore \
  --data-only \
  --exit-on-error \
  -d mydb \
  backup.dump
```

Be careful about:

* Foreign keys
* Triggers
* Existing rows
* Duplicate primary keys
* Sequences
* Application writes during restore

---

# 100. Real-Time Scenario 39 – Restore Schema Only

Requirement:

> Development needs production structure but no production data.

Use:

```bash
pg_restore \
  --schema-only \
  -d devdb \
  backup.dump
```

Benefits:

* No customer data copied
* Developers get object definitions
* Useful for development and testing

Still review:

* Functions
* Security-definer functions
* Extensions
* Ownership
* Privileges

---

# 101. Real-Time Scenario 40 – Validate Backup Before Deleting Old Backup

Never use:

```text
New backup exists
→ Delete old backup immediately
```

Better process:

```text
Take new backup
      ↓
Check command status
      ↓
Check backup size
      ↓
Review log
      ↓
Verify archive/manifest
      ↓
Perform scheduled restore test
      ↓
Confirm retention policy
      ↓
Expire old backup
```

---

# 102. Production Backup Checklist

Before backup:

```text
1. Confirm available disk space.
2. Confirm backup destination.
3. Check database size.
4. Check backup user.
5. Check network if remote.
6. Check current server load.
7. Confirm backup format.
8. Decide parallel jobs.
9. Confirm retention.
10. Confirm monitoring.
```

During backup:

```text
1. Monitor CPU.
2. Monitor disk I/O.
3. Monitor filesystem usage.
4. Monitor database connections.
5. Monitor logs.
6. Monitor backup progress.
7. Monitor application impact.
```

After backup:

```text
1. Check exit status.
2. Review backup log.
3. Check backup size.
4. Inspect archive.
5. Verify physical backup.
6. Protect backup permissions.
7. Copy off-site if required.
8. Apply retention.
9. Record backup duration.
10. Schedule restore validation.
```

---

# 103. Restore Checklist

Before restore:

```text
1. Confirm correct backup.
2. Confirm backup timestamp.
3. Confirm source version.
4. Confirm target version.
5. Confirm target database.
6. Confirm free storage.
7. Confirm roles.
8. Confirm tablespaces.
9. Confirm extensions.
10. Confirm rollback plan.
```

After restore:

```text
1. Check PostgreSQL logs.
2. Check databases.
3. Check schemas.
4. Check tables.
5. Check row counts.
6. Check sequences.
7. Check indexes.
8. Check constraints.
9. Check roles.
10. Check privileges.
11. Check extensions.
12. Run ANALYZE where appropriate.
13. Test application login.
14. Test critical queries.
15. Record actual restore time.
```

---

# 104. Logical Restore Statistics

After a large logical restore, query planner statistics may need attention.

Example:

```bash
vacuumdb \
  -d mydb \
  --analyze
```

Or:

```sql
ANALYZE;
```

Do not automatically run aggressive maintenance without considering database size and workload.

---

# 105. RPO

RPO means:

> Recovery Point Objective.

Question:

> How much data can the business afford to lose?

Example:

```text
RPO = 15 minutes
```

means the recovery strategy should aim to limit data loss to approximately 15 minutes or less.

---

# 106. RTO

RTO means:

> Recovery Time Objective.

Question:

> How long can the business wait for database service to be restored?

Example:

```text
RTO = 1 hour
```

A backup design must satisfy both:

```text
RPO
+
RTO
```

---

# 107. Example Production Backup Architecture

```text
                  PostgreSQL Production
                           |
          +----------------+----------------+
          |                                 |
   Logical Backups                   Physical Backup
          |                                 |
   pg_dump / pg_dumpall               pg_basebackup
          |                                 |
          |                          Full / Incremental
          |                                 |
          +----------------+----------------+
                           |
                    WAL Archiving
                           |
                    Backup Storage
                           |
                +----------+----------+
                |                     |
           Local Copy             Off-site Copy
                                      |
                              DR / Object Storage
                                      |
                               Restore Testing
```

---

# 108. Recommended Multi-Layer Strategy

A production environment should normally consider:

```text
Logical Backup
      +
Physical Backup
      +
Continuous WAL Archiving
      +
Off-site Backup
      +
Backup Verification
      +
Restore Testing
      +
Monitoring and Alerting
```

No single backup method solves every recovery problem.

---

# 109. Important Interview Questions

## Q1. Which tool backs up one PostgreSQL database?

```text
pg_dump
```

---

## Q2. Which utility restores custom-format backups?

```text
pg_restore
```

---

## Q3. Which utility restores plain SQL dumps?

```text
psql
```

---

## Q4. Which pg_dump format supports parallel backup?

```text
Directory format (-Fd)
```

---

## Q5. Can custom-format backups be restored in parallel?

Yes.

```bash
pg_restore -j ...
```

---

## Q6. Does tar format support pg_dump compression?

No.

---

## Q7. Can tar format use parallel pg_dump?

No.

---

## Q8. Can pg_dump back up all databases in one invocation?

No.

`pg_dump` handles one database per invocation.

Use:

```text
pg_dumpall
```

for cluster-wide logical dumps.

---

## Q9. How do we back up roles and tablespaces?

```bash
pg_dumpall --globals-only
```

---

## Q10. Can pg_basebackup back up one table?

No.

It performs physical cluster backup.

---

## Q11. What is required for PostgreSQL incremental physical backup?

```text
summarize_wal = on
```

and required WAL summaries must be available.

---

## Q12. Where are WAL summary files stored?

```text
pg_wal/summaries
```

---

## Q13. How do we reconstruct an incremental physical backup?

```text
pg_combinebackup
```

---

## Q14. How do we verify a physical base backup?

```text
pg_verifybackup
```

---

## Q15. Does pg_verifybackup replace restore testing?

No.

A real restore test is still required.

---

## Q16. How many connections can `pg_dump -j 4` use?

Approximately:

```text
4 workers + 1 leader = 5 connections
```

---

## Q17. What happens if a required incremental parent backup is deleted?

The dependent incremental backup might no longer be reconstructable.

---

# 110. Quick Command Reference

## Plain table backup

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -f emp.sql
```

## Custom table backup

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -Fc \
  -f emp.dump
```

## Structure only

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -s \
  -f emp_struct.sql
```

## Data only

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -a \
  -f emp_data.sql
```

## Schema

```bash
pg_dump \
  -d mydb \
  -n raj1 \
  -Fc \
  -f raj1.dump
```

## Database

```bash
pg_dump \
  -d mydb \
  -Fc \
  -f mydb.dump
```

## Directory parallel backup

```bash
pg_dump \
  -d mydb \
  -Fd \
  -j 4 \
  -f mydb_dir
```

## Parallel restore

```bash
pg_restore \
  -d mydb \
  -j 4 \
  --exit-on-error \
  mydb_dir
```

## Globals

```bash
pg_dumpall \
  -g \
  -f globals.sql
```

## Full logical cluster

```bash
pg_dumpall \
  -f cluster.sql
```

## Physical backup

```bash
pg_basebackup \
  -D /backup/full \
  -X stream \
  -P
```

## Verify

```bash
pg_verifybackup \
  /backup/full
```

## Incremental backup

```bash
pg_basebackup \
  -D /backup/inc1 \
  -X stream \
  -P \
  --incremental=/backup/full/backup_manifest
```

## Combine

```bash
pg_combinebackup \
  /backup/full \
  /backup/inc1 \
  -o /backup/final
```

## Verify combined backup

```bash
pg_verifybackup \
  /backup/final
```

---

# 111. Practice Lab – Beginner

Complete the following without looking at the commands above.

### Task 1

Create:

```text
Database : trainingdb
Schema   : finance
Table    : employees
```

Insert 10 rows.

Take:

```text
Plain backup
```

Drop the table.

Restore it.

Verify row count.

---

### Task 2

Take:

```text
Structure-only backup
```

Drop the table.

Restore structure.

Verify:

```text
0 rows
```

---

### Task 3

Take:

```text
Data-only backup
```

Restore data into the existing table.

Verify row count.

---

# 112. Practice Lab – Intermediate

Create:

```text
Schema: hr

Tables:
employees
departments
locations
jobs
```

Perform:

```text
1. Backup all four tables.
2. Drop employees.
3. Restore only employees.
4. Drop schema hr.
5. Restore complete schema.
6. Inspect archive using pg_restore --list.
7. Restore into another database.
```

---

# 113. Practice Lab – Advanced

Create:

```text
Database size target:
Large enough to observe parallel processing
```

Perform:

```text
1. Directory-format backup with -j 1.
2. Record duration.
3. Directory-format backup with -j 2.
4. Record duration.
5. Directory-format backup with -j 4.
6. Record duration.
7. Compare CPU.
8. Compare disk I/O.
9. Compare connections.
10. Compare backup size.
```

Goal:

> Understand that more parallel jobs do not always mean proportionally faster backups.

---

# 114. Practice Lab – Physical Backup

Perform:

```text
1. Take pg_basebackup.
2. Run pg_verifybackup.
3. Copy backup to restore directory.
4. Isolate configuration.
5. Change port.
6. Start separate PostgreSQL instance.
7. Connect.
8. Verify databases.
9. Verify row counts.
10. Stop restore instance.
```

Record:

```text
Backup duration
Backup size
Restore duration
Validation duration
```

---

# 115. Practice Lab – Incremental Backup

Perform:

```text
1. Enable WAL summarization.
2. Verify summarize_wal.
3. Verify wal_level.
4. Take full backup.
5. Generate database changes.
6. Take Inc1.
7. Generate more changes.
8. Take Inc2 referencing Inc1.
9. Generate more changes.
10. Take Inc3 referencing Inc2.
11. Combine Full + Inc1 + Inc2 + Inc3.
12. Run pg_verifybackup.
13. Restore as separate instance.
14. Validate latest data.
```

Then repeat using:

```text
Full → Inc1
Full → Inc2
Full → Inc3
```

Compare the dependency designs.

---

# 116. Final DBA Rules

Remember these points:

```text
1. Never trust a backup without testing restore.

2. Never delete WAL blindly.

3. Never delete a full backup while dependent
   incremental backups still require it.

4. Never start a cloned physical database without
   isolating it from production.

5. Never assume more parallel jobs always means
   faster backup.

6. Never assume a selective table/schema backup
   automatically contains every external dependency.

7. Protect backup files like production data.

8. Monitor backup duration and restore duration.

9. Design backup strategy around RPO and RTO.

10. Maintain more than one recovery mechanism
    for critical production databases.
```

---

# 117. Final Summary

The complete PostgreSQL backup architecture can be remembered as:

```text
                     PostgreSQL
                         |
          +--------------+--------------+
          |                             |
       Logical                       Physical
          |                             |
      pg_dump                     pg_basebackup
      pg_dumpall                        |
          |                     +-------+-------+
          |                     |               |
       Restore                 Full        Incremental
          |                     |               |
     +----+----+                |        WAL Summaries
     |         |                |               |
   psql    pg_restore           +-------+-------+
                                             |
                                      pg_combinebackup
                                             |
                                      Synthetic Full
                                             |
                                      pg_verifybackup
                                             |
                                       Restore Test
                                             |
                                   Application Validation
```

## Most Important Rule

> **Backup success is not the final goal. Recovery success is the final goal.**

A production DBA should know:

```text
What was backed up?
Where is the backup?
Is it protected?
Has it been verified?
What other backups/WAL does it depend on?
How long will restore take?
Has restore actually been tested?
Can the application use the restored database?
```

If those questions can be answered confidently, the PostgreSQL backup strategy is in a much stronger position.

[1]: https://www.postgresql.org/docs/18/app-pgdump.html "PostgreSQL: Documentation: 18: pg_dump"

Official references: 
[PostgreSQL 18 pg_dump documentation](https://www.postgresql.org/docs/18/app-pgdump.html) 
[pg_restore documentation](https://www.postgresql.org/docs/18/app-pgrestore.html) 
[pg_basebackup documentation](https://www.postgresql.org/docs/18/app-pgbasebackup.html) 
[pg_combinebackup documentation](https://www.postgresql.org/docs/18/app-pgcombinebackup.html)

[1]: https://www.postgresql.org/docs/18/app-pgdump.html "PostgreSQL: Documentation: 18: pg_dump"
[2]: https://www.postgresql.org/docs/18/app-pg-dumpall.html "PostgreSQL: Documentation: 18: pg_dumpall"
[3]: https://www.postgresql.org/docs/18/app-pgbasebackup.html "PostgreSQL: Documentation: 18: pg_basebackup"
[4]: https://www.postgresql.org/docs/18/continuous-archiving.html "PostgreSQL: Documentation: 18: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
