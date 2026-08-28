# PostgreSQL 18 Point-in-Time Recovery (PITR) – Class Notes

**Class Date:** 27th August 2026 – Thursday

These notes are organized from the supplied class session and preserve the demonstrated PostgreSQL 18 PITR workflow. 

---

## 1. What is Point-in-Time Recovery?

**Point-in-Time Recovery (PITR)** is a PostgreSQL recovery method used to restore an entire PostgreSQL cluster to a specific point in time.

Consider this example:

* Full backup completed at **5:00 PM**
* A new table was created at **5:30 PM**
* Around **50 GB of data** was loaded into the table
* The table was accidentally dropped at **7:00 PM**
* The application team needs the table back

### Can the 5:00 PM full backup alone recover the table?

**No.**

The table did not exist when the 5:00 PM backup was taken.

However, if PostgreSQL has:

* A valid base backup
* Continuous WAL archiving
* Required WAL files covering the period after the backup

PostgreSQL can restore the base backup and replay WAL records until a point immediately before the table was dropped.

This process is called **Point-in-Time Recovery**.

### Basic PITR Flow

**Base Backup**

↓

**Archived WAL Files**

↓

**Restore Base Backup**

↓

**Replay WAL**

↓

**Stop Before DROP TABLE**

↓

**Recover the Table**

---

# 2. Lab Scenario

The actual class demonstration used the following timeline:

* Base backup: approximately **13:27 UTC**
* Table creation: **13:32:04 UTC**
* Table drop: **13:34:57 UTC**
* Recovery target: **13:34:52.045128+00**

The recovery target was therefore set a few seconds before the table was dropped.

---

# 3. Take a PostgreSQL Base Backup

The class used `pg_basebackup` to create the base backup.

**Command:**

`pg_basebackup -D /var/lib/pgsql/backup -X fetch -P -c fast`

### Important Options

**`-D /var/lib/pgsql/backup`**

Specifies the destination directory for the backup.

**`-X fetch`**

Includes the WAL files required to make the backup consistent.

**`-P`**

Displays backup progress.

**`-c fast`**

Requests a fast checkpoint before taking the backup.

After the backup completes, move to the backup directory.

**Command:**

`cd /var/lib/pgsql/backup`

Check the backup.

**Command:**

`ls -ltr`

Important files and directories include:

* `backup_label`
* `backup_manifest`
* `PG_VERSION`
* `postgresql.conf`
* `postgresql.auto.conf`
* `pg_hba.conf`
* `pg_ident.conf`
* `base/`
* `global/`
* `pg_wal/`
* `pg_xact/`

The successful backup in the class contained a complete PostgreSQL cluster copy.

---

# 4. Understand the WAL Backup Information

Move to the primary PostgreSQL WAL directory.

**Command:**

`cd /var/lib/pgsql/18/data/pg_wal/`

Check the WAL files.

**Command:**

`ls -ltr`

The class showed WAL files such as:

* `0000000100000000000000D1`
* `0000000100000000000000D2`
* `0000000100000000000000D3`

A backup history file was also generated:

`0000000100000000000000D2.00000028.backup`

Display it using:

**Command:**

`cat 0000000100000000000000D2.00000028.backup`

Important information recorded in the file included:

* Start WAL location
* Stop WAL location
* Checkpoint location
* Backup method
* Backup source
* Start time
* Stop time
* Timeline

For the class backup:

* Start time: `2026-07-04 13:27:40 UTC`
* Stop time: `2026-07-04 13:27:41 UTC`
* Timeline: `1`

---

# 5. Identify the Required Recovery Time

For time-based PITR, the DBA needs to determine the appropriate recovery target.

In this lab, PostgreSQL statement logging was enabled, which allowed the CREATE and DROP operations to be located in the database logs.

Check logging configuration.

**SQL:**

`SHOW logging_collector;`

Result:

`on`

Check statement logging.

**SQL:**

`SHOW log_statement;`

Result:

`all`

Check statement duration logging.

**SQL:**

`SHOW log_min_duration_statement;`

Result:

`0`

### Important Note

PITR itself does **not depend on SQL statement logging**.

PITR depends on the base backup and required WAL records.

Detailed logging is useful in this example because it helps the DBA determine **when the accidental DROP TABLE occurred**, so an appropriate `recovery_target_time` can be selected.

---

# 6. Create the Test Table

Create the test table.

**SQL:**

`CREATE TABLE pitrtest`

`(eno int, ename varchar, esal int, eadd varchar);`

Load test data.

**SQL:**

`INSERT INTO pitrtest`

`SELECT generate_series(1,100000),`

`'emp' || generate_series(1,100000),`

`generate_series(1,100000),`

`'addr' || generate_series(1,100000);`

Verify the row count.

**SQL:**

`SELECT count(*) FROM pitrtest;`

Result:

`100000`

The table was therefore created **after the base backup** and contained 100,000 rows.

---

# 7. Find When the Table Was Created

Move to the PostgreSQL log directory.

**Command:**

`cd /var/lib/pgsql/18/data/pg_log`

Search for the CREATE TABLE statement.

**Command:**

`grep -i 'create table pitr' postgresql-2026-07-04_130135.log`

The log showed:

* Table creation time: `2026-07-04 13:32:04 UTC`

The timeline was therefore:

* Base backup: approximately `13:27`
* Table created: `13:32:04`

Therefore, `pitrtest` was **not part of the base backup**.

Its creation and subsequent data changes must be reconstructed through WAL replay.

---

# 8. Simulate the Accidental Table Drop

The table was intentionally dropped to demonstrate recovery.

**SQL:**

`DROP TABLE pitrtest;`

The table no longer exists after this operation.

---

# 9. Find the DROP TABLE Time

Search the PostgreSQL log.

**Command:**

`grep -i 'drop table pitr' postgresql-2026-07-04_130135.log`

The log showed:

* DROP TABLE time: `2026-07-04 13:34:57 UTC`

Therefore, recovery must stop **before this transaction is committed**.

The class selected:

`2026-07-04 13:34:52.045128+00`

This is approximately five seconds before the DROP operation.

---

# 10. Make Sure Required WAL Is Archived

Check the active WAL directory.

**Command:**

`cd /var/lib/pgsql/18/data/pg_wal`

**Command:**

`ls -ltr`

Then check the archive directory.

**Command:**

`cd /var/lib/pgsql/archive/`

**Command:**

`ls -ltr`

The archive must contain all WAL segments required to replay from the base backup up to the selected recovery target.

---

# 11. Force a WAL Switch

The class forced PostgreSQL to switch to a new WAL segment so the current completed WAL segment could become eligible for archiving.

**SQL:**

`SELECT pg_switch_wal();`

Afterward, check the archive directory again.

**Command:**

`ls -ltr /var/lib/pgsql/archive/`

In the demonstration, WAL segment:

`0000000100000000000000D3`

appeared in the archive directory.

### Important Point

Do not depend on a rule that the archive directory should always contain exactly one fewer WAL file than `pg_wal`.

The important requirement is that **all WAL files needed for recovery are safely archived and accessible**.

---

# 12. Prepare a Separate PITR Instance

The class did not recover directly over the production/original PostgreSQL instance.

Instead, the base backup was used as a **separate temporary PostgreSQL cluster**.

This is a useful approach when only one accidentally dropped table needs to be recovered.

The original instance remained on:

* Port: `5432`
* Data directory: `/var/lib/pgsql/18/data`

The temporary PITR instance used:

* Port: `5433`
* Data directory: `/var/lib/pgsql/backup`

---

# 13. Configure the PITR Instance

Move to the backup directory.

**Command:**

`cd /var/lib/pgsql/backup/`

Edit:

`postgresql.conf`

Configure the temporary instance.

**Port:**

`port = 5433`

**Cluster name:**

`cluster_name = 'pitr'`

**Restore command:**

`restore_command = 'cp /var/lib/pgsql/archive/%f %p'`

**Recovery target:**

`recovery_target_time = '2026-07-04 13:34:52.045128+00'`

### restore_command

`restore_command` tells PostgreSQL how to retrieve archived WAL files during recovery.

Here:

`%f`

represents the requested WAL filename.

`%p`

represents the destination path where PostgreSQL expects the WAL file.

---

# 14. Create recovery.signal

PostgreSQL needs to know that this cluster should start in archive recovery mode.

Create:

`recovery.signal`

**Command:**

`touch /var/lib/pgsql/backup/recovery.signal`

Verify it.

**Command:**

`ls -ltr /var/lib/pgsql/backup/`

The presence of `recovery.signal` instructs PostgreSQL to perform recovery when the cluster starts.

---

# 15. Set Data Directory Permissions

The PostgreSQL data directory must have appropriate permissions.

**Command:**

`chmod 0700 /var/lib/pgsql/backup/`

The directory should also belong to the PostgreSQL operating-system account when it is used as a PostgreSQL data directory.

---

# 16. Start the PITR Instance

Start the temporary recovery instance.

**Command:**

`/usr/pgsql-18/bin/pg_ctl -D /var/lib/pgsql/backup/ start`

The PITR instance starts on:

* Port: `5433`

while the original instance continues running on:

* Port: `5432`

This allows both PostgreSQL clusters to run independently.

---

# 17. Check Recovery Status

Connect to the PITR instance.

**Command:**

`psql -p 5433`

Check whether PostgreSQL is in recovery.

**SQL:**

`SELECT pg_is_in_recovery();`

During recovery, the result is:

`true`

This confirms that PostgreSQL is replaying WAL as part of recovery.

---

# 18. Review the PITR Recovery Log

The recovery instance log directory in the class was:

`/var/lib/pgsql/backup/pg_log`

The log showed PostgreSQL performing the following operations:

* Starting PostgreSQL on port `5433`
* Starting backup recovery
* Restoring archived WAL
* Starting point-in-time recovery
* Replaying WAL
* Reaching a consistent recovery state
* Stopping before the DROP transaction

A particularly important log message was:

`recovery stopping before commit of transaction 1157`

The transaction time shown was:

`2026-07-04 13:34:57.528515+00`

This demonstrates that recovery stopped before the transaction containing the DROP TABLE operation was committed.

---

# 19. Recovery Target Action

The class recovery stopped and paused at the target.

The recovery log instructed the DBA to execute:

**SQL:**

`SELECT pg_wal_replay_resume();`

After executing it, check recovery status again.

**SQL:**

`SELECT pg_is_in_recovery();`

Result:

`false`

The temporary cluster is now out of recovery.

### Production Recommendation

For a controlled PITR procedure, explicitly configure the required recovery behavior.

For example:

`recovery_target_action = 'pause'`

This makes the intended behavior clear instead of depending only on the default recovery-target action.

---

# 20. Verify the Recovered PITR Instance

Check the port.

**SQL:**

`SHOW port;`

Result:

`5433`

Check the data directory.

**SQL:**

`SHOW data_directory;`

Result:

`/var/lib/pgsql/backup`

Verify the recovered table.

**SQL:**

`\dt+ pitrtest`

The table exists.

Check the row count.

**SQL:**

`SELECT count(*) FROM pitrtest;`

Result:

`100000`

The dropped table has now been recovered successfully in the temporary PITR instance.

---

# 21. Why Recover to a Separate Instance?

The original PostgreSQL instance was **not rolled back**.

This is important because PITR operates at the **PostgreSQL cluster level**, not as a direct single-table restore mechanism.

If the original production cluster were recovered to the earlier time, changes made after that recovery target would no longer be part of that recovered cluster state.

For this scenario, only one table needed to be restored.

Therefore:

**Original Production Instance**

Port `5432`

↓

Continues running normally

**Temporary PITR Instance**

Port `5433`

↓

Recover entire cluster to before DROP TABLE

↓

Extract `pitrtest`

↓

Load it back into the original instance

This minimizes impact on the original database environment.

---

# 22. Export the Recovered Table

The recovered table now exists in the PITR instance on port `5433`.

The class used `pg_dump` and piped the output directly into `psql` connected to the original instance.

**Command:**

`pg_dump -d postgres -p 5433 -t pitrtest | psql -d postgres -p 5432`

This performs:

**Temporary PITR Instance – Port 5433**

↓

`pg_dump`

↓

Extract `pitrtest`

↓

Pipe SQL/Data

↓

`psql`

↓

**Original Instance – Port 5432**

The output showed:

* `CREATE TABLE`
* `ALTER TABLE`
* `COPY 100000`

This indicates that the table definition and 100,000 rows were restored into the original database.

---

# 23. Verify the Original Instance

Connect to the original PostgreSQL instance.

**Command:**

`psql -p 5432`

Check the port.

**SQL:**

`SHOW port;`

Result:

`5432`

Check the data directory.

**SQL:**

`SHOW data_directory;`

Result:

`/var/lib/pgsql/18/data`

Verify the recovered table.

**SQL:**

`SELECT count(*) FROM pitrtest;`

Result:

`100000`

The table is now restored into the original PostgreSQL instance.

---

# 24. recovery.signal After Recovery

The class notes show that `recovery.signal` was no longer present after recovery completed and the instance left recovery.

The PostgreSQL cluster had transitioned from recovery mode to normal operation.

The recovered data directory remained:

`/var/lib/pgsql/backup`

and the temporary recovered instance was running independently from the original cluster.

---

# 25. Complete PITR Workflow

The entire process can be remembered as:

**Take Base Backup**

↓

**Continue WAL Archiving**

↓

**Table Created After Backup**

↓

**Data Loaded**

↓

**Table Accidentally Dropped**

↓

**Determine DROP Time**

↓

**Ensure Required WAL Is Archived**

↓

**Prepare Separate Copy of Base Backup**

↓

**Configure restore_command**

↓

**Configure recovery_target_time**

↓

**Create recovery.signal**

↓

**Start Temporary PostgreSQL Instance**

↓

**Replay WAL**

↓

**Stop Before DROP Transaction**

↓

**Verify Recovered Table**

↓

**Dump Required Table**

↓

**Restore Table into Original Instance**

↓

**Validate Data**

---

# 26. Important PITR Components

### Base Backup

Provides the starting physical copy of the PostgreSQL cluster.

Example:

`/var/lib/pgsql/backup`

### WAL Archive

Contains the WAL history required to move the base backup forward in time.

Example:

`/var/lib/pgsql/archive`

### restore_command

Retrieves archived WAL during recovery.

Example:

`restore_command = 'cp /var/lib/pgsql/archive/%f %p'`

### recovery_target_time

Defines the target timestamp.

Example:

`recovery_target_time = '2026-07-04 13:34:52.045128+00'`

### recovery.signal

Tells PostgreSQL to start archive recovery.

### pg_is_in_recovery()

Shows whether the server is currently in recovery.

**SQL:**

`SELECT pg_is_in_recovery();`

---

# 27. Important Class Takeaways

* PITR requires a suitable **base backup plus the required WAL history**.
* A table created after the base backup can still be recovered if the required WAL is available.
* PITR restores the **PostgreSQL cluster state**, rather than directly restoring only one table.
* A temporary PITR instance is useful when only a specific object needs to be extracted.
* The temporary instance must use a different port when the original instance is running on the same server.
* `restore_command` retrieves WAL files from the archive.
* `recovery_target_time` determines where recovery should stop.
* `recovery.signal` activates archive recovery.
* `pg_is_in_recovery()` verifies recovery status.
* `pg_dump` can extract the required table from the recovered temporary instance.
* `psql` can restore that table into the original instance.
* Statement/audit logging is useful for finding the incident timestamp, but **it is not itself a requirement for PITR**.
* `pg_switch_wal()` can be used when you need the current WAL segment to be switched so a completed segment can be archived.
* Before performing PITR, verify that the required WAL chain is actually available rather than relying on WAL-file counts alone.

---

# 28. Final Lab Architecture

**Original PostgreSQL Instance**

Port: `5432`

Data directory:

`/var/lib/pgsql/18/data`

↓

**Continuous WAL Archiving**

↓

Archive directory:

`/var/lib/pgsql/archive`

---

**Base Backup**

Data directory:

`/var/lib/pgsql/backup`

↓

Configure PITR

↓

`restore_command`

*

`recovery_target_time`

*

`recovery.signal`

↓

Start on port `5433`

↓

Replay archived WAL

↓

Stop before DROP TABLE

↓

Recover `pitrtest`

↓

`pg_dump`

↓

Restore into port `5432`

↓

**Original Instance Has the Table Again**

---

# 29. Quick Command Reference

**Take base backup:**

`pg_basebackup -D /var/lib/pgsql/backup -X fetch -P -c fast`

**Force WAL switch:**

`SELECT pg_switch_wal();`

**Create recovery signal:**

`touch /var/lib/pgsql/backup/recovery.signal`

**Set permissions:**

`chmod 0700 /var/lib/pgsql/backup`

**Start PITR instance:**

`/usr/pgsql-18/bin/pg_ctl -D /var/lib/pgsql/backup start`

**Connect to PITR instance:**

`psql -p 5433`

**Check recovery status:**

`SELECT pg_is_in_recovery();`

**Resume paused recovery:**

`SELECT pg_wal_replay_resume();`

**Verify recovered table:**

`SELECT count(*) FROM pitrtest;`

**Move recovered table back to the original instance:**

`pg_dump -d postgres -p 5433 -t pitrtest | psql -d postgres -p 5432`

**Verify original instance:**

`SELECT count(*) FROM pitrtest;`

---

## 30. One-Line Definition for Class

**PostgreSQL Point-in-Time Recovery (PITR) restores a PostgreSQL cluster from a base backup and replays archived WAL records until a specified recovery target, such as a time immediately before an accidental data change.**
