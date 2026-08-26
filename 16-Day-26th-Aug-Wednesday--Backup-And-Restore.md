# PostgreSQL 18 – Backup and Restore (16-Day-26th-Aug-Wednesday)

## 1. Backup Types in PostgreSQL

PostgreSQL backup methods can be broadly divided into:

### Logical Backup

Logical backups export database objects and/or data in a logical form.

Main utilities:

```text
pg_dump
pg_restore
pg_dumpall
psql
```

Typical use cases:

* Backup a table
* Backup selected tables
* Backup a schema
* Backup selected schemas
* Backup one database
* Backup all databases
* Backup roles and tablespaces
* Migrate logical objects between PostgreSQL systems

### Physical Backup

Physical backups copy the PostgreSQL database cluster files.

Main utilities:

```text
pg_basebackup
pg_combinebackup
pg_verifybackup
```

Typical use cases:

* Full physical backup
* Incremental physical backup
* Standby creation
* Disaster recovery
* Point-in-Time Recovery when combined with WAL archiving

---

# 2. Logical Backup Architecture

A useful way to remember the PostgreSQL utilities is:

| Requirement                          | Utility                     |
| ------------------------------------ | --------------------------- |
| One table                            | `pg_dump`                   |
| Multiple tables                      | `pg_dump`                   |
| One schema                           | `pg_dump`                   |
| Multiple schemas                     | `pg_dump`                   |
| One database                         | `pg_dump`                   |
| Entire cluster logical dump          | `pg_dumpall`                |
| Roles/tablespaces only               | `pg_dumpall --globals-only` |
| Restore plain SQL dump               | `psql`                      |
| Restore custom/tar/directory archive | `pg_restore`                |
| Physical cluster backup              | `pg_basebackup`             |
| Combine incremental physical backups | `pg_combinebackup`          |

**Important:** `pg_dump` operates on one database at a time. Cluster-wide objects such as roles and tablespaces are handled by `pg_dumpall`.

---

# 3. Lab Environment Setup

Create the test database:

```sql
CREATE DATABASE mydb;
```

Connect to it:

```text
\c mydb
```

Create three schemas:

```sql
CREATE SCHEMA raj1;
CREATE SCHEMA raj2;
CREATE SCHEMA raj3;
```

Create tables:

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

Create equivalent tables under `raj2` and `raj3`.

Verify:

```text
\dt raj1.*
\dt raj2.*
\dt raj3.*
```

Insert sample data:

```sql
INSERT INTO raj1.emp
VALUES
(1,'ram'),
(2,'sam');
```

If this statement is executed twice, `raj1.emp` contains four rows.

Verify:

```sql
SELECT count(*) FROM raj1.emp;
```

---

# 4. Scenario 1 – Table Backup in Plain Format

Create a backup directory:

```bash
mkdir -p /var/lib/pgsql/backup
cd /var/lib/pgsql/backup
```

Take backup:

```bash
pg_dump -d mydb -t raj1.emp -U postgres > emp_backup.sql
```

Better production-style syntax:

```bash
pg_dump \
  -d mydb \
  -U postgres \
  -t raj1.emp \
  -Fp \
  -f emp_backup.sql
```

`-Fp` means plain format.

The dump normally contains:

* Session settings
* `CREATE TABLE`
* Ownership information
* Table data
* `COPY` commands

Example:

```sql
CREATE TABLE raj1.emp
(
    eno integer,
    ename character varying
);

COPY raj1.emp (eno, ename) FROM stdin;
1    ram
2    sam
\.
```

## Security Note

A plain SQL dump can expose application data because it is human-readable.

Therefore:

* Restrict file permissions.
* Restrict access to backup directories.
* Encrypt backups at rest where required.
* Encrypt backup transfers.
* Do not place production dumps in public/shared directories.
* Treat backup files with the same sensitivity as the source database.

Saying that plain format itself is a "security violation" is too broad. The real issue is **how the backup is protected**.

---

# 5. Scenario 2 – Restore a Plain Table Backup

Plain SQL dumps are restored using `psql`.

Example:

```bash
psql -d mydb -U postgres -f emp_backup.sql
```

Capture output:

```bash
psql \
  -d mydb \
  -U postgres \
  -f emp_backup.sql \
  -o restore.log
```

If `raj1.emp` already exists and the dump contains:

```sql
CREATE TABLE raj1.emp ...
```

the restore can fail because the object already exists.

One option is:

```sql
DROP TABLE raj1.emp;
```

Then:

```bash
psql -d mydb -U postgres -f emp_backup.sql
```

### Important Correction

It is not correct to say that `psql` itself must provide a special "drop and recreate" option.

`psql` executes the SQL contained in the dump.

For plain dumps, cleanup commands can also be generated at **dump time** using suitable `pg_dump` options such as `--clean`.

For archive-format backups, cleanup is normally controlled during `pg_restore`.

---

# 6. Scenario 3 – Structure-Only Table Backup

Use:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  --schema-only \
  -f emp_struct.sql
```

Short form:

```bash
pg_dump -d mydb -t raj1.emp -s -f emp_struct.sql
```

The backup contains the object definition but not table rows.

Typical content:

```sql
CREATE TABLE raj1.emp
(
    eno integer,
    ename character varying
);
```

Use cases:

* Recreate table definitions
* Compare structures
* Migration preparation
* Development/test environment creation

---

# 7. Scenario 4 – Data-Only Table Backup

Use:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  --data-only \
  -f emp_data.sql
```

Short form:

```bash
pg_dump -d mydb -t raj1.emp -a -f emp_data.sql
```

The target table must already exist before restoring this file.

Restore:

```bash
psql -d mydb -f emp_data.sql
```

---

# 8. Scenario 5 – Restore Structure and Data Separately

First restore structure:

```bash
psql \
  -d mydb \
  -f emp_struct.sql \
  -o struct_restore.log
```

Then restore data:

```bash
psql \
  -d mydb \
  -f emp_data.sql \
  -o data_restore.log
```

Verify:

```sql
SELECT count(*) FROM raj1.emp;
```

This method is useful when schema deployment and data loading need to be controlled separately.

---

# 9. pg_dump Backup Formats

PostgreSQL supports several dump formats.

| Format    | Option | Human Readable |                       Compression | Restore Tool | Parallel Dump |
| --------- | ------ | -------------: | --------------------------------: | ------------ | ------------: |
| Plain     | `-Fp`  |            Yes | Not inherently archive-compressed | `psql`       |            No |
| Custom    | `-Fc`  |             No |                               Yes | `pg_restore` |            No |
| Tar       | `-Ft`  |             No |        Depends on options/content | `pg_restore` |            No |
| Directory | `-Fd`  |             No |                    Yes by default | `pg_restore` |           Yes |

### Important Correction

Do not describe custom format simply as "binary format."

The PostgreSQL term is **custom archive format**.

Also, tar format should not be selected merely because "tar means smaller." Compression behavior depends on PostgreSQL options and format characteristics.

---

# 10. Scenario 6A – Custom Format Backup

Take backup:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -Fc \
  -f emp.dump \
  -v \
  2>emp_backup.log
```

Restore:

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t emp \
  -c \
  -v \
  emp.dump \
  2>restore.log
```

`-c` / `--clean` tells `pg_restore` to drop objects before restoring them.

A useful production option is:

```bash
--if-exists
```

Example:

```bash
pg_restore \
  -d mydb \
  --clean \
  --if-exists \
  emp.dump
```

---

# 11. Important pg_restore `-d` and `-f` Difference

This command is incorrect for direct restore:

```bash
pg_restore -d mydb -f output.sql emp.dump
```

when the intention is to restore directly and simultaneously write SQL output using `-f`.

`-d` means:

> Connect to the database and execute the restore.

`-f` means:

> Generate restore SQL into a file.

For direct restore:

```bash
pg_restore -d mydb emp.dump
```

To generate SQL instead:

```bash
pg_restore -f restore.sql emp.dump
```

Then execute it separately if required:

```bash
psql -d mydb -f restore.sql
```

---

# 12. Scenario 6B – Tar Format

Backup:

```bash
pg_dump \
  -d mydb \
  -Ft \
  -t raj1.emp \
  -f emp.tar \
  -v \
  2>emp_backup.log
```

Restore:

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t emp \
  -c \
  -v \
  emp.tar \
  2>restore.log
```

Usually, specifying `-Ft` during restore is unnecessary because `pg_restore` can detect the archive format.

---

# 13. Parallel Backup with Tar Format

Example:

```bash
pg_dump \
  -d mydb \
  -Ft \
  -t raj1.emp \
  -j 2 \
  -f emp.tar
```

This fails because parallel dump is supported only with **directory format**.

Expected message:

```text
parallel backup only supported by the directory format
```

---

# 14. Scenario 6C – Directory Format

Backup:

```bash
pg_dump \
  -Fd \
  -d mydb \
  -t raj1.emp \
  -f empdir \
  -v \
  2>emp_backup.log
```

Restore:

```bash
pg_restore \
  -Fd \
  -d mydb \
  -n raj1 \
  -t emp \
  -c \
  -v \
  empdir \
  2>restore.log
```

Directory format is especially useful for large databases because it supports parallel dumps.

---

# 15. Scenario 6D – Parallel Backup and Restore

Parallel dump:

```bash
pg_dump \
  -Fd \
  -d mydb \
  -t raj1.emp \
  -j 2 \
  -f empdir \
  -v \
  2>backup.log
```

Parallel restore:

```bash
pg_restore \
  -Fd \
  -d mydb \
  -n raj1 \
  -t emp \
  -c \
  -j 2 \
  -v \
  empdir \
  2>restore.log
```

## Understanding `-j`

`-j` means the number of parallel jobs.

Do not blindly set it equal to CPU count.

Consider:

* CPU capacity
* Disk IOPS
* Storage throughput
* Number and size of tables
* Database workload
* Network throughput for remote operations
* Locks created by parallel dump workers
* Impact on production users

Start conservatively and benchmark.

---

# 16. Important Parallel Restore Detail

Parallel **dump** requires directory format.

Parallel **restore** can be used with suitable custom and directory archives.

Therefore, remember:

```text
pg_dump -j     → Directory format required

pg_restore -j  → Custom or directory archives can support parallel restore
```

---

# 17. Backup Multiple Tables

Example:

```bash
pg_dump \
  -d mydb \
  -t raj1.emp \
  -t raj1.hr \
  -t raj1.dept \
  -Fc \
  -f multi.dump \
  -v \
  2>backup.log
```

This creates one archive containing the selected tables.

---

# 18. Restore Multiple Tables

Example:

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t emp \
  -t dept \
  -t hr \
  -c \
  multi.dump \
  -v \
  2>restore.log
```

---

# 19. Restore One Table from a Multi-Table Backup

Example:

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -t hr \
  -c \
  multi.dump \
  -v \
  2>restore.log
```

This is an important advantage of archive formats: they allow selective restoration.

### Dependency Warning

When selectively restoring a table, PostgreSQL does not automatically guarantee that every dependency required by that table will also be restored.

Always validate:

* Schema
* Sequences
* Types
* Functions
* Extensions
* Constraints
* Referenced objects

---

# 20. Inspect Archive Contents

Use:

```bash
pg_restore --list multi.dump
```

Short form:

```bash
pg_restore -l multi.dump
```

This displays the archive Table of Contents.

It can show objects such as:

```text
TABLE raj1 dept
TABLE raj1 emp
TABLE raj1 hr
TABLE DATA raj1 dept
TABLE DATA raj1 emp
TABLE DATA raj1 hr
```

This is more reliable than depending only on the verbose backup log to understand archive contents.

---

# 21. Schema Backup

Backup complete schema:

```bash
pg_dump \
  -d mydb \
  -n raj1 \
  -Fc \
  -f raj1_schema.dump \
  -v \
  2>schema_backup.log
```

This includes objects selected by the schema pattern and their associated data unless `--schema-only` is also specified.

Restore:

```bash
pg_restore \
  -d mydb \
  -n raj1 \
  -c \
  raj1_schema.dump \
  -v \
  2>restore.log
```

---

# 22. Schema Backup Excluding a Table

Example:

```bash
pg_dump \
  -Fc \
  -d mydb \
  -n raj1 \
  -T raj1.emp \
  -f raj1_without_emp.dump \
  -v \
  2>backup.log
```

Here:

```text
-n
```

includes a schema.

```text
-T
```

excludes matching tables.

---

# 23. Backup Multiple Schemas

Example:

```bash
pg_dump \
  -Fc \
  -d mydb \
  -n raj1 \
  -n raj2 \
  -n raj3 \
  -f multischema.dump \
  -v \
  2>backup.log
```

Restore only `raj2`:

```bash
pg_restore \
  -d mydb \
  -n raj2 \
  -c \
  multischema.dump \
  -v \
  2>restore.log
```

---

# 24. Complete Database Backup

Correct backup command:

```bash
pg_dump \
  -Fc \
  -d mydb \
  -f mydb.dump \
  -v \
  2>backup.log
```

Restore:

```bash
pg_restore \
  -d mydb \
  -c \
  mydb.dump \
  -v \
  2>restore.log
```

### Correction from Original Notes

The command:

```bash
pg_restore -Fc -d mydb -c db.bin
```

is a **restore command**, not a command for taking the database backup.

The backup must first be created with `pg_dump`.

---

# 25. Restore into Another Database

A logical database dump can be restored into another existing database.

Create target:

```sql
CREATE DATABASE saidb;
```

Restore:

```bash
pg_restore \
  -d saidb \
  mydb.dump \
  -v \
  2>restore.log
```

Verify:

```text
\c saidb
\dn
\dt raj1.*
\dt raj2.*
\dt raj3.*
```

This is useful for:

* Database refresh
* Testing
* Migration
* Cloning logical database contents

---

# 26. Schema/Table Remapping

PostgreSQL `pg_restore` does not provide an Oracle Data Pump-style:

```text
REMAP_SCHEMA
REMAP_TABLE
```

Directly restoring a dump into another database is straightforward, but changing schema/table names during restore generally requires a different migration strategy or post-restore changes.

---

# 27. Entire Cluster Logical Backup – pg_dumpall

Use:

```bash
pg_dumpall > fullbackup.sql
```

Better:

```bash
pg_dumpall -f fullbackup.sql
```

`pg_dumpall` creates a plain SQL script.

It handles:

* Databases
* Database objects
* Roles
* Tablespaces
* Cluster-level information supported by the utility

Restore:

```bash
psql -X -f fullbackup.sql postgres
```

`-X` prevents user `psqlrc` configuration from affecting the restore.

---

# 28. Global Objects Backup

Use:

```bash
pg_dumpall --globals-only > globals.sql
```

Short form:

```bash
pg_dumpall -g > globals.sql
```

Global objects include primarily:

* Roles
* Tablespaces
* Relevant cluster-wide grants/settings

Restore:

```bash
psql -X -d postgres -f globals.sql
```

---

# 29. Security Warning – Role Passwords

A global dump can contain PostgreSQL role password verifiers.

For example:

```text
SCRAM-SHA-256$...
```

These are not plaintext passwords, but they are still sensitive authentication material.

Protect `globals.sql` carefully.

PostgreSQL also provides:

```bash
pg_dumpall --globals-only --no-role-passwords
```

when role password verifiers should not be included.

---

# 30. Parallel Logical Backup Strategy for Multiple Databases

Assume:

```text
db1 = 100 GB
db2 = 100 GB
db3 = 100 GB
db4 = 100 GB
db5 = 100 GB
```

Instead of serially dumping every database, independent database backups can run concurrently.

Example:

```bash
pg_dump -d db1 -Fd -f /backup/db1 -j 2
pg_dump -d db2 -Fd -f /backup/db2 -j 2
pg_dump -d db3 -Fd -f /backup/db3 -j 2
pg_dump -d db4 -Fd -f /backup/db4 -j 2
pg_dump -d db5 -Fd -f /backup/db5 -j 2
```

Also back up global objects:

```bash
pg_dumpall -g > /backup/globals.sql
```

### Production Warning

Running five databases simultaneously with two workers each does **not** automatically mean the operation will finish in the time required for one database.

Performance depends on:

* CPU
* Storage
* Network
* Cache
* Database activity
* Number of objects
* Compression
* Worker count

Therefore, statements such as "all backups will complete in 5 minutes" should be treated as lab examples, not production guarantees.

---

# 31. Remote Logical Backup

Example:

```bash
pg_dump \
  -h 192.168.56.201 \
  -U postgres \
  -d db1 \
  -Fd \
  -f /tmp/backup/db1 \
  -j 2 \
  -v \
  2>db1_backup.log
```

Important:

Use standard ASCII hyphens:

```text
-U
```

not typographic dashes copied from formatted documents.

The backup is written on the machine where `pg_dump` itself is running.

---

# 32. Physical Backup with pg_basebackup

`pg_basebackup` creates a physical backup of the entire PostgreSQL cluster.

Example:

```bash
pg_basebackup \
  -D /var/lib/pgsql/backup \
  -X stream \
  -P
```

Important properties:

* Entire PostgreSQL cluster is backed up.
* Individual tables cannot be selected.
* Individual databases cannot be selected.
* Suitable for physical recovery.
* Commonly used to build physical standby servers.
* Can be combined with WAL archiving for PITR.

---

# 33. `-X fetch` vs `-X stream`

Example from the lab:

```bash
pg_basebackup \
  -D /var/lib/pgsql/backup \
  -X fetch \
  -P \
  --checkpoint=fast
```

With:

```text
-X fetch
```

required WAL is collected at the end of the backup.

There is a risk: if required WAL is recycled before collection, the backup can fail.

For many environments:

```text
-X stream
```

is safer because WAL is streamed while the backup is running.

Example:

```bash
pg_basebackup \
  -D /var/lib/pgsql/backup \
  -X stream \
  -P
```

---

# 34. Fast Checkpoint

The original command used:

```text
-cfast
```

A clearer form is:

```bash
--checkpoint=fast
```

Example:

```bash
pg_basebackup \
  -D /var/lib/pgsql/backup \
  -X stream \
  -P \
  --checkpoint=fast
```

### Production Note

A fast checkpoint can create additional I/O pressure.

Do not use it blindly on a heavily loaded production system.

---

# 35. Base Backup Contents

A plain-format physical backup can contain directories/files such as:

```text
base/
global/
pg_wal/
pg_xact/
pg_multixact/
pg_tblspc/
postgresql.conf
pg_hba.conf
PG_VERSION
backup_label
backup_manifest
```

It represents the physical state of the PostgreSQL cluster.

---

# 36. Verify Physical Backup

A major production improvement is to verify the base backup.

Use:

```bash
pg_verifybackup /var/lib/pgsql/backup
```

A backup should not be considered operationally reliable simply because `pg_basebackup` returned successfully.

Regular restore testing is also essential.

---

# 37. WAL Archive Cleanup

The lab uses:

```bash
pg_archivecleanup \
  /var/lib/pgsql/archive \
  0000000100000000000000CA.00000028.backup
```

`pg_archivecleanup` can remove archived WAL files older than the specified cutoff.

### Critical Production Warning

Do **not** automatically delete WAL merely because a new backup completed.

Old WAL might still be required for:

* Point-in-Time Recovery
* Older retained backups
* Standby recovery
* Recovery SLA requirements
* Other consumers of the archive

Retention must be based on the complete recovery policy.

---

# 38. Restore Physical Backup as Separate Instance

Suppose the original instance uses:

```text
Port: 5432
Data directory: /var/lib/pgsql/18/data
```

A restored copy can be tested separately.

Edit the restored configuration:

```text
port = 5433
cluster_name = 'backupcluster'
```

Set ownership and permissions:

```bash
chown -R postgres:postgres /var/lib/pgsql/backup
chmod 0700 /var/lib/pgsql/backup
```

Start:

```bash
/usr/pgsql-18/bin/pg_ctl \
  -D /var/lib/pgsql/backup \
  start
```

Connect:

```bash
psql -p 5433
```

Verify:

```sql
SHOW port;
SHOW data_directory;
```

---

# 39. Why Separate Restore Testing Is Preferred

Testing a backup as a separate PostgreSQL instance is generally safer because the existing production instance remains untouched.

Benefits:

* No overwrite of production data
* Easier validation
* Easier comparison
* Lower recovery-test risk
* Can verify databases and objects before cutover

Requirements:

* Separate port
* Separate data directory
* Enough storage
* Correct ownership
* Correct permissions
* No conflicting tablespace paths
* Care with Unix-domain socket locations and other shared resources

---

# 40. Restoring Over Existing Instance

High-level lab process:

```bash
systemctl stop postgresql-18
```

Move existing data directory:

```bash
mv /var/lib/pgsql/18/data \
   /var/lib/pgsql/18/data_old
```

Move restored backup:

```bash
mv /var/lib/pgsql/backup \
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

### Warning

This is destructive if performed incorrectly.

Before performing it in production:

* Confirm backup validity.
* Confirm PostgreSQL version compatibility.
* Confirm tablespace paths.
* Preserve the old data directory.
* Validate configuration.
* Have a rollback procedure.
* Confirm recovery requirements.

---

# 41. PostgreSQL Incremental Physical Backup

PostgreSQL supports incremental physical base backups.

The feature works with:

```text
pg_basebackup --incremental
pg_combinebackup
WAL summarization
```

An incremental backup contains changed relation blocks compared with the reference backup, while some other files are still included fully.

---

# 42. Enable WAL Summarization

Incremental backup requires:

```text
summarize_wal = on
```

Check:

```sql
SHOW summarize_wal;
```

If it is disabled, an incremental backup fails with an error indicating that WAL summarization must be enabled.

### Important Correction

`summarize_wal` is not a simple reload-only parameter.

Plan to configure it correctly and **restart PostgreSQL** when required rather than assuming:

```bash
pg_ctl reload
```

is sufficient.

---

# 43. WAL Summary Retention

PostgreSQL stores WAL summary information used for incremental backup.

Relevant parameter:

```text
wal_summary_keep_time
```

The retention period must be long enough to preserve summaries required by future incremental backups.

If required WAL summaries disappear, a later incremental backup depending on them can fail.

---

# 44. Take Full Physical Backup

Example:

```bash
pg_basebackup \
  -D /var/lib/pgsql/fullbackup \
  -X stream \
  -P
```

The backup creates:

```text
backup_manifest
```

This manifest is important for incremental backup operations.

---

# 45. Take Incremental Backup

Reference the earlier backup manifest:

```bash
pg_basebackup \
  -D /var/lib/pgsql/incremental \
  -X stream \
  -P \
  --incremental=/var/lib/pgsql/fullbackup/backup_manifest
```

The reference manifest must come from an appropriate earlier backup from the same server lineage.

---

# 46. Incremental Backup Cannot Be Started Directly

An incremental backup is not itself a normal standalone PostgreSQL data directory.

It must first be reconstructed with:

```text
pg_combinebackup
```

---

# 47. Combine Full and Incremental Backup

Example:

```bash
pg_combinebackup \
  /var/lib/pgsql/fullbackup \
  /var/lib/pgsql/incremental \
  -o /var/lib/pgsql/finalbackup
```

The output is a reconstructed **synthetic full backup**.

Verify it:

```bash
pg_verifybackup /var/lib/pgsql/finalbackup
```

Then the reconstructed backup can be used for recovery.

---

# 48. Incremental Backup Chain – Important Correction

Suppose:

```text
Sunday    Full backup
Monday    Incremental 1
Tuesday   Incremental 2
Wednesday Incremental 3
```

Whether Wednesday depends only on Sunday or on the complete chain depends on **which backup manifest was used as the reference for each incremental backup**.

If each incremental references the original Sunday full backup:

```text
Sunday Full
   ├── Monday Incremental
   ├── Tuesday Incremental
   └── Wednesday Incremental
```

Wednesday can be reconstructed using its actual dependency relationship.

However, if backups form a chain:

```text
Full → Inc1 → Inc2 → Inc3
```

then all required dependent backups must be supplied to `pg_combinebackup`, oldest to newest.

Therefore, never assume:

```text
Full + latest incremental is always enough
```

unless the latest incremental actually references that full backup directly.

---

# 49. Chained Incremental Example

Example:

```bash
pg_combinebackup \
  /backup/full \
  /backup/inc1 \
  /backup/inc2 \
  /backup/inc3 \
  -o /backup/final
```

The backups must be supplied:

```text
oldest → newest
```

`pg_combinebackup` validates whether the supplied backups form a valid dependency chain.

---

# 50. Full vs Incremental Backup

| Area                       | Full Physical Backup | Incremental Physical Backup   |
| -------------------------- | -------------------- | ----------------------------- |
| Tool                       | `pg_basebackup`      | `pg_basebackup --incremental` |
| Size                       | Larger               | Usually smaller               |
| Backup time                | Usually longer       | Usually shorter               |
| Standalone                 | Yes                  | No                            |
| WAL summarization required | No                   | Yes                           |
| Combine before restore     | No                   | Yes                           |
| Complexity                 | Lower                | Higher                        |
| Storage                    | Higher               | Potentially lower             |

---

# 51. Logical vs Physical Backup

| Feature                       | Logical          | Physical                    |
| ----------------------------- | ---------------- | --------------------------- |
| Main tool                     | `pg_dump`        | `pg_basebackup`             |
| Table-level backup            | Yes              | No                          |
| Schema-level backup           | Yes              | No                          |
| Single database               | Yes              | No                          |
| Entire cluster                | Via `pg_dumpall` | Yes                         |
| Selective restore             | Strong           | Limited                     |
| Human-readable option         | Yes              | No                          |
| Standby creation              | No               | Yes                         |
| PITR foundation               | No               | Yes                         |
| Incremental physical backup   | No               | Yes                         |
| Cross-major-version migration | Often suitable   | Not direct physical restore |

---

# 52. Production Backup Design

A production PostgreSQL backup strategy should normally define all of the following:

### RPO

Recovery Point Objective:

> How much data can the business afford to lose?

Example:

```text
RPO = 15 minutes
```

### RTO

Recovery Time Objective:

> How quickly must PostgreSQL be restored?

Example:

```text
RTO = 1 hour
```

The backup design must support both.

---

# 53. Recommended Production Layers

A mature PostgreSQL environment normally uses more than one backup mechanism.

Example:

```text
Logical Backups
        +
Physical Base Backups
        +
Continuous WAL Archiving
        +
Off-site Backup Copies
        +
Backup Verification
        +
Regular Restore Testing
```

Each solves a different problem.

---

# 54. Backup Security

Backups contain production information and must be protected.

Recommended controls:

* Encryption at rest
* Encryption in transit
* Restricted OS permissions
* Separate backup service accounts
* Restricted network access
* Off-site copies
* Immutable/object-locked copies where appropriate
* Audit logging
* Defined retention
* Secure deletion after retention expires
* Protection of role password verifiers

Example:

```bash
chmod 0700 /backup
```

Do not rely only on filesystem permissions if the backup contains highly sensitive information.

---

# 55. Production Restore Validation Checklist

After restoring a PostgreSQL database, verify:

```text
1. PostgreSQL starts successfully
2. Correct PostgreSQL version
3. Correct databases exist
4. Correct schemas exist
5. Tables exist
6. Row counts are reasonable
7. Roles exist
8. Ownership is correct
9. Privileges are correct
10. Extensions exist
11. Sequences are correct
12. Application login works
13. Tablespaces are valid
14. Database logs show no critical errors
15. Application queries work
```

For logical restores, consider running:

```bash
vacuumdb -a -z
```

or appropriate `ANALYZE` operations so the optimizer has current statistics.

---

# 56. Backup Verification Checklist

Before declaring a backup successful:

```text
[ ] Backup command completed successfully
[ ] Exit status checked
[ ] Backup size checked
[ ] Backup log reviewed
[ ] Archive contents inspected where applicable
[ ] Physical backup verified with pg_verifybackup
[ ] Backup copied to protected storage
[ ] Retention policy applied
[ ] Restore test completed
[ ] Recovery time recorded
```

---

# 57. Common Commands – Quick Reference

## Table backup

```bash
pg_dump -d mydb -t raj1.emp -Fc -f emp.dump
```

## Structure only

```bash
pg_dump -d mydb -t raj1.emp -s -f emp_struct.sql
```

## Data only

```bash
pg_dump -d mydb -t raj1.emp -a -f emp_data.sql
```

## Schema

```bash
pg_dump -d mydb -n raj1 -Fc -f raj1.dump
```

## Exclude table

```bash
pg_dump -d mydb -n raj1 -T raj1.emp -Fc -f raj1.dump
```

## Database

```bash
pg_dump -d mydb -Fc -f mydb.dump
```

## Multiple schemas

```bash
pg_dump \
  -d mydb \
  -n raj1 \
  -n raj2 \
  -n raj3 \
  -Fc \
  -f schemas.dump
```

## Parallel dump

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
  mydb_dir
```

## Globals

```bash
pg_dumpall -g > globals.sql
```

## Entire logical cluster

```bash
pg_dumpall > cluster.sql
```

## Physical backup

```bash
pg_basebackup \
  -D /backup/full \
  -X stream \
  -P
```

## Verify physical backup

```bash
pg_verifybackup /backup/full
```

## Incremental backup

```bash
pg_basebackup \
  -D /backup/inc1 \
  -X stream \
  -P \
  --incremental=/backup/full/backup_manifest
```

## Combine backups

```bash
pg_combinebackup \
  /backup/full \
  /backup/inc1 \
  -o /backup/final
```

---

# 58. Important Interview Points

### Q: Does pg_dump back up the entire PostgreSQL server?

No.

`pg_dump` handles one database per invocation.

---

### Q: How do we back up roles and tablespaces?

Use:

```bash
pg_dumpall --globals-only
```

---

### Q: Which pg_dump format supports parallel backup?

Directory format:

```text
-Fd
```

---

### Q: Can custom format be restored in parallel?

Yes, suitable custom archives can be restored using `pg_restore -j`.

---

### Q: Can plain SQL dump be restored using pg_restore?

No.

Use:

```bash
psql
```

---

### Q: Which utility restores custom, tar and directory archives?

```text
pg_restore
```

---

### Q: Can pg_basebackup back up one table?

No.

It backs up the physical PostgreSQL cluster.

---

### Q: Can an incremental pg_basebackup be started directly?

No.

Reconstruct a full backup first with:

```text
pg_combinebackup
```

---

### Q: What feature must be enabled for incremental physical backups?

```text
summarize_wal = on
```

---

### Q: How should a physical backup be validated?

Use:

```bash
pg_verifybackup
```

and perform regular real restore tests.

---

# 59. Key Production Lessons

1. **Do not depend on only one backup type.**

   Logical and physical backups solve different recovery problems.

2. **A completed backup command does not prove recoverability.**

   Restore testing is the real proof.

3. **Protect backup files like production data.**

   Backups can contain both business data and authentication information.

4. **Use parallelism carefully.**

   More jobs do not automatically mean faster backup.

5. **Prefer tested restore procedures.**

   During an outage, the DBA should follow an already tested runbook.

6. **Maintain WAL according to recovery requirements.**

   Never remove archived WAL simply because it looks old.

7. **Use backup manifests and verification.**

   Physical backups should be validated with `pg_verifybackup`.

8. **Know the dependency chain of incremental backups.**

   Losing a required parent backup can make a dependent incremental backup unusable.

9. **Measure RPO and RTO.**

   Backup architecture should be driven by business recovery requirements.

10. **Regularly perform disaster-recovery drills.**

    A backup strategy is incomplete until the organization has proven it can restore the database within the required recovery window.

---

# 60. Final Architecture View

```text
                    PostgreSQL Backup Strategy
                              |
              +---------------+---------------+
              |                               |
        Logical Backup                  Physical Backup
              |                               |
       pg_dump / pg_dumpall              pg_basebackup
              |                               |
     +--------+--------+              +-------+--------+
     |        |        |              |                |
   Table    Schema   Database       Full          Incremental
     |        |        |              |                |
   psql / pg_restore                  |        summarize_wal
                                      |                |
                                      +------> pg_combinebackup
                                               |
                                               v
                                      Synthetic Full Backup
                                               |
                                        pg_verifybackup
                                               |
                                               v
                                         Restore Test
```

## Golden Rule

> **A backup is valuable only when it can be restored successfully within the required RPO and RTO.**

The main technical inputs above are based on the PostgreSQL 18 documentation: 

`pg_dump` is a single-database logical export tool, while directory format is required for parallel **dumping**; 

parallel `pg_restore` can work with custom or directory archives. ([PostgreSQL][1]) 

`pg_dumpall` handles all databases plus cluster-wide objects, and `--globals-only` covers roles and tablespaces. ([PostgreSQL][2])

Incremental-backup section: 
PostgreSQL 18 incremental base backups require WAL summarization, cannot be used directly, and must be reconstructed with `pg_combinebackup`; 

when there is a dependency chain, the required backups are supplied oldest to newest. ([PostgreSQL][3]) 

The WAL cleanup warning is important as well—archive retention has to account for all backups, PITR needs, and any standby consumers before older WAL is removed. ([PostgreSQL][4])

Official references: 
[PostgreSQL 18 pg_dump documentation](https://www.postgresql.org/docs/18/app-pgdump.html) 
[pg_restore documentation](https://www.postgresql.org/docs/18/app-pgrestore.html) 
[pg_basebackup documentation](https://www.postgresql.org/docs/18/app-pgbasebackup.html) 
[pg_combinebackup documentation](https://www.postgresql.org/docs/18/app-pgcombinebackup.html)

[1]: https://www.postgresql.org/docs/18/app-pgdump.html "PostgreSQL: Documentation: 18: pg_dump"
[2]: https://www.postgresql.org/docs/18/app-pg-dumpall.html "PostgreSQL: Documentation: 18: pg_dumpall"
[3]: https://www.postgresql.org/docs/18/app-pgbasebackup.html "PostgreSQL: Documentation: 18: pg_basebackup"
[4]: https://www.postgresql.org/docs/18/continuous-archiving.html "PostgreSQL: Documentation: 18: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
