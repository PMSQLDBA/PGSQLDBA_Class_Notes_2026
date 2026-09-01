# PostgreSQL 18 Class Notes

## Major Version Upgrade, Minor Version Patching, and Replication Fundamentals

**Class Date:** 1 September 2026
**Validated Against:** PostgreSQL 18 official documentation
**Operating System Used in Lab:** RHEL-compatible Linux
**Major Upgrade Example:** PostgreSQL 16 → PostgreSQL 18
**Minor Upgrade Example:** PostgreSQL 18.2 → PostgreSQL 18.4

---

## 1. PostgreSQL Replication Overview

PostgreSQL replication copies database changes from one PostgreSQL server to another.

```text
Primary PostgreSQL Server
        |
        | WAL/Data Changes
        v
Standby or Subscriber Server
```

Replication is commonly used for:

* High availability
* Disaster recovery
* Read-only workload distribution
* Reporting servers
* Database migrations
* Major-version upgrades with reduced downtime
* Geographic disaster-recovery environments

### Correct PostgreSQL terminology

| Older or Generic Term | Preferred PostgreSQL Term |
| --------------------- | ------------------------- |
| Master                | Primary                   |
| Slave                 | Standby or Subscriber     |
| Source server         | Primary or Publisher      |
| Destination server    | Standby or Subscriber     |
| Physical replica      | Standby                   |
| Logical replica       | Subscriber                |

---

## 2. High Availability and Disaster Recovery

### 2.1 High Availability

High availability, or HA, reduces application downtime when the primary database server fails.

A typical HA solution includes:

* Primary PostgreSQL server
* One or more standby servers
* Replication
* Failure detection
* Standby promotion
* Application connection redirection

PostgreSQL supports standby promotion, but built-in streaming replication does not automatically detect failures and redirect applications. 

Production HA normally uses additional components such as Patroni, repmgr, pg_auto_failover, HAProxy, Keepalived, or a cloud-managed database service.

### 2.2 Disaster Recovery

Disaster recovery, or DR, protects the database against larger failures such as:

* Complete server failure
* Storage failure
* Data-center outage
* Regional cloud outage
* Accidental data deletion
* Database corruption
* Ransomware or security incidents

A proper DR design normally combines:

* Physical or logical replication
* Regular base backups
* WAL archiving
* Point-in-time recovery
* Off-site backup storage
* Documented recovery procedures
* Regular recovery testing

Replication alone is not a backup. For example, an accidental `DROP TABLE` can also be replicated to the standby.

PostgreSQL database servers can work together for failover and read scaling, but the complete HA design requires failure management and client redirection. ([postgresql.org][1])

---

## 3. PostgreSQL Replication Types

PostgreSQL mainly supports two built-in replication methods:

1. Physical replication
2. Logical replication

---

## 4. Physical Streaming Replication

Physical streaming replication transfers WAL records from a primary PostgreSQL cluster to one or more standby clusters.

```text
Primary Cluster
      |
      | WAL stream
      v
Physical Standby
```

### Characteristics

* Replicates the entire PostgreSQL cluster.
* Works at the physical storage and WAL level.
* Replicates all databases in the cluster.
* Replicates table changes, indexes, DDL, sequences, roles stored in the cluster, and most other database objects.
* A standby normally remains in recovery mode.
* Read-only queries are allowed when Hot Standby is enabled.
* A physical standby can be promoted as a new primary.
* Streaming replication is asynchronous by default.
* Physical replication is normally used between servers running the same PostgreSQL major version.

Streaming replication sends WAL records as they are generated instead of waiting for a complete WAL segment. ([postgresql.org][2])

### One-to-one physical replication

```text
Primary
   |
   v
Standby 1
```

This design provides a single failover standby.

### Multiple-standby replication

```text
              +--> Standby 1
Primary ------+
              +--> Standby 2
```

Uses include:

* More than one failover candidate
* Read-only reporting
* Geographic DR
* Backup processing from a standby
* Separation of reporting and production workloads

### Important correction

Standbys do not “allow only reads” under every condition.

* When `hot_standby = on`, users can connect and run read-only queries during recovery.
* When Hot Standby is unavailable or disabled, the standby still applies WAL but does not accept normal read-only sessions.
* Temporary objects, sequence operations, DDL, and normal writes are not allowed during Hot Standby recovery.

The default value of `hot_standby` is `on`, and it only takes effect during recovery or standby operation. ([postgresql.org][3])

---

## 5. Asynchronous Replication

Streaming replication operates asynchronously by default.

### How it works

1. A transaction commits on the primary.
2. The primary confirms the commit without waiting for a standby.
3. WAL is sent to the standby.
4. The standby receives and replays the WAL.

### Advantages

* Lower transaction latency
* Primary activity can continue if a standby becomes unavailable
* Suitable for long-distance replication
* Less impact from slow standby servers or networks

### Risk

If the primary fails before the latest WAL reaches the standby, recently committed transactions can be lost.

### Important correction

The statement “the primary will continue writing without considering the standby” is generally correct for asynchronous replication, but WAL retention still requires attention. 

If a replication slot is used and the standby remains unavailable, WAL can accumulate on the primary and fill its disk.

---

## 6. Synchronous Replication

Synchronous replication makes a transaction commit wait for confirmation from one or more configured synchronous standbys.

### Basic flow

```text
Application
     |
     v
Primary writes WAL
     |
     v
Configured standby acknowledges WAL
     |
     v
Primary confirms COMMIT
```

### Advantages

* Reduces or eliminates committed-transaction loss during primary failure
* Provides stronger durability
* Appropriate for critical financial or transactional systems

### Considerations

* Commit latency increases.
* Network latency affects transaction response time.
* A slow synchronous standby can slow the primary.
* If every required synchronous standby becomes unavailable, affected commits can wait indefinitely unless the configuration or standby selection is changed.
* Existing sessions and read activity on the primary do not necessarily stop.

### Important correction

A failed synchronous standby does not mean that the primary database itself is down. 

The primary can remain online, but transactions waiting for the required synchronous acknowledgement may block.

The precise durability guarantee depends on `synchronous_commit`, including:

* `remote_apply`
* `on`
* `remote_write`
* `local`
* `off`

PostgreSQL streaming replication is asynchronous by default, and synchronous behavior must be explicitly configured. ([postgresql.org][2])

---

## 7. Cascading Replication

Cascading replication allows a standby to receive WAL from an upstream server and forward it to another standby.

```text
Primary
   |
   v
Standby 1
   |
   v
Standby 2
```

### Benefits

* Reduces replication connections to the primary
* Reduces network traffic across data centers
* Supports multi-level DR designs
* Useful when remote standbys cannot connect directly to the primary

### Important points

* Standby 1 acts as both a WAL receiver and WAL sender.
* Standby 2 is called the downstream standby.
* Standby 1 is the upstream server for Standby 2.
* Every standby connects to only one upstream server.
* PostgreSQL does not impose a fixed topology on downstream cascading standbys.
* Cascading physical replication is currently asynchronous.
* Failure of the intermediate standby interrupts the downstream stream until connectivity is restored or the downstream is redirected.
* Replication delay may increase at every additional level.

PostgreSQL describes a cascading standby as a relay that streams WAL to downstream standbys. ([postgresql.org][2])

---

## 8. Logical Replication

Logical replication copies selected data changes using a publish-and-subscribe model.

```text
Publisher
    |
    | Logical row changes
    v
Subscriber
```

### Main components

* **Publisher:** Server containing the source tables.
* **Publication:** Defines which tables and operations are published.
* **Subscriber:** Server receiving the changes.
* **Subscription:** Defines the connection to the publisher and the publications to consume.
* **Logical replication slot:** Retains the changes required by the subscriber.

### Characteristics

* Can replicate selected tables.
* Works at the logical row-change level.
* Can replicate between different PostgreSQL major versions, subject to compatibility rules.
* Can be used for migrations with reduced downtime.
* Can consolidate selected tables from multiple systems.
* Publisher and subscriber may have different indexes.
* The subscriber can contain additional local tables.

PostgreSQL defines logical replication as replicating data objects and their changes based on a replication identity, normally a primary key. ([postgresql.org][4])

### Logical replication limitations

By default, logical replication does not automatically replicate:

* DDL changes
* Table creation
* Schema changes
* Roles and users
* Tablespaces
* Large objects
* Sequence state
* Changes to unlogged tables
* Changes to temporary tables

The required table definitions normally must already exist on the subscriber.

### Replica identity

Tables publishing `UPDATE` or `DELETE` operations require a suitable replica identity.

The preferred choice is a primary key.

If a table does not have a suitable key, `REPLICA IDENTITY FULL` can be used, but it may increase processing cost:

```sql
ALTER TABLE hr.employee REPLICA IDENTITY FULL;
```

Published tables require a replica identity so that the subscriber can identify the correct rows for `UPDATE` and `DELETE` operations. ([postgresql.org][5])

### Cross-version example

```text
PostgreSQL 16 Publisher
          |
          | Logical replication
          v
PostgreSQL 18 Subscriber
```

For a major-version migration:

1. Create compatible schemas on PostgreSQL 18.
2. Create publications on PostgreSQL 16.
3. Create subscriptions on PostgreSQL 18.
4. Allow the initial data copy to complete.
5. Monitor replication lag.
6. Stop or quiesce application writes.
7. Synchronize sequence values separately.
8. Confirm that the subscriber has received the final changes.
9. Redirect the application to PostgreSQL 18.

Schema differences that are incompatible with incoming rows can stop logical replication until the subscriber schema is corrected. ([postgresql.org][6])

---

## 9. Corrected Streaming Replication Prerequisites

The original notes listed several conditions as mandatory. The corrected requirements are below.

| Requirement                   | Validation                                                                                    |
| ----------------------------- | --------------------------------------------------------------------------------------------- |
| Same operating system         | Not strictly required, but the platforms and builds must be physically compatible             |
| Same PostgreSQL major version | Required for normal physical streaming replication                                            |
| Same minor version            | Strongly recommended; patch standbys before or with the primary according to release guidance |
| Same database names           | Automatically true because physical replication copies the complete cluster                   |
| Same data-directory path      | Not required                                                                                  |
| Same directory structure      | Not required, except tablespace mapping and configuration must be handled correctly           |
| Same hardware                 | Not required                                                                                  |
| Same disk capacity            | Not required, but the standby must have enough capacity                                       |
| Same CPU and memory           | Not required                                                                                  |
| Hostname resolution           | Recommended; IP addresses can also be used                                                    |
| Port 5432                     | Not mandatory; PostgreSQL can use any configured and permitted port                           |
| Network connectivity          | Required between the standby and WAL sender                                                   |
| Replication authentication    | Required in `pg_hba.conf`                                                                     |
| Replication user              | Required unless another suitably privileged role is used                                      |
| WAL settings                  | Required on the primary                                                                       |
| Base backup                   | Normally required to initialize a physical standby                                            |

Typical primary settings include:

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

Example replication role:

```sql
CREATE ROLE replicator
WITH LOGIN REPLICATION
PASSWORD 'use-a-secure-password';
```

Example `pg_hba.conf` rule:

```conf
hostssl replication replicator 192.168.10.0/24 scram-sha-256
```

---

# Part II – PostgreSQL 16 to 18 Major-Version Upgrade

## 10. Major Upgrade Fundamentals

PostgreSQL 16 and PostgreSQL 18 are different major versions.

The PostgreSQL 16 data directory cannot be started directly using PostgreSQL 18 binaries. A supported migration method is required.

Supported approaches include:

* `pg_upgrade`
* Dump and restore
* Logical replication
* An application-specific migration method

`pg_upgrade` creates new system catalogs and reuses, copies, clones, or swaps user relation files. It avoids a complete logical export and import. ([postgresql.org][7])

---

## 11. `pg_upgrade` Transfer Modes

| Mode            | Option              |                            Disk Requirement | Rollback Characteristics                            |
| --------------- | ------------------- | ------------------------------------------: | --------------------------------------------------- |
| Copy            | `--copy`            | Approximately another copy of relation data | Old cluster remains independent                     |
| Link            | `--link` or `-k`    |  Very little additional relation-file space | Old cluster becomes unsafe after new cluster starts |
| Clone           | `--clone`           |                    Uses filesystem reflinks | Old cluster remains independent                     |
| File-range copy | `--copy-file-range` |                        Filesystem dependent | Depends on filesystem behavior                      |
| Swap            | `--swap`            |                             Low extra space | Old cluster becomes unusable once transfer begins   |

### Correct meaning of link mode

`--link` creates hard links between the old and new clusters’ user relation files.

A hard link:

* Points to the same underlying file data.
* Normally has the same inode number.
* Does not create a complete second copy.
* Consumes very little additional storage.
* Requires the old and new data directories to reside on the same filesystem.
* Does not mean that an inode itself is a data file.

PostgreSQL confirms that link mode is faster and uses less disk, but the old cluster cannot safely be used after the new cluster starts. ([postgresql.org][8])

---

## 12. Critical Link-Mode Rollback Rule

> After PostgreSQL 18 has been started following a link-mode upgrade, PostgreSQL 16 must not be restarted as a rollback.

This is because both clusters share the linked user relation files. PostgreSQL 18 can modify those files in ways that PostgreSQL 16 does not understand.

### Unsafe procedure removed from the notes

The following is not a safe rollback after PostgreSQL 18 has started:

```bash
mv pg_control.old pg_control
systemctl start postgresql-16
```

Renaming `pg_control.old` is allowed only when the new cluster has not yet been started.

### Safe rollback options

* Before PostgreSQL 18 is started, restore the old `pg_control` filename and start PostgreSQL 16.
* After PostgreSQL 18 has started, restore the PostgreSQL 16 cluster from a verified backup or storage snapshot.
* Alternatively, perform the upgrade using `--copy` or supported `--clone` mode when simple rollback independence is required.

---

## 13. Pre-Upgrade Checklist

Before upgrading:

* Read the PostgreSQL 18 release notes.
* Confirm that the upgrade path is supported.
* Test the complete upgrade in a non-production environment.
* Take a full physical backup.
* Take a storage snapshot if available.
* Verify that the backup can be restored.
* Record cluster size and database list.
* Record users, extensions, tablespaces, and configuration.
* Check for prepared transactions.
* Install PostgreSQL 18 versions of all required extensions.
* Confirm compatible locale, encoding, block size, WAL segment size, and checksum settings.
* Review application driver compatibility.
* Schedule application downtime.
* Define an approved rollback plan.
* Confirm sufficient disk space.
* Run `pg_upgrade --check`.

Example inventory queries:

```sql
SELECT version();

SHOW data_directory;
SHOW server_encoding;
SHOW data_checksums;

SELECT datname
FROM pg_database
ORDER BY datname;

SELECT extname, extversion
FROM pg_extension
ORDER BY extname;

SELECT spcname, pg_tablespace_location(oid)
FROM pg_tablespace;
```

---

## 14. Initialize the PostgreSQL 18 Cluster

Install the PostgreSQL 18 packages:

```bash
sudo dnf install -y \
  postgresql18 \
  postgresql18-server \
  postgresql18-contrib
```

Initialize the new cluster:

```bash
sudo -u postgres /usr/pgsql-18/bin/initdb \
  -D /var/lib/pgsql/18/data
```

### Checksum correction

Do not automatically use `--no-data-checksums`.

The new cluster must be initialized with settings compatible with the old cluster. Check the source first:

```bash
sudo -u postgres /usr/pgsql-16/bin/pg_controldata \
  /var/lib/pgsql/16/data |
grep "Data page checksum version"
```

If the source uses checksums, initialize the target with checksums:

```bash
sudo -u postgres /usr/pgsql-18/bin/initdb \
  --data-checksums \
  -D /var/lib/pgsql/18/data
```

The new cluster does not need to be manually started before running `pg_upgrade`. ([postgresql.org][7])

---

## 15. Configure a Temporary Target Port

Configure PostgreSQL 18 to use a temporary port during validation:

```conf
port = 5433
```

This avoids a conflict with PostgreSQL 16 on port 5432.

Remember that `pg_upgrade` temporarily controls both clusters during the upgrade. The services must not already be running during the actual upgrade.

---

## 16. Run the Pre-Upgrade Check

The precheck can be run while the old cluster is online, although some checks may be more complete with controlled conditions.

```bash
sudo -u postgres /usr/pgsql-18/bin/pg_upgrade \
  --check \
  --link \
  --old-bindir=/usr/pgsql-16/bin \
  --new-bindir=/usr/pgsql-18/bin \
  --old-datadir=/var/lib/pgsql/16/data \
  --new-datadir=/var/lib/pgsql/18/data \
  --old-port=5432 \
  --new-port=5433
```

Resolve every error before the production upgrade.

Common findings include:

* Missing extension libraries
* Incompatible data types
* Unsupported extensions
* Prepared transactions
* Authentication problems
* Incorrect ownership
* Incompatible initialization settings
* Tablespace conflicts

---

## 17. Stop Application Traffic

Before the final upgrade:

1. Ask the application team to stop application services.
2. Stop batch jobs and ETL processes.
3. Stop monitoring tools that create database sessions.
4. Terminate remaining application sessions if approved.
5. Confirm that no business writes are occurring.
6. Record the cutover start time.

Check sessions:

```sql
SELECT pid,
       usename,
       application_name,
       client_addr,
       state
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
```

---

## 18. Stop PostgreSQL 16

```bash
sudo systemctl stop postgresql-16
```

Confirm that no cluster process remains:

```bash
pgrep -a postgres
```

---

## 19. Run the Link-Mode Upgrade

Run the PostgreSQL 18 copy of `pg_upgrade`:

```bash
sudo -u postgres /usr/pgsql-18/bin/pg_upgrade \
  --link \
  --old-bindir=/usr/pgsql-16/bin \
  --new-bindir=/usr/pgsql-18/bin \
  --old-datadir=/var/lib/pgsql/16/data \
  --new-datadir=/var/lib/pgsql/18/data \
  --old-port=5432 \
  --new-port=5433 \
  --jobs="$(nproc)"
```

Important points:

* Always use the `pg_upgrade` binary from the new PostgreSQL version.
* Run it as the PostgreSQL operating-system account.
* Run it from a writable working directory.
* Review the complete output.
* Do not start PostgreSQL 16 after PostgreSQL 18 has been started.
* Preserve all generated logs.

PostgreSQL recommends using parallel jobs for multi-database and multi-tablespace clusters. ([postgresql.org][7])

---

## 20. Merge Configuration Files

`pg_upgrade` does not automatically merge every custom configuration setting.

Review and merge:

* `postgresql.conf`
* `postgresql.auto.conf`
* `pg_hba.conf`
* `pg_ident.conf`
* SSL certificate paths
* Archive commands
* Logging configuration
* Memory settings
* Connection limits
* Replication settings
* Shared preload libraries
* Tablespace-related configuration

Do not blindly overwrite the PostgreSQL 18 default configuration with the PostgreSQL 16 file. New parameters and changed defaults could be lost.

After the upgrade, configure the production port:

```conf
port = 5432
```

Validate configuration syntax:

```bash
sudo -u postgres /usr/pgsql-18/bin/postgres \
  -D /var/lib/pgsql/18/data \
  -C port
```

---

## 21. Start PostgreSQL 18

```bash
sudo systemctl start postgresql-18
sudo systemctl status postgresql-18 --no-pager
```

Confirm the version and data directory:

```bash
sudo -u postgres psql -X -d postgres -c "
SELECT version();
SHOW data_directory;
SHOW port;
"
```

---

## 22. Post-Upgrade Statistics

PostgreSQL 18 transfers most optimizer statistics by default, but some statistics are not transferred. 

These include certain extended, extension-provided, and cumulative statistics. ([postgresql.org][7])

Run the commands generated or recommended by `pg_upgrade`:

```bash
sudo -u postgres /usr/pgsql-18/bin/vacuumdb \
  --all \
  --analyze-in-stages \
  --missing-stats-only
```

Then:

```bash
sudo -u postgres /usr/pgsql-18/bin/vacuumdb \
  --all \
  --analyze-only
```

`ANALYZE` updates optimizer statistics so the query planner can select efficient execution plans. ([postgresql.org][9])

---

## 23. Extension Updates

Check installed extensions:

```sql
SELECT extname, extversion
FROM pg_extension
ORDER BY extname;
```

Run the extension update script generated by `pg_upgrade`, if present.

An individual extension can be updated with:

```sql
ALTER EXTENSION extension_name UPDATE;
```

Do not run this blindly. Review compatibility, extension release notes, and vendor instructions first.

---

## 24. Post-Upgrade Validation

Validate:

```sql
SELECT version();

SELECT datname
FROM pg_database
ORDER BY datname;

SELECT extname, extversion
FROM pg_extension
ORDER BY extname;

SELECT schemaname,
       count(*) AS table_count
FROM pg_tables
GROUP BY schemaname
ORDER BY schemaname;
```

Additional checks:

* Compare important table row counts.
* Validate application logins.
* Validate business queries.
* Confirm indexes and constraints.
* Test application transactions.
* Review PostgreSQL logs.
* Check background workers.
* Validate backup jobs.
* Validate WAL archiving.
* Validate monitoring.
* Validate replication if configured.
* Compare important query plans and performance.
* Confirm that scheduled jobs are running.

Do not delete the old cluster immediately.

---

## 25. Remove the Old Cluster

Only after the upgrade is accepted and rollback retention has expired should the old cluster be removed.

Use the script generated by `pg_upgrade`, for example:

```bash
./delete_old_cluster.sh
```

Before running it:

* Confirm application approval.
* Confirm backup completion.
* Confirm restore testing.
* Confirm monitoring is healthy.
* Confirm the rollback period has ended.
* Confirm the exact directory the script will remove.

This operation is destructive.

---

# Part III – PostgreSQL 18 Minor-Version Upgrade

## 26. Minor-Version Upgrade Fundamentals

Examples:

```text
18.2 → 18.4
16.13 → 16.14
```

A minor release normally includes:

* Security fixes
* Bug fixes
* Data-corruption fixes
* Stability improvements
* Low-risk improvements to existing behavior

A minor release does not normally introduce the storage-format changes associated with a major release.

Therefore:

* `pg_upgrade` is not required.
* Dump and restore are not normally required.
* The existing data directory is retained.
* Stop the server, replace the binaries, and restart it.
* Always read all intervening release notes.

PostgreSQL states that minor releases retain compatible internal storage formats and that the data directory remains unchanged. ([postgresql.org][10])

---

## 27. Minor Upgrade Procedure

### Step 1: Review release notes

Review every minor release between the installed and target versions.

Check for:

* Security fixes
* Extension compatibility
* Required reindex operations
* Recommended maintenance
* Replication-related fixes
* Platform-specific issues

### Step 2: Record the current version

```bash
sudo -u postgres psql -X -Atc "
SELECT version();
SHOW data_directory;
"
```

Check installed packages:

```bash
rpm -qa |
grep '^postgresql18' |
sort
```

### Step 3: Take a verified backup

Before changing packages:

* Take a physical or logical backup.
* Confirm WAL archiving.
* Take a storage snapshot where available.
* Confirm the recovery procedure.

### Step 4: Stop application traffic

* Stop applications.
* Stop jobs.
* Confirm that active writes have stopped.
* Record the maintenance start time.

### Step 5: Stop PostgreSQL

```bash
sudo systemctl stop postgresql-18
```

Confirm shutdown:

```bash
pgrep -a postgres
```

### Step 6: Upgrade all PostgreSQL packages together

```bash
sudo dnf upgrade -y \
  postgresql18 \
  postgresql18-libs \
  postgresql18-server \
  postgresql18-contrib
```

Keeping all PostgreSQL packages at a matching release level avoids package inconsistencies.

### Step 7: Start PostgreSQL

```bash
sudo systemctl start postgresql-18
sudo systemctl status postgresql-18 --no-pager
```

### Step 8: Validate

```bash
sudo -u postgres psql -X -d postgres -c "
SELECT version();
SHOW data_directory;
"
```

Review logs:

```bash
sudo journalctl -u postgresql-18 \
  --since "30 minutes ago" \
  --no-pager
```

### Step 9: Validate the application

* Test login.
* Run a read query.
* Run an approved test transaction.
* Validate application logs.
* Confirm scheduled jobs.
* Confirm backups.
* Confirm monitoring and alerts.

### Step 10: Start application services

Start application traffic only after technical and application validation is successful.

---

## 28. Minor-Version Downgrade

PostgreSQL minor releases within the same major version use a compatible data-directory format. However, a package downgrade should not be treated as the primary production rollback plan.

Preferred rollback order:

1. Review the release notes for any special warnings.
2. Stop the application.
3. Stop PostgreSQL.
4. Restore the pre-upgrade VM or storage snapshot when available.
5. Otherwise, reinstall the previous complete package set.
6. Start PostgreSQL.
7. Validate logs and application functionality.

Example package downgrade:

```bash
sudo dnf downgrade -y \
  postgresql18-18.2\* \
  postgresql18-libs-18.2\* \
  postgresql18-server-18.2\* \
  postgresql18-contrib-18.2\*
```

The exact package version depends on what remains available in the configured PGDG repository or local package archive.

---

## 29. Major and Minor Upgrade Comparison

| Item                  | Major Upgrade                     | Minor Upgrade                       |
| --------------------- | --------------------------------- | ----------------------------------- |
| Example               | 16 → 18                           | 18.2 → 18.4                         |
| Storage compatibility | Not directly compatible           | Compatible within the major version |
| `pg_upgrade`          | Supported and commonly used       | Not required                        |
| Dump and restore      | Supported option                  | Normally unnecessary                |
| Logical replication   | Supported option                  | Normally unnecessary                |
| New data directory    | Required for `pg_upgrade`         | Not required                        |
| New binaries          | Required                          | Required                            |
| Extension review      | Required                          | Required                            |
| Downtime              | Depends on method and data size   | Usually short                       |
| Rollback              | Depends heavily on migration mode | Snapshot or previous packages       |
| Release-note review   | Mandatory                         | Mandatory                           |

---

## 30. Key Corrections Applied

* Replaced **master/slave** terminology with **primary/standby**.
* Corrected the claim that replicas are always read-only and queryable.
* Clarified that streaming replication is asynchronous by default.
* Corrected the behavior when a synchronous standby fails.
* Corrected physical-replication prerequisites.
* Removed the incorrect requirement for identical hardware and directory paths.
* Clarified that port 5432 is the default, not a mandatory replication port.
* Clarified that logical replication works at the table and row-change level.
* Added logical-replication limitations and replica-identity requirements.
* Corrected the explanation of inode and hard links.
* Clarified that link mode requires the same filesystem, not simply the same server.
* Removed the unsafe link-mode rollback after PostgreSQL 18 has started.
* Added `pg_upgrade --check`.
* Added configuration and extension migration requirements.
* Corrected the PostgreSQL 18 statistics behavior.
* Added post-upgrade statistics and validation.
* Added backup, snapshot, monitoring, and application validation steps.
* Clarified that minor upgrades do not require `pg_upgrade`.
* Added safe minor-version rollback guidance.

## Official PostgreSQL References

* [PostgreSQL 18 – pg_upgrade](https://www.postgresql.org/docs/18/pgupgrade.html)
* [PostgreSQL 18 – Upgrading a PostgreSQL Cluster](https://www.postgresql.org/docs/18/upgrading.html)
* [PostgreSQL 18 – High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/18/high-availability.html)
* [PostgreSQL 18 – Log-Shipping Standby Servers](https://www.postgresql.org/docs/18/warm-standby.html)
* [PostgreSQL 18 – Replication Configuration](https://www.postgresql.org/docs/18/runtime-config-replication.html)
* [PostgreSQL 18 – Logical Replication](https://www.postgresql.org/docs/18/logical-replication.html)
* [PostgreSQL 18 – Logical Replication Restrictions](https://www.postgresql.org/docs/18/logical-replication-restrictions.html)
* [PostgreSQL Versioning Policy](https://www.postgresql.org/support/versioning/)

[1]: https://www.postgresql.org/docs/18/high-availability.html "PostgreSQL: Documentation: 18: Chapter 26. High Availability, Load Balancing, and Replication"
[2]: https://www.postgresql.org/docs/current/warm-standby.html "PostgreSQL: Documentation: 18: 26.2. Log-Shipping Standby Servers"
[3]: https://www.postgresql.org/docs/18/runtime-config-replication.html "PostgreSQL: Documentation: 18: 19.6. Replication"
[4]: https://www.postgresql.org/docs/current/logical-replication.html "Documentation: 18: Chapter 29. Logical Replication"
[5]: https://www.postgresql.org/docs/current/logical-replication-publication.html "PostgreSQL: Documentation: 18: 29.1. Publication"
[6]: https://www.postgresql.org/docs/current/logical-replication-restrictions.html "Documentation: 18: 29.8. Restrictions"
[7]: https://www.postgresql.org/docs/18/pgupgrade.html "PostgreSQL: Documentation: 18: pg_upgrade"
[8]: https://www.postgresql.org/docs/current/pgupgrade.html "PostgreSQL: Documentation: 18: pg_upgrade"
[9]: https://www.postgresql.org/docs/18/sql-analyze.html "PostgreSQL: Documentation: 18: ANALYZE"
[10]: https://www.postgresql.org/support/versioning/ "Versioning Policy"
