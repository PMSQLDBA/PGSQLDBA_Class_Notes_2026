# PostgreSQL 18 — Source Code Installation & DBA Prerequisites

**Class Date:** 13 August 2026
**Lab Version:** PostgreSQL 18.4
**Current 18.x Minor Release:** PostgreSQL 18.6
**Installation Method:** Source Code / Autoconf + Make
**Primary OS Context:** Linux

> **Version Note:** PostgreSQL 18.4 is retained as the historical lab example. For a fresh installation, normally use the latest supported PostgreSQL 18 minor release unless the project specifically requires 18.4.

---

# PART I — INSTALLATION PREREQUISITES

## 1. Capacity Planning

Before installing PostgreSQL, perform an operating-system and infrastructure-level capacity assessment.

### 1.1 Software Installation Storage

Check available space on `/usr`:

```bash
df -h /usr
```

> **Trainer Note:**
> The often-mentioned **100 MB minimum free space in `/usr`** is a lab assumption, not an official PostgreSQL requirement. Actual requirements depend on PostgreSQL version, installation prefix, optional components, compiler dependencies, and OS packages.

---

## 2. PostgreSQL Storage Planning

A PostgreSQL environment can conceptually be divided into:

| Storage Area | Example Capacity | Primary Purpose             |
| ------------ | ---------------: | --------------------------- |
| DATA         |           200 GB | PostgreSQL database cluster |
| PG_WAL       |           100 GB | WAL                         |
| PG_LOG       |            50 GB | PostgreSQL/server logs      |
| ARCHIVE      |           100 GB | Archived WAL                |
| BACKUP       |           200 GB | Local backup storage        |

Example logical layout:

```text
DATA
PG_WAL
PG_LOG
ARCHIVE
BACKUP
```

### 2.1 Why Separate Storage?

Separating workloads can reduce competition between:

* Database data I/O
* WAL I/O
* Log I/O
* Backup I/O
* Archive I/O

However:

> **DBA Note — Mount Point ≠ Physical Disk**

Five mount points do **not** necessarily mean five physical storage devices.

For example, multiple AWS mount points may ultimately depend on the same underlying storage architecture.

If actual I/O isolation is required, verify the underlying storage design.

---

# PART II — POSTGRESQL STORAGE COMPONENTS

## 3. DATA

The PostgreSQL data directory contains the database cluster.

Example:

```text
/var/lib/pgsql/data
```

It contains PostgreSQL cluster-level information such as:

```text
base/
global/
pg_wal/
pg_xact/
pg_tblspc/
postgresql.conf
pg_hba.conf
```

The exact contents can vary by PostgreSQL version and configuration.

---

## 4. PG_WAL

WAL means:

**Write-Ahead Log**

WAL is fundamental to:

* Crash recovery
* Durability
* Replication
* Point-in-Time Recovery
* WAL archiving

Normally:

```text
$PGDATA/pg_wal
```

### 4.1 WAL Sizing

Do **not** treat:

```text
PG_WAL = 100 GB
```

as a PostgreSQL requirement.

WAL consumption depends on:

* Transaction volume
* INSERT/UPDATE/DELETE activity
* Bulk operations
* Checkpoint behavior
* Replication
* Replication slots
* WAL retention
* Archiving delays
* Long-running transactions

> **DBA Rule:**
> Size WAL storage according to workload and retention requirements.

---

# PART III — ARCHIVING AND BACKUP

## 5. ARCHIVE

Archived WAL supports:

* Point-in-Time Recovery
* Disaster Recovery
* Long-term recovery requirements

Conceptually:

```text
PostgreSQL
    |
    +---- DATA
    |
    +---- PG_WAL
    |
    +---- ARCHIVE
              |
              +---- S3 / NAS / Object Storage
```

### 5.1 Production DBA Rule

Do not rely exclusively on a local archive filesystem.

A stronger architecture is:

```text
Local Archive
      |
      +---- S3
      |
      +---- NAS
      |
      +---- Remote Backup Server
```

---

## 6. BACKUP

Example:

```text
BACKUP = 200 GB
```

The amount and retention of backup storage should be determined by:

* RPO
* RTO
* Backup frequency
* Backup size
* Retention period
* Storage cost
* Compliance requirements
* Disaster recovery requirements

PostgreSQL broadly supports:

1. SQL dumps
2. File-system-level backups
3. Continuous archiving / PITR

### 6.1 Critical DBA Principle

```text
Backup
   ↓
Restore Test
   ↓
Validation
   ↓
Recovery Readiness
```

> **Remember:**
> A successful backup job does not automatically prove that recovery will succeed.

---

# PART IV — CPU, CONNECTIONS AND MEMORY

## 7. Connections vs Concurrency vs TPS

Suppose:

```text
Average concurrent connections = 8
```

These terms are different.

### Connections

Number of client sessions connected to PostgreSQL.

### Concurrency

Number of operations/sessions actively executing or waiting at the same time.

### TPS

Transactions Per Second.

Therefore:

```text
8 connections
```

does **not** mean:

```text
8 TPS
```

---

## 8. CPU Planning

Do not use:

```text
1 CPU = 4 connections
2 CPUs = 8 connections
```

as PostgreSQL sizing rules.

PostgreSQL normally uses a separate server process for each client connection, while the operating system schedules those processes across available CPUs.

CPU requirements depend on:

* SQL complexity
* Query plans
* Index usage
* Sorting
* Hash operations
* Parallel queries
* Active sessions
* Transaction rate
* Lock contention
* I/O latency
* Application workload

### 8.1 DBA Monitoring

Monitor:

```text
CPU utilization
Load average
TPS
Query latency
Active sessions
Wait events
I/O utilization
Lock waits
```

> **DBA Rule:**
> Never size CPU solely from connection count.

---

## 9. RAM / Memory Planning

The following can be used as **training/lab guidelines only**:

```text
2 CPUs  → 8 GB RAM
4 CPUs  → 16 GB RAM
```

They are **not official PostgreSQL sizing formulas**.

PostgreSQL memory consumption involves:

```text
shared_buffers
work_mem
maintenance_work_mem
temp_buffers
OS page cache
Backend/connection memory
Other PostgreSQL processes
```

### 9.1 Swap

Conceptually:

```text
Insufficient RAM
       ↓
Memory pressure
       ↓
Possible swapping
       ↓
Additional disk I/O
       ↓
Performance degradation
```

Excessive connections can also increase memory consumption.

> **DBA Note:**
> Connection pooling can be preferable to simply increasing `max_connections`.

---

# PART V — CHECK EXISTING POSTGRESQL

## 10. Check Whether PostgreSQL Is Running

```bash
ps -ef | grep postgres
```

Example:

```text
root  2082  2052  0 14:08 pts/3  00:00:00 grep --color=auto postgres
```

This example indicates that no PostgreSQL server process is visible.

---

## 11. Check Whether PostgreSQL Is Installed

Check the binaries:

```bash
which psql
which postgres
which pg_config
```

Check versions:

```bash
psql --version
postgres --version
pg_config --version
```

You can also search for `pg_config`:

```bash
find / -name pg_config 2>/dev/null
```

If PostgreSQL was installed through an RPM-based package manager, also check installed packages.

---

# PART VI — SOURCE CODE INSTALLATION

## 12. When Should Source Installation Be Used?

For normal production installations, binary packages are generally preferable when suitable packages are available.

Source installation is useful for:

* Custom build requirements
* Development
* Extension development
* Learning PostgreSQL internals
* Special compilation requirements

> **Training Note:**
> This lab uses source installation to understand the PostgreSQL installation process.

---

# PART VII — BUILD PREREQUISITES

## 13. Prepare the Working Directory

```bash
cd /var/lib/pgsql/
pwd
```

Expected:

```text
/var/lib/pgsql
```

---

## 14. Install Build Dependencies

Example:

```bash
yum install gcc wget perl zlib-devel bison icu libicu libicu-devel flex readline-devel -y
```

> **DBA Note:**
> Package names vary by Linux distribution and version.

PostgreSQL 18 requires, among other things:

* GNU make
* C compiler
* tar/gzip or equivalent archive tools
* Flex
* Bison
* Perl
* Readline
* zlib
* ICU

Optional features may require additional libraries such as:

* OpenSSL
* LZ4
* Zstandard
* Kerberos
* LDAP
* PAM

---

# PART VIII — DOWNLOAD AND EXTRACT SOURCE

## 15. Download PostgreSQL

Historical lab example:

```bash
wget https://ftp.postgresql.org/pub/source/v18.4/postgresql-18.4.tar.gz
```

Verify:

```bash
ls -ltr
```

> **Version Note:**
> For a new installation, normally use the latest supported minor release. As of **13 August 2026**, PostgreSQL **18.6** is the current 18.x minor release.

---

## 16. Extract Source

```bash
tar -xf postgresql-18.4.tar.gz
```

Verify:

```bash
ls -ltr
```

Move into the source directory:

```bash
cd postgresql-18.4
pwd
```

Expected:

```text
/var/lib/pgsql/postgresql-18.4
```

---

# PART IX — UNDERSTAND THE SOURCE TREE

## 17. Important Source Files and Directories

Typical contents include:

```text
src/
contrib/
doc/
configure
configure.ac
meson.build
Makefile
README.md
```

The PostgreSQL server source is primarily written in C.

---

# PART X — BUILD POSTGRESQL

## 18. Traditional Build Flow

The traditional Autoconf/Make process is:

```text
./configure
      ↓
make
      ↓
make install
```

PostgreSQL also supports the Meson build system, but this lab uses Autoconf/Make.

---

## 19. Run configure

```bash
./configure
```

### What does configure do?

Conceptually:

```text
configure
    ↓
Check OS
    ↓
Check compiler
    ↓
Check libraries
    ↓
Check available features
    ↓
Generate build configuration
```

Immediately verify the exit status:

```bash
echo $?
```

Result:

```text
0
```

means success.

Non-zero means failure.

> **Important:**
> `echo $?` immediately after `./configure` is the standard simple way to check the command's exit status.

---

## 20. Run make

```bash
make
```

Conceptually:

```text
configure
    =
Prepare build environment

make
    =
Compile PostgreSQL

make install
    =
Install PostgreSQL
```

---

## 21. Run make install

```bash
make install
```

Unless another prefix was supplied, the traditional source installation defaults to:

```text
/usr/local/pgsql
```

---

# PART XI — CONTRIB MODULES

## 22. Build and Install contrib

Move into:

```bash
cd contrib/
```

Verify:

```bash
pwd
```

Build:

```bash
make
```

Install:

```bash
make install
```

### Important Concept

`contrib` contains additional supplied modules, extensions, and utilities.

It is not simply another mandatory component of the PostgreSQL core server.

### Extension Installation vs Extension Creation

These are different operations:

```text
Build/install extension files
            ↓
CREATE EXTENSION
            ↓
Extension becomes available in a specific database
```

Example:

```sql
CREATE EXTENSION extension_name;
```

The SQL command is executed inside the database where the extension is required.

---

# PART XII — INSTALLATION VALIDATION

## 23. Validate PostgreSQL Binaries

```bash
cd /usr/local/pgsql/bin/
ls -ltr
```

Important executables include:

| Utility          | Purpose                     |
| ---------------- | --------------------------- |
| `postgres`       | PostgreSQL server           |
| `psql`           | Command-line client         |
| `pg_ctl`         | Server control              |
| `initdb`         | Initialize cluster          |
| `pg_dump`        | Logical backup              |
| `pg_restore`     | Restore dump archive        |
| `pg_basebackup`  | Physical base backup        |
| `pg_upgrade`     | Major-version upgrade       |
| `pg_isready`     | Connection readiness        |
| `pg_config`      | Build/install information   |
| `pg_controldata` | Cluster control information |
| `pg_checksums`   | Checksum management         |
| `pg_waldump`     | Inspect WAL                 |
| `pgbench`        | Benchmarking                |

---

## 24. Validate pg_config

```bash
./pg_config
```

Important information includes:

```text
BINDIR
LIBDIR
INCLUDEDIR
SHAREDIR
VERSION
```

Example:

```text
BINDIR = /usr/local/pgsql/bin
LIBDIR = /usr/local/pgsql/lib
INCLUDEDIR = /usr/local/pgsql/include
SHAREDIR = /usr/local/pgsql/share
VERSION = PostgreSQL 18.4
```

`pg_config` is especially useful when building PostgreSQL extensions.

---

# PART XIII — CREATE AND PREPARE PGDATA

## 25. Create Data Directory

The PostgreSQL binaries are installed, but the cluster does not yet exist.

Create the data directory:

```bash
mkdir -p /var/lib/pgsql/data
```

---

## 26. Set PostgreSQL Ownership

Create/use the dedicated OS account:

```text
postgres
```

Set ownership:

```bash
chown -R postgres:postgres /usr/local/pgsql/
chown -R postgres:postgres /var/lib/pgsql/data/
```

> **Security Rule:**
> Never run the PostgreSQL server as `root`.

---

# PART XIV — INITIALIZE DATABASE CLUSTER

## 27. Switch to postgres User

```bash
su - postgres
```

Verify:

```bash
whoami
```

Expected:

```text
postgres
```

---

## 28. Run initdb

```bash
/usr/local/pgsql/bin/initdb -D /var/lib/pgsql/data/
```

### What does initdb do?

It creates a PostgreSQL database cluster containing items such as:

```text
System catalogs
postgresql.conf
pg_hba.conf
pg_ident.conf
PG_VERSION
pg_wal
base
global
Other cluster directories
```

> **Important Concept:**
> `make install` installs PostgreSQL software.
> `initdb` creates the PostgreSQL database cluster.

---

# PART XV — AUTHENTICATION

## 29. Review initdb Authentication

You may see:

```text
initdb: warning: enabling "trust" authentication for local connections
```

This must not be ignored in production.

`trust` authentication can allow matching local connections without password authentication.

Review:

```text
$PGDATA/pg_hba.conf
```

and configure authentication according to organizational security requirements.

---

# PART XVI — VERIFY PGDATA

## 30. Inspect the Data Directory

```bash
cd /var/lib/pgsql/data/
ls -ltr
```

You may see:

```text
base/
global/
pg_wal/
pg_xact/
pg_multixact/
pg_logical/
pg_replslot/
pg_stat/
pg_stat_tmp/
pg_tblspc/
pg_twophase/
postgresql.conf
postgresql.auto.conf
pg_hba.conf
pg_ident.conf
PG_VERSION
```

The exact contents vary by PostgreSQL version and configuration.

---

# PART XVII — SERVER CONTROL

## 31. Check Status

```bash
pg_ctl -D /var/lib/pgsql/data/ status
```

If stopped:

```text
pg_ctl: no server running
```

---

## 32. Start PostgreSQL

```bash
pg_ctl -D /var/lib/pgsql/data/ start
```

Expected:

```text
server started
```

---

## 33. Check Status Again

```bash
pg_ctl -D /var/lib/pgsql/data/ status
```

Example:

```text
pg_ctl: server is running (PID: 7961)
/usr/local/pgsql/bin/postgres "-D" "/var/lib/pgsql/data"
```

---

## 34. Stop PostgreSQL

```bash
pg_ctl -D /var/lib/pgsql/data/ stop
```

Default shutdown mode:

```text
fast
```

PostgreSQL supports:

```text
smart
fast
immediate
```

### Shutdown Modes

**Smart**

Wait for clients to disconnect.

**Fast**

Prevent new connections, terminate active sessions, roll back active transactions, and perform a clean shutdown.

**Immediate**

Terminate server processes immediately.

> **DBA Rule:**
> Avoid `immediate` shutdown unless there is a specific reason. The next startup may require crash recovery.

---

## 35. Start Again

```bash
pg_ctl -D /var/lib/pgsql/data/ start
```

---

## 36. Restart

```bash
pg_ctl -D /var/lib/pgsql/data/ restart
```

Conceptually:

```text
restart
   ↓
stop
   ↓
start
```

A restart causes service interruption.

---

# PART XVIII — RELOAD

## 37. Configuration Reload

Reload rereads configuration without shutting down PostgreSQL.

```bash
pg_ctl -D $PGDATA reload
```

or:

```bash
pg_ctl reload
```

Conceptually:

```text
Configuration change
       ↓
pg_ctl reload
       ↓
Configuration reread
       ↓
No database shutdown
```

For example, after changing `pg_hba.conf`, a reload is normally required.

> **Important:**
> Not every configuration parameter supports reload. Some parameters require a restart.

---

# PART XIX — POSTGRESQL PROCESSES

## 38. View PostgreSQL Processes

```bash
ps -ef | grep postgres
```

PostgreSQL 18 can show processes such as:

```text
postgres
postgres: io worker
postgres: checkpointer
postgres: background writer
postgres: walwriter
postgres: autovacuum launcher
postgres: logical replication launcher
```

Client sessions also appear as PostgreSQL server processes.

> **Version Note:**
> Do not memorize one universal PostgreSQL process list. The exact processes depend on PostgreSQL version and configuration.

---

## 39. Important Background Processes

### Checkpointer

Performs checkpoints and helps flush dirty pages to durable storage.

### Background Writer

Writes dirty shared buffers in the background.

### WAL Writer

Writes WAL data from memory to WAL files.

### Autovacuum Launcher

Coordinates autovacuum workers.

### Logical Replication Launcher

Coordinates logical replication workers when configured.

### I/O Workers

PostgreSQL 18 includes I/O worker processes supporting asynchronous I/O processing.

---

# PART XX — ENVIRONMENT VARIABLES

## 40. PostgreSQL User Home

```bash
cd
pwd
```

Expected:

```text
/home/postgres
```

Display hidden files:

```bash
ls -la
```

Examples:

```text
.bash_profile
.bashrc
.bash_logout
```

---

## 41. Set PATH and PGDATA

Edit:

```bash
vi ~/.bash_profile
```

Add:

```bash
export PATH=$PATH:/usr/local/pgsql/bin
export PGDATA=/var/lib/pgsql/data
```

Reload:

```bash
source ~/.bash_profile
```

Verify:

```bash
echo $PATH
echo $PGDATA
```

Expected:

```text
/var/lib/pgsql/data
```

### Why PGDATA?

With `PGDATA`:

```bash
pg_ctl status
```

can be used instead of:

```bash
pg_ctl -D /var/lib/pgsql/data status
```

---

# PART XXI — AUTO-START AFTER REBOOT

## 42. Lab Method — rc.local

A manual source installation does not automatically become a Linux service.

For the lab, configure:

```text
/etc/rc.d/rc.local
```

Add:

```bash
su - postgres -c '/usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/data start'
```

Make executable:

```bash
chmod +x /etc/rc.d/rc.local
```

Reboot:

```bash
reboot
```

After reconnecting:

```bash
ps -ef | grep postgres
```

---

## 43. Production Method — systemd

For modern Linux production environments, prefer **systemd**.

Benefits include:

* Service dependency management
* Standard start/stop/restart control
* Status monitoring
* Logging integration
* Failure handling
* Restart policies

Typical administration:

```bash
systemctl start postgresql
systemctl stop postgresql
systemctl restart postgresql
systemctl reload postgresql
systemctl status postgresql
```

> **Classroom Rule**

```text
Lab
 ↓
rc.local → Understand the concept

Production
 ↓
systemd → Preferred approach
```

---

# PART XXII — UNINSTALLATION

## 44. Stop PostgreSQL First

Switch to PostgreSQL user:

```bash
su - postgres
```

Check:

```bash
pg_ctl status
```

Stop:

```bash
pg_ctl stop
```

Verify:

```bash
ps -ef | grep postgres
```

---

## 45. Clean Source Tree

Move to:

```bash
cd /var/lib/pgsql/postgresql-18.4
```

Run:

```bash
make clean
```

### Difference

`make clean` removes generated build artifacts while keeping the configured source tree.

For a more complete reset:

```bash
make distclean
```

---

## 46. Uninstall PostgreSQL

```bash
make uninstall
```

This removes files installed by the source installation.

It does **not** automatically remove separately created directories such as the database cluster.

For contrib:

```bash
cd contrib/
make clean
make uninstall
```

---

## 47. Remove Installation Directory

If no longer required:

```bash
rm -rf /usr/local/pgsql
```

> **CRITICAL:** Before using `rm -rf`, verify:

```bash
pwd
ls -ld /usr/local/pgsql
```

Never casually execute destructive commands on a production system.

---

## 48. Remove Data Directory — Destructive Operation

The source uninstall does **not** remove the database cluster.

Conceptually:

```text
make uninstall
       ↓
Remove PostgreSQL software
       ↓
Database cluster remains
```

Only if the entire lab is intentionally being destroyed:

```bash
rm -rf /var/lib/pgsql/data
```

> **CRITICAL PRODUCTION WARNING**
>
> This permanently destroys the PostgreSQL cluster.

Before doing this, verify:

```text
Correct server
Correct PGDATA
Correct environment
Correct host
Correct purpose
Valid backup
```

Never remove `$PGDATA` merely because PostgreSQL software is being uninstalled.

---

# PART XXIII — COMPLETE INSTALLATION FLOW

## 49. DBA Memory Map

```text
Linux Server
      |
      ↓
Capacity Planning
      |
      ↓
Storage / CPU / RAM Planning
      |
      ↓
Check Existing PostgreSQL
      |
      ↓
Install OS Build Dependencies
      |
      ↓
Download PostgreSQL Source
      |
      ↓
Extract Source
      |
      ↓
./configure
      |
      ↓
make
      |
      ↓
make install
      |
      ↓
Build/install contrib
      |
      ↓
Validate binaries
      |
      ↓
Create PGDATA
      |
      ↓
Set postgres Ownership
      |
      ↓
initdb
      |
      ↓
Review pg_hba.conf
      |
      ↓
Start PostgreSQL
      |
      ↓
Check Status
      |
      ↓
Configure PATH / PGDATA
      |
      ↓
Configure Auto-start
      |
      ↓
Reboot Test
      |
      ↓
Final Validation
```

---

# PART XXIV — IMPORTANT DBA CONCEPTS

## 50. Mount Point ≠ Physical Disk

```text
Mount point
    ≠
Physical storage device
```

Always verify the underlying storage architecture.

---

## 51. Connections ≠ TPS

```text
Connections = Sessions
TPS         = Transaction throughput
```

Eight connections do not mean eight TPS.

---

## 52. CPU Cores ≠ Connection Capacity

Do not use:

```text
1 CPU = 4 connections
```

as a PostgreSQL sizing formula.

Base sizing on workload and performance measurements.

---

## 53. RAM Is Workload Dependent

Do not blindly apply:

```text
2 CPU = 8 GB
4 CPU = 16 GB
```

to production.

Consider:

* Workload
* `shared_buffers`
* `work_mem`
* Connection count
* OS memory
* Query behavior
* Performance testing

---

## 54. WAL Is Critical

Conceptually:

```text
Transaction
     ↓
WAL
     ↓
Durable storage
     ↓
Data files
```

WAL is fundamental to crash recovery, replication, archiving, and PITR.

---

## 55. Backup Must Be Tested

```text
Backup completed
       ≠
Recovery guaranteed
```

A DBA should periodically perform restore/recovery testing.

---

## 56. RELOAD ≠ RESTART

```text
RELOAD
   ↓
Reread configuration
   ↓
No database shutdown
```

versus:

```text
RESTART
   ↓
Stop
   ↓
Start
   ↓
Service interruption
```

Whether a particular parameter requires reload or restart depends on its configuration context.

---

## 57. Software Installation ≠ Database Initialization

```text
make install
     ↓
PostgreSQL software installed
```

versus:

```text
initdb
     ↓
PostgreSQL database cluster created
```

---

## 58. Extension Installation ≠ CREATE EXTENSION

```text
Build/install extension files
          ↓
CREATE EXTENSION
          ↓
Extension enabled in database
```

These are separate operations.

---

# PART XXV — FINAL VALIDATION

## 59. Binary Validation

```bash
which postgres
which psql
which pg_ctl
which pg_config
```

---

## 60. Version Validation

```bash
postgres --version
psql --version
pg_ctl --version
pg_config --version
```

---

## 61. Environment Validation

```bash
echo $PGDATA
```

---

## 62. Server Validation

```bash
pg_ctl status
ps -ef | grep postgres
pg_isready
```

---

## 63. Database Validation

Connect:

```bash
psql
```

Inside PostgreSQL:

```sql
SELECT version();
```

Then:

```sql
SHOW data_directory;
```

---

# PART XXVI — FINAL CLASSROOM SUMMARY

## PostgreSQL Source Installation

```text
Prerequisites
      ↓
Capacity Planning
      ↓
Download
      ↓
Extract
      ↓
./configure
      ↓
make
      ↓
make install
      ↓
contrib
      ↓
initdb
      ↓
Configuration
      ↓
Start
      ↓
Validate
      ↓
Auto-start
      ↓
Reboot Test
```

## DBA Mindset

Always think in terms of:

```text
Capacity
   +
Performance
   +
Availability
   +
Security
   +
Backup
   +
Recovery
   +
Monitoring
```

### Core DBA Principles

> **Installation is only the beginning.**

A PostgreSQL DBA should understand:

* Why each component exists
* How PostgreSQL uses storage
* How WAL supports durability and recovery
* How memory and CPU affect performance
* How connections differ from workload throughput
* How authentication protects the server
* How configuration reload and restart differ
* How backups are restored
* How PITR works
* How PostgreSQL starts and stops
* How failures are handled
* How recovery is tested

---

# OFFICIAL REFERENCE SECTION

Use these as the authoritative documentation section for the class:

1. PostgreSQL 18 — Installation from Source Code
2. PostgreSQL 18 — Source Build with Autoconf and Make
3. PostgreSQL 18 — `pg_ctl`
4. PostgreSQL 18 — Database File Layout
5. PostgreSQL 18 — WAL Internals
6. PostgreSQL 18 — Continuous Archiving and PITR
7. PostgreSQL 18 — Backup and Restore
8. PostgreSQL 18 — Installation Requirements

This segregation also makes the material much more suitable as **DBA training notes**: prerequisites → capacity → storage → installation → initialization → administration → startup → uninstallation → validation → DBA concepts.
