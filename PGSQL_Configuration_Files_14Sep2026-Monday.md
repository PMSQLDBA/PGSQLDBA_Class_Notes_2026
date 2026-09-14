In PostgreSQL, configuration is mainly controlled through a few important files.

The exact files available can vary by PostgreSQL version, operating system, and installation method, but the following are the **most important files a PostgreSQL DBA should know**.

## Important PostgreSQL Configuration Files

| # | Configuration File         | Main Purpose                                    | DBA Importance |
| - | -------------------------- | ----------------------------------------------- | -------------- |
| 1 | `postgresql.conf`          | Main PostgreSQL server configuration            | ⭐⭐⭐⭐⭐          |
| 2 | `pg_hba.conf`              | Client authentication and access control        | ⭐⭐⭐⭐⭐          |
| 3 | `pg_ident.conf`            | Maps operating-system users to PostgreSQL users | ⭐⭐⭐            |
| 4 | `postgresql.auto.conf`     | Stores settings changed using `ALTER SYSTEM`    | ⭐⭐⭐⭐           |
| 5 | `recovery.signal`          | Tells PostgreSQL to start recovery              | ⭐⭐⭐            |
| 6 | `standby.signal`           | Makes a server start as a standby               | ⭐⭐⭐⭐           |
| 7 | `tablespace_map`           | Maps tablespaces during recovery/restore        | ⭐⭐             |
| 8 | `pg_internal.init`         | Internal system-cache initialization file       | ⭐              |
| 9 | `postgresql.conf` includes | Allows modular configuration files              | ⭐⭐⭐⭐           |

> **Important:** `recovery.conf` is **not used in modern PostgreSQL versions**. 
Starting with PostgreSQL 12, recovery/standby settings are handled through `postgresql.conf`/`postgresql.auto.conf` together with signal files such as `standby.signal` and `recovery.signal`.

---

# 1. `postgresql.conf`

### What is it?

`postgresql.conf` is the **primary PostgreSQL server configuration file**.

Think of it as the main configuration file for the PostgreSQL instance.

### Common location

On Linux installations, it is commonly located under the PostgreSQL data directory, for example:

```text
/var/lib/pgsql/18/data/postgresql.conf
```

or:

```text
/var/lib/postgresql/18/main/postgresql.conf
```

The exact location depends on the installation method.

### What does it control?

It contains settings for:

* Server listening address
* PostgreSQL port
* Memory
* Connections
* WAL
* Checkpoints
* Logging
* Query planning
* Autovacuum
* Replication
* SSL
* Parallel processing
* Background workers
* Monitoring

### Important parameters

#### Connection settings

```conf
listen_addresses = '*'
port = 5432
max_connections = 200
```

**Explanation:**

* `listen_addresses` → Determines which network interfaces PostgreSQL listens on.
* `port` → PostgreSQL network port.
* `max_connections` → Maximum concurrent client connections.

---

### Memory settings

```conf
shared_buffers = 4GB
work_mem = 64MB
maintenance_work_mem = 512MB
effective_cache_size = 12GB
```

These influence how PostgreSQL uses memory for:

* Data caching
* Sort operations
* Hash operations
* Index creation
* VACUUM
* Query planning

---

### WAL settings

```conf
wal_level = replica
max_wal_size = 4GB
min_wal_size = 1GB
```

These are important for:

* Crash recovery
* Streaming replication
* PITR
* WAL management

---

### Logging

```conf
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_min_duration_statement = 1000
```

These control PostgreSQL logging.

For example:

```text
log_min_duration_statement = 1000
```

means PostgreSQL can log statements that take **1 second or longer**.

---

### Autovacuum

```conf
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
```

These control automatic VACUUM/ANALYZE activity.

---

### Replication

Examples include:

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

These are commonly required for physical streaming replication.

---

# 2. `pg_hba.conf`

### What is it?

`pg_hba.conf` stands for:

**PostgreSQL Host-Based Authentication configuration file**

This file controls:

> **Who can connect, from where, to which database, using which authentication method.**

This is one of the **most important files for a PostgreSQL DBA**.

---

### Example

```conf
host    all    all    192.168.1.0/24    scram-sha-256
```

This means:

* `host` → TCP/IP connection
* `all` → Any database
* `all` → Any PostgreSQL user
* `192.168.1.0/24` → Clients from this network
* `scram-sha-256` → Password authentication using SCRAM

---

### Another example

```conf
host    dvdrental    appuser    192.168.1.50/32    scram-sha-256
```

This allows:

* Database → `dvdrental`
* User → `appuser`
* Client → `192.168.1.50`
* Authentication → SCRAM

---

### Authentication methods

Common methods include:

```text
scram-sha-256
md5
peer
trust
reject
cert
```

### Important DBA point

`postgresql.conf` determines **whether PostgreSQL accepts network connections**.

`pg_hba.conf` determines **whether a particular client/user is allowed to connect and how they authenticate**.

For example:

```text
postgresql.conf
       ↓
listen_addresses = '*'
       ↓
PostgreSQL listens for network connections
       ↓
pg_hba.conf
       ↓
Is this client/user/database allowed?
       ↓
Authentication
```

---

# 3. `pg_ident.conf`

### What is it?

`pg_ident.conf` is used for **mapping external operating-system usernames to PostgreSQL usernames**.

It is mainly useful with authentication methods such as:

```text
peer
ident
```

### Example

```conf
mymap    linuxuser    postgresuser
```

This creates a mapping between:

```text
Linux user
    ↓
linuxuser

PostgreSQL user
    ↓
postgresuser
```

Then `pg_hba.conf` can reference the map.

Example:

```conf
local   all   all   peer map=mymap
```

### Where is it useful?

It can be useful when:

* Linux/Unix users need database access
* OS usernames don't match PostgreSQL usernames
* You want controlled user mapping
* Using `peer` authentication

For many modern application environments using SCRAM/password or certificates, you may rarely need to modify this file.

---

# 4. `postgresql.auto.conf`

### What is it?

This file contains configuration settings that PostgreSQL writes when you use:

```sql
ALTER SYSTEM
```

For example:

```sql
ALTER SYSTEM SET work_mem = '64MB';
```

PostgreSQL writes the setting into:

```text
postgresql.auto.conf
```

You might see:

```conf
work_mem = '64MB'
```

---

### Why is it important?

It allows DBAs to change configuration using SQL instead of manually editing `postgresql.conf`.

Example:

```sql
ALTER SYSTEM SET max_connections = 200;
```

Then:

```sql
SHOW max_connections;
```

Depending on the parameter, you may need:

```sql
SELECT pg_reload_conf();
```

or a PostgreSQL restart.

---

### Important precedence concept

If the same parameter exists in both:

```text
postgresql.conf
postgresql.auto.conf
```

the value in:

```text
postgresql.auto.conf
```

takes precedence because it is processed later.

### DBA best practice

Don't blindly mix manual changes and `ALTER SYSTEM`.

For production, establish a clear configuration-management approach so DBAs know **where the authoritative value is maintained**.

---

# 5. `standby.signal`

### What is it?

`standby.signal` is a **signal file** that tells PostgreSQL:

> Start this PostgreSQL server as a standby and enter standby/recovery mode.

This is commonly used when configuring **physical streaming replication**.

---

### Example

On a standby server:

```text
$PGDATA/
   ├── postgresql.conf
   ├── pg_hba.conf
   ├── standby.signal
   └── ...
```

When PostgreSQL starts and sees:

```text
standby.signal
```

it starts in standby mode.

---

### Important use cases

Used for:

* Streaming replication
* Read replicas
* HA architectures
* Disaster recovery
* Standby servers

---

### Important point

The file itself doesn't contain the replication configuration.

The replication-related settings are configured through PostgreSQL parameters and connection information.

For example, a standby may use:

```conf
primary_conninfo = 'host=primary-db port=5432 user=replicator ...'
```

---

# 6. `recovery.signal`

### What is it?

`recovery.signal` tells PostgreSQL to enter **archive recovery**.

It is commonly associated with:

* Point-in-Time Recovery (PITR)
* Restoring a PostgreSQL backup
* WAL archive recovery

---

### Example

During PITR:

```text
$PGDATA/
    |
    +-- recovery.signal
    +-- postgresql.conf
    +-- ...
```

PostgreSQL sees:

```text
recovery.signal
```

and enters recovery mode.

---

### Example scenario

Suppose you have:

```text
Base Backup
    +
WAL Archive
    +
recovery.signal
```

PostgreSQL can replay WAL records and recover the database to the required recovery target.

---

# 7. `tablespace_map`

### What is it?

`tablespace_map` is an internal file associated with PostgreSQL recovery.

It contains information PostgreSQL needs to map tablespaces during certain recovery/restore operations.

### DBA relevance

You normally **do not manually edit this file**.

It becomes relevant when working with:

* Physical backups
* Recovery
* Tablespaces
* WAL replay

For normal PostgreSQL administration, treat it as an internal PostgreSQL-managed file.

---

# 8. `pg_internal.init`

### What is it?

`pg_internal.init` is an **internal PostgreSQL file** used for system catalog cache initialization.

You may find it inside database-specific directories under the PostgreSQL data directory.

For example:

```text
$PGDATA/base/<database_oid>/pg_internal.init
```

### Important points

* PostgreSQL manages it automatically.
* It is not a normal DBA configuration file.
* Do not manually edit it.
* It can be recreated by PostgreSQL when required.

You generally don't need to modify or manage this file.

---

# 9. Configuration Include Files

PostgreSQL also supports breaking configuration into multiple files.

For example:

```conf
include 'my_custom.conf'
```

or:

```conf
include_dir = 'conf.d'
```

You could have:

```text
$PGDATA/
│
├── postgresql.conf
├── pg_hba.conf
├── pg_ident.conf
│
└── conf.d/
    ├── memory.conf
    ├── logging.conf
    ├── replication.conf
    └── security.conf
```

This can make a production configuration much easier to manage.

---

# Recommended Production Structure

For a production PostgreSQL 18 server, I would conceptually organize configuration like this:

```text
$PGDATA/
│
├── postgresql.conf
├── postgresql.auto.conf
├── pg_hba.conf
├── pg_ident.conf
│
├── standby.signal          ← Standby server only
│
├── recovery.signal         ← PITR/archive recovery only
│
└── conf.d/
    ├── 01-connections.conf
    ├── 02-memory.conf
    ├── 03-wal.conf
    ├── 04-replication.conf
    ├── 05-logging.conf
    ├── 06-autovacuum.conf
    └── 07-security.conf
```

This is especially useful when multiple DBAs are managing the environment.

---

# How to Find the Actual Configuration Files

**Don't assume the configuration files are in `$PGDATA`.** PostgreSQL can be configured to keep them elsewhere.

From `psql`:

```sql
SHOW config_file;
```

Example:

```text
/var/lib/pgsql/18/data/postgresql.conf
```

Find `pg_hba.conf`:

```sql
SHOW hba_file;
```

Find `pg_ident.conf`:

```sql
SHOW ident_file;
```

Find the data directory:

```sql
SHOW data_directory;
```

Find the current configuration directory:

```sql
SHOW config_file;
```

---

# Very Useful DBA Queries

### 1. Show all configuration parameters

```sql
SELECT name,
       setting,
       unit,
       context,
       source
FROM pg_settings
ORDER BY name;
```

### 2. Find parameters changed from their default

```sql
SELECT name,
       setting,
       unit,
       source
FROM pg_settings
WHERE source <> 'default'
ORDER BY name;
```

### 3. Find configuration parameters that require restart

```sql
SELECT name,
       setting,
       context
FROM pg_settings
WHERE context = 'postmaster'
ORDER BY name;
```

`context = 'postmaster'` generally means the parameter requires a **server restart** to take effect.

### 4. Reload configuration

After changing reloadable parameters:

```sql
SELECT pg_reload_conf();
```

### 5. Check configuration errors

```sql
SELECT *
FROM pg_file_settings
WHERE error IS NOT NULL;
```

This is an **excellent DBA troubleshooting query**.

It helps identify configuration files containing invalid settings or syntax problems.

---

# Configuration Change Hierarchy

A useful way to remember PostgreSQL configuration is:

```text
                 PostgreSQL Configuration
                          |
          +---------------+---------------+
          |                               |
   postgresql.conf                  pg_hba.conf
          |                               |
   Server behavior                 Client access
          |
   +------+-------+
   |              |
Memory          WAL
Logging        Replication
Autovacuum     Connections
          |
   postgresql.auto.conf
          |
     ALTER SYSTEM
```

And for recovery:

```text
Primary
   |
   | Streaming Replication
   ↓
Standby
   |
   +-- standby.signal
```

For PITR:

```text
Base Backup
     +
WAL Archive
     +
recovery.signal
     ↓
Archive Recovery / PITR
```

## ⭐ PostgreSQL DBA Priority

If you're preparing for **real-time PostgreSQL DBA production support**, focus on these first:

| Priority | File / Concept            | What you should know                                       |
| -------- | ------------------------- | ---------------------------------------------------------- |
| 🔴 1     | `postgresql.conf`         | Memory, connections, WAL, logging, autovacuum, replication |
| 🔴 2     | `pg_hba.conf`             | Authentication and client access                           |
| 🟠 3     | `postgresql.auto.conf`    | `ALTER SYSTEM` and configuration precedence                |
| 🟠 4     | `standby.signal`          | Physical standby/streaming replication                     |
| 🟠 5     | `recovery.signal`         | PITR/archive recovery                                      |
| 🟡 6     | `pg_ident.conf`           | OS-to-PostgreSQL user mapping                              |
| 🟡 7     | `include_dir` / `include` | Modular production configuration                           |
| 🟢 8     | `tablespace_map`          | Recovery/internal handling                                 |
| 🟢 9     | `pg_internal.init`        | Internal PostgreSQL file; don't manually modify            |

**The two files you should be extremely comfortable with are `postgresql.conf` and `pg_hba.conf`.** Most day-to-day PostgreSQL configuration and connectivity troubleshooting starts there.
