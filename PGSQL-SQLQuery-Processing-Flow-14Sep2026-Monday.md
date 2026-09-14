# Technical SOP Article

# PostgreSQL SQL Query Processing Flow

**Document Type:** PostgreSQL Technical SOP / Training Article
**Applicable Version:** PostgreSQL 18.x
**Purpose:** Explain the correct end-to-end flow of a SQL statement inside PostgreSQL
**Reference:** PostgreSQL 18.6 Official Documentation

---

## 1. Purpose

This SOP explains how PostgreSQL processes a SQL statement from the time it is received from a client application until the result is returned.

The flow is important for PostgreSQL DBAs because it helps in understanding:

* SQL parsing
* Object and data-type resolution
* Query rewriting
* Query planning and optimization
* Execution
* Index scans and sequential scans
* Joins and sorting
* Query performance
* `EXPLAIN` and `EXPLAIN ANALYZE`
* Common query-processing errors

PostgreSQL officially describes the main query-processing stages as:

**Connection → Parser → Rewrite System → Planner/Optimizer → Executor → Result**

([PostgreSQL][1])

---

# 2. High-Level PostgreSQL Query Processing Flow

The recommended technical flow is:

```text
                    PostgreSQL Query Processing

Client / Application
        |
        v
Connection Established
        |
        v
Authentication
        |
        v
SQL Statement Received
        |
        v
+----------------------+
|  1. Parser           |
|  Syntax Validation   |
|  Query Tree          |
+----------------------+
        |
        v
+------------------------------+
|  2. Parse Analysis /         |
|     Transformation           |
|  Names / Types / Objects     |
+------------------------------+
        |
        v
+------------------------------+
|  3. Rewrite System           |
|  Rules / Views / Rewriting   |
+------------------------------+
        |
        v
+------------------------------+
|  4. Planner / Optimizer      |
|  Paths / Costs / Best Plan   |
+------------------------------+
        |
        v
+------------------------------+
|  5. Executor                 |
|  Executes the Selected Plan |
+------------------------------+
        |
        v
+------------------------------+
|  Storage / Access Methods    |
|  Seq Scan / Index Scan etc.  |
+------------------------------+
        |
        v
Result / Command Status
        |
        v
Client / Application
```

**Important:** The PostgreSQL documentation's formal high-level stages are Parser, Rewrite System, Planner/Optimizer, and Executor. Parse analysis/transformation is part of the parser stage internally, so it is useful to show it separately for training, but it should not be confused with a separate official top-level component. ([PostgreSQL][1])

---

# 3. Step 1 — Client Connection

Before PostgreSQL can process a SQL statement, a client must establish a connection to the PostgreSQL server.

Examples of clients include:

* `psql`
* pgAdmin
* Application servers
* Python applications
* Java applications
* .NET applications
* JDBC/ODBC clients
* Monitoring tools

Example:

```sql
psql -h pgdb01 -U postgres -d dvdrental
```

The client establishes a connection and sends the SQL statement to PostgreSQL.

The official PostgreSQL documentation describes establishing the connection as the first step in the query path. ([PostgreSQL][1])

---

# 4. Step 2 — Authentication

PostgreSQL authenticates the connecting user according to the configured authentication rules.

One of the most important files involved is:

```text
pg_hba.conf
```

For example:

```text
host    all    all    192.168.1.0/24    scram-sha-256
```

This determines how a connection from the specified network is authenticated.

### Important distinction

Do **not** describe "Security Check" as a formal PostgreSQL query-processing stage.

Security involves several areas, including:

* Authentication
* Role membership
* Database privileges
* Schema privileges
* Table privileges
* Column privileges
* Row-level security
* Object ownership

Therefore, **Security is broader than one step in the query-processing pipeline.**

---

# 5. Step 3 — Parser

Once PostgreSQL receives the SQL statement, the **parser** checks its syntax.

Example:

```sql
SELECT first_name, last_name
FROM customer
WHERE customer_id = 10;
```

The parser checks whether the SQL follows PostgreSQL's SQL grammar.

If the syntax is invalid:

```sql
SELECT FROM customer WHERE;
```

PostgreSQL returns a syntax error.

### Parser responsibilities

The parser:

* Reads the SQL statement.
* Checks SQL syntax.
* Identifies SQL keywords.
* Identifies expressions.
* Identifies tables and columns syntactically.
* Builds an internal **query tree**.

The official PostgreSQL documentation explicitly states that the parser checks the query for correct syntax and creates a query tree. ([PostgreSQL][1])

---

# 6. Step 4 — Parse Analysis / Transformation

This is where the trainer's term **"Resolver"** can be useful as a teaching concept.

However:

> **Resolver is not the official name of a separate PostgreSQL query-processing stage.**

PostgreSQL performs parse analysis and transformation after parsing.

This stage gives PostgreSQL more semantic information about the query.

For example:

```sql
SELECT first_name
FROM customer;
```

PostgreSQL needs to determine:

* Which `customer` table is being referenced?
* Which `first_name` column is being referenced?
* What data type does `first_name` have?
* Which schema should PostgreSQL search?
* Which functions/operators are being referenced?
* Are the referenced objects valid?

The PostgreSQL documentation has a dedicated **Transformation Process** section under the parser stage. ([PostgreSQL][2])

### Example

If the query contains:

```sql
SELECT c.first_name
FROM customer c;
```

PostgreSQL needs to associate:

```text
c
 |
 +----> customer table
```

and:

```text
c.first_name
 |
 +----> customer.first_name
```

This is why calling this concept **name/object resolution** can be useful when teaching beginners.

---

# 7. Step 5 — Rewrite System

After parsing, PostgreSQL passes the query tree through the **Rewrite System**.

This is an important stage that was missing from the trainer's diagram.

The Rewrite System is positioned between the parser and the planner/optimizer. ([PostgreSQL][3])

### Main purpose

The rewrite system can transform the query according to PostgreSQL's rules.

One important example is **views**.

Suppose we create:

```sql
CREATE VIEW customer_view AS
SELECT customer_id, first_name, last_name
FROM customer;
```

The user executes:

```sql
SELECT *
FROM customer_view;
```

Internally, PostgreSQL can rewrite the query so that it operates against the underlying tables represented by the view definition.

The PostgreSQL documentation specifically identifies views as an important application of the rewrite system. ([PostgreSQL][1])

### Important

The Rewrite System is **not the same thing as:**

* Stored procedures
* Functions
* Triggers
* Optimizer

It is specifically part of PostgreSQL's query rewriting mechanism. ([PostgreSQL][4])

---

# 8. Step 6 — Planner / Optimizer

After rewriting, PostgreSQL sends the query tree to the **Planner/Optimizer**.

This is one of the most important components for database performance.

The planner considers different possible execution paths.

For example:

```sql
SELECT *
FROM customer
WHERE customer_id = 10;
```

PostgreSQL may consider:

```text
Option 1
Sequential Scan
       |
Read the table
       |
Find customer_id = 10
```

or:

```text
Option 2
Index Scan
       |
Use customer_id index
       |
Find required row
```

The planner estimates the cost of the available paths and selects what it considers the cheapest appropriate plan. ([PostgreSQL][1])

---

# 9. What Does the Optimizer Consider?

The planner can consider factors such as:

* Table size
* Available indexes
* Column statistics
* Estimated number of rows
* Selectivity
* Join conditions
* Join algorithms
* Sort requirements
* Aggregation
* Parallel execution
* Cost parameters
* Query structure

For example, it may choose:

```text
Sequential Scan
```

instead of:

```text
Index Scan
```

even when an index exists.

### Important DBA point

**Index exists ≠ PostgreSQL must use the index.**

The planner chooses the plan based on its cost estimates and other planning information.

---

# 10. Step 7 — Execution Plan

The planner produces an **execution plan**.

Example:

```text
Index Scan using customer_pkey on customer
  Index Cond: (customer_id = 10)
```

This plan tells the executor what it should do.

You can view the plan using:

```sql
EXPLAIN
SELECT *
FROM customer
WHERE customer_id = 10;
```

PostgreSQL's `EXPLAIN` command displays the execution plan generated by the planner. ([PostgreSQL][5])

---

# 11. Step 8 — Executor

The **Executor** receives the plan created by the planner and actually executes it.

This is where the trainer's word **"Dispatcher"** should be corrected.

The official PostgreSQL terminology is:

> **Executor**

The executor processes the plan recursively and obtains the required rows.

([PostgreSQL][6])

For example:

```text
Planner
   |
   v
Index Scan
   |
   v
Executor
   |
   v
Access required rows
```

The executor can process operations such as:

* Sequential scans
* Index scans
* Index-only scans
* Nested Loop joins
* Hash joins
* Merge joins
* Sorts
* Aggregations
* Filters
* Inserts
* Updates
* Deletes
* MERGE operations

The official documentation states that the executor is used for `SELECT`, `INSERT`, `UPDATE`, `DELETE`, and `MERGE`. ([PostgreSQL][6])

---

# 12. Step 9 — Storage and Access Methods

During execution, the executor interacts with PostgreSQL's storage and access mechanisms.

For example:

```text
Executor
   |
   +---- Sequential Scan
   |
   +---- Index Scan
   |
   +---- Index Only Scan
   |
   +---- Bitmap Heap Scan
   |
   +---- Bitmap Index Scan
```

For a sequential scan:

```text
Executor
   |
   v
Seq Scan
   |
   v
Table pages
   |
   v
Required rows
```

For an index scan:

```text
Executor
   |
   v
Index Scan
   |
   v
Index
   |
   v
Table / Heap
   |
   v
Required rows
```

The executor uses the storage system while scanning relations and performing other operations. ([PostgreSQL][6])

---

# 13. Step 10 — Result Returned to Client

After execution completes, PostgreSQL returns the result to the client.

For example:

```sql
SELECT first_name, last_name
FROM customer
WHERE customer_id = 10;
```

The client may receive:

```text
first_name    last_name
-----------   ---------
MARY          SMITH
```

For DML statements, PostgreSQL may instead return command status information such as:

```text
INSERT 0 1
```

or:

```text
UPDATE 5
```

---

# 14. Complete Example

Consider:

```sql
SELECT first_name, last_name
FROM customer
WHERE customer_id = 10;
```

The simplified processing flow is:

```text
1. Client
      |
      v
2. Connection / Authentication
      |
      v
3. Parser
      |
      |-- Check SQL syntax
      |-- Build query tree
      |
      v
4. Parse Analysis / Transformation
      |
      |-- Resolve table
      |-- Resolve columns
      |-- Resolve data types
      |
      v
5. Rewrite System
      |
      |-- Apply applicable rewrite rules
      |
      v
6. Planner / Optimizer
      |
      |-- Consider possible paths
      |-- Estimate costs
      |-- Select best plan
      |
      v
7. Execution Plan
      |
      v
8. Executor
      |
      |-- Execute selected plan
      |
      v
9. Access Method
      |
      |-- Index Scan / Seq Scan etc.
      |
      v
10. Result
      |
      v
11. Client
```

---

# 15. Example Using `EXPLAIN`

Run:

```sql
EXPLAIN
SELECT first_name, last_name
FROM customer
WHERE customer_id = 10;
```

PostgreSQL might produce a plan similar to:

```text
Index Scan using customer_pkey on customer
  Index Cond: (customer_id = 10)
```

The flow is:

```text
SQL
 |
 v
Parser
 |
 v
Parse Analysis
 |
 v
Rewrite
 |
 v
Planner
 |
 |---- chooses Index Scan
 |
 v
Executor
 |
 v
Index Scan
 |
 v
Required Row
 |
 v
Client
```

---

# 16. `EXPLAIN` vs `EXPLAIN ANALYZE`

This is very important for DBAs.

### `EXPLAIN`

```sql
EXPLAIN
SELECT *
FROM customer
WHERE customer_id = 10;
```

Shows the plan generated by the planner.

It does **not** normally execute the query.

### `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
SELECT *
FROM customer
WHERE customer_id = 10;
```

Actually executes the statement and provides actual execution information.

For example:

```text
Planning Time: 0.150 ms
Execution Time: 0.080 ms
```

PostgreSQL documentation notes that planning time includes generating and optimizing the plan, while execution time covers execution by the executor and related execution work. ([PostgreSQL][5])

**Production warning:** Be careful with `EXPLAIN ANALYZE` on `INSERT`, `UPDATE`, `DELETE`, and `MERGE`, because the statement is actually executed. ([PostgreSQL][7])

---

# 17. Where Does "Security Check" Fit?

This is where the trainer's diagram can create confusion.

Do not teach:

```text
Parser
   ↓
Security Check
   ↓
Resolver
```

as if these were PostgreSQL's official stages.

Instead, understand security as a **cross-cutting area**.

It includes:

```text
Connection
    |
    +--> Authentication
    |
    +--> Role
    |
    +--> Database privileges
    |
    +--> Schema privileges
    |
    +--> Table/Column privileges
    |
    +--> Row-Level Security
```

For example:

```sql
SELECT *
FROM customer;
```

PostgreSQL must ultimately ensure that the connected role has the required access.

---

# 18. Trainer's Terms vs Official PostgreSQL Terms

This table is useful for interviews and training.

| Trainer's terminology | Recommended PostgreSQL terminology                                    |
| --------------------- | --------------------------------------------------------------------- |
| Syntaxer              | **Parser / Parsing**                                                  |
| Security Check        | **Authentication + Authorization/Privileges**                         |
| Resolver              | **Parse Analysis / Transformation**                                   |
| Parser                | **Parser**                                                            |
| Optimizer             | **Planner / Optimizer**                                               |
| Dispatcher            | **Do not use as the official term; use Executor for query execution** |
| Missing               | **Rewrite System**                                                    |
| Missing               | **Executor**                                                          |

### Key takeaway

The trainer's terms can be used as **conceptual teaching terminology**, but they should not replace PostgreSQL's official terminology in your technical documentation.

---

# 19. Final Recommended PostgreSQL Architecture Diagram

For your PostgreSQL DBA notes, I recommend using this exact flow:

```text
                  POSTGRESQL QUERY PROCESSING

                    Client / Application
                            |
                            v
                 Connection / Authentication
                            |
                            v
                     SQL Statement
                            |
                            v
                 +----------------------+
                 |       PARSER         |
                 | Syntax checking      |
                 | Query tree creation  |
                 +----------------------+
                            |
                            v
                 +----------------------+
                 | Parse Analysis /     |
                 | Transformation       |
                 | Object / Name / Type |
                 | Resolution           |
                 +----------------------+
                            |
                            v
                 +----------------------+
                 |   REWRITE SYSTEM     |
                 | Rules / Views /      |
                 | Query Rewriting      |
                 +----------------------+
                            |
                            v
                 +----------------------+
                 | PLANNER / OPTIMIZER  |
                 | Paths / Statistics   |
                 | Cost / Best Plan     |
                 +----------------------+
                            |
                            v
                 +----------------------+
                 |      EXECUTOR        |
                 | Executes the plan    |
                 +----------------------+
                            |
                            v
                 +----------------------+
                 | STORAGE / ACCESS     |
                 | METHODS              |
                 | Seq Scan / Index     |
                 | Scan / Joins / etc.  |
                 +----------------------+
                            |
                            v
                       Result Set
                            |
                            v
                   Client / Application
```

This is the flow it is recommended to use in **PostgreSQL 18 technical training/SOP material**. 
The core stages are directly aligned with the PostgreSQL 18.6 documentation: **Parser → Rewrite System → Planner/Optimizer → Executor**, with connection establishment before query processing. ([PostgreSQL][1])

### Official PostgreSQL reference

[PostgreSQL 18 — The Path of a Query](https://www.postgresql.org/docs/18/query-path.html)

[PostgreSQL 18 — Overview of PostgreSQL Internals](https://www.postgresql.org/docs/18/overview.html)

[PostgreSQL 18 — Planner/Optimizer](https://www.postgresql.org/docs/18/planner-optimizer.html)

[PostgreSQL 18 — Executor](https://www.postgresql.org/docs/18/executor.html)

[1]: https://www.postgresql.org/docs/18/query-path.html "PostgreSQL: Documentation: 18: 51.1. The Path of a Query"
[2]: https://www.postgresql.org/docs/18/overview.html "PostgreSQL: Documentation: 18: Chapter 51. Overview of PostgreSQL Internals"
[3]: https://www.postgresql.org/docs/18/rule-system.html "PostgreSQL: Documentation: 18: 51.4. The PostgreSQL Rule System"
[4]: https://www.postgresql.org/docs/18/rules.html "PostgreSQL: Documentation: 18: Chapter 39. The Rule System"
[5]: https://www.postgresql.org/docs/18/using-explain.html "PostgreSQL: Documentation: 18: 14.1. Using EXPLAIN"
[6]: https://www.postgresql.org/docs/18/executor.html "PostgreSQL: Documentation: 18: 51.6. Executor"
[7]: https://www.postgresql.org/docs/18/sql-explain.html "PostgreSQL: Documentation: 18: EXPLAIN"
