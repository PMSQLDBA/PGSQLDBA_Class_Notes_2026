13-Day-21st-Aug-Thursday-METADATA TABLES-DATA DICTIONARY

### METADATA TABLES / DATA DICTIONARY

**Metadata:** Data about data.

Metadata helps us identify information such as:

* How many tables are available?
* How many schemas are available?
* How many sequences, functions, and triggers are available?
* How many sessions are connected?
* What is the replication status?

PostgreSQL provides system metadata mainly through:

**1. `pg_catalog`**

* PostgreSQL-specific system catalog schema.
* Contains system catalogs, built-in objects, and many useful system views.
* Used heavily by PostgreSQL DBAs for administration and troubleshooting.

**2. `information_schema`**

* Provides SQL-standard metadata views.
* Useful when we want a more portable, SQL-standard way of retrieving database metadata.

**Additional Note:**
Instead of saying these are "two hidden schemas," it is better to call them **system/metadata schemas**. 

PostgreSQL officially describes `pg_catalog` as the system catalog schema, while `information_schema` is automatically available in every database. ([PostgreSQL][1])

---

### `pg_catalog` Views

To list the views:

```sql
\dv pg_catalog.*
```

#### `pg_available_extension_versions`

Shows the available versions of PostgreSQL extensions.

```sql
SELECT *
FROM pg_available_extension_versions;
```

Useful columns include:

```text
name
version
installed
superuser
trusted
relocatable
schema
requires
comment
```

**Classroom Note:**
Use this view when you want to check which versions of an extension are available on the server.

---

#### `pg_available_extensions`

Shows the extensions available for installation and their default/installed versions.

```sql
SELECT *
FROM pg_available_extensions;
```

Important columns:

```text
name
default_version
installed_version
comment
```

**Example:**

```text
plpgsql    1.0    1.0
amcheck    1.5
bloom      1.0
dblink     1.2
```

**Additional Note:**
`installed_version` being `NULL` means the extension is available but is not currently installed in that database.

---

#### `pg_backend_memory_contexts`

Displays memory-context information for the **current server process/session**.

```sql
SELECT *
FROM pg_backend_memory_contexts;
```

It helps us understand memory allocated in different PostgreSQL memory contexts.

**Important DBA Note:**
Do not describe this simply as memory consumed "for each operation." It reports memory contexts for the server process attached to the current session.

---

#### `pg_config`

Displays PostgreSQL installation and build configuration information.

```sql
SELECT *
FROM pg_config;
```

Examples:

```text
BINDIR
DOCDIR
INCLUDEDIR
LIBDIR
PKGLIBDIR
SHAREDIR
SYSCONFDIR
```

**Classroom Note:**
Useful for checking where PostgreSQL binaries, libraries, include files, documentation, and other installation-related files are located.

---

#### `pg_file_settings`

Displays configuration settings read from PostgreSQL configuration files.

```sql
SELECT *
FROM pg_file_settings;
```

Important columns:

```text
sourcefile
sourceline
name
setting
applied
error
```

**DBA Use:**
Very useful when troubleshooting `postgresql.conf` configuration problems.

For example:

```sql
SELECT sourcefile,
       sourceline,
       name,
       setting,
       applied,
       error
FROM pg_file_settings
WHERE error IS NOT NULL;
```

**Additional Note:**
Your classroom note says this shows "parameters without `#`".  I would retain the trainer's idea but correct it to: **this view shows configuration-file entries and whether PostgreSQL can successfully apply them.**

---

### `pg_stat_activity` — Session Monitoring

One of the most important DBA monitoring views.

```sql
SELECT datname,
       pid,
       usename,
       application_name,
       client_addr,
       state,
       query
FROM pg_stat_activity;
```

It can show:

```text
Database name
PID
Username
Application name
Client address
Session state
Current/last query
```

**Classroom Note:**
Use `pg_stat_activity` to monitor database sessions and query activity.

**Useful DBA Query:**

```sql
SELECT datname,
       usename,
       state,
       count(*)
FROM pg_stat_activity
GROUP BY datname, usename, state
ORDER BY datname, usename;
```

This helps us understand how many sessions are connected and their states.

Your original notes correctly identify `pg_stat_activity` as the main session-monitoring view. 

---

### ARCHIVE LOG MANAGEMENT

WAL archiving preserves completed WAL segments outside `pg_wal`.

Check archive mode:

```sql
SHOW archive_mode;
```

Enable archiving in `postgresql.conf`:

```text
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/archive/%f'
```

Restart PostgreSQL because `archive_mode` requires server-start configuration:

```bash
systemctl restart postgresql-18
```

Verify:

```sql
SHOW archive_mode;
SHOW archive_command;
```

Monitor archiving:

```sql
SELECT *
FROM pg_stat_archiver;
```

Important columns:

```text
archived_count
last_archived_wal
last_archived_time
failed_count
last_failed_wal
last_failed_time
stats_reset
```

**DBA Note:**
Always check `failed_count` and `last_failed_time` when troubleshooting WAL archive problems.

**Additional Official Note:**
PostgreSQL archives **completed WAL segments** when `archive_mode` is enabled and an `archive_command` or `archive_library` is configured. ([PostgreSQL][2])

---

### PREPARED TRANSACTIONS

This section needs one important terminology correction from the classroom notes.

Your example uses:

```sql
BEGIN;

CREATE TABLE hr
(
    eno   int,
    ename varchar,
    esal  int
);

INSERT INTO hr VALUES (1, 'ram', 100);

PREPARE TRANSACTION 'ram_txn';
```

This is a **prepared transaction / two-phase commit (2PC)**, not an SQL prepared statement.

Check:

```sql
SELECT *
FROM pg_prepared_xacts;
```

Complete the transaction:

```sql
COMMIT PREPARED 'ram_txn';
```

Or cancel it:

```sql
ROLLBACK PREPARED 'ram_txn';
```

PostgreSQL officially defines `PREPARE TRANSACTION` as preparing the current transaction for **two-phase commit**. Once prepared, it is no longer tied to the original session. ([PostgreSQL][3])

**Important DBA Note:**
Do not leave prepared transactions open unnecessarily. They can continue holding locks and can prevent `VACUUM` from cleaning old row versions. ([PostgreSQL][3])

---

[1]: https://www.postgresql.org/docs/18/ddl-schemas.html "PostgreSQL: Documentation: 18: 5.10. Schemas"
[2]: https://www.postgresql.org/docs/18/runtime-config-wal.html "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[3]: https://www.postgresql.org/docs/18/sql-prepare-transaction.html "PostgreSQL: Documentation: 18: PREPARE TRANSACTION"
[4]: https://www.postgresql.org/docs/18/catalogs.html "PostgreSQL: Documentation: 18: Chapter 52. System Catalogs"
