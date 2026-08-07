# PostgreSQL Database Administration — Course Outline

**Day 1 — 2nd August 2026**

---

## 1. What Is a Database, and What Databases Are Available?
An overview of what a database is and a survey of popular database systems in the market (e.g., Oracle, MySQL, PostgreSQL, SQL Server, MongoDB).

*Note: Databases are broadly classified as relational (RDBMS) — Oracle, PostgreSQL, MySQL, SQL Server — and non-relational (NoSQL) — MongoDB, Cassandra, Redis.*

## 2. How Did the PostgreSQL Database Come Into the Market?
The origin story of PostgreSQL — its evolution from the POSTGRES project at the University of California, Berkeley.

## 3. History and Introduction to PostgreSQL
A deeper look at PostgreSQL's history and a general introduction to the database.

*Note: PostgreSQL began as "Postgres" (1986, Berkeley, led by Michael Stonebraker), was renamed PostgreSQL in 1996 after SQL support was added, and is maintained today by the open-source PostgreSQL Global Development Group.*

## 4. Why Choose PostgreSQL Over Other Databases?
With so many databases available, this section covers why PostgreSQL stands out — its open-source licensing, extensibility, standards compliance, and enterprise-grade features.

## 5. PostgreSQL Database Architecture
Architecture is studied from three angles — query execution/process flow, and memory utilization:

- **5a. General Architecture**
- **5b. Memory-Based Architecture**
- **5c. Process-Based Architecture**

Followed by a discussion of the full/complete PostgreSQL architecture.

## 6. Linux Basics and Setup on Your Laptop
Fundamental Linux commands and environment setup needed before installing PostgreSQL.

## 7. PostgreSQL Software Installation
- **7a. Source Code Installation** (manual installation)
- **7b. YUM Installation** (automatic installation)
- **7c. Software Uninstallation and Proper Cleanup**

## 8. Data Directory Explanation
Understanding the structure and purpose of the PostgreSQL data directory (`PGDATA`).

## 9. PostgreSQL Service Auto-Startup
Configuring the PostgreSQL service to start automatically on system boot.

## 10. PostgreSQL Environment Variable Setup (`.bash_profile`)
Setting environment variables such as `PATH`, `PGDATA`, and `PGHOME` for smooth database operations.

## 11a. Object Flow in PostgreSQL
How database objects relate to and depend on one another.

## 11. PostgreSQL Database Connectivity
Methods of connecting to a PostgreSQL database (psql, pgAdmin, JDBC/ODBC, etc.).

## 12. Object Management in PostgreSQL
Covers databases, schemas, tables, views, sequences, functions, and users.

## 13. Audit Logging in PostgreSQL
Tracking and logging database activity for security and compliance purposes.

## 14. Host-Based Authentication (High-Level Security)
Understanding `pg_hba.conf` and how PostgreSQL controls client authentication based on host, database, and user.

## 15. User Access Management
Covers users, user access rights, roles, and permissions.

## 16. Metadata Tables (Dictionary Tables)
Used to retrieve information about the database system itself:
- `pg_catalog`
- `information_schema`

## 17. Physical Structure of PostgreSQL
How data is physically stored on disk (files, pages/blocks, etc.).

## 18. Tablespace Management in PostgreSQL
What a tablespace is, how to create one, and how to manage it.

## 19. Data Export and Import
Exporting table data to a file and importing file data back into a table.

- **COPY** — utility for data export and import.

## 20. Backup and Restore in PostgreSQL

**20a. Logical Backups**
- `pg_dump`
- `pg_dumpall`
- `psql`
- `pg_restore`

**20b. Physical Backup**
- `pg_basebackup`
  - Restore as a separate instance
  - Restore on top of an existing instance

*Note: 20+ backup and restore scenarios will be covered in hands-on labs.*

## 21. Point-in-Time Recovery (PITR)
Restoring/recovering a database to a specific point in time is called **Point-in-Time Recovery**.

Requirements: a full backup **plus** the required amount of transaction logs.

- Up to PostgreSQL version 16, there was no built-in concept of **incremental backup**.
- From **PostgreSQL 17 onward**, incremental backup is a built-in feature.

**Current PostgreSQL version: 18**
**PostgreSQL 19** is expected to be released around **September 24, 2026** (per the standard annual release cycle — confirm against the official release announcement closer to the date).

- Transaction log management is handled via **WAL files** (Write-Ahead Log files), which are automatically recycled after a specific threshold.

## 22. Archive Log Management
Taking backups of transaction logs (WAL archiving) for recovery purposes.

## 23. Prepared Transactions

```sql
BEGIN;
CREATE TABLE EMP (...);
INSERT INTO EMP VALUES (...);  -- 100 records
END;
```

Now log in to the database again and run:
```sql
SELECT * FROM EMP;
```
**Question: How many records will be printed?**
**Answer:** The table won't exist at all.

*Explanation: If you perform work inside a `BEGIN` block, you must explicitly run `COMMIT` for the changes to be saved. Running `END` (or simply exiting the session) without a `COMMIT` results in an automatic `ROLLBACK`.*

So, in a `BEGIN...END` block, the commit/rollback decision is scoped to that session — this is standard **BTET** (Begin Transaction, End Transaction) behavior.

**Prepared Transaction:** Do the work today; commit whenever you want (i.e., the transaction can be "prepared" and persisted independently of the session, then committed later — even by a different session).

## 24. PgBench
A tool for benchmarking and estimating database capacity.

## 25. PgBadger
A free log analyzer tool that reviews audit/query logs and generates a report in HTML format.

## 26. Database Maintenance Activities
- `ANALYZE`
- `VACUUM`
- `VACUUM FULL`
- `REINDEX`

## 27. Database Upgrade and Patching
- Major Upgrade
- Minor Upgrade
- Patching
- Rollback Upgrade
- Upgrade With Extra Space
- Upgrade Without Extra Space

## 28. PostgreSQL Replication Setup

**Types and ways of replication:**

**Streaming Replication** — complete server-level replication:
- One-to-one replication (A → B)
- Multi-standby replication (A → B, C)
- Cascading replication (A → B → C)

**Logical Replication** — replicate one or more selected objects from one server to another.

**PG_REWIND** — resynchronize a diverged standby with the primary.

*Note: Replication alone does not provide High Availability (HA) by default — manual promotion is required.*
- Manual Failover
- Manual Fallback

To achieve **automatic** high availability, a **cluster setup** is needed, which relies on HA tools:
- Auto Failover Extension (a 3-node cluster will be built in labs)
- Auto Switchover
- Auto Failover
- Auto Failback

## 30. PgBouncer
Connection pooling / reserved connections for PostgreSQL.

## 31. Inheritance
Table inheritance in PostgreSQL.

## 32. Partitioning
Splitting a table into smaller pieces to speed up read operations:
- Range Partitioning
- List Partitioning
- Hash-Based Partitioning

## 33. DBLink
For cross-database connections.

## 34. Monitoring
- OS-level monitoring and database-level monitoring
- `pg_stat_activity`

## 35. Performance Tuning
- OS-Level Tuning
- Database-Level Tuning
- Query-Level Tuning

## 36. Row-Level Security (RLS) — Policy
Restricting row visibility/access based on defined policies.

## 37. MVCC (Multi-Version Concurrency Control)
How PostgreSQL manages concurrent access without locking readers/writers against each other.

## 38. Transaction Isolation Levels
- Read Committed
- Repeatable Read
- Serializable

## 39. ACID Properties
- **A**tomicity
- **C**onsistency
- **I**solation
- **D**urability

## 40. AWS Cloud Basics and Login Creation
- RDS Setup and Management

## 41. Oracle to PostgreSQL Migration (Cloud Platform)
- PostgreSQL → PostgreSQL (simple replication)
- PostgreSQL 15 → PostgreSQL 18 (upgrade path)

## 42. Extensions
Beyond the regular built-in features, PostgreSQL supports extensions such as:
- `pg_stat_statements`
- `pg_activity`
- `dblink`
- `passwordcheck`
- `pg_buffercache`
- `hypopg` *(corrected from "pg_hypopg" — the extension is commonly named `hypopg`)*

## 43. Difference Between Open Source PostgreSQL and EDB PostgreSQL
Comparing the community (open source) edition of PostgreSQL with EDB's (EnterpriseDB) enhanced/commercial distribution.

---
