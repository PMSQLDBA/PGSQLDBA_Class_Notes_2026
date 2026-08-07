# PostgreSQL Database Administration — Course Outline

**Day 2 — 3rd August 2026**

---

## Types of Databases

- **RDBMS** — data is managed in relational (table) format.
- **NoSQL Databases** — data is managed in unstructured/semi-structured formats such as documents, key-value stores, or wide-column stores.

For real-time operations, **RDBMS** is the suitable choice.

For historical data storage, **NoSQL databases** are typically used — e.g., Aerospike, MongoDB, Redshift, Hadoop.

### RDBMS (Relational Databases)

**OLTP — Online Transaction Processing**
Real-time operations such as ATM transactions, bank transactions, ticket booking, and live tracking fall into this category.

- Status must be updated immediately.
- These are called **OLTP transactions** — real-time transactions that need to be updated within a fraction of a second.
- Examples: Oracle, SQL Server, PostgreSQL, MySQL, MariaDB, DB2, Sybase.

**OLAP — Online Analytical Processing**
Used for online analytics purposes — in most cases, queries are written against historical data.

- Involves huge volumes of data, on top of which analytics are performed.
- Examples: Teradata, Pivotal HAWQ, Greenplum, PostgreSQL, MS SQL, Oracle (these databases can also be used for OLAP purposes).

*Note: Performance matters most for OLTP databases, since they serve live, latency-sensitive operations.*

Data loading into OLAP databases is usually a bulk/batch process rather than manual insertion, one row at a time.

For OLTP databases, application services insert data via **APIs**. Every API call contains SQL statements in the background.

**Example — Banking APIs:**
- **Deposit API** → `INSERT` + `UPDATE` SQL statements
- **Withdraw API** → `DELETE` *(typically a debit/`UPDATE`, not literally a row delete — confirm intended meaning)* + `UPDATE` SQL statements
- **Postings API** → interest calculation
- **Loan API**

OLAP database data loading is done using **ETL tools** (Informatica PowerCenter, Talend, Pentaho, etc.).

**E — Extraction:** Raw data (junk data, combined from different sources, with no proper format or meaning) is filtered using ETL tools to extract only the required amount of data.
*Example: Raw data of 10 TB → ETL operations → 500 GB of useful data.*

**T — Transformation:** The extracted data's format is changed to match the target database's requirements. Once transformed, the data is typically available as flat files or spreadsheet files.

**L — Loading:** Developers load the transformed data into the target database.

This is how data typically arrives in OLAP databases.

---

## OLTP Database Technologies

1. Oracle
2. SQL Server (MS SQL)
3. PostgreSQL
4. MySQL
5. MariaDB
6. MongoDB
7. DB2
8. Sybase
9. Teradata
10. Greenplum
11. Cassandra
12. ClickHouse
13. Aerospike
14. DynamoDB
15. MemSQL (now SingleStore)
16. Redis
17. ElastiCache
18. Couchbase
19. CockroachDB
20. TimescaleDB
21. Pivotal HAWQ
22. Hive
23. Redshift
24. GraphDB
25. TiDB

*Note: This list mixes OLTP RDBMS with several OLAP/analytical and NoSQL systems (e.g., Teradata, Greenplum, Redshift, Hive, Cassandra, Redis) — worth clarifying in class which entries are strictly OLTP vs. OLAP/NoSQL, since a few (Teradata, Greenplum, Redshift, Hive, Pivotal HAWQ) were already listed above as OLAP examples.*

---

## Why We Need to Choose PostgreSQL

1. **Open-source database** — the software is available free of cost; anyone can download and use it.
2. A largely comparable, simplified alternative to Oracle in terms of feature depth *(originally stated as "100% simplified version of Oracle" — treat as a teaching simplification rather than a literal technical claim)*.
3. Simple to use.
4. Supports both **OLTP** and **OLAP** transactions.
5. Also functions as a **NoSQL database** (via `HSTORE` and `JSON`/`JSONB` data types).
6. Offers high-level, 3-layer security free of cost:
   - Client/server-level authentication
   - Schema-level access control
   - Object-level access control
7. Supports **High Availability**.
8. Supports **Disaster Recovery** capabilities.
9. Supports **Point-in-Time Recovery (PITR)**:
   - Backup taken at 6 AM.
   - 7 AM — a large table is created and 100 GB of data is loaded.
   - 10 AM — the table gets dropped.
   - Even without a backup taken after the table creation, recovery to a specific point in time is still possible using the **full backup + required transaction logs**. This is called **PITR**.
10. User-friendly GUI available (**pgAdmin 4**).
11. Built-in **incremental backup** option from version 17 onward.
12. Strong third-party/commercial support ecosystem has grown around PostgreSQL:
    - EDB (EnterpriseDB)
    - Cybertec
    - Percona
    - Fujitsu
    - DB Experts
    - DigitalOcean
    - Citus Data
    - Crunchy Data
    - DBi
    - 2ndQuadrant (merged with EDB)
    - OpenSCG (merged with AWS)
    - MyDBOps
    - Ashnik
    - Datavail
    - ...and others offering paid support.

    *If there is a bug/issue with the software itself, users can report it directly to the open-source PostgreSQL team. The earlier response ETA was 24 hours; over the last 2 years, this has reduced to approximately 4 hours.*

13. **EDB's role:** Taking advantage of this support opportunity, EDB entered the picture — selling PostgreSQL along with licensing and support.
    - There is a mutual understanding between the open-source community and the EDB team.
    - EDB provides funding to the open-source team to help implement new features and versions — including funding in the form of developer resources.
    - EDB rebrands and sells the product as **EDB Postgres**.
    - Using Java frameworks, EDB has built additional tools on top of open-source PostgreSQL:
      - **EDB BART** (Backup and Recovery Tool) — for incremental backup purposes; comparable to **pgBackRest** (used up through PostgreSQL 16; from 17 onward, incremental backup is a direct built-in feature).
      - **PEM** (Postgres Enterprise Manager) — for monitoring; comparable to **Nagios**/**PMM**.
      - **EDB MTK** (Migration Toolkit) — for migrating data from Oracle to PostgreSQL; comparable to **Ora2Pg**.
      - **EFM** (Enterprise Failover Manager) — for high availability; comparable to **Patroni**/**repmgr**/auto-failover extensions.

    **Standard edition vs. enterprise edition?**
    Unlike many other databases, PostgreSQL's open-source and EDB editions share the same core source code, so the underlying features are the same.
    - There is an agreement between the open-source team and EDB: EDB should not make direct changes to the core software.
    - In practice, roughly 95% of users run open-source PostgreSQL, while the remaining ~5% use EDB PostgreSQL.

14. Licensing and support costs for other databases are very high by comparison. PostgreSQL is very reasonably priced — even EDB PostgreSQL is cited as being up to **18 times cheaper** than Oracle *(figure as stated in class — treat as an illustrative comparison rather than a verified benchmark)*.

---

## History of PostgreSQL

*Note: The dates and core facts below are corrected/clarified from the original notes.*

- The project that became PostgreSQL originated at the **University of California, Berkeley** *(corrected from "Barclays university" in the original notes)*, led by **Professor Michael Stonebraker** and his research team, building on their earlier **Ingres** project.
- The successor project, designed to address Ingres's limitations, was named **POSTGRES** (started around 1986).
- In **1996**, the project was renamed **PostgreSQL** to reflect its added SQL support.
- Early version numbering started low (e.g., 1.0, 1.5) and, over time, followed a versioning style similar to other major databases (e.g., 6.0, 7.0, 7.0.x, 7.1, 7.2, 7.3, 8.0, 9.0 ... 10, 11, 12, 13, 14, 15, 16, 17, 17.1, 17.2, 17.3, 18, 18.1, 18.2, 18.3).
- **PostgreSQL 19** is currently in beta/testing phase and is expected to be released around **September 2026** *(confirm exact date against the official PostgreSQL release announcement closer to the time)*.
- Across major and minor releases combined, PostgreSQL has published **150+ versions** to date.
- PostgreSQL has been recognized as a top-growing RDBMS in industry rankings for several consecutive years.
- Looking ahead, versions **20/21** are anticipated to introduce **RAC-like (Real Application Clusters) functionality** — a capability often cited as a key remaining differentiator for Oracle.

---
