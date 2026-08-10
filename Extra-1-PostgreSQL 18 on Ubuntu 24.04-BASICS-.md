# PostgreSQL 18 on Ubuntu 24.04

## A Beginner-Friendly Technical Article for Freshers

**Target audience:** PostgreSQL beginners, junior DBAs, Linux administrators, and developers starting with PostgreSQL.

**Version covered:** PostgreSQL 18
**Operating System:** Ubuntu 24.04
**Reference:** PostgreSQL 18 Official Documentation

This article is structured as a learning document rather than simply a list of commands. 
PostgreSQL's own documentation recommends that administrators first understand installation, server setup, configuration, users/databases, maintenance, backup/recovery, HA/replication, and monitoring. ([PostgreSQL][1])

---

# 1. What Is PostgreSQL?

PostgreSQL is an open-source **object-relational database management system (ORDBMS)**.

It provides:

* SQL-based data management
* Transactions
* ACID properties
* Concurrency control
* Indexes
* Views
* Functions and procedures
* Replication
* High-availability capabilities
* Backup and recovery
* Security and authentication
* Extensibility

For a beginner, the most important concept is:

> **PostgreSQL is a client/server database system.**

A client application connects to a running PostgreSQL server, sends SQL statements, and receives the results. 

PostgreSQL's official tutorial introduces these concepts beginning with installation, architectural fundamentals, databases, SQL, tables, queries, joins, transactions, and other core topics. ([PostgreSQL][2])

---

# 2. PostgreSQL Architecture — Start With the Basics

A simple way to visualize PostgreSQL is:

```text
                    CLIENT
                      │
                      │ SQL / PostgreSQL Protocol
                      ▼
              PostgreSQL SERVER
                 postgres
                      │
                      ▼
              DATABASE CLUSTER
                  18/main
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Database    Database    Database
          │           │           │
          ▼           ▼           ▼
        Tables      Tables      Tables
```

A beginner should understand the following terms:

| Term       | Meaning                                                |
| ---------- | ------------------------------------------------------ |
| PostgreSQL | Database management system                             |
| Server     | Running PostgreSQL `postgres` instance                 |
| Cluster    | Collection of databases managed by one server instance |
| Database   | Logical container for database objects                 |
| Schema     | Namespace inside a database                            |
| Table      | Stores rows and columns                                |
| Role       | PostgreSQL security identity                           |
| SQL        | Language used to interact with PostgreSQL              |

---

# 3. What Is a PostgreSQL Database Cluster?

This is one of the **most important concepts for a PostgreSQL DBA**.

A PostgreSQL **database cluster** is a collection of databases managed by a single running PostgreSQL server instance. PostgreSQL stores the cluster in a common data directory. ([PostgreSQL][3])

Do not confuse:

```text
Cluster ≠ Database
```

Instead:

```text
Cluster
│
├── Database 1
├── Database 2
├── Database 3
└── Database N
```

A newly initialized PostgreSQL cluster normally contains databases such as:

```text
postgres
template0
template1
```

as described in the PostgreSQL documentation. ([PostgreSQL][3])

---

# 4. PostgreSQL Engine vs Server vs Cluster

Freshers often confuse these three concepts.

## PostgreSQL Software

The PostgreSQL software contains executables, libraries, utilities, and supporting files.

For your Ubuntu 24.04 installation:

```text
/usr/lib/postgresql/18/bin/
```

contains PostgreSQL 18 binaries.

---

## PostgreSQL Server

The primary server executable is:

```text
postgres
```

PostgreSQL's official documentation defines `postgres` as the PostgreSQL database server. ([PostgreSQL][4])

For your server:

```text
/usr/lib/postgresql/18/bin/postgres
```

---

## PostgreSQL Cluster

Your PostgreSQL server manages:

```text
18/main
```

where:

```text
18   = PostgreSQL major version
main = cluster name
```

Therefore:

```text
PostgreSQL 18 Software
        │
        ▼
   postgres server
        │
        ▼
     18/main
        │
        ├── Database
        ├── Database
        └── Database
```

The official documentation states that one `postgres` instance manages exactly one database cluster. 

Multiple `postgres` instances can run on the same host if they use different data areas and communication ports. ([PostgreSQL][4])

---

# 5. Your PostgreSQL 18 Environment

Your Ubuntu server currently reports:

```text
Ver Cluster Port Status Owner    Data directory
18  main    5432 online postgres /var/lib/postgresql/18/main
```

This means:

| Property           | Value                         |
| ------------------ | ----------------------------- |
| PostgreSQL version | **18**                        |
| Cluster name       | **main**                      |
| Cluster            | **18/main**                   |
| Status             | **online**                    |
| Port               | **5432**                      |
| OS owner           | `postgres`                    |
| Data directory     | `/var/lib/postgresql/18/main` |

The command `pg_lsclusters` is particularly useful on Ubuntu/Debian because it comes from the Debian/Ubuntu PostgreSQL packaging ecosystem.

---

# 6. What Is the PostgreSQL `postgres` Process?

The PostgreSQL server process is:

```text
postgres
```

Your running process looks like:

```text
/usr/lib/postgresql/18/bin/postgres
```

and your command line showed:

```text
-D /var/lib/postgresql/18/main
```

This tells us:

> The PostgreSQL 18 server is managing the cluster located at `/var/lib/postgresql/18/main`.

The official PostgreSQL documentation explains that the server must know the location of the data area, commonly through `-D` or `PGDATA`. ([PostgreSQL][4])

---

# 7. Why Are There Multiple PostgreSQL Processes?

A fresher may run:

```bash
ps -ef | grep postgres
```

and see multiple processes.

For example:

```text
postgres
postgres: checkpointer
postgres: background writer
postgres: walwriter
postgres: autovacuum launcher
postgres: logical replication launcher
```

This does **not** mean multiple PostgreSQL installations.

They are processes associated with the same PostgreSQL server instance.

PostgreSQL uses multiple server processes to perform different responsibilities. 

Its documented architecture uses a supervisor/server process and backend processes for client connections, with shared memory and synchronization mechanisms for concurrent access. ([PostgreSQL][5])

Conceptually:

```text
                PostgreSQL Server
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Checkpointer   WAL Writer    Background Writer
        │              │              │
        ├──────────────┼──────────────┤
        │
   Autovacuum Launcher
        │
   Client Backend Processes
```

---

# 8. PostgreSQL Client/Server Connection

A typical connection looks like:

```text
Application
     │
     │ TCP/IP
     │ Port 5432
     ▼
PostgreSQL Server
     │
     ▼
Backend Process
     │
     ▼
SQL Parser
     │
     ▼
Planner / Optimizer
     │
     ▼
Executor
     │
     ▼
Result
     │
     ▼
Application
```

PostgreSQL uses a process-per-user/client model in which a client connection is handled by a backend server process. 

The backend parses the query, creates an execution plan, executes it, and returns the result. ([PostgreSQL][5])

This is the foundation for understanding PostgreSQL performance later.

---

# 9. PostgreSQL Data Directory

Your PostgreSQL cluster data directory is:

```text
/var/lib/postgresql/18/main
```

Think of this as:

> **The physical storage area for the PostgreSQL cluster.**

The PostgreSQL documentation calls this the **data directory** or **data area**. ([PostgreSQL][3])

Conceptually:

```text
/var/lib/postgresql/18/main/
│
├── base/
├── global/
├── pg_wal/
├── pg_tblspc/
├── pg_xact/
├── postgresql.auto.conf
└── other PostgreSQL internal files
```

A fresher should **never manually modify or delete PostgreSQL data files**.

Database administration should be performed using PostgreSQL utilities and SQL.

---

# 10. Configuration Directory

On your Ubuntu installation, PostgreSQL configuration is located under:

```text
/etc/postgresql/18/main/
```

Your running PostgreSQL process explicitly showed:

```text
-c config_file=/etc/postgresql/18/main/postgresql.conf
```

So your environment uses:

```text
/etc/postgresql/18/main/
```

for the primary configuration files.

Important files include:

```text
postgresql.conf
pg_hba.conf
pg_ident.conf
```

### Important terminology

The exact filesystem layout can differ depending on how PostgreSQL was installed.

The upstream PostgreSQL documentation explains the standard configuration concepts and allows configuration files to be located separately from the data directory. ([PostgreSQL][6])

Therefore, for **your Ubuntu installation**, use:

```text
Configuration:
 /etc/postgresql/18/main/
```

---

# 11. `postgresql.conf`

The main server configuration file is:

```text
/etc/postgresql/18/main/postgresql.conf
```

It contains PostgreSQL server configuration parameters.

Examples:

```text
listen_addresses
port
max_connections
shared_buffers
work_mem
maintenance_work_mem
max_worker_processes
wal_level
logging_collector
```

PostgreSQL 18 organizes configuration into areas including connections, authentication, resource consumption, WAL, background writer, I/O, worker processes, and more. ([PostgreSQL][6])

---

# 12. `pg_hba.conf`

The file:

```text
/etc/postgresql/18/main/pg_hba.conf
```

controls **client authentication**.

HBA means:

> **Host-Based Authentication**

For example, a rule can define:

```text
Connection Type
      +
Database
      +
User
      +
Client Address
      +
Authentication Method
```

The official PostgreSQL documentation states that `pg_hba.conf` controls client authentication. ([PostgreSQL][7])

A simplified example:

```text
host    all    all    192.168.1.0/24    scram-sha-256
```

This does not simply mean "give access." 

It defines how PostgreSQL should authenticate matching connection attempts. Database and object privileges are a separate security layer.

---

# 13. `pg_ident.conf`

The file:

```text
/etc/postgresql/18/main/pg_ident.conf
```

supports **identity mapping** for authentication configurations that use user maps.

A beginner should remember:

```text
postgresql.conf
        ↓
Server configuration

pg_hba.conf
        ↓
Client authentication

pg_ident.conf
        ↓
Identity mapping
```

---

# 14. PostgreSQL Port 5432

Your PostgreSQL cluster is listening on:

```text
5432
```

This is PostgreSQL's standard/default port.

You can verify the configured port from SQL:

```sql
SHOW port;
```

Or from Linux:

```bash
sudo ss -lntp | grep 5432
```

The connection concept is:

```text
Client
   │
   │ TCP
   │
   ▼
Server IP : 5432
   │
   ▼
PostgreSQL 18
```

---

# 15. PostgreSQL Roles

PostgreSQL uses the concept of **roles** for access control.

A role can function as:

* a database user
* a group role
* an owner of database objects
* a member of another role

The PostgreSQL documentation explicitly notes that PostgreSQL uses roles rather than maintaining separate "users" and "groups" concepts. ([PostgreSQL][8])

For example:

```text
Role: postgres
Role: app_user
Role: reporting_user
Role: dba_role
```

A beginner should understand the difference between:

```text
Authentication
      ↓
Who are you?

Authorization
      ↓
What are you allowed to do?
```

---

# 16. Connecting Using `psql`

`psql` is PostgreSQL's interactive terminal.

On your server:

```bash
sudo -u postgres psql
```

You should see something similar to:

```text
postgres=#
```

Now try:

```sql
SELECT version();
```

Then:

```sql
SELECT current_database();
```

And:

```sql
SELECT current_user;
```

Exit:

```text
\q
```

---

# 17. Essential Beginner Commands

## Check PostgreSQL clusters

```bash
pg_lsclusters
```

Ubuntu/Debian-specific.

---

## Check PostgreSQL service

```bash
systemctl status postgresql
```

---

## Check your PostgreSQL 18 cluster service

```bash
systemctl status postgresql@18-main
```

---

## Check PostgreSQL version

```bash
sudo -u postgres psql -c "SELECT version();"
```

---

## Check port

```bash
sudo -u postgres psql -c "SHOW port;"
```

---

## Connect to PostgreSQL

```bash
sudo -u postgres psql
```

---

# 18. Starting and Stopping PostgreSQL

PostgreSQL provides the official `pg_ctl` utility for initializing, starting, stopping, restarting, reloading, and checking the status of a PostgreSQL server. ([PostgreSQL][9])

For example:

```bash
pg_ctl status
```

or, when specifying the data directory:

```bash
pg_ctl -D /path/to/data status
```

On Ubuntu/Debian, you will also encounter:

```bash
pg_ctlcluster
```

which is part of the distribution's PostgreSQL cluster-management tooling.

For your cluster:

```bash
sudo pg_ctlcluster 18 main status
```

Start:

```bash
sudo pg_ctlcluster 18 main start
```

Stop:

```bash
sudo pg_ctlcluster 18 main stop
```

Restart:

```bash
sudo pg_ctlcluster 18 main restart
```

Reload:

```bash
sudo pg_ctlcluster 18 main reload
```

---

# 19. Restart vs Reload

This distinction is very important for a DBA.

### Reload

Reload causes PostgreSQL to reread configuration files.

For example:

```bash
sudo pg_ctlcluster 18 main reload
```

PostgreSQL's official documentation explains that configuration can be reread using `pg_ctl reload` or `pg_reload_conf()`. ([PostgreSQL][10])

### Restart

Restart stops and starts the server.

```bash
sudo pg_ctlcluster 18 main restart
```

Some configuration parameters require a restart before they take effect.

Therefore:

```text
Configuration Change
       │
       ├── Reload required
       │
       └── Restart required
```

A DBA should always determine which applies before making a production change.

---

# 20. Understanding `postgresql.service`

On Ubuntu you may see:

```bash
systemctl status postgresql
```

with:

```text
Active: active (exited)
```

This should not automatically be interpreted as:

> PostgreSQL database is down.

Ubuntu/Debian uses its own service and cluster-management framework.

The better checks are:

```bash
pg_lsclusters
```

and:

```bash
systemctl status postgresql@18-main
```

Your server currently reports:

```text
18 main 5432 online
```

and:

```text
Active: active (running)
```

Therefore your PostgreSQL 18 server is running.

---

# 21. PostgreSQL Installation vs Cluster

A fresher should understand that these are different concepts.

You can have:

```text
PostgreSQL 18 Software
│
├── Cluster A
│   └── Port 5432
│
├── Cluster B
│   └── Port 5433
│
└── Cluster C
    └── Port 5434
```

These can be separate PostgreSQL server instances on the same host, each using a different data directory and port. PostgreSQL's official documentation explicitly allows multiple `postgres` instances on one system under those conditions. ([PostgreSQL][4])

Your current server is simpler:

```text
PostgreSQL 18
     │
     └── 18/main
            │
            └── Port 5432
```

---

# 22. How to Verify a Fresh PostgreSQL Installation

After installing PostgreSQL, a fresher should perform these checks.

### Step 1 — Check the cluster

```bash
pg_lsclusters
```

Expected for your environment:

```text
18 main 5432 online postgres /var/lib/postgresql/18/main
```

### Step 2 — Check service

```bash
systemctl status postgresql@18-main
```

Look for:

```text
Active: active (running)
```

### Step 3 — Check version

```bash
sudo -u postgres psql -c "SELECT version();"
```

### Step 4 — Check port

```bash
sudo -u postgres psql -c "SHOW port;"
```

### Step 5 — Check connectivity

```bash
sudo -u postgres psql -c "SELECT current_database(), current_user;"
```

### Step 6 — Check TCP listener

```bash
sudo ss -lntp | grep 5432
```

---

# 23. Beginner Troubleshooting Flow

If PostgreSQL is not working, **don't immediately reinstall it**.

Use this sequence:

```text
                 PostgreSQL Problem
                         │
                         ▼
                  pg_lsclusters
                         │
              ┌──────────┴──────────┐
              │                     │
            online                down
              │                     │
              ▼                     ▼
        Check connectivity      Check service
                                    │
                                    ▼
                         systemctl status
                                    │
                                    ▼
                           Check PostgreSQL log
                                    │
                                    ▼
                         Identify root cause
```

Useful commands:

```bash
pg_lsclusters
```

```bash
systemctl status postgresql@18-main
```

```bash
sudo journalctl -u postgresql@18-main --no-pager
```

```bash
sudo tail -100 /var/log/postgresql/postgresql-18-main.log
```

---

# 24. Important DBA Safety Rules

A fresher should remember these from day one.

### Rule 1 — Don't delete the data directory

Never casually execute:

```bash
rm -rf /var/lib/postgresql/18/main
```

That directory contains your PostgreSQL cluster.

---

### Rule 2 — Don't run `initdb` on an existing cluster

`initdb` creates a new PostgreSQL database cluster. ([PostgreSQL][3])

Do not run it against an existing production cluster.

---

### Rule 3 — Don't edit PostgreSQL files blindly

Understand whether you are changing:

```text
postgresql.conf
pg_hba.conf
pg_ident.conf
```

and understand whether the change requires reload or restart.

---

### Rule 4 — Always verify before changing

Before performing administration:

```bash
pg_lsclusters
```

Then:

```bash
systemctl status postgresql@18-main
```

Know which cluster you are working on.

---

# 25. Beginner Mental Model

If you remember only one diagram from this article, remember this:

```text
                    PostgreSQL 18
                   Software/Binaries
                          │
                          ▼
                   postgres server
                          │
                          ▼
                    18/main Cluster
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
       Database        Database        Database
          │               │               │
          ▼               ▼               ▼
       Schemas          Schemas          Schemas
          │
          ▼
        Tables
          │
          ▼
        Rows
```

And on Ubuntu:

```text
PostgreSQL 18
│
├── Binaries
│   └── /usr/lib/postgresql/18/bin/
│
├── Data
│   └── /var/lib/postgresql/18/main/
│
└── Configuration
    └── /etc/postgresql/18/main/
```

---

# 26. What a Fresher Should Learn Next

This article gives you the **foundation**. Don't jump directly into performance tuning or replication.

A sensible PostgreSQL DBA learning sequence is:

### Level 1 — Fundamentals

1. PostgreSQL architecture
2. PostgreSQL server
3. Database clusters
4. Databases
5. Schemas
6. Tables
7. Roles/users
8. `psql`
9. SQL fundamentals

### Level 2 — Administration

10. Installation
11. Configuration
12. `postgresql.conf`
13. `pg_hba.conf`
14. Users and privileges
15. Database management
16. Tablespaces
17. Logging
18. Service management

### Level 3 — Maintenance

19. VACUUM
20. ANALYZE
21. Autovacuum
22. REINDEX
23. Statistics
24. Bloat
25. Log management

### Level 4 — Backup & Recovery

26. `pg_dump`
27. `pg_dumpall`
28. `pg_restore`
29. Physical backups
30. WAL
31. Point-in-Time Recovery
32. Recovery procedures

### Level 5 — Performance

33. `EXPLAIN`
34. `EXPLAIN ANALYZE`
35. Indexes
36. Query tuning
37. Memory configuration
38. Connection management
39. I/O
40. Wait/event analysis

### Level 6 — HA/DR

41. Streaming replication
42. Primary/standby
43. Replication slots
44. Synchronous replication
45. Asynchronous replication
46. Failover
47. Switchover
48. HA architectures

### Level 7 — Advanced PostgreSQL

49. Logical replication
50. Partitioning
51. Connection pooling
52. Security
53. SSL/TLS
54. Monitoring
55. Automation
56. PostgreSQL upgrades
57. Major-version upgrades

This progression closely follows the subject areas covered by PostgreSQL's official administration documentation, which includes server setup, configuration, authentication, roles, databases, maintenance, backup/recovery, HA/replication, monitoring, WAL, and logical replication. ([PostgreSQL][1])

---

# 27. Quick Reference — Commands Every Fresher Should Know

| Purpose            | Command                                        |
| ------------------ | ---------------------------------------------- |
| List clusters      | `pg_lsclusters`                                |
| PostgreSQL service | `systemctl status postgresql`                  |
| Cluster service    | `systemctl status postgresql@18-main`          |
| PostgreSQL version | `psql --version`                               |
| Server version     | `sudo -u postgres psql -c "SELECT version();"` |
| Connect            | `sudo -u postgres psql`                        |
| Cluster status     | `pg_ctlcluster 18 main status`                 |
| Start              | `sudo pg_ctlcluster 18 main start`             |
| Stop               | `sudo pg_ctlcluster 18 main stop`              |
| Restart            | `sudo pg_ctlcluster 18 main restart`           |
| Reload             | `sudo pg_ctlcluster 18 main reload`            |
| Check port         | `sudo -u postgres psql -c "SHOW port;"`        |
| Check listener     | `sudo ss -lntp \| grep 5432`                   |
| Exit `psql`        | `\q`                                           |

---

# 28. Final Takeaway for a Fresher

The most important concepts to understand before moving into PostgreSQL DBA administration are:

```text
PostgreSQL
    ↓
Server Process (`postgres`)
    ↓
Database Cluster
    ↓
Databases
    ↓
Schemas
    ↓
Tables
    ↓
Rows
```

For your Ubuntu 24.04 lab:

```text
PostgreSQL 18
      │
      ▼
   18/main
      │
      ├── Port: 5432
      ├── Status: ONLINE
      ├── Data:
      │   /var/lib/postgresql/18/main
      │
      └── Configuration:
          /etc/postgresql/18/main
```

Once this mental model is clear, the next subjects—**PostgreSQL memory architecture, process architecture, WAL, MVCC, VACUUM, indexes, backup/recovery, streaming replication, and performance tuning**—become much easier to understand.

### Official PostgreSQL 18 references

* [PostgreSQL 18 Documentation](https://www.postgresql.org/docs/18/?utm_source=chatgpt.com)
* [PostgreSQL 18 Tutorial](https://www.postgresql.org/docs/18/tutorial.html?utm_source=chatgpt.com)
* [PostgreSQL 18 Server Administration](https://www.postgresql.org/docs/18/admin.html?utm_source=chatgpt.com)
* [PostgreSQL 18 Server Setup and Operation](https://www.postgresql.org/docs/18/runtime.html?utm_source=chatgpt.com)
* [PostgreSQL 18 Server Configuration](https://www.postgresql.org/docs/18/runtime-config.html?utm_source=chatgpt.com)
* [PostgreSQL 18 Database Cluster](https://www.postgresql.org/docs/18/creating-cluster.html?utm_source=chatgpt.com)
* [PostgreSQL 18 `pg_ctl` Reference](https://www.postgresql.org/docs/18/app-pg-ctl.html?utm_source=chatgpt.com)

These references are based on the current PostgreSQL 18 official documentation. ([PostgreSQL][11])

[1]: https://www.postgresql.org/docs/18/admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Part III. Server Administration"
[2]: https://www.postgresql.org/docs/18/tutorial.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Part I. Tutorial"
[3]: https://www.postgresql.org/docs/18/creating-cluster.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.2. Creating a Database Cluster"
[4]: https://www.postgresql.org/docs/current/app-postgres.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: postgres"
[5]: https://www.postgresql.org/docs/18/connect-estab.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 51.2. How Connections Are Established"
[6]: https://www.postgresql.org/docs/18/runtime-config.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 19. Server Configuration"
[7]: https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[8]: https://www.postgresql.org/docs/18/user-manag.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 21. Database Roles"
[9]: https://www.postgresql.org/docs/18/app-pg-ctl.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_ctl"
[10]: https://www.postgresql.org/docs/18/config-setting.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.1. Setting Parameters"
[11]: https://www.postgresql.org/docs/18/?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: PostgreSQL 18.4 Documentation"
