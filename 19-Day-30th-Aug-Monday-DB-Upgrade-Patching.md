# PostgreSQL 18 – Database Upgrade & Patching

**Class Date:** 31st August 2026 – Monday

**Topic:** PostgreSQL Database Upgrade, Minor Version Upgrade & Patching

**Lab Environment:** Ubuntu 26.04 | PostgreSQL 16.15 → PostgreSQL 18.6 | `dvdrental`

---

# Index

1. Database Upgrade and Patching – Basics
2. PostgreSQL Version Numbering
3. Major vs Minor Version Upgrade
4. Why Database Upgrades Are Required
5. PostgreSQL Major Upgrade Methods
6. `pg_upgrade` Utility
7. PostgreSQL 16 to 18 – Upgrade Using Copy Mode
8. Load / Validate Sample Database
9. Major Upgrade Pre-Checks
10. CPU, Memory and Capacity
11. PostgreSQL 18 Installation and Initialization
12. Data Checksum Compatibility
13. Configure Temporary PostgreSQL 18 Port
14. Configuration Migration
15. Extension Pre-Check
16. `pg_upgrade --check`
17. Production Upgrade Preparation
18. Capture Tested Baseline Information
19. Stop PostgreSQL 16
20. Actual PostgreSQL 16 → 18 Upgrade – Copy Mode
21. PostgreSQL 18 `pg_upgrade` Statistics Improvement
22. Configure Production Port
23. Start PostgreSQL 18
24. Validate DVD Rental Database
25. Additional Post-Upgrade Validation
26. Statistics Maintenance
27. Application Handover
28. Rollback – Copy Mode
29. Critical Rollback Point – New Production Writes
30. PostgreSQL 18 → 16 Logical Downgrade Warning
31. Clean Up PostgreSQL 16
32. PostgreSQL 16 → 18 Link Mode Upgrade
33. Linux Inode Basics
34. Why Link Mode Uses Hard Links
35. Link Mode Filesystem Requirement
36. Link Mode Upgrade Preparation
37. Stop Application and PostgreSQL 16
38. Perform Link Mode Upgrade
39. Configure PostgreSQL 18
40. Start PostgreSQL 18
41. Critical Link-Mode Rollback Rule
42. Link Mode Rollback Window
43. PostgreSQL 18 Clone Mode
44. Copy vs Link vs Clone
45. Minor Version Upgrade / Patching
46. Current Minor Version Example
47. Ubuntu Package Validation
48. PostgreSQL 18 Minor Upgrade Preparation
49. PostgreSQL 18.x → 18.6 Minor Upgrade
50. Start PostgreSQL After Minor Upgrade
51. Minor Upgrade Production Procedure
52. Important Minor-Release Release-Notes Rule
53. Minor-Version Rollback
54. Example Controlled Package Rollback
55. Major Upgrade vs Minor Upgrade
56. Major Upgrade Options
57. `pg_upgrade` Production Checklist
58. Additional Useful Commands
59. Backup Validation Before Upgrade
60. Do Not Forget Replication
61. Do Not Delete the Old Cluster Immediately
62. Major Corrections to Original Class Notes
63. Production Recommendation – PostgreSQL 16 → 18
64. Quick Reference Cheat Sheet
65. Final DBA Takeaways
66. Official PostgreSQL References
67. Summary

---

# 1. Database Upgrade and Patching – Basics

## Upgrade

An **upgrade** means moving PostgreSQL from one version to another version.

Typical reasons include:

* Getting new PostgreSQL features.
* Moving from an older supported version to a newer supported version.
* Maintaining application compatibility.
* Improving performance or manageability.
* Moving away from an End-of-Life PostgreSQL version.

Example:

```text
PostgreSQL 16 → PostgreSQL 18
```

This is a **major version upgrade**.

---

## Patching

Patching generally means applying a PostgreSQL maintenance release containing:

* Security fixes.
* Bug fixes.
* Data corruption fixes.
* Stability fixes.
* Low-risk improvements.

PostgreSQL maintenance fixes are delivered through **minor releases**.

Example:

```text
PostgreSQL 18.4 → PostgreSQL 18.6
```

This is a **minor version update**.

---

# 2. PostgreSQL Version Numbering

Starting with PostgreSQL 10, PostgreSQL uses:

```text
Major.Minor
```

Examples:

```text
16.15
17.11
18.6
```

For:

```text
18.6
```

the meaning is:

```text
18 = Major Version
 6 = Minor Version
```

Therefore:

```text
16 → 18
```

is a **major upgrade**.

Whereas:

```text
18.4 → 18.6
```

is a **minor version update**.

Before PostgreSQL 10, the numbering worked differently.

Example:

```text
9.5.3
```

means:

```text
9.5 = Major Version
3   = Minor Release
```

---

# 3. Major vs Minor Version Upgrade

| Upgrade Type | Example | Data Directory Reusable Directly? | Typical Method |
| --- | --- | ---: | --- |
| Major Upgrade | 16 → 18 | No | `pg_upgrade`, dump/restore, logical replication |
| Minor Upgrade | 18.4 → 18.6 | Yes | Update binaries/packages and restart |
| Security/Bug Fix | Delivered through minor release | Yes | Minor version update |

A PostgreSQL major release can change:

* System catalogs.
* Internal metadata.
* SQL behavior.
* Configuration parameters.
* Extension interfaces.
* Internal structures.

Therefore PostgreSQL 16 and PostgreSQL 18 cannot simply reuse the same cluster directory without a supported major-upgrade mechanism.

For minor releases, PostgreSQL maintains physical storage compatibility within the same major release.

---

# 4. Why Database Upgrades Are Required

Database upgrades may be required for:

### New Features

PostgreSQL 18 introduced enhancements including:

* Asynchronous I/O.
* UUIDv7 support.
* B-tree skip scans.
* OAuth authentication support.
* Improved `pg_upgrade`.
* Additional SQL and monitoring enhancements.

### Security Fixes

Minor releases regularly contain security fixes.

### Bug Fixes

Minor releases can contain fixes for:

* Crashes.
* Incorrect query results.
* Replication issues.
* WAL issues.
* Data corruption risks.
* Planner issues.

### End of Life

PostgreSQL major versions are supported for approximately five years.

### Application Compatibility

Applications may require a newer PostgreSQL major release.

### Infrastructure Compatibility

Operating-system or platform changes may require newer PostgreSQL packages.

---

# 5. PostgreSQL Major Upgrade Methods

PostgreSQL major-version migrations commonly use:

## Method 1 – Logical Dump and Restore

Utilities:

```bash
pg_dump
pg_dumpall
pg_restore
psql
```

Architecture:

```text
PostgreSQL 16
      |
      | logical export
      v
Dump Files
      |
      | restore
      v
PostgreSQL 18
```

Advantages:

* Simple concept.
* Clean logical rebuild.
* Can support platform changes.
* Useful for smaller environments.

Disadvantages:

* Can take significant time.
* Requires dump storage.
* Downtime can be high for large databases.

---

## Method 2 – `pg_upgrade`

Architecture:

```text
PostgreSQL 16
      |
      | pg_upgrade
      v
PostgreSQL 18
```

Advantages:

* Much faster than full logical dump/restore for large databases.
* Reuses existing user relation files through copy/link/clone/swap methods.
* Good for same-host or same-platform major upgrades.

---

## Method 3 – Logical Replication

Useful where very low application downtime is required.

Architecture:

```text
PostgreSQL 16
      |
      | continuous logical changes
      v
PostgreSQL 18
      |
      | short cutover
      v
Application
```

This requires separate planning for sequences, DDL, large objects, replication slots, cutover, rollback and application behavior.

---

# 6. `pg_upgrade` Utility

`pg_upgrade` was officially introduced with **PostgreSQL 9.0**.

For a PostgreSQL 16 → 18 upgrade, always use the **new PostgreSQL version's `pg_upgrade` binary**:

```text
/usr/lib/postgresql/18/bin/pg_upgrade
```

`pg_upgrade` avoids logically exporting and restoring every row.

PostgreSQL 18 `pg_upgrade` supports multiple transfer modes, including:

```text
copy
link
clone
copy_file_range
swap
```

For this lab, we use the default:

```text
COPY MODE
```

---

# 7. PostgreSQL 16 to 18 – Upgrade Using Copy Mode

## Tested Lab Environment

```text
Operating System : Ubuntu 26.04
Host             : pgdb01

Source Version   : PostgreSQL 16.15
Source Cluster   : 16/main
Source Port      : 5432
Source Status    : ONLINE

Target Version   : PostgreSQL 18.6
Target Cluster   : 18/main
Preparation Port : 5433
Target Status    : DOWN

Database         : dvdrental
Upgrade Method   : pg_upgrade
Transfer Mode    : COPY
```

Important Ubuntu paths:

```text
PG16 Binary Directory
/usr/lib/postgresql/16/bin

PG18 Binary Directory
/usr/lib/postgresql/18/bin

PG16 Data Directory
/var/lib/postgresql/16/main

PG18 Data Directory
/var/lib/postgresql/18/main

PG16 Configuration Directory
/etc/postgresql/16/main

PG18 Configuration Directory
/etc/postgresql/18/main
```

---

# 8. Load / Validate Sample Database

The lab uses the `dvdrental` database.

If it is already loaded, simply validate it:

```bash
sudo -u postgres psql -p 5432 -d dvdrental
```

List tables:

```sql
\dt
```

List views:

```sql
\dv
```

The tested environment already contains the DVD Rental objects and baseline row counts shown later in these notes.

---

# 9. Major Upgrade Pre-Checks

Check clusters:

```bash
pg_lsclusters
```

Expected preparation state:

```text
16 main 5432 online
18 main 5433 down
```

Check versions:

```bash
/usr/lib/postgresql/16/bin/postgres --version
/usr/lib/postgresql/18/bin/postgres --version
```

Expected:

```text
postgres (PostgreSQL) 16.15
postgres (PostgreSQL) 18.6
```

Check source data directory:

```bash
sudo -u postgres psql -p 5432 -Atc "SHOW data_directory;"
```

Expected:

```text
/var/lib/postgresql/16/main
```

Check source port:

```bash
sudo -u postgres psql -p 5432 -Atc "SHOW port;"
```

Expected:

```text
5432
```

Check disk:

```bash
df -h /
```

Check inode availability:

```bash
df -i /
```

Check source cluster size:

```bash
sudo du -sh /var/lib/postgresql/16/main
```

Check tablespaces:

```bash
sudo -u postgres psql -p 5432 -c "
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
"
```

COPY mode requires enough storage for the new cluster and copied relation files plus WAL, temporary files and operational headroom.

---

# 10. CPU, Memory and Capacity

Check memory:

```bash
free -h
```

Check CPU:

```bash
lscpu
```

Check CPU count:

```bash
nproc
```

The old and new production clusters are not expected to serve normal production traffic simultaneously during the actual `pg_upgrade`, but sufficient CPU, memory and I/O capacity are still required for:

* Compatibility checks.
* Temporary server startup.
* File transfer.
* Statistics regeneration.
* Post-upgrade validation.

---

# 11. PostgreSQL 18 Installation and Initialization

Install PostgreSQL 18 packages from the configured PostgreSQL repository:

```bash
sudo apt update
sudo apt install postgresql-18 postgresql-client-18 postgresql-contrib-18
```

Validate:

```bash
/usr/lib/postgresql/18/bin/postgres --version
```

Check clusters:

```bash
pg_lsclusters
```

For this tested lab, PostgreSQL 18 is already prepared as cluster:

```text
18/main
```

on temporary port:

```text
5433
```

The PostgreSQL 18 cluster must be initialized before `pg_upgrade`, but it does **not** need to remain started.

---

# 12. Data Checksum Compatibility

PostgreSQL 18 enables data checksums by default for newly initialized clusters.

Check PostgreSQL 16:

```bash
sudo -u postgres psql -p 5432 -Atc "SHOW data_checksums;"
```

Tested result:

```text
off
```

Check PostgreSQL 18:

```bash
sudo -u postgres \
/usr/lib/postgresql/18/bin/pg_controldata \
/var/lib/postgresql/18/main \
| grep -i "Data page checksum"
```

For this lab, the target was initialized to match the source:

```text
PG16 checksums = OFF
PG18 checksums = OFF
```

If a PostgreSQL 18 cluster must be recreated to match a non-checksum source, initialize it with:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/initdb \
-D /var/lib/postgresql/18/main \
--no-data-checksums
```

On Ubuntu/Debian, use the normal cluster-management tooling and permissions for your environment when recreating a cluster.

Do **not** disable checksums automatically in every environment. Match the source cluster's checksum state.

---

# 13. Configure Temporary PostgreSQL 18 Port

During preparation:

```text
PG16 = 5432
PG18 = 5433
```

PostgreSQL 18 configuration file:

```text
/etc/postgresql/18/main/postgresql.conf
```

Set:

```conf
port = 5433
```

Check:

```bash
grep -E '^[[:space:]]*port[[:space:]]*=' \
/etc/postgresql/18/main/postgresql.conf
```

During the actual `pg_upgrade`, both normal cluster instances are stopped and `pg_upgrade` temporarily starts the clusters itself.

---

# 14. Configuration Migration

Do **not** blindly copy PostgreSQL 16's complete configuration over PostgreSQL 18.

Compare and review:

```bash
diff -u \
/etc/postgresql/16/main/postgresql.conf \
/etc/postgresql/18/main/postgresql.conf
```

Review important parameters such as:

```text
listen_addresses
port
max_connections
shared_buffers
work_mem
maintenance_work_mem
wal_level
max_wal_senders
max_replication_slots
archive_mode
archive_command
shared_preload_libraries
ssl
logging settings
```

Also review:

```text
pg_hba.conf
pg_ident.conf
postgresql.auto.conf
include
include_if_exists
include_dir
```

Migrate only settings that are still valid and appropriate for PostgreSQL 18.

---

# 15. Extension Pre-Check

Check installed extensions:

```bash
sudo -u postgres psql -p 5432 -d dvdrental -c "
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
"
```

If third-party extensions exist, install their PostgreSQL 18-compatible binaries/libraries before running `pg_upgrade`.

Do not copy PostgreSQL 16 extension `.so` files into PostgreSQL 18.

---

# 16. `pg_upgrade --check`

Create a working directory:

```bash
sudo mkdir -p /var/lib/postgresql/pg_upgrade_work
sudo chown postgres:postgres /var/lib/postgresql/pg_upgrade_work
sudo chmod 700 /var/lib/postgresql/pg_upgrade_work
```

Check cluster state:

```bash
pg_lsclusters
```

Expected:

```text
16 main 5432 online
18 main 5433 down
```

Run:

```bash
sudo -u postgres bash -c '
cd /var/lib/postgresql/pg_upgrade_work

/usr/lib/postgresql/18/bin/pg_upgrade \
  --old-bindir=/usr/lib/postgresql/16/bin \
  --new-bindir=/usr/lib/postgresql/18/bin \
  --old-datadir=/etc/postgresql/16/main \
  --new-datadir=/etc/postgresql/18/main \
  --old-options="-D /var/lib/postgresql/16/main" \
  --new-options="-D /var/lib/postgresql/18/main" \
  --old-port=5432 \
  --new-port=5433 \
  --username=postgres \
  --check
'
```

For PostgreSQL 18, `-d/--old-datadir` and `-D/--new-datadir` refer to the old/new **cluster configuration directories**. Ubuntu/Debian stores configuration separately from the physical data directory, so the physical PGDATA paths are supplied to the temporary servers using the additional server options.

Expected result:

```text
Clusters are compatible
```

If compatibility checking fails:

```text
STOP
  ↓
Read the exact pg_upgrade error
  ↓
Correct the problem
  ↓
Run --check again
```

Do not continue until the check succeeds.

---

# 17. Production Upgrade Preparation

Before the production change:

```text
Change approved
Maintenance window approved
Application downtime approved
Backup completed
Backup verification completed
Rollback plan documented
pg_upgrade --check successful
Extension compatibility confirmed
Disk space confirmed
Tablespaces reviewed
Checksum state confirmed
```

Stop or suspend:

```text
Application services
Batch jobs
ETL jobs
Scheduled writers
Write-based reports
Database maintenance jobs
```

Check remaining sessions:

```bash
sudo -u postgres psql -p 5432 -c "
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
"
```

---

# 18. Capture Tested Baseline Information

## Tested Row Counts

```text
actor        200
customer     599
film         1000
inventory   4581
rental      16044
payment      14596
```

Capture:

```bash
sudo -u postgres psql -p 5432 -d dvdrental -c "
SELECT 'actor' AS table_name, COUNT(*) FROM actor
UNION ALL
SELECT 'customer', COUNT(*) FROM customer
UNION ALL
SELECT 'film', COUNT(*) FROM film
UNION ALL
SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL
SELECT 'rental', COUNT(*) FROM rental
UNION ALL
SELECT 'payment', COUNT(*) FROM payment;
"
```

## Tested Object Counts

```text
Tables       15
Views         7
Indexes      32
Sequences    13
```

Capture:

```bash
sudo -u postgres psql -p 5432 -d dvdrental -c "
SELECT 'Tables' AS object_type, COUNT(*) AS object_count
FROM pg_tables
WHERE schemaname='public'

UNION ALL

SELECT 'Views', COUNT(*)
FROM pg_views
WHERE schemaname='public'

UNION ALL

SELECT 'Indexes', COUNT(*)
FROM pg_indexes
WHERE schemaname='public'

UNION ALL

SELECT 'Sequences', COUNT(*)
FROM pg_sequences
WHERE schemaname='public';
"
```

These are the tested lab values to compare after upgrade.

---

# 19. Stop PostgreSQL 16

After application traffic is stopped:

```bash
sudo pg_ctlcluster 16 main stop
```

Make sure PostgreSQL 18 is also down:

```bash
sudo pg_ctlcluster 18 main stop
```

Validate:

```bash
pg_lsclusters
```

Required:

```text
16 main 5432 down
18 main 5433 down
```

---

# 20. Actual PostgreSQL 16 → 18 Upgrade – Copy Mode

Use the same command that passed `--check`, removing only `--check`:

```bash
sudo -u postgres bash -c '
cd /var/lib/postgresql/pg_upgrade_work

/usr/lib/postgresql/18/bin/pg_upgrade \
  --old-bindir=/usr/lib/postgresql/16/bin \
  --new-bindir=/usr/lib/postgresql/18/bin \
  --old-datadir=/etc/postgresql/16/main \
  --new-datadir=/etc/postgresql/18/main \
  --old-options="-D /var/lib/postgresql/16/main" \
  --new-options="-D /var/lib/postgresql/18/main" \
  --old-port=5432 \
  --new-port=5433 \
  --username=postgres
'
```

No `--link` option is specified.

Therefore this uses:

```text
COPY MODE
```

Successful completion should end with an upgrade-complete message.

Do not delete PostgreSQL 16.

---

# 21. PostgreSQL 18 `pg_upgrade` Statistics Improvement

PostgreSQL 18 `pg_upgrade` transfers **most optimizer statistics** by default.

It does not transfer everything.

Examples that may need regeneration include:

```text
Extended statistics created with CREATE STATISTICS
Extension-defined statistics
Cumulative statistics
```

PostgreSQL therefore recommends rebuilding missing and cumulative statistics after the upgrade.

---

# 22. Configure Production Port

PostgreSQL 16 used:

```text
5432
```

PostgreSQL 18 used temporary preparation port:

```text
5433
```

After PG16 is stopped, configure PG18 to use:

```text
5432
```

Edit:

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

Set:

```conf
port = 5432
```

Verify:

```bash
grep -E '^[[:space:]]*port[[:space:]]*=' \
/etc/postgresql/18/main/postgresql.conf
```

---

# 23. Start PostgreSQL 18

Start:

```bash
sudo pg_ctlcluster 18 main start
```

Validate:

```bash
pg_lsclusters
```

Expected:

```text
16 main 5432 down
18 main 5432 online
```

Check listener:

```bash
sudo ss -lntp | grep ':5432'
```

Check version:

```bash
sudo -u postgres psql -p 5432 -c "SELECT version();"
```

Check:

```bash
sudo -u postgres psql -p 5432 -c "SHOW server_version;"
```

---

# 24. Validate DVD Rental Database

Connect:

```bash
sudo -u postgres psql -p 5432 -d dvdrental
```

Check:

```sql
SELECT current_database();
SELECT version();
```

List tables:

```sql
\dt
```

List views:

```sql
\dv
```

---

# 25. Additional Post-Upgrade Validation

Re-run tested row counts:

```sql
SELECT 'actor' AS table_name, COUNT(*) FROM actor
UNION ALL
SELECT 'customer', COUNT(*) FROM customer
UNION ALL
SELECT 'film', COUNT(*) FROM film
UNION ALL
SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL
SELECT 'rental', COUNT(*) FROM rental
UNION ALL
SELECT 'payment', COUNT(*) FROM payment;
```

Expected:

```text
actor        200
customer     599
film         1000
inventory   4581
rental      16044
payment      14596
```

Re-run object counts:

```sql
SELECT 'Tables' AS object_type, COUNT(*) AS object_count
FROM pg_tables
WHERE schemaname='public'

UNION ALL

SELECT 'Views', COUNT(*)
FROM pg_views
WHERE schemaname='public'

UNION ALL

SELECT 'Indexes', COUNT(*)
FROM pg_indexes
WHERE schemaname='public'

UNION ALL

SELECT 'Sequences', COUNT(*)
FROM pg_sequences
WHERE schemaname='public';
```

Expected:

```text
Tables       15
Views         7
Indexes      32
Sequences    13
```

Check invalid indexes:

```sql
SELECT
    n.nspname,
    c.relname
FROM pg_index i
JOIN pg_class c
  ON c.oid = i.indexrelid
JOIN pg_namespace n
  ON n.oid = c.relnamespace
WHERE NOT i.indisvalid;
```

Check extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY 1;
```

Review logs:

```bash
sudo tail -100 /var/log/postgresql/postgresql-18-main.log
```

---

# 26. Statistics Maintenance

First generate statistics where missing:

```bash
sudo -u postgres \
/usr/lib/postgresql/18/bin/vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only
```

Then refresh cumulative statistics:

```bash
sudo -u postgres \
/usr/lib/postgresql/18/bin/vacuumdb \
--all \
--analyze-only
```

For large production environments, parallel jobs can be considered after reviewing CPU and I/O capacity:

```bash
sudo -u postgres \
/usr/lib/postgresql/18/bin/vacuumdb \
--all \
--analyze-only \
--jobs=4
```

---

# 27. Application Handover

Once DBA validation succeeds, ask the application team to start services and test:

```text
Login
SELECT
INSERT
UPDATE
DELETE
Functions
Batch jobs
Reports
APIs
Connection pooling
```

Monitor:

```text
PostgreSQL logs
CPU
Memory
Disk I/O
Connections
Locks
Long-running queries
Application errors
Replication lag
Backup jobs
Monitoring alerts
```

---

# 28. Rollback – Copy Mode

COPY mode keeps PostgreSQL 16 relation files independent from PostgreSQL 18.

If PG18 validation fails **before new production writes are accepted**:

Stop PG18:

```bash
sudo pg_ctlcluster 18 main stop
```

Restore PG16 endpoint/configuration if needed.

Start PG16:

```bash
sudo pg_ctlcluster 16 main start
```

Validate:

```bash
pg_lsclusters
```

Connect:

```bash
sudo -u postgres psql -p 5432 -d dvdrental
```

This is why COPY mode provides the simplest immediate rollback position.

---

# 29. Critical Rollback Point – New Production Writes

Once PostgreSQL 18 accepts new production writes:

```text
PG16 = data at original cutover
PG18 = cutover data + new production transactions
```

PostgreSQL 16 is now stale.

Do not simply stop PG18 and restart PG16.

A rollback after new PG18 writes requires a separately designed data-reconciliation/recovery strategy.

---

# 30. PostgreSQL 18 → 16 Logical Downgrade Warning

Do not assume:

```text
PG18 pg_dump
      ↓
Restore to PG16
```

is a guaranteed rollback method.

Logical dumps from newer PostgreSQL versions are not guaranteed to restore successfully into older major versions.

PostgreSQL 18 may use:

```text
New syntax
New data types
New object definitions
New features
```

that PostgreSQL 16 does not understand.

A PG18 → PG16 downgrade must therefore be separately designed and tested.

---

# 31. Clean Up PostgreSQL 16

Do not remove PostgreSQL 16 immediately.

Wait until:

```text
DBA validation complete
Application validation complete
Business validation complete
Performance acceptable
PG18 backup successful
Restore capability confirmed
Monitoring stable
Replication healthy
Rollback window closed
Change approved for closure
```

Review the cleanup instructions generated by `pg_upgrade`.

Only then remove the old cluster and, when no longer needed, the PostgreSQL 16 packages.

---

# 32. PostgreSQL 16 → 18 Link Mode Upgrade

Link mode uses:

```text
--link
```

or:

```text
-k
```

Example uses the same Ubuntu split configuration/data-directory layout:

```bash
sudo -u postgres bash -c '
cd /var/lib/postgresql/pg_upgrade_work

/usr/lib/postgresql/18/bin/pg_upgrade \
  --old-bindir=/usr/lib/postgresql/16/bin \
  --new-bindir=/usr/lib/postgresql/18/bin \
  --old-datadir=/etc/postgresql/16/main \
  --new-datadir=/etc/postgresql/18/main \
  --old-options="-D /var/lib/postgresql/16/main" \
  --new-options="-D /var/lib/postgresql/18/main" \
  --old-port=5432 \
  --new-port=5433 \
  --username=postgres \
  --link
'
```

Advantages:

```text
Very fast
Very low additional data-file storage
```

Main disadvantage:

```text
Rollback becomes highly restricted
```

---

# 33. Linux Inode Basics

An inode is **not the physical file**.

An inode is filesystem metadata describing a filesystem object.

It contains information such as:

```text
File type
Owner
Permissions
Size
Timestamps
Link count
Pointers to data blocks
```

Display inode numbers:

```bash
ls -li
```

---

# 34. Why Link Mode Uses Hard Links

A hard link creates another directory entry pointing to the same inode/data blocks.

Therefore, with `pg_upgrade --link`, old and new relation filenames can refer to the same underlying data.

That is why link mode:

```text
Uses very little additional storage
Runs very quickly
```

It is also why rollback becomes dangerous after PostgreSQL 18 starts modifying those shared files.

---

# 35. Link Mode Filesystem Requirement

Hard links cannot cross filesystems.

Therefore old and new relation files that must be linked need to be on compatible locations within the same filesystem.

User tablespaces require the same consideration.

---

# 36. Link Mode Upgrade Preparation

Before link mode:

```text
Backup verified
Rollback strategy approved
PG18 binaries installed
PG18 cluster initialized
Checksum state matched
Extensions installed
pg_upgrade --check passed
Application downtime approved
Filesystem requirement validated
```

Run compatibility check first using the same `--check` command shown earlier.

---

# 37. Stop Application and PostgreSQL 16

Stop application traffic.

Then:

```bash
sudo pg_ctlcluster 16 main stop
sudo pg_ctlcluster 18 main stop
```

Validate:

```bash
pg_lsclusters
```

Both clusters must be down.

---

# 38. Perform Link Mode Upgrade

Run the command shown in Section 32 with:

```text
--link
```

Pay close attention to the warning generated by `pg_upgrade`.

---

# 39. Configure PostgreSQL 18

Set the production port:

```conf
port = 5432
```

in:

```text
/etc/postgresql/18/main/postgresql.conf
```

Complete configuration review before starting PostgreSQL 18.

---

# 40. Start PostgreSQL 18

Start:

```bash
sudo pg_ctlcluster 18 main start
```

Run statistics maintenance and validation as described in the COPY-mode procedure.

---

# 41. Critical Link-Mode Rollback Rule

The earlier idea of restoring only:

```text
pg_control.old
```

after PG18 has already started is **not safe**.

With link mode, PostgreSQL 16 and PostgreSQL 18 can share user relation files.

Once PG18 starts and modifies those files:

```text
DO NOT START THE ORIGINAL LINKED PG16 CLUSTER
```

Restoring `pg_control.old` does not make the shared relation files PostgreSQL-16-compatible again.

---

# 42. Link Mode Rollback Window

Before PostgreSQL 18 has ever been started, follow the exact rollback instructions generated by `pg_upgrade`.

After PostgreSQL 18 has started:

```text
Do not restart the original linked PostgreSQL 16 cluster.
```

If strong rollback requirements exist, prefer:

```text
COPY mode
Filesystem/storage snapshot
Clone mode where supported
Separate physical backup
Logical replication migration
```

depending on architecture.

---

# 43. PostgreSQL 18 Clone Mode

PostgreSQL also supports:

```text
--clone
```

where supported by the filesystem.

Clone mode uses copy-on-write filesystem cloning.

Potential benefits:

```text
Fast
Low initial additional disk usage
Old cluster remains independent
```

Filesystem support must be validated before choosing this mode.

---

# 44. Copy vs Link vs Clone

| Mode | Data Transfer | Extra Space | Speed | Old Cluster After New Starts |
| --- | --- | ---: | --- | --- |
| Copy | Full copy | Higher | Slower | Independent |
| Link | Hard links | Very low | Very fast | Unsafe to restart |
| Clone | Copy-on-write | Low initially | Very fast | Independent |
| `copy_file_range` | Filesystem dependent | Depends | Fast | Independent |

For this tested lab:

```text
COPY MODE
```

is the preferred teaching and rollback-friendly method.

---

# 45. Minor Version Upgrade / Patching

Minor releases contain fixes for:

```text
Security vulnerabilities
Data corruption risks
Crashes
Replication problems
Query correctness
Performance bugs
Low-risk improvements
```

Example:

```text
18.4 → 18.6
```

A minor update normally does **not** require:

```text
pg_upgrade
new data directory
pg_dump
pg_restore
```

The existing PostgreSQL 18 cluster remains in place.

---

# 46. Current Minor Version Example

For this class/lab date:

```text
PostgreSQL 18.6
PostgreSQL 16.15
```

are the versions being used.

Example minor patch path:

```text
18.4 → 18.6
```

---

# 47. Ubuntu Package Validation

Check installed PostgreSQL 18 packages:

```bash
dpkg -l | grep '^ii' | grep 'postgresql-18'
```

Check candidate version:

```bash
apt-cache policy postgresql-18
```

Refresh package metadata:

```bash
sudo apt update
```

Before production patching, confirm the exact package version that APT plans to install.

---

# 48. PostgreSQL 18 Minor Upgrade Preparation

Before a minor update:

```text
Read release notes
Review every skipped minor release
Take verified backup
Confirm package repository
Obtain change approval
Stop application traffic
Confirm extension compatibility
Prepare rollback plan
```

Check current version:

```bash
sudo -u postgres psql -p 5432 -c "SELECT version();"
```

---

# 49. PostgreSQL 18.x → 18.6 Minor Upgrade

Stop application traffic.

For a controlled maintenance window, stop the PostgreSQL 18 cluster:

```bash
sudo pg_ctlcluster 18 main stop
```

Update packages:

```bash
sudo apt update

sudo apt install --only-upgrade \
postgresql-18 \
postgresql-client-18 \
postgresql-contrib-18
```

Do not assume the package manager will always behave identically across environments; verify package policy before production changes.

---

# 50. Start PostgreSQL After Minor Upgrade

Start:

```bash
sudo pg_ctlcluster 18 main start
```

Validate:

```bash
pg_lsclusters
```

Check version:

```bash
sudo -u postgres psql -p 5432 -c "SELECT version();"
```

Check data directory:

```bash
sudo -u postgres psql -p 5432 -Atc "SHOW data_directory;"
```

The same PostgreSQL 18 data directory remains in use.

---

# 51. Minor Upgrade Production Procedure

Recommended sequence:

```text
1. Read all applicable release notes.
2. Test in non-production.
3. Confirm exact package versions.
4. Take and verify backup.
5. Obtain change approval.
6. Stop application traffic.
7. Stop PostgreSQL if using a controlled outage.
8. Update PostgreSQL packages.
9. Start PostgreSQL.
10. Confirm server version.
11. Review logs.
12. Validate application connectivity.
13. Execute release-specific follow-up actions.
14. Start application.
15. Monitor.
```

---

# 52. Important Minor-Release Release-Notes Rule

Do not assume every minor update is only:

```text
replace binaries → restart → finished
```

Some release notes may recommend:

```text
REINDEX
Configuration changes
Security changes
Data cleanup
Extension maintenance
```

Always read every skipped minor-release note.

---

# 53. Minor-Version Rollback

PostgreSQL minor releases within the same major version maintain physical storage compatibility.

However, do **not** blindly downgrade packages.

A newer minor release may have involved:

```text
REINDEX
Extension updates
Configuration changes
Security changes
Manual maintenance
```

Rollback planning should include:

```text
Exact previous packages
Repository availability
Backup or snapshot
Extension compatibility
Release-note review
Application validation
```

---

# 54. Example Controlled Package Rollback

Only use a package rollback when:

```text
The exact previous packages are available
Release notes have been reviewed
Extension compatibility is confirmed
No incompatible follow-up action has been performed
Rollback has been tested
```

Stop PostgreSQL:

```bash
sudo pg_ctlcluster 18 main stop
```

Install the required previous package versions using the approved APT repository and explicit version pinning.

Then start:

```bash
sudo pg_ctlcluster 18 main start
```

Validate:

```bash
sudo -u postgres psql -p 5432 -c "SELECT version();"
```

Do not treat package downgrade as a universal rollback method.

---

# 55. Major Upgrade vs Minor Upgrade

| Area | Major Upgrade | Minor Upgrade |
| --- | --- | --- |
| Example | 16 → 18 | 18.4 → 18.6 |
| New data directory | Yes | No |
| `pg_upgrade` needed | Possible | No |
| Dump/restore possible | Yes | Normally unnecessary |
| Logical replication possible | Yes | Normally unnecessary |
| Physical storage compatibility | Not guaranteed across majors | Maintained within major |
| Application testing | Mandatory | Recommended |
| Release-note review | Mandatory | Mandatory |
| Downtime | Depends on method | Usually short |

---

# 56. Major Upgrade Options

PostgreSQL major upgrades can use:

```text
pg_dump / pg_restore
pg_dumpall
pg_upgrade
Logical replication
Managed / third-party migration tooling
```

For PostgreSQL 16 → 18, PostgreSQL 17 is **not required as an intermediate upgrade**.

A direct:

```text
16 → 18
```

upgrade is supported when the chosen method and compatibility checks allow it.

Still review PostgreSQL 17 and PostgreSQL 18 release notes for compatibility-impacting changes.

---

# 57. `pg_upgrade` Production Checklist

Before `pg_upgrade`, verify:

```text
Source PostgreSQL version
Target PostgreSQL version
Operating system compatibility
Old/new binary directories
Old/new configuration directories
Old/new physical data directories
Tablespaces
Disk space
Filesystem
Data checksum state
Locale
Encoding
Extensions
Shared libraries
Authentication
SSL
Archive configuration
Replication
Backup
Restore capability
Application downtime
Monitoring
Rollback plan
```

---

# 58. Additional Useful Commands

Check version:

```sql
SELECT version();
```

Server version:

```sql
SHOW server_version;
```

Data directory:

```sql
SHOW data_directory;
```

Checksums:

```sql
SHOW data_checksums;
```

WAL level:

```sql
SHOW wal_level;
```

Archive mode:

```sql
SHOW archive_mode;
```

Connections:

```sql
SHOW max_connections;
```

Database sizes:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Replication slots:

```sql
SELECT * FROM pg_replication_slots;
```

Replication status:

```sql
SELECT * FROM pg_stat_replication;
```

Prepared transactions:

```sql
SELECT * FROM pg_prepared_xacts;
```

Extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension;
```

---

# 59. Backup Validation Before Upgrade

Do not stop at:

```text
Backup completed successfully
```

Also verify recovery capability.

For a logical dump:

```bash
sudo -u postgres \
/usr/lib/postgresql/16/bin/pg_restore \
-l /DBBackups2026/PG16_PreUpgrade/dvdrental_pg16.dump \
| head
```

For a backup file checksum:

```bash
sha256sum -c dvdrental_pg16.dump.sha256
```

For physical backup tools, use the validation method appropriate to that tool and perform periodic restore testing.

---

# 60. Do Not Forget Replication

If PostgreSQL 16 is part of HA/replication, this standalone procedure is not enough.

The upgrade plan must cover:

```text
Primary
Standbys
Replication slots
Archive/WAL
Replication users
Failover tooling
Patroni/repmgr
Load balancer or VIP
Connection endpoints
Backup
Monitoring
```

Use a dedicated HA/replication upgrade runbook.

---

# 61. Do Not Delete the Old Cluster Immediately

PostgreSQL 18 starting successfully does **not** mean PostgreSQL 16 should be deleted immediately.

Wait until:

```text
Application testing complete
Smoke testing complete
Performance acceptable
Backups working
Restore capability confirmed
Monitoring working
Replication healthy
Business acceptance complete
Rollback period closed
```

Only then perform old-cluster cleanup.

---

# 62. Major Corrections to Original Class Notes

| Earlier / Original Item | Corrected Understanding |
| --- | --- |
| `16.9.10 = major.minor.patch` | Invalid for modern PostgreSQL. `16.15` = major 16, minor 15 |
| PostgreSQL 11 onward uses two-part numbering | Two-part numbering began with PostgreSQL 10 |
| `pg_upgrade` came from 8.4 | Official `pg_upgrade` appeared with PostgreSQL 9.0 |
| RHEL `/usr/pgsql-*` paths | Ubuntu uses `/usr/lib/postgresql/*/bin` |
| RHEL `/var/lib/pgsql/*` data paths | Ubuntu lab uses `/var/lib/postgresql/*/main` |
| `yum` / `dnf` commands | Ubuntu lab uses `apt` |
| PG16/PG18 config inside PGDATA | Ubuntu stores cluster config under `/etc/postgresql/<version>/main` |
| PG18 checksums must always be disabled | Only match the source cluster checksum state |
| Blindly copy old `postgresql.conf` | Review and merge only valid settings |
| `pg_upgrade` transfers no statistics | PostgreSQL 18 transfers most optimizer statistics |
| Inode is physical file | Inode is filesystem metadata |
| LINK-mode PG16 can restart after PG18 runs | Unsafe |
| Restore `pg_control.old` after PG18 starts | Does not make linked files safe for PG16 |
| PG18 dump can always restore to PG16 | Not guaranteed |
| Minor package downgrade is always safe | Requires release-note, extension and follow-up-action review |
| `pg_stat_user_tables` gives cluster inventory | It only describes the connected database |
| Fixed 100 MB disk rule | No universal safe fixed number |
| Need PG17 between PG16 and PG18 | Direct 16 → 18 is supported when compatibility checks succeed |

---

# 63. Production Recommendation – PostgreSQL 16 → 18

For a normal production environment where enough storage is available:

```text
PostgreSQL 16
       |
       | Prechecks
       | Backup
       | pg_upgrade --check
       v
Maintenance Window
       |
       | Stop Application
       | Stop PG16 and PG18
       v
pg_upgrade COPY MODE
       |
       v
PostgreSQL 18
       |
       | Configuration Review
       | Port Cutover
       | Start PG18
       | Statistics Maintenance
       | Validation
       v
Production
```

Advantages:

```text
Faster than full dump/restore
Old cluster remains independent
Immediate rollback is simpler before new PG18 writes
```

For very large databases where storage or downtime is critical, evaluate:

```text
pg_upgrade --link
pg_upgrade --clone
pg_upgrade --copy-file-range
pg_upgrade --swap
Logical replication
```

with a tested rollback design.

---

# 64. Quick Reference Cheat Sheet

## Major Upgrade

```text
16 → 18
```

Methods:

```text
pg_upgrade
pg_dump / pg_restore
pg_dumpall
Logical replication
```

## Tested COPY-Mode Flow

```text
PRECHECK
   ↓
PREPARE PG18
   ↓
BACKUP + BASELINE
   ↓
pg_upgrade --check
   ↓
STOP APPLICATION
   ↓
STOP PG16 + PG18
   ↓
pg_upgrade COPY
   ↓
CONFIGURE PG18
   ↓
START PG18
   ↓
VALIDATE
```

## `pg_upgrade --check`

```bash
sudo -u postgres bash -c '
cd /var/lib/postgresql/pg_upgrade_work

/usr/lib/postgresql/18/bin/pg_upgrade \
  --old-bindir=/usr/lib/postgresql/16/bin \
  --new-bindir=/usr/lib/postgresql/18/bin \
  --old-datadir=/etc/postgresql/16/main \
  --new-datadir=/etc/postgresql/18/main \
  --old-options="-D /var/lib/postgresql/16/main" \
  --new-options="-D /var/lib/postgresql/18/main" \
  --old-port=5432 \
  --new-port=5433 \
  --username=postgres \
  --check
'
```

## COPY Mode

Use the same command without:

```text
--check
```

and without:

```text
--link
```

## Link Mode

Add:

```text
--link
```

Remember:

```text
Once PG18 starts in link mode,
do not restart the original linked PG16 cluster.
```

## PostgreSQL 18 Post-Upgrade Statistics

```bash
sudo -u postgres \
/usr/lib/postgresql/18/bin/vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only
```

```bash
sudo -u postgres \
/usr/lib/postgresql/18/bin/vacuumdb \
--all \
--analyze-only
```

## Minor Upgrade

```text
18.4 → 18.6
```

Typical process:

```text
Read release notes
       ↓
Backup
       ↓
Stop application
       ↓
Update packages
       ↓
Start PostgreSQL
       ↓
Validate
```

---

# 65. Final DBA Takeaways

### Major Version

```text
16 → 18
```

requires a supported migration mechanism.

### Minor Version

```text
18.4 → 18.6
```

normally keeps the existing data directory.

### `pg_upgrade`

Always use the **new PostgreSQL version's `pg_upgrade` binary**.

For this lab:

```text
/usr/lib/postgresql/18/bin/pg_upgrade
```

### Ubuntu Split Layout

For this lab:

```text
Configuration
/etc/postgresql/<version>/main

Physical Data
/var/lib/postgresql/<version>/main
```

This distinction matters when building the `pg_upgrade` command.

### COPY Mode

Uses more storage but keeps the old cluster independent.

### LINK Mode

Very fast and storage-efficient, but the original PG16 linked cluster becomes unsafe to restart after PG18 starts using the shared relation files.

### PostgreSQL 18 Checksums

New PostgreSQL 18 clusters enable checksums by default.

For `pg_upgrade`, match the source checksum state.

### PostgreSQL 18 Statistics

PostgreSQL 18 `pg_upgrade` transfers most optimizer statistics, but missing and cumulative statistics still need post-upgrade maintenance.

### Configuration

Never blindly replace PostgreSQL 18 configuration with PostgreSQL 16 configuration.

### Rollback

The safe immediate rollback window for COPY mode is before PostgreSQL 18 accepts meaningful new production writes.

A production upgrade is complete only after:

```text
Database Validation
        +
Application Validation
        +
Performance Validation
        +
Backup Validation
        +
Monitoring Validation
        +
Business Acceptance
```

---

# 66. Official PostgreSQL References

**PostgreSQL Versioning Policy**  
https://www.postgresql.org/support/versioning/

**PostgreSQL 18 – Upgrading a PostgreSQL Cluster**  
https://www.postgresql.org/docs/18/upgrading.html

**PostgreSQL 18 – pg_upgrade**  
https://www.postgresql.org/docs/18/pgupgrade.html

**PostgreSQL 18 – initdb**  
https://www.postgresql.org/docs/18/app-initdb.html

**PostgreSQL 18 – Data Checksums**  
https://www.postgresql.org/docs/18/checksums.html

**PostgreSQL 18 – pg_checksums**  
https://www.postgresql.org/docs/18/app-pgchecksums.html

**PostgreSQL 18 – pg_dump**  
https://www.postgresql.org/docs/18/app-pgdump.html

**PostgreSQL 18 – pg_dumpall**  
https://www.postgresql.org/docs/18/app-pg-dumpall.html

**PostgreSQL 18 Release Notes**  
https://www.postgresql.org/docs/18/release-18.html

**PostgreSQL 18.6 Release Notes**  
https://www.postgresql.org/docs/18/release-18-6.html

---

# 67. Summary

```text
MAJOR UPGRADE
16 → 18

Requires:
pg_upgrade / dump-restore / logical replication


TESTED LAB
Ubuntu 26.04
PostgreSQL 16.15 → 18.6
Database: dvdrental
Method: pg_upgrade
Mode: COPY


MINOR UPGRADE
18.4 → 18.6

Requires:
Release-note review
Backup
Package update
Restart / validation


COPY MODE
More storage
Old cluster remains independent
Simpler immediate rollback


LINK MODE
Very little extra storage
Very fast
Original linked PG16 unsafe after PG18 starts


POSTGRESQL 18
Checksums enabled by default for newly initialized clusters
pg_upgrade transfers most optimizer statistics
```
