# PostgreSQL 18.4 Source Build, Administration, Backup & PITR

## Complete Hands-On DBA Lab Guide

**Platform used in this lab**

* PostgreSQL: **18.4**
* Installation method: **Build from source**
* Linux: Ubuntu under WSL2
* PostgreSQL software home: `/opt/postgresql/18`
* Primary port: `5432`
* Restore-validation port: `5433`
* PITR-validation port: `5434`
* PostgreSQL OS account: `postgres`
* DBA login account used during build: `dell`

---

# 1. Target Architecture

The completed lab uses this layout:

```text
/opt/postgresql/18/
├── bin/
├── include/
├── lib/
└── share/

/var/lib/postgresql/18/
├── data/                       # Primary PGDATA
├── wal/                        # Primary active WAL
├── log/                        # Primary PostgreSQL logs
├── backup/                     # Physical base backups
├── archive_log/                # Archived WAL
├── restore_test/               # Restore-validation instance
│   ├── data/
│   ├── wal/
│   └── log/
└── pitr_test/                  # PITR-validation instance
    ├── data/
    ├── wal/
    └── log/
```

Primary architecture:

```text
                     PostgreSQL 18.4
                           |
                    systemd service
                           |
                  postgresql-18.service
                           |
              +------------+------------+
              |                         |
           PGDATA                    Active WAL
              |                         |
/var/lib/postgresql/18/data    /var/lib/postgresql/18/wal
              |
              +---- Logs
              |     /var/lib/postgresql/18/log
              |
              +---- Base Backups
              |     /var/lib/postgresql/18/backup
              |
              +---- Archived WAL
                    /var/lib/postgresql/18/archive_log
```

---

# 2. Source Tree

The PostgreSQL 18.4 source tree used in this lab is:

```bash
/usr/local/src/postgresql/postgresql-18.4
```

Enter it:

```bash
cd /usr/local/src/postgresql/postgresql-18.4
```

Verify:

```bash
pwd
```

Expected:

```text
/usr/local/src/postgresql/postgresql-18.4
```

---

# 3. Configure the PostgreSQL Source Build

Configure PostgreSQL with the installation prefix:

```bash
./configure --prefix=/opt/postgresql/18
```

After configuration, verify the prefix:

```bash
grep '^prefix' src/Makefile.global
```

Expected:

```text
prefix := /opt/postgresql/18
```

This is important because `make install` will install PostgreSQL under this location.

---

# 4. Compile PostgreSQL

Compile using all available CPU cores:

```bash
make -j$(nproc)
```

Check the return code immediately afterward:

```bash
echo $?
```

Expected:

```text
0
```

Do not assume a build succeeded merely because `cd` or another unrelated command returned `0`.

---

# 5. Locate the Compiled PostgreSQL Binaries

Do not guess source-tree binary locations.

Locate the server:

```bash
find src -type f -name postgres -perm -111 2>/dev/null
```

Result:

```text
src/backend/postgres
```

Locate `psql`:

```bash
find src -type f -name psql -perm -111 2>/dev/null
```

Result:

```text
src/bin/psql/psql
```

A key PostgreSQL source-tree detail is:

```text
src/backend/postgres
```

is itself the executable.

It is **not**:

```text
src/backend/postgres/postgres
```

Verify:

```bash
ls -ld src/backend/postgres
```

Example:

```text
-rwxr-xr-x ... src/backend/postgres
```

Test the source-tree binaries:

```bash
./src/backend/postgres --version
./src/bin/psql/psql --version
```

Expected:

```text
postgres (PostgreSQL) 18.4
psql (PostgreSQL) 18.4
```

---

# 6. Run PostgreSQL Regression Tests

Before installation:

```bash
make check
```

The lab result was:

```text
1..231
# All 231 tests passed.
```

This establishes:

```text
configure        PASS
make             PASS
postgres binary  PASS
psql binary      PASS
make check       PASS — 231/231
```

Do not proceed to installation if regression tests fail without first investigating the failure.

---

# 7. Install PostgreSQL

Verify the prefix one final time:

```bash
grep '^prefix' src/Makefile.global
```

Expected:

```text
prefix := /opt/postgresql/18
```

Install:

```bash
sudo make install
```

Verify installed binaries:

```bash
/opt/postgresql/18/bin/postgres --version
/opt/postgresql/18/bin/psql --version
/opt/postgresql/18/bin/initdb --version
```

Expected:

```text
postgres (PostgreSQL) 18.4
psql (PostgreSQL) 18.4
initdb (PostgreSQL) 18.4
```

---

# 8. Create the PostgreSQL Operating-System Account

Check first:

```bash
getent passwd postgres
```

If nothing is returned, create the account:

```bash
sudo useradd \
  --system \
  --create-home \
  --home-dir /var/lib/postgresql \
  --shell /bin/bash \
  postgres
```

Verify:

```bash
getent passwd postgres
id postgres
```

Lab result:

```text
postgres:x:999:986::/var/lib/postgresql:/bin/bash

uid=999(postgres) gid=986(postgres) groups=986(postgres)
```

Never run the PostgreSQL database server as `root`.

---

# 9. Create the Storage Layout

Create separate areas for:

* PGDATA
* active WAL
* logs
* backups
* archived WAL

Run:

```bash
sudo mkdir -p /var/lib/postgresql/18/{data,wal,log,backup,archive_log}
```

Set ownership:

```bash
sudo chown -R postgres:postgres /var/lib/postgresql/18
```

Secure the directories:

```bash
sudo chmod 700 /var/lib/postgresql/18/data
sudo chmod 700 /var/lib/postgresql/18/wal
sudo chmod 700 /var/lib/postgresql/18/log
sudo chmod 700 /var/lib/postgresql/18/backup
sudo chmod 700 /var/lib/postgresql/18/archive_log
```

Verify as root:

```bash
sudo ls -ld /var/lib/postgresql
sudo ls -ld /var/lib/postgresql/18
sudo ls -ld /var/lib/postgresql/18/{data,wal,log,backup,archive_log}
```

Expected child permissions:

```text
drwx------ postgres postgres ... archive_log
drwx------ postgres postgres ... backup
drwx------ postgres postgres ... data
drwx------ postgres postgres ... log
drwx------ postgres postgres ... wal
```

Also verify as `postgres`:

```bash
sudo -u postgres ls -ld /var/lib/postgresql/18
sudo -u postgres ls -ld /var/lib/postgresql/18/{data,wal,log,backup,archive_log}
```

## Why the normal DBA account may get Permission Denied

If:

```bash
ls -ld /var/lib/postgresql/18
```

returns:

```text
Permission denied
```

inspect the parent:

```bash
sudo ls -ld /var/lib/postgresql
```

In this lab:

```text
drwxr-x--- postgres postgres /var/lib/postgresql
```

The normal `dell` account therefore cannot traverse the directory.

Do **not** weaken PostgreSQL directory permissions merely to make inspection easier.

Use:

```bash
sudo ls ...
```

or:

```bash
sudo -u postgres ls ...
```

---

# 10. Initialize the Primary Cluster with Separate WAL

Run `initdb` as the `postgres` OS account:

```bash
sudo -u postgres /opt/postgresql/18/bin/initdb \
  -D /var/lib/postgresql/18/data \
  --waldir=/var/lib/postgresql/18/wal
```

Important options:

```text
-D
```

specifies `PGDATA`.

```text
--waldir
```

places active WAL in a separate directory.

The successful initialization reported:

```text
Data page checksums are enabled.

Success. You can now start the database server...
```

It also warned:

```text
enabling "trust" authentication for local connections
```

That will be corrected later.

---

# 11. Verify PGDATA and WAL

Check the WAL link:

```bash
sudo -u postgres ls -ld /var/lib/postgresql/18/data/pg_wal
```

Expected:

```text
/var/lib/postgresql/18/data/pg_wal -> /var/lib/postgresql/18/wal
```

Resolve it:

```bash
sudo -u postgres readlink -f /var/lib/postgresql/18/data/pg_wal
```

Expected:

```text
/var/lib/postgresql/18/wal
```

Verify cluster major version:

```bash
sudo -u postgres cat /var/lib/postgresql/18/data/PG_VERSION
```

Expected:

```text
18
```

---

# 12. Understand WAL vs Archived WAL

These are different things.

## Active WAL

```text
/var/lib/postgresql/18/wal
```

contains WAL currently managed by PostgreSQL.

## Archived WAL

```text
/var/lib/postgresql/18/archive_log
```

contains completed WAL segments copied by `archive_command`.

Conceptually:

```text
Transactions
     |
     v
Active WAL
/var/lib/postgresql/18/wal
     |
     | completed segment
     v
archive_command
     |
     v
Archived WAL
/var/lib/postgresql/18/archive_log
```

Never manually create fake WAL files in either directory.

---

# 13. Configure Logging and WAL Archiving

These are PostgreSQL configuration parameters, **not Linux shell commands**.

Do not type:

```text
logging_collector = on
```

at the Bash prompt.

That causes:

```text
logging_collector: command not found
```

Instead edit:

```bash
sudo -u postgres nano /var/lib/postgresql/18/data/postgresql.conf
```

Add:

```conf
logging_collector = on
log_directory = '/var/lib/postgresql/18/log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/18/archive_log/%f && cp %p /var/lib/postgresql/18/archive_log/%f'
```

Verify:

```bash
sudo -u postgres grep -E \
'^(logging_collector|log_directory|log_filename|archive_mode|archive_command)' \
/var/lib/postgresql/18/data/postgresql.conf
```

Expected:

```text
logging_collector = on
log_directory = '/var/lib/postgresql/18/log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/18/archive_log/%f && cp %p /var/lib/postgresql/18/archive_log/%f'
```

---

# 14. Review Initial Authentication

Inspect active `pg_hba.conf` lines:

```bash
sudo -u postgres grep -vE '^[[:space:]]*(#|$)' \
/var/lib/postgresql/18/data/pg_hba.conf
```

Immediately after `initdb`, the lab had:

```text
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
```

Do not switch to SCRAM before setting a password for `postgres`, otherwise you can create an avoidable authentication problem.

The safe sequence is:

```text
temporary trust
      |
      v
start PostgreSQL
      |
      v
set postgres role password
      |
      v
verify SCRAM password encryption
      |
      v
change pg_hba.conf
      |
      v
reload
      |
      v
test TCP password authentication
```

---

# 15. First PostgreSQL Startup

Start temporarily using `pg_ctl`:

```bash
sudo -u postgres /opt/postgresql/18/bin/pg_ctl \
  -D /var/lib/postgresql/18/data \
  -l /var/lib/postgresql/18/log/startup.log \
  start
```

Expected:

```text
waiting for server to start.... done
server started
```

Check:

```bash
sudo -u postgres /opt/postgresql/18/bin/pg_ctl \
  -D /var/lib/postgresql/18/data \
  status
```

Expected:

```text
pg_ctl: server is running
```

---

# 16. Verify Runtime Configuration

Connect:

```bash
sudo -u postgres /opt/postgresql/18/bin/psql
```

Run:

```sql
SELECT version();

SHOW data_directory;
SHOW hba_file;
SHOW config_file;
SHOW archive_mode;
SHOW archive_command;
SHOW logging_collector;
SHOW log_directory;
```

Lab results included:

```text
PostgreSQL 18.4
```

```text
data_directory
/var/lib/postgresql/18/data
```

```text
archive_mode
on
```

```text
logging_collector
on
```

```text
log_directory
/var/lib/postgresql/18/log
```

---

# 17. Configure SCRAM Authentication

While connected through the temporary `trust` configuration, set the PostgreSQL `postgres` role password:

```text
\password postgres
```

Enter the password twice.

Do not post or store that password in lab notes.

Verify:

```sql
SHOW password_encryption;
```

Expected:

```text
scram-sha-256
```

Exit:

```text
\q
```

## Important psql lesson

`\password` is a **psql meta-command**.

It works at:

```text
postgres=#
```

It does **not** work at:

```text
dell@PMSQLDBA:~$
```

Running it from Bash results in something like:

```text
Command 'password' not found
```

Also execute `\password` separately.

Do not paste:

```text
\password postgres
SHOW password_encryption;
```

as though both were SQL.

Instead:

```text
postgres=# \password postgres
```

Finish the password prompts, then:

```sql
SHOW password_encryption;
```

---

# 18. Change pg_hba.conf to SCRAM

Edit:

```bash
sudo -u postgres nano /var/lib/postgresql/18/data/pg_hba.conf
```

Replace the `trust` rules with:

```conf
local   all             all                                     scram-sha-256
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256

local   replication     all                                     scram-sha-256
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```

Reload:

```bash
sudo -u postgres /opt/postgresql/18/bin/pg_ctl \
  -D /var/lib/postgresql/18/data \
  reload
```

Expected:

```text
server signaled
```

---

# 19. Verify SCRAM Authentication

Connect explicitly through TCP:

```bash
/opt/postgresql/18/bin/psql \
  -h 127.0.0.1 \
  -U postgres \
  -d postgres
```

A password should be requested.

Verify:

```sql
SELECT current_user, current_database();
```

Expected:

```text
postgres | postgres
```

Verify HBA parsing:

```sql
SELECT line_number,
       type,
       database,
       user_name,
       address,
       auth_method
FROM pg_hba_file_rules
ORDER BY line_number;
```

The lab showed all six entries using:

```text
scram-sha-256
```

---

# 20. Test WAL Archiving

Force a WAL switch:

```sql
SELECT pg_switch_wal();
```

Inspect archiver statistics:

```sql
SELECT archived_count,
       failed_count,
       last_archived_wal,
       last_archived_time,
       last_failed_wal,
       last_failed_time
FROM pg_stat_archiver;
```

A healthy result has:

```text
archived_count > 0
failed_count   = 0
```

Exit:

```text
\q
```

Verify physically:

```bash
sudo -u postgres ls -lh /var/lib/postgresql/18/archive_log
```

Example:

```text
-rw------- postgres postgres 16M ... 000000010000000000000001
```

The standard WAL segment size in this build is 16 MB.

A second switch produced additional files:

```text
000000010000000000000001
000000010000000000000002
000000010000000000000003
```

---

# 21. Create a systemd Service

Create:

```bash
sudo nano /etc/systemd/system/postgresql-18.service
```

Use:

```ini
[Unit]
Description=PostgreSQL 18.4 Database Server
After=network.target

[Service]
Type=forking
User=postgres
Group=postgres

Environment=PGDATA=/var/lib/postgresql/18/data

ExecStart=/opt/postgresql/18/bin/pg_ctl start -D ${PGDATA} -s -w -t 300
ExecStop=/opt/postgresql/18/bin/pg_ctl stop -D ${PGDATA} -s -m fast
ExecReload=/opt/postgresql/18/bin/pg_ctl reload -D ${PGDATA} -s

TimeoutSec=300
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Because the server was started manually, stop it once before transferring control:

```bash
sudo -u postgres /opt/postgresql/18/bin/pg_ctl \
  -D /var/lib/postgresql/18/data \
  stop -m fast
```

Enable:

```bash
sudo systemctl enable postgresql-18
```

Start:

```bash
sudo systemctl start postgresql-18
```

Check:

```bash
sudo systemctl status postgresql-18 --no-pager
```

Expected:

```text
Active: active (running)
```

Verify boot enablement:

```bash
systemctl is-enabled postgresql-18
```

Expected:

```text
enabled
```

---

# 22. PostgreSQL Service Management

After systemd takes ownership, manage the primary using:

```bash
sudo systemctl start postgresql-18
sudo systemctl stop postgresql-18
sudo systemctl restart postgresql-18
sudo systemctl reload postgresql-18
sudo systemctl status postgresql-18
```

Avoid manually starting the primary using `pg_ctl` while systemd is supposed to own it.

---

# 23. Verify PostgreSQL After systemd Migration

Run:

```bash
/opt/postgresql/18/bin/psql \
  -h 127.0.0.1 \
  -U postgres \
  -d postgres \
  -c "SELECT version();"
```

Expected:

```text
PostgreSQL 18.4
```

Test WAL archiving again:

```bash
/opt/postgresql/18/bin/psql \
  -h 127.0.0.1 \
  -U postgres \
  -d postgres
```

Then:

```sql
SELECT pg_switch_wal();

SELECT archived_count,
       failed_count,
       last_archived_wal,
       last_archived_time
FROM pg_stat_archiver;
```

Verify physical files:

```bash
sudo -u postgres ls -lh /var/lib/postgresql/18/archive_log
```

This proves archiving continues correctly when PostgreSQL is managed by systemd.

---

# 24. Configure the postgres Shell Environment

Create/edit:

```bash
sudo -u postgres nano /var/lib/postgresql/.bash_profile
```

Add:

```bash
# PostgreSQL 18 Environment

export PGHOME=/opt/postgresql/18
export PGDATA=/var/lib/postgresql/18/data
export PGBACKUP=/var/lib/postgresql/18/backup
export PGARCHIVE=/var/lib/postgresql/18/archive_log
export PGLOG=/var/lib/postgresql/18/log

export PATH=$PGHOME/bin:$PATH

export PGHOST=127.0.0.1
export PGPORT=5432
```

Do not put passwords here.

Login:

```bash
sudo -iu postgres
```

Load explicitly if necessary:

```bash
source ~/.bash_profile
```

Verify:

```bash
echo $PGHOME
echo $PGDATA
echo $PGBACKUP
echo $PGARCHIVE
echo $PGLOG
echo $PGHOST
echo $PGPORT
```

Expected:

```text
/opt/postgresql/18
/var/lib/postgresql/18/data
/var/lib/postgresql/18/backup
/var/lib/postgresql/18/archive_log
/var/lib/postgresql/18/log
127.0.0.1
5432
```

Verify executables:

```bash
which psql
which pg_ctl
which pg_dump
which pg_basebackup
which pg_verifybackup
```

Expected:

```text
/opt/postgresql/18/bin/psql
/opt/postgresql/18/bin/pg_ctl
/opt/postgresql/18/bin/pg_dump
/opt/postgresql/18/bin/pg_basebackup
/opt/postgresql/18/bin/pg_verifybackup
```

Verify versions:

```bash
psql --version
pg_basebackup --version
```

Expected:

```text
PostgreSQL 18.4
```

---

# 25. Optional Environment for the DBA Login Account

For the normal DBA login account:

```bash
nano ~/.bashrc
```

Add:

```bash
# PostgreSQL 18 DBA Environment

export PGHOME=/opt/postgresql/18
export PGDATA=/var/lib/postgresql/18/data
export PGBACKUP=/var/lib/postgresql/18/backup
export PGARCHIVE=/var/lib/postgresql/18/archive_log
export PGLOG=/var/lib/postgresql/18/log

export PATH=$PGHOME/bin:$PATH

export PGHOST=127.0.0.1
export PGPORT=5432
```

Load:

```bash
source ~/.bashrc
```

This allows:

```bash
psql
pg_dump
pg_basebackup
pg_verifybackup
```

without typing `/opt/postgresql/18/bin/`.

Having `pg_ctl` on the PATH does **not** mean the normal DBA account should manually control the primary cluster.

Use systemd for the primary.

---

# 26. Take the First Physical Base Backup

Login as `postgres`:

```bash
sudo -iu postgres
```

Create a timestamped destination:

```bash
BACKUP_DIR="$PGBACKUP/base_$(date +%Y%m%d_%H%M%S)"
echo "$BACKUP_DIR"
```

Lab example:

```text
/var/lib/postgresql/18/backup/base_20260813_150239
```

Take the physical backup:

```bash
pg_basebackup \
  -h 127.0.0.1 \
  -U postgres \
  -D "$BACKUP_DIR" \
  -Fp \
  -X stream \
  -P
```

Options:

```text
-h 127.0.0.1
```

connect using TCP.

```text
-U postgres
```

connect as PostgreSQL role `postgres`.

```text
-D
```

backup destination.

```text
-Fp
```

plain-format backup.

```text
-X stream
```

stream required WAL during backup.

```text
-P
```

show progress.

The lab completed:

```text
23724/23724 kB (100%), 1/1 tablespace
```

Check immediately:

```bash
echo $?
```

Expected:

```text
0
```

---

# 27. Verify the Physical Backup

Run:

```bash
pg_verifybackup "$BACKUP_DIR"
```

Lab result:

```text
backup successfully verified
```

Check version:

```bash
cat "$BACKUP_DIR/PG_VERSION"
```

Expected:

```text
18
```

Inspect:

```bash
ls -lh "$BACKUP_DIR" | head -20
```

Important files include:

```text
PG_VERSION
backup_label
backup_manifest
base/
global/
pg_hba.conf
postgresql.conf
...
```

The `backup_manifest` allows `pg_verifybackup` to validate the backup contents.

---

# 28. Backup Architecture

At this stage:

```text
Physical Base Backup
/var/lib/postgresql/18/backup/base_YYYYMMDD_HHMMSS

                     +

Continuous WAL Archive
/var/lib/postgresql/18/archive_log/

                     =
                     
Foundation for PITR
```

A backup is not truly proven until recovery has been tested.

---

# 29. Create a Restore-Validation Cluster

Never restore over a healthy primary merely to test a backup.

Use:

```text
Primary
Port:   5432
PGDATA: /var/lib/postgresql/18/data

Restore Test
Port:   5433
PGDATA: /var/lib/postgresql/18/restore_test/data
```

Create:

```bash
mkdir -p /var/lib/postgresql/18/restore_test/data
mkdir -p /var/lib/postgresql/18/restore_test/wal
mkdir -p /var/lib/postgresql/18/restore_test/log

chmod 700 /var/lib/postgresql/18/restore_test
chmod 700 /var/lib/postgresql/18/restore_test/data
chmod 700 /var/lib/postgresql/18/restore_test/wal
chmod 700 /var/lib/postgresql/18/restore_test/log
```

Restore:

```bash
BACKUP_DIR=/var/lib/postgresql/18/backup/base_20260813_150239

cp -a "$BACKUP_DIR"/. /var/lib/postgresql/18/restore_test/data/
```

---

# 30. Handle pg_wal Carefully During Restore

Inspect:

```bash
ls -ld /var/lib/postgresql/18/restore_test/data/pg_wal
```

Determine whether it is a directory or symlink:

```bash
if [ -L /var/lib/postgresql/18/restore_test/data/pg_wal ]; then
    echo "pg_wal is a symlink"
    readlink /var/lib/postgresql/18/restore_test/data/pg_wal
else
    echo "pg_wal is a directory"
fi
```

The restore cluster must never accidentally point at:

```text
/var/lib/postgresql/18/wal
```

because that is the primary's active WAL.

If it is a normal directory:

```bash
cp -a /var/lib/postgresql/18/restore_test/data/pg_wal/. \
      /var/lib/postgresql/18/restore_test/wal/

rm -rf /var/lib/postgresql/18/restore_test/data/pg_wal
```

If it is a symlink, remove only the link:

```bash
rm /var/lib/postgresql/18/restore_test/data/pg_wal
```

Then:

```bash
ln -s /var/lib/postgresql/18/restore_test/wal \
      /var/lib/postgresql/18/restore_test/data/pg_wal
```

Verify:

```bash
readlink -f /var/lib/postgresql/18/restore_test/data/pg_wal
```

Required:

```text
/var/lib/postgresql/18/restore_test/wal
```

---

# 31. Configure Restore-Validation Instance

Append to its `postgresql.conf`:

```bash
cat >> /var/lib/postgresql/18/restore_test/data/postgresql.conf <<'EOF'

# Restore-validation instance
port = 5433
listen_addresses = '127.0.0.1'

logging_collector = on
log_directory = '/var/lib/postgresql/18/restore_test/log'
log_filename = 'restore-test-%Y-%m-%d_%H%M%S.log'

archive_mode = off
EOF
```

The test instance should not write into the primary's WAL archive.

Check port:

```bash
ss -ltn | grep ':5433 ' || echo "Port 5433 is free"
```

Start:

```bash
pg_ctl \
  -D /var/lib/postgresql/18/restore_test/data \
  -l /var/lib/postgresql/18/restore_test/log/startup.log \
  start
```

Check:

```bash
pg_ctl -D /var/lib/postgresql/18/restore_test/data status
```

Verify primary:

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d postgres \
  -c "SELECT version(), current_setting('port'), current_setting('data_directory');"
```

Verify restore:

```bash
psql -h 127.0.0.1 -p 5433 -U postgres -d postgres \
  -c "SELECT version(), current_setting('port'), current_setting('data_directory');"
```

Expected separation:

```text
PRIMARY
Port   5432
PGDATA /var/lib/postgresql/18/data

RESTORE TEST
Port   5433
PGDATA /var/lib/postgresql/18/restore_test/data
```

After validation:

```bash
pg_ctl \
  -D /var/lib/postgresql/18/restore_test/data \
  stop -m fast
```

---

# 32. PITR Scenario — Simulate Accidental Data Loss

Connect to primary:

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d postgres
```

Create database:

```sql
CREATE DATABASE pitr_lab;
```

Connect using a **separate psql meta-command**:

```text
\c pitr_lab
```

Do not accidentally paste `\c` together with unrelated SQL.

Create test table:

```sql
CREATE TABLE employees (
    id      integer PRIMARY KEY,
    name    text NOT NULL,
    salary  numeric(10,2)
);
```

Insert:

```sql
INSERT INTO employees VALUES
(1, 'Alice', 70000),
(2, 'Bob',   80000);
```

Verify:

```sql
SELECT * FROM employees;
```

Expected:

```text
1 | Alice | 70000.00
2 | Bob   | 80000.00
```

---

# 33. Record the PITR Recovery Target

Run:

```sql
SELECT clock_timestamp() AS recovery_target;
```

The successful lab target was:

```text
2026-08-13 15:19:03.756569-05
```

The target must represent a point where the desired data definitely exists.

Wait to establish separation:

```sql
SELECT pg_sleep(10);
```

Then simulate accidental damage:

```sql
DELETE FROM employees;
```

Successful scenario:

```text
DELETE 2
```

Verify damage:

```sql
SELECT * FROM employees;
```

Expected:

```text
(0 rows)
```

Force WAL switch:

```sql
SELECT pg_switch_wal();
```

Check archive:

```sql
SELECT archived_count,
       failed_count,
       last_archived_wal,
       last_archived_time
FROM pg_stat_archiver;
```

The lab eventually showed:

```text
archived_count = 9
failed_count   = 0
```

This is the scenario:

```text
Base backup
    |
    v
WAL replay
    |
    +---- CREATE pitr_lab
    |
    +---- CREATE employees
    |
    +---- INSERT Alice
    |
    +---- INSERT Bob
    |
    +---- 15:19:03.756569-05  <-- recovery target
    |
    +---- DELETE 2            <-- accidental damage
    |
    v
Current primary = 0 rows
```

Goal:

```text
Recover to target
       |
       v
Alice + Bob exist
DELETE not replayed
```

---

# 34. Shell Commands vs SQL Commands

This was an important practical lesson during the lab.

At:

```text
pitr_lab=#
```

you are inside `psql`.

Do not type:

```bash
pg_ctl -D ...
```

there.

PostgreSQL will attempt to parse it as SQL and report:

```text
ERROR: syntax error at or near "pg_ctl"
```

Use:

```text
\q
```

to leave psql first.

Then execute `pg_ctl` at:

```text
postgres@PMSQLDBA:~$
```

Quick rule:

```text
postgres@host:~$       Linux shell
postgres=#             psql / SQL
pitr_lab=#             psql / SQL
```

---

# 35. Build the PITR Recovery Instance

Exit primary psql:

```text
\q
```

Create:

```bash
rm -rf /var/lib/postgresql/18/pitr_test

mkdir -p /var/lib/postgresql/18/pitr_test/data
mkdir -p /var/lib/postgresql/18/pitr_test/wal
mkdir -p /var/lib/postgresql/18/pitr_test/log

chmod 700 /var/lib/postgresql/18/pitr_test
chmod 700 /var/lib/postgresql/18/pitr_test/data
chmod 700 /var/lib/postgresql/18/pitr_test/wal
chmod 700 /var/lib/postgresql/18/pitr_test/log
```

Restore the verified base:

```bash
BACKUP_DIR=/var/lib/postgresql/18/backup/base_20260813_150239

cp -a "$BACKUP_DIR"/. /var/lib/postgresql/18/pitr_test/data/
```

---

# 36. Give PITR Its Own WAL Directory

Run:

```bash
if [ -L /var/lib/postgresql/18/pitr_test/data/pg_wal ]; then
    rm /var/lib/postgresql/18/pitr_test/data/pg_wal
else
    cp -a /var/lib/postgresql/18/pitr_test/data/pg_wal/. \
          /var/lib/postgresql/18/pitr_test/wal/
    rm -rf /var/lib/postgresql/18/pitr_test/data/pg_wal
fi
```

Create the link carefully:

```bash
ln -s /var/lib/postgresql/18/pitr_test/wal \
      /var/lib/postgresql/18/pitr_test/data/pg_wal
```

Verify:

```bash
ls -ld /var/lib/postgresql/18/pitr_test/data/pg_wal

readlink /var/lib/postgresql/18/pitr_test/data/pg_wal

readlink -f /var/lib/postgresql/18/pitr_test/data/pg_wal
```

Required:

```text
/var/lib/postgresql/18/pitr_test/wal
```

## Failure encountered in the lab

The initial `ln` command was accidentally incomplete:

```text
ln -s /var/lib/postgresql/18/pitr_test/wal \
```

Therefore:

```text
data/pg_wal
```

did not exist.

PostgreSQL then failed:

```text
pg_ctl: could not start server
Examine the log output.
```

Checking:

```bash
ls -ld /var/lib/postgresql/18/pitr_test/data/pg_wal \
       /var/lib/postgresql/18/pitr_test/wal
```

revealed:

```text
data/pg_wal: No such file or directory
```

The correction was:

```bash
ln -s /var/lib/postgresql/18/pitr_test/wal \
      /var/lib/postgresql/18/pitr_test/data/pg_wal
```

After that:

```bash
readlink -f /var/lib/postgresql/18/pitr_test/data/pg_wal
```

correctly returned:

```text
/var/lib/postgresql/18/pitr_test/wal
```

This is an excellent recovery troubleshooting lesson: **verify storage paths before starting a restored cluster.**

---

# 37. Configure PITR

Append:

```bash
cat >> /var/lib/postgresql/18/pitr_test/data/postgresql.conf <<'EOF'

# PITR test instance
port = 5434
listen_addresses = '127.0.0.1'

logging_collector = on
log_directory = '/var/lib/postgresql/18/pitr_test/log'
log_filename = 'pitr-%Y-%m-%d_%H%M%S.log'

archive_mode = off

restore_command = 'cp /var/lib/postgresql/18/archive_log/%f %p'
recovery_target_time = '2026-08-13 15:19:03.756569-05'
recovery_target_action = 'pause'
EOF
```

Important parameters:

```conf
restore_command
```

tells PostgreSQL how to retrieve archived WAL.

```conf
recovery_target_time
```

tells PostgreSQL when to stop replay.

```conf
recovery_target_action = 'pause'
```

tells PostgreSQL to pause when the target is reached instead of immediately promoting.

---

# 38. Create recovery.signal

Run:

```bash
touch /var/lib/postgresql/18/pitr_test/data/recovery.signal
```

The presence of:

```text
recovery.signal
```

tells PostgreSQL to perform archive recovery.

Check port:

```bash
ss -ltn | grep ':5434 ' || echo "Port 5434 is free"
```

Expected:

```text
Port 5434 is free
```

---

# 39. Start the PITR Instance

Run:

```bash
pg_ctl \
  -D /var/lib/postgresql/18/pitr_test/data \
  -l /var/lib/postgresql/18/pitr_test/log/startup.log \
  start
```

Successful lab result:

```text
waiting for server to start..... done
server started
```

Check:

```bash
pg_ctl -D /var/lib/postgresql/18/pitr_test/data status
```

Example:

```text
pg_ctl: server is running
/opt/postgresql/18/bin/postgres "-D" "/var/lib/postgresql/18/pitr_test/data"
```

If startup fails:

```bash
tail -50 /var/lib/postgresql/18/pitr_test/log/startup.log
```

Do not immediately destroy and recreate the recovery environment. Inspect the error first.

---

# 40. Verify PITR

Connect specifically to port `5434`:

```bash
psql \
  -h 127.0.0.1 \
  -p 5434 \
  -U postgres \
  -d pitr_lab
```

Check recovery:

```sql
SELECT pg_is_in_recovery();
```

Lab result:

```text
t
```

Check replay state:

```sql
SELECT pg_get_wal_replay_pause_state();
```

Lab result:

```text
paused
```

Finally:

```sql
SELECT * FROM employees;
```

Lab result:

```text
 id | name  |  salary
----+-------+----------
  1 | Alice | 70000.00
  2 | Bob   | 80000.00
```

**PITR succeeded.**

The primary database had:

```text
0 rows
```

after the accidental delete.

The recovered database had:

```text
Alice
Bob
```

because WAL replay stopped before the damaging transaction.

---

# 41. PITR Architecture

The completed recovery flow is:

```text
Verified Base Backup
base_20260813_150239
        |
        v
Restore into isolated PGDATA
        |
        v
recovery.signal
        |
        v
restore_command
        |
        +------------------------------+
        |                              |
        v                              |
archive_log/                           |
000000010000000000000001               |
000000010000000000000002               |
000000010000000000000003               |
...                                    |
        |                              |
        v                              |
WAL replay ----------------------------+
        |
        v
2026-08-13 15:19:03.756569-05
        |
        X STOP / PAUSE
        |
        |   DELETE transaction exists later
        |   but is NOT replayed
        v

Recovered database:

Alice
Bob
```

---

# 42. Shut Down the PITR Test

Because this is a recovery test rather than a replacement primary, exit:

```text
\q
```

Stop:

```bash
pg_ctl \
  -D /var/lib/postgresql/18/pitr_test/data \
  stop -m fast
```

Verify:

```bash
pg_ctl -D /var/lib/postgresql/18/pitr_test/data status
```

Expected:

```text
no server running
```

The primary remains managed separately by:

```bash
sudo systemctl status postgresql-18
```

---

# 43. Port Map for the Lab

Keep this mapping clear:

| Instance     |   Port | Purpose                            |
| ------------ | -----: | ---------------------------------- |
| Primary      | `5432` | Main PostgreSQL 18.4 cluster       |
| Restore test | `5433` | Physical backup restore validation |
| PITR test    | `5434` | Point-in-time recovery validation  |

Never rely only on the prompt name when multiple instances are running.

Explicitly specify:

```bash
-h 127.0.0.1 -p PORT
```

---

# 44. Directory Map

## PostgreSQL software

```text
/opt/postgresql/18
```

## Primary PGDATA

```text
/var/lib/postgresql/18/data
```

## Primary active WAL

```text
/var/lib/postgresql/18/wal
```

## Logs

```text
/var/lib/postgresql/18/log
```

## Physical backups

```text
/var/lib/postgresql/18/backup
```

## Archived WAL

```text
/var/lib/postgresql/18/archive_log
```

## Restore validation

```text
/var/lib/postgresql/18/restore_test
```

## PITR validation

```text
/var/lib/postgresql/18/pitr_test
```

---

# 45. Essential DBA Commands

## PostgreSQL version

```bash
postgres --version
psql --version
pg_basebackup --version
```

## Primary service status

```bash
sudo systemctl status postgresql-18
```

## Restart

```bash
sudo systemctl restart postgresql-18
```

## Reload configuration

```bash
sudo systemctl reload postgresql-18
```

## Connect

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d postgres
```

## Show PGDATA

```sql
SHOW data_directory;
```

## Show config file

```sql
SHOW config_file;
```

## Show HBA file

```sql
SHOW hba_file;
```

## Verify archive mode

```sql
SHOW archive_mode;
SHOW archive_command;
```

## Force WAL switch

```sql
SELECT pg_switch_wal();
```

## Archiver health

```sql
SELECT *
FROM pg_stat_archiver;
```

## Recovery status

```sql
SELECT pg_is_in_recovery();
```

## Recovery pause status

```sql
SELECT pg_get_wal_replay_pause_state();
```

---

# 46. Troubleshooting Lessons From This Lab

## Problem 1 — Wrong source binary path

Wrong:

```text
src/backend/postgres/postgres
```

Correct:

```text
src/backend/postgres
```

Lesson:

```bash
find src -type f -name postgres -perm -111
```

is better than guessing.

---

## Problem 2 — Misinterpreting `$?`

Running:

```bash
cd ...
echo $?
```

only verifies the `cd`.

To verify `make`:

```bash
make -j$(nproc)
echo $?
```

must be executed consecutively.

---

## Problem 3 — Permission denied inspecting PGDATA

Cause:

```text
/var/lib/postgresql
```

was intentionally inaccessible to the normal user.

Solution:

```bash
sudo ls ...
```

or:

```bash
sudo -u postgres ls ...
```

Do not solve database security problems by blindly running:

```bash
chmod 777
```

---

## Problem 4 — PostgreSQL configuration entered into Bash

Wrong:

```bash
logging_collector = on
```

Correct location:

```text
postgresql.conf
```

Know which environment you are working in.

---

## Problem 5 — `\password` entered at Bash

Wrong environment:

```text
dell@PMSQLDBA:~$ \password postgres
```

Correct:

```text
postgres=# \password postgres
```

`\password` belongs to `psql`.

---

## Problem 6 — Shell command entered in psql

Wrong:

```text
pitr_lab=# pg_ctl ...
```

Correct:

```text
postgres@PMSQLDBA:~$ pg_ctl ...
```

Exit with:

```text
\q
```

first.

---

## Problem 7 — Incomplete pg_wal symlink

The incomplete command caused:

```text
data/pg_wal: No such file or directory
```

and PostgreSQL startup failed.

Always verify:

```bash
readlink -f "$PGDATA/pg_wal"
```

before starting a restored instance.

---

## Problem 8 — Repeated DELETE

The first valid destructive transaction returned:

```text
DELETE 2
```

A later repetition returned:

```text
DELETE 0
```

because the rows were already gone.

For PITR practice, explicitly establish:

```text
known good data
    ↓
record target
    ↓
known destructive transaction
```

Do not infer the timeline.

---

# 47. Operational Safety Rules

1. Never run PostgreSQL as root.

2. Never run `initdb` as root.

3. Never point a test restore at the primary's active WAL directory.

4. Never restore a backup directly over a healthy primary just to test it.

5. Never delete WAL manually from `pg_wal`.

6. Never assume archived WAL can be deleted merely because it looks old.

7. Never weaken PGDATA permissions just for convenience.

8. Never store database passwords in shell profiles.

9. Never treat a successful `pg_basebackup` as sufficient proof of recoverability.

10. Always run:

```bash
pg_verifybackup
```

and perform actual restore tests.

11. Use explicit ports when multiple PostgreSQL instances exist.

12. Keep the primary under one service-management mechanism—here, systemd.

---

# 48. Full Practice Sequence

For repeated practice, use this sequence:

```text
1. Obtain PostgreSQL source
             |
             v
2. ./configure --prefix=/opt/postgresql/18
             |
             v
3. make -j$(nproc)
             |
             v
4. Verify source-tree binaries
             |
             v
5. make check
             |
             v
6. sudo make install
             |
             v
7. Verify installed binaries
             |
             v
8. Create postgres OS account
             |
             v
9. Create data/wal/log/backup/archive directories
             |
             v
10. initdb --waldir
             |
             v
11. Verify pg_wal symlink
             |
             v
12. Configure logging
             |
             v
13. Configure WAL archiving
             |
             v
14. Start temporarily
             |
             v
15. Set postgres role password
             |
             v
16. Convert pg_hba.conf to SCRAM
             |
             v
17. Verify TCP authentication
             |
             v
18. Test WAL archiving
             |
             v
19. Create systemd service
             |
             v
20. Transfer primary control to systemd
             |
             v
21. Configure shell environment
             |
             v
22. pg_basebackup
             |
             v
23. pg_verifybackup
             |
             v
24. Restore test on 5433
             |
             v
25. Validate restored cluster
             |
             v
26. Create PITR test data
             |
             v
27. Record recovery target
             |
             v
28. Simulate DELETE
             |
             v
29. Force/archive WAL
             |
             v
30. Restore base backup
             |
             v
31. Configure restore_command
             |
             v
32. Configure recovery_target_time
             |
             v
33. Create recovery.signal
             |
             v
34. Start PITR instance on 5434
             |
             v
35. Verify recovery paused
             |
             v
36. Verify deleted rows recovered
```

---

# 49. What This Lab Proves

By completing the entire exercise successfully, you demonstrated hands-on understanding of:

* PostgreSQL source compilation
* PostgreSQL build-tree structure
* regression testing
* custom installation prefixes
* PostgreSQL OS-account management
* cluster initialization
* PGDATA management
* external WAL placement
* filesystem ownership and permissions
* PostgreSQL logging
* WAL archiving
* `pg_hba.conf`
* SCRAM-SHA-256 authentication
* PostgreSQL runtime configuration verification
* systemd integration
* service startup/shutdown/reload
* PostgreSQL environment variables
* `pg_basebackup`
* streamed backup WAL
* backup manifests
* `pg_verifybackup`
* physical restore testing
* multi-instance PostgreSQL administration
* WAL switching
* `pg_stat_archiver`
* recovery configuration
* `restore_command`
* `recovery.signal`
* `recovery_target_time`
* recovery pause behavior
* Point-in-Time Recovery
* recovery troubleshooting
* separation between Linux shell, psql meta-commands, and SQL

---

# 50. Final Lab State

```text
PostgreSQL 18.4
Source Build                              PASS

configure                                 PASS
make                                      PASS
make check                                PASS — 231/231
make install                              PASS

postgres OS account                       PASS
PGDATA                                    PASS
separate WAL                              PASS
separate logs                             PASS
backup directory                          PASS
WAL archive directory                     PASS

SCRAM-SHA-256 authentication              PASS

systemd service                           PASS
enabled at boot                           PASS

WAL archiving                             PASS
archive failures                          0

Physical pg_basebackup                    PASS
pg_verifybackup                           PASS

Restore-validation architecture           PASS

PITR simulation                           PASS
recovery target                           PASS
archived WAL replay                       PASS
recovery paused at target                 PASS
Alice/Bob recovered                       PASS
accidental DELETE excluded                PASS
```

## Core DBA Principle Demonstrated

A backup command completing successfully is only the beginning.

The stronger standard is:

```text
BACKUP
   +
VERIFY
   +
RESTORE
   +
RECOVER
   +
VALIDATE DATA
   =
PROVEN RECOVERABILITY
```

That is the central lesson of this PostgreSQL 18.4 lab.
