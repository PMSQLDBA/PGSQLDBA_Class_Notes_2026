# PostgreSQL – 10th Class – 15th August 2026

## Topics Covered

1. YUM / DNF, Repository, RPM and Package concepts
2. PostgreSQL PGDG repository configuration
3. PostgreSQL 18 installation
4. PostgreSQL cluster initialization
5. PostgreSQL service management
6. PostgreSQL automatic startup
7. PostgreSQL uninstallation and reinstallation
8. PostgreSQL Data Directory (`PGDATA`)
9. Important files and directories inside `PGDATA`
10. Configuration file priority
11. Important PostgreSQL administration commands

---

# 1. YUM / DNF, Repository, RPM and Package Basics

## Repository

A **repository (repo)** is a configured source from which Linux package managers such as `dnf` or `yum` can locate and install software packages.

A repository may point to:

* An online package repository
* A local/offline repository
* A directory containing RPM packages
* A network-accessible package location

On RHEL-based systems, repository configuration files are normally stored under:

```bash
/etc/yum.repos.d/
```

Example:

```bash
cd /etc/yum.repos.d/
ls -ltr
```

A repository configuration file normally has the `.repo` extension.

Example:

```text
pgdg-redhat-all.repo
```

---

# 2. RPM

**RPM** stands for **RPM Package Manager**.

An RPM package contains software files together with package metadata such as:

* Package name
* Version
* Architecture
* Dependencies
* Installation information

Example PostgreSQL RPM packages:

```text
postgresql18
postgresql18-libs
postgresql18-server
postgresql18-contrib
```

To display installed PostgreSQL-related RPM packages:

```bash
rpm -qa | grep postgres
```

---

# 3. YUM and DNF

`yum` and `dnf` are package-management commands used on Red Hat-based Linux distributions.

They can:

* Search for packages
* Install packages
* Remove packages
* Resolve dependencies
* Access configured repositories
* Update packages

Examples:

```bash
dnf search postgresql
```

```bash
dnf install postgresql18-server -y
```

```bash
dnf remove postgresql18-server -y
```

On newer RHEL releases, `dnf` is the primary package manager.

---

# 4. PostgreSQL PGDG Repository

PostgreSQL provides the **PostgreSQL Global Development Group (PGDG) Yum Repository** for supported Red Hat-family Linux distributions.

Official PostgreSQL Red Hat download instructions should be used to select the correct:

* Linux distribution
* Distribution version
* CPU architecture
* PostgreSQL major version

For this class, the environment was based on:

```text
PostgreSQL major version : 18
Architecture              : x86_64
Linux family              : Red Hat
```

The class used a PGDG repository RPM similar to:

```bash
dnf install -y \
https://download.postgresql.org/pub/repos/yum/reporpms/EL-10-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

After installing the repository RPM:

```bash
cd /etc/yum.repos.d/
ls -ltr
```

You should see the PGDG repository configuration, for example:

```text
pgdg-redhat-all.repo
```

---

# 5. Verify PostgreSQL Packages

After configuring the repository, refresh/search the package metadata:

```bash
dnf search postgresql
```

For a PostgreSQL 18-specific search:

```bash
dnf search postgresql18
```

You may find packages such as:

```text
postgresql18
postgresql18-libs
postgresql18-server
postgresql18-contrib
```

### Package Purpose

**`postgresql18`**

Contains PostgreSQL client programs and related components.

**`postgresql18-libs`**

Contains libraries required by PostgreSQL.

**`postgresql18-server`**

Contains the PostgreSQL database server programs.

**`postgresql18-contrib`**

Contains additional supplied modules/extensions and utilities packaged separately by PGDG.

---

# 6. Install PostgreSQL 18 Server

Install the PostgreSQL server:

```bash
dnf install postgresql18-server -y
```

Install the contrib package:

```bash
dnf install postgresql18-contrib -y
```

Or install both together:

```bash
dnf install postgresql18-server postgresql18-contrib -y
```

Verify:

```bash
rpm -qa | grep postgresql
```

Example:

```text
postgresql18-libs-...
postgresql18-...
postgresql18-server-...
postgresql18-contrib-...
```

---

# 7. PostgreSQL Binary Directory

With the PGDG PostgreSQL 18 packages, PostgreSQL binaries are normally installed under:

```bash
/usr/pgsql-18/bin/
```

Check:

```bash
cd /usr/pgsql-18/bin/
ls -ltr
```

Important utilities include:

```text
postgres
psql
pg_ctl
initdb
pg_dump
pg_dumpall
pg_restore
pg_basebackup
pg_upgrade
pg_rewind
pg_isready
pg_controldata
pg_checksums
pg_waldump
pg_receivewal
pg_amcheck
createdb
dropdb
createuser
dropuser
vacuumdb
reindexdb
clusterdb
pgbench
```

### Important Utilities

`postgres`

The PostgreSQL database server executable.

`psql`

PostgreSQL command-line client.

`pg_ctl`

Controls a PostgreSQL server: start, stop, restart, reload and status operations.

`initdb`

Creates and initializes a new PostgreSQL database cluster.

`pg_dump`

Creates a logical backup of one database.

`pg_dumpall`

Creates logical backups of all databases and cluster-wide objects.

`pg_restore`

Restores archives created by `pg_dump` in non-plain formats.

`pg_basebackup`

Creates a physical base backup of a PostgreSQL cluster.

`pg_upgrade`

Upgrades a PostgreSQL cluster between major versions.

`pg_isready`

Checks whether a PostgreSQL server is accepting connections.

`pg_controldata`

Displays control information about a PostgreSQL database cluster.

---

# 8. Initialize the PostgreSQL Database Cluster

Installing the PostgreSQL server software does **not** by itself create a usable database cluster.

The database cluster must be initialized.

With the PGDG PostgreSQL 18 RPM installation:

```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
```

Expected output:

```text
Initializing database ... OK
```

Initialization creates the PostgreSQL data directory and its required internal files.

For the class installation, the data directory is:

```text
/var/lib/pgsql/18/data/
```

---

# 9. What Is a PostgreSQL Database Cluster?

In PostgreSQL terminology, a **database cluster** is a collection of databases managed by a single PostgreSQL server instance.

This should not be confused with an operating-system or high-availability cluster.

One PostgreSQL database cluster contains:

* Multiple databases
* Roles/users
* Tablespaces
* Configuration
* WAL
* Transaction information
* Database objects
* Internal metadata

The cluster is represented on disk primarily by its **data directory (`PGDATA`)**.

---

# 10. PostgreSQL Data Directory – PGDATA

The PostgreSQL data directory is one of the most important directories in PostgreSQL.

It contains the database cluster's:

* Database files
* WAL files
* Configuration files
* Transaction status
* Replication slot information
* Statistics files
* Internal metadata
* Server state files

For this installation:

```text
PGDATA=/var/lib/pgsql/18/data/
```

Check the contents:

```bash
cd /var/lib/pgsql/18/data/
ls -ltr
```

---

# 11. PostgreSQL Service Management

The PGDG package provides a systemd service for PostgreSQL 18.

Check available PostgreSQL service files:

```bash
systemctl list-unit-files | grep postgresql
```

Example:

```text
postgresql-18.service
```

Check service status:

```bash
systemctl status postgresql-18
```

---

# 12. Start PostgreSQL

```bash
systemctl start postgresql-18
```

Verify:

```bash
systemctl status postgresql-18
```

A running server should show:

```text
Active: active (running)
```

Another useful verification:

```bash
ps -ef | grep postgres
```

A PostgreSQL instance normally has several processes, such as:

```text
postgres
checkpointer
background writer
walwriter
autovacuum launcher
logical replication launcher
```

Depending on the PostgreSQL version and configuration, additional background processes may also be present.

---

# 13. Stop PostgreSQL

```bash
systemctl stop postgresql-18
```

Verify:

```bash
systemctl status postgresql-18
```

The service should show:

```text
Active: inactive (dead)
```

---

# 14. Restart PostgreSQL

```bash
systemctl restart postgresql-18
```

A restart performs a stop followed by a start.

Existing database connections are interrupted, so a restart can cause application downtime.

Use a restart only when it is actually required.

---

# 15. Reload PostgreSQL Configuration

Many PostgreSQL configuration changes can be applied without restarting the database.

Using systemd:

```bash
systemctl reload postgresql-18
```

Using `pg_ctl`:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data reload
```

From SQL:

```sql
SELECT pg_reload_conf();
```

A reload sends PostgreSQL a configuration reload request.

However, **not every PostgreSQL parameter can be changed with a reload**. Some parameters require a complete server restart.

---

# 16. Check Server Status Using pg_ctl

Switch to the PostgreSQL operating-system user:

```bash
su - postgres
```

Then:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data status
```

Example:

```text
pg_ctl: server is running
```

`pg_ctl` is PostgreSQL's native server-control utility.

On a systemd-based Linux server, administrators normally use `systemctl` for the packaged PostgreSQL service.

---

# 17. Enable PostgreSQL Automatic Startup

Starting PostgreSQL manually does not necessarily mean it will start automatically after a server reboot.

Enable the service:

```bash
systemctl enable postgresql-18
```

A convenient command that enables and starts it is:

```bash
systemctl enable --now postgresql-18
```

Verify:

```bash
systemctl status postgresql-18
```

and:

```bash
systemctl is-enabled postgresql-18
```

Expected:

```text
enabled
```

---

# 18. PostgreSQL Data Directory Location

For this PGDG PostgreSQL 18 installation:

```text
/var/lib/pgsql/18/data/
```

The service configuration determines which data directory is used by the server.

Before changing `PGDATA`, make sure:

* The new directory exists
* Ownership is correct
* Permissions are correct
* Existing database files are safely migrated
* SELinux configuration is considered
* The service configuration points to the correct directory

Do **not** simply change the path on a production database without planning the migration.

Useful SQL command:

```sql
SHOW data_directory;
```

This shows the data directory currently being used by PostgreSQL.

---

# 19. PostgreSQL Uninstallation

First stop the service:

```bash
systemctl stop postgresql-18
```

Remove the required packages:

```bash
dnf remove postgresql18-server postgresql18-contrib -y
```

Check remaining PostgreSQL packages:

```bash
rpm -qa | grep postgresql
```

### Important Warning

Removing PostgreSQL packages and deleting the PostgreSQL data directory are **two different operations**.

Commands such as:

```bash
rm -rf /var/lib/pgsql/18
```

permanently delete the database cluster.

Never run such commands on an important system unless:

* A verified backup exists
* The database is no longer required
* The correct path has been confirmed
* The deletion has been approved

---

# 20. Reinstall PostgreSQL

Install:

```bash
dnf install postgresql18-server postgresql18-contrib -y
```

Initialize a new cluster:

```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
```

Start:

```bash
systemctl start postgresql-18
```

Enable automatic startup if required:

```bash
systemctl enable postgresql-18
```

Verify:

```bash
systemctl status postgresql-18
```

---

# 21. PostgreSQL Data Directory Structure

A typical PostgreSQL 18 `PGDATA` contains items such as:

```text
PG_VERSION
base
global
pg_commit_ts
pg_dynshmem
pg_logical
pg_multixact
pg_notify
pg_replslot
pg_serial
pg_snapshots
pg_stat
pg_stat_tmp
pg_subtrans
pg_tblspc
pg_twophase
pg_wal
pg_xact

postgresql.conf
postgresql.auto.conf
pg_hba.conf
pg_ident.conf

postmaster.opts
postmaster.pid
current_logfiles
```

Some files/directories exist only under particular configurations or while PostgreSQL is running.

---

# 22. PG_VERSION

`PG_VERSION` contains the PostgreSQL **major version** of the database cluster.

Example:

```bash
cat PG_VERSION
```

Output:

```text
18
```

This tells us that this data directory belongs to a PostgreSQL 18 cluster.

---

# 23. base Directory

```text
$PGDATA/base/
```

The `base` directory contains **per-database subdirectories** for databases stored in the default tablespace.

For example:

```bash
cd $PGDATA/base
ls -ltr
```

You may see:

```text
1
4
5
...
```

These numbers are database OIDs.

To map database names to their OIDs:

```sql
SELECT oid, datname
FROM pg_database
ORDER BY oid;
```

Example concept:

```text
Database Name    OID
-------------    ---
template1          1
template0          4
postgres           5
```

The exact OIDs should always be checked on the actual system.

---

# 24. Relation Files Under base

Inside a database directory, PostgreSQL stores relation data in files whose names are generally based on relation file identifiers.

For example:

```text
1255
1255_fsm
1255_vm
```

Possible suffixes include:

```text
_fsm
_vm
_init
```

### `_fsm`

Free Space Map.

Tracks available free space in relation pages.

### `_vm`

Visibility Map.

Tracks pages whose tuples are visible to all transactions and pages known to contain only frozen tuples.

### `_init`

Initialization fork used by unlogged relations.

DBAs normally **must not manually modify files inside `base`**.

---

# 25. global Directory

```text
$PGDATA/global/
```

The `global` directory contains **cluster-wide system catalog tables**.

These are objects that apply to the whole PostgreSQL cluster rather than to one individual database.

An example is the underlying storage associated with cluster-wide catalog information such as databases and roles.

Do not manually edit files in this directory.

---

# 26. current_logfiles

```text
$PGDATA/current_logfiles
```

When PostgreSQL's logging collector is enabled, this file records the log file or files currently being written by the logging collector.

Example:

```bash
cat current_logfiles
```

Possible output:

```text
stderr log/postgresql-Tue.log
```

### Correction

`current_logfiles` does **not** mean “all audit log files currently available.”

It specifically records the current log files being written by the PostgreSQL logging collector.

---

# 27. log Directory

Example:

```text
$PGDATA/log/
```

The actual log location is controlled by PostgreSQL configuration, especially:

```text
logging_collector
log_directory
log_filename
log_destination
```

Check:

```sql
SHOW logging_collector;
SHOW log_directory;
SHOW log_filename;
SHOW log_destination;
```

Therefore, PostgreSQL logs are not guaranteed to be stored in `$PGDATA/log` on every server.

---

# 28. pg_commit_ts

```text
$PGDATA/pg_commit_ts/
```

Contains **transaction commit timestamp data**.

This feature is associated with PostgreSQL's ability to track transaction commit timestamps.

It should not be described simply as “permanently stored data.”

---

# 29. pg_dynshmem

```text
$PGDATA/pg_dynshmem/
```

Contains files used by PostgreSQL's **dynamic shared memory subsystem**.

Dynamic shared memory can be used internally by PostgreSQL features such as parallel processing.

---

# 30. pg_ident.conf

```text
$PGDATA/pg_ident.conf
```

`pg_ident.conf` defines **user-name maps**.

It maps an authenticated external/system user name to a PostgreSQL database user name when an authentication method requires such mapping.

### Correction

It is not simply “the LDAP source and target user file.”

LDAP authentication does not generally require `pg_ident.conf`.

User-name maps are particularly relevant to authentication methods that support mapped identities.

---

# 31. pg_logical

```text
$PGDATA/pg_logical/
```

Contains status information related to **logical decoding**.

Logical decoding is the mechanism used to extract database changes from WAL in a logical form.

It is used by features such as logical replication.

---

# 32. pg_notify

```text
$PGDATA/pg_notify/
```

Contains status information for PostgreSQL's:

```sql
LISTEN
NOTIFY
```

functionality.

### Correction

It is not a general location for PostgreSQL error, warning and informational messages.

Those messages normally belong in PostgreSQL's logging system.

---

# 33. pg_replslot

```text
$PGDATA/pg_replslot/
```

Contains **replication slot state data**.

Replication slots allow PostgreSQL to retain resources required by replication consumers.

For physical replication slots, this can prevent required WAL from being removed before a standby/consumer has received it.

For logical replication slots, PostgreSQL also retains information needed for logical decoding.

### Important Operational Point

Replication slots are powerful but must be monitored.

If a replication consumer stops consuming WAL for a long time, a slot can cause WAL retention and potentially consume a large amount of disk space.

Useful query:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

---

# 34. pg_serial

```text
$PGDATA/pg_serial/
```

Contains information about **committed serializable transactions**.

It supports PostgreSQL's Serializable Snapshot Isolation implementation.

---

# 35. pg_snapshots

```text
$PGDATA/pg_snapshots/
```

Contains **exported snapshots**.

Snapshots may be exported using PostgreSQL functionality such as:

```sql
SELECT pg_export_snapshot();
```

### Correction

This directory should not be described as a collection of snapshots automatically created for server-crash analysis.

Its documented purpose is to contain exported snapshots.

---

# 36. pg_stat

```text
$PGDATA/pg_stat/
```

Contains permanent files used by PostgreSQL's statistics subsystem.

---

# 37. pg_stat_tmp

```text
$PGDATA/pg_stat_tmp/
```

Contains temporary files used by PostgreSQL's statistics subsystem.

The exact implementation and use of statistics files can change between PostgreSQL versions, so DBAs should avoid manually manipulating these files.

---

# 38. pg_subtrans

```text
$PGDATA/pg_subtrans/
```

Contains **subtransaction status data**.

Subtransactions can be created through mechanisms such as savepoints.

Example:

```sql
BEGIN;

SAVEPOINT s1;

UPDATE employee
SET salary = salary + 1000
WHERE emp_id = 10;

ROLLBACK TO SAVEPOINT s1;

COMMIT;
```

---

# 39. pg_tblspc

```text
$PGDATA/pg_tblspc/
```

Contains **symbolic links to user-defined tablespaces**.

Example:

```bash
cd $PGDATA/pg_tblspc
ls -ltr
```

When no user-defined tablespaces exist, the directory may be empty.

Check tablespaces from SQL:

```sql
SELECT
    oid,
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

---

# 40. pg_twophase

```text
$PGDATA/pg_twophase/
```

Contains state files for **prepared transactions** used by PostgreSQL's two-phase commit functionality.

Typical SQL:

```sql
BEGIN;

-- SQL operations

PREPARE TRANSACTION 'txn_001';
```

Later:

```sql
COMMIT PREPARED 'txn_001';
```

or:

```sql
ROLLBACK PREPARED 'txn_001';
```

This is used when a transaction needs to be prepared first and committed or rolled back later.

---

# 41. pg_wal

```text
$PGDATA/pg_wal/
```

Contains PostgreSQL **Write-Ahead Log (WAL)** files.

Example:

```bash
cd $PGDATA/pg_wal
ls -ltr
```

A WAL segment may look like:

```text
000000010000000000000001
```

WAL is fundamental to:

* Crash recovery
* Durability
* Physical replication
* Point-in-time recovery
* Backup/recovery
* Logical decoding

---

# 42. WAL Segment Size

The default WAL segment size is:

```text
16 MB
```

However, an important additional point is that PostgreSQL can be initialized with a different WAL segment size using an `initdb` option.

Therefore:

> **16 MB is the default WAL segment size, not an absolute fixed size for every PostgreSQL cluster.**

A useful command is:

```bash
pg_controldata $PGDATA | grep "Bytes per WAL segment"
```

---

# 43. pg_xact

```text
$PGDATA/pg_xact/
```

Contains **transaction commit status data**.

### Correction

`pg_xact` does not store “query runtime details.”

It tracks transaction status information needed by PostgreSQL's transaction and visibility mechanisms.

---

# 44. pg_hba.conf

```text
$PGDATA/pg_hba.conf
```

HBA stands for:

```text
Host-Based Authentication
```

This file controls **client authentication**.

It can determine:

* Which clients may connect
* Which databases they may connect to
* Which PostgreSQL users they may connect as
* Which network addresses are allowed
* Which authentication method is used

Typical entry format:

```text
TYPE  DATABASE  USER  ADDRESS  METHOD
```

Example:

```text
host  all  all  192.168.1.0/24  scram-sha-256
```

---

# 45. Reloading pg_hba.conf

After changing `pg_hba.conf`, a full PostgreSQL restart is normally unnecessary.

Reload:

```bash
systemctl reload postgresql-18
```

or:

```sql
SELECT pg_reload_conf();
```

or:

```bash
pg_ctl -D /var/lib/pgsql/18/data reload
```

### Important

Always validate authentication changes carefully.

A wrong `pg_hba.conf` rule can block applications or administrators from connecting.

---

# 46. postgresql.conf

```text
$PGDATA/postgresql.conf
```

This is PostgreSQL's main server configuration file.

It contains parameters controlling areas such as:

* Connections
* Memory
* WAL
* Replication
* Logging
* Autovacuum
* Query planning
* Resource consumption
* Background processes

Examples:

```text
listen_addresses
port
max_connections
shared_buffers
work_mem
maintenance_work_mem
wal_level
max_wal_size
checkpoint_timeout
logging_collector
log_directory
autovacuum
```

---

# 47. Reload vs Restart Parameters

Not every PostgreSQL parameter behaves the same way.

Some changes can take effect:

* Immediately for the current session
* For new sessions
* After a configuration reload
* Only after a server restart

To check parameter details:

```sql
SELECT
    name,
    setting,
    unit,
    context,
    pending_restart
FROM pg_settings
WHERE name = 'max_connections';
```

The `context` column helps identify when a setting can be changed.

`pending_restart = true` means the configured value differs from the active value and a restart is required.

---

# 48. postgresql.auto.conf

```text
$PGDATA/postgresql.auto.conf
```

This file stores configuration values written by:

```sql
ALTER SYSTEM
```

Example:

```sql
ALTER SYSTEM SET max_connections = 200;
```

The setting is written to:

```text
postgresql.auto.conf
```

Depending on the parameter, perform either a reload or restart before the new value becomes active.

---

# 49. IMPORTANT CORRECTION – Configuration Priority

The original statement:

> `postgresql.conf is always high priority`

is **incorrect**.

For settings loaded from these two files:

```text
postgresql.auto.conf
```

takes precedence over:

```text
postgresql.conf
```

Example:

`postgresql.conf`:

```text
max_connections = 150
```

`postgresql.auto.conf`:

```text
max_connections = 200
```

The effective configuration from these files will be:

```text
max_connections = 200
```

because `postgresql.auto.conf` is processed after `postgresql.conf`.

This is particularly important when troubleshooting why a manually changed value in `postgresql.conf` does not become effective.

---

# 50. ALTER SYSTEM

Syntax:

```sql
ALTER SYSTEM SET parameter = value;
```

Example:

```sql
ALTER SYSTEM SET max_connections = 200;
```

Check:

```sql
SHOW max_connections;
```

For a parameter requiring restart:

```bash
systemctl restart postgresql-18
```

To remove the `ALTER SYSTEM` setting:

```sql
ALTER SYSTEM RESET max_connections;
```

To remove all settings written through `ALTER SYSTEM`:

```sql
ALTER SYSTEM RESET ALL;
```

Use these commands carefully on production databases.

---

# 51. How to Check Where a Parameter Came From

A very useful DBA query is:

```sql
SELECT
    name,
    setting,
    source,
    sourcefile,
    sourceline,
    pending_restart
FROM pg_settings
WHERE name = 'max_connections';
```

This can help identify whether a parameter came from:

* Default configuration
* Configuration file
* Command line
* Environment
* Database setting
* User setting
* Session setting
* Other PostgreSQL configuration sources

---

# 52. pg_file_settings

PostgreSQL provides the `pg_file_settings` view to inspect configuration-file contents and detect configuration problems.

Example:

```sql
SELECT
    sourcefile,
    sourceline,
    name,
    setting,
    applied,
    error
FROM pg_file_settings;
```

This is very useful after editing configuration files.

For example, it can help identify:

* Invalid values
* Syntax problems
* Duplicate settings
* Settings that were overridden

---

# 53. postmaster.opts

```text
$PGDATA/postmaster.opts
```

Records the command-line options with which the PostgreSQL server was last started.

Example:

```bash
cat postmaster.opts
```

Possible output:

```text
/usr/pgsql-18/bin/postgres "-D" "/var/lib/pgsql/18/data/"
```

This is useful when troubleshooting how the current PostgreSQL instance was started.

---

# 54. postmaster.pid

```text
$PGDATA/postmaster.pid
```

This file exists while the PostgreSQL server is running and contains information about the running server instance.

It includes information such as:

* PostgreSQL server PID
* Data directory
* Server start time
* Port
* Unix-domain socket directory
* Listen address information
* Server status information

Example:

```bash
cat postmaster.pid
```

### Important

Do **not** manually delete `postmaster.pid` simply because PostgreSQL fails to start.

First verify whether a PostgreSQL process is actually running.

Example:

```bash
ps -ef | grep postgres
```

or:

```bash
pg_ctl -D /var/lib/pgsql/18/data status
```

Deleting a valid PID file while PostgreSQL is running can cause serious administrative problems.

---

# 55. Useful PostgreSQL Verification Commands

## PostgreSQL version

```bash
/usr/pgsql-18/bin/postgres --version
```

or:

```bash
psql --version
```

From SQL:

```sql
SELECT version();
```

---

## Data directory

```sql
SHOW data_directory;
```

---

## Configuration file

```sql
SHOW config_file;
```

---

## HBA file

```sql
SHOW hba_file;
```

---

## Ident file

```sql
SHOW ident_file;
```

---

## Port

```sql
SHOW port;
```

Default PostgreSQL port:

```text
5432
```

---

## Listen addresses

```sql
SHOW listen_addresses;
```

---

## Server uptime

```sql
SELECT now() - pg_postmaster_start_time();
```

---

## Server start time

```sql
SELECT pg_postmaster_start_time();
```

---

## Current database

```sql
SELECT current_database();
```

---

## Current PostgreSQL user

```sql
SELECT current_user;
```

---

# 56. Useful Linux-Level DBA Checks

Check service:

```bash
systemctl status postgresql-18
```

Check processes:

```bash
ps -ef | grep '[p]ostgres'
```

Check listening port:

```bash
ss -lntp | grep 5432
```

Check disk usage:

```bash
df -h
```

Check PGDATA size:

```bash
du -sh /var/lib/pgsql/18/data
```

Check WAL size:

```bash
du -sh /var/lib/pgsql/18/data/pg_wal
```

Check logs:

```bash
ls -lh /var/lib/pgsql/18/data/log/
```

The actual log directory should always be confirmed from PostgreSQL configuration.

---

# 57. Important DBA Safety Rules

### Rule 1 – Never manually edit database files

Do not modify files inside directories such as:

```text
base
global
pg_wal
pg_xact
pg_multixact
```

Use PostgreSQL SQL commands and supported utilities.

### Rule 2 – Never delete WAL files manually

Do not run commands such as:

```bash
rm -f $PGDATA/pg_wal/*
```

PostgreSQL manages WAL itself.

Manual deletion can make the database cluster unrecoverable.

### Rule 3 – Be careful with replication slots

Inactive replication slots can retain WAL and consume disk space.

Monitor:

```sql
SELECT * FROM pg_replication_slots;
```

### Rule 4 – Back up configuration files

Before significant configuration changes, preserve:

```text
postgresql.conf
postgresql.auto.conf
pg_hba.conf
pg_ident.conf
```

### Rule 5 – Know whether a parameter requires reload or restart

Check:

```sql
SELECT name, context, pending_restart
FROM pg_settings
WHERE name = 'parameter_name';
```

---

# 58. Corrected Data Directory Quick Reference

| File / Directory       | Correct Purpose                                |
| ---------------------- | ---------------------------------------------- |
| `PG_VERSION`           | PostgreSQL major version                       |
| `base`                 | Per-database directories                       |
| `global`               | Cluster-wide system catalog data               |
| `current_logfiles`     | Current files written by logging collector     |
| `pg_commit_ts`         | Transaction commit timestamp data              |
| `pg_dynshmem`          | Dynamic shared memory files                    |
| `pg_logical`           | Logical decoding status data                   |
| `pg_multixact`         | Multitransaction status data                   |
| `pg_notify`            | LISTEN/NOTIFY status data                      |
| `pg_replslot`          | Replication slot state                         |
| `pg_serial`            | Committed serializable transaction information |
| `pg_snapshots`         | Exported snapshots                             |
| `pg_stat`              | Permanent statistics subsystem files           |
| `pg_stat_tmp`          | Temporary statistics subsystem files           |
| `pg_subtrans`          | Subtransaction status data                     |
| `pg_tblspc`            | Symbolic links to tablespaces                  |
| `pg_twophase`          | Prepared transaction state files               |
| `pg_wal`               | Write-Ahead Log files                          |
| `pg_xact`              | Transaction commit status data                 |
| `postgresql.conf`      | Main PostgreSQL configuration                  |
| `postgresql.auto.conf` | Settings written by `ALTER SYSTEM`             |
| `pg_hba.conf`          | Client authentication rules                    |
| `pg_ident.conf`        | User-name mappings                             |
| `postmaster.opts`      | Last server startup command-line options       |
| `postmaster.pid`       | Information about the running server           |

---

# 59. Important Corrections From the Original Class Notes

The following corrections are especially important for interviews and real production work.

### Incorrect

```text
postgresql.conf always has higher priority.
```

### Correct

For these two configuration files:

```text
postgresql.auto.conf
```

overrides corresponding settings in:

```text
postgresql.conf
```

---

### Incorrect

```text
pg_xact contains query runtime details.
```

### Correct

```text
pg_xact contains transaction commit status data.
```

---

### Incorrect

```text
pg_notify stores errors, warnings and notification messages.
```

### Correct

```text
pg_notify stores LISTEN/NOTIFY status data.
```

---

### Incorrect

```text
pg_snapshots contains crash-analysis snapshots.
```

### Correct

```text
pg_snapshots contains exported snapshots.
```

---

### Incorrect

```text
current_logfiles contains the currently available audit logs.
```

### Correct

```text
current_logfiles records the log files currently being written
by PostgreSQL's logging collector.
```

---

### Incorrect

```text
pg_ident.conf is specifically for LDAP source/target users.
```

### Correct

```text
pg_ident.conf contains PostgreSQL user-name maps used by
authentication methods that support identity mapping.
```

---

### Incomplete

```text
Replication slots completely eliminate replication failure.
```

### Correct

Replication slots help prevent required WAL from being removed too early, but they do **not** guarantee that replication can never fail.

Replication can still fail because of:

* Network problems
* Configuration errors
* Authentication problems
* Disk failure
* Corruption
* Timeline problems
* Application/operational mistakes
* Replication slot problems

An inactive slot can also retain large amounts of WAL and fill the primary server's disk.

---

# 60. End-to-End PostgreSQL 18 Installation Flow

The complete class flow can be remembered as:

```text
1. Configure PGDG repository
          |
          v
2. Install PostgreSQL packages
          |
          v
3. Initialize database cluster
          |
          v
4. Verify PGDATA
          |
          v
5. Start PostgreSQL
          |
          v
6. Enable automatic startup
          |
          v
7. Verify processes and port
          |
          v
8. Connect using psql
          |
          v
9. Configure PostgreSQL
          |
          v
10. Configure authentication
```

Example command sequence:

```bash
# Install repository
dnf install -y <PGDG-repository-RPM>

# Install PostgreSQL
dnf install -y postgresql18-server postgresql18-contrib

# Initialize cluster
/usr/pgsql-18/bin/postgresql-18-setup initdb

# Start PostgreSQL
systemctl start postgresql-18

# Enable startup after reboot
systemctl enable postgresql-18

# Verify
systemctl status postgresql-18

# Check processes
ps -ef | grep '[p]ostgres'

# Check port
ss -lntp | grep 5432
```

---

# 61. Interview / Revision Points

Remember these points clearly:

**Repository**
→ Source from which Linux packages are downloaded and installed.

**RPM**
→ Package format/package-management technology used on Red Hat-family Linux.

**PGDG**
→ PostgreSQL Global Development Group repository.

**PGDATA**
→ PostgreSQL database cluster's data directory.

**initdb**
→ Initializes a new PostgreSQL database cluster.

**postgres**
→ PostgreSQL server executable.

**psql**
→ PostgreSQL command-line client.

**pg_ctl**
→ PostgreSQL native server-control utility.

**systemctl**
→ Linux systemd service-management command.

**base**
→ Per-database directories.

**global**
→ Cluster-wide catalog data.

**pg_wal**
→ Write-Ahead Log files.

**pg_xact**
→ Transaction commit status.

**pg_replslot**
→ Replication slot state.

**pg_tblspc**
→ Links to user-created tablespaces.

**pg_hba.conf**
→ Client authentication configuration.

**pg_ident.conf**
→ User-name mappings.

**postgresql.conf**
→ Main PostgreSQL configuration file.

**postgresql.auto.conf**
→ Configuration written by `ALTER SYSTEM`; overrides matching settings from `postgresql.conf`.

**postmaster.pid**
→ Information about the currently running PostgreSQL server.

**postmaster.opts**
→ Command-line options used to start the server.

**Default PostgreSQL port**
→ `5432`

**Default WAL segment size**
→ `16 MB`

---

# 62. Final DBA Summary

For a PostgreSQL DBA, understanding the installation and `PGDATA` structure is fundamental.

The most important sequence is:

```text
Software Installation
        ↓
Repository / RPM Packages
        ↓
initdb
        ↓
PGDATA
        ↓
PostgreSQL Service
        ↓
Configuration
        ↓
Authentication
        ↓
Database Operation
```

A DBA should know not only **which file exists**, but also:

1. What the file/directory is used for
2. Whether PostgreSQL manages it automatically
3. Whether it is safe to edit manually
4. Whether changes require reload or restart
5. What happens if the file or directory is damaged
6. How it affects backup, recovery and replication

That understanding is much more useful in real production support and PostgreSQL interviews than simply memorizing directory names.

The biggest technical correction is the configuration priority: 

PostgreSQL 18 officially states that `postgresql.auto.conf` is intended for settings written by `ALTER SYSTEM`, and its settings **override matching settings in `postgresql.conf`**. ([PostgreSQL][1])

I also aligned the `PGDATA` descriptions with PostgreSQL's official storage layout. 

The official documentation defines `base` as per-database directories, `pg_notify` as LISTEN/NOTIFY status, `pg_snapshots` as exported snapshots, `pg_replslot` as replication-slot data, and `pg_xact` as transaction commit-status data. ([PostgreSQL][2])

Official references used: 
[PostgreSQL 18 – Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html), 
[PostgreSQL 18 – Setting Parameters](https://www.postgresql.org/docs/18/config-setting.html), 
[PostgreSQL 18 – ALTER SYSTEM](https://www.postgresql.org/docs/18/sql-altersystem.html), and 
[PostgreSQL – Red Hat Linux Downloads](https://www.postgresql.org/download/linux/redhat/).

[1]: https://www.postgresql.org/docs/18/config-setting.html "PostgreSQL: Documentation: 18: 19.1. Setting Parameters"
[2]: https://www.postgresql.org/docs/18/storage-file-layout.html "PostgreSQL: Documentation: 18: 66.1. Database File Layout"

====================================================================================================================================

Absolutely. Here is a **short, easy-to-revise PostgreSQL 18 Class 10 note**, keeping the important DBA/interview points and removing repetition.

# PostgreSQL 18 – Class 10 Quick Notes

## 1. YUM / DNF / RPM / Repository

### Repository

A repository is a source from where Linux gets software packages.

```bash
/etc/yum.repos.d/
```

Check repositories:

```bash
ls -ltr /etc/yum.repos.d/
```

### RPM

RPM = **RPM Package Manager**

It manages `.rpm` software packages.

```bash
rpm -qa | grep postgresql
```

### DNF

`dnf` installs, removes and updates packages.

```bash
dnf search postgresql18
dnf install postgresql18-server -y
dnf remove postgresql18-server -y
```

---

# 2. PGDG Repository

PGDG = **PostgreSQL Global Development Group**

It provides PostgreSQL packages for Red Hat-based Linux systems.

For PostgreSQL 18:

```text
PostgreSQL : 18
Architecture: x86_64
```

Repository files are normally under:

```bash
/etc/yum.repos.d/
```

---

# 3. Install PostgreSQL 18

Install server + contrib:

```bash
dnf install postgresql18-server postgresql18-contrib -y
```

Verify:

```bash
rpm -qa | grep postgresql
```

Important packages:

| Package                | Purpose                  |
| ---------------------- | ------------------------ |
| `postgresql18`         | Client tools             |
| `postgresql18-libs`    | Libraries                |
| `postgresql18-server`  | Database server          |
| `postgresql18-contrib` | Additional modules/tools |

---

# 4. PostgreSQL Binary Location

PGDG PostgreSQL 18 binaries:

```text
/usr/pgsql-18/bin/
```

Important commands:

```text
postgres       → Database server
psql           → SQL client
pg_ctl         → Start/stop/reload/status
initdb         → Create database cluster
pg_dump        → Logical backup
pg_restore     → Restore backup
pg_basebackup  → Physical backup
pg_upgrade     → Major-version upgrade
pg_isready     → Check server availability
```

---

# 5. Initialize PostgreSQL

Installing PostgreSQL does **not** automatically create the database cluster.

Initialize:

```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
```

Default PGDATA:

```text
/var/lib/pgsql/18/data/
```

Think of it simply:

```text
Install software
      ↓
initdb
      ↓
Create PostgreSQL cluster
      ↓
Start PostgreSQL
```

---

# 6. PostgreSQL Database Cluster

A PostgreSQL cluster is a collection of databases managed by one PostgreSQL server instance.

It contains:

```text
Databases
Roles
Configuration
WAL
Transaction information
System catalogs
```

The cluster is stored primarily inside **PGDATA**.

---

# 7. PGDATA

For our PostgreSQL 18 installation:

```text
/var/lib/pgsql/18/data/
```

Check:

```bash
ls -ltr /var/lib/pgsql/18/data/
```

SQL:

```sql
SHOW data_directory;
```

---

# 8. Important PGDATA Files/Directories

| Item                   | Easy explanation             |
| ---------------------- | ---------------------------- |
| `PG_VERSION`           | PostgreSQL major version     |
| `base`                 | Database files/directories   |
| `global`               | Cluster-wide catalog data    |
| `pg_wal`               | WAL files                    |
| `pg_xact`              | Transaction commit status    |
| `pg_replslot`          | Replication slot information |
| `pg_tblspc`            | Tablespace symbolic links    |
| `pg_subtrans`          | Subtransaction information   |
| `pg_twophase`          | Prepared transaction state   |
| `pg_notify`            | LISTEN/NOTIFY state          |
| `pg_logical`           | Logical decoding information |
| `pg_snapshots`         | Exported snapshots           |
| `pg_stat`              | Statistics files             |
| `pg_stat_tmp`          | Temporary statistics files   |
| `pg_multixact`         | Multitransaction information |
| `pg_commit_ts`         | Commit timestamp data        |
| `pg_dynshmem`          | Dynamic shared-memory files  |
| `pg_hba.conf`          | Client authentication        |
| `pg_ident.conf`        | Username mappings            |
| `postgresql.conf`      | Main configuration           |
| `postgresql.auto.conf` | `ALTER SYSTEM` settings      |
| `postmaster.pid`       | Running server information   |
| `postmaster.opts`      | Server startup options       |

---

# 9. Most Important Directories

### `base`

Contains per-database directories.

```text
base/
 ├── 1
 ├── 4
 ├── 5
 └── ...
```

Numbers are database OIDs.

Check:

```sql
SELECT oid, datname
FROM pg_database;
```

### `global`

Contains cluster-wide system catalog information.

### `pg_wal`

Contains **Write-Ahead Log**.

Used for:

```text
Crash recovery
Replication
PITR
Durability
Logical decoding
```

Default WAL segment size:

```text
16 MB
```

But it can be different if configured during `initdb`.

Check:

```bash
pg_controldata $PGDATA | grep "Bytes per WAL segment"
```

---

# 10. PostgreSQL Configuration

### `postgresql.conf`

Main PostgreSQL configuration.

Examples:

```text
port
listen_addresses
max_connections
shared_buffers
work_mem
wal_level
logging
autovacuum
```

### `postgresql.auto.conf`

Created/modified by:

```sql
ALTER SYSTEM
```

Example:

```sql
ALTER SYSTEM SET max_connections = 200;
```

**Important:**

```text
postgresql.auto.conf
        ↓
overrides
        ↓
postgresql.conf
```

This is an important interview point.

---

# 11. Check Configuration Source

Very useful DBA query:

```sql
SELECT
    name,
    setting,
    source,
    sourcefile,
    sourceline,
    pending_restart
FROM pg_settings
WHERE name = 'max_connections';
```

Check configuration-file problems:

```sql
SELECT
    sourcefile,
    sourceline,
    name,
    setting,
    applied,
    error
FROM pg_file_settings;
```

---

# 12. `pg_hba.conf`

`pg_hba.conf` = **Host-Based Authentication**

Controls:

```text
Who can connect
Which database
Which user
From which network
Which authentication method
```

Example:

```text
host  all  all  192.168.1.0/24  scram-sha-256
```

After changing it:

```bash
systemctl reload postgresql-18
```

or:

```sql
SELECT pg_reload_conf();
```

Usually **no restart is required**.

---

# 13. `pg_ident.conf`

Used for **username/identity mapping**.

It is **not specifically an LDAP user mapping file**.

---

# 14. PostgreSQL Service

Check:

```bash
systemctl status postgresql-18
```

Start:

```bash
systemctl start postgresql-18
```

Stop:

```bash
systemctl stop postgresql-18
```

Restart:

```bash
systemctl restart postgresql-18
```

Reload:

```bash
systemctl reload postgresql-18
```

Enable automatic startup:

```bash
systemctl enable postgresql-18
```

Best:

```bash
systemctl enable --now postgresql-18
```

Verify:

```bash
systemctl is-enabled postgresql-18
```

Expected:

```text
enabled
```

---

# 15. Reload vs Restart

### Reload

Used when configuration can be changed without restarting PostgreSQL.

```bash
systemctl reload postgresql-18
```

### Restart

Completely restarts PostgreSQL.

```bash
systemctl restart postgresql-18
```

Existing connections are interrupted.

Check whether restart is required:

```sql
SELECT
    name,
    context,
    pending_restart
FROM pg_settings
WHERE name = 'max_connections';
```

---

# 16. `pg_ctl`

`pg_ctl` is PostgreSQL's native server-control utility.

**Important:** Don't run it as `root`.

Switch to:

```bash
su - postgres
```

Then:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data status
```

Expected:

```text
pg_ctl: server is running
```

If you're inside `psql`:

```text
postgres=#
```

don't run `pg_ctl` there.

Exit first:

```sql
\q
```

Then run `pg_ctl` from the Linux shell.

---

# 17. Useful DBA Checks

### PostgreSQL version

```bash
/usr/pgsql-18/bin/postgres --version
```

### Running processes

```bash
ps -ef | grep '[p]ostgres'
```

### Port

```bash
ss -lntp | grep 5432
```

### PGDATA size

```bash
du -sh /var/lib/pgsql/18/data
```

### WAL size

```bash
du -sh /var/lib/pgsql/18/data/pg_wal
```

### PostgreSQL readiness

```bash
/usr/pgsql-18/bin/pg_isready
```

---

# 18. Important SQL Commands

```sql
SELECT version();
```

```sql
SHOW data_directory;
```

```sql
SHOW config_file;
```

```sql
SHOW hba_file;
```

```sql
SHOW ident_file;
```

```sql
SHOW port;
```

```sql
SHOW listen_addresses;
```

```sql
SELECT current_database();
```

```sql
SELECT current_user;
```

Server start time:

```sql
SELECT pg_postmaster_start_time();
```

Uptime:

```sql
SELECT now() - pg_postmaster_start_time();
```

Reload configuration:

```sql
SELECT pg_reload_conf();
```

---

# 19. Replication Slots

Check:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

Important:

> Replication slots help retain required WAL, but they **do not guarantee replication success**.

An inactive slot can retain WAL and eventually consume disk space.

---

# 20. Safety Rules 🚨

### Never manually modify:

```text
base/
global/
pg_wal/
pg_xact/
pg_multixact/
```

### Never manually delete WAL:

```bash
rm -rf $PGDATA/pg_wal/*
```

### Never blindly delete:

```text
postmaster.pid
```

First check:

```bash
ps -ef | grep '[p]ostgres'
```

### Before major configuration changes

Back up:

```text
postgresql.conf
postgresql.auto.conf
pg_hba.conf
pg_ident.conf
```

---

# 21. Installation Flow — Memorize This

```text
PGDG Repository
       ↓
DNF Install
       ↓
PostgreSQL Packages
       ↓
initdb
       ↓
PGDATA
       ↓
Start Service
       ↓
Enable Startup
       ↓
Check Process
       ↓
Check Port
       ↓
Connect using psql
       ↓
Configure PostgreSQL
       ↓
Configure Authentication
```

### ⭐ 10 Things to Remember for DBA Interviews

```text
1. PGDG        → PostgreSQL repository
2. DNF         → Package manager
3. initdb      → Creates cluster
4. PGDATA      → Cluster data directory
5. postgres    → Database server
6. psql        → SQL client
7. pg_ctl      → Server control utility
8. pg_wal      → Write-Ahead Log
9. pg_hba.conf → Client authentication
10. auto.conf  → ALTER SYSTEM settings
```

And the **big configuration priority point**:

```text
postgresql.conf
       ↓
postgresql.auto.conf
       ↓
auto.conf wins for matching settings
```

This is the compact version I'd use as your **Class 10 revision sheet**.

