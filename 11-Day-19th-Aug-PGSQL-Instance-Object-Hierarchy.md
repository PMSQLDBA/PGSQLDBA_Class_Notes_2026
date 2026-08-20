Reviewed **11th Class – 19th August** notes. 

The class covers PostgreSQL instance/object hierarchy, connectivity, databases and schemas, roles, tables/views/materialized views, sequences, connection limits, and password handling. 

Checked the points against the **official PostgreSQL 18 documentation**. 

A few statements in the original notes need important technical corrections—especially around `template0/template1`, passwordless login, role attributes, materialized views, and psql commands. ([PostgreSQL][1])

# PostgreSQL – 11th Class – 19th August 2026

## Object Flow, Database Connectivity, Schemas, Objects, Roles and Connections

---

# 1. PostgreSQL Object Hierarchy

A PostgreSQL installation can contain one or more PostgreSQL **database clusters/instances**.

A simplified hierarchy is:

```text
PostgreSQL Instance / Database Cluster
│
├── Server Process
├── Data Directory (PGDATA)
├── Port
│
├── Database 1
│   ├── Schema 1
│   │   ├── Tables
│   │   ├── Views
│   │   ├── Materialized Views
│   │   ├── Sequences
│   │   ├── Functions
│   │   ├── Procedures
│   │   └── Other Objects
│   │
│   └── Schema 2
│       └── Objects
│
├── Database 2
│   └── Schemas
│       └── Objects
│
└── Database N
```

### Important Points

* One PostgreSQL instance/database cluster uses one primary **data directory**.
* A PostgreSQL server normally listens on a configured **port**.
* The default PostgreSQL port is:

```text
5432
```

* A single PostgreSQL instance can contain multiple databases.
* A database can contain multiple schemas.
* A schema can contain many database objects.
* Roles are defined at the PostgreSQL cluster level rather than separately inside each database.

---

# 2. Multiple PostgreSQL Instances on the Same Linux Server

It is possible to run multiple PostgreSQL instances on the same Linux server.

For example:

```text
Linux Server
│
├── PostgreSQL Instance 1
│   ├── Port: 5432
│   └── Data Directory: /pgdata/instance1
│
├── PostgreSQL Instance 2
│   ├── Port: 5433
│   └── Data Directory: /pgdata/instance2
│
└── PostgreSQL Instance 3
    ├── Port: 5434
    └── Data Directory: /pgdata/instance3
```

Each running instance must use a separate data directory.

Instances running simultaneously on the same network address also need different listening ports.

The PostgreSQL software binaries can be shared when the instances use the same PostgreSQL installation/version.

---

# 3. Cross-Database and Cross-Schema Access

PostgreSQL does not directly support normal SQL joins between tables located in separate databases through a single PostgreSQL connection.

For example, this is not a normal PostgreSQL cross-database query:

```text
database1.schema1.table1
JOIN
database2.schema1.table2
```

A PostgreSQL connection is established to one database at a time.

However, PostgreSQL supports joins between schemas within the **same database**.

Example:

```sql
SELECT *
FROM schema1.emp e
JOIN schema2.dept d
  ON e.deptno = d.deptno;
```

For cross-database access, additional mechanisms such as Foreign Data Wrappers can be used when required.

---

# 4. PostgreSQL Database Connectivity

From the Linux `postgres` operating-system account:

```bash
[postgres@mypg ~]$ psql
psql (18.4)
Type "help" for help.

postgres=#
```

No database name or PostgreSQL user name was explicitly provided in this command.

That does **not** mean PostgreSQL always allows login without a password.

When connection parameters are omitted, `psql` uses defaults. In particular:

* The default PostgreSQL user name is normally the operating-system user name.
* After determining the database user, that same name is normally used as the default database name.
* Authentication is then controlled by PostgreSQL's authentication configuration.

Therefore, when the Linux user is:

```text
postgres
```

running:

```bash
psql
```

commonly attempts to connect as:

```text
Database User : postgres
Database      : postgres
```

Whether a password is required depends on the matching authentication rule, usually configured through `pg_hba.conf`.

---

# 5. Check Current Connection

Use:

```text
\c
```

Example:

```text
postgres=# \c
You are now connected to database "postgres" as user "postgres".
```

Another useful command is:

```text
\conninfo
```

Example:

```text
postgres=# \conninfo
```

It displays information about the current connection, such as:

```text
Database
Client User
Socket Directory
Server Port
Protocol Version
Password Used
SSL Connection
Backend PID
Superuser
```

`\conninfo` is especially useful when troubleshooting connectivity.

---

# 6. Why Can Local Login Work Without a Password?

The original session showed:

```text
Password Used : false
```

This does not mean that only the `postgres` database user can ever connect without a password.

Passwordless access depends on the authentication method configured for the connection.

PostgreSQL controls client authentication mainly through:

```text
pg_hba.conf
```

For local Unix-domain socket connections, **peer authentication** is commonly used.

Peer authentication obtains the operating-system user name and checks it against the requested PostgreSQL database user.

For example:

```text
Linux User      : postgres
PostgreSQL User : postgres
```

can be allowed without entering a PostgreSQL password when an appropriate peer-authentication rule exists.

Other authentication methods include:

```text
peer
trust
scram-sha-256
md5
password
GSSAPI
LDAP
PAM
certificate
OAuth
```

For production systems, the authentication method should be selected according to the security requirements.

---

# 7. List Databases

Use:

```text
\l
```

or:

```text
\list
```

Example:

```text
postgres=# \l
```

A newly initialized PostgreSQL cluster normally contains:

```text
postgres
template0
template1
```

---

# 8. Standard PostgreSQL Databases

## postgres

The `postgres` database is created during cluster initialization and is intended as a convenient default database for users, utilities, and applications to connect to.

---

## template1

`template1` is the default template used by `CREATE DATABASE`.

When we execute:

```sql
CREATE DATABASE ramdb;
```

PostgreSQL normally creates the new database by copying:

```text
template1
```

Therefore, objects intentionally installed in `template1` can be inherited by databases created from it later.

Example:

```text
template1
   |
   | CREATE DATABASE ramdb
   v
ramdb
```

---

## template0

`template0` represents a pristine copy of the original database environment.

Normally:

```text
template0
```

does not accept connections.

It should generally **not be modified**.

It is useful when a clean database is required.

Example:

```sql
CREATE DATABASE clean_db TEMPLATE template0;
```

### Important Correction

Do **not** install common packages or custom objects into `template0`.

If site-specific objects need to appear automatically in newly created databases, `template1` is the appropriate standard template to customize.

`template0` should remain pristine.

---

# 9. List Schemas

Use:

```text
\dn
```

Example:

```text
postgres=# \dn
```

The output may show:

```text
Name   | Owner
-------+------------------
public | pg_database_owner
```

The `public` schema normally exists in a newly created database.

A schema is a namespace used to organize database objects.

For example:

```text
ramdb
│
├── public
│   └── emp
│
└── rajschema
    ├── emp
    ├── emp_v
    ├── emp_mv
    └── ram_s
```

---

# 10. Create a Database

Example:

```sql
CREATE DATABASE ramdb;
```

Output:

```text
CREATE DATABASE
```

Check it using:

```text
\l
```

Connect to it:

```text
\c ramdb
```

Example:

```text
You are now connected to database "ramdb" as user "postgres".
```

---

# 11. Create a Schema

Create a schema:

```sql
CREATE SCHEMA rajschema;
```

Check schemas:

```text
\dn
```

Example result:

```text
public
rajschema
```

A schema helps separate and organize objects inside the same database.

---

# 12. Create a Table in a Specific Schema

Syntax:

```sql
CREATE TABLE schema_name.table_name
(
    column definitions
);
```

Example:

```sql
CREATE TABLE rajschema.emp
(
    eno   integer,
    ename varchar
);
```

Insert records:

```sql
INSERT INTO rajschema.emp VALUES (1, 'ram');
INSERT INTO rajschema.emp VALUES (1, 'ram');
INSERT INTO rajschema.emp VALUES (1, 'ram');
```

Query:

```sql
SELECT *
FROM rajschema.emp;
```

Result:

```text
eno | ename
----+------
1   | ram
1   | ram
1   | ram
```

---

# 13. Create a Table Without Specifying a Schema

Example:

```sql
CREATE TABLE emp
(
    eno   integer,
    ename varchar
);
```

If no schema is explicitly specified, PostgreSQL determines the target schema using the current:

```text
search_path
```

In a normal default configuration, the object will commonly be created in the `public` schema.

To check the current search path:

```sql
SHOW search_path;
```

This is an important concept because unqualified object names are resolved using `search_path`.

---

# 14. Update Data

Example:

```sql
UPDATE rajschema.emp
SET eno = 100
WHERE eno = 1;
```

Output:

```text
UPDATE 3
```

Verify:

```sql
SELECT *
FROM rajschema.emp;
```

Result:

```text
eno | ename
----+------
100 | ram
100 | ram
100 | ram
```

---

# 15. PostgreSQL Views

Create a view:

```sql
CREATE VIEW rajschema.emp_v AS
SELECT *
FROM rajschema.emp;
```

A normal view stores the **query definition**, not a separate physical copy of the query result.

Query the view:

```sql
SELECT *
FROM rajschema.emp_v;
```

The view executes its underlying query and reflects the current data visible from the base objects.

### Simple Understanding

```text
TABLE
  |
  | queried by
  v
VIEW
```

A view can be thought of as a saved query or logical representation of data.

It is not necessarily a 1:1 representation of a single table. A view can contain:

* joins
* filters
* expressions
* aggregations
* multiple tables

Example:

```sql
CREATE VIEW rajschema.active_emp AS
SELECT eno, ename
FROM rajschema.emp
WHERE eno > 50;
```

---

# 16. Materialized Views

Create a materialized view:

```sql
CREATE MATERIALIZED VIEW rajschema.emp_mv AS
SELECT *
FROM rajschema.emp;
```

A materialized view stores the result of its defining query.

Query it:

```sql
SELECT *
FROM rajschema.emp_mv;
```

Conceptually:

```text
Normal View
    -> stores query definition
    -> reads current underlying data when queried

Materialized View
    -> stores query definition
    -> also persists the query result
    -> may contain stale data
    -> must be refreshed when updated results are required
```

---

# 17. Table vs View vs Materialized View

| Object            | Stores Data/Result?             | Main Purpose                         |
| ----------------- | ------------------------------- | ------------------------------------ |
| Table             | Yes                             | Primary persistent data storage      |
| View              | No separate stored query result | Logical/saved query representation   |
| Materialized View | Yes, stores query result        | Faster access to precomputed results |

Materialized views are commonly useful for:

* reporting
* dashboards
* expensive joins
* aggregation queries
* data that does not need to be real-time

### Important Correction

A materialized view should **not normally be treated as a backup or safety copy of a table**.

Backups and disaster recovery should use proper PostgreSQL backup/restore and replication strategies.

---

# 18. Materialized View Refresh Behavior

Suppose the table initially contains:

```text
100 | ram
100 | ram
100 | ram
```

The normal view and materialized view initially show the same result.

Now delete the source table records:

```sql
DELETE FROM rajschema.emp;
```

The table becomes empty:

```sql
SELECT *
FROM rajschema.emp;
```

Result:

```text
0 rows
```

The normal view also reflects the current underlying table:

```sql
SELECT *
FROM rajschema.emp_v;
```

Result:

```text
0 rows
```

But the materialized view may still contain its previously stored result:

```sql
SELECT *
FROM rajschema.emp_mv;
```

Result:

```text
100 | ram
100 | ram
100 | ram
```

Refresh it:

```sql
REFRESH MATERIALIZED VIEW rajschema.emp_mv;
```

After refreshing:

```sql
SELECT *
FROM rajschema.emp_mv;
```

Result:

```text
0 rows
```

By default, PostgreSQL does not automatically refresh a materialized view whenever its source table changes.

---

# 19. Sequences

A PostgreSQL sequence is a special database object commonly used to generate numeric values, such as identifiers.

Create a sequence:

```sql
CREATE SEQUENCE rajschema.ram_s
START WITH 1
INCREMENT BY 1
MAXVALUE 100;
```

Important sequence functions include:

```text
nextval()
currval()
lastval()
setval()
```

---

# 20. `nextval()`

Execute:

```sql
SELECT nextval('rajschema.ram_s');
```

Result:

```text
1
```

`nextval()` advances the sequence and returns the new value.

PostgreSQL performs this operation atomically, so concurrent sessions can safely obtain sequence values.

---

# 21. `currval()`

If we execute:

```sql
SELECT currval('rajschema.ram_s');
```

before calling `nextval()` in the current session, PostgreSQL returns an error:

```text
ERROR: currval of sequence "ram_s" is not yet defined in this session
```

First execute:

```sql
SELECT nextval('rajschema.ram_s');
```

Then:

```sql
SELECT currval('rajschema.ram_s');
```

`currval()` returns the most recent value obtained by `nextval()` for that sequence **in the current session**.

### Important Sequence Point

Sequences are not designed to guarantee gapless numbers.

For example, sequence values can be skipped because of:

* rolled-back transactions
* failed inserts
* `ON CONFLICT` processing
* crashes in certain situations

Therefore, do not assume:

```text
1, 2, 3, 4, 5...
```

will always be perfectly gapless.

---

# 22. List Tables

To list tables in a specific schema:

```text
\dt rajschema.*
```

Example:

```text
Schema    | Name | Type
----------+------+------
rajschema | emp  | table
```

To list visible tables:

```text
\dt
```

---

# 23. List Views

Use:

```text
\dv rajschema.*
```

Example:

```text
rajschema | emp_v | view
```

---

# 24. List Materialized Views

Use:

```text
\dm rajschema.*
```

Example:

```text
rajschema | emp_mv | materialized view
```

---

# 25. List Sequences

Use:

```text
\ds rajschema.*
```

Example:

```text
rajschema | ram_s | sequence
```

---

# 26. List Functions

Use:

```text
\df rajschema.*
```

`\df` lists functions and related routine information.

Useful variants are available to filter different routine types.

---

# 27. Important Correction – `\dT` Does Not List Triggers

The original notes mention:

```text
List of Triggers:
\dT
```

This is incorrect.

In `psql`:

```text
\dT
```

lists **data types**, not triggers.

To inspect a table and its associated information, including triggers when present, use:

```text
\d schema_name.table_name
```

or for more details:

```text
\d+ schema_name.table_name
```

For example:

```text
\d+ rajschema.emp
```

For a database-wide trigger query, PostgreSQL system catalogs or `information_schema.triggers` can also be queried.

---

# 28. List Tablespaces

Use:

```text
\db
```

Example:

```text
Name       | Owner
-----------+---------
pg_default | postgres
pg_global  | postgres
```

The standard tablespaces include:

### pg_default

Used as the default tablespace for ordinary database objects unless another tablespace is specified.

### pg_global

Used for shared system catalog objects belonging to the entire PostgreSQL cluster.

---

# 29. PostgreSQL Roles and Users

PostgreSQL uses the concept of **roles**.

A role can represent:

* a login user
* a group
* an object owner
* a privilege holder

`CREATE USER` is essentially a convenient form of `CREATE ROLE` that includes the `LOGIN` attribute.

Example:

```sql
CREATE USER raj
WITH PASSWORD 'ram123';
```

Check roles:

```text
\du
```

Example:

```text
Role name | Attributes
----------+-----------
postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
raj       |
```

---

# 30. PostgreSQL Role Attributes

Important role attributes include:

```text
SUPERUSER
CREATEDB
CREATEROLE
LOGIN
REPLICATION
BYPASSRLS
CONNECTION LIMIT
VALID UNTIL
```

These are role attributes/capabilities rather than ordinary object privileges such as:

```sql
GRANT SELECT ON table_name TO user_name;
```

Role attributes can be specified with `CREATE ROLE`/`CREATE USER` and changed later with `ALTER ROLE`, subject to PostgreSQL permission rules.

---

# 31. SUPERUSER

Example:

```sql
ALTER ROLE raj SUPERUSER;
```

A superuser bypasses essentially all access restrictions inside PostgreSQL.

This is an extremely powerful privilege and should be assigned only when genuinely required.

Normal application users should not be superusers.

---

# 32. CREATEDB

Allows a role to create databases.

Example:

```sql
ALTER ROLE raj CREATEDB;
```

Verify:

```text
\du raj
```

---

# 33. CREATEROLE

Allows a role to perform role-management operations, subject to PostgreSQL's permission rules.

Example:

```sql
ALTER ROLE raj CREATEROLE;
```

`CREATEROLE` does **not** make the role equivalent to a superuser.

There are restrictions around highly privileged roles and attributes.

---

# 34. REPLICATION

A role with the `REPLICATION` attribute can perform replication-related operations such as connecting in replication mode and managing replication slots, subject to the required permissions and configuration.

Example:

```sql
CREATE ROLE repl_user
WITH LOGIN REPLICATION PASSWORD 'strong_password';
```

Replication privileges should be assigned carefully because they are highly privileged.

---

# 35. BYPASSRLS

`BYPASSRLS` means the role bypasses Row-Level Security policies.

It does **not** mean that masked data automatically becomes visible.

Correct concept:

```text
BYPASSRLS
    ->
bypasses Row-Level Security policies
```

Example:

```sql
ALTER ROLE raj BYPASSRLS;
```

This is a powerful security attribute.

---

# 36. Reset a User Password

Example:

```sql
ALTER USER raj
WITH PASSWORD 'sams123';
```

Equivalent role-oriented syntax:

```sql
ALTER ROLE raj
WITH PASSWORD 'sams123';
```

For interactive administration in `psql`, the `\password` command is useful because it avoids placing a clear-text password directly in the SQL command/history.

Example:

```text
\password raj
```

---

# 37. Password Validity

Example:

```sql
ALTER ROLE raj
VALID UNTIL '2026-09-24';
```

Check:

```text
\du raj
```

Important:

`VALID UNTIL` controls the expiration of the role's **password**.

It does not expire or disable the role itself.

It matters when password-based authentication is being used.

To remove password expiration:

```sql
ALTER ROLE raj
VALID UNTIL 'infinity';
```

---

# 38. Maximum Connections

Check:

```sql
SHOW max_connections;
```

Example:

```text
max_connections
---------------
100
```

The default is typically:

```text
100
```

but it is configurable.

`max_connections` determines the maximum number of concurrent connections allowed by the PostgreSQL server.

Increasing it also affects PostgreSQL resource requirements.

---

# 39. Connection Limit for a Specific User

Suppose the PostgreSQL server allows:

```text
max_connections = 100
```

We may not want one application role to consume nearly all available connection slots.

A per-role connection limit can be configured.

Example:

```sql
ALTER ROLE raj
CONNECTION LIMIT 10;
```

Verify:

```text
\du raj
```

Example:

```text
raj | Create DB
    | 10 connections
    | Password valid until ...
```

`CONNECTION LIMIT -1` means no role-specific limit.

A superuser is not restricted by this role connection limit in the same way as an ordinary role.

---

# 40. Disable New Connections to a Database

Example:

```sql
ALTER DATABASE ramdb
ALLOW_CONNECTIONS false;
```

A new connection attempt then fails:

```text
FATAL: database "ramdb" is not currently accepting connections
```

Enable connections again:

```sql
ALTER DATABASE ramdb
ALLOW_CONNECTIONS true;
```

Then:

```text
\c ramdb
```

can connect again, assuming all other authentication and permission requirements are satisfied.

### Important Point

Setting:

```sql
ALLOW_CONNECTIONS false
```

prevents **new connections**.

It does not automatically terminate sessions that are already connected.

---

# 41. Why `template0` Normally Rejects Connections

By default, `template0` is configured not to accept connections.

This helps protect it from accidental changes.

Although a superuser can technically change this setting, normal administration should preserve `template0` as the pristine system template.

Therefore, avoid treating this as a routine operation:

```sql
ALTER DATABASE template0
ALLOW_CONNECTIONS true;
```

`template0` should normally remain protected.

---

# 42. Reserved Connections

Check:

```sql
SHOW superuser_reserved_connections;
```

Example:

```text
superuser_reserved_connections
------------------------------
3
```

The default value is normally:

```text
3
```

These slots are reserved for PostgreSQL superusers.

This is useful when ordinary connection capacity has been exhausted and an administrator still needs emergency access.

PostgreSQL 18 also has the separate:

```text
reserved_connections
```

setting, which can reserve slots for roles that have privileges of the predefined:

```text
pg_use_reserved_connections
```

role.

The superuser-reserved slots remain the final emergency reserve.

---

# 43. Password Complexity

PostgreSQL does not use an Oracle-style **PROFILE** object for password policies.

PostgreSQL provides the supplied module:

```text
passwordcheck
```

The `passwordcheck` module checks passwords when they are set through commands such as:

```text
CREATE ROLE
ALTER ROLE
```

Weak passwords can be rejected.

In PostgreSQL 18, the module also provides:

```text
passwordcheck.min_password_length
```

with a default minimum length of 8 bytes.

However, PostgreSQL's official documentation warns that `passwordcheck` has limitations, particularly when clients submit already-encrypted password values.

For higher security requirements, organizations commonly integrate PostgreSQL with stronger external authentication systems and security policies.

---

# 44. Important `psql` Commands from This Class

| Command          | Purpose                                    |
| ---------------- | ------------------------------------------ |
| `\c database`    | Connect/switch to a database               |
| `\conninfo`      | Display current connection information     |
| `\l`             | List databases                             |
| `\dn`            | List schemas                               |
| `\du`            | List roles                                 |
| `\dt`            | List tables                                |
| `\dv`            | List views                                 |
| `\dm`            | List materialized views                    |
| `\ds`            | List sequences                             |
| `\df`            | List functions/routines                    |
| `\db`            | List tablespaces                           |
| `\d object`      | Describe an object                         |
| `\d+ object`     | Describe an object with additional details |
| `\dT`            | List data types                            |
| `\password role` | Change a role's password interactively     |
| `\q`             | Exit `psql`                                |

---

# 45. SQL Commands Covered

```sql
CREATE DATABASE ramdb;

CREATE SCHEMA rajschema;

CREATE TABLE rajschema.emp
(
    eno integer,
    ename varchar
);

INSERT INTO rajschema.emp
VALUES (1, 'ram');

UPDATE rajschema.emp
SET eno = 100
WHERE eno = 1;

DELETE FROM rajschema.emp;

CREATE VIEW rajschema.emp_v AS
SELECT *
FROM rajschema.emp;

CREATE MATERIALIZED VIEW rajschema.emp_mv AS
SELECT *
FROM rajschema.emp;

REFRESH MATERIALIZED VIEW rajschema.emp_mv;

CREATE SEQUENCE rajschema.ram_s
START WITH 1
INCREMENT BY 1
MAXVALUE 100;

SELECT nextval('rajschema.ram_s');

SELECT currval('rajschema.ram_s');

CREATE USER raj
WITH PASSWORD 'ram123';

ALTER ROLE raj
WITH PASSWORD 'sams123';

ALTER ROLE raj
VALID UNTIL '2026-09-24';

ALTER ROLE raj
CONNECTION LIMIT 10;

ALTER ROLE raj CREATEDB;

ALTER DATABASE ramdb
ALLOW_CONNECTIONS false;

ALTER DATABASE ramdb
ALLOW_CONNECTIONS true;

SHOW max_connections;

SHOW superuser_reserved_connections;
```

---

# 46. Important Interview Points

### Q1. Can one PostgreSQL instance contain multiple databases?

Yes. One PostgreSQL database cluster/server can manage multiple databases.

### Q2. Can one database contain multiple schemas?

Yes.

### Q3. Can we directly join tables from different schemas?

Yes, when the schemas belong to the same database and the user has the required privileges.

### Q4. Can PostgreSQL directly perform a normal cross-database join?

Not through normal schema-qualified SQL in a single database connection. Additional mechanisms such as Foreign Data Wrappers can be used when cross-database access is required.

### Q5. What is the default PostgreSQL port?

```text
5432
```

### Q6. What is `template1`?

It is the default source template copied by `CREATE DATABASE`.

### Q7. What is `template0`?

It is the pristine standard template and should normally remain unchanged.

### Q8. What is the difference between a view and a materialized view?

A normal view stores a query definition and retrieves current underlying data when queried. A materialized view persists the result of its defining query and requires refresh to obtain updated results.

### Q9. Does a materialized view automatically refresh when the base table changes?

No, not by default.

Use:

```sql
REFRESH MATERIALIZED VIEW name;
```

### Q10. Can `currval()` be called before `nextval()`?

Not successfully for that sequence in a new session. `nextval()` must first establish the sequence's current value for that session.

### Q11. Does a PostgreSQL sequence guarantee gapless values?

No.

### Q12. What does `BYPASSRLS` mean?

It allows the role to bypass Row-Level Security policies.

### Q13. What does `VALID UNTIL` expire?

The role's password, not the role itself.

### Q14. What does `CONNECTION LIMIT` control?

The maximum number of concurrent connections allowed for a login role.

### Q15. What does `ALLOW_CONNECTIONS false` do?

It prevents new connections to the database. It does not automatically disconnect existing sessions.

### Q16. What is `superuser_reserved_connections`?

It reserves connection slots for superusers so administrators can still connect when normal connection capacity has been consumed.

---

# 47. Key Corrections from the Original Class Notes

1. **Passwordless login is not exclusive to the `postgres` user.**
   It depends on the authentication rules in `pg_hba.conf`, such as peer or trust authentication.

2. **Do not install packages/custom objects into `template0`.**
   `template0` should remain pristine. Customize `template1` when objects need to be inherited by newly created databases.

3. **A view is not necessarily a 1:1 copy of a table.**
   It is a stored query definition and can contain joins, filters, expressions and aggregations.

4. **A materialized view is not a database backup.**
   It stores a query result primarily for use cases such as faster querying and reporting.

5. **`BYPASSRLS` means bypass Row-Level Security.**
   It is not specifically a data-masking privilege.

6. **`\dT` lists data types, not triggers.**

7. **`VALID UNTIL` expires a password, not the PostgreSQL role itself.**

8. **`max_connections = 100` is typically the default, not a fixed PostgreSQL limit.**

9. **`superuser_reserved_connections = 3` is normally the default and is configurable.**

10. **`passwordcheck` is a supplied PostgreSQL module for basic password-strength checking, but it has documented security limitations.**

---

# 48. Final Architecture Summary

```text
                         PostgreSQL Server / Cluster
                                   |
                    +--------------+--------------+
                    |                             |
               Database A                    Database B
                    |                             |
            +-------+-------+               +-----+-----+
            |               |               |           |
         Schema 1         Schema 2        Schema 1    Schema 2
            |               |               |
       +----+----+      +----+----+          |
       |    |    |      |    |    |          |
     Table View  MV   Table Func Sequence   Objects
```

Remember the hierarchy as:

```text
PostgreSQL Cluster / Instance
        ↓
     Database
        ↓
      Schema
        ↓
      Objects
```

And remember these three important administration layers:

```text
postgresql.conf
    -> Server configuration

pg_hba.conf
    -> Client authentication rules

Roles / Privileges
    -> User capabilities and object access
```

The most important correction is the **template database section**. 

Original notes say to install common packages in `template0` so new databases inherit them.  

Official PostgreSQL documentation says the opposite: `template0` should remain unchanged/pristine; custom objects that should be copied into future databases belong in `template1`. ([PostgreSQL][1])

Also corrected the authentication statement around passwordless `postgres` login. 

Class output does show a local socket connection with no password,  but whether a password is required is determined by authentication configuration such as `peer`, `trust`, or password authentication in `pg_hba.conf`; it isn't a special universal rule for only the `postgres` user. ([PostgreSQL][2])

The other major fixes are: `BYPASSRLS` bypasses RLS policies; `VALID UNTIL` is password expiration rather than role expiration; materialized views persist query results but aren't backups; `currval()` is session-specific and requires an earlier `nextval()` for that sequence; and `\dT` lists **data types**, not triggers. ([PostgreSQL][3])

Official references used: 
[PostgreSQL 18 — psql Documentation](https://www.postgresql.org/docs/18/app-psql.html)
[PostgreSQL 18 — Template Databases](https://www.postgresql.org/docs/18/manage-ag-templatedbs.html)
[PostgreSQL 18 — Materialized Views](https://www.postgresql.org/docs/18/rules-materializedviews.html)
[PostgreSQL 18 — CREATE ROLE](https://www.postgresql.org/docs/current/sql-createrole.html)
[PostgreSQL 18 — Connections and Authentication](https://www.postgresql.org/docs/18/runtime-config-connection.html)
[PostgreSQL 18 — Sequence Functions](https://www.postgresql.org/docs/18/functions-sequence.html)

[1]: https://www.postgresql.org/docs/18/manage-ag-templatedbs.html "PostgreSQL: Documentation: 18: 22.3. Template Databases"
[2]: https://www.postgresql.org/docs/current/auth-pg-hba-conf.html "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[3]: https://www.postgresql.org/docs/current/sql-createrole.html "PostgreSQL: Documentation: 18: CREATE ROLE"
