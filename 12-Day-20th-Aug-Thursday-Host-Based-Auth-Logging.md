Reviewed PostgreSQL notes and checked the technical points against the current PostgreSQL 18.4 official documentation. Your notes cover `pg_hba.conf`, local/remote authentication, CIDR, role-based HBA rules, and PostgreSQL logging. 

A few important corrections are needed: 
`host` does not simply mean “remote”; it means TCP/IP connections, including TCP connections from localhost. 

`peer` is for local Unix-domain socket connections.

`OPENSSL` is not a `pg_hba.conf` authentication method; encrypted TCP connections are handled using SSL/TLS with record types such as `hostssl`. 

Also, MD5 was not limited to PostgreSQL 10—it remained supported for many later versions, but PostgreSQL 18 has deprecated MD5 password authentication. ([PostgreSQL][1])

=======================================================================

# PostgreSQL Host-Based Authentication and Logging

**PostgreSQL Version: 18.x**

---

## 1. Host-Based Authentication (HBA)

PostgreSQL controls client authentication mainly through the `pg_hba.conf` file.

HBA stands for **Host-Based Authentication**.

The file determines:

* Which clients are allowed to connect
* Which database they can connect to
* Which PostgreSQL role/user they can use
* Which client IP addresses or networks are allowed
* Which authentication method must be used

The default `pg_hba.conf` file is normally stored inside the PostgreSQL data directory.

Example:

```text
/var/lib/pgsql/18/data/pg_hba.conf
```

The basic format is:

```text
# TYPE  DATABASE  USER  ADDRESS  METHOD
```

Example:

```text
host    newdb     raj   49.205.108.95/32   scram-sha-256
```

### Important Rule

PostgreSQL reads `pg_hba.conf` entries from **top to bottom**.

The **first matching rule is used**.

If authentication fails for the first matching rule, PostgreSQL does not continue searching for another matching rule.

---

# 2. TYPE

The `TYPE` column specifies the type of connection.

Common values are:

### local

Used for connections through a Unix-domain socket.

Example:

```text
local   newdb   raj   scram-sha-256
```

There is no `ADDRESS` field for a `local` rule.

### host

Used for TCP/IP connections.

Example:

```text
host   newdb   raj   192.168.1.100/32   scram-sha-256
```

A `host` rule can match either SSL or non-SSL TCP/IP connections.

### hostssl

Matches TCP/IP connections only when SSL/TLS encryption is being used.

Example:

```text
hostssl   newdb   raj   192.168.1.100/32   scram-sha-256
```

SSL must also be enabled on the PostgreSQL server.

### hostnossl

Matches only TCP/IP connections that are **not using SSL**.

Example:

```text
hostnossl   newdb   raj   192.168.1.100/32   reject
```

A common security design is:

```text
hostnossl   newdb   raj   192.168.1.100/32   reject
hostssl     newdb   raj   192.168.1.100/32   scram-sha-256
```

This forces the client to use an encrypted SSL connection.

---

# 3. DATABASE

The `DATABASE` field specifies which database the rule applies to.

Example:

```text
host   newdb   raj   192.168.1.100/32   scram-sha-256
```

Here the rule applies only to:

```text
newdb
```

Special values such as `all` can also be used, but production environments should use the narrowest access required whenever practical.

---

# 4. USER

The `USER` field specifies the PostgreSQL role/user allowed by the rule.

Example:

```text
host   newdb   raj   192.168.1.100/32   scram-sha-256
```

This rule applies to user:

```text
raj
```

PostgreSQL can also match members of a PostgreSQL role by prefixing the role name with `+`.

Example:

```text
host   newdb   +app_users   192.168.1.0/24   scram-sha-256
```

This rule matches roles that are members of `app_users`.

---

# 5. ADDRESS

The `ADDRESS` field specifies the client IP address or network from which connections are accepted.

It applies to TCP/IP connection types such as:

```text
host
hostssl
hostnossl
```

For `local` connections, the address field is not used.

Example for one client:

```text
192.168.56.201/32
```

`/32` means the complete IPv4 address must match.

Example:

```text
host   newdb   raj   192.168.56.201/32   scram-sha-256
```

Only the client with IP:

```text
192.168.56.201
```

matches this rule.

---

# 6. CIDR Address Ranges

PostgreSQL supports CIDR notation for defining IP ranges.

Examples:

```text
192.168.1.10/32
192.168.1.0/24
10.0.0.0/8
0.0.0.0/0
```

### /32

Matches exactly one IPv4 address.

```text
192.168.1.10/32
```

### /24

Matches addresses in the corresponding 24-bit network.

For example:

```text
192.168.1.0/24
```

normally covers:

```text
192.168.1.0 - 192.168.1.255
```

### /0

```text
0.0.0.0/0
```

matches every IPv4 address.

This should generally **not** be used for unrestricted database access in production.

Prefer allowing only required:

* Application server networks
* VPN networks
* Bastion/jump hosts
* DBA administration networks

---

# 7. Authentication Methods

## peer

`peer` authentication obtains the operating-system user name of the connecting process and checks it against the requested PostgreSQL user.

It is used for local connections.

Example:

```text
local   all   postgres   peer
```

If the operating-system identity matches the expected PostgreSQL role, PostgreSQL can authenticate the connection without asking for a database password.

---

## trust

`trust` allows the connection without requiring authentication credentials.

Example:

```text
local   newdb   raj   trust
```

Then:

```bash
psql -d newdb -U raj
```

can connect without a password if this HBA rule is the first matching rule.

### Security Warning

`trust` should be used only in environments where every client capable of reaching the PostgreSQL server through that rule can be fully trusted.

Avoid using `trust` for normal production remote access.

---

## scram-sha-256

This is the recommended password authentication method.

Example:

```text
host   newdb   raj   192.168.1.10/32   scram-sha-256
```

The client must authenticate using the user's password.

SCRAM-SHA-256 provides stronger password authentication than PostgreSQL's older MD5 mechanism.

For PostgreSQL 18, this should normally be preferred for password-based authentication.

---

## md5

PostgreSQL historically supported MD5-based password authentication.

Example:

```text
host   newdb   raj   192.168.1.10/32   md5
```

However, **MD5 password authentication is deprecated in PostgreSQL 18 and is planned for removal in a future PostgreSQL release.**

New implementations should use:

```text
scram-sha-256
```

instead.

The default `password_encryption` setting in PostgreSQL 18 is:

```text
scram-sha-256
```

---

## password

PostgreSQL also supports:

```text
password
```

This method sends the password in clear text over the connection.

Therefore, it should not be used over an untrusted network unless the connection itself is protected by SSL/TLS.

SCRAM-SHA-256 is generally the better choice.

---

## reject

`reject` unconditionally rejects a matching connection.

Example:

```text
host   newdb   raj   192.168.1.100/32   reject
```

This is useful when a specific user or network needs to be blocked.

---

# 8. SSL/TLS Is Not an Authentication Method

`OPENSSL` is **not** a valid authentication method in `pg_hba.conf`.

SSL/TLS provides encryption for data travelling between the PostgreSQL client and server.

A typical SSL-enabled HBA entry is:

```text
hostssl   newdb   raj   192.168.1.100/32   scram-sha-256
```

Here:

```text
hostssl
```

requires an SSL/TLS TCP connection, while:

```text
scram-sha-256
```

performs password authentication.

Therefore:

```text
SSL/TLS              -> connection encryption
SCRAM-SHA-256        -> password authentication
```

These solve different security requirements.

---

# 9. Allow User raj to Connect Locally with a Password

Edit:

```bash
vi /var/lib/pgsql/18/data/pg_hba.conf
```

Add:

```text
# TYPE  DATABASE  USER  METHOD
local   newdb     raj   scram-sha-256
```

Reload PostgreSQL:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data reload
```

Test:

```bash
psql -d newdb -U raj
```

Expected:

```text
Password for user raj:
```

After successful authentication:

```text
newdb=>
```

---

# 10. Allow raj Without a Password

Change the HBA entry:

```text
local   newdb   raj   trust
```

Reload:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data reload
```

Connect:

```bash
psql -d newdb -U raj
```

No password will be requested when this `trust` rule matches.

Again, `trust` should be used carefully.

---

# 11. Reject User raj

Configure:

```text
local   newdb   raj   reject
```

Reload:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data reload
```

Test:

```bash
psql -d newdb -U raj
```

The connection should be rejected.

---

# 12. Enabling Remote PostgreSQL Connections

There are several layers involved in a remote PostgreSQL connection:

```text
Client
   |
   v
Network / Firewall / Cloud Security Group
   |
   v
PostgreSQL listen_addresses
   |
   v
pg_hba.conf
   |
   v
Authentication
   |
   v
Database privileges
```

All relevant layers must permit the connection.

---

# 13. Step 1 - Check Network Access

If the connection hangs or times out, first check:

* Cloud security group
* Network ACL
* Operating-system firewall
* Routing
* PostgreSQL port

PostgreSQL's default TCP port is:

```text
5432
```

---

# 14. Step 2 - Check listen_addresses

Check the current setting:

```sql
SHOW listen_addresses;
```

The default is normally:

```text
localhost
```

This means PostgreSQL accepts TCP connections only through the local loopback interface.

To listen on all available interfaces:

```text
listen_addresses = '*'
```

in:

```text
postgresql.conf
```

Example:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

Configure:

```text
listen_addresses = '*'
```

Because `listen_addresses` is a server-start parameter, a PostgreSQL **restart** is required.

Example:

```bash
systemctl restart postgresql-18
```

Verify:

```sql
SHOW listen_addresses;
```

Expected:

```text
*
```

### Production Recommendation

Instead of listening on every interface, use specific interfaces when possible.

For example:

```text
listen_addresses = 'localhost,172.31.1.108'
```

---

# 15. Step 3 - Configure pg_hba.conf

Assume the client IP is:

```text
49.205.108.95
```

Add:

```text
host   newdb   raj   49.205.108.95/32   scram-sha-256
```

Reload PostgreSQL:

```bash
/usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data reload
```

Now test from the client:

```bash
psql -h 65.0.71.207 -U raj -d newdb
```

The user should be prompted for a password.

---

# 16. Understanding Common Connection Errors

## Connection timeout / hanging

Usually investigate:

```text
Network routing
Firewall
Security group
Network ACL
```

---

## Connection refused

Example:

```text
connection to server failed: Connection refused
```

Common causes include:

* PostgreSQL is not running
* PostgreSQL is not listening on the requested interface
* Wrong port
* Firewall rejection

Check:

```sql
SHOW listen_addresses;
SHOW port;
```

---

## No pg_hba.conf entry

Example:

```text
FATAL: no pg_hba.conf entry for host ...
```

This means the client reached PostgreSQL, but PostgreSQL could not find an HBA rule matching the connection parameters.

Check:

```text
TYPE
DATABASE
USER
ADDRESS
SSL/non-SSL connection type
```

---

## Password authentication failed

Example:

```text
FATAL: password authentication failed for user "raj"
```

This normally means PostgreSQL found an applicable authentication rule but authentication was unsuccessful.

Possible reasons include:

* Incorrect password
* User has no password
* Authentication/password storage configuration mismatch
* Connecting as the wrong role

---

# 17. Validate pg_hba.conf

PostgreSQL provides the system view:

```sql
pg_hba_file_rules
```

It can help identify errors in HBA configuration.

Example:

```sql
SELECT *
FROM pg_hba_file_rules;
```

A particularly useful check is:

```sql
SELECT line_number,
       type,
       database,
       user_name,
       address,
       auth_method,
       error
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

If no rows are returned, PostgreSQL did not detect parsing errors in the HBA rules.

---

# 18. Reloading pg_hba.conf

After modifying `pg_hba.conf`, reload PostgreSQL:

```bash
pg_ctl reload
```

or:

```sql
SELECT pg_reload_conf();
```

A full restart is normally not required for `pg_hba.conf` changes.

However, settings such as:

```text
listen_addresses
```

require a server restart.

---

# 19. Role-Based HBA Authentication

Suppose we have users:

```text
u1
u2
u3
u4
u5
u6
```

Instead of adding six separate HBA rules, create a group role.

Recommended design:

```sql
CREATE ROLE app_users NOLOGIN;
```

Then grant membership:

```sql
GRANT app_users TO u1;
GRANT app_users TO u2;
GRANT app_users TO u3;
GRANT app_users TO u4;
GRANT app_users TO u5;
GRANT app_users TO u6;
```

Configure:

```text
host   newdb   +app_users   49.205.108.95/32   scram-sha-256
```

The `+` means:

> Match roles that are members of this PostgreSQL role.

This is cleaner than maintaining one HBA rule per user.

---

# 20. Important Difference: Authentication vs Authorization

`pg_hba.conf` controls **authentication and connection matching**.

It does not automatically grant privileges on database objects.

For example, allowing:

```text
host   newdb   +app_users   49.205.108.95/32   scram-sha-256
```

does not automatically give the users permission to:

```text
SELECT
INSERT
UPDATE
DELETE
CREATE
```

on tables.

Object privileges must be managed separately with SQL commands such as:

```sql
GRANT CONNECT ON DATABASE newdb TO app_users;
GRANT USAGE ON SCHEMA public TO app_users;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_users;
```

This distinction is important:

```text
pg_hba.conf
     |
     +--> Can the client authenticate/connect?

GRANT / REVOKE
     |
     +--> What can the authenticated user do?
```

---

# 21. Production HBA Recommendations

Avoid broad rules such as:

```text
host   all   all   0.0.0.0/0   trust
```

This is extremely permissive.

Prefer specific rules:

```text
hostssl   appdb   +app_users   10.20.30.0/24   scram-sha-256
```

Use:

* Specific databases
* Specific roles/group roles
* Specific IP ranges
* SCRAM-SHA-256
* SSL/TLS where network encryption is required

---

# PostgreSQL Server Logging

## 22. Logging Overview

PostgreSQL provides extensive server logging.

Server logs can capture information such as:

* Connections
* Disconnections
* SQL statements
* Errors
* Slow statements
* Autovacuum operations
* Temporary files
* Replication commands

These PostgreSQL server logs should not automatically be considered a complete security auditing solution. Logging and formal database auditing are related but different concepts.

---

# 23. Logging Configuration

Edit:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

Example configuration:

```text
log_destination = 'stderr'
logging_collector = on

log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

log_rotation_age = 1d
log_rotation_size = 10MB

log_connections = on
log_disconnections = on

log_line_prefix = '%a %t %u %d %h %p '

log_replication_commands = on
log_temp_files = 0
```

---

# 24. logging_collector

Configure:

```text
logging_collector = on
```

The logging collector is a background process that captures messages sent to `stderr` and redirects them into log files.

Important:

`logging_collector` can only be changed at server start.

Therefore, changing it requires a PostgreSQL restart rather than only a reload.

---

# 25. log_directory

Example:

```text
log_directory = 'pg_log'
```

When a relative path is used, PostgreSQL creates/uses the directory relative to the PostgreSQL data directory.

For example:

```text
/var/lib/pgsql/18/data/pg_log
```

---

# 26. log_filename

Example:

```text
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
```

This generates timestamp-based log file names.

Example:

```text
postgresql-2026-06-29_135055.log
```

---

# 27. Log Rotation

Configure:

```text
log_rotation_age = 1d
log_rotation_size = 10MB
```

`log_rotation_age` controls time-based creation of new log files.

`log_rotation_size` controls size-based creation of new log files.

Important:

**Rotation does not mean automatic retention/deletion.**

PostgreSQL can rotate to new log files, but old log files still need an appropriate retention/housekeeping strategy.

---

# 28. Slow Query Logging

Your original setting:

```text
log_min_duration_statement = 0
```

does **not** mean 10 seconds.

It means:

> Log the duration of every completed SQL statement.

For queries running 10 seconds or longer, configure:

```text
log_min_duration_statement = 10s
```

or:

```text
log_min_duration_statement = 10000
```

because a value without units is interpreted as milliseconds.

Recommended readable format:

```text
log_min_duration_statement = '10s'
```

---

# 29. log_statement

Possible settings include:

```text
none
ddl
mod
all
```

Example:

```text
log_statement = 'all'
```

This logs all SQL statements that PostgreSQL processes at the appropriate stage.

### Production Warning

Logging every SQL statement can:

* Generate very large log files
* Increase I/O
* Add overhead
* Expose sensitive SQL data
* Potentially record sensitive values contained in SQL statements

Therefore, `log_statement = 'all'` should be enabled only when there is a clear operational or auditing requirement.

---

# 30. log_connections

Enable:

```text
log_connections = on
```

This logs connection-related activity.

It is useful for troubleshooting client authentication and connection problems.

---

# 31. log_disconnections

Enable:

```text
log_disconnections = on
```

This records session termination information and session duration.

---

# 32. Autovacuum Logging

Example:

```text
log_autovacuum_min_duration = 0
```

A value of `0` logs all autovacuum actions.

A value of:

```text
-1
```

disables this logging.

In a busy production database, logging every autovacuum action may create considerable log volume.

---

# 33. Temporary File Logging

Example:

```text
log_temp_files = 0
```

This logs all temporary files.

Temporary files can be created when operations such as sorts or hashes exceed available memory.

This setting can therefore help investigate queries that are spilling operations to disk.

---

# 34. Replication Command Logging

Enable:

```text
log_replication_commands = on
```

This records replication commands and certain replication-slot-related activity.

It is useful when troubleshooting replication environments.

---

# 35. Log Line Prefix

Example:

```text
log_line_prefix = '%a %t %u %d %h %p '
```

Typical information can include:

```text
%a -> application name
%t -> timestamp
%u -> user name
%d -> database name
%h -> remote host
%p -> process ID
```

A useful prefix makes troubleshooting much easier because multiple sessions can be distinguished in the logs.

---

# 36. Log Housekeeping

Log files can continue accumulating over time.

Therefore, production environments should implement a retention policy.

Example policy:

```text
0 - 3 days:
Keep normal .log files.

3 - 30 days:
Compress older log files.

More than 30 days:
Delete according to the organization's retention policy.
```

Example conceptual workflow:

```text
Daily housekeeping job
       |
       +--> Compress logs older than 3 days
       |
       +--> Delete logs older than 30 days
```

The exact retention period should follow the organization's:

* Security policy
* Compliance requirements
* Troubleshooting requirements
* Storage capacity
* Audit requirements

---

# 37. Reading Compressed Logs

If older logs are compressed using gzip:

```bash
zcat postgresql-old.log.gz
```

To search them:

```bash
zgrep "ERROR" postgresql-old.log.gz
```

Example:

```bash
zgrep "raj" postgresql-old.log.gz
```

---

# 38. Finding the Current PostgreSQL Log

When PostgreSQL's logging collector is being used with supported destinations, PostgreSQL maintains:

```text
current_logfiles
```

This file identifies the log file currently in use.

Example:

```bash
cat /var/lib/pgsql/18/data/current_logfiles
```

This can be easier than manually searching the log directory.

---

# 39. Useful Troubleshooting Commands

Check the data directory:

```sql
SHOW data_directory;
```

Check the HBA file location:

```sql
SHOW hba_file;
```

Check the main configuration file:

```sql
SHOW config_file;
```

Check listening addresses:

```sql
SHOW listen_addresses;
```

Check port:

```sql
SHOW port;
```

Check password encryption:

```sql
SHOW password_encryption;
```

Check logging collector:

```sql
SHOW logging_collector;
```

Check log directory:

```sql
SHOW log_directory;
```

Check slow-query threshold:

```sql
SHOW log_min_duration_statement;
```

Check HBA parsing:

```sql
SELECT *
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

Reload configuration:

```sql
SELECT pg_reload_conf();
```

---

# 40. Connection Troubleshooting Flow

When a remote PostgreSQL connection fails, troubleshoot in this order:

```text
1. Is the PostgreSQL server running?
              |
              v
2. Is TCP port 5432 reachable?
              |
              v
3. Does listen_addresses include the required interface?
              |
              v
4. Does pg_hba.conf contain a matching rule?
              |
              v
5. Is the authentication method correct?
              |
              v
6. Is the user's password correct?
              |
              v
7. Does the role have CONNECT permission?
              |
              v
8. Does the role have required schema/table privileges?
```

This makes troubleshooting much faster than changing multiple settings at the same time.

---

# 41. Recommended Production Example

Assume:

```text
Database        : newdb
Application role: app_users
Application CIDR: 10.20.30.0/24
Authentication  : SCRAM-SHA-256
Encryption      : SSL/TLS
```

A stronger HBA configuration would be:

```text
hostssl   newdb   +app_users   10.20.30.0/24   scram-sha-256
```

Instead of:

```text
host   all   all   0.0.0.0/0   trust
```

The first configuration follows the principle of least privilege because it restricts:

```text
Database
   +
Role
   +
Network
   +
Authentication method
   +
Encrypted transport
```

---

# Key Points to Remember

1. `pg_hba.conf` controls client authentication rules.

2. PostgreSQL evaluates HBA rules from top to bottom, and the **first matching rule wins**.

3. `local` means Unix-domain socket connections.

4. `host` means TCP/IP connections; it does not simply mean "remote."

5. `hostssl` requires TCP/IP over SSL/TLS.

6. `peer` uses operating-system identity for local authentication.

7. `trust` allows matching connections without authentication and should be used very carefully.

8. `scram-sha-256` is the preferred password authentication method.

9. MD5 password authentication is deprecated in PostgreSQL 18.

10. `OPENSSL` is not a `pg_hba.conf` authentication method.

11. `/32` represents one exact IPv4 address.

12. `0.0.0.0/0` matches every IPv4 address and should normally be avoided for unrestricted production database access.

13. `listen_addresses` determines which network interfaces PostgreSQL listens on.

14. Changing `listen_addresses` requires a PostgreSQL restart.

15. Most `pg_hba.conf` changes require only a reload.

16. Use `pg_hba_file_rules` to validate and troubleshoot HBA rules.

17. `+role_name` in the HBA user field can match members of a PostgreSQL role.

18. HBA authentication does not automatically grant database-object permissions.

19. `log_min_duration_statement = 0` logs every statement's duration; use `10s` or `10000` for a 10-second threshold.

20. PostgreSQL log rotation creates new files but should be combined with a proper retention and housekeeping policy.

21. `logging_collector` requires a restart when its value is changed.

22. Avoid `log_statement = 'all'` in busy production systems unless there is a clear requirement because it can generate substantial log volume and may expose sensitive SQL data.

The main technical corrections above come directly from the PostgreSQL 18 documentation. 

In particular, PostgreSQL confirms that HBA rules are processed sequentially with no fall-through, `hostssl` is the SSL-specific record type, and `pg_hba_file_rules` can be used to check HBA configuration problems. ([PostgreSQL][1]) PostgreSQL also confirms that `listen_addresses` defaults to `localhost`, `*` listens on all available interfaces, and changing it requires a server start/restart. ([PostgreSQL][2])

For password authentication, PostgreSQL 18 recommends SCRAM-SHA-256 over MD5, and MD5 password authentication is now deprecated.([PostgreSQL][3]) 

For logging, I corrected the important `log_min_duration_statement` point: `0` logs every completed statement's duration, while `10000` means 10 seconds when no unit is specified. 

I also separated log rotation from log retention because rotation itself does not provide a general 30-day cleanup policy. ([PostgreSQL][4])

Official references: 
[PostgreSQL 18 — pg_hba.conf Documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html) 
[PostgreSQL 18 — Password Authentication](https://www.postgresql.org/docs/current/auth-password.html) 
[PostgreSQL 18 — Connections and Authentication](https://www.postgresql.org/docs/current/runtime-config-connection.html)
[PostgreSQL 18 — Logging Configuration](https://www.postgresql.org/docs/current/runtime-config-logging.html)

[1]: https://www.postgresql.org/docs/current/auth-pg-hba-conf.html "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[2]: https://www.postgresql.org/docs/current/runtime-config-connection.html "PostgreSQL: Documentation: 18: 19.3. Connections and Authentication"
[3]: https://www.postgresql.org/docs/current/auth-password.html "PostgreSQL: Documentation: 18: 20.5. Password Authentication"
[4]: https://www.postgresql.org/docs/current/runtime-config-logging.html "PostgreSQL: Documentation: 18: 19.8. Error Reporting and Logging"
