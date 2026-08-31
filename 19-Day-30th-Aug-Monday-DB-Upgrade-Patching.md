# PostgreSQL 18 – Database Upgrade & Patching

**Class Date:** 31st August 2026 – Monday
**Topic:** PostgreSQL Database Upgrade, Minor Version Upgrade & Patching

---

# Index

1. Database Upgrade and Patching – Basics
2. PostgreSQL Version Numbering
3. Major vs Minor Version Upgrade
4. Why Database Upgrades Are Required
5. PostgreSQL Major Upgrade Methods
6. `pg_upgrade` Utility
7. PostgreSQL 16 to 18 – Upgrade Using Copy Mode
8. Major Upgrade Pre-Checks
9. PostgreSQL 18 Installation and Initialization
10. Data Checksum Compatibility
11. `pg_upgrade --check` Compatibility Validation
12. Production Upgrade Preparation
13. Actual PostgreSQL 16 to 18 Upgrade
14. Post-Upgrade Configuration
15. Post-Upgrade Validation
16. Statistics After PostgreSQL 18 Upgrade
17. Rollback – Copy Mode
18. Old PostgreSQL Cluster Cleanup
19. PostgreSQL 16 to 18 – Link Mode Upgrade
20. Linux Inode Basics
21. Important Link Mode Restrictions
22. Link Mode Rollback Rules
23. Minor Version Upgrade / Patching
24. PostgreSQL 18.2 to 18.4 Example
25. Minor Version Rollback
26. Production Best Practices
27. Major Corrections to Original Class Notes
28. Quick Reference Cheat Sheet
29. Official PostgreSQL References

---

# 1. Database Upgrade and Patching – Basics

## Upgrade

An **upgrade** means moving PostgreSQL from one version to another version.

Typical reasons include:

* Getting new PostgreSQL features.
* Moving from an older supported version to a newer version.
* Maintaining application compatibility.
* Improving performance or manageability.
* Moving away from an End-of-Life PostgreSQL version.

Example:

```text
PostgreSQL 16  → PostgreSQL 18
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

PostgreSQL does not normally distribute individual database patch files in the way some other database products do.

PostgreSQL maintenance fixes are delivered through **minor releases**.

Example:

```text
PostgreSQL 18.2 → PostgreSQL 18.6
```

This is a **minor version update**.

---

# 2. PostgreSQL Version Numbering

This is an important correction.

The example:

```text
16.9.10
```

is **not the current PostgreSQL version-numbering model**.

Starting with PostgreSQL 10, PostgreSQL uses:

```text
Major.Minor
```

Examples:

```text
16.14
16.15
17.11
18.4
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
16 → 17
```

or:

```text
16 → 18
```

is a **major upgrade**.

Whereas:

```text
18.2 → 18.4
```

or:

```text
18.4 → 18.6
```

is a **minor upgrade**.

PostgreSQL does not have a separate third "patch" component for versions 10 and later.

Before PostgreSQL 10, the version numbering worked differently.

For example:

```text
9.5.3
```

means:

```text
9.5 = Major Version
3   = Minor Release
```

Starting with PostgreSQL 10:

```text
10.1
11.2
16.15
18.6
```

use the two-part model.

---

# 3. Major vs Minor Version Upgrade

| Upgrade Type     | Example                         | Data Directory Reusable Directly? | Typical Method                                  |
| ---------------- | ------------------------------- | --------------------------------: | ----------------------------------------------- |
| Major Upgrade    | 16 → 18                         |                                No | `pg_upgrade`, dump/restore, logical replication |
| Minor Upgrade    | 18.2 → 18.6                     |                               Yes | Replace binaries/packages and restart           |
| Security/Bug Fix | Delivered through minor release |                               Yes | Minor version update                            |

A PostgreSQL major release can change:

* System catalogs.
* Internal metadata.
* SQL behavior.
* Configuration parameters.
* Extension interfaces.
* Internal structures.

Therefore PostgreSQL 16 binaries cannot simply start a PostgreSQL 18 data directory, and PostgreSQL 18 binaries cannot simply start a PostgreSQL 16 data directory.

A major migration method is required.

For minor releases, PostgreSQL guarantees storage compatibility within the same major release.

---

# 4. Why Database Upgrades Are Required

Database upgrades may be required for the following reasons.

### New Features

A major PostgreSQL version may introduce:

* New SQL functionality.
* Performance improvements.
* New monitoring capabilities.
* New replication features.
* New security features.
* Better query optimization.

PostgreSQL 18, for example, introduced major improvements such as asynchronous I/O, UUIDv7 support, skip scans, OAuth authentication support, improved `pg_upgrade`, and other enhancements.

### Security Fixes

Minor releases frequently contain security fixes.

Running an old minor version unnecessarily exposes the database to already-fixed issues.

### Bug Fixes

Minor releases contain fixes for:

* Crashes.
* Incorrect query results.
* Replication problems.
* WAL issues.
* Data corruption risks.
* Planner problems.
* Extension problems.

### End of Life

PostgreSQL normally supports each major release for approximately five years.

Once a major release reaches End of Life:

* No new bug fixes are released.
* No normal security updates are released.
* Production use becomes increasingly risky.

### Application Compatibility

A new application version may require:

```text
PostgreSQL >= 16
```

or:

```text
PostgreSQL >= 18
```

The database therefore needs to be upgraded before the application upgrade.

### Infrastructure / Operating System Compatibility

Operating system upgrades may also require newer PostgreSQL packages.

### Organization or Management Requirements

Organizations may maintain policies such as:

```text
Only PostgreSQL N and N-1 major versions are allowed.
```

Upgrades may therefore be required for compliance.

---

# 5. PostgreSQL Major Upgrade Methods

PostgreSQL supports several major-version migration approaches.

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
* Clean database rebuild.
* Can migrate between different hardware architectures.
* Good for small and medium databases.

Disadvantages:

* Can take significant time.
* Requires storage for dump files.
* Downtime can be high for large databases.

---

# 6. `pg_upgrade` Utility

An important correction to the original notes:

`pg_upgrade` was officially introduced with **PostgreSQL 9.0**, not PostgreSQL 8.4.

The PostgreSQL 9.0 release included `pg_upgrade`, formerly known as `pg_migrator`.

Modern PostgreSQL 18 `pg_upgrade` supports upgrading clusters from PostgreSQL 9.2 and later.

Example:

```text
PostgreSQL 16
      |
      | pg_upgrade
      v
PostgreSQL 18
```

`pg_upgrade` avoids logically exporting and importing every row.

This makes it considerably faster for large databases.

---

# 7. PostgreSQL 16 to 18 – Upgrade Using Copy Mode

Assume PostgreSQL 16 is installed.

Example packages:

```bash
yum install postgresql16-server postgresql16-contrib -y
```

Initialize the PostgreSQL 16 cluster:

```bash
/usr/pgsql-16/bin/postgresql-16-setup initdb
```

Start PostgreSQL:

```bash
systemctl start postgresql-16
```

Validate:

```bash
systemctl status postgresql-16
```

Check processes:

```bash
ps -ef | grep '[p]ostgres'
```

Example PostgreSQL 16 process:

```text
/usr/pgsql-16/bin/postgres -D /var/lib/pgsql/16/data/
```

---

# 8. Load Sample Database

Download the DVD Rental sample database:

```bash
wget https://neon.com/postgresqltutorial/dvdrental.zip
```

Extract:

```bash
unzip dvdrental.zip
```

Move:

```bash
mv dvdrental.tar /tmp/
```

Switch user:

```bash
su - postgres
```

Connect:

```bash
psql
```

Create database:

```sql
CREATE DATABASE dvdrental;
```

Exit:

```sql
\q
```

Restore:

```bash
pg_restore -Ft -d dvdrental /tmp/dvdrental.tar -v 2>restore.log
```

Validate:

```bash
psql -d dvdrental
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

# 9. Major Upgrade Pre-Checks

Before upgrading PostgreSQL 16 to PostgreSQL 18, collect the environment details.

## PostgreSQL 16

```text
Binary Directory:
 /usr/pgsql-16/bin

Data Directory:
 /var/lib/pgsql/16/data

Port:
 5432
```

Commands:

```bash
/usr/pgsql-16/bin/postgres --version
```

```bash
psql -p 5432 -c "SHOW data_directory;"
```

```bash
psql -p 5432 -c "SHOW port;"
```

---

## Capacity Check

For normal `pg_upgrade` **copy mode**, the PostgreSQL 18 cluster needs enough storage for approximately another copy of the user relation data plus system files, WAL, temporary upgrade files and operational headroom.

Do not use a fixed rule such as:

```text
100 MB free space is sufficient for PostgreSQL 18 software.
```

Package size and dependencies differ across distributions and releases.

Validate actual filesystem capacity:

```bash
df -h
```

Check inode capacity:

```bash
df -i
```

Check PostgreSQL directory size:

```bash
du -sh /var/lib/pgsql/16/data
```

Check tablespaces:

```sql
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

---

# 10. CPU and Memory

Normally the old and new database clusters do **not** need to serve production traffic simultaneously during the actual `pg_upgrade`.

However, sufficient CPU, memory and storage should still be available for:

* Upgrade checks.
* Upgrade operations.
* Post-upgrade statistics collection.
* Application validation.
* Temporary cluster startup by `pg_upgrade`.

Check:

```bash
free -h
```

```bash
lscpu
```

```bash
nproc
```

---

# 11. Install PostgreSQL 18 Software

Install the PostgreSQL 18 packages:

```bash
yum install postgresql18-server postgresql18-contrib -y
```

Validate:

```bash
rpm -qa | grep postgresql18
```

Check version:

```bash
/usr/pgsql-18/bin/postgres --version
```

---

# 12. Data Checksum Compatibility

PostgreSQL 18 changed an important default.

New PostgreSQL 18 clusters created by `initdb` have data checksums enabled by default.

Check PostgreSQL 16:

```sql
SHOW data_checksums;
```

Example:

```text
off
```

For `pg_upgrade`, the old and new clusters need compatible checksum settings.

If PostgreSQL 16 has:

```text
data_checksums = off
```

initialize PostgreSQL 18 with:

```bash
/usr/pgsql-18/bin/initdb \
-D /var/lib/pgsql/18/data \
--no-data-checksums
```

If PostgreSQL 16 already has checksums enabled, do **not** disable them unnecessarily on PostgreSQL 18.

PostgreSQL also provides:

```bash
pg_checksums
```

to enable, disable or verify checksums while the cluster is offline.

---

# 13. Configure Temporary PostgreSQL 18 Port

PostgreSQL 16 is using:

```text
5432
```

For upgrade preparation, PostgreSQL 18 can be configured with:

```text
5433
```

Edit:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

Set:

```conf
port = 5433
```

During the actual `pg_upgrade`, both production clusters must be stopped; `pg_upgrade` temporarily starts them itself as required.

---

# 14. Configuration Migration

Do **not blindly copy** PostgreSQL 16's complete `postgresql.conf` over PostgreSQL 18.

Some parameters may have:

* Changed.
* Been removed.
* Changed defaults.
* Changed behavior.

Instead compare the files:

```bash
diff -u \
/var/lib/pgsql/16/data/postgresql.conf \
/var/lib/pgsql/18/data/postgresql.conf
```

Review and migrate required settings individually.

Important files include:

```text
postgresql.conf
postgresql.auto.conf
pg_hba.conf
pg_ident.conf
```

Also review any configuration files referenced through:

```conf
include
include_if_exists
include_dir
```

For authentication rules, carefully migrate:

```text
pg_hba.conf
```

rather than assuming the old file is automatically appropriate.

---

# 15. Extension Pre-Check

Check installed extensions in every database.

Example:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY 1;
```

New PostgreSQL 18 versions of required extension binaries must exist before `pg_upgrade`.

Examples might include:

```text
pgcrypto
postgis
pg_stat_statements
uuid-ossp
pgaudit
```

Do not simply assume PostgreSQL 16 extension `.so` files are compatible with PostgreSQL 18.

---

# 16. `pg_upgrade --check`

Before performing the actual upgrade, validate compatibility.

Stop PostgreSQL 18 if it was started:

```bash
systemctl stop postgresql-18
```

PostgreSQL 16 may remain running for the check.

Switch user:

```bash
su - postgres
```

Run:

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433 \
-c
```

Here:

```text
-b = old PostgreSQL binary directory
-B = new PostgreSQL binary directory
-d = old cluster data/config directory
-D = new cluster data/config directory
-p = old cluster temporary port
-P = new cluster temporary port
-c = check only
```

Expected result:

```text
Clusters are compatible
```

If compatibility checking fails, **do not continue with production upgrade**.

Correct the reported issues first.

---

# 17. Production Upgrade Preparation

Before actual production upgrade:

### Change Approval

Ensure approved:

```text
Change ticket
Maintenance window
Application downtime
Rollback plan
Business approval
```

### Backup

Take a verified physical backup using the organization's standard backup solution.

Possible tools include:

```text
pgBackRest
Barman
pg_basebackup
Storage snapshot
Enterprise backup solution
```

Example:

```bash
pg_basebackup \
-h localhost \
-p 5432 \
-U replication_user \
-D /backup/pg16_preupgrade \
-F p \
-X stream \
-P
```

A backup that has never been tested for restoration should not be treated as a complete recovery strategy.

### VM Snapshot

A VM/storage snapshot can provide another protection layer, but it should follow the virtualization/storage vendor's consistency requirements.

A random live VM snapshot should not be treated as a substitute for a PostgreSQL backup.

### Application Downtime

Ask the application team to:

```text
Stop application services.
Stop batch jobs.
Stop ETL jobs.
Stop reporting jobs that write data.
Stop scheduled database jobs.
Confirm no application connections remain.
```

Check sessions:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
```

---

# 18. Capture Baseline Information

Before stopping PostgreSQL 16, capture the database inventory.

Databases:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
WHERE datallowconn
ORDER BY 1;
```

Roles:

```sql
SELECT rolname
FROM pg_roles
ORDER BY 1;
```

Tablespaces:

```sql
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

Extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY 1;
```

Tables:

```sql
SELECT COUNT(*)
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog','information_schema');
```

Views:

```sql
SELECT COUNT(*)
FROM pg_views
WHERE schemaname NOT IN ('pg_catalog','information_schema');
```

Indexes:

```sql
SELECT COUNT(*)
FROM pg_indexes
WHERE schemaname NOT IN ('pg_catalog','information_schema');
```

Sequences:

```sql
SELECT COUNT(*)
FROM pg_sequences
WHERE schemaname NOT IN ('pg_catalog','information_schema');
```

Note that queries against:

```text
pg_stat_user_tables
pg_stat_user_indexes
```

apply to the currently connected database and should not be treated as a complete cluster-wide object inventory.

---

# 19. Stop PostgreSQL 16

After application traffic is stopped:

```bash
systemctl stop postgresql-16
```

Validate:

```bash
systemctl status postgresql-16
```

Check processes:

```bash
ps -ef | grep '[p]ostgres'
```

Make sure the PostgreSQL 16 server is completely stopped.

---

# 20. Actual PostgreSQL 16 → 18 Upgrade – Copy Mode

Run as the PostgreSQL operating system user:

```bash
su - postgres
```

Execute:

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433
```

This uses the default:

```text
copy mode
```

`pg_upgrade` performs operations such as:

```text
Checking cluster versions
Checking connection settings
Checking prepared transactions
Checking required libraries
Creating global-object dump
Creating database schema dumps
Creating new system catalogs
Migrating transaction information
Restoring schemas
Copying user relation files
Checking extensions
Synchronizing data
```

Successful completion ends with:

```text
Upgrade Complete
```

---

# 21. PostgreSQL 18 `pg_upgrade` Improvement

Older PostgreSQL versions did not carry normal planner statistics through `pg_upgrade`.

PostgreSQL 18 changed this behavior.

PostgreSQL 18 `pg_upgrade` now transfers **most optimizer statistics** by default.

However, not every type of statistic is transferred.

For example, some information such as:

```text
CREATE STATISTICS statistics
extension-defined statistics
cumulative statistics
```

may still require regeneration.

Therefore PostgreSQL 18 may recommend:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only
```

followed by:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-only
```

This is an important correction to the statement:

```text
pg_upgrade does not transfer statistics.
```

For a PostgreSQL 16 → PostgreSQL 18 upgrade, **most optimizer statistics can now be transferred**.

---

# 22. Configure Production Port

After successful upgrade, configure PostgreSQL 18 to use the original application port.

Edit:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

Set:

```conf
port = 5432
```

Also validate:

```conf
listen_addresses
max_connections
shared_buffers
work_mem
maintenance_work_mem
wal_level
max_wal_senders
max_replication_slots
archive_mode
archive_command
logging_collector
ssl
```

Do not automatically copy old values without reviewing them against PostgreSQL 18.

---

# 23. Start PostgreSQL 18

Start:

```bash
systemctl start postgresql-18
```

Check:

```bash
systemctl status postgresql-18
```

Check processes:

```bash
ps -ef | grep '[p]ostgres'
```

Connect explicitly:

```bash
/usr/pgsql-18/bin/psql -p 5432
```

Check version:

```sql
SELECT version();
```

Check server version:

```sql
SHOW server_version;
```

Check data directory:

```sql
SHOW data_directory;
```

Check port:

```sql
SHOW port;
```

Check checksum status:

```sql
SHOW data_checksums;
```

---

# 24. Validate DVD Rental Database

Connect:

```sql
\c dvdrental
```

List tables:

```sql
\dt
```

Expected tables include:

```text
actor
address
category
city
country
customer
film
film_actor
film_category
inventory
language
payment
rental
staff
store
```

List views:

```sql
\dv
```

Examples:

```text
actor_info
customer_list
film_list
nicer_but_slower_film_list
sales_by_film_category
sales_by_store
staff_list
```

---

# 25. Additional Post-Upgrade Validation

Validate database sizes:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
WHERE datallowconn;
```

Validate invalid indexes:

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

Validate extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY 1;
```

Look for available extension updates:

```sql
SELECT
    name,
    installed_version,
    default_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL
ORDER BY 1;
```

Check logs carefully.

Application validation should include:

```text
Login
SELECT
INSERT
UPDATE
DELETE
Stored procedures/functions
Batch jobs
Reports
API transactions
Performance
Connection pooling
Replication
Backup
Monitoring
```

---

# 26. Statistics Maintenance

Run the commands recommended by PostgreSQL 18 `pg_upgrade`.

First:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only
```

Then:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-only
```

For large environments parallel workers may be considered:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-only \
--jobs=4
```

Choose concurrency according to CPU, I/O and production workload.

---

# 27. Application Handover

Once DBA validation succeeds:

```text
Database upgrade completed.
PostgreSQL 18 is running successfully.
Database connectivity is available.
Database objects have been validated.
Please start application services and perform application validation.
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
Replication lag
Application errors
Backup jobs
Monitoring alerts
```

---

# 28. Rollback – Copy Mode

When normal/default **copy mode** is used, the old PostgreSQL 16 data files are not modified by the file-copy operation.

If PostgreSQL 18 validation fails before production writes are accepted:

```bash
systemctl stop postgresql-18
```

Change PostgreSQL 18 away from port 5432 if required.

Start PostgreSQL 16:

```bash
systemctl start postgresql-16
```

Validate:

```bash
/usr/pgsql-16/bin/psql -p 5432
```

This makes copy mode much easier to roll back during the immediate validation window.

---

# 29. Critical Rollback Point – New Production Writes

Suppose PostgreSQL 18 runs for:

```text
10 days
```

and applications have inserted or changed significant data.

You cannot simply stop PostgreSQL 18 and start PostgreSQL 16.

PostgreSQL 16 would contain data only up to the original upgrade cutover.

Any PostgreSQL 18 transactions performed after cutover would be missing.

---

# 30. PostgreSQL 18 → 16 Logical Downgrade Warning

The idea:

```text
Take pg_dumpall from PostgreSQL 18
Restore directly to PostgreSQL 16
```

must **not be treated as a guaranteed rollback procedure**.

PostgreSQL officially guarantees logical dumps for migration toward newer versions.

A dump generated from a newer PostgreSQL version is **not guaranteed to load successfully into an older major version**.

PostgreSQL 18 may contain:

* New SQL syntax.
* New data types.
* New system behavior.
* Features PostgreSQL 16 does not understand.

Therefore a PostgreSQL 18 → 16 logical downgrade must be:

```text
Designed separately
Tested thoroughly
Validated application-by-application
```

It is not a standard PostgreSQL major-version rollback mechanism.

For production, the safest approach is to define a short rollback window before allowing irreversible PostgreSQL 18-only changes.

---

# 31. Clean Up PostgreSQL 16

Do not immediately delete PostgreSQL 16 after application cutover.

Keep it until:

```text
DBA validation complete
Application validation complete
Business validation complete
Monitoring stable
Backup successful
Restore capability confirmed
Rollback window closed
Change approved for closure
```

`pg_upgrade` generates a cleanup script similar to:

```bash
./delete_old_cluster.sh
```

Review it before executing:

```bash
cat ./delete_old_cluster.sh
```

Example:

```bash
rm -rf '/var/lib/pgsql/16/data'
```

Only execute after final approval:

```bash
./delete_old_cluster.sh
```

Then remove old PostgreSQL binaries when appropriate:

```bash
yum remove postgresql16-server postgresql16-contrib -y
```

Do not remove PostgreSQL 16 packages while they are still needed for rollback.

---

# 32. PostgreSQL 16 → 18 Link Mode Upgrade

Another `pg_upgrade` option is:

```text
--link
```

short option:

```text
-k
```

Example:

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433 \
-k
```

Link mode creates hard links between the old and new cluster's user relation files instead of copying them.

Advantages:

```text
Very fast
Minimal additional data-file storage
Useful for very large databases
```

Main disadvantage:

```text
Rollback becomes significantly more restrictive.
```

---

# 33. Linux Inode Basics

The original statement:

```text
Inode file is actual physical file.
```

is not accurate.

An **inode** is a Linux filesystem metadata structure representing a filesystem object.

It contains information such as:

```text
File type
Owner
Permissions
File size
Timestamps
Pointers to data blocks
Link count
```

The directory maps a filename to an inode.

Display inode numbers:

```bash
ls -li
```

Example:

```text
18295529 -rw------- postgres postgres 3 PG_VERSION
```

Here:

```text
18295529
```

is the inode number.

---

# 34. Why Link Mode Uses Hard Links

A hard link is another directory entry pointing to the **same inode/data blocks**.

Therefore PostgreSQL 16 and PostgreSQL 18 relation filenames can reference the same underlying data.

That is why link mode consumes very little additional storage.

It is also why rollback becomes dangerous once PostgreSQL 18 starts modifying those pages.

---

# 35. Link Mode Filesystem Requirement

For `pg_upgrade --link`, old and new data directories must be on the **same filesystem**.

For example:

```text
/pgdata/16/data
/pgdata/18/data
```

on the same filesystem can work.

But:

```text
/mount1/pgsql16
/mount2/pgsql18
```

on separate filesystems cannot be hard-linked.

User tablespaces have corresponding filesystem requirements as well.

---

# 36. Link Mode Upgrade Preparation

Install PostgreSQL 18:

```bash
yum install postgresql18-server postgresql18-contrib -y
```

Initialize with checksum settings matching PostgreSQL 16:

```bash
/usr/pgsql-18/bin/initdb \
-D /var/lib/pgsql/18/data \
--no-data-checksums
```

Configure temporary port:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

```conf
port = 5433
```

Run compatibility check first:

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433 \
-c
```

---

# 37. Stop Application and PostgreSQL 16

Stop application traffic.

Then:

```bash
systemctl stop postgresql-16
```

Validate:

```bash
ps -ef | grep '[p]ostgres'
```

---

# 38. Perform Link Mode Upgrade

Run:

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433 \
-k
```

Typical message:

```text
Adding ".old" suffix to old "global/pg_control"
```

and:

```text
Because "link" mode was used, the old cluster cannot be safely
started once the new cluster has been started.
```

This warning is extremely important.

---

# 39. Configure PostgreSQL 18

Set production port:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

```conf
port = 5432
```

Complete configuration migration.

---

# 40. Start PostgreSQL 18

Start:

```bash
systemctl start postgresql-18
```

Run required post-upgrade analysis:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only
```

Then:

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-only
```

Validate:

```bash
psql -d dvdrental
```

Tables:

```sql
\dt
```

Views:

```sql
\dv
```

---

# 41. Critical Link-Mode Rollback Rule

This is the most important correction in the class notes.

The original sequence suggested:

```bash
systemctl stop postgresql-18

cd /var/lib/pgsql/16/data/global

mv pg_control.old pg_control

systemctl start postgresql-16
```

after PostgreSQL 18 had already been started and used.

**Do not do this.**

With `--link`, the old and new clusters share user relation files.

Once PostgreSQL 18 starts, it can modify those shared files.

At that point PostgreSQL 16 must **not** be started against them.

Restoring:

```bash
mv pg_control.old pg_control
```

does not make the shared data PostgreSQL-16-compatible again.

Doing so can result in corruption or an inconsistent cluster.

---

# 42. Link Mode Rollback Window

Before PostgreSQL 18 has ever been started, the old cluster can potentially be restored according to the exact `pg_upgrade` generated instructions.

Once PostgreSQL 18 has started:

```text
Do not start the original PostgreSQL 16 linked cluster.
```

If strong rollback requirements exist, prefer:

```text
pg_upgrade copy mode
```

or use:

```text
Storage snapshots
Filesystem snapshots
Clone mode where supported
Separate physical backup
Logical replication migration
```

depending on the production architecture.

---

# 43. PostgreSQL 18 Clone Mode

PostgreSQL also supports:

```text
--clone
```

on supported filesystems.

Clone mode can provide:

```text
Fast upgrade
Low initial additional disk usage
Better rollback characteristics than link mode
```

because copy-on-write filesystem cloning does not make the old cluster unusable simply because the new cluster starts.

Currently supported examples include appropriate Linux Btrfs/XFS configurations and macOS APFS.

Filesystem support must be validated before choosing this mode.

---

# 44. Copy vs Link vs Clone

| Mode              | Data Copy            |   Extra Space | Speed     | Old Cluster After New Starts |
| ----------------- | -------------------- | ------------: | --------- | ---------------------------- |
| Copy              | Full copy            |          High | Slower    | Remains independent          |
| Link              | Hard links           |      Very low | Very fast | Unsafe to restart            |
| Clone             | Copy-on-write        | Low initially | Very fast | Remains independent          |
| `copy_file_range` | Filesystem dependent |       Depends | Fast      | Independent                  |

For most environments where rollback simplicity matters, copy mode is easier to manage.

For multi-terabyte clusters where downtime is critical, link/clone or logical-replication-based approaches may be more attractive after proper testing.

---

# 45. Minor Version Upgrade / Patching

PostgreSQL minor releases contain fixes for:

```text
Security vulnerabilities
Data corruption risks
Crashes
Replication problems
Query correctness issues
Performance bugs
Low-risk improvements
```

Example:

```text
18.2 → 18.4
```

or, as of August 2026:

```text
18.4 → 18.6
```

A minor update does not require:

```text
pg_upgrade
pg_dump
pg_restore
new data directory
```

The existing data directory remains in place.

---

# 46. Current Minor Version Example

As of 31 August 2026:

```text
PostgreSQL 18 current minor release = 18.6
PostgreSQL 16 current minor release = 16.15
```

Therefore a current example would be:

```text
18.4 → 18.6
```

rather than treating 18.4 as the latest available release.

---

# 47. Install Specific PostgreSQL Minor Version

Example installation:

```bash
yum install \
postgresql18-server-18.2 \
postgresql18-contrib-18.2
```

Depending on repository/package dependencies, matching packages may also include:

```text
postgresql18
postgresql18-libs
```

Validate:

```bash
rpm -qa | grep postgresql18
```

Example:

```text
postgresql18-libs-18.2
postgresql18-18.2
postgresql18-server-18.2
postgresql18-contrib-18.2
```

---

# 48. Initialize PostgreSQL 18

For a fresh laboratory cluster:

```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
```

Start:

```bash
systemctl start postgresql-18
```

Connect:

```bash
su - postgres
```

```bash
psql
```

Check:

```sql
SELECT version();
```

Create database:

```sql
CREATE DATABASE dvdrental;
```

Exit:

```sql
\q
```

Restore:

```bash
pg_restore \
-Ft \
-d dvdrental \
/tmp/dvdrental.tar \
-v \
2>restore.log
```

Validate:

```bash
psql -d dvdrental
```

```sql
\dt
```

---

# 49. PostgreSQL 18.2 → Latest PostgreSQL 18 Minor Release

Before update:

```bash
psql -c "SELECT version();"
```

Stop application traffic.

Stop PostgreSQL:

```bash
systemctl stop postgresql-18
```

Update packages:

```bash
yum update postgresql18\* -y
```

or using the repository packaging command appropriate for the environment.

The class example used:

```bash
yum install postgresql18-server postgresql18-contrib -y
```

which can upgrade installed packages when a newer repository version is available.

For a production environment, explicitly confirm what package version the repository will install:

```bash
yum list --showduplicates postgresql18-server
```

or:

```bash
dnf list --showduplicates postgresql18-server
```

Then perform the controlled update.

---

# 50. Start PostgreSQL After Minor Upgrade

Start:

```bash
systemctl start postgresql-18
```

Validate:

```bash
systemctl status postgresql-18
```

Check:

```bash
/usr/pgsql-18/bin/psql
```

```sql
SELECT version();
```

Check database list:

```sql
\l
```

Check data directory:

```sql
SHOW data_directory;
```

The data directory remains:

```text
/var/lib/pgsql/18/data
```

because a minor upgrade does not require a new PostgreSQL data directory.

---

# 51. Minor Upgrade Production Procedure

Recommended sequence:

```text
1. Read release notes.
2. Review security fixes and required follow-up actions.
3. Test upgrade in non-production.
4. Confirm package repository.
5. Take verified backup.
6. Obtain change approval.
7. Stop application traffic.
8. Stop PostgreSQL.
9. Update PostgreSQL packages.
10. Start PostgreSQL.
11. Confirm server version.
12. Review PostgreSQL logs.
13. Validate application connectivity.
14. Execute release-specific follow-up actions.
15. Start application.
16. Monitor environment.
```

---

# 52. Important Minor-Release Release-Notes Rule

Do not assume every minor upgrade means only:

```text
replace binaries → restart → finished
```

That is the normal core process, but PostgreSQL release notes may require additional actions.

For example, some releases can contain recommendations involving:

```text
REINDEX
Configuration changes
Data cleanup
Extension maintenance
Security-related configuration changes
```

Therefore:

```text
Always read every skipped minor release note.
```

This is especially important when moving across multiple minor versions.

---

# 53. Minor-Version Rollback

PostgreSQL minor releases within the same major release maintain compatible physical data formats.

Therefore, in principle:

```text
18.2 data directory
18.4 data directory
18.6 data directory
```

belong to the same PostgreSQL 18 major-version storage family.

However, production rollback should not simply mean:

```text
downgrade packages blindly
```

because a newer minor release may have required:

```text
REINDEX operations
Security changes
Configuration changes
Extension updates
Manual cleanup
```

These actions may not automatically reverse when old binaries are installed.

Therefore the production rollback plan should include:

```text
Exact packages
Repository availability
Snapshot/backup
Extension compatibility
Release-note review
Application validation
```

---

# 54. Example Package Downgrade

The lab demonstrated:

```bash
systemctl stop postgresql-18
```

Then installation of an older package:

```bash
yum install \
postgresql18-server-18.2 \
postgresql18-contrib-18.2 \
-y
```

Depending on the package manager, explicitly using a downgrade operation can be clearer:

```bash
dnf downgrade \
postgresql18 \
postgresql18-libs \
postgresql18-server \
postgresql18-contrib
```

The exact package versions must exist in the repository.

Start:

```bash
systemctl start postgresql-18
```

Validate:

```bash
psql
```

```sql
SELECT version();
```

```sql
\c dvdrental
```

```sql
\dt
```

---

# 55. Major Upgrade vs Minor Upgrade

| Area                                 | Major Upgrade     | Minor Upgrade       |
| ------------------------------------ | ----------------- | ------------------- |
| Example                              | 16 → 18           | 18.4 → 18.6         |
| New data directory                   | Yes               | No                  |
| `pg_upgrade` needed                  | Possible          | No                  |
| Dump/restore possible                | Yes               | Not normally needed |
| Logical replication possible         | Yes               | Not normally needed |
| Storage format guaranteed compatible | No                | Yes                 |
| Application compatibility testing    | Mandatory         | Recommended         |
| Release-note review                  | Mandatory         | Mandatory           |
| Downtime                             | Depends on method | Usually short       |

---

# 56. Major Upgrade Options

PostgreSQL major upgrades can normally use:

```text
pg_dump / pg_restore
pg_dumpall
pg_upgrade
Logical replication
Third-party/managed migration tooling
```

For PostgreSQL 16 → 18, PostgreSQL does **not require PostgreSQL 17 as an intermediate step**.

A direct upgrade can be performed:

```text
16 → 18
```

provided the selected migration method supports the environment and compatibility checks succeed.

Always read release notes for:

```text
PostgreSQL 17
PostgreSQL 18
```

before performing the direct jump.

---

# 57. `pg_upgrade` Production Checklist

Before `pg_upgrade`, verify:

```text
Source PostgreSQL version
Target PostgreSQL version
Operating system compatibility
Binary directories
Data directories
Tablespaces
Disk space
Filesystem
Data checksum state
Locale
Encoding
Extensions
Shared libraries
Authentication configuration
SSL configuration
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

Server version only:

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
SELECT *
FROM pg_replication_slots;
```

Replication status:

```sql
SELECT *
FROM pg_stat_replication;
```

Prepared transactions:

```sql
SELECT *
FROM pg_prepared_xacts;
```

Installed extensions:

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
Backup completed successfully.
```

Also verify the backup.

For `pg_basebackup` backups containing a manifest:

```bash
pg_verifybackup /backup/pg16_preupgrade
```

For pgBackRest:

```bash
pgbackrest --stanza=prod check
```

and review:

```bash
pgbackrest --stanza=prod info
```

The exact verification method depends on the backup architecture.

---

# 60. Do Not Forget Replication

If PostgreSQL 16 is part of a primary/standby environment, upgrading only the primary is not the complete procedure.

The upgrade plan must cover:

```text
Primary
Standbys
Replication slots
Archive/WAL
Replication users
Replication configuration
Failover tooling
Patroni/repmgr if applicable
Connection endpoints
```

`pg_upgrade` documentation contains separate considerations for standby servers.

---

# 61. Do Not Delete the Old Cluster Immediately

After upgrade:

```text
PostgreSQL 18 starts successfully
```

does **not** mean:

```text
delete PostgreSQL 16 immediately
```

Wait until:

```text
Application testing complete
Smoke testing complete
Performance acceptable
Backups working
Monitoring working
Replication healthy
Business acceptance complete
Rollback period closed
```

Only then run:

```bash
./delete_old_cluster.sh
```

---

# 62. Major Corrections to Original Class Notes

| Original Statement                               | Corrected Understanding                                                                 |
| ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| `16.9.10 = major.minor.patch`                    | Invalid for modern PostgreSQL. `16.15` = major 16, minor 15                             |
| PostgreSQL 11 onward no separate patch           | Modern two-part numbering began with PostgreSQL 10                                      |
| `pg_upgrade` came from 8.4                       | Official `pg_upgrade` appeared in PostgreSQL 9.0                                        |
| PostgreSQL 18 checksums must always be disabled  | Only if required to match the source cluster's checksum state                           |
| `pg_upgrade` does not transfer statistics        | PostgreSQL 18 transfers most optimizer statistics                                       |
| Copy `postgresql.conf` from 16 to 18             | Merge and review settings; do not blindly replace                                       |
| 100 MB free disk is enough for PG18 software     | No safe universal fixed number; validate actual packages/filesystem                     |
| inode is the physical file                       | Inode is filesystem metadata identifying the filesystem object                          |
| Link-mode PG16 can restart after PG18 has run    | Unsafe; old cluster must not be restarted after new linked cluster starts               |
| PG18 `pg_dumpall` can always restore to PG16     | Not guaranteed; PostgreSQL does not guarantee dump compatibility toward older majors    |
| All minor downgrade is automatically safe        | Storage format is compatible, but release-note actions and extensions still need review |
| `pg_stat_user_tables` gives cluster object count | It only describes the currently connected database                                      |

---

# 63. Production Recommendation – PostgreSQL 16 → 18

For a normal production environment where sufficient storage is available:

```text
PostgreSQL 16
       |
       | pg_upgrade --check
       v
Maintenance Window
       |
       | Stop Application
       | Stop PostgreSQL 16
       |
       v
pg_upgrade COPY MODE
       |
       v
PostgreSQL 18
       |
       | Post-upgrade validation
       | Statistics maintenance
       | Extension validation
       | Application validation
       v
Production
```

Advantages:

```text
Fast compared with dump/restore
Old cluster remains independent
Rollback is simpler during the immediate validation window
```

For very large databases where downtime and disk space are major concerns, evaluate:

```text
pg_upgrade --link
pg_upgrade --clone
Logical replication
```

with a fully tested rollback design.

---

# 64. Quick Reference Cheat Sheet

## Major Upgrade

```text
16 → 18
```

Methods:

```text
pg_upgrade
pg_dump/pg_restore
pg_dumpall
Logical replication
```

---

## Minor Upgrade

```text
18.4 → 18.6
```

Typical process:

```text
Stop application
Stop PostgreSQL
Update packages
Start PostgreSQL
Validate
Perform release-specific actions
Start application
```

No:

```text
pg_upgrade
new data directory
dump/restore
```

normally required.

---

## `pg_upgrade` Check

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433 \
-c
```

---

## Copy Mode

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433
```

---

## Link Mode

```bash
/usr/pgsql-18/bin/pg_upgrade \
-b /usr/pgsql-16/bin \
-B /usr/pgsql-18/bin \
-d /var/lib/pgsql/16/data \
-D /var/lib/pgsql/18/data \
-p 5432 \
-P 5433 \
-k
```

Remember:

```text
Once PG18 starts in link mode,
do not restart the original PG16 linked cluster.
```

---

## PostgreSQL 18 Post-Upgrade Statistics

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only
```

```bash
/usr/pgsql-18/bin/vacuumdb \
--all \
--analyze-only
```

---

# 65. Final DBA Takeaways

### Major Version

```text
16 → 18
```

requires a migration mechanism because the data directory is not directly reusable across major versions.

### Minor Version

```text
18.4 → 18.6
```

keeps the same data directory and normally requires replacing/updating binaries and restarting PostgreSQL.

### `pg_upgrade`

Use the **new PostgreSQL version's `pg_upgrade` binary**.

For PostgreSQL 16 → 18:

```text
/usr/pgsql-18/bin/pg_upgrade
```

### Copy Mode

Uses additional storage but gives the cleanest immediate rollback position because the old cluster remains independent.

### Link Mode

Uses very little extra space and is fast, but the rollback window effectively ends once the PostgreSQL 18 cluster starts using the linked files.

### PostgreSQL 18 Checksums

PostgreSQL 18 enables data checksums by default for new clusters. Ensure source and target checksum settings match for `pg_upgrade`.

### PostgreSQL 18 Statistics

PostgreSQL 18 `pg_upgrade` transfers most optimizer statistics, but the recommended post-upgrade statistics maintenance should still be performed.

### Configuration

Never blindly replace PostgreSQL 18 configuration with PostgreSQL 16 configuration.

Compare, review and migrate only appropriate parameters.

### Rollback

A good PostgreSQL production upgrade is not simply:

```text
Can we upgrade?
```

The correct question is:

```text
Can we upgrade, validate, recover and safely roll back if required?
```

That rollback plan must be tested before production.

---

# Official PostgreSQL References

**PostgreSQL Versioning Policy**
[https://www.postgresql.org/support/versioning/](https://www.postgresql.org/support/versioning/)

**PostgreSQL 18 – Upgrading a PostgreSQL Cluster**
[https://www.postgresql.org/docs/18/upgrading.html](https://www.postgresql.org/docs/18/upgrading.html)

**PostgreSQL 18 – pg_upgrade**
[https://www.postgresql.org/docs/18/pgupgrade.html](https://www.postgresql.org/docs/18/pgupgrade.html)

**PostgreSQL 18 – pg_dump**
[https://www.postgresql.org/docs/18/app-pgdump.html](https://www.postgresql.org/docs/18/app-pgdump.html)

**PostgreSQL 18 – pg_dumpall**
[https://www.postgresql.org/docs/18/app-pg-dumpall.html](https://www.postgresql.org/docs/18/app-pg-dumpall.html)

**PostgreSQL 18 – Data Checksums**
[https://www.postgresql.org/docs/18/checksums.html](https://www.postgresql.org/docs/18/checksums.html)

**PostgreSQL 18 Release Notes**
[https://www.postgresql.org/docs/18/release-18.html](https://www.postgresql.org/docs/18/release-18.html)

**PostgreSQL 18.6 Release Notes**
[https://www.postgresql.org/docs/18/release-18-6.html](https://www.postgresql.org/docs/18/release-18-6.html)

---

## Summary

```text
MAJOR UPGRADE
16 → 18
Requires:
pg_upgrade / dump-restore / logical replication

MINOR UPGRADE
18.4 → 18.6
Requires:
Update binaries/packages + restart

COPY MODE
More storage
Simpler immediate rollback

LINK MODE
Very little additional storage
Very fast
Old cluster unsafe after new cluster starts

POSTGRESQL 18
Checksums enabled by default for newly initialized clusters
pg_upgrade transfers most optimizer statistics
```

[1]: https://www.postgresql.org/support/versioning/?utm_source=chatgpt.com "PostgreSQL: Versioning Policy"
[2]: https://www.postgresql.org/docs/18/pgupgrade.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_upgrade"
