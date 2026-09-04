# PostgreSQL High Availability Using pg_auto_failover — Class Notes

**Class Date:** 03rd September 2026 — Thursday
**Topic:** PostgreSQL High Availability, `pg_auto_failover`, Failover/Switchover, Connection URIs, Synchronous Replication, Maintenance, and Database Upgrades

For PostgreSQL core behavior, the notes below are cross-checked against the current PostgreSQL 18 documentation. PostgreSQL 18.6 is the current PostgreSQL 18 minor release as of the class date; 

PostgreSQL 19 is still in beta, so PostgreSQL 18 is the appropriate current production reference. ([PostgreSQL][1])

> **Important distinction:** `pg_auto_failover` is **not a PostgreSQL core feature**.

PostgreSQL provides streaming replication, standby promotion, synchronous replication, `pg_rewind`, libpq multi-host connections, and related HA building blocks.
`pg_auto_failover` is an external HA solution that orchestrates these PostgreSQL capabilities. 
Therefore, commands such as `pg_autoctl` cannot be validated as PostgreSQL-core commands from PostgreSQL.org.

---

## 1. Why Do We Need a PostgreSQL HA Tool?

PostgreSQL provides the underlying replication technology required to build an HA environment:

* Primary/standby architecture
* WAL-based streaming replication
* Hot standby
* Synchronous and asynchronous replication
* Standby promotion
* `pg_promote()`
* `pg_ctl promote`
* `pg_rewind`
* Multi-host libpq connections

However, PostgreSQL core does **not by itself provide a complete automatic failover manager** that continuously decides which standby should become primary.

PostgreSQL documentation explicitly describes promotion during failover but also warns that the old primary must be prevented from returning as another writable primary. 

This fencing/STONITH requirement is important to prevent split-brain and possible data loss. ([PostgreSQL][2])

This is why external HA managers are commonly used.

### HA tools discussed in class

| Tool                  | Main Purpose                                  | PostgreSQL Core? |
| --------------------- | --------------------------------------------- | ---------------- |
| Patroni               | PostgreSQL HA orchestration                   | No               |
| repmgr                | Replication and failover management           | No               |
| EFM                   | EnterpriseDB Failover Manager                 | No               |
| pg_auto_failover      | PostgreSQL failover orchestration             | No               |
| Streaming Replication | WAL replication                               | **Yes**          |
| `pg_rewind`           | Re-synchronize a diverged PostgreSQL instance | **Yes**          |

PostgreSQL streaming replication itself is asynchronous by default. Synchronous replication can be enabled when stronger durability is required. ([PostgreSQL][3])

---

# 2. pg_auto_failover Architecture

The class architecture uses **four servers**:

```text
                       +----------------------+
                       |       MONITOR        |
                       |    pg_auto_failover  |
                       +----------+-----------+
                                  |
                    monitors cluster state
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v

       +-------------+     +-------------+     +-------------+
       |     DB1     |     |     DB2     |     |     DB3     |
       |   PRIMARY   | --> |  SECONDARY  |     |  SECONDARY  |
       |    R/W      | WAL |     R/O     |     |     R/O     |
       +-------------+     +-------------+     +-------------+

              PostgreSQL Streaming Replication
```

### Example class configuration

| Server  | Role                |   Example IP |
| ------- | ------------------- | -----------: |
| monitor | HA Monitor          |   172.31.5.4 |
| db1     | Initial Primary     |  172.31.6.20 |
| db2     | Standby/Secondary 1 | 172.31.8.185 |
| db3     | Standby/Secondary 2 | 172.31.8.172 |

The IP addresses above are **lab examples**, not PostgreSQL requirements.

---

# 3. Role of the Monitor

The monitor acts as the controller for the `pg_auto_failover` formation.

Conceptually it:

1. Maintains information about registered PostgreSQL nodes.
2. Receives health/state information.
3. Coordinates failover decisions.
4. Determines the desired state of nodes.
5. Coordinates promotion/demotion.
6. Helps coordinate planned switchover.
7. Tracks failover candidates.

The monitor itself is **not the application database primary**.

It is part of the HA control plane.

---

# 4. PostgreSQL Primary and Standby Architecture

The database nodes use PostgreSQL physical streaming replication underneath the HA manager.

PostgreSQL streaming replication works by having the standby connect to the primary and receive WAL records as they are generated. ([PostgreSQL][3])

Conceptually:

```text
                    WAL
PRIMARY --------------------------------> STANDBY 1
   |
   +------------------------------------> STANDBY 2
                    WAL
```

The primary handles normal writes.

Standbys continuously replay WAL received from the primary.

---

# 5. Basic Server Preparation

## Step 1 — Configure Hostnames

Example:

```bash
sudo hostnamectl hostname monitor
```

```bash
sudo hostnamectl hostname db1
```

```bash
sudo hostnamectl hostname db2
```

```bash
sudo hostnamectl hostname db3
```

Validate:

```bash
hostname
```

---

# 6. Configure Name Resolution

If DNS is not available in the lab, `/etc/hosts` can be used.

Example:

```text
172.31.5.4    monitor
172.31.6.20   db1
172.31.8.185  db2
172.31.8.172  db3
```

Keep the entries consistent across all four servers.

Validate:

```bash
getent hosts monitor
getent hosts db1
getent hosts db2
getent hosts db3
```

---

# 7. Network and Security Requirements

All required communication must be allowed between:

```text
Monitor <----> Database nodes
DB1     <----> DB2
DB1     <----> DB3
DB2     <----> DB3
Application --> Database nodes
```

In AWS, this means checking:

* VPC routing
* Subnets
* Security Groups
* Network ACLs where applicable
* OS firewall
* PostgreSQL listening address
* PostgreSQL `pg_hba.conf`

A Security Group and `pg_hba.conf` solve different problems.

```text
AWS Security Group
        |
        v
Can network traffic reach the server?
        |
        v
PostgreSQL pg_hba.conf
        |
        v
Is this PostgreSQL client allowed to connect/authenticate?
```

---

# 8. Avoid Broad `pg_hba.conf` Rules

The original notes contain examples such as:

```text
0.0.0.0/0
```

That is useful temporarily in isolated labs, but it should **not be the normal production configuration**.

Prefer specific application/database subnets or individual hosts.

Example:

```text
host    all          postgres    172.31.6.20/32     scram-sha-256
host    replication  repl_user   172.31.8.185/32    scram-sha-256
host    replication  repl_user   172.31.8.172/32    scram-sha-256
```

Similarly, the class examples use:

```text
trust
```

extensively.

`trust` is convenient for a controlled lab but should not be treated as the production authentication standard.

---

# 9. Install PostgreSQL and pg_auto_failover

The uploaded notes use PGDG RPM repositories and examples for PostgreSQL 15 and PostgreSQL 17. 

The important correction is:

> Do not mix PostgreSQL 15 commands, PostgreSQL 17 paths, and PostgreSQL 18 packages within one installation procedure.

Pick one PostgreSQL major version and keep:

```text
PostgreSQL server version
PostgreSQL client version
contrib package
extension binaries
PATH
PGDATA
pg_auto_failover package
```

consistent.

For example, a PostgreSQL 17 environment should not use:

```text
/usr/pgsql-15/bin/
```

for some nodes and:

```text
/usr/pgsql-17/bin/
```

for others without a specific upgrade reason.

---

# 10. PostgreSQL Environment Variables

Typical lab configuration:

```bash
export PGDATA=/var/lib/pgsql/17/data
export PATH=/usr/pgsql-17/bin:$PATH
export PGPORT=5432
```

Reload:

```bash
source ~/.bash_profile
```

Validate:

```bash
which postgres
which psql
which pg_autoctl

postgres --version
psql --version
pg_autoctl --version
```

---

# 11. Create the Monitor

The class uses:

```bash
pg_autoctl create monitor \
  --ssl-self-signed \
  --auth trust
```

After initialization, start the `pg_autoctl` process according to the selected service-management design.

The class lab used:

```bash
nohup pg_autoctl run &
```

Validate:

```bash
jobs
```

and:

```bash
pg_autoctl show uri
```

---

# 12. Production Note About `nohup`

The class uses:

```bash
nohup pg_autoctl run &
```

This is fine for demonstrating the process.

For a production HA implementation, the HA agent should normally be supervised by an operating-system service manager rather than depending on an interactive shell/background process.

The design objective is:

```text
Server restart
     |
     v
HA service automatically starts
     |
     v
Node automatically rejoins HA management
```

---

# 13. Build the First Database Node

The first PostgreSQL database node becomes the initial primary.

Conceptually:

```bash
pg_autoctl create postgres \
  --dbname autodb \
  --monitor '<monitor-connection-string>'
```

After successful creation:

```bash
pg_autoctl run
```

The node registers with the monitor.

Architecture:

```text
Monitor
   |
   |
   +------ DB1
           PRIMARY
```

---

# 14. Add the First Standby

Run the node-creation process on DB2 using the same monitor formation.

Conceptually:

```bash
pg_autoctl create postgres \
  --dbname autodb \
  --monitor '<monitor-connection-string>'
```

Then start its HA agent.

Architecture becomes:

```text
Monitor
   |
   +-------- DB1
   |         PRIMARY
   |
   +-------- DB2
             SECONDARY
```

The standby obtains the required PostgreSQL data and joins the streaming-replication topology.

---

# 15. Add the Second Standby

Repeat the process for DB3.

Final class architecture:

```text
                         MONITOR
                            |
              +-------------+-------------+
              |             |             |
              v             v             v

             DB1           DB2           DB3
           PRIMARY       SECONDARY     SECONDARY
             |               ^             ^
             +------WAL------+-------------+
```

---

# 16. Check Cluster State

A key administrative command from the class is:

```bash
pg_autoctl show state
```

Typical output concept:

```text
Name     Host       Connection     Reported State
-------------------------------------------------
node_1   db1:5432   read-write     primary
node_2   db2:5432   read-only      secondary
node_3   db3:5432   read-only      secondary
```

The important items are:

* Host
* Port
* Current connection state
* Reported state
* Assigned state

---

# 17. Validate PostgreSQL Streaming Replication

On the primary:

```sql
SELECT
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
    sync_state
FROM pg_stat_replication;
```

Healthy physical replicas should normally show:

```text
state = streaming
```

Depending on the HA configuration:

```text
sync_state = sync
sync_state = quorum
sync_state = async
```

PostgreSQL officially exposes replication state through `pg_stat_replication`. ([PostgreSQL][3])

---

# 18. Multi-Host Application Connection URI

This is one of the most useful PostgreSQL-native concepts in the class.

Instead of configuring an application with only:

```text
db1
```

libpq supports multiple hosts.

Example:

```text
postgresql://db1:5432,db2:5432,db3:5432/autodb
```

The client can then try multiple database hosts.

---

# 19. Read-Write Connection URI

For applications that must connect to a writable server:

```text
postgresql://db1:5432,db2:5432,db3:5432/autodb?target_session_attrs=read-write
```

`target_session_attrs=read-write` tells libpq that the selected session must accept read-write transactions.

This behavior is officially supported by PostgreSQL libpq. ([PostgreSQL][4])

Therefore:

```text
DB1 Primary
    |
    X fails
    |
DB2 promoted
    |
Application reconnects
    |
libpq searches host list
    |
DB2 satisfies read-write requirement
```

The application does not need to know which physical server is currently primary.

---

# 20. Read-Only Connection URI

A separate connection can be used when the workload specifically requires a read-only server:

```text
postgresql://db1:5432,db2:5432,db3:5432/autodb?target_session_attrs=read-only
```

PostgreSQL supports `read-only` as a `target_session_attrs` value. ([PostgreSQL][4])

However, there is an important correction:

> `target_session_attrs=read-only` is not itself a sophisticated read-balancing mechanism.

It selects an acceptable read-only connection from the host list. It should not be described as automatically distributing queries evenly across all replicas.

---

# 21. Application Retry Is Still Required

The class correctly identifies this important point.

A failover can terminate an existing connection.

For example:

```text
Application
    |
    | existing session
    v
Old Primary
    X
    |
    +---- failover ----> New Primary
```

The old TCP/database session does not magically move to another PostgreSQL server.

Therefore the application or connection pool should support:

* Connection retry
* Reconnection
* Appropriate connection timeout
* Transaction retry where safe
* Multi-host connection configuration

After reconnecting, libpq can use the host list and `target_session_attrs` to find an appropriate server. ([PostgreSQL][4])

This is a very important production HA concept.

---

# 22. Switchover

A **switchover** is a planned role transition.

Class command:

```bash
pg_autoctl perform switchover
```

Conceptually:

```text
BEFORE

DB1
PRIMARY
  |
  +------ DB2
  |       SECONDARY
  |
  +------ DB3
          SECONDARY


AFTER SWITCHOVER

DB1
SECONDARY

DB2
PRIMARY

DB3
SECONDARY
```

A switchover is useful for:

* OS maintenance
* PostgreSQL minor updates
* Hardware maintenance
* Planned testing
* HA validation

PostgreSQL's own documentation recommends regular planned switching as a useful way to test failover readiness. ([PostgreSQL][2])

---

# 23. Failover

A **failover** is an unplanned role transition caused by primary failure.

Example:

```text
DB1 PRIMARY
     X
   failure
     |
     v
Monitor detects/coordinates failure
     |
     v
Eligible standby promoted
     |
     v
DB2 PRIMARY
```

The remaining nodes should follow the new primary according to the HA manager's state machine.

---

# 24. Failover vs Switchover

| Switchover                          | Failover                         |
| ----------------------------------- | -------------------------------- |
| Planned                             | Unplanned                        |
| Current primary is normally healthy | Primary is unavailable/unhealthy |
| Used for maintenance/testing        | Used for failure recovery        |
| Controlled role transition          | Emergency HA transition          |
| Lower operational risk              | Greater operational risk         |

---

# 25. Split-Brain Protection

This point needs to be added to the original class notes.

Imagine:

```text
DB1 believes: "I am primary"

DB2 believes: "I am primary"
```

If both accept writes, the databases can diverge.

This is called **split brain**.

PostgreSQL documentation specifically states that after promotion, the old primary needs a mechanism telling it that it is no longer primary; this is commonly called STONITH/fencing. ([PostgreSQL][2])

A proper HA solution must therefore consider:

```text
failure detection
+
promotion
+
demotion
+
fencing
+
old-primary rejoin
```

not just promotion.

---

# 26. Rejoining the Failed Primary

Once the failed server is repaired, it must not simply resume accepting writes as the old primary.

Conceptually:

```text
Old Primary
     |
     | recover
     v
Synchronize with current Primary
     |
     v
Rejoin as Standby
```

PostgreSQL provides `pg_rewind` as one mechanism that can help synchronize a diverged old primary with the new primary. PostgreSQL specifically mentions it as useful when re-preparing the old primary after failover. ([PostgreSQL][2])

In a managed `pg_auto_failover` setup, the HA manager handles its own rejoin workflow, so follow its supported procedure rather than manually forcing PostgreSQL roles.

---

# 27. Candidate Priority

The class demonstrates:

```bash
pg_autoctl set node candidate-priority 90
```

Candidate priority controls the HA manager's preference when selecting promotion candidates.

Conceptually:

```text
DB1 = Primary

DB2 priority = 90
DB3 priority = 50

If DB1 fails:

DB2 is preferred
```

This setting belongs to `pg_auto_failover`, not PostgreSQL core.

Therefore PostgreSQL.org does not define the semantics of the `candidate-priority` command.

---

# 28. Synchronous Replication

The class environment shows:

```text
number_sync_standbys = 1
```

and an underlying PostgreSQL setting similar to:

```text
synchronous_standby_names =
'ANY 1 (standby1, standby2)'
```

The PostgreSQL part is important.

`ANY` means quorum-based synchronous replication.

For example:

```text
ANY 1 (standby1, standby2)
```

means a commit can proceed after acknowledgment from **any one** eligible synchronous standby. ([PostgreSQL][3])

---

# 29. Why Writes Can Wait When Standbys Are Unavailable

Suppose:

```text
PRIMARY
   |
   +---- Standby1 DOWN
   |
   +---- Standby2 DOWN
```

and PostgreSQL is configured to require a synchronous standby acknowledgment.

The primary may continue running, but transactions waiting for synchronous confirmation can wait indefinitely.

This is expected synchronous-replication behavior, not necessarily a PostgreSQL failure.

PostgreSQL documentation explicitly warns that synchronous commits may never complete if the required synchronous standbys are unavailable. ([PostgreSQL][3])

---

# 30. Changing the Synchronous Standby Requirement

The class uses:

```bash
pg_autoctl set formation number-sync-standbys 0
```

This is a `pg_auto_failover` command.

At the PostgreSQL level, reducing/disabling the synchronous standby requirement means commits no longer wait for the previous number of standby acknowledgments.

This creates a trade-off:

```text
Higher synchronous requirement
        |
        +--> stronger durability
        |
        +--> lower write availability if replicas disappear


Lower/no synchronous requirement
        |
        +--> better write availability
        |
        +--> increased potential data-loss window
```

Do **not** automatically change this setting just because replicas are unavailable.

It should be a documented production decision based on the required **RPO**.

PostgreSQL documentation itself recommends reducing the synchronous requirement only when you cannot maintain the required number of synchronous standbys. ([PostgreSQL][3])

---

# 31. Maintenance Mode

Class command:

```bash
pg_autoctl enable maintenance
```

Maintenance mode is useful when intentionally taking a node out of normal HA participation for activities such as:

* OS patching
* PostgreSQL maintenance
* Hardware maintenance
* Troubleshooting
* Planned restart

Again, this command is a `pg_auto_failover` feature rather than PostgreSQL core.

---

# 32. PostgreSQL SSL/TLS — Correction to Class Notes

The notes mention:

```text
DBA: SSL ON
DB server: ca.cert, root.cert
User receives public cert
```

This needs more precise terminology.

A PostgreSQL server typically uses:

```text
server.crt
server.key
```

For CA-based verification, trusted CA certificate information is also configured.

On the client side, PostgreSQL/libpq supports parameters such as:

```text
sslmode
sslrootcert
sslcert
sslkey
```

PostgreSQL also exposes corresponding environment variables including `PGSSLMODE`, `PGSSLROOTCERT`, `PGSSLCERT`, and `PGSSLKEY`. ([PostgreSQL][5])

---

# 33. Recommended Client SSL Verification

For production environments, stronger verification is generally preferable to merely encrypting the connection.

Conceptually:

```text
Application
     |
     | TLS
     v
PostgreSQL Server
     |
     +--> Certificate chain validation
     +--> Hostname validation
```

For libpq, `sslmode=verify-full` provides certificate-chain and server-hostname verification when correctly configured.

Self-signed certificates may be acceptable for a controlled lab, but production PKI should be designed carefully.

---

# 34. PostgreSQL Major Upgrade

Example:

```text
PostgreSQL 17
      |
      v
PostgreSQL 18
```

This is a **major version upgrade**.

PostgreSQL major versions can contain incompatible system-catalog changes. Therefore simply replacing PostgreSQL binaries is not enough.

Supported approaches include:

```text
pg_dump / pg_restore
pg_upgrade
logical replication
```

PostgreSQL officially states that major upgrades require dump/reload or an upgrade mechanism such as `pg_upgrade`. ([PostgreSQL][6])

---

# 35. Important Correction — Physical Replication Across Major Versions

The class statement is correct in principle:

> PostgreSQL physical streaming replication should not be expected to continue between different major versions.

For example, do **not** design physical streaming replication as:

```text
PG17 Primary
      |
      | physical streaming replication
      v
PG18 Standby
```

For cross-major-version replication requirements, PostgreSQL logical replication is the relevant built-in technology. PostgreSQL explicitly lists replication between different PostgreSQL major versions as a logical-replication use case. ([PostgreSQL][7])

---

# 36. Corrected Major Upgrade Strategy for an HA Cluster

The original diagram suggests:

1. Destroy both replicas.
2. Upgrade monitor.
3. Upgrade primary.
4. Rebuild replicas.

That is **one possible operational approach**, but it should not be presented as PostgreSQL's only or mandatory upgrade method.

PostgreSQL documents a more specific `pg_upgrade` procedure for streaming-replication environments.

A safe high-level workflow is:

```text
1. Read PostgreSQL target-version release notes.

2. Validate extensions and application compatibility.

3. Take and validate backups.

4. Install new PostgreSQL binaries.

5. Run:
   pg_upgrade --check

6. Ensure old standbys are caught up with primary.

7. Stop application writes.

8. Ensure standbys receive the final WAL.

9. Stop old primary and standbys.

10. Upgrade the primary with pg_upgrade.

11. Upgrade/rebuild standbys using a supported method.

12. Restore/validate configuration.

13. Start new primary.

14. Start/rebuild standbys.

15. Validate replication.

16. Run required post-upgrade processing.

17. Re-enable application traffic.

18. Validate HA/failover.
```

PostgreSQL specifically requires streaming standbys to be caught up before the primary upgrade when using the documented standby-upgrade workflow. ([PostgreSQL][8])

---

# 37. Standbys Do Not Necessarily Need `pg_upgrade`

This is an important addition.

For a physical standby, PostgreSQL documents methods where you **do not run `pg_upgrade` independently on each standby**.

With an appropriate `pg_upgrade` strategy, standbys can be synchronized using the documented `rsync` procedure.

Alternatively:

> Recreate the standby after the new primary has been successfully upgraded.

PostgreSQL explicitly says that rebuilding the standbys is an easier supported alternative when the rsync/link-mode procedure is not being used. ([PostgreSQL][8])

This aligns well with the class idea of rebuilding Replica 1 and Replica 2, but the PostgreSQL-supported procedure should be followed.

---

# 38. PostgreSQL Minor Upgrade

Example:

```text
17.10
   |
   v
17.11
```

This is a **minor release update**.

A minor update does **not** require:

```text
pg_upgrade
```

or normally:

```text
pg_dump / pg_restore
```

PostgreSQL says the normal process is:

```text
Stop PostgreSQL
      |
Install updated binaries/packages
      |
Start PostgreSQL
```

Always check the specific release notes because some updates require additional post-update actions. ([PostgreSQL][6])

---

# 39. Minor Upgrade in an HA Cluster

The class proposes a rolling approach:

```text
Replica 2
   ↓
Replica 1
   ↓
Monitor
   ↓
Switchover
   ↓
Old Primary
```

The general principle is good:

> Upgrade standby nodes first and leave the current primary until later to reduce application disruption.

A simplified operational sequence can be:

```text
1. Verify backup and replication health.

2. Put standby DB3 into maintenance.

3. Stop DB3 PostgreSQL/HA service.

4. Install the minor update.

5. Start DB3.

6. Verify it catches up.

7. Repeat for DB2.

8. Patch HA/monitor components according to the
   pg_auto_failover compatibility procedure.

9. Perform planned switchover.

10. Patch the old primary.

11. Start/rejoin it as a standby.

12. Validate the entire cluster.
```

But note:

> The exact rolling-upgrade compatibility of `pg_auto_failover` versions must be verified against the `pg_auto_failover` project's documentation. PostgreSQL.org only validates PostgreSQL's side of this process.

---

# 40. Minor Upgrade Does Not Automatically Mean "Zero Downtime"

The diagram says:

```text
minor version upgrade = no downtime
```

This should be corrected to:

> **A PostgreSQL minor update can potentially be performed with very low application disruption in a correctly designed HA environment, but PostgreSQL itself does not guarantee zero downtime.**

Each individual PostgreSQL instance still has to be restarted after its binaries are updated.

Application availability depends on:

* HA design
* Switchover
* Connection retry
* Connection pool behavior
* Driver behavior
* Transaction retry
* Load balancer/proxy behavior
* HA manager behavior

---

# 41. Major Upgrade Does Not Always Require Long Downtime

The diagram says:

```text
major upgrade = downtime as usual
```

A better statement is:

> Major upgrades require a migration/upgrade process and normally involve some cutover downtime when using `pg_upgrade`, but logical replication can be used to design lower-downtime major-version migrations.

PostgreSQL logical replication explicitly supports replication between different major versions. ([PostgreSQL][7])

---

# 42. PostgreSQL 18 Upgrade Improvement

PostgreSQL 18 introduces an important `pg_upgrade` improvement:

> `pg_upgrade` now retains optimizer statistics.

This can help the upgraded environment reach expected query performance faster after a major upgrade. ([PostgreSQL][9])

Also remember that PostgreSQL 18 enables data checksums by default for newly initialized clusters. This can matter when preparing a PG18 target cluster for `pg_upgrade`. ([PostgreSQL][9])

---

# 43. Read-Write and Read-Only Connectivity Design

A clean application design is:

```text
                 APPLICATION
                     |
          +----------+----------+
          |                     |
          v                     v

     READ-WRITE URI        READ-ONLY URI

 target_session_attrs    target_session_attrs
     =read-write             =read-only

          |                     |
          v                     v

       PRIMARY              acceptable
                            read-only node
```

PostgreSQL officially supports both session-selection modes. ([PostgreSQL][4])

---

# 44. Failover Flow

```text
                    NORMAL

Application
    |
    | read-write URI
    v

 DB1 PRIMARY
   /       \
  / WAL     \ WAL
 v           v
DB2         DB3
Standby     Standby


                 DB1 FAILURE

 DB1
  X
  |
  +---- Monitor/HA manager detects failure
                    |
                    v
             DB2 promoted
                    |
                    v

                  DB2
                PRIMARY
                   |
             +-----+-----+
             |           |
             v           v
            DB3      DB1 after
          Standby     recovery
                      rejoins
```

---

# 45. What the Application Team Needs to Know

The statement from the class that:

> "Developers/application team are nowhere involved even though failover or switchover happens"

is too absolute.

A better production statement is:

> **Applications should not require manual connection-string changes during normal HA failover if multi-host connectivity or another stable endpoint is correctly configured. However, application teams must ensure that the driver/pool supports reconnect and appropriate transaction retry behavior.**

That distinction is important.

HA infrastructure can remove the need for:

```text
"DBA, please give me the new primary IP."
```

but cannot remove the application's responsibility to handle a broken database connection correctly.

---

# 46. HA Does Not Mean Existing Sessions Survive

This is another key interview and production point.

When the primary fails:

```text
Existing DB session
        X
```

After promotion:

```text
New connection
      |
      v
New Primary
```

Failover normally requires clients to reconnect.

The database connection itself is not transferred from one PostgreSQL process/server to another.

---

# 47. Recommended Health Checks

### PostgreSQL role

```sql
SELECT pg_is_in_recovery();
```

Interpretation:

```text
false = Primary
true  = Standby
```

### Server IP

```sql
SELECT inet_server_addr();
```

### Replication

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

### HA state

```bash
pg_autoctl show state
```

### HA settings

```bash
pg_autoctl show settings
```

### Connection URI

```bash
pg_autoctl show uri
```

---

# 48. Recommended Failover Test

A proper lab should test more than just whether a standby becomes primary.

Validate:

```text
1. Insert test data on primary.

2. Verify data on standbys.

3. Record current primary.

4. Simulate primary failure.

5. Verify new primary.

6. Test read-write URI.

7. Test read-only URI.

8. Verify application reconnect.

9. Verify replication topology.

10. Recover old primary.

11. Verify old primary rejoins as standby.

12. Verify no split brain.

13. Perform another controlled switchover.

14. Verify application connectivity again.
```

---

# 49. Key Corrections to the Original Class Notes

The most important corrections are:

| Original idea                                                 | Corrected note                                                                                                   |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| pg_auto_failover is built into PostgreSQL                     | **No. It is an external HA solution.**                                                                           |
| PostgreSQL automatically handles full HA                      | PostgreSQL supplies replication/promotion building blocks; HA orchestration requires additional design/tooling.  |
| `0.0.0.0/0 trust` is normal                                   | Suitable only for tightly controlled labs; use restricted networks and secure authentication in production.      |
| Minor upgrade = no downtime                                   | HA can minimize disruption, but zero downtime is not guaranteed.                                                 |
| Major upgrade always means normal/long downtime               | `pg_upgrade` needs cutover downtime; logical replication can support lower-downtime migration designs.           |
| Physical replication works between major versions             | **No.** Logical replication supports cross-major-version scenarios.                                              |
| Every standby must run `pg_upgrade`                           | Not necessarily; PostgreSQL documents rsync-based standby upgrades or rebuilding standbys.                       |
| Read-only URI provides load balancing                         | Not by itself. It selects a server satisfying the requested session attributes.                                  |
| Failover means application is unaffected                      | Existing connections can break; clients need reconnect/retry logic.                                              |
| Application team has no involvement                           | No manual endpoint change may be required, but application retry/reconnect behavior remains important.           |
| If synchronous standbys fail, immediately set sync count to 0 | This is an RPO/availability decision and should follow an approved procedure.                                    |
| Self-signed SSL is sufficient for production                  | Useful for labs; production TLS should use properly managed certificates and validation.                         |
| Old primary can simply restart after failover                 | It must safely rejoin as a standby and split brain must be prevented.                                            |
| PG15/PG17 paths can be mixed in one setup                     | Keep binaries, `PGDATA`, packages and HA extension versions consistent unless performing an intentional upgrade. |

---

# 50. Final Class Summary

The complete topic can be remembered as:

```text
             PostgreSQL HA

                   |
        +----------+----------+
        |                     |
   PostgreSQL Core       HA Manager
        |                     |
 Streaming Replication   pg_auto_failover
 WAL                     Monitor
 Hot Standby             Health/state management
 Sync Replication        Promotion coordination
 Promotion               Switchover
 pg_rewind               Candidate selection
 libpq multi-host        Maintenance
        |
        +---------------------+
                   |
                   v

             APPLICATION

        read-write connection
        read-only connection
        retry/reconnect logic
```

### Major upgrade

```text
PG17
  |
  | pg_upgrade / dump-restore /
  | logical replication
  v
PG18
```

Physical replication should not be used directly across those major versions. PostgreSQL provides a documented `pg_upgrade` procedure for primary/standby environments. ([PostgreSQL][7])

### Minor update

```text
PG18.x
  |
  | stop instance
  | update binaries
  | restart
  v
newer PG18.x
```

No `pg_upgrade` or dump/restore is normally required, but the exact release notes must always be checked. ([PostgreSQL][6])

### Most important HA principle

**HA is not just automatic promotion.**

A production-ready solution requires:

**Replication + failure detection + safe promotion + fencing + client reconnection + old-primary rejoin + monitoring + tested operational procedures.** 

PostgreSQL's official HA documentation reinforces the need to protect against dual-primary/split-brain situations and recommends documented, regularly tested failover procedures. ([PostgreSQL][2])

**Official PostgreSQL references used for validation:** 
  [PostgreSQL 18 High Availability, Load Balancing and Replication](https://www.postgresql.org/docs/18/high-availability.html) 
· [PostgreSQL Streaming and Synchronous Replication](https://www.postgresql.org/docs/current/warm-standby.html) 
· [PostgreSQL Failover Documentation](https://www.postgresql.org/docs/18/warm-standby-failover.html) 
· [PostgreSQL pg_upgrade Documentation](https://www.postgresql.org/docs/18/pgupgrade.htm) 
· [PostgreSQL libpq Connection Documentation](https://www.postgresql.org/docs/18/libpq-connect.html) 
· [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html) 
· [PostgreSQL Versioning Policy](https://www.postgresql.org/support/versioning/).

[1]: https://www.postgresql.org/about/news/postgresql-186-1711-1615-1519-1424-and-19-beta-3-released-3365/"PostgreSQL: PostgreSQL 18.6, 17.11, 16.15, 15.19, 14.24 and 19 Beta 3 Released!"
[2]: https://www.postgresql.org/docs/18/warm-standby-failover.html"PostgreSQL: Documentation: 18: 26.3. Failover"
[3]: https://www.postgresql.org/docs/current/warm-standby.html"PostgreSQL: Documentation: 18: 26.2. Log-Shipping Standby Servers"
[4]: https://www.postgresql.org/docs/18/libpq-connect.html"PostgreSQL: Documentation: 18: 32.1. Database Connection Control Functions"
[5]: https://www.postgresql.org/docs/current/libpq-envars.html"PostgreSQL: Documentation: 18: 32.15. Environment Variables"
[6]: https://www.postgresql.org/support/versioning/"PostgreSQL: Versioning Policy"
[7]: https://www.postgresql.org/docs/current/logical-replication.html"PostgreSQL: Documentation: 18: Chapter 29. Logical Replication"
[8]: https://www.postgresql.org/docs/current/pgupgrade.html"PostgreSQL: Documentation: 18: pg_upgrade"
[9]: https://www.postgresql.org/about/news/postgresql-18-released-3142/"PostgreSQL: PostgreSQL 18 Released!"
