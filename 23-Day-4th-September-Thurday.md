The first file mainly covers **pgBench and pgBadger** , while the second covers **Logical Replication, monitoring, and a logical-replication-based upgrade approach**. 
Validated the PostgreSQL-core material against the current **PostgreSQL 18 official documentation**. 
One important distinction: **pgBadger is not part of PostgreSQL core**, so PostgreSQL configuration statements were checked against PostgreSQL documentation, while pgBadger-specific behavior was checked against the pgBadger project documentation.


# PostgreSQL DBA Class Notes

## pgBench, pgBadger, Logical Replication, Monitoring and Upgrade Concepts

---

# 1. pgBench – PostgreSQL Benchmarking Utility

## 1.1 What is pgBench?

`pgbench` is PostgreSQL's built-in benchmarking utility.

It can run a defined sequence of SQL commands repeatedly across multiple concurrent database sessions and calculate performance statistics such as:

* Transactions per second (TPS)
* Average transaction latency
* Failed transactions
* Number of transactions processed
* Initial connection time
* Performance under different client loads

The default benchmark workload is loosely based on TPC-B, but it is **not an official TPC-B benchmark**.

Custom SQL workloads can also be tested using script files.

Official PostgreSQL documentation confirms that `pgbench` is intended for benchmark testing and can simulate multiple concurrent database sessions. ([PostgreSQL][1])

---

# 2. Why DBAs Use pgBench

A DBA can use pgBench to:

1. Establish a database performance baseline.
2. Test the effect of increasing concurrent database sessions.
3. Compare PostgreSQL configuration changes.
4. Compare server hardware or storage.
5. Test read-only workloads.
6. Test read/write workloads.
7. Test application-like SQL using custom scripts.
8. Measure TPS and transaction latency.
9. Perform controlled load testing before production changes.
10. Compare performance before and after PostgreSQL upgrades.

### Important

pgBench does **not** automatically tell us:

> "This PostgreSQL server supports exactly 500 users."

Instead, we test progressively increasing workloads and determine the point where latency, throughput, errors, CPU, memory, I/O, locks, or other metrics no longer satisfy the application's SLA.

---

# 3. Important pgBench Parameters

| Parameter | Meaning                                         |
| --------- | ----------------------------------------------- |
| `-i`      | Initialize pgBench benchmark tables             |
| `-c`      | Number of concurrent clients                    |
| `-t`      | Number of transactions per client               |
| `-T`      | Run benchmark for a specified number of seconds |
| `-j`      | Number of pgBench worker threads                |
| `-S`      | Run built-in SELECT-only workload               |
| `-f`      | Execute a custom pgBench script                 |
| `-P`      | Display progress periodically                   |
| `-L`      | Define transaction latency limit                |
| `-n`      | Do not VACUUM before running the test           |
| `-s`      | Scale factor                                    |

`-t` and `-T` are mutually exclusive. ([PostgreSQL][1])

---

# 4. Initialize the pgBench Database

Before running the built-in benchmark for the first time, initialize the test database.

Example:

```bash
/usr/pgsql-18/bin/pgbench -i postgres
```

pgBench creates its standard benchmark objects, including tables such as:

```text
pgbench_accounts
pgbench_branches
pgbench_history
pgbench_tellers
```

Initialization normally performs activities such as:

* Creating benchmark tables
* Generating test data
* Vacuuming
* Creating primary keys

---

# 5. Scale Factor – Very Important

For meaningful load testing, do not always use the default scale factor of `1`.

Example:

```bash
/usr/pgsql-18/bin/pgbench -i -s 50 postgres
```

Here:

```text
-s 50
```

means a scale factor of 50.

For the default pgBench workload, PostgreSQL documentation recommends making the scale factor at least as large as the largest number of clients being tested. 

Otherwise, excessive contention on the small number of branch rows can distort the benchmark. ([PostgreSQL][1])

Therefore, testing:

```bash
pgbench -c 100 ...
```

against a database initialized with:

```text
scale factor = 1
```

is usually not a good capacity benchmark for the standard read/write workload.

---

# 6. Basic Read/Write Benchmark

Example:

```bash
/usr/pgsql-18/bin/pgbench -c 10 -t 10 postgres
```

Meaning:

* `-c 10` → 10 concurrent clients
* `-t 10` → each client executes 10 transactions

Expected total:

```text
10 clients × 10 transactions = 100 transactions
```

Example result:

```text
number of clients: 10
number of transactions per client: 10
number of transactions actually processed: 100/100
number of failed transactions: 0
latency average = 9.388 ms
tps = 1065.144221
```

---

# 7. Understanding pgBench Output

## 7.1 Number of clients

```text
number of clients: 10
```

Represents concurrent pgBench client sessions.

It should not automatically be interpreted as the number of real application users.

---

## 7.2 Transactions processed

```text
100/100
```

means all 100 requested transactions were processed.

---

## 7.3 Failed transactions

```text
number of failed transactions: 0
```

Ideally, benchmark failures should remain zero unless the test intentionally examines failure/retry scenarios.

---

## 7.4 Average latency

Example:

```text
latency average = 9.388 ms
```

This represents average transaction latency during the test.

Lower is generally better, but acceptable latency depends on the application SLA.

---

## 7.5 TPS

Example:

```text
tps = 1065.144221
```

TPS means:

> Transactions Per Second.

Higher TPS can indicate greater throughput, but TPS should **never be evaluated alone**.

Always evaluate TPS together with latency, errors, CPU, memory, storage I/O, locking and the actual workload.

---

# 8. Testing Higher Concurrency

Example:

```bash
/usr/pgsql-18/bin/pgbench -c 20 -t 30 postgres
```

This means:

```text
20 clients × 30 transactions
= 600 transactions
```

The class test produced approximately:

```text
latency average = 18.610 ms
tps = 1074.675627
```

As concurrency increases, latency commonly increases because more sessions compete for CPU, memory, locks, buffers and storage resources.

---

# 9. SELECT-Only Benchmark

Use:

```bash
/usr/pgsql-18/bin/pgbench -c 20 -t 30 -S postgres
```

`-S` is shorthand for the built-in `select-only` workload. ([PostgreSQL][1])

Example class result:

```text
number of clients: 20
number of transactions: 600
failed transactions: 0
latency average = 2.671 ms
tps = 7488.112621
```

This result should **not** be directly compared with the default read/write TPS as though both represent the same workload.

SELECT-only transactions are substantially different from the default pgBench read/write transactions.

---

# 10. Custom Workload Testing

pgBench supports custom SQL scripts.

Example:

```bash
/usr/pgsql-18/bin/pgbench \
-c 20 \
-t 30 \
-f test.sql \
postgres
```

The `-f` option tells pgBench to execute a custom transaction script. ([PostgreSQL][1])

This is useful when the DBA wants to simulate application-specific SQL rather than the built-in pgBench workload.

### Best Practice

Do not place hundreds of repeated hard-coded INSERT statements into the script when a cleaner parameterized workload can represent the same test.

Custom pgBench scripts should ideally model the application's real:

* SELECT pattern
* INSERT pattern
* UPDATE pattern
* DELETE pattern
* Transaction boundaries
* Data distribution
* Think time/rate
* Concurrency

---

# 11. Better Benchmarking Using Time-Based Tests

For capacity testing, a duration-based test is usually easier to compare than very short transaction-count tests.

Example:

```bash
pgbench -c 20 -j 4 -T 300 -P 10 postgres
```

Meaning:

* 20 clients
* 4 pgBench worker threads
* Run for 300 seconds
* Report progress every 10 seconds

PostgreSQL specifically warns that very short pgBench tests can produce meaningless results and recommends running benchmarks long enough to average out noise—typically at least several minutes. ([PostgreSQL][1])

---

# 12. Testing a 10-Second Latency Requirement

Suppose the requirement is:

> Transactions should complete within 10 seconds.

pgBench supports a latency limit.

Example:

```bash
pgbench \
-c 50 \
-j 4 \
-T 300 \
-P 10 \
-L 10000 \
postgres
```

Because `-L` uses milliseconds:

```text
10000 ms = 10 seconds
```

Transactions exceeding the latency limit are reported as late. ([PostgreSQL][1])

This provides a much better approach than simply asking:

> "How many connections can PostgreSQL execute?"

Increase concurrency gradually and monitor when the application's latency SLA starts being violated.

---

# 13. Recommended Capacity-Test Pattern

Example progression:

```text
10 clients
20 clients
50 clients
100 clients
200 clients
...
```

At every level record:

* TPS
* Average latency
* Late transactions
* Failed transactions
* CPU utilization
* Memory utilization
* Disk latency
* Disk throughput
* WAL generation
* Locks
* Checkpoint activity
* Active sessions

The maximum acceptable concurrency is the point where the workload still satisfies the required SLA with adequate operational headroom.

---

# 14. pgBench Best Practices

PostgreSQL documentation provides several important warnings about pgBench. ([PostgreSQL][1])

### 14.1 Do not run very short tests

Run tests for several minutes or longer.

### 14.2 Repeat tests

Run the same benchmark several times and compare results.

### 14.3 Use a realistic scale factor

The test database should be large enough to represent the intended workload.

### 14.4 Watch autovacuum

Autovacuum activity can influence benchmark results.

### 14.5 Consider a separate load-generator server

For large tests, pgBench itself can become the bottleneck.

A better architecture can be:

```text
pgBench Load Generator
        |
        | Network
        v
PostgreSQL Database Server
```

### 14.6 Monitor the database while benchmarking

Use PostgreSQL monitoring views such as:

```sql
SELECT * FROM pg_stat_activity;
```

and appropriate OS/cloud monitoring.

---

# 15. Performance Tuning After pgBench Testing

The original note stated:

> Increase `shared_buffers` and `work_mem` to improve response time without reducing connections.

This is **not universally correct**.

Increasing these parameters does not automatically improve every workload.

`shared_buffers` controls PostgreSQL's shared buffer cache. PostgreSQL documentation suggests approximately 25% of system memory as a reasonable starting point for a dedicated database server with 1 GB or more RAM, while noting that workloads vary. ([PostgreSQL][2])

`work_mem` controls memory available to individual query operations such as sorts and hashes, and it can be allocated multiple times by multiple concurrent sessions.

Therefore:

> Never increase `work_mem` simply because concurrency increased.

Excessive `work_mem` can create significant total memory pressure under high concurrency.

Performance tuning should be based on evidence from the workload.

---

# 16. pgBadger – PostgreSQL Log Analyzer

## 16.1 What is pgBadger?

pgBadger is a third-party PostgreSQL log-analysis tool.

It is **not part of PostgreSQL core**.

It parses PostgreSQL log files and generates detailed reports, commonly in HTML format.

It can help analyze:

* Slow queries
* Most frequent queries
* Query duration
* Errors
* Connections
* Sessions
* Checkpoints
* Locks
* Temporary files
* VACUUM activity
* Autovacuum activity

The current pgBadger project describes it as a PostgreSQL log analyzer capable of generating detailed reports and graphs. ([GitHub][3])

---

# 17. pgBadger Version Correction

The class notes used:

```text
pgBadger 12.4
```

That version is old.

As of this note revision, pgBadger **13.2** is the latest stable release identified from the project's releases and PostgreSQL community announcement. ([GitHub][4])

For new installations, always check the current pgBadger release instead of hard-coding an old download URL.

---

# 18. Installing pgBadger

Typical source installation:

```bash
tar xzf pgbadger-<version>.tar.gz
cd pgbadger-<version>

perl Makefile.PL
make
sudo make install
```

Verify:

```bash
pgbadger --version
```

The installation method can vary depending on the operating system and available package repository.

---

# 19. PostgreSQL Logging for pgBadger

pgBadger requires PostgreSQL to generate suitable logs.

A useful starting configuration can include:

```conf
logging_collector = on

log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '

log_min_duration_statement = 1000

log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

The exact settings should depend on workload and logging requirements.

---

# 20. Important Correction – Do Not Blindly Log Everything

The original notes enabled:

```conf
log_min_duration_statement = 0
log_statement = 'all'
```

for pgBadger.

That is not a recommended production configuration.

The pgBadger documentation warns against enabling overlapping statement/duration logging because it can produce incorrect counters and drastically increase log volume. It recommends `log_min_duration_statement` for query-performance reporting. ([GitHub][3])

For example:

```conf
log_statement = 'none'
log_min_duration_statement = 1000
```

would log statements taking 1 second or longer.

The correct threshold depends on the application.

---

# 21. PostgreSQL Logging Overhead

Be careful with:

```conf
log_min_duration_statement = 0
```

because it logs every completed SQL statement with duration.

Likewise:

```conf
log_statement = 'all'
```

logs every SQL statement.

On a busy production database, excessive logging can:

* Generate very large log files
* Increase storage usage
* Increase I/O
* Make analysis harder
* Add logging overhead

Therefore, logging should be designed according to the monitoring requirement.

---

# 22. Generate pgBadger HTML Report

Example:

```bash
pgbadger postgresql.log -o pgbadger_report.html
```

Verify:

```bash
ls -lh pgbadger_report.html
```

The HTML file can then be transferred securely to an administration workstation and opened in a browser.

---

# 23. Logical Replication

## 23.1 What is Logical Replication?

PostgreSQL logical replication replicates data changes based on logical objects and replication identity rather than copying physical database blocks.

It uses a:

```text
Publication → Subscription
```

model. ([PostgreSQL][5])

Architecture:

```text
Publisher
   |
   | WAL
   |
   | Logical Decoding
   |
   | pgoutput
   v
Subscriber
   |
   v
Apply Worker
```

The publisher's WAL sender logically decodes changes and sends them to the subscriber, where apply workers apply the changes. ([PostgreSQL][6])

---

# 24. Logical Replication Terminology

## Publisher

The PostgreSQL database containing the publication.

## Publication

Defines which tables/schemas and which supported changes are published.

## Subscriber

The PostgreSQL database receiving the changes.

## Subscription

Defines the connection to one or more publications.

## Replication Slot

Tracks the subscriber's position and helps ensure required WAL remains available.

## Replica Identity

Identifies rows for UPDATE and DELETE replication.

Usually this is the table's primary key.

---

# 25. Logical Replication Use Cases

PostgreSQL officially identifies use cases including: ([PostgreSQL][5])

1. Replicating incremental database changes.
2. Replicating selected tables.
3. Consolidating multiple databases for analytics.
4. Replicating between different PostgreSQL major versions.
5. Replicating between different platforms.
6. Giving different users access to replicated data.
7. Sharing subsets of databases.
8. Building more complex replication topologies.

A subscriber can itself become a publisher, allowing cascading designs.

---

# 26. Logical Replication vs Physical Replication

| Physical Replication                                                | Logical Replication                             |
| ------------------------------------------------------------------- | ----------------------------------------------- |
| Block/WAL-level replication                                         | Logical row-change replication                  |
| Typically whole cluster                                             | Selected tables/schemas possible                |
| Primary/standby architecture                                        | Publisher/subscriber architecture               |
| Closely tied to physical structure                                  | More flexible                                   |
| Common HA/DR mechanism                                              | Common migration/integration mechanism          |
| Same major-version requirements apply to physical upgrade scenarios | Different major PostgreSQL versions can be used |

---

# 27. Supported Operations

A publication can publish combinations of:

```text
INSERT
UPDATE
DELETE
TRUNCATE
```

By default, all four are published. ([PostgreSQL][7])

Example:

```sql
CREATE PUBLICATION mypub
FOR TABLE public.test_rep;
```

---

# 28. Replica Identity Requirement

The original note stated:

> Tables must have a primary key or unique key.

This needs correction.

A primary key is **not mandatory for every logical replication scenario**.

However, a table needs an appropriate **replica identity** for UPDATE and DELETE replication.

By default, PostgreSQL uses the primary key.

A suitable unique index can also be configured as replica identity.

Another option is:

```sql
ALTER TABLE table_name
REPLICA IDENTITY FULL;
```

although this can have performance implications. ([PostgreSQL][7])

---

# 29. Logical Replication Restrictions

Important PostgreSQL 18 restrictions include: ([PostgreSQL][8])

### 29.1 DDL is not automatically replicated

Commands such as:

```text
CREATE
ALTER
DROP
```

must normally be managed separately.

### 29.2 Schema definitions are not automatically replicated

Create compatible subscriber tables before starting replication.

### 29.3 Sequence state is not replicated

For example, an identity/serial column's values are replicated as table data, but the underlying sequence state itself is not synchronized automatically.

### 29.4 Large objects are not replicated

PostgreSQL large objects are not supported by built-in logical replication.

### 29.5 Supported target relations are tables

Views, materialized views and foreign tables cannot be normal logical replication targets.

---

# 30. Subscriber Table Requirements

The original note stated:

> The tables must have exactly the same schema.

That is too strict.

PostgreSQL matches tables using their fully qualified names:

```text
schema.table
```

Columns are matched by **column name**, not physical column order. ([PostgreSQL][9])

Therefore, the subscriber can have:

* Different column order
* Additional columns
* In some cases, compatible but different data types

However, the schemas must remain compatible enough for incoming replicated data to be applied successfully.

---

# 31. Same-Cluster Replication Correction

The original notes stated:

> You cannot stream to the same host.

This is incorrect as a general rule.

PostgreSQL can perform logical replication between databases in the same PostgreSQL cluster, but special handling is required.

A normal:

```sql
CREATE SUBSCRIPTION
```

that attempts to create its own replication slot against the same cluster can hang.

PostgreSQL documents a method where the logical replication slot is created separately and the subscription is created with:

```text
create_slot = false
```

Therefore:

> Separate servers are common and easier operationally, but they are not an absolute PostgreSQL requirement. ([PostgreSQL][10])

---

# 32. Publisher Configuration

The critical publisher setting is:

```conf
wal_level = logical
```

PostgreSQL also requires adequate values for:

```conf
max_replication_slots
max_wal_senders
```

PostgreSQL recommends:

* `max_replication_slots` sufficient for subscriptions plus synchronization workers.
* `max_wal_senders` at least as large as `max_replication_slots`, plus capacity for physical replicas. ([PostgreSQL][11])

---

# 33. Is wal_keep_size Required?

The original notes configured:

```conf
wal_keep_size = 1024
```

as though it were a logical replication requirement.

It is **not a basic requirement for logical replication**.

Logical replication normally uses replication slots to retain the WAL required by subscribers.

`wal_keep_size` can be relevant to WAL retention strategies, particularly physical replication, but should not be presented as a mandatory logical replication setting.

---

# 34. Network Configuration

For remote subscriber connections, PostgreSQL must listen on an appropriate network interface.

Example:

```conf
listen_addresses = '*'
```

However, production environments should bind only to required interfaces where practical.

Firewall/security-group access must also allow the PostgreSQL port, normally:

```text
5432
```

between the required systems.

---

# 35. pg_hba.conf Configuration

Avoid:

```text
trust
```

authentication for normal production replication.

A better approach is to use a dedicated replication user and SCRAM authentication.

Example concept:

```conf
host    source_rep    repuser    <subscriber-ip>/32    scram-sha-256
```

The exact entry depends on the database, user and network architecture.

PostgreSQL's logical replication quick setup demonstrates granting appropriate host access to a replication user. ([PostgreSQL][12])

---

# 36. Create a Dedicated Replication User

Example concept:

```sql
CREATE ROLE repuser
WITH LOGIN REPLICATION
PASSWORD 'StrongPassword';
```

Grant the required table privileges:

```sql
GRANT SELECT ON TABLE
    public.test_rep,
    public.test_rep_other
TO repuser;
```

Using the `postgres` superuser for application replication connections should generally be avoided.

---

# 37. Create Source Database

Example:

```sql
CREATE DATABASE source_rep;
```

Connect:

```sql
\c source_rep
```

Create tables:

```sql
CREATE TABLE public.test_rep
(
    id   integer PRIMARY KEY,
    name varchar(100)
);

CREATE TABLE public.test_rep_other
(
    id   integer PRIMARY KEY,
    name varchar(100)
);
```

---

# 38. Insert Test Data

Example:

```sql
INSERT INTO public.test_rep
SELECT g, 'data' || g
FROM generate_series(1,100) AS g;

INSERT INTO public.test_rep_other
SELECT g, 'data' || g
FROM generate_series(1,100) AS g;
```

Validate:

```sql
SELECT count(*)
FROM public.test_rep;

SELECT count(*)
FROM public.test_rep_other;
```

Expected:

```text
100
100
```

---

# 39. Create Publication

```sql
CREATE PUBLICATION mypub
FOR TABLE
    public.test_rep,
    public.test_rep_other;
```

Creating the publication itself does not start replication. It defines what can be consumed by subscribers. ([PostgreSQL][13])

---

# 40. Validate Publication

```sql
SELECT *
FROM pg_publication;
```

Check published tables:

```sql
SELECT *
FROM pg_publication_tables;
```

This is cleaner than manually joining catalog OIDs when the goal is simply to list publication tables.

---

# 41. Create Subscriber Schema

Logical replication does **not** automatically create the required tables. ([PostgreSQL][8])

One clean method is to export the source schema.

Publisher:

```bash
pg_dump \
-d source_rep \
--schema-only \
--no-owner \
-f source_rep_schema.sql
```

Copy the file to the subscriber and restore it:

```bash
psql \
-d target_rep \
-f source_rep_schema.sql
```

Verify:

```sql
\dt
```

---

# 42. Create Subscription

Connect to the subscriber database:

```sql
\c target_rep
```

Example:

```sql
CREATE SUBSCRIPTION mysub
CONNECTION
'host=<publisher-ip>
 port=5432
 dbname=source_rep
 user=repuser
 password=<password>'
PUBLICATION mypub;
```

By default, PostgreSQL normally creates the associated replication slot on the publisher and starts synchronization. ([PostgreSQL][10])

---

# 43. Initial Data Synchronization

When a subscription is created, PostgreSQL normally:

1. Creates/uses the logical replication slot.
2. Starts an apply worker.
3. Starts table synchronization workers.
4. Copies existing table data.
5. Continues streaming incremental changes.

Each subscription normally uses one replication slot, with additional temporary slots potentially required during initial table synchronization. ([PostgreSQL][9])

---

# 44. Validate Replicated Data

Subscriber:

```sql
SELECT count(*)
FROM public.test_rep;

SELECT count(*)
FROM public.test_rep_other;
```

Then insert additional data on the publisher:

```sql
INSERT INTO public.test_rep
VALUES (101, 'data101');
```

Subscriber:

```sql
SELECT *
FROM public.test_rep
WHERE id = 101;
```

The new row should appear after replication applies the transaction.

---

# 45. Monitoring – Publisher

Use:

```sql
SELECT *
FROM pg_stat_replication;
```

This can show WAL sender activity associated with connected logical replication subscribers.

Also check:

```sql
SELECT *
FROM pg_replication_slots;
```

This is important because an inactive logical replication slot can retain WAL.

---

# 46. Monitoring – Subscriber

Use:

```sql
SELECT *
FROM pg_stat_subscription;
```

PostgreSQL documents `pg_stat_subscription` as the primary subscription monitoring view. It can contain one or more workers depending on synchronization and parallel apply activity. ([PostgreSQL][14])

Also use:

```sql
SELECT *
FROM pg_stat_subscription_stats;
```

PostgreSQL 18 provides conflict statistics through this view.

---

# 47. Important Monitoring Correction

The original notes stated:

> If `pg_stat_subscription` has no rows, recreate the subscription.

Do **not** immediately recreate the subscription.

Zero active worker rows can occur because the subscription is:

* Disabled
* Crashed
* Unable to connect
* Failing due to authentication
* Failing because of schema mismatch
* Experiencing another replication error

PostgreSQL explicitly states that disabled or crashed subscriptions can have zero active workers. ([PostgreSQL][14])

Always investigate first.

---

# 48. Logical Replication Troubleshooting

Recommended order:

### Step 1 – Check subscription

```sql
SELECT *
FROM pg_subscription;
```

### Step 2 – Check active workers

```sql
SELECT *
FROM pg_stat_subscription;
```

### Step 3 – Check subscription statistics

```sql
SELECT *
FROM pg_stat_subscription_stats;
```

### Step 4 – Check publisher slot

```sql
SELECT *
FROM pg_replication_slots;
```

### Step 5 – Check publisher connections

```sql
SELECT *
FROM pg_stat_replication;
```

### Step 6 – Check PostgreSQL logs

The server logs often contain the exact apply-worker or connection error.

---

# 49. DDL Changes

DDL is not automatically replicated.

For example:

```sql
ALTER TABLE public.test_rep
ADD COLUMN email text;
```

on the publisher does not automatically alter the subscriber table.

If subsequent replicated data requires that column and the subscriber schema is incompatible, replication can fail until the subscriber schema is corrected. ([PostgreSQL][8])

---

# 50. Best Practice for Additive Schema Changes

For compatible additive changes, a safer sequence is often:

```text
1. Apply compatible DDL to subscriber.
2. Apply DDL to publisher.
3. Validate replication.
```

The exact sequence depends on the application and DDL operation.

---

# 51. Adding New Tables to a Publication

Publisher:

```sql
ALTER PUBLICATION mypub
ADD TABLE public.new_table;
```

Subscriber:

Create the compatible target table first.

Then:

```sql
ALTER SUBSCRIPTION mysub
REFRESH PUBLICATION;
```

PostgreSQL requires a publication refresh for newly added publication tables to become known to an existing subscription. ([PostgreSQL][15])

---

# 52. REFRESH PUBLICATION

Command:

```sql
ALTER SUBSCRIPTION mysub
REFRESH PUBLICATION;
```

By default:

```text
copy_data = true
```

for newly subscribed tables. ([PostgreSQL][16])

Explicit form:

```sql
ALTER SUBSCRIPTION mysub
REFRESH PUBLICATION
WITH (copy_data = true);
```

---

# 53. Do Not Recreate Subscriptions Unnecessarily

The original notes repeatedly recommended dropping and recreating subscriptions when problems occur.

This should be a **last-resort action**.

Dropping and recreating a subscription loses synchronization information and can require data to be synchronized again. ([PostgreSQL][9])

First:

1. Read PostgreSQL logs.
2. Identify the root cause.
3. Fix schema/authentication/data conflict.
4. Enable/refresh subscription where appropriate.
5. Recreate only when genuinely required.

---

# 54. Logical Replication Conflicts

Replication can stop when incoming changes violate subscriber constraints.

Examples include:

* Duplicate primary/unique keys
* Permission failures
* Schema mismatch
* Constraint violations
* Certain row-level-security situations

PostgreSQL 18 provides enhanced conflict logging and statistics. ([PostgreSQL][17])

Therefore, avoid uncontrolled writes to subscriber tables that are also being populated through logical replication.

---

# 55. Publication for All Tables

To publish all eligible tables in the database:

```sql
CREATE PUBLICATION alltables
FOR ALL TABLES;
```

Important:

> A publication belongs to one database.

`FOR ALL TABLES` means all eligible tables in that database, not every database in the PostgreSQL cluster. ([PostgreSQL][7])

---

# 56. Publishing Tables in a Schema

Modern PostgreSQL also supports publishing all eligible tables in selected schemas.

Example:

```sql
CREATE PUBLICATION app_pub
FOR TABLES IN SCHEMA app;
```

This means the **tables in the schema** are included in the publication.

It does **not** mean that the schema definition itself is replicated.

---

# 57. Cross-Version Logical Replication

Logical replication can be used between different PostgreSQL major versions. ([PostgreSQL][5])

Example:

```text
PostgreSQL 16 Publisher
          |
          | Logical Replication
          v
PostgreSQL 18 Subscriber
```

This capability makes logical replication useful in certain low-downtime migration and upgrade designs.

However, extension compatibility, data types, schema, application behavior and version-specific features must be validated carefully.

---

# 58. Correcting the pg_dumpall Upgrade Note

The original class notes stated:

```text
pg_dumpall -S
```

means:

> all database DDLs + users + access rights.

This is incorrect.

In `pg_dumpall`:

```text
-S username
```

means:

> specify the superuser username used in a particular restore-related context.

It does **not** mean schema-only. ([PostgreSQL][18])

Schema-only is:

```bash
pg_dumpall --schema-only
```

or:

```bash
pg_dumpall -s
```

Global objects can be dumped using:

```bash
pg_dumpall --globals-only
```

Roles only:

```bash
pg_dumpall --roles-only
```

This is an important command-line correction.

---

# 59. Logical Replication Upgrade Concept

A simplified logical-replication migration pattern can be:

```text
PostgreSQL 16
Publisher
     |
     | Logical Replication
     v
PostgreSQL 18
Subscriber
```

High-level process:

```text
1. Install PostgreSQL 18.
2. Validate application/extension compatibility.
3. Prepare target schema.
4. Configure logical replication.
5. Create publication.
6. Create subscription.
7. Allow initial synchronization.
8. Monitor replication lag.
9. Prevent incompatible DDL during synchronization.
10. Plan application cutover.
11. Stop/redirect application writes.
12. Verify final synchronization.
13. Point application to PostgreSQL 18.
14. Perform post-cutover validation.
```

PostgreSQL 18 contains dedicated documentation for logical replication and upgrade scenarios. ([PostgreSQL][19])

---

# 60. Logical Replication Is Not Automatically HA

Logical replication should not automatically be described as a complete HA/DR solution.

It provides data replication at the logical level, but an HA architecture additionally needs planning for:

* Automatic failover
* Client routing
* DDL synchronization
* Sequence synchronization
* Conflict management
* Monitoring
* RPO
* RTO
* Failback
* Application behavior

Physical streaming replication is commonly used when the requirement is a byte-level standby for PostgreSQL HA/DR.

Logical replication is particularly valuable when selective replication, cross-version replication or data integration is required.

---

# 61. Security Best Practices

For production logical replication:

* Do not use `trust` unless there is a very specific controlled reason.
* Use SCRAM authentication.
* Use a dedicated replication role.
* Restrict network access.
* Restrict `pg_hba.conf` by source address.
* Avoid using the PostgreSQL superuser for application replication.
* Protect subscription connection strings.
* Use TLS where required.
* Monitor replication slots.
* Apply least privilege.

---

# 62. Key DBA Monitoring Queries

### Current connections

```sql
SELECT *
FROM pg_stat_activity;
```

### Publisher replication connections

```sql
SELECT *
FROM pg_stat_replication;
```

### Replication slots

```sql
SELECT *
FROM pg_replication_slots;
```

### Publications

```sql
SELECT *
FROM pg_publication;
```

### Published tables

```sql
SELECT *
FROM pg_publication_tables;
```

### Subscriptions

```sql
SELECT *
FROM pg_subscription;
```

### Subscription workers

```sql
SELECT *
FROM pg_stat_subscription;
```

### Subscription conflict statistics

```sql
SELECT *
FROM pg_stat_subscription_stats;
```

---

# 63. Key Corrections from the Original Class Notes

The following corrections are important for interview and production knowledge:

| Original Understanding                                                   | Correct Understanding                                                                                                     |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| pgBench tells exact maximum connections                                  | pgBench helps benchmark workloads; capacity must be established against an SLA                                            |
| Scale factor 1 is fine for any load                                      | Scale factor must represent the workload; default test should normally have scale ≥ tested clients                        |
| Increase `shared_buffers` and `work_mem` to fix latency                  | Tune memory only after workload analysis                                                                                  |
| pgBadger is PostgreSQL core                                              | pgBadger is a third-party PostgreSQL log analyzer                                                                         |
| pgBadger 12.4 should be installed                                        | 12.4 is old; use a supported/current release                                                                              |
| Enable `log_statement='all'` and `log_min_duration_statement=0` together | Avoid excessive/overlapping logging for normal pgBadger use                                                               |
| Logical replication requires a primary key on every table                | UPDATE/DELETE require suitable replica identity; PK is the normal default                                                 |
| Subscriber column order must match                                       | Columns are matched by name                                                                                               |
| Subscriber data types must always be identical                           | Compatible conversion can be possible in text mode                                                                        |
| Logical replication requires two different servers                       | Same-cluster configurations are possible with special slot handling                                                       |
| `wal_keep_size` is mandatory                                             | It is not a basic logical-replication requirement                                                                         |
| Subscriber also always needs `wal_level=logical`                         | Required on a node acting as a logical publisher; subscriber-specific worker settings are the main receiving-side concern |
| `trust` should be used for replication                                   | Prefer dedicated users and SCRAM in production                                                                            |
| No `pg_stat_subscription` row means recreate subscription                | Diagnose the subscription and logs first                                                                                  |
| Any DDL means recreate subscription                                      | Correct schema compatibility first; recreation is not automatically required                                              |
| `pg_dumpall -S` means schema-only                                        | `-s` is schema-only; uppercase `-S` specifies a superuser name                                                            |
| Logical replication automatically provides full DR                       | It is data replication; HA/DR requires additional architecture                                                            |

---

# 64. PostgreSQL DBA Learning Sequence

For class preparation, study these topics in this order:

```text
PostgreSQL Architecture
        ↓
Installation & Configuration
        ↓
Users / Roles / Security
        ↓
Database & Schema Administration
        ↓
Backup & Recovery
        ↓
VACUUM / Autovacuum / ANALYZE
        ↓
Performance Monitoring
        ↓
Query & Index Tuning
        ↓
pgBench
        ↓
PostgreSQL Logging
        ↓
pgBadger
        ↓
Physical Streaming Replication
        ↓
Logical Replication
        ↓
Replication Slots
        ↓
High Availability
        ↓
PgBouncer
        ↓
Partitioning
        ↓
Row-Level Security
        ↓
Transaction Isolation / ACID
        ↓
Extensions / dblink / FDW
        ↓
Major-Version Upgrade
        ↓
AWS RDS / Aurora PostgreSQL
        ↓
Migration
        ↓
Production Troubleshooting
```

---

# 65. Quick Interview Revision

### What is pgBench?

PostgreSQL's built-in benchmarking utility used to measure transaction throughput and latency under controlled concurrent workloads.

### What is TPS?

Transactions processed per second.

### What does `-c` mean?

Number of concurrent pgBench clients.

### What does `-T` mean?

Benchmark duration in seconds.

### What does `-S` mean?

Built-in SELECT-only benchmark.

### What does `-f` mean?

Run a custom pgBench script.

### What is pgBadger?

A third-party PostgreSQL log-analysis and reporting tool.

### What is logical replication?

PostgreSQL's publish/subscribe replication mechanism that logically replicates selected data changes.

### Publisher?

Database containing the publication.

### Subscriber?

Database containing the subscription.

### Required publisher parameter?

```conf
wal_level = logical
```

### Are DDL changes replicated?

No.

### Are sequence states replicated?

No.

### Are PostgreSQL large objects replicated?

No.

### Are INSERT/UPDATE/DELETE/TRUNCATE supported?

Yes, subject to publication configuration and replica-identity requirements.

### Can different PostgreSQL major versions participate?

Yes, which is one of logical replication's important use cases.

### Main publisher monitoring view?

```sql
pg_stat_replication
```

### Main subscriber monitoring view?

```sql
pg_stat_subscription
```

### How do we see logical replication slots?

```sql
pg_replication_slots
```

### How do we add newly published tables to an existing subscription?

```sql
ALTER SUBSCRIPTION subscription_name
REFRESH PUBLICATION;
```

### Does logical replication automatically create subscriber tables?

No.

### Does a table always need a primary key?

Not necessarily, but UPDATE and DELETE require a suitable replica identity.

---

# 66. Final DBA Takeaway

### pgBench

Use pgBench to **measure PostgreSQL performance under a controlled workload**.

Do not treat one TPS number as the server's absolute capacity.

---

### pgBadger

Use pgBadger to **convert PostgreSQL logs into useful performance and operational reports**.

It is a third-party tool, not PostgreSQL core.

---

### Logical Replication

Use logical replication when you need:

```text
Selective replication
Cross-version migration
Data distribution
Reporting/analytics feeds
Cross-platform PostgreSQL replication
Controlled data sharing
```

Remember:

```text
Publication = what to send
Subscription = what to receive
Replication Slot = WAL position/retention mechanism
Replica Identity = how changed rows are identified
```

For production administration, always monitor:

```text
pg_stat_replication
pg_replication_slots
pg_stat_subscription
pg_stat_subscription_stats
PostgreSQL server logs
```

A few corrections in these notes are especially important: **`pg_dumpall -S` was being interpreted incorrectly, `wal_keep_size` was presented as mandatory for logical replication when it is not, a primary key was presented as universally mandatory when the actual requirement is replica identity for UPDATE/DELETE, and subscription recreation was being recommended too quickly.** Those have all been corrected using PostgreSQL 18 documentation. ([PostgreSQL][18])

Official references used include the PostgreSQL documentation for [pgBench](https://www.postgresql.org/docs/18/pgbench.html?utm_source=chatgpt.com), [Logical Replication](https://www.postgresql.org/docs/18/logical-replication.html?utm_source=chatgpt.com), [Logical Replication Restrictions](https://www.postgresql.org/docs/18/logical-replication-restrictions.html?utm_source=chatgpt.com), [Logical Replication Configuration](https://www.postgresql.org/docs/18/logical-replication-config.html?utm_source=chatgpt.com), and [pg_dumpall](https://www.postgresql.org/docs/18/app-pg-dumpall.html?utm_source=chatgpt.com). For pgBadger-specific behavior, I used the project's current documentation; PostgreSQL's site also records the pgBadger 13.2 release from December 2025. ([GitHub][3])

[1]: https://www.postgresql.org/docs/current/pgbench.html"PostgreSQL: Documentation: 18: pgbench"
[2]: https://www.postgresql.org/docs/18/runtime-config-resource.html"PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[3]: https://github.com/darold/pgbadger"GitHub - darold/pgbadger: A fast PostgreSQL Log Analyzer · GitHub"
[4]: https://github.com/darold/pgbadger/releases"Releases · darold/pgbadger · GitHub"
[5]: https://www.postgresql.org/docs/18/logical-replication.html"PostgreSQL: Documentation: 18: Chapter 29. Logical Replication"
[6]: https://www.postgresql.org/docs/18/logical-replication-architecture.html"PostgreSQL: Documentation: 18: 29.9. Architecture"
[7]: https://www.postgresql.org/docs/18/logical-replication-publication.html"PostgreSQL: Documentation: 18: 29.1. Publication"
[8]: https://www.postgresql.org/docs/18/logical-replication-restrictions.html"PostgreSQL: Documentation: 18: 29.8. Restrictions"
[9]: https://www.postgresql.org/docs/18/logical-replication-subscription.html"PostgreSQL: Documentation: 18: 29.2. Subscription"
[10]: https://www.postgresql.org/docs/18/sql-createsubscription.html"PostgreSQL: Documentation: 18: CREATE SUBSCRIPTION"
[11]: https://www.postgresql.org/docs/18/logical-replication-config.html"PostgreSQL: Documentation: 18: 29.12. Configuration Settings"
[12]: https://www.postgresql.org/docs/18/logical-replication-quick-setup.html"PostgreSQL: Documentation: 18: 29.14. Quick Setup"
[13]: https://www.postgresql.org/docs/18/sql-createpublication.html"PostgreSQL: Documentation: 18: CREATE PUBLICATION"
[14]: https://www.postgresql.org/docs/18/logical-replication-monitoring.html"PostgreSQL: Documentation: 18: 29.10. Monitoring"
[15]: https://www.postgresql.org/docs/18/sql-alterpublication.html"PostgreSQL: Documentation: 18: ALTER PUBLICATION"
[16]: https://www.postgresql.org/docs/18/sql-altersubscription.html"PostgreSQL: Documentation: 18: ALTER SUBSCRIPTION"
[17]: https://www.postgresql.org/docs/18/logical-replication-conflicts.html"PostgreSQL: Documentation: 18: 29.7. Conflicts"
[18]: https://www.postgresql.org/docs/18/app-pg-dumpall.html"PostgreSQL: Documentation: 18: pg_dumpall"
[19]: https://www.postgresql.org/docs/18/logical-replication-upgrade.html"PostgreSQL: Documentation: 18: 29.13. Upgrade"
