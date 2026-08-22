**PostgreSQL Version Used in Examples:** PostgreSQL 18

**Class Date:** 22 August 2026

## User Access Management, Privileges, Roles, and Physical Storage

---

# PART 1 – USER ACCESS MANAGEMENT

## 1. PostgreSQL Access Control Overview

PostgreSQL security can be understood at multiple levels:

1. **Server / Client Authentication Level**

   * Controlled mainly through `pg_hba.conf`.
   * Determines **who can connect, from where, to which database, and using which authentication method**.

2. **Database Level**

   * Important privileges include:

     * `CONNECT`
     * `CREATE`
     * `TEMPORARY`

3. **Schema Level**

   * Important privileges:

     * `USAGE`
     * `CREATE`

4. **Object Level**

   * Examples:

     * `SELECT`
     * `INSERT`
     * `UPDATE`
     * `DELETE`
     * `TRUNCATE`
     * `REFERENCES`
     * `TRIGGER`
     * `MAINTAIN`
     * `EXECUTE`
     * Sequence privileges such as `USAGE`, `SELECT`, and `UPDATE`

---

# 2. Authentication vs Authorization

These two concepts should not be confused.

### Authentication

Authentication answers:

> **Can this user connect to PostgreSQL?**

Authentication is controlled mainly through `pg_hba.conf`.

Example:

```text
local   newdb   ram   scram-sha-256
```

Meaning:

* Connection type = local Unix-domain socket
* Database = `newdb`
* User/role = `ram`
* Authentication method = `scram-sha-256`

After modifying `pg_hba.conf`, reload PostgreSQL:

```bash
/usr/pgsql-18/bin/pg_ctl \
    -D /var/lib/pgsql/18/data reload
```

A reload is normally enough for `pg_hba.conf` changes; a PostgreSQL restart is not normally required.

---

## 3. Authorization

Authorization answers:

> **After connecting, what is the user allowed to do?**

Authorization is controlled using PostgreSQL privileges and roles.

For example:

```sql
GRANT USAGE ON SCHEMA test1 TO ram;
GRANT SELECT ON test1.emp TO ram;
```

The first command allows `ram` to access objects through the `test1` schema.

The second allows `ram` to read the `emp` table.

Both permissions can be necessary.

---

# 4. Creating a Database, Schema, and Table

Connect as a privileged user:

```sql
CREATE DATABASE newdb;
```

Connect to the database:

```sql
\c newdb
```

Create a schema:

```sql
CREATE SCHEMA test1;
```

Create a table:

```sql
CREATE TABLE test1.emp
(
    eno   integer,
    ename varchar
);
```

Check the table:

```sql
\dt test1.emp
```

Check the schema:

```sql
\dn test1
```

For detailed schema privileges:

```sql
\dn+ test1
```

---

# 5. Schema Privileges

PostgreSQL has two important schema privileges.

## USAGE

```sql
GRANT USAGE ON SCHEMA test1 TO ram;
```

`USAGE` allows the role to access or look up objects inside the schema.

However:

> `USAGE` does **not** automatically grant permission on the tables inside that schema.

For example, after granting only `USAGE`:

```sql
SELECT * FROM test1.emp;
```

may still return:

```text
ERROR: permission denied for table emp
```

The user therefore needs both:

```text
Schema permission
        +
Object permission
```

Example:

```sql
GRANT USAGE ON SCHEMA test1 TO ram;
GRANT SELECT ON test1.emp TO ram;
```

---

## CREATE on a Schema

To allow a user to create objects inside a schema:

```sql
GRANT CREATE ON SCHEMA test1 TO ram;
```

Now `ram` can do:

```sql
CREATE TABLE test1.employee
(
    id integer
);
```

### Important

`CREATE` on a schema means:

> The user can create new objects inside that schema.

It does **not** mean that the user automatically owns or can alter existing objects created by other users.

---

# 6. Table Privileges

Common table privileges include:

```text
SELECT
INSERT
UPDATE
DELETE
TRUNCATE
REFERENCES
TRIGGER
MAINTAIN
```

Example:

```sql
GRANT SELECT ON test1.emp TO ram;
```

Multiple privileges can be granted together:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON test1.emp
TO ram;
```

Then `ram` can perform:

```sql
SELECT * FROM test1.emp;

INSERT INTO test1.emp
VALUES (1, 'Raj');

UPDATE test1.emp
SET eno = 10
WHERE eno = 1;

DELETE FROM test1.emp
WHERE eno = 10;
```

---

# 7. Granting Permissions on All Existing Tables

Assume schema `test1` contains many tables.

Instead of granting permissions table by table:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA test1
TO ram;
```

This grants permissions on **all tables that currently exist** in the schema.

### Very Important

It does not automatically cover tables created later.

For example:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA test1
TO ram;
```

and later:

```sql
CREATE TABLE test1.future_table
(
    id integer
);
```

`ram` does not automatically receive `SELECT` on `future_table`.

For future objects, use **default privileges**.

---

# 8. ALTER DEFAULT PRIVILEGES

Example:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA test1
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES
TO ram;
```

Now tables created in the future by the role whose default privileges were modified can automatically grant these permissions to `ram`.

Example:

```sql
CREATE TABLE test1.mytest
(
    eno integer
);
```

Then:

```sql
SELECT * FROM test1.mytest;
```

can work for `ram`, provided the applicable default privileges were defined correctly.

Check default privileges using:

```sql
\ddp
```

---

## Important Correction About Default Privileges

`ALTER DEFAULT PRIVILEGES` is associated with the **role that will create the future objects**.

It is not simply a global rule for every future table created by every user.

For example, if `postgres` will create future tables:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
IN SCHEMA test1
GRANT SELECT ON TABLES TO ram;
```

This applies to tables subsequently created by `postgres`.

If another role, such as `app_owner`, creates tables, its default privileges must be configured separately:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE app_owner
IN SCHEMA test1
GRANT SELECT ON TABLES TO ram;
```

This is an important PostgreSQL administration concept.

---

# 9. Object Ownership

When a PostgreSQL object is created, it has an owner.

Normally, the role that executes the `CREATE` statement becomes the owner.

Example:

```sql
CREATE TABLE test1.emp
(
    eno integer
);
```

If `postgres` creates the table, `postgres` normally owns it.

---

## ALTER and DROP

Privileges such as:

```text
SELECT
INSERT
UPDATE
DELETE
```

do not give a user ownership of the table.

For example, `ram` may have:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON test1.emp
TO ram;
```

but this does not automatically allow:

```sql
ALTER TABLE test1.emp ADD COLUMN salary integer;
```

or:

```sql
DROP TABLE test1.emp;
```

PostgreSQL treats the right to alter or destroy an object as part of **ownership**, rather than as an ordinary grantable table privilege.

Normally, the user must be the object's owner or have suitable elevated authority.

---

# 10. Database CREATE Privilege

To create a schema inside a database, the user needs `CREATE` on the database.

Without it:

```sql
CREATE SCHEMA newschema;
```

may return:

```text
ERROR: permission denied for database newdb
```

Grant permission:

```sql
GRANT CREATE ON DATABASE newdb TO ram;
```

Afterward:

```sql
CREATE SCHEMA newschema;
```

can succeed.

### Meaning

`CREATE` on a **database** allows operations such as creating schemas in that database.

`CREATE` on a **schema** allows objects to be created inside that particular schema.

These are different privileges.

---

# 11. Database CONNECT Privilege

PostgreSQL also has:

```sql
GRANT CONNECT ON DATABASE newdb TO ram;
```

`CONNECT` controls whether a role is permitted to connect to the database.

Remember that a successful connection can depend on **both**:

```text
pg_hba.conf authentication rules
             +
database CONNECT privilege
```

---

# 12. PostgreSQL Roles and Users

PostgreSQL internally uses the concept of **roles**.

A "user" is essentially a role that has the `LOGIN` attribute.

Example:

```sql
CREATE ROLE ram LOGIN PASSWORD 'StrongPassword';
```

The commonly used command:

```sql
CREATE USER ram PASSWORD 'StrongPassword';
```

is effectively a convenient form of creating a login-capable role.

---

# 13. Group Roles

Older terminology often uses the word **group**.

In modern PostgreSQL administration, it is clearer to think in terms of:

> **Login roles + group roles + role membership**

Create a group-style role:

```sql
CREATE ROLE test_g NOLOGIN;
```

Grant privileges to it:

```sql
GRANT USAGE ON SCHEMA test1 TO test_g;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA test1
TO test_g;
```

Set default privileges if needed:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA test1
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES
TO test_g;
```

Add a user to the role:

```sql
GRANT test_g TO veeru;
```

This is preferable to thinking of:

```sql
ALTER GROUP test_g ADD USER veeru;
```

because modern PostgreSQL role management uses `GRANT role TO role`.

---

# 14. Role-Based Access Control – RBAC

Suppose many users require the same permissions.

Do not repeatedly grant every privilege directly to every user.

Instead:

```text
              test_g
                |
       --------------------
       |        |         |
     ram      veeru      user3
```

Grant permissions once to `test_g`.

Then:

```sql
GRANT test_g TO ram;
GRANT test_g TO veeru;
GRANT test_g TO user3;
```

This makes privilege administration easier.

---

## Recommended Pattern

Create separate roles according to responsibilities.

Example:

```text
app_readonly
app_readwrite
app_admin
```

Read-only role:

```sql
CREATE ROLE app_readonly NOLOGIN;

GRANT USAGE ON SCHEMA test1
TO app_readonly;

GRANT SELECT
ON ALL TABLES IN SCHEMA test1
TO app_readonly;
```

Read/write role:

```sql
CREATE ROLE app_readwrite NOLOGIN;

GRANT USAGE ON SCHEMA test1
TO app_readwrite;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA test1
TO app_readwrite;
```

Then:

```sql
GRANT app_readonly TO report_user;
GRANT app_readwrite TO application_user;
```

This is much easier to manage than large numbers of direct user grants.

---

# 15. Direct Grants vs Role-Based Grants

## Direct Access

```sql
GRANT SELECT ON test1.emp TO ram;
GRANT SELECT ON test1.emp TO veeru;
GRANT SELECT ON test1.emp TO ravi;
```

This becomes difficult to manage when many users exist.

## Role-Based Access

```sql
CREATE ROLE read_group NOLOGIN;

GRANT SELECT ON test1.emp
TO read_group;

GRANT read_group TO ram;
GRANT read_group TO veeru;
GRANT read_group TO ravi;
```

Preferred concept:

```text
Users → Roles → Privileges → Objects
```

---

# 16. WITH GRANT OPTION

A privilege can optionally be granted with permission to pass it to another role.

Example:

```sql
GRANT SELECT
ON test1.emp
TO ram
WITH GRANT OPTION;
```

Now `ram` can grant that `SELECT` privilege to another role.

This should be used carefully.

---

# 17. REVOKE

Permissions can be removed with `REVOKE`.

Example:

```sql
REVOKE SELECT
ON test1.emp
FROM ram;
```

Remove all explicitly granted privileges:

```sql
REVOKE ALL
ON test1.emp
FROM ram;
```

---

# 18. PUBLIC Role

`PUBLIC` is a special PostgreSQL concept representing **all roles**.

Example:

```sql
GRANT SELECT
ON test1.emp
TO PUBLIC;
```

This makes the table readable by all roles that can otherwise reach the object.

For security-sensitive environments, always review privileges inherited through `PUBLIC`.

---

# 19. Checking Privileges

## Schema Privileges

```sql
\dn+ test1
```

## Table Privileges

```sql
\dp test1.emp
```

or:

```sql
\z test1.emp
```

## Default Privileges

```sql
\ddp
```

## Table Privileges Through Information Schema

```sql
SELECT *
FROM information_schema.table_privileges
WHERE grantee = 'ram';
```

---

# 20. PostgreSQL ACL Abbreviations

When checking privileges, PostgreSQL may display letters such as:

```text
r = SELECT
a = INSERT
w = UPDATE
d = DELETE
D = TRUNCATE
x = REFERENCES
t = TRIGGER
C = CREATE
c = CONNECT
T = TEMPORARY
X = EXECUTE
U = USAGE
m = MAINTAIN
```

For example:

```text
ram=arwd/postgres
```

means approximately:

```text
a → INSERT
r → SELECT
w → UPDATE
d → DELETE
```

granted to `ram`, with `postgres` shown as the grantor.

---

# 21. PostgreSQL Privilege Flow

For a table in a non-public schema, think of access like this:

```text
Can client authenticate?
        |
        v
     pg_hba.conf
        |
        v
Can role connect to database?
        |
        v
 DATABASE CONNECT
        |
        v
Can role reach schema?
        |
        v
  SCHEMA USAGE
        |
        v
Can role perform operation?
        |
        v
TABLE / OBJECT PRIVILEGE
```

Example for `SELECT`:

```text
pg_hba.conf permits connection
        +
CONNECT ON DATABASE newdb
        +
USAGE ON SCHEMA test1
        +
SELECT ON test1.emp
        =
SELECT can succeed
```

subject to other PostgreSQL security mechanisms such as row-level security where applicable.

---

# PART 2 – PHYSICAL STORAGE IN POSTGRESQL

# 22. PostgreSQL Data Directory – PGDATA

A PostgreSQL database server instance manages a **database cluster**.

The main data directory is commonly called:

```text
PGDATA
```

Example installation path:

```text
/var/lib/pgsql/18/data
```

The exact location depends on how PostgreSQL was installed and configured.

Important files/directories commonly associated with the cluster include:

```text
base/
global/
pg_wal/
pg_tblspc/
pg_xact/
PG_VERSION
postgresql.conf
pg_hba.conf
pg_ident.conf
```

Configuration files can also be placed outside `PGDATA`, depending on the installation.

---

# 23. The base Directory

For databases using the default tablespace, database relation files are stored under:

```text
$PGDATA/base
```

Example:

```bash
cd /var/lib/pgsql/18/data/base
ls -ltr
```

You may see directories such as:

```text
1
4
5
16388
16912
```

These directory numbers correspond to database OIDs for databases stored in the default tablespace.

Check database OIDs using:

```sql
SELECT oid, datname
FROM pg_database;
```

Example:

```text
oid    datname
-----  ---------
5      postgres
1      template1
4      template0
16388  mydb
16912  newdb
```

Therefore:

```text
base/16912
```

represents the default-tablespace storage directory for database `newdb` in this example.

---

# 24. Database Objects and Storage

It is better not to divide every PostgreSQL object simply into "physical" and "logical" using the original list.

A more accurate approach is:

## Objects with relation storage

Examples include:

* ordinary tables
* indexes
* materialized views
* sequences
* TOAST tables
* some system catalog relations

### Important Correction

A sequence should **not** be classified as an object that occupies no storage.

Sequences are relations and have persistent state.

---

## Objects Without Their Own Ordinary Table Data File

Examples include concepts such as:

* ordinary views
* schemas
* functions
* procedures
* triggers

Their definitions and metadata are stored in PostgreSQL system catalogs, but they do not have an ordinary heap data file representing rows in the same way as a regular table.

An ordinary view stores a query definition, not a materialized copy of the rows.

A **materialized view**, however, stores its result physically and therefore occupies relation storage.

---

# 25. Relation Files

A table or index is represented using relation files.

The primary relation file is called the:

```text
main fork
```

Suppose the filenode is:

```text
16914
```

The main data file may be:

```text
16914
```

Additional relation forks can exist.

Common examples:

```text
16914
16914_fsm
16914_vm
```

Where:

```text
16914      → main fork
16914_fsm  → Free Space Map
16914_vm   → Visibility Map
```

Unlogged relations can additionally have:

```text
16914_init
```

which is the initialization fork.

---

# 26. Free Space Map – FSM

Suffix:

```text
_fsm
```

Example:

```text
16914_fsm
```

The Free Space Map records information about available space in relation pages.

PostgreSQL can use this information when deciding where a new tuple can be placed.

Think of it as:

```text
Which pages still have enough free space?
```

It does **not** store the actual table rows.

---

# 27. Visibility Map – VM

Suffix:

```text
_vm
```

Example:

```text
16914_vm
```

The Visibility Map stores information about heap pages whose tuples satisfy visibility-related conditions.

It is important for operations such as:

```text
VACUUM
Index-only scans
```

### Correction

The Visibility Map does **not** mean:

> "The disk location where the actual records are stored."

Actual row data belongs in the relation's **main fork**.

The VM contains visibility metadata.

---

# 28. Main Fork

Example:

```text
16914
```

This is the primary relation fork containing the actual relation data.

For a heap table, its table pages and tuples are stored through this main fork.

---

# 29. Does Every Object Have Exactly Three Files?

No.

The statement:

> "Every object has a minimum of three files"

is not a safe rule.

Different relation types use different forks.

For example:

```text
Main fork
FSM fork
VM fork
Init fork
```

may exist depending on the relation type and state.

Tables have visibility maps, while indexes do not use visibility maps in the same way.

Unlogged relations have an initialization fork.

Therefore, do not memorize:

```text
one object = exactly three files
```

Instead memorize:

```text
A PostgreSQL relation can have multiple forks.
```

---

# 30. Relation File Segment Size

PostgreSQL normally splits very large relation forks into file segments.

The default segment size is typically:

```text
1 GB
```

Suppose the filenode is:

```text
1249
```

The files can look like:

```text
1249
1249.1
1249.2
1249.3
...
```

### Correction to Original Notes

The sequence is:

```text
1249
1249.1
1249.2
1249.3
```

not:

```text
1249
1249.1
1249.3
```

If a main relation fork requires approximately 100 GB, it can therefore have around 100 default-sized segments, although exact disk usage and fork behavior should not be reduced to a simple table-size/file-count formula in all cases.

Also, 1 GB is the PostgreSQL **default** relation segment size. It can be changed when PostgreSQL itself is built.

---

# 31. OID vs Filenode

Do not assume:

```text
Table OID = physical file name
```

They may sometimes appear to match, but PostgreSQL does not guarantee that they always match.

Some operations can change a relation's filenode while its logical object identity remains.

Examples include certain cases involving:

```text
TRUNCATE
REINDEX
CLUSTER
ALTER TABLE
```

Use PostgreSQL functions to identify relation storage instead of guessing from the OID.

---

# 32. Finding the Physical File of a Table

Example:

```sql
SELECT pg_relation_filepath('test1.emp');
```

Possible result:

```text
base/16912/16914
```

Meaning:

```text
base
  |
  +-- 16912       database directory
       |
       +-- 16914  relation file
```

Another useful function is:

```sql
SELECT pg_relation_filenode('test1.emp');
```

This returns the current filenode associated with the relation.

---

# 33. Important Correction – pg_filenode.map

The original statement:

> "`pg_filenode.map` contains information about all data files"

is incorrect.

`pg_filenode.map` is not a general inventory file mapping every user table to every physical data file.

PostgreSQL primarily maintains relation metadata through its catalogs, including `pg_class`, and provides functions such as:

```sql
pg_relation_filepath()
pg_relation_filenode()
```

for resolving relation storage.

Do not treat `pg_filenode.map` as a list of all table data files.

---

# 34. TOAST Storage

A useful additional PostgreSQL storage concept is **TOAST**.

TOAST stands for:

```text
The Oversized-Attribute Storage Technique
```

PostgreSQL database pages have limited size.

When a row contains large values such as:

```text
large text
large JSON
large bytea
```

PostgreSQL may compress the value or move it to a separate TOAST relation.

A table can therefore have an associated TOAST table.

The catalog field:

```text
pg_class.reltoastrelid
```

can identify its TOAST relation where applicable.

---

# 35. Never Delete PostgreSQL Relation Files Manually

Suppose:

```sql
SELECT pg_relation_filepath('test1.emp');
```

returns:

```text
base/16912/16914
```

Do **not** run:

```bash
rm -rf 16914
```

on a running or stopped PostgreSQL cluster as a normal administration procedure.

Deleting PostgreSQL relation files directly can corrupt the database.

After manual deletion, PostgreSQL may report errors such as:

```text
could not open file ... No such file or directory
```

### Correct Administration Rule

Database objects must normally be managed using PostgreSQL commands such as:

```sql
DROP TABLE
DROP INDEX
TRUNCATE
ALTER TABLE
```

Do not manually manipulate relation files unless following a carefully controlled PostgreSQL recovery procedure backed by expert knowledge.

---

# 36. Tablespaces

By default, databases and their relations commonly use PostgreSQL's default tablespaces.

Tablespaces allow database administrators to place PostgreSQL objects on different filesystem locations.

Conceptually:

```text
PostgreSQL Cluster
       |
       +-- Default tablespace storage
       |
       +-- Tablespace 1 --> Storage location A
       |
       +-- Tablespace 2 --> Storage location B
```

Create a tablespace:

```sql
CREATE TABLESPACE fast_space
LOCATION '/pgdata/fast_space';
```

Then an object can be placed there, for example:

```sql
CREATE TABLE employee
(
    id integer
)
TABLESPACE fast_space;
```

---

# 37. pg_tblspc

User-defined tablespaces are represented through links under:

```text
$PGDATA/pg_tblspc
```

These links point toward the physical tablespace locations.

Because of tablespaces, the statement:

> "All PostgreSQL data files are always underneath `base/`"

is incorrect.

A better statement is:

> Databases using the default tablespace have their ordinary database relation files under `PGDATA/base`; user-defined tablespaces place relation storage elsewhere and are represented through `PGDATA/pg_tblspc`.

---

# 38. Important PostgreSQL Storage Directories

A simplified view:

```text
PGDATA
|
|-- base/
|     Database files in default tablespace
|
|-- global/
|     Cluster-wide system catalogs
|
|-- pg_wal/
|     Write-Ahead Log files
|
|-- pg_tblspc/
|     Links to user-defined tablespaces
|
|-- pg_xact/
|     Transaction commit-status information
|
|-- PG_VERSION
|
|-- postgresql.conf
|
|-- pg_hba.conf
|
`-- pg_ident.conf
```

---

# 39. Key Interview Points – Access Management

Remember these points:

**1. `pg_hba.conf` controls client authentication.**

**2. `CONNECT` controls permission to connect to a database.**

**3. `USAGE` on a schema allows access to objects through that schema.**

**4. Object-level privileges are still required.**

Example:

```sql
GRANT USAGE ON SCHEMA test1 TO ram;
GRANT SELECT ON test1.emp TO ram;
```

**5. `CREATE ON SCHEMA` allows creation of objects inside the schema.**

```sql
GRANT CREATE ON SCHEMA test1 TO ram;
```

**6. `CREATE ON DATABASE` allows activities such as creating schemas in the database.**

```sql
GRANT CREATE ON DATABASE newdb TO ram;
```

**7. `GRANT ... ON ALL TABLES` affects existing tables.**

**8. `ALTER DEFAULT PRIVILEGES` is needed for automatic privileges on future objects.**

**9. Default privileges depend on the role that creates the future objects.**

**10. ALTER/DROP rights mainly come from object ownership rather than ordinary table privileges.**

**11. PostgreSQL users and groups are implemented using roles.**

**12. Prefer RBAC instead of giving many direct grants to individual users.**

---

# 40. Key Interview Points – Physical Storage

Remember:

```text
PGDATA
   |
   +-- base/
         |
         +-- Database OID
                  |
                  +-- Relation files
```

Example:

```text
base/16912/16914
```

Where:

```text
16912 → database OID directory
16914 → relation filenode
```

Relation forks:

```text
16914       → main
16914_fsm   → Free Space Map
16914_vm    → Visibility Map
16914_init  → initialization fork for an unlogged relation
```

Large relation segments:

```text
16914
16914.1
16914.2
16914.3
```

Useful functions:

```sql
SELECT pg_relation_filepath('test1.emp');

SELECT pg_relation_filenode('test1.emp');
```

---

# 41. Quick Revision Table

| Level                 | Privilege / Configuration | Purpose                                                       |
| --------------------- | ------------------------- | ------------------------------------------------------------- |
| Client authentication | `pg_hba.conf`             | Determines whether/how a client authenticates                 |
| Database              | `CONNECT`                 | Allows connection to database                                 |
| Database              | `CREATE`                  | Allows creation of schemas and certain database-level objects |
| Database              | `TEMPORARY`               | Allows temporary tables                                       |
| Schema                | `USAGE`                   | Allows access/look-up of objects in schema                    |
| Schema                | `CREATE`                  | Allows creation of objects in schema                          |
| Table                 | `SELECT`                  | Read rows                                                     |
| Table                 | `INSERT`                  | Insert rows                                                   |
| Table                 | `UPDATE`                  | Modify rows                                                   |
| Table                 | `DELETE`                  | Delete rows                                                   |
| Table                 | `TRUNCATE`                | Truncate table                                                |
| Table                 | `REFERENCES`              | Create foreign-key references                                 |
| Table                 | `TRIGGER`                 | Create triggers                                               |
| Table                 | `MAINTAIN`                | Perform supported maintenance operations                      |
| Function/Procedure    | `EXECUTE`                 | Execute routine                                               |
| Role membership       | `GRANT role TO user`      | Inherit/use role privileges according to role settings        |

---

# 42. Complete Example – Recommended Access Model

Create database:

```sql
CREATE DATABASE newdb;
```

Connect:

```sql
\c newdb
```

Create schema:

```sql
CREATE SCHEMA test1;
```

Create table:

```sql
CREATE TABLE test1.emp
(
    eno   integer,
    ename varchar(100)
);
```

Create group-style role:

```sql
CREATE ROLE application_rw NOLOGIN;
```

Allow schema access:

```sql
GRANT USAGE
ON SCHEMA test1
TO application_rw;
```

Grant existing table permissions:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA test1
TO application_rw;
```

Configure future tables created by `postgres`:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
IN SCHEMA test1
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES
TO application_rw;
```

Create login:

```sql
CREATE ROLE ram
LOGIN
PASSWORD 'StrongPassword';
```

Add user to access role:

```sql
GRANT application_rw TO ram;
```

Access model:

```text
ram
 |
 v
application_rw
 |
 +-- USAGE on test1
 |
 +-- SELECT
 +-- INSERT
 +-- UPDATE
 `-- DELETE
      on test1 tables
```

This is cleaner and easier to maintain than repeatedly granting privileges directly to every application user.

---

# 43. Final Summary

PostgreSQL security involves two major stages:

```text
Authentication
     +
Authorization
```

Authentication decides whether the client can connect.

Authorization decides what the connected role can do.

A common object-access path is:

```text
pg_hba.conf
     ↓
Database CONNECT
     ↓
Schema USAGE
     ↓
Object privilege
```

For scalable administration:

```text
User/Login Role
      ↓
Group/Access Role
      ↓
Privileges
      ↓
Database Objects
```

For physical storage:

```text
PostgreSQL Cluster
        ↓
      PGDATA
        ↓
base/ or tablespace
        ↓
Database storage directory
        ↓
Relation filenode
        ↓
Main / FSM / VM / other forks
```

The most important administration rule is:

> Use SQL and PostgreSQL administration commands to manage database objects. Never manually remove relation files as a normal database operation.
