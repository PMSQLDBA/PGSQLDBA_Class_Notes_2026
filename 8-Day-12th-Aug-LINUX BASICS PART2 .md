# 8th Class — Linux Basics Part 2

## Date: 12th August 2026

### Topic: Linux Administration Basics for PostgreSQL DBAs

---

## 1. Switch to the Root Mount Point

### Objective

Navigate to the Linux root (`/`) directory and verify the current working directory.

### Procedure

```bash
cd /
pwd
```

### Expected Output

```text
/
```

### Technical Note

`/` is the **root directory** of the Linux filesystem hierarchy.

All other directories are organized under `/`.

Example:

```text
/
├── boot
├── dev
├── etc
├── home
├── opt
├── proc
├── root
├── run
├── sys
├── tmp
├── usr
└── var
```

---

# 2. List Files and Directories

### Command

```bash
ls -ltrh
```

### Meaning of Options

| Option | Meaning                   |
| ------ | ------------------------- |
| `-l`   | Long listing format       |
| `-t`   | Sort by modification time |
| `-r`   | Reverse the sorting order |
| `-h`   | Human-readable file sizes |

Example:

```text
drwxr-xr-x.   2 root root    6 Apr  2  2025 srv
lrwxrwxrwx.   1 root root    8 Apr  2  2025 sbin -> usr/sbin
```

### How to Identify the Entry Type

The **first character** of the permission field identifies the type.

```text
d   = directory
-   = regular file
l   = symbolic link
c   = character device
b   = block device
s   = socket
p   = named pipe (FIFO)
```

### Important Correction

Do **not** say:

> "Starting with `d` is a folder and everything else is a file."

A better statement is:

> **An entry beginning with `d` is a directory, `-` represents a regular file, and `l` represents a symbolic link.**

Your own output demonstrates this:

```text
sbin -> usr/sbin
bin  -> usr/bin
lib  -> usr/lib
```

These are symbolic links, not regular files. 

---

# 3. Check Disk Usage of Directories and Files

### Command

```bash
du -sh *
```

### Meaning

| Option | Meaning        |
| ------ | -------------- |
| `du`   | Disk usage     |
| `-s`   | Summary        |
| `-h`   | Human-readable |

Example:

```text
246M  boot
24M   etc
20K   home
1.4G  usr
17M   var
```

### Important Technical Note — `/proc`

You may see errors such as:

```text
du: cannot access 'proc/...': No such file or directory
```

This is normal when running `du` across `/proc`.

`/proc` is a **virtual filesystem** that exposes kernel and process information. Its contents are dynamic, so a process-related entry may disappear while `du` is scanning it.

Your class output demonstrates exactly this situation. 

### DBA Relevance

As a PostgreSQL DBA, disk-usage commands are important for monitoring:

```text
PostgreSQL data directory
WAL directory
Log directory
Backup directories
Temporary files
Operating-system filesystem usage
```

---

# 4. Navigate to `/var/lib`

### Commands

```bash
cd /var
pwd
```

Expected:

```text
/var
```

Then:

```bash
cd lib
pwd
```

Expected:

```text
/var/lib
```

Your class exercise creates a PostgreSQL-related directory here. 

---

# 5. Create a Directory

### Command

```bash
mkdir pgsql
```

Then:

```bash
cd pgsql
pwd
```

Expected:

```text
/var/lib/pgsql
```

### Technical Note

`mkdir` means **make directory**.

Syntax:

```bash
mkdir <directory_name>
```

For nested directories:

```bash
mkdir -p /var/lib/pgsql/data
```

The `-p` option creates parent directories when required.

---

# 6. Create an Empty File

### Command

```bash
touch test
```

Verify:

```bash
ls -ltr
```

Expected:

```text
-rw-r--r--. 1 root root 0 ... test
```

### Technical Note

`touch` can:

1. Create an empty file if it does not exist.
2. Update timestamps if the file already exists.

It does **not** specifically mean "create file only."

---

# 7. Edit a File Using `vi`

### Command

```bash
vi test
```

### Basic `vi` Modes

```text
Command mode
     ↓
Press i
     ↓
Insert mode
     ↓
Type data
     ↓
Press Esc
     ↓
Command mode
     ↓
:wq
     ↓
Save and exit
```

### Common Commands

```text
i       → Insert mode
Esc     → Return to command mode
:w      → Save
:q      → Quit
:wq     → Save and quit
:q!     → Quit without saving
```

---

# 8. Read File Contents

### Command

```bash
cat test
```

Example:

```text
hi ram how are you
you came from where
thanks for asking
```

### Technical Note

`cat` displays file contents to standard output.

For large files, prefer:

```bash
less filename
```

or:

```bash
more filename
```

For PostgreSQL log files, `less` is generally more convenient than `cat` because it allows interactive navigation.

---

# 9. Count Number of Lines

### Command

```bash
cat test | wc -l
```

Output:

```text
3
```

### Better Command

The pipe is unnecessary here.

You can directly use:

```bash
wc -l test
```

### Technical Note

`wc -l` counts newline characters/lines.

Other useful forms:

```bash
wc -l file.txt    # lines
wc -w file.txt    # words
wc -c file.txt    # bytes
```

---

# 10. Copy a File

### Command

```bash
cp test newtest
```

Verify:

```bash
ls -ltr
```

Now both files exist:

```text
test
newtest
```

### Syntax

```bash
cp <source> <destination>
```

### DBA Relevance

`cp` is commonly used for ordinary Linux file operations, but **do not use simple `cp` as a substitute for PostgreSQL-aware backup procedures**.

For PostgreSQL backups, use PostgreSQL-supported mechanisms such as:

```text
pg_dump
pg_dumpall
pg_basebackup
```

depending on the backup requirement.

---

# 11. Rename or Move a File

### Command

```bash
mv newtest donttest
```

Verify:

```bash
ls -ltr
```

### Technical Note

`mv` can be used to:

* Move a file
* Move a directory
* Rename a file
* Rename a directory

Example:

```bash
mv oldname newname
```

---

# 12. Compare Two Files

### Command

```bash
diff test donttest
```

Example output:

```text
3a4,6
> ok bye
> good night
> thank you
```

### Interpretation

The output indicates that lines were added to the second file.

`diff` is useful for comparing:

```text
Configuration files
Scripts
SQL files
Environment files
PostgreSQL configuration changes
```

For example:

```bash
diff postgresql.conf postgresql.conf.backup
```

---

# 13. Search for a Word

### Command

```bash
cat donttest | grep 'are'
```

Output:

```text
hi ram how are you
```

### Better Command

The `cat` command is unnecessary:

```bash
grep 'are' donttest
```

### Useful `grep` Options

```bash
grep 'postgres' file
grep -i 'postgres' file
grep -n 'postgres' file
grep -r 'postgres' directory
```

| Option | Meaning          |
| ------ | ---------------- |
| `-i`   | Ignore case      |
| `-n`   | Show line number |
| `-r`   | Recursive search |

### PostgreSQL DBA Example

Search PostgreSQL logs:

```bash
grep -i "error" postgresql.log
```

---

# 14. Find and Replace Text Using `vi`

Open:

```bash
vi donttest
```

Command:

```text
:%s/you/rambabu/g
```

### Meaning

```text
:       → Execute vi command
%       → Entire file
s       → Substitute
you     → Search text
rambabu → Replacement text
g       → Replace all occurrences on each line
```

Then:

```text
Esc
:wq
```

to save and exit.

### Important Note

Before performing bulk replacement in an important configuration file, create a backup and verify the result.

---

# 15. Delete a File

### Command

```bash
rm donttest
```

Depending on the environment, you may receive a confirmation prompt.

### Force Delete

Your notes use:

```bash
rm -rf test
```

### Correct Meaning

```text
-r = recursive
-f = force
```

Therefore:

```bash
rm -rf
```

means **recursively remove files/directories without normal confirmation behavior**.

### ⚠️ DBA Safety Rule

Never execute:

```bash
rm -rf
```

against a PostgreSQL data directory unless you are **100% certain** that the directory is no longer required.

Deleting a PostgreSQL cluster's data files can permanently destroy the database.

Before deleting anything:

```bash
pwd
ls -ltr
```

Then verify the target carefully.

---

# 16. Switch User

Exit the root shell:

```bash
exit
```

Then:

```bash
sudo su -
```

### Technical Note

For day-to-day administration, prefer controlled privilege escalation such as:

```bash
sudo <command>
```

rather than operating continuously as root.

The PostgreSQL server itself should **not normally run as root**.

---

# 17. Linux Users

### Command

```bash
cat /etc/passwd | grep /bin/bash
```

Better:

```bash
grep '/bin/bash' /etc/passwd
```

Example:

```text
root:x:0:0:Super User:/root:/bin/bash
ec2-user:x:1000:1000:Cloud User:/home/ec2-user:/bin/bash
```

### `/etc/passwd`

`/etc/passwd` contains information about local Linux user accounts.

The fields are colon-separated:

```text
username:password_placeholder:UID:GID:GECOS:home_directory:login_shell
```

Example:

```text
postgres:x:1001:1001::/home/postgres:/bin/bash
```

---

# 18. Create the PostgreSQL Operating-System User

### Command

```bash
useradd postgres
```

Verify:

```bash
grep postgres /etc/passwd
```

### PostgreSQL Official Recommendation

PostgreSQL should run under a **separate operating-system user account**.

The PostgreSQL documentation commonly uses the name `postgres`, although another OS username can be used.

The PostgreSQL server account should own the data managed by PostgreSQL and should not be shared with unrelated daemons. PostgreSQL also recommends that the server account not own the PostgreSQL executable files, reducing the risk that a compromised server process could modify them. ([PostgreSQL][1])

### Important Distinction

Do not confuse:

```text
Linux user:       postgres
PostgreSQL role:  postgres
```

They are different concepts.

A PostgreSQL role is a **database-level identity**.

PostgreSQL uses roles to manage database access and privileges. Roles can behave as users, groups, or both. ([PostgreSQL][2])

---

# 19. Linux User vs Linux Group

### Linux User

A user represents an operating-system account.

Example:

```text
postgres
root
ec2-user
```

### Linux Group

A group is a collection of users.

Groups are used by the operating system when determining group-based access to files and resources.

### Corrected Statement

Instead of:

> "Group → it will have set of permissions"

Use:

> **A group is a collection of Linux users. File and directory permissions can be assigned to the group, allowing group members to access those resources according to the assigned permissions.**

---

# 20. Linux User/Group vs PostgreSQL Role

This distinction is very important for a PostgreSQL DBA.

```text
OPERATING SYSTEM
----------------
Linux user
Linux group
File ownership
File permissions


POSTGRESQL
----------
PostgreSQL role
Database privileges
Object ownership
Role membership
Authentication
```

PostgreSQL has unified the historical concepts of database users and groups into **roles**. A role can function as a login user, a group-like privilege container, or both. ([PostgreSQL][3])

---

# 21. Linux File Ownership and Permissions

Example:

```text
-rwx------. 1 postgres postgres 0 test
```

Breakdown:

```text
-        → Regular file
rwx      → Owner permissions
---      → Group permissions
---      → Others permissions
postgres → Owner
postgres → Group
0        → File size
test     → Filename
```

### Permission Categories

```text
Owner
Group
Others
```

### Permission Types

```text
r = read
w = write
x = execute
```

Numeric values:

```text
r = 4
w = 2
x = 1
```

Therefore:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
3 = -wx
2 = -w-
1 = --x
0 = ---
```

---

# 22. `chmod` — Change File Permissions

### Syntax

```bash
chmod <permissions> <file>
```

Example:

```bash
chmod 444 test
```

Result:

```text
-r--r--r--
```

Meaning:

```text
Owner  → read
Group  → read
Others → read
```

---

## Example: `chmod 666`

```bash
chmod 666 test
```

Result:

```text
-rw-rw-rw-
```

Meaning:

```text
Owner  → read/write
Group  → read/write
Others → read/write
```

### ⚠️ Security Note

Avoid giving unnecessary write permission to **Others**.

For PostgreSQL-related files, permissions should be based on the minimum access required.

---

# 23. Example: `chmod 700`

```bash
chmod 700 test
```

Result:

```text
-rwx------
```

Meaning:

```text
Owner  → read/write/execute
Group  → no permission
Others → no permission
```

### DBA Relevance

Restrictive permissions are particularly important for:

```text
PostgreSQL data directories
Private keys
Password files
Sensitive configuration files
Backup credentials
```

---

# 24. Change File Ownership

### Command

```bash
chown postgres:postgres test
```

Verify:

```bash
ls -ltr
```

Expected:

```text
-rwx------. 1 postgres postgres ... test
```

### Syntax

```bash
chown <user>:<group> <file>
```

Example:

```bash
chown postgres:postgres /var/lib/pgsql
```

### PostgreSQL Relevance

The PostgreSQL server process needs appropriate ownership/access to the database cluster and related files.

PostgreSQL's official source-installation procedure explicitly creates a PostgreSQL OS account and assigns ownership of the database data directory to that account. ([PostgreSQL][4])

---

# 25. View Running Processes

### Command

```bash
ps -ef
```

This displays running processes.

Important columns include:

```text
UID
PID
PPID
C
STIME
TTY
TIME
CMD
```

### PostgreSQL DBA Example

```bash
ps -ef | grep postgres
```

Your class output showed only the `grep` process, indicating that a PostgreSQL server process was not visible under that search at that time. 

### Better Alternatives

On a system using systemd:

```bash
systemctl status postgresql
```

And:

```bash
systemctl list-units --type=service | grep -i postgres
```

---

# 26. Set the Hostname

Check current hostname:

```bash
hostname
```

Example:

```text
ip-172-31-0-51.ap-south-1.compute.internal
```

Set hostname:

```bash
hostnamectl hostname db1
```

Verify:

```bash
hostname
```

Expected:

```text
db1
```

### Technical Note

A hostname identifies the Linux system on the network.

For a PostgreSQL environment, meaningful hostnames can help distinguish:

```text
db1
db2
pg-primary
pg-standby
pg-backup
```

---

# 27. `/etc/hosts`

View:

```bash
cat /etc/hosts
```

Example:

```text
127.0.0.1   localhost
::1         localhost
172.31.0.51 db1
```

### Technical Note

`/etc/hosts` provides local hostname-to-IP address mappings.

It can be useful in controlled environments, but production DNS architecture should be considered before relying heavily on static `/etc/hosts` entries.

---

# 28. Check IP Address

### Command

```bash
hostname -i
```

Example:

```text
172.31.0.51
```

### Additional Commands

```bash
hostname -I
ip addr
ip route
```

For troubleshooting PostgreSQL connectivity, `ip addr` and `ip route` provide more detailed network information.

---

# PostgreSQL Software Installation

## 29. PostgreSQL Installation Methods

PostgreSQL can be installed using different approaches.

For Linux, the major approaches are:

### Method 1 — Package Installation

Examples:

```text
RPM
DNF/YUM
DEB/APT
```

### Method 2 — Source-Code Installation

Build PostgreSQL from its source code.

PostgreSQL's official documentation separates installation from binaries/packages and installation from source code. ([PostgreSQL][5])

---

# 30. Package-Based Installation

For Red Hat-family Linux systems, PostgreSQL provides packages through the PostgreSQL Yum Repository, in addition to versions supplied by the Linux distribution. The official PostgreSQL download page covers RHEL, Rocky Linux, AlmaLinux, Fedora, and related distributions. ([PostgreSQL][6])

Example distribution package:

```bash
dnf install postgresql-server
```

However, the PostgreSQL version installed depends on the operating-system repository/module configuration.

The PostgreSQL project also provides its own repository when a specific supported PostgreSQL version is required. ([PostgreSQL][6])

---

# 31. Package Installation — Important Correction

Your original notes say:

> "Complete automated installation. Everything will be installed automatically with a single command."

This is too broad.

A package manager can install the PostgreSQL software and dependencies, but **initialization and service configuration can still be required**.

For example, on supported Red Hat-family distributions, the official PostgreSQL download instructions show:

```bash
postgresql-setup --initdb
systemctl enable postgresql.service
systemctl start postgresql.service
```

The exact commands can vary by distribution/version/package source. ([PostgreSQL][6])

Therefore:

> **Package installation automates software installation, but DBA/server initialization and service configuration may still be required.**

---

# 32. Source-Code Installation

### Definition

Source-code installation means:

> Download the PostgreSQL source code, configure it for the target system, compile it, install the binaries, create/configure the PostgreSQL operating-system account, initialize a database cluster, and start the server.

PostgreSQL's official source-installation documentation provides the complete procedure. ([PostgreSQL][4])

---

# 33. Source Installation — Prerequisites

PostgreSQL source compilation requires several development tools and libraries.

Important requirements include:

```text
GNU make 3.81+
C compiler
tar
gzip or bzip2
Flex
Bison
Perl
Readline
zlib
ICU
OpenSSL — when encrypted connections are required
```

PostgreSQL 18 also documents **Meson** as an alternative build system; for non-Windows platforms it is currently considered experimental. ([PostgreSQL][7])

---

# 34. Obtain PostgreSQL Source Code

Official source distributions can be downloaded from the PostgreSQL website.

Example:

```bash
tar xf postgresql-18.x.tar.bz2
cd postgresql-18.x
```

PostgreSQL's official documentation describes downloading and unpacking the source distribution before starting the build process. ([PostgreSQL][8])

---

# 35. Source-Code Build Process

The traditional Autoconf/Make procedure is:

```bash
./configure
make
make install
```

PostgreSQL's official short version also includes creation of the PostgreSQL OS account, data directory, initialization, and server startup. ([PostgreSQL][4])

### High-Level SOP

```text
1. Install build dependencies
        ↓
2. Download PostgreSQL source
        ↓
3. Extract source archive
        ↓
4. Run ./configure
        ↓
5. Run make
        ↓
6. Run make install
        ↓
7. Create PostgreSQL OS user
        ↓
8. Create PostgreSQL data directory
        ↓
9. Change ownership to PostgreSQL OS user
        ↓
10. Run initdb
        ↓
11. Start PostgreSQL
        ↓
12. Verify PostgreSQL
```

---

# 36. Source Installation Location

The original notes state:

```text
/usr/local/pgsql/bin
```

This is commonly associated with the default source installation prefix.

### Important Correction

It is **not mandatory**.

The installation prefix can be changed during configuration.

For example:

```bash
./configure --prefix=/opt/postgresql/18
```

The PostgreSQL documentation explains that installation files are placed according to the directories selected during configuration. ([PostgreSQL][4])

Therefore:

> **`/usr/local/pgsql` is a default/example installation location, not a mandatory PostgreSQL source-installation directory.**

---

# 37. PostgreSQL Data Directory

The PostgreSQL **installation directory** and **data directory** are different.

### Installation Directory

Contains PostgreSQL software such as:

```text
bin/
lib/
include/
share/
```

### Data Directory

Contains the PostgreSQL database cluster and configuration/data files.

Example:

```text
/usr/local/pgsql/data
```

or another location chosen by the DBA.

PostgreSQL's `initdb` command initializes the database cluster in the specified data directory. ([PostgreSQL][9])

---

# 38. Initialize the Database Cluster

### Command

```bash
initdb -D /usr/local/pgsql/data
```

or:

```bash
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
```

### Technical Definition

`initdb` creates a new PostgreSQL **database cluster**.

A PostgreSQL cluster is the collection of databases managed by one PostgreSQL server instance.

### Important

`initdb` must be run by the appropriate PostgreSQL operating-system user.

The bootstrap superuser normally takes the name of the operating-system user running `initdb`, unless another name is specified with `-U`. ([PostgreSQL][9])

---

# 39. Start PostgreSQL from a Source Installation

PostgreSQL's official source-installation example uses:

```bash
pg_ctl -D /usr/local/pgsql/data -l logfile start
```

([PostgreSQL][4])

### Meaning

```text
pg_ctl       → PostgreSQL server control utility
-D           → PostgreSQL data directory
-l           → Log file
start        → Start the server
```

Verify:

```bash
pg_ctl -D /usr/local/pgsql/data status
```

---

# 40. Create a Database and Connect

After starting PostgreSQL:

```bash
createdb test
```

Connect:

```bash
psql test
```

PostgreSQL's official source installation procedure uses this as a basic post-installation verification step. ([PostgreSQL][4])

---

# 41. Package Installation vs Source Installation

| Feature                    | Package Installation         | Source Installation                      |
| -------------------------- | ---------------------------- | ---------------------------------------- |
| Installation method        | Pre-built packages           | Build from source                        |
| Dependency handling        | Mostly package manager       | DBA/admin responsibility                 |
| Compilation                | Not normally required        | Required                                 |
| Installation customization | More limited                 | Highly customizable                      |
| Version selection          | Repository/package dependent | Source version selected by administrator |
| OS integration             | Usually better               | Requires more manual setup               |
| Administration effort      | Lower                        | Higher                                   |
| Learning/lab use           | Good                         | Excellent                                |
| Production suitability     | Commonly preferred           | Suitable when requirements justify it    |

The official PostgreSQL Linux download guidance notes that package installation is generally recommended when PostgreSQL is integrated with the platform's package management because it provides OS integration and management functionality. ([PostgreSQL][10])

---

# 42. Multiple PostgreSQL Versions on One Linux Server

### Important Correction to Original Notes

The statement:

> "If I install 17.3, 17.3 software will override 17.2 software."

is **not a general PostgreSQL rule**.

Multiple PostgreSQL versions can coexist on the same Linux server when they are installed/configured appropriately.

For example, a server could conceptually contain:

```text
PostgreSQL 17
PostgreSQL 18
```

with separate:

```text
installation directories
data directories
ports
services
configuration
```

However, running multiple versions requires careful planning to avoid port conflicts, service conflicts, and accidental connections to the wrong cluster.

The PostgreSQL package repositories provide packages for multiple supported PostgreSQL versions. ([PostgreSQL][6])

---

# 43. Production vs Lab Installation

### Source-Code Installation

Best suited for:

```text
Learning
Development
Testing
Custom builds
Special compilation requirements
```

It can also be used in production when there is a specific operational reason and the organization can manage the additional build/patching responsibilities.

### Package Installation

Generally preferable when you want:

```text
OS integration
Package management
Security updates
Dependency management
Service management
Operational consistency
```

The official PostgreSQL Linux download page recommends package-managed installation when available and appropriate. ([PostgreSQL][10])

So instead of writing:

> "Source installation is not for realtime."

use:

> **Source installation is valid for production, but package-based installation is often operationally simpler because it integrates with the operating system's package-management and service-management mechanisms.**

---

# 44. PostgreSQL DBA Technical SOP — Installation Checklist

## Pre-Installation

* [ ] Verify Linux OS and version.
* [ ] Verify CPU architecture.
* [ ] Verify available RAM.
* [ ] Verify disk space.
* [ ] Verify filesystem mount points.
* [ ] Verify network configuration.
* [ ] Set an appropriate hostname.
* [ ] Confirm DNS/`/etc/hosts` requirements.
* [ ] Decide package installation vs source installation.
* [ ] Decide PostgreSQL major version.
* [ ] Plan installation directory.
* [ ] Plan data directory.
* [ ] Plan WAL location if required.
* [ ] Plan PostgreSQL service management.
* [ ] Plan backup and monitoring.

## Source Installation

* [ ] Install compiler and build dependencies.
* [ ] Download PostgreSQL source.
* [ ] Extract source archive.
* [ ] Run `configure`.
* [ ] Run `make`.
* [ ] Run `make install`.
* [ ] Create PostgreSQL OS account.
* [ ] Create data directory.
* [ ] Assign correct ownership.
* [ ] Initialize cluster using `initdb`.
* [ ] Start PostgreSQL using `pg_ctl`.
* [ ] Verify server status.
* [ ] Connect using `psql`.
* [ ] Create test database.
* [ ] Configure environment variables.
* [ ] Configure automatic startup if required.
* [ ] Configure logging.
* [ ] Configure authentication.
* [ ] Configure firewall/network access.
* [ ] Validate backup strategy.

---

# 45. Quick DBA Revision

```text
LINUX
│
├── /                     → Root directory
├── /etc                  → System configuration
├── /var                  → Variable data
├── /var/lib              → Variable application data
├── /proc                 → Virtual process/kernel information
├── /tmp                  → Temporary files
├── /usr                  → Userland programs/libraries
└── /root                 → Root user's home directory
```

```text
LINUX FILE
│
├── Owner
├── Group
└── Others
     │
     ├── Read    = 4
     ├── Write   = 2
     └── Execute = 1
```

```text
POSTGRESQL INSTALLATION
│
├── Package Installation
│
└── Source Installation
       │
       ├── Dependencies
       ├── Source Code
       ├── configure
       ├── make
       ├── make install
       ├── PostgreSQL OS User
       ├── Data Directory
       ├── initdb
       ├── pg_ctl
       └── psql
```

## Key Takeaways

1. **Linux user ≠ PostgreSQL role.**
2. **Linux group ≠ PostgreSQL role**, although both can be used to organize permissions.
3. `d` means **directory**; `l` means **symbolic link**; `-` means **regular file**.
4. `chmod` changes permissions; `chown` changes ownership.
5. `ps -ef` displays processes.
6. `systemctl` is commonly used for service management on systemd-based Linux systems.
7. `initdb` initializes a PostgreSQL database cluster. ([PostgreSQL][9])
8. `pg_ctl` controls a PostgreSQL server instance.
9. PostgreSQL should run under a **dedicated OS account**, not root. ([PostgreSQL][1])
10. Package installation and source installation are different deployment approaches.
11. `/usr/local/pgsql` is an example/default source-installation prefix, **not a mandatory path**. ([PostgreSQL][4])
12. Multiple PostgreSQL major versions can coexist when properly separated and configured.
13. For production, package-managed installation is often operationally easier when supported by the platform. ([PostgreSQL][10])
14. Never use `rm -rf` casually against PostgreSQL data directories.
15. Always distinguish the **PostgreSQL software/installation directory** from the **PostgreSQL data directory**.

[1]: https://www.postgresql.org/docs/current/postgres-user.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.1. The PostgreSQL User Account"
[2]: https://www.postgresql.org/docs/18/user-manag.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 21. Database Roles"
[3]: https://www.postgresql.org/docs/current/sql-grant.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: GRANT"
[4]: https://www.postgresql.org/docs/current/install-make.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 17.3. Building and Installation with Autoconf and Make"
[5]: https://www.postgresql.org/docs/current/admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Part III. Server Administration"
[6]: https://www.postgresql.org/download/linux/redhat/?utm_source=chatgpt.com "PostgreSQL: Linux downloads (Red Hat family)"
[7]: https://www.postgresql.org/docs/current/install-requirements.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 17.1. Requirements"
[8]: https://www.postgresql.org/docs/current/install-getsource.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 17.2. Getting the Source"
[9]: https://www.postgresql.org/docs/current/app-initdb.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: initdb"
[10]: https://www.postgresql.org/download/linux/?utm_source=chatgpt.com "PostgreSQL: Linux downloads (other)"
