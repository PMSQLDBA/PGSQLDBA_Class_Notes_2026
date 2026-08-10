# Technical SOP: Connect pgAdmin 4 from Windows to PostgreSQL 18 on Ubuntu 24.04 VM

[PDF] (https://github.com/PMSQLDBA/PGSQLDBA_Class_Notes_2026/blob/main/Technical_SOP_pgAdmin4_PostgreSQL18_UbuntuVM.pdf)

**Document Type:** Technical SOP

**Technology:** PostgreSQL 18, Ubuntu 24.04, VMware Workstation, pgAdmin 4

**Purpose:** Configure and validate remote connectivity from pgAdmin 4 on a Windows workstation to PostgreSQL 18 running inside an Ubuntu 24.04 VMware virtual machine.

---

## 1. Objective

This SOP explains how to configure a PostgreSQL 18 server running on an Ubuntu 24.04 VMware VM so that **pgAdmin 4 running on a Windows local machine** can connect remotely over TCP/IP.

### Environment

| Component            | Details            |
| -------------------- | ------------------ |
| Client OS            | Windows            |
| Client Tool          | pgAdmin 4          |
| Windows IP           | `192.168.0.73`     |
| Virtualization       | VMware Workstation |
| Database Server OS   | Ubuntu 24.04       |
| PostgreSQL Version   | PostgreSQL 18      |
| PostgreSQL Cluster   | `18/main`          |
| PostgreSQL Server IP | `192.168.0.145`    |
| PostgreSQL Port      | `5432`             |
| Database User        | `postgres`         |
| Authentication       | `scram-sha-256`    |

---

# 2. Architecture

```text
┌─────────────────────────────────────────────┐
│             Windows Local Machine           │
│                                             │
│  IP: 192.168.0.73                           │
│  pgAdmin 4                                  │
└──────────────────────┬──────────────────────┘
                       │
                       │ TCP/5432
                       │
                       ▼
┌─────────────────────────────────────────────┐
│             VMware Workstation              │
│                                             │
│  Ubuntu 24.04 VM                            │
│  IP: 192.168.0.145                          │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │ PostgreSQL 18                         │  │
│  │ Cluster: 18/main                      │  │
│  │ Port: 5432                            │  │
│  │                                       │  │
│  │ listen_addresses = 192.168.0.145      │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  Ubuntu UFW                                 │
│  TCP/5432 ← 192.168.0.73                    │
└─────────────────────────────────────────────┘
```

---

# 3. Prerequisites

Before starting, verify:

* Ubuntu 24.04 VM is running.
* PostgreSQL 18 is installed.
* PostgreSQL cluster `18/main` exists.
* VM has IP `192.168.0.145`.
* Windows machine can reach the VM.
* pgAdmin 4 is installed on Windows.
* PostgreSQL `postgres` user password is known.
* VMware networking permits communication between Windows and VM.

---

# 4. Verify PostgreSQL Cluster

On Ubuntu:

```bash
pg_lsclusters
```

Expected:

```text
Ver Cluster Port Status Owner    Data directory
18  main    5432 online postgres /var/lib/postgresql/18/main
```

This confirms that the PostgreSQL 18 cluster is:

```text
Version : 18
Cluster : main
Port    : 5432
Status  : online
```

---

# 5. Verify PostgreSQL Service

Run:

```bash
systemctl status postgresql
```

The Ubuntu PostgreSQL packaging uses a general `postgresql.service` service and cluster-specific services such as:

```bash
postgresql@18-main.service
```

Verify the actual cluster:

```bash
systemctl status postgresql@18-main
```

Expected:

```text
Active: active (running)
```

---

# 6. Verify PostgreSQL IP Configuration

Initially, the server was configured with:

```text
listen_addresses = localhost
```

This allows local TCP/IP connections but does not allow PostgreSQL to listen on the VM's network interface.

Check:

```bash
sudo -u postgres psql -c "SHOW listen_addresses;"
```

### Initial state

```text
localhost
```

### Required configuration

Edit:

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

Set:

```text
listen_addresses = '192.168.0.145'
```

### Why?

PostgreSQL must listen on the VM's network interface so that remote clients such as pgAdmin can establish a TCP connection.

PostgreSQL's `listen_addresses` parameter controls the TCP/IP addresses on which the server listens.

---

# 7. Restart PostgreSQL

Because `listen_addresses` is a server-start parameter, restart PostgreSQL:

```bash
sudo systemctl restart postgresql@18-main
```

Verify:

```bash
sudo -u postgres psql -c "SHOW listen_addresses;"
```

Expected:

```text
 listen_addresses
------------------
 192.168.0.145
```

---

# 8. Verify PostgreSQL Port

Run:

```bash
sudo -u postgres psql -c "SHOW port;"
```

Expected:

```text
5432
```

You can also verify the operating-system listener:

```bash
sudo ss -lntp | grep 5432
```

Expected result should indicate PostgreSQL is listening on:

```text
192.168.0.145:5432
```

---

# 9. Configure `pg_hba.conf`

PostgreSQL uses `pg_hba.conf` to control client authentication.

The configuration file is:

```text
/etc/postgresql/18/main/pg_hba.conf
```

View it:

```bash
sudo cat /etc/postgresql/18/main/pg_hba.conf
```

### Existing local rules

Typical entries include:

```text
local   all             postgres                                peer
local   all             all                                     peer

host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
```

These primarily cover local connections.

---

# 10. Allow Remote LAN Connections

For the lab environment, the following rule was added:

```text
host    all    all    192.168.0.0/24    scram-sha-256
```

This allows clients from the `192.168.0.0/24` subnet to authenticate using SCRAM.

### Security improvement

Because the actual Windows client IP is:

```text
192.168.0.73
```

a more restrictive production-style rule could be:

```text
host    all    all    192.168.0.73/32    scram-sha-256
```

`/32` means one specific IPv4 address.

For this lab, the existing `/24` rule worked correctly.

---

# 11. Reload PostgreSQL

After changing `pg_hba.conf`, reload PostgreSQL:

```bash
sudo systemctl reload postgresql@18-main
```

A complete restart is not required just for a `pg_hba.conf` change.

---

# 12. Validate `pg_hba.conf`

PostgreSQL provides the `pg_hba_file_rules` view.

Run:

```bash
sudo -u postgres psql -c "SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules;"
```

Expected relevant entry:

```text
line_number | type | database | user_name | address     | auth_method  | error
------------+------+----------+-----------+-------------+--------------+------
135         | host | {all}    | {all}     | 192.168.0.0 | scram-sha-256|
```

The important point is:

```text
error = NULL
```

This confirms that PostgreSQL successfully parsed the rule.

---

# 13. Configure Ubuntu Firewall

The initial UFW configuration was:

```text
Status: active
Default: deny (incoming)
```

Existing rules included:

```text
27017    ALLOW IN    192.168.0.73
22       ALLOW IN    192.168.0.73
```

There was initially **no rule for PostgreSQL port 5432**.

Therefore, even though PostgreSQL was listening correctly, Ubuntu's firewall blocked the connection.

---

# 14. Allow PostgreSQL From Windows

Because only the Windows machine needs access, the recommended firewall rule is:

```bash
sudo ufw allow from 192.168.0.73 to any port 5432 proto tcp
```

Verify:

```bash
sudo ufw status numbered
```

Expected:

```text
22/tcp     ALLOW IN    192.168.0.73
27017      ALLOW IN    192.168.0.73
5432/tcp   ALLOW IN    192.168.0.73
```

### Security principle

Avoid unnecessarily opening:

```text
5432 from anywhere
```

Do not use:

```text
sudo ufw allow 5432
```

when only one client requires access.

Restricting the source IP reduces the attack surface.

---

# 15. Test Network Connectivity

From Windows PowerShell:

```powershell
Test-NetConnection 192.168.0.145 -Port 5432
```

### Initial result

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

This was an important troubleshooting result.

It proved:

```text
Windows → VM
```

was working, but:

```text
Windows → VM TCP/5432
```

was blocked.

### Root cause

Ubuntu UFW was blocking port `5432`.

After adding the firewall rule, the test returned:

```text
ComputerName     : 192.168.0.145
RemoteAddress    : 192.168.0.145
RemotePort       : 5432
SourceAddress    : 192.168.0.73
TcpTestSucceeded : True
```

This confirmed that the network path was fully operational.

---

# 16. Configure PostgreSQL Password Authentication

The `pg_hba.conf` rule uses:

```text
scram-sha-256
```

Therefore the PostgreSQL user needs a valid password.

To reset the `postgres` password:

```bash
sudo -u postgres psql
```

Then:

```sql
\password postgres
```

Enter the new password twice.

Exit:

```sql
\q
```

Verify the role:

```bash
sudo -u postgres psql -c "SELECT rolname, rolcanlogin, rolsuper FROM pg_roles WHERE rolname = 'postgres';"
```

Expected:

```text
rolname  | rolcanlogin | rolsuper
---------+-------------+---------
postgres | t           | t
```

---

# 17. pgAdmin 4 Configuration

On the Windows machine:

**pgAdmin 4 → Servers → Register → Server**

## General tab

```text
Name:
PostgreSQL 18 - Ubuntu VM
```

## Connection tab

Use:

| Parameter            | Value                          |
| -------------------- | ------------------------------ |
| Host name/address    | `192.168.0.145`                |
| Port                 | `5432`                         |
| Maintenance database | `postgres`                     |
| Username             | `postgres`                     |
| Password             | PostgreSQL `postgres` password |

Click:

**Save**

---

# 18. Authentication Error Encountered

During testing, pgAdmin initially returned:

```text
FATAL: password authentication failed for user "postgres"
```

At this stage:

```text
Network connectivity        ✅
Port 5432                   ✅
PostgreSQL listener         ✅
pg_hba.conf                 ✅
UFW                         ✅
Authentication              ❌
```

### Resolution

The `postgres` password was reset using:

```bash
sudo -u postgres psql
```

followed by:

```sql
\password postgres
```

The new password was then entered into pgAdmin.

The connection succeeded.

---

# 19. Final Working Configuration

## PostgreSQL

```text
Version:
18

Cluster:
18/main

IP:
192.168.0.145

Port:
5432
```

## `postgresql.conf`

```text
listen_addresses = '192.168.0.145'
```

## `pg_hba.conf`

```text
host    all    all    192.168.0.0/24    scram-sha-256
```

## UFW

```text
5432/tcp    ALLOW IN    192.168.0.73
```

## Windows

```text
IP:
192.168.0.73
```

## pgAdmin

```text
Host:
192.168.0.145

Port:
5432

Database:
postgres

User:
postgres
```

---

# 20. Troubleshooting Decision Tree

Use this sequence whenever pgAdmin cannot connect.

```text
                 pgAdmin Connection Failed
                           │
                           ▼
               Can Windows ping VM?
                           │
                 ┌─────────┴─────────┐
                 │                   │
                NO                  YES
                 │                   │
          Check VMware         Test TCP 5432
          networking                │
                             ┌──────┴──────┐
                             │             │
                            FAIL          PASS
                             │             │
                             ▼             ▼
                       Check UFW       Check pg_hba
                       Check listener      │
                                           ▼
                                  Authentication error?
                                      │
                               ┌──────┴──────┐
                               │             │
                              YES           NO
                               │             │
                               ▼             ▼
                         Reset password   pgAdmin
                         Check role       connection
```

---

# 21. Common Errors and Their Meaning

| Error                            | Likely Cause             | Action                            |
| -------------------------------- | ------------------------ | --------------------------------- |
| `Connection timed out`           | Network/firewall         | Check VMware/UFW                  |
| `Connection refused`             | PostgreSQL not listening | Check `ss -lntp`                  |
| `no pg_hba.conf entry`           | Client not authorized    | Check `pg_hba.conf`               |
| `password authentication failed` | Incorrect password       | Reset PostgreSQL password         |
| `database does not exist`        | Incorrect database name  | Check database                    |
| `role does not exist`            | Incorrect username       | Check `pg_roles`                  |
| `TcpTestSucceeded: False`        | TCP port inaccessible    | Check listener/UFW                |
| `TcpTestSucceeded: True`         | Network path works       | Move to PostgreSQL authentication |

---

# 22. Useful Validation Commands

### Check clusters

```bash
pg_lsclusters
```

### Check PostgreSQL service

```bash
systemctl status postgresql@18-main
```

### Check listener

```bash
sudo ss -lntp | grep 5432
```

### Check listening addresses

```bash
sudo -u postgres psql -c "SHOW listen_addresses;"
```

### Check port

```bash
sudo -u postgres psql -c "SHOW port;"
```

### Check HBA configuration

```bash
sudo -u postgres psql -c "SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules;"
```

### Check firewall

```bash
sudo ufw status verbose
```

### Check PostgreSQL roles

```bash
sudo -u postgres psql -c "SELECT rolname, rolcanlogin, rolsuper FROM pg_roles;"
```

### Test from Windows

```powershell
Test-NetConnection 192.168.0.145 -Port 5432
```

---

# 23. Security Best Practices

For a learning environment, the current configuration is appropriate. For production, improve it further.

### 1. Restrict `pg_hba.conf`

Instead of:

```text
192.168.0.0/24
```

prefer a specific trusted client or application subnet where practical.

For your current Windows client:

```text
host    all    all    192.168.0.73/32    scram-sha-256
```

### 2. Restrict UFW

Use:

```bash
sudo ufw allow from 192.168.0.73 to any port 5432 proto tcp
```

rather than allowing the entire Internet.

### 3. Don't use `trust`

Avoid:

```text
host all all 0.0.0.0/0 trust
```

### 4. Don't expose PostgreSQL directly to the Internet

Do not open port `5432` globally unless there is a specific, secured architecture requiring it.

### 5. Avoid using `postgres` for applications

`postgres` is a superuser.

Applications should use dedicated roles with minimum required privileges.

### 6. Consider TLS

For production or untrusted networks, configure PostgreSQL SSL/TLS and appropriate certificate validation.

---

# 24. Official PostgreSQL References

The key PostgreSQL concepts used in this configuration are documented in the official PostgreSQL documentation:

* [PostgreSQL 18 — Connections and Authentication](https://www.postgresql.org/docs/18/client-authentication.html?utm_source=chatgpt.com)
* [PostgreSQL 18 — `pg_hba.conf`](https://www.postgresql.org/docs/18/auth-pg-hba-conf.html?utm_source=chatgpt.com)
* [PostgreSQL 18 — Connection Settings / `listen_addresses`](https://www.postgresql.org/docs/18/runtime-config-connection.html?utm_source=chatgpt.com)
* [PostgreSQL 18 — Database Roles](https://www.postgresql.org/docs/18/user-manag.html?utm_source=chatgpt.com)
* [PostgreSQL 18 — SSL/TLS Encryption](https://www.postgresql.org/docs/18/ssl-tcp.html?utm_source=chatgpt.com)

---

# 25. Final SOP Summary

The complete troubleshooting and configuration sequence was:

```text
1. Verify PostgreSQL 18 cluster
        ↓
2. Identify VM IP = 192.168.0.145
        ↓
3. Change listen_addresses
        ↓
4. Restart PostgreSQL
        ↓
5. Configure pg_hba.conf
        ↓
6. Reload PostgreSQL
        ↓
7. Validate pg_hba_file_rules
        ↓
8. Check Ubuntu UFW
        ↓
9. Allow TCP/5432 from 192.168.0.73
        ↓
10. Test Windows TCP connectivity
        ↓
11. Reset postgres password
        ↓
12. Configure pgAdmin 4
        ↓
13. Successfully connect
```

### Final result

```text
                ┌─────────────────────┐
                │   Windows PC        │
                │   192.168.0.73      │
                │                     │
                │   pgAdmin 4         │
                └──────────┬──────────┘
                           │
                           │ TCP 5432
                           │
                           ▼
                ┌─────────────────────┐
                │   Ubuntu 24.04 VM   │
                │   192.168.0.145     │
                │                     │
                │   UFW               │
                │   5432 ALLOW        │
                │   from .73          │
                │                     │
                │ PostgreSQL 18       │
                │ Cluster: 18/main    │
                │ Port: 5432          │
                └─────────────────────┘

                     CONNECTION
                         ✅
```

**Root cause encountered:** Ubuntu UFW had a default `deny (incoming)` policy and did not initially allow TCP/5432.

**Secondary authentication issue:** The `postgres` password used by pgAdmin was incorrect; resetting the PostgreSQL role password resolved the authentication failure.

**Final outcome:** Windows pgAdmin 4 successfully connects remotely to PostgreSQL 18 running on the Ubuntu 24.04 VMware VM.
