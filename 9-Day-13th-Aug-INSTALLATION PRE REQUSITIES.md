As of 13 August 2026, PostgreSQL 18.6 is the current 18.x release, so the installation example using 18.4 is a valid historical lab example, but for a fresh installation you should normally use the latest supported minor release unless the project specifically requires 18.4. 

INSTALLATION PREREQUISITES
Class Date: 13 August 2026

Before installing PostgreSQL, we must first perform an operating-system and infrastructure-level capacity check.

1. Storage Planning

Make sure the /usr filesystem has sufficient free space for PostgreSQL software installation.

Trainer Note:
The requirement of “minimum 100 MB free space in /usr” is a lab-specific sizing assumption, not an official PostgreSQL requirement. The actual requirement depends on the PostgreSQL version, installation prefix, optional components, and operating-system packages.

For the PostgreSQL data directory, check whether the respective mount point has sufficient free storage.

Example Capacity Planning

Suppose the application is expected to generate approximately 100 GB of database data by the end of the year.

A possible initial storage plan is:

Mount Point	Suggested Capacity	Purpose
DATA	200 GB	PostgreSQL database data
PG_WAL	100 GB	PostgreSQL WAL
PG_LOG	50 GB	PostgreSQL server logs
ARCHIVE	100 GB	Archived WAL files
BACKUP	200 GB	Local backup storage

Therefore, we have five logical storage areas:

DATA
PG_WAL
PG_LOG
ARCHIVE
BACKUP

Why separate storage?

If all database activity uses the same storage device, database data, WAL, logs, and backup operations can compete for the same I/O resources.

Separating workloads onto different storage devices can help isolate I/O workloads and improve predictability.

However:

Important DBA Note:
Separate mount points do not automatically mean separate physical disks.

For example, on AWS, five mount points could still ultimately depend on the same underlying storage architecture. If true storage isolation is required, verify the actual EBS/storage design.

PostgreSQL's documentation specifically notes that putting pg_wal on a different disk from the main database files can be advantageous. 
P
PostgreSQL

Also remember:

One disk failure does not necessarily mean the entire database is lost if redundancy, replication, snapshots, or backups are properly configured.

The real protection comes from storage redundancy + backups + WAL archiving + recovery testing, not simply from creating multiple mount points.

2. DATA, WAL, ARCHIVE AND BACKUP Planning
DATA

The DATA filesystem stores the PostgreSQL cluster.

For example:

/var/lib/pgsql/data


This directory contains the database files, configuration files, transaction information, WAL directory, and other cluster-level files.

PostgreSQL's official file-layout documentation describes directories such as base, global, pg_wal, pg_xact, pg_tblspc, and others. 
P
PostgreSQL

PG_WAL

pg_wal contains PostgreSQL's Write-Ahead Log (WAL).

WAL is critical for:

Crash recovery
Data durability
Replication
Point-in-Time Recovery (PITR)
WAL archiving

WAL is normally stored inside:

$PGDATA/pg_wal


PostgreSQL documentation states that WAL is automatically enabled and that the administrator must ensure sufficient disk space for WAL. 
P
PostgreSQL

Important DBA Note

Do not think of PG_WAL = 100 GB as a fixed PostgreSQL requirement.

Actual WAL consumption depends on:

Transaction volume
INSERT/UPDATE/DELETE activity
Bulk operations
Checkpoint configuration
Replication
WAL retention
Replication slots
WAL archiving delays
Long-running transactions

Therefore, WAL storage should be sized based on workload and retention requirements.

3. ARCHIVE

Archived WAL is used for:

Point-in-Time Recovery
Disaster recovery
Long-term recovery requirements

PostgreSQL continuously maintains WAL in pg_wal. When WAL archiving is enabled, completed WAL segments can be copied to archive storage using archive_command or archive_library. 
P
PostgreSQL
+1

Example architecture:

PostgreSQL
    |
    +---- DATA
    |
    +---- PG_WAL
    |
    +---- ARCHIVE
             |
             +---- S3 / NAS / Object Storage

DBA Rule

Do not treat the local archive filesystem as the only copy of your recovery data.

For production environments, consider an off-server/off-site copy such as:

Local Archive
      |
      +---- S3
      |
      +---- NAS
      |
      +---- Remote Backup Server

4. BACKUP

A local backup filesystem can be used for operational recovery.

Example:

BACKUP = 200 GB


The statement:

“Keep only one backup locally and copy remaining backups to S3/NAS.”

is a backup-retention policy, not a PostgreSQL requirement.

The correct retention policy should be determined by:

RPO — Recovery Point Objective
RTO — Recovery Time Objective
Backup size
Backup frequency
Retention period
Storage cost
Compliance requirements
Disaster recovery requirements

PostgreSQL officially describes three major backup approaches:

SQL dump
File-system-level backup
Continuous archiving/PITR 
P
PostgreSQL
Important DBA Note

A backup is useful only if it can actually be restored.

Therefore:

Backup
   ↓
Restore Test
   ↓
Validation
   ↓
Recovery Readiness


Never assume that a successful backup job automatically means that recovery will succeed.

5. CONCURRENCY, CONNECTIONS AND TPS

Suppose the application currently has:

Average concurrent connections = 8


Three terms must be clearly understood:

Connections

Number of client sessions connected to PostgreSQL.

Concurrency

Number of operations/sessions actively executing or waiting at the same time.

TPS

Transactions Per Second.

For example:

8 concurrent connections


does not mean:

8 TPS


These are completely different measurements.

Important Correction to the CPU Explanation

The following statement is an oversimplification:

“1 CPU can handle 4 connections with nanoseconds gap.”

This should not be used as a PostgreSQL sizing rule.

PostgreSQL normally uses a separate server process for each client connection, and the operating system schedules those processes on available CPU cores. PostgreSQL also has background and worker processes. 
P
PostgreSQL

Therefore:

2 CPUs = 8 connections


is not a valid PostgreSQL sizing formula.

CPU requirements depend on:

SQL complexity
Query execution plans
Index usage
Sorting
Hash operations
Parallel queries
Number of active sessions
Transaction rate
Lock contention
I/O latency
Application workload
DBA Rule

Do not size CPU based only on the number of connections.

Instead, monitor:

CPU utilization
Load average
TPS
Query latency
Active sessions
Wait events
I/O utilization
Lock waits

6. RAM / MEMORY PLANNING

Memory requirements depend on the workload and PostgreSQL configuration.

The following:

2 CPUs  -> 8 GB RAM
4 CPUs  -> 16 GB RAM


can be used as a training/lab sizing guideline, but it is not an official PostgreSQL sizing formula.

PostgreSQL uses memory for several purposes, including:

shared_buffers
work_mem
maintenance_work_mem
temp_buffers
Operating-system page cache
Connection/backend overhead
Other PostgreSQL processes

The default shared_buffers value is typically 128 MB, but PostgreSQL documentation notes that significantly higher values are normally required for good performance. 
P
PostgreSQL

SWAP

If physical memory is insufficient, Linux may use swap.

This can introduce additional disk I/O and latency.

Therefore:

Insufficient RAM
       ↓
Memory pressure
       ↓
Possible swapping
       ↓
Additional disk I/O
       ↓
Performance degradation


PostgreSQL documentation also warns that excessive connections and memory-related settings can contribute to memory exhaustion; in some cases, reducing max_connections and using connection pooling is preferable. 
P
PostgreSQL

7. CHECK WHETHER POSTGRESQL IS ALREADY RUNNING

Before installing PostgreSQL, verify whether a PostgreSQL server is already running.

[root@mypg ~]# ps -ef | grep postgres


Example:

root        2082    2052  0 14:08 pts/3    00:00:00 grep --color=auto postgres


In this example, PostgreSQL is not running.

8. CHECK WHETHER POSTGRESQL IS ALREADY INSTALLED

Check whether pg_config already exists:

[root@mypg ~]# find / -name pg_config


If no result is returned, pg_config may not be present.

Better DBA Validation

Also check:

which psql
which postgres
which pg_config


and:

psql --version
postgres --version
pg_config --version


If PostgreSQL was installed using an RPM/package manager, also check the installed packages.

SOURCE CODE INSTALLATION
9. Important Installation Note

For normal production deployments, PostgreSQL documentation recommends using binary packages when they are available. Source installation is particularly useful when you need custom build options, development work, or extension development. 
P
PostgreSQL

In this training, we are learning source-code installation because it is important for DBA understanding.

STEP 1 — INSTALL BUILD PREREQUISITES

Move to the PostgreSQL working directory:

[root@mypg ~]# cd /var/lib/pgsql/


Verify:

[root@mypg pgsql]# pwd
/var/lib/pgsql


Install required development packages.

Example from the lab:

[root@mypg pgsql]# yum install gcc wget perl zlib-devel bison icu libicu libicu-devel flex readline-devel -y

DBA Note

The exact package names vary by Linux distribution.

PostgreSQL 18 requires, among other things:

GNU make
C compiler
tar + gzip/bzip2
Flex
Bison
Perl
Readline
zlib
ICU

Additional libraries such as OpenSSL, LZ4, Zstandard, Kerberos, LDAP, PAM, etc. are required when corresponding optional features are enabled. 
P
PostgreSQL

Always verify prerequisites against the PostgreSQL version and Linux distribution being used.

STEP 2 — DOWNLOAD POSTGRESQL SOURCE CODE

PostgreSQL source packages are available from the official PostgreSQL download area. 
P
PostgreSQL

Example:

[root@mypg pgsql]# wget https://ftp.postgresql.org/pub/source/v18.4/postgresql-18.4.tar.gz


Verify:

[root@mypg pgsql]# ls -ltr


Example:

-rw-r--r--. 1 root root 29477735 May 11 19:54 postgresql-18.4.tar.gz

DBA Note

For a new installation, use the latest supported minor release unless the application specifically requires another version.

For example, on 13 August 2026, PostgreSQL 18.6 is the current 18.x release. 
P
PostgreSQL

STEP 3 — EXTRACT THE SOURCE CODE
[root@mypg pgsql]# tar -xf postgresql-18.4.tar.gz


Verify:

[root@mypg pgsql]# ls -ltr


Expected:

drwxrwxr-x. 7 root root     4096 ... postgresql-18.4
-rw-r--r--. 1 root root 29477735 ... postgresql-18.4.tar.gz


Move into the source directory:

[root@mypg pgsql]# cd postgresql-18.4


Verify:

[root@mypg postgresql-18.4]# pwd
/var/lib/pgsql/postgresql-18.4

STEP 4 — UNDERSTAND THE SOURCE TREE

You will see directories/files such as:

src/
contrib/
doc/
configure
configure.ac
meson.build
Makefile
README.md


The source code is primarily written in C.

STEP 5 — CONFIGURE

The traditional source-build process is:

./configure
       ↓
make
       ↓
make install


PostgreSQL also supports a Meson build system, but the traditional Autoconf/Make process is being used in this lab. 
P
PostgreSQL
+1

Run:

[root@mypg postgresql-18.4]# ./configure

What does configure do?

configure examines the operating system, compiler, libraries, and available features and creates configuration/build files appropriate for the current system. 
P
PostgreSQL

Classroom Explanation
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


If configure completes successfully, check the shell exit status:

[root@mypg postgresql-18.4]# echo $?
0

Important Note

Checking config.status is not the standard way to determine whether configure succeeded.

Use:

echo $?


immediately after the command.

Exit status:

0      = success
non-zero = failure

STEP 6 — MAKE

Run:

[root@mypg postgresql-18.4]# make


The make command compiles/builds PostgreSQL according to the generated build configuration.

Classroom Explanation
configure
   =
Prepare the build environment

make
   =
Compile/build PostgreSQL

make install
   =
Install the compiled files


The official PostgreSQL documentation describes ./configure, make, and make install as the standard Autoconf/Make installation procedure. 
P
PostgreSQL

STEP 7 — MAKE INSTALL
[root@mypg postgresql-18.4]# make install


By default, the source installation uses:

/usr/local/pgsql


unless a different --prefix was supplied during configure. 
P
PostgreSQL

STEP 8 — INSTALL CONTRIB MODULES

PostgreSQL provides additional modules and extensions under the contrib directory.

Move to:

[root@mypg postgresql-18.4]# cd contrib/


Verify:

[root@mypg contrib]# pwd
/var/lib/pgsql/postgresql-18.4/contrib


Build:

[root@mypg contrib]# make


Install:

[root@mypg contrib]# make install

Important DBA Note

contrib is not simply another mandatory part of the PostgreSQL core server.

It contains additional supplied modules, extensions, and utilities.

The PostgreSQL documentation states that these optional components are not automatically built unless the appropriate world target is used. They can be built and installed from the contrib directory. 
P
PostgreSQL
+1

Also remember:

Install extension files
        ≠
Enable extension in database


After the supporting files are installed, an extension may need:

CREATE EXTENSION extension_name;


and this is done inside the required database. 
P
PostgreSQL

VALIDATION
9. Check PostgreSQL Binaries
[root@mypg contrib]# cd /usr/local/pgsql/bin/


Check:

[root@mypg bin]# ls -ltr


You should see PostgreSQL executables such as:

postgres
psql
initdb
pg_ctl
pg_config
pg_dump
pg_restore
pg_basebackup
pg_rewind
pg_upgrade
pg_isready
pg_controldata
pg_checksums
pg_waldump
pgbench
createdb
createuser
dropdb
dropuser
vacuumdb
reindexdb
clusterdb

Classroom Note

Each PostgreSQL utility has a specific purpose.

Examples:

postgres       → PostgreSQL server
psql            → PostgreSQL command-line client
pg_ctl          → Start/stop/restart/control server
initdb          → Initialize a database cluster
pg_dump         → Logical backup of a database
pg_restore      → Restore a pg_dump archive
pg_basebackup   → Physical base backup
pg_upgrade      → Major-version upgrade
pg_isready      → Check server connection readiness
pg_config       → Display PostgreSQL build/install information
pg_controldata  → Display cluster control information
pg_waldump      → Inspect WAL records

10. PG_CONFIG

Run:

[root@mypg bin]# ./pg_config


Important output includes:

BINDIR = /usr/local/pgsql/bin
LIBDIR = /usr/local/pgsql/lib
INCLUDEDIR = /usr/local/pgsql/include
SHAREDIR = /usr/local/pgsql/share
VERSION = PostgreSQL 18.4


pg_config is particularly useful when compiling PostgreSQL extensions because it tells build systems where the PostgreSQL headers, libraries, binaries, and supporting files are installed.

CREATE THE DATA DIRECTORY

PostgreSQL binaries are installed, but the database cluster has not yet been initialized.

Create the data directory:

[root@mypg bin]# mkdir -p /var/lib/pgsql/data

11. POSTGRES USER OWNERSHIP

PostgreSQL should run under a dedicated operating-system account, normally called:

postgres


Ensure the PostgreSQL installation and data directory have appropriate ownership.

Example:

[root@mypg bin]# chown -R postgres:postgres /usr/local/pgsql/
[root@mypg bin]# chown -R postgres:postgres /var/lib/pgsql/data/

DBA Security Note

Do not run the PostgreSQL server as root.

The PostgreSQL server must be run by an appropriate unprivileged operating-system user.

12. SWITCH TO POSTGRES USER
[root@mypg bin]# su - postgres


Verify:

[postgres@mypg ~]$ whoami
postgres

DATABASE CLUSTER INITIALIZATION
13. INITDB

PostgreSQL requires a database cluster before the server can start.

Run:

[postgres@mypg ~]$ /usr/local/pgsql/bin/initdb -D /var/lib/pgsql/data/

What does initdb do?

initdb creates a new PostgreSQL database cluster.

It creates:

System catalogs
Default configuration files
postgresql.conf
pg_hba.conf
pg_ident.conf
PG_VERSION
pg_wal
base
global
Other required cluster directories

PostgreSQL documentation defines a database cluster as the collection of databases managed by a single server instance. 
P
PostgreSQL

14. INITDB AUTHENTICATION WARNING

The lab output shows:

initdb: warning: enabling "trust" authentication for local connections

IMPORTANT SECURITY NOTE

Do not ignore this message in production.

trust authentication means that PostgreSQL can accept a connection without password authentication under the matching pg_hba.conf rule.

Review:

$PGDATA/pg_hba.conf


and configure authentication according to the organization's security requirements.

15. CHECK DATA DIRECTORY
[postgres@mypg ~]$ cd /var/lib/pgsql/data/


Then:

[postgres@mypg data]$ ls -ltr


You will see directories/files such as:

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


The exact list can vary slightly by PostgreSQL version and configuration.

START / STOP / RESTART / STATUS
16. CHECK SERVER STATUS
[postgres@mypg data]$ /usr/local/pgsql/bin/pg_ctl \
-D /var/lib/pgsql/data/ status


If the server is not running:

pg_ctl: no server running

17. START POSTGRESQL
[postgres@mypg data]$ /usr/local/pgsql/bin/pg_ctl \
-D /var/lib/pgsql/data/ start


Expected message:

server started


PostgreSQL documentation recommends using pg_ctl to start, stop, restart, reload, and check the status of a server. 
P
PostgreSQL

18. CHECK STATUS AGAIN
[postgres@mypg data]$ /usr/local/pgsql/bin/pg_ctl \
-D /var/lib/pgsql/data/ status


Example:

pg_ctl: server is running (PID: 7961)
/usr/local/pgsql/bin/postgres "-D" "/var/lib/pgsql/data"

19. STOP POSTGRESQL
[postgres@mypg data]$ /usr/local/pgsql/bin/pg_ctl \
-D /var/lib/pgsql/data/ stop


By default, pg_ctl stop uses fast shutdown.

Fast shutdown:

Prevents new connections
Rolls back active transactions
Disconnects clients
Performs a clean shutdown

PostgreSQL also supports:

smart
fast
immediate


shutdown modes. 
P
PostgreSQL

Important DBA Note

Avoid immediate shutdown unless you have a specific reason.

Immediate shutdown terminates server processes immediately and causes crash recovery during the next startup. 
P
PostgreSQL

20. START AGAIN
[postgres@mypg data]$ /usr/local/pgsql/bin/pg_ctl \
-D /var/lib/pgsql/data/ start

21. RESTART
[postgres@mypg data]$ /usr/local/pgsql/bin/pg_ctl \
-D /var/lib/pgsql/data/ restart


restart effectively performs a stop followed by a start and is useful when configuration changes require a server restart. 
P
PostgreSQL

22. RELOAD
Definition

RELOAD causes PostgreSQL to reread its configuration files without shutting down the database server.

Command:

pg_ctl reload


or:

pg_ctl -D $PGDATA reload


It sends a SIGHUP signal to the PostgreSQL server, causing configuration files such as postgresql.conf and pg_hba.conf to be reread. 
P
PostgreSQL

Example

If you modify:

pg_hba.conf


you normally need to reload PostgreSQL for the change to be reread.

Configuration change
       ↓
pg_ctl reload
       ↓
No database shutdown

Important Note

Not every parameter can be changed by reload.

Some parameters require:

Restart


Always check the parameter's context in PostgreSQL documentation.

23. POSTGRESQL SERVER PROCESSES

Check the processes:

[postgres@mypg data]$ ps -ef | grep postgres


Example:

postgres    8014       1  0 13:51 ? 00:00:00 /usr/local/pgsql/bin/postgres -D /var/lib/pgsql/data
postgres    8015    8014  0 13:51 ? 00:00:00 postgres: io worker 1
postgres    8016    8014  0 13:51 ? 00:00:00 postgres: io worker 0
postgres    8017    8014  0 13:51 ? 00:00:00 postgres: io worker 2
postgres    8018    8014  0 13:51 ? 00:00:00 postgres: checkpointer
postgres    8019    8014  0 13:51 ? 00:00:00 postgres: background writer
postgres    8021    8014  0 13:51 ? 00:00:00 postgres: walwriter
postgres    8022    8014  0 13:51 ? 00:00:00 postgres: autovacuum launcher
postgres    8023    8014  0 13:51 ? 00:00:00 postgres: logical replication launcher


The main PostgreSQL server process is the parent of these server/background processes.

Classroom Explanation
PostgreSQL Main Server
        |
        +---- IO workers
        +---- Checkpointer
        +---- Background writer
        +---- WAL writer
        +---- Autovacuum launcher
        +---- Logical replication launcher
        +---- Client backend processes


The exact process list depends on PostgreSQL version and configuration.

PostgreSQL 18 includes additional I/O worker processes, so do not memorize an old process list as universally applicable. PostgreSQL's monitoring documentation shows the primary server process and its background processes and notes that individual client connections also appear as PostgreSQL server processes. 
P
PostgreSQL

24. IMPORTANT SERVER PROCESSES
Checkpointer

Responsible for performing checkpoints.

Checkpoints flush dirty data pages to durable storage.

Background Writer

Helps write dirty shared buffers to disk in the background.

WAL Writer

Writes WAL data from shared memory to WAL files.

Autovacuum Launcher

Coordinates autovacuum workers.

Autovacuum is highly recommended and automates VACUUM and ANALYZE activity. 
P
PostgreSQL

Logical Replication Launcher

Coordinates logical replication workers when logical replication is configured.

I/O Workers

PostgreSQL 18 includes I/O worker processes that participate in asynchronous I/O processing.

DBA Note

The process list seen in ps depends on PostgreSQL version and configuration. Always understand the version-specific architecture instead of assuming that every server will show exactly the same processes.

ENVIRONMENT VARIABLES SETUP
25. POSTGRES USER HOME DIRECTORY
[postgres@mypg data]$ cd


Check:

[postgres@mypg ~]$ pwd
/home/postgres


Check hidden files:

[postgres@mypg ~]$ ls -la

Classroom Note
ls


does not normally display hidden files.

ls -la


displays hidden files such as:

.bash_profile
.bashrc
.bash_logout

26. SET PATH AND PGDATA

Edit:

[postgres@mypg ~]$ vi ~/.bash_profile


Add:

export PATH=$PATH:/usr/local/pgsql/bin
export PGDATA=/var/lib/pgsql/data


Reload the shell environment:

[postgres@mypg ~]$ source ~/.bash_profile


Verify:

[postgres@mypg ~]$ echo $PATH


and:

[postgres@mypg ~]$ echo $PGDATA


Expected:

/var/lib/pgsql/data

DBA Note

PGDATA allows PostgreSQL utilities such as pg_ctl to locate the cluster without specifying -D every time.

For example:

pg_ctl status


instead of:

pg_ctl -D /var/lib/pgsql/data status


The official pg_ctl documentation confirms that PGDATA is used as the default data-directory location when -D is not supplied. 
P
PostgreSQL

27. CHECK POSTGRESQL STATUS

Without specifying the full path:

[postgres@mypg ~]$ pg_ctl status


Example:

pg_ctl: server is running (PID: 8069)
/usr/local/pgsql/bin/postgres "-D" "/var/lib/pgsql/data"

POSTGRESQL AUTO START AFTER LINUX REBOOT
28. Problem

With a manual source installation, PostgreSQL does not automatically become a Linux service merely because make install was executed.

We must configure operating-system startup.

29. rc.local METHOD

In the lab, we use:

/etc/rc.d/rc.local


Edit:

[root@mypg ~]# vi /etc/rc.d/rc.local


Add:

su - postgres -c '/usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/data start'


Make it executable:

[root@mypg rc.d]# chmod +x /etc/rc.d/rc.local


Then reboot:

[root@mypg rc.d]# reboot


After reconnecting, verify:

[root@mypg ~]# ps -ef | grep postgres


If PostgreSQL starts automatically, you should see the PostgreSQL server processes.

PostgreSQL's documentation describes rc.local as one possible Linux startup approach and also provides a systemd-based approach. 
P
PostgreSQL

30. PRODUCTION DBA NOTE — SYSTEMD

For modern Linux production environments, systemd is generally preferable to relying on rc.local because it provides:

Service dependency management
Start/stop/restart control
Service status
Logging integration
Failure handling
Automatic restart policies
Standard Linux service management

PostgreSQL's documentation provides an example systemd service for source installations. 
P
PostgreSQL

Typical administration becomes:

systemctl start postgresql
systemctl stop postgresql
systemctl restart postgresql
systemctl reload postgresql
systemctl status postgresql

Classroom Rule

For the lab:

rc.local → Understand the concept


For a production Linux server:

systemd → Preferred service-management approach


Also ensure the PostgreSQL operating-system account is configured appropriately. PostgreSQL documentation discusses systemd and operating-system user considerations for source installations. 
P
PostgreSQL

UNINSTALLATION
31. STOP POSTGRESQL FIRST

Before uninstalling PostgreSQL binaries:

[root@mypg ~]# su - postgres


Check status:

[postgres@mypg ~]$ pg_ctl status


Stop PostgreSQL:

[postgres@mypg ~]$ pg_ctl stop


Verify:

ps -ef | grep postgres

32. CLEAN THE SOURCE TREE

Move to the PostgreSQL source directory:

[root@mypg ~]# cd /var/lib/pgsql/postgresql-18.4


Run:

[root@mypg postgresql-18.4]# make clean

Difference
make clean


removes generated build artifacts but keeps the configured source tree.

PostgreSQL documentation also provides:

make distclean


when you want to return the source tree closer to its original distribution state and reconfigure it. 
P
PostgreSQL

33. UNINSTALL POSTGRESQL SOFTWARE

Run:

[root@mypg postgresql-18.4]# make uninstall


PostgreSQL officially documents make uninstall as the method for undoing a source installation. It does not remove directories that were created separately. 
P
PostgreSQL

Then uninstall the contrib installation:

[root@mypg postgresql-18.4]# cd contrib/
[root@mypg contrib]# make clean
[root@mypg contrib]# make uninstall


Verify:

[root@mypg contrib]# cd /usr/local/pgsql/bin/
[root@mypg bin]# ls -ltr


The PostgreSQL binaries should no longer be present after a successful uninstall.

34. REMOVE THE INSTALLATION DIRECTORY

If the directory is no longer required:

[root@mypg ~]# rm -rf /usr/local/pgsql

VERY IMPORTANT

Do not execute rm -rf commands casually on a production server.

Always verify the directory first:

pwd
ls -ld /usr/local/pgsql


Then remove it only when you are certain that it contains the intended PostgreSQL installation.

35. DATA DIRECTORY

The source-code uninstall operation does not automatically remove the PostgreSQL data directory.

This is an important distinction:

make uninstall
       ↓
Remove installed PostgreSQL software
       ↓
Does NOT automatically remove database cluster


If the lab is being completely destroyed, the data directory can be removed separately:

[root@mypg ~]# rm -rf /var/lib/pgsql/data

CRITICAL PRODUCTION WARNING

This command permanently destroys the PostgreSQL cluster.

Before executing it, verify:

pg_ctl status


and confirm:

Correct server
Correct PGDATA
Correct environment
Correct host
Correct purpose
Valid backup


Never run:

rm -rf $PGDATA


or:

rm -rf /var/lib/pgsql/data


on production merely to “uninstall PostgreSQL.”

36. FINAL INSTALLATION FLOW — DBA MEMORY MAP

Remember the complete source installation process as:

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
make install in contrib
      |
      ↓
Validate pg_config / binaries
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
Configure Auto Start
      |
      ↓
Test Reboot
      |
      ↓
Validate PostgreSQL

37. IMPORTANT DBA CONCEPTS TO REMEMBER
Concept 1 — Mount Point ≠ Physical Disk
Mount point
    ≠
Physical storage device


Always verify the underlying storage architecture.

Concept 2 — Connections ≠ TPS
Connections = Sessions
TPS         = Transaction throughput


Eight connections do not mean eight TPS.

Concept 3 — CPU Cores ≠ Connection Capacity

Do not use:

1 CPU = 4 connections


as a PostgreSQL sizing rule.

Size CPU based on actual workload and performance measurements.

Concept 4 — RAM Is Workload Dependent

Do not blindly use:

2 CPU = 8 GB
4 CPU = 16 GB


as a production sizing formula.

Use workload characteristics, PostgreSQL memory settings, operating-system memory, connection count, and performance testing.

Concept 5 — WAL Is Critical
Transaction
    ↓
WAL
    ↓
Durable storage
    ↓
Data files


WAL is fundamental to PostgreSQL crash recovery and also enables continuous archiving and PITR. 
P
PostgreSQL
+1

Concept 6 — Backup Must Be Tested
Backup completed
       ≠
Recovery guaranteed


A DBA must periodically perform restore/recovery testing.

Concept 7 — RELOAD ≠ RESTART
RELOAD
   ↓
Reread configuration
   ↓
No database shutdown


while:

RESTART
   ↓
Stop
   ↓
Start
   ↓
Downtime


Whether a configuration parameter requires reload or restart depends on that parameter's context. pg_ctl reload specifically rereads the PostgreSQL configuration files. 
P
PostgreSQL

Concept 8 — Software Installation ≠ Database Initialization

These are two different activities:

make install
     ↓
PostgreSQL software installed


and:

initdb
     ↓
PostgreSQL database cluster created

Concept 9 — Extension Installation ≠ CREATE EXTENSION
make install
     ↓
Extension files installed


then:

CREATE EXTENSION extension_name;


must be executed in the database where the extension is required. 
P
PostgreSQL
+1

38. USEFUL DBA VALIDATION COMMANDS

After installation, always perform basic validation:

which postgres
which psql
which pg_ctl
which pg_config

postgres --version
psql --version
pg_ctl --version
pg_config --version

echo $PGDATA

pg_ctl status

ps -ef | grep postgres

pg_isready


Then connect:

psql


Inside PostgreSQL:

SELECT version();


and:

SHOW data_directory;

39. FINAL CLASSROOM SUMMARY
PostgreSQL Source Installation
Prerequisites
      ↓
Download
      ↓
Extract
      ↓
configure
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

DBA Mindset

Always think in terms of:

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


A PostgreSQL DBA should not only know how to install PostgreSQL, but also understand why each component is required, what happens internally, how storage and memory affect performance, and how the database can be recovered when something fails.

Official PostgreSQL References
PostgreSQL 18 — Installation from Source Code
PostgreSQL 18 — Source Build with Autoconf and Make
PostgreSQL 18 — pg_ctl
PostgreSQL 18 — Database File Layout
PostgreSQL 18 — WAL Internals
PostgreSQL 18 — Continuous Archiving and PITR
PostgreSQL 18 — Backup and Restore
PostgreSQL 18 — Installation Requirements
