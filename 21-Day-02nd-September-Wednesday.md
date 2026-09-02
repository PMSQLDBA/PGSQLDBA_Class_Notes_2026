# PostgreSQL 18 – Physical Streaming Replication, Failover, Failback and `pg_rewind`

**Class Date:** 02 September 2026 – Wednesday
**PostgreSQL Version:** PostgreSQL 18
**Topic:** Streaming Replication, Synchronous and Asynchronous Replication, Multi-Standby, Cascading Replication, Replication Lag, Failover, Failback and `pg_rewind`

---

# 1. Lab Environment

The lab contains three PostgreSQL servers.

| Server |      IP Address | Initial Role        |
| ------ | --------------: | ------------------- |
| DB1    |  `172.31.0.184` | Primary             |
| DB2    | `172.31.13.188` | Standby / Replica 1 |
| DB3    |  `172.31.13.99` | Standby / Replica 2 |

Initial architecture:

```text
Application
     |
     v
+------------------------+
| DB1 - Primary          |
| 172.31.0.184           |
+------------------------+
             |
             | WAL Streaming
             v
+------------------------+
| DB2 - Standby          |
| 172.31.13.188          |
+------------------------+
```

Later the lab will be extended to:

```text
                    +-----------------------+
                    | DB2 - Standby 1       |
                    | 172.31.13.188         |
                    +-----------------------+
                           ^
                           |
                           |
+----------------------+   |
| DB1 - Primary        |---+
| 172.31.0.184         |
+----------------------+   |
                           |
                           v
                    +-----------------------+
                    | DB3 - Standby 2       |
                    | 172.31.13.99          |
                    +-----------------------+
```

---

# 2. What is PostgreSQL Replication?

Replication is the process of maintaining one or more copies of PostgreSQL data on other PostgreSQL servers.

In physical streaming replication:

1. Applications perform transactions on the primary server.
2. PostgreSQL records those changes in the Write-Ahead Log.
3. WAL records are transmitted from the primary to the standby.
4. The standby receives the WAL.
5. The standby replays the WAL.
6. The standby therefore maintains a physical copy of the primary PostgreSQL cluster.

Architecture:

```text
Application
    |
    | INSERT / UPDATE / DELETE
    v
+------------------+
| Primary          |
+------------------+
         |
         | Generate WAL
         v
      pg_wal
         |
         | WAL Sender
         v
      Network
         |
         | WAL Receiver
         v
+------------------+
| Standby          |
+------------------+
         |
         | WAL Replay
         v
 Standby Database
```

Important processes:

```text
Primary   : WAL Sender
Standby   : WAL Receiver
```

---

# 3. Primary and Standby Terminology

For a basic replication setup, a minimum of two PostgreSQL servers is required.

### Server 1

Common names:

```text
Primary
Source
Primary Server
```

### Server 2

Common names:

```text
Standby
Replica
Secondary
Destination
```

PostgreSQL official documentation mainly uses:

```text
Primary
Standby
```

---

# 4. Physical Replication vs Logical Replication

PostgreSQL provides different replication approaches.

## Physical Streaming Replication

Physical replication copies changes at the WAL/storage level.

It generally replicates the complete PostgreSQL cluster.

Typical use cases:

```text
High Availability
Disaster Recovery
Read-only standby
Failover
Infrastructure redundancy
```

Example:

```text
PostgreSQL 18 Primary
        |
        | Physical WAL
        v
PostgreSQL 18 Standby
```

## Logical Replication

Logical replication replicates logical data changes.

It can be used for:

```text
Selected tables
Migration
Major-version upgrade scenarios
Data distribution
Selective replication
```

Physical replication and logical replication therefore solve different problems.

---

# 5. High Availability

High Availability means keeping database services available even when one server fails.

Example:

```text
DB1 Primary
     |
     X Failure
     |
     v
DB2 Standby
     |
     | Promote
     v
DB2 New Primary
```

Streaming replication is an important building block for HA.

However:

> PostgreSQL physical replication alone does not provide complete automatic failover.

PostgreSQL provides mechanisms to promote a standby, but failure detection, fencing, leader election and connection routing normally require an external HA solution.

---

# 6. Disaster Recovery

Disaster Recovery focuses on restoring services and data after major failures.

Replication can be part of DR.

However:

```text
Replication != Backup
```

A better DR design combines:

```text
Streaming Replication
        +
Base Backups
        +
WAL Archiving
        +
Point-in-Time Recovery
        +
Off-site Backup Copies
```

This protects against more than server failure.

Examples:

```text
Hardware failure
Storage failure
Site failure
Accidental deletion
Data corruption
Operational mistakes
```

---

# 7. Replication Modes

The two important replication commit behaviors are:

```text
1. Asynchronous Replication
2. Synchronous Replication
```

By default, PostgreSQL streaming replication is asynchronous.

---

# 8. Asynchronous Replication

In asynchronous replication, the primary normally does not wait for the standby before acknowledging the transaction commit to the application.

Flow:

```text
Client
  |
  | Transaction
  v
Primary
  |
  | Generate WAL
  |
  | Commit locally
  v
Client receives success
  |
  +----------> WAL sent to Standby
```

Sequence:

1. User executes a write transaction.
2. PostgreSQL generates WAL.
3. Transaction commits on the primary.
4. Primary responds to the client.
5. WAL is transmitted to the standby.
6. Standby receives the WAL.
7. Standby replays it.

### Advantages

* Lower transaction latency.
* Standby outage does not normally stop primary writes.
* Better performance than synchronous replication in many workloads.

### Disadvantage

If the primary crashes before some committed WAL reaches the standby, recent transactions may be lost during failover.

Therefore:

```text
Asynchronous Replication
        |
        v
Possible Replication Lag
        |
        v
Possible RPO > 0
```

---

# 9. Testing Asynchronous Replication

Check replication:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Expected:

```text
sync_state = async
```

Stop DB2:

```bash
sudo systemctl stop postgresql-18
```

Create a database on DB1:

```sql
CREATE DATABASE asynctest;
```

Expected:

```text
CREATE DATABASE
```

The transaction succeeds even though DB2 is unavailable.

Start DB2:

```bash
sudo systemctl start postgresql-18
```

After it catches up:

```sql
\l
```

The database should appear on DB2.

---

# 10. Synchronous Replication

With synchronous replication, transactions can wait for acknowledgement from the required synchronous standby before the client receives commit success.

Simplified architecture:

```text
Client
  |
  | COMMIT
  v
DB1 Primary
  |
  | WAL
  v
DB2 Synchronous Standby
  |
  | WAL flushed
  |
  | ACK
  v
DB1 Primary
  |
  v
Client receives success
```

This provides stronger durability.

The trade-off is additional transaction latency.

---

# 11. `synchronous_commit`

Important values include:

```text
on
remote_apply
remote_write
local
off
```

## `on`

The transaction waits until the WAL record is durably flushed on the selected synchronous standby.

## `remote_write`

The primary waits until the WAL has been written to the standby operating system.

## `remote_apply`

The transaction waits until the WAL has actually been replayed on the standby.

This can give stronger read-after-write behavior on the standby but may add more latency.

---

# 12. Synchronous Standby Failure Behavior

Suppose the environment is:

```text
DB1 Primary
     |
     | Synchronous
     v
DB2 Standby
```

If DB2 goes down:

```text
DB2 unavailable
       |
       v
DB1 still running
       |
       v
Synchronous commits wait
```

Important correction:

The primary does **not** automatically stop.

Instead, transactions waiting for synchronous acknowledgement may remain blocked.

The application may therefore experience what looks like a database hang.

---

# 13. Why Multiple Standbys are Useful

Instead of:

```text
1 Primary
+
1 Synchronous Standby
```

a better design may use:

```text
1 Primary
+
1 Synchronous Standby
+
1 Potential Synchronous Standby
```

Example:

```text
             +--> DB2
             |    SYNC
DB1 PRIMARY -|
             |
             +--> DB3
                  POTENTIAL
```

If DB2 becomes unavailable, DB3 can become synchronous depending on the configured synchronous standby rules.

---

# 14. Server Preparation

Configure hostnames.

### DB1

```bash
sudo hostnamectl set-hostname db1
```

### DB2

```bash
sudo hostnamectl set-hostname db2
```

### DB3

```bash
sudo hostnamectl set-hostname db3
```

Verify:

```bash
hostname
hostnamectl
```

---

# 15. Configure `/etc/hosts`

Add the following on all three servers:

```text
172.31.0.184    db1
172.31.13.188   db2
172.31.13.99    db3
```

Verify:

```bash
getent hosts db1
getent hosts db2
getent hosts db3
```

Test connectivity:

```bash
ping db1
ping db2
ping db3
```

Test PostgreSQL port where required:

```bash
nc -zv db1 5432
nc -zv db2 5432
nc -zv db3 5432
```

---

# 16. PostgreSQL Version Requirement

Physical streaming replication should use the same PostgreSQL major version.

Example:

```text
DB1 = PostgreSQL 18
DB2 = PostgreSQL 18
DB3 = PostgreSQL 18
```

For a stable lab and production setup, keeping the same minor release is strongly recommended.

---

# 17. Correct Physical Replication Requirements

The following are **not strict PostgreSQL requirements**:

```text
Same CPU
Same RAM
Same OS version
Same PostgreSQL port
```

The standby may have different hardware.

However, the standby should have sufficient:

```text
CPU
RAM
Disk I/O
Network bandwidth
Storage performance
```

to keep up with the primary workload.

A weak standby can create significant replication lag.

---

# 18. Prepare DB1 as Primary

Primary:

```text
DB1
172.31.0.184
```

Important PostgreSQL settings:

```conf
listen_addresses = '*'
wal_keep_size = '1GB'
wal_log_hints = on
```

Also verify:

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
SHOW wal_keep_size;
SHOW listen_addresses;
SHOW wal_log_hints;
```

For physical replication:

```text
wal_level = replica
```

is sufficient.

---

# 19. Purpose of `wal_keep_size`

`wal_keep_size` specifies the minimum amount of old WAL files to retain in `pg_wal` for standby servers.

Example:

```conf
wal_keep_size = '1GB'
```

It can help a standby that temporarily falls behind.

However:

```text
wal_keep_size
     !=
Replication Slot
```

Replication slots provide stronger WAL retention control for specific consumers.

---

# 20. Purpose of `wal_log_hints`

```conf
wal_log_hints = on
```

is not required just to enable streaming replication.

It is useful in this class because later we use:

```text
pg_rewind
```

`pg_rewind` requires either:

```text
wal_log_hints = on
```

or data checksums enabled.

---

# 21. Restart Primary

Some settings require restart.

```bash
sudo systemctl restart postgresql-18
```

Check:

```bash
sudo systemctl status postgresql-18
```

Validate:

```sql
SHOW listen_addresses;
SHOW wal_keep_size;
SHOW wal_log_hints;
SHOW wal_level;
```

---

# 22. Create Replication User

On DB1:

```sql
CREATE ROLE replicator
WITH LOGIN REPLICATION PASSWORD 'StrongPasswordHere';
```

Verify:

```sql
\du replicator
```

Expected attribute:

```text
Replication
```

A dedicated replication account is better than using the `postgres` superuser for normal replication connectivity.

---

# 23. Configure `pg_hba.conf`

Allow DB2:

```conf
host    replication    replicator    172.31.13.188/32    scram-sha-256
```

Allow DB3:

```conf
host    replication    replicator    172.31.13.99/32     scram-sha-256
```

Reload:

```sql
SELECT pg_reload_conf();
```

or:

```bash
sudo systemctl reload postgresql-18
```

---

# 24. Important Security Correction

The original class notes used:

```text
trust
```

for replication authentication.

Example:

```conf
host replication repuser 172.31.x.x/32 trust
```

This may be acceptable only in a temporary controlled lab.

For production, prefer secure authentication such as:

```text
scram-sha-256
```

---

# 25. Create Physical Replication Slot

On DB1:

```sql
SELECT pg_create_physical_replication_slot('db2_slot');
```

Verify:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status
FROM pg_replication_slots;
```

Expected:

```text
slot_name = db2_slot
slot_type = physical
```

---

# 26. What Does a Replication Slot Do?

A physical replication slot tells PostgreSQL that a standby still needs certain WAL.

This prevents required WAL from being removed too early.

However:

```text
Replication Slot
      !=
Replication can never fail
```

If the standby stays offline:

```text
Standby offline
      |
      v
WAL retained
      |
      v
pg_wal grows
      |
      v
Possible disk-space problem
```

Replication slots therefore require monitoring.

---

# 27. Important Replication Slot Correction

Replication slots were **not introduced in PostgreSQL 10**.

Therefore the statement:

```text
Replication slots started from PostgreSQL 10
```

is incorrect.

Replication slots existed before PostgreSQL 10.

---

# 28. Build DB2 Standby

Stop PostgreSQL on DB2:

```bash
sudo systemctl stop postgresql-18
```

Ensure the target data directory is empty.

Run:

```bash
pg_basebackup \
  -h 172.31.0.184 \
  -U replicator \
  -D /var/lib/pgsql/18/data \
  -X stream \
  -P \
  -R \
  -S db2_slot
```

---

# 29. Class Command Using `-X fetch`

The original class command used:

```bash
pg_basebackup \
  -D /var/lib/pgsql/18/data \
  -h 172.31.0.184 \
  -U replicator \
  -X fetch \
  -P \
  -R
```

This is valid.

However, the important difference is:

```text
-X fetch
```

collects required WAL after the backup.

The primary must retain enough WAL until the backup finishes.

For a streaming standby, this class note uses:

```text
-X stream
```

because WAL is streamed while the backup is running.

---

# 30. Important `pg_basebackup` Options

```text
-h
```

Primary hostname/IP.

```text
-U
```

Replication user.

```text
-D
```

Destination PostgreSQL data directory.

```text
-P
```

Displays progress.

```text
-R
```

Creates standby configuration.

```text
-X stream
```

Streams WAL during the base backup.

```text
-S db2_slot
```

Uses the specified replication slot.

---

# 31. What Does `-R` Create?

Using:

```text
-R
```

causes `pg_basebackup` to create:

```text
standby.signal
```

and write connection settings into:

```text
postgresql.auto.conf
```

Example:

```conf
primary_conninfo =
'host=172.31.0.184 port=5432 user=replicator ...'

primary_slot_name = 'db2_slot'
```

---

# 32. Meaning of `standby.signal`

The file:

```text
standby.signal
```

tells PostgreSQL to start in standby/recovery mode.

Check:

```bash
ls -l /var/lib/pgsql/18/data/standby.signal
```

---

# 33. Start DB2

```bash
sudo systemctl start postgresql-18
```

Verify:

```bash
sudo systemctl status postgresql-18
```

---

# 34. Confirm DB2 is a Standby

Connect:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

Also:

```sql
SHOW transaction_read_only;
```

Expected:

```text
on
```

---

# 35. Test Write on Standby

On DB2:

```sql
CREATE DATABASE repdb;
```

Expected:

```text
ERROR:
cannot execute CREATE DATABASE in a read-only transaction
```

This confirms DB2 is not writable while operating as a physical standby.

---

# 36. Verify Replication Processes

On DB1:

```bash
ps -ef | grep sender
```

Expected process:

```text
walsender
```

On DB2:

```bash
ps -ef | grep receiver
```

Expected:

```text
walreceiver
```

---

# 37. Check Replication Using `pg_stat_replication`

On DB1:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_priority,
    sync_state
FROM pg_stat_replication;
```

For asynchronous replication:

```text
state      = streaming
sync_state = async
```

---

# 38. Check Replication Slot

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status
FROM pg_replication_slots;
```

Expected for active DB2:

```text
slot_name = db2_slot
active    = true
```

---

# 39. Check WAL Receiver on DB2

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

This view shows information such as:

```text
status
sender_host
sender_port
written_lsn
flushed_lsn
latest_end_lsn
```

---

# 40. Replication Lag

Replication lag is the difference between WAL generated by the primary and WAL received/replayed by the standby.

Example:

```text
Primary WAL Position
        |
        | Gap
        |
        v
Standby Replay Position
```

---

# 41. Measure Replication Lag in Bytes

On DB1:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(
        pg_wal_lsn_diff(sent_lsn, replay_lsn)
    ) AS replay_lag
FROM pg_stat_replication;
```

---

# 42. LSN Values

Important fields include:

```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
```

Meaning:

```text
sent_lsn
   |
   | WAL sent by primary
   v

write_lsn
   |
   | WAL written by standby
   v

flush_lsn
   |
   | WAL flushed by standby
   v

replay_lsn
   |
   | WAL replayed by standby
   v
```

If these values are identical at a specific moment, the standby was caught up at that observation point.

It does not guarantee permanent zero lag.

---

# 43. Common Reasons for Replication Lag

## 1. Heavy Write Workload

Large amounts of:

```text
INSERT
UPDATE
DELETE
COPY
Bulk Load
```

generate significant WAL.

## 2. Large Maintenance Operations

Examples:

```text
VACUUM FULL
CREATE INDEX
REINDEX
Large DDL operations
```

may generate significant WAL.

## 3. Network Latency

Examples:

```text
High latency
Low bandwidth
Packet loss
Network congestion
```

## 4. Slow Standby Disk

A standby must write and replay WAL fast enough.

## 5. CPU Pressure

High CPU utilization can slow recovery.

## 6. Heavy Read Workload on Standby

Read queries can consume resources that WAL replay also needs.

## 7. Hot Standby Conflicts

Some standby queries can conflict with recovery operations.

## 8. Cascading Replication

For:

```text
DB1 → DB2 → DB3
```

DB3 depends on both upstream links.

---

# 44. Reducing Replication Lag

Possible actions include:

```text
Increase network bandwidth
Improve storage performance
Increase CPU capacity
Reduce heavy write workload
Schedule large maintenance jobs carefully
Monitor WAL generation
Tune standby resources
Investigate long-running standby queries
Monitor replay conflicts
```

Do not assume that simply adding RAM always fixes replication lag.

The actual bottleneck should be identified first.

---

# 45. Convert Async Replication to Sync

Check:

```sql
SHOW synchronous_standby_names;
```

For a basic lab:

```conf
synchronous_standby_names = '*'
```

Reload:

```sql
SELECT pg_reload_conf();
```

Verify:

```sql
SHOW synchronous_standby_names;
```

Check:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_priority,
    sync_state
FROM pg_stat_replication;
```

Expected DB2:

```text
sync_state = sync
```

---

# 46. Test Synchronous Behavior

Stop DB2:

```bash
sudo systemctl stop postgresql-18
```

On DB1:

```sql
CREATE DATABASE synctest;
```

The command may remain waiting.

Reason:

```text
DB1 generated WAL
       |
       v
Waiting for required ACK
       |
       X
DB2 unavailable
```

Start DB2:

```bash
sudo systemctl start postgresql-18
```

Once the required acknowledgement is received, the waiting commit can complete.

---

# 47. Production-Style Synchronous Standby Configuration

Instead of:

```conf
synchronous_standby_names = '*'
```

use explicit standby names.

Example:

```conf
synchronous_standby_names =
'FIRST 1 (db2, db3)'
```

This means:

```text
db2 = preferred synchronous standby
db3 = potential synchronous standby
```

---

# 48. `FIRST` Synchronous Replication

Example:

```conf
synchronous_standby_names =
'FIRST 1 (db2, db3)'
```

Architecture:

```text
             DB2
             SYNC
              ^
              |
DB1 PRIMARY --+
              |
              v
             DB3
          POTENTIAL
```

If DB2 becomes unavailable:

```text
DB3 -> SYNC
```

---

# 49. Important Standby Selection Correction

The statement:

```text
The standby with the least lag automatically becomes sync
```

is not correct.

With:

```text
FIRST
```

PostgreSQL uses priority/order configured in:

```text
synchronous_standby_names
```

---

# 50. Quorum Synchronous Replication

PostgreSQL also supports:

```conf
synchronous_standby_names =
'ANY 1 (db2, db3)'
```

This means acknowledgement from either DB2 or DB3 is sufficient.

Example with three standbys:

```conf
synchronous_standby_names =
'ANY 2 (db2, db3, db4)'
```

The primary waits for acknowledgements from any two eligible standbys.

Remember:

```text
FIRST = Priority based
ANY   = Quorum based
```

---

# 51. Multi-Standby Replication

Now add DB3.

Architecture:

```text
               +---------- DB2
               |
DB1 PRIMARY ---+
               |
               +---------- DB3
```

DB1 streams WAL independently to both standby servers.

---

# 52. Prepare DB3

DB3:

```text
172.31.13.99
```

Configure hostname:

```bash
sudo hostnamectl set-hostname db3
```

Configure `/etc/hosts`:

```text
172.31.0.184    db1
172.31.13.188   db2
172.31.13.99    db3
```

Install PostgreSQL 18.

---

# 53. Allow DB3 on DB1

Add:

```conf
host    replication    replicator    172.31.13.99/32    scram-sha-256
```

Reload:

```sql
SELECT pg_reload_conf();
```

---

# 54. Create DB3 Replication Slot

On DB1:

```sql
SELECT pg_create_physical_replication_slot('db3_slot');
```

Verify:

```sql
SELECT *
FROM pg_replication_slots;
```

---

# 55. Build DB3

```bash
pg_basebackup \
  -h 172.31.0.184 \
  -U replicator \
  -D /var/lib/pgsql/18/data \
  -X stream \
  -P \
  -R \
  -S db3_slot
```

Start:

```bash
sudo systemctl start postgresql-18
```

---

# 56. Verify Multi-Standby Replication

On DB1:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

You should see two rows:

```text
DB2
DB3
```

---

# 57. Sync + Potential Standby Test

With:

```conf
synchronous_standby_names =
'FIRST 1 (db2, db3)'
```

Expected:

```text
DB2 = sync
DB3 = potential
```

Stop DB2:

```bash
sudo systemctl stop postgresql-18
```

Check DB1:

```sql
SELECT
    application_name,
    client_addr,
    sync_state
FROM pg_stat_replication;
```

DB3 should become:

```text
sync
```

Start DB2 again and verify the configured priority behavior.

---

# 58. Cascading Replication

Multi-standby configuration generates separate WAL streaming traffic from DB1 to every directly connected standby.

Another option is cascading replication.

Architecture:

```text
DB1 PRIMARY
     |
     | WAL
     v
DB2 STANDBY
     |
     | WAL
     v
DB3 CASCADING STANDBY
```

DB3 receives WAL from DB2 instead of directly from DB1.

---

# 59. Benefit of Cascading Replication

Direct architecture:

```text
DB1
 | \
 |  \
 v   v
DB2 DB3
```

Cascading architecture:

```text
DB1
 |
 v
DB2
 |
 v
DB3
```

This can reduce:

```text
Direct replication connections to DB1
Network traffic from DB1
WAL sender load on DB1
```

However, it adds another dependency.

If DB2 is unavailable, DB3 loses its upstream source until it is redirected or DB2 recovers.

---

# 60. Configure DB2 for Cascading Replication

Allow DB3:

```conf
host    replication    replicator    172.31.13.99/32    scram-sha-256
```

Reload:

```sql
SELECT pg_reload_conf();
```

Create DB3 slot on DB2:

```sql
SELECT pg_create_physical_replication_slot('db3_slot');
```

---

# 61. Point DB3 to DB2

DB3 should use:

```conf
primary_conninfo =
'host=172.31.13.188 port=5432 user=replicator ...'
```

and:

```conf
primary_slot_name = 'db3_slot'
```

Now:

```text
DB3 upstream = DB2
```

---

# 62. Verify Cascading Replication

On DB1:

```sql
SELECT
    client_addr,
    state
FROM pg_stat_replication;
```

DB1 should show DB2.

On DB2:

```sql
SELECT
    client_addr,
    state
FROM pg_stat_replication;
```

DB2 should show DB3.

Final topology:

```text
DB1
PRIMARY
 |
 | walsender
 v
DB2
STANDBY
 |
 | walsender
 v
DB3
CASCADING STANDBY
```

---

# 63. Native PostgreSQL Failover Behavior

Return to:

```text
DB1 Primary
     |
     v
DB2 Standby
```

If DB1 fails:

```text
DB1 = Down
DB2 = Still standby
```

DB2 does **not automatically become primary** just because native physical replication is configured.

Check:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

A write still fails.

---

# 64. Manual Failover

Before promoting DB2, ensure DB1 cannot accept writes.

This is called fencing.

Then promote DB2:

```bash
/usr/pgsql-18/bin/pg_ctl \
  -D /var/lib/pgsql/18/data \
  promote
```

or:

```sql
SELECT pg_promote();
```

Verify:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

DB2 is now the primary.

---

# 65. Functional Test After Promotion

On DB2:

```sql
CREATE DATABASE failover1;
CREATE DATABASE failover2;
CREATE DATABASE failover3;
```

These commands now succeed because DB2 is writable.

Topology:

```text
DB1
OLD PRIMARY
DOWN

DB2
NEW PRIMARY
READ + WRITE
```

---

# 66. Split-Brain

After DB2 is promoted, do not simply start DB1 and allow applications to connect to it.

Otherwise:

```text
        DB1
    Old Primary
       / 
      /
Applications
      \
       \
        DB2
    New Primary
```

Both servers may accept writes independently.

This is called:

```text
Split Brain
```

It can cause severe data divergence and data loss.

---

# 67. Failover

Failover is generally an **unplanned** role change caused by primary failure.

Example:

```text
DB1 Primary
     X
     |
     v
DB2 promoted
```

---

# 68. Switchover

Switchover is a **planned** role change.

Example:

```text
DB1 Primary
     |
     | Planned maintenance
     v
DB2 Primary
```

Typical use cases:

```text
Patching
Server maintenance
Hardware maintenance
Controlled HA testing
Datacenter work
```

---

# 69. Failback

Failback means returning toward the preferred/original topology after the failed server has been recovered and resynchronized.

Example:

```text
Original:

DB1 Primary
DB2 Standby

After failover:

DB1 Failed
DB2 New Primary

After recovery:

DB2 Primary
DB1 Standby

Later controlled switchover:

DB1 Primary
DB2 Standby
```

---

# 70. Why Old Primary Cannot Simply Become Primary Again

After failover:

```text
DB2
NEW PRIMARY
```

may have received new transactions.

DB1 does not contain those transactions.

Therefore:

```text
DB1 data
   !=
DB2 data
```

DB1 must first be synchronized.

---

# 71. Option 1 – Rebuild Using `pg_basebackup`

One option is:

```text
Stop DB1
     |
     v
Remove/prepare old data directory
     |
     v
Take fresh pg_basebackup from DB2
     |
     v
Build DB1 as standby
```

This is reliable but copies the entire database cluster.

For very large databases, it can take significant time.

Do not assume a fixed duration such as:

```text
1 TB = 5–6 hours
```

because the duration depends on:

```text
Storage throughput
Network throughput
CPU
Compression
Database size
Tablespaces
Concurrent workload
```

---

# 72. Option 2 – `pg_rewind`

PostgreSQL provides:

```text
pg_rewind
```

This can synchronize a diverged PostgreSQL data directory with another copy of the same cluster.

Typical scenario:

```text
DB1 Old Primary
       \
        \ Diverged
         \
        DB2 New Primary
```

Run `pg_rewind` against DB1 to bring it back to the new timeline.

---

# 73. How `pg_rewind` Works

`pg_rewind` finds the point where the two clusters diverged.

It then synchronizes the target using changed blocks and required files instead of copying the complete cluster.

Therefore, when only a smaller part of a large database changed:

```text
pg_rewind
```

can be considerably faster than:

```text
Full pg_basebackup
```

---

# 74. `pg_rewind` Prerequisites

The target cluster must have either:

```text
Data checksums enabled
```

or:

```conf
wal_log_hints = on
```

The old primary should be stopped cleanly before rewind.

The source server must be reachable.

Required access must be allowed.

---

# 75. Correct `pg_rewind` Workflow

Assume:

```text
DB1 = Old Primary
DB2 = Current Primary
```

Correct high-level process:

```text
1. Fence DB1
2. Keep DB1 stopped
3. Confirm DB2 is current primary
4. Allow required connection from DB1 to DB2
5. Run pg_rewind on DB1
6. Configure DB1 as standby of DB2
7. Start DB1
8. Verify recovery mode
9. Verify streaming replication
10. Perform controlled switchover later if DB1 must regain primary role
```

---

# 76. Run `pg_rewind`

On DB1:

```bash
/usr/pgsql-18/bin/pg_rewind \
  --target-pgdata=/var/lib/pgsql/18/data \
  --source-server="host=172.31.13.188 port=5432 dbname=postgres user=postgres"
```

Typical output:

```text
servers diverged at WAL location ...
rewinding from last common checkpoint ...
Done!
```

---

# 77. Important `pg_rewind` Correction

After `pg_rewind`, do **not** immediately start DB1 as an independent primary.

Instead:

```text
DB1
Old Primary
     |
     | pg_rewind
     v
Configure as standby
     |
     v
Follow DB2
```

---

# 78. Rejoin DB1 as Standby

Configure appropriate:

```conf
primary_conninfo =
'host=172.31.13.188 port=5432 user=replicator ...'
```

If using a slot:

```conf
primary_slot_name = 'db1_slot'
```

Ensure:

```text
standby.signal
```

exists.

Start DB1:

```bash
sudo systemctl start postgresql-18
```

Verify:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

---

# 79. Topology After Rewind

```text
DB2
NEW PRIMARY
172.31.13.188
      |
      | WAL Streaming
      v
DB1
REJOINED STANDBY
172.31.0.184
```

At this point DB1 is again synchronized.

---

# 80. Returning DB1 to Primary

If DB1 is the preferred production primary because it has better hardware, do not simply promote it while DB2 is still primary.

Perform a controlled switchover.

Conceptually:

```text
DB2 Primary
DB1 Standby
      |
      | Confirm caught up
      |
      v
Stop application writes
      |
      v
Controlled role change
      |
      v
DB1 Primary
DB2 Standby
```

---

# 81. Three-Node HA Cluster Concept

Manual failover introduces operational downtime.

A production HA solution normally automates:

```text
Failure detection
Leader election
Standby promotion
Split-brain protection
Node rejoin
Connection routing
Health checking
Monitoring
Failover
Switchover
```

Conceptual design:

```text
                +-------------------+
                | Quorum / Control  |
                +-------------------+
                       |
            +----------+----------+
            |                     |
            v                     v
      +-----------+         +-----------+
      | Primary   |  WAL    | Standby   |
      | PostgreSQL| ------> | PostgreSQL|
      +-----------+         +-----------+
            ^                     ^
            |                     |
            +-------- HA ---------+
```

---

# 82. Automatic Failover Goal

Desired behavior:

```text
Application
     |
     v
Stable Endpoint
     |
     v
DB1 Primary
     |
     X Failure
     |
     v
Failure detected
     |
     v
DB2 promoted
     |
     v
Traffic redirected
     |
     v
Application continues
```

The application should ideally continue using the same stable endpoint.

---

# 83. External PostgreSQL HA Solutions

Common PostgreSQL ecosystem solutions include:

```text
Patroni
repmgr
EDB Failover Manager
pg_auto_failover
HAProxy
Consul
etcd
Keepalived
```

Important:

These are not all PostgreSQL core components.

Therefore their detailed behavior should be validated against their respective official documentation rather than PostgreSQL core documentation.

---

# 84. PostgreSQL Core vs HA Tool Responsibility

PostgreSQL provides:

```text
Physical streaming replication
Synchronous replication
Replication slots
Hot standby
Cascading replication
Standby promotion
pg_rewind
WAL archiving
PITR
```

External HA tooling may provide:

```text
Automatic failure detection
Automatic promotion
Leader election
Distributed consensus
Fencing
Virtual IP
Proxy routing
Node rejoin automation
```

---

# 85. Complete Class Lab Sequence

The complete class flow is:

```text
1. Create three servers
       ↓
2. Configure hostnames
       ↓
3. Configure /etc/hosts
       ↓
4. Install PostgreSQL 18
       ↓
5. Prepare DB1 as primary
       ↓
6. Configure listen_addresses
       ↓
7. Configure WAL settings
       ↓
8. Configure wal_log_hints
       ↓
9. Create replication user
       ↓
10. Configure pg_hba.conf
       ↓
11. Create DB2 replication slot
       ↓
12. Run pg_basebackup from DB2
       ↓
13. Configure primary_slot_name
       ↓
14. Start DB2
       ↓
15. Verify physical replication
       ↓
16. Verify pg_stat_replication
       ↓
17. Verify pg_stat_wal_receiver
       ↓
18. Test read-only standby
       ↓
19. Test asynchronous replication
       ↓
20. Convert to synchronous replication
       ↓
21. Test synchronous behavior
       ↓
22. Add DB3
       ↓
23. Build multi-standby replication
       ↓
24. Configure sync + potential standby
       ↓
25. Test synchronous standby failure
       ↓
26. Configure cascading replication
       ↓
27. Monitor replication lag
       ↓
28. Study causes of replication lag
       ↓
29. Return to DB1 + DB2 topology
       ↓
30. Simulate primary failure
       ↓
31. Manually promote DB2
       ↓
32. Understand split-brain risk
       ↓
33. Understand failover
       ↓
34. Understand failback
       ↓
35. Understand switchover
       ↓
36. Run pg_rewind on old DB1
       ↓
37. Rejoin DB1 as standby
       ↓
38. Understand 3-node HA architecture
       ↓
39. Understand automatic failover
```

---

# 86. Important Monitoring Queries

## Check if server is primary or standby

```sql
SELECT pg_is_in_recovery();
```

Interpretation:

```text
false = Primary
true  = Standby / Recovery
```

## Primary replication status

```sql
SELECT *
FROM pg_stat_replication;
```

## Replication slot status

```sql
SELECT *
FROM pg_replication_slots;
```

## Standby WAL receiver

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

## Synchronous standby configuration

```sql
SHOW synchronous_standby_names;
```

## Synchronous commit configuration

```sql
SHOW synchronous_commit;
```

## WAL level

```sql
SHOW wal_level;
```

## WAL retention

```sql
SHOW wal_keep_size;
```

---

# 87. Recommended Replication Monitoring Query

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(sent_lsn, replay_lsn)
    ) AS replay_lag
FROM pg_stat_replication;
```

This provides a cleaner replication health view than:

```sql
SELECT * FROM pg_stat_replication;
```

---

# 88. Replication Slot Monitoring Query

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status,
    safe_wal_size,
    inactive_since
FROM pg_replication_slots;
```

Important things to watch:

```text
Inactive slot
Large WAL retention
Invalidated slot
Growing pg_wal
```

---

# 89. Key Corrections from the Raw Class Notes

The following points should be remembered in their corrected form.

| Raw Concept                                                | Correct PostgreSQL Behavior                                       |
| ---------------------------------------------------------- | ----------------------------------------------------------------- |
| Physical replication needs identical RAM/CPU               | Not required                                                      |
| Physical replication needs identical OS version            | Not a strict PostgreSQL requirement                               |
| Ports must be identical                                    | Not required                                                      |
| `wal_log_hints` is required for replication                | No; mainly relevant here for `pg_rewind`                          |
| Replication slots started in PostgreSQL 10                 | Incorrect                                                         |
| Slot guarantees replication will never break               | Incorrect                                                         |
| Lowest-lag standby becomes sync                            | Incorrect                                                         |
| Sync standby failure brings primary down                   | Primary normally stays running; required synchronous commits wait |
| Same LSN means permanently zero lag                        | It only means caught up at that instant                           |
| After `pg_rewind`, old primary can simply start as primary | Unsafe; normally rejoin it as standby first                       |
| Replication equals DR backup                               | Incorrect                                                         |
| Native streaming replication provides automatic failover   | It does not provide a complete automatic failover solution        |

---

# 90. Final Architecture Reference

## One-to-One Replication

```text
DB1 PRIMARY
     |
     | WAL
     v
DB2 STANDBY
```

## Multi-Standby Replication

```text
               +----> DB2 STANDBY
               |
DB1 PRIMARY ---+
               |
               +----> DB3 STANDBY
```

## Synchronous + Potential

```text
               +----> DB2 SYNC
               |
DB1 PRIMARY ---+
               |
               +----> DB3 POTENTIAL
```

## Cascading Replication

```text
DB1 PRIMARY
     |
     v
DB2 STANDBY
     |
     v
DB3 STANDBY
```

## Failover

```text
DB1 PRIMARY
     X

DB2 STANDBY
     |
     | Promote
     v
DB2 NEW PRIMARY
```

## Failback Preparation

```text
DB2 NEW PRIMARY
       |
       | pg_rewind + replication
       v
DB1 REJOINED STANDBY
```

---

# 91. Final Points to Remember

1. PostgreSQL physical streaming replication uses WAL to maintain standby servers.
2. Streaming replication is asynchronous by default.
3. Asynchronous replication gives lower transaction latency but may allow some data loss during failover.
4. Synchronous replication provides stronger durability but adds commit latency.
5. If the only required synchronous standby fails, synchronous commits can wait.
6. `pg_basebackup` is commonly used to initialize a physical standby.
7. `-R` prepares the base backup to start as a standby.
8. `standby.signal` tells PostgreSQL to operate in standby mode.
9. `primary_conninfo` defines how the standby connects to the upstream server.
10. `primary_slot_name` identifies the physical replication slot the standby should use.
11. Replication slots help retain WAL needed by standbys.
12. Replication slots must be monitored because an inactive slot can cause large WAL accumulation.
13. `pg_stat_replication` is one of the main monitoring views on the primary.
14. `pg_stat_wal_receiver` is useful on the standby.
15. `pg_is_in_recovery()` is one of the simplest ways to determine the server role.
16. A physical standby is normally read-only while it remains in recovery.
17. Multi-standby replication allows a primary to stream WAL to several standbys.
18. `FIRST` synchronous replication is priority based.
19. `ANY` synchronous replication is quorum based.
20. Cascading replication allows one standby to provide WAL to another standby.
21. Replication lag can be caused by workload, storage, network, CPU and recovery conflicts.
22. Native PostgreSQL replication does not automatically promote a standby after primary failure.
23. Manual promotion can be done using `pg_ctl promote` or `pg_promote()`.
24. The old primary must be fenced after failover to prevent split brain.
25. Failover is usually an unplanned role change.
26. Switchover is usually a planned role change.
27. Failback means returning toward the preferred topology after the failed node has been recovered.
28. `pg_rewind` can efficiently resynchronize a diverged former primary when its prerequisites are satisfied.
29. After `pg_rewind`, the old primary should normally rejoin as a standby first.
30. Automatic HA requires additional failure-detection, coordination and connection-routing mechanisms.
31. Physical replication should never replace normal backups.
32. Production environments should combine replication with backup, WAL archiving and PITR.

This is now the **combined master version** of both portions of your class notes, properly ordered from basic replication concepts through the 3-node HA/automatic failover discussion.
