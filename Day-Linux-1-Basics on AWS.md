Day:6-Linux-1-Basics on AWS

# Linux Setup & Linux Basics — Properly Segregated

## 1. Linux Lab / Environment Setup

### 1.1 VMware

* VMware Player setup → currently not working because it became licensed. 

### 1.2 AWS Cloud Environment

* AWS Cloud can be used to create a Linux server.
* Notes mention a **6-month free trial** environment. 

### 1.3 Why Cloud Instead of On-Premises?

* Complete infrastructure maintenance is not entirely in the user's hands.
* High availability is possible.
* Cloud providers provide a ready-made infrastructure that can be used directly.
* Charges apply based on usage/trial terms. 

---

# 2. AWS Cloud Fundamentals

## 2.1 EC2 Instance

In AWS, virtual servers are called **EC2 Instances (Elastic Compute Cloud)**. 

Before working with a cloud environment, basic AWS concepts should be understood.

---

## 2.2 AWS Region

A **Region** represents the geographical location where AWS resources are created.

Example from the notes:

* Mumbai Region
* AWS Region code: `ap-south-1` 

---

## 2.3 Availability Zones

An AWS Region contains multiple Availability Zones.

Example:

```text
Mumbai Region
│
├── Availability Zone 1
├── Availability Zone 2
└── Availability Zone 3
```

Purpose:

* High Availability
* Disaster Recovery

The notes refer to Mumbai-1, Mumbai-2 and Mumbai-3 as three data centers. 

---

# 3. AWS Networking

## 3.1 VPC — Virtual Private Cloud

A **VPC** is the main virtual network in AWS.

```text
AWS Region
   │
   └── VPC
       │
       ├── Subnet
       ├── Subnet
       └── Subnet
```

A company can have multiple VPCs with different IP address ranges. 

---

## 3.2 Subnets

The notes describe the VPC as being subdivided into subnet groups.

Example:

```text
Mumbai Region
│
├── AZ-1
│   └── Subnet Group 1
│
├── AZ-2
│   └── Subnet Group 2
│
└── AZ-3
    └── Subnet Group 3
```

Notes:

* Mumbai-1 → subnet group 1
* Mumbai-2 → subnet group 2
* Mumbai-3 → subnet group 3 

---

# 4. AWS Security

## 4.1 Security Group

A **Security Group** controls network traffic to and from an EC2 instance.

It determines:

* Who can access the server
* Which IP address can connect
* Which port can be accessed
* Which outbound traffic is allowed 

### 4.1.1 Inbound Rules — Ingress

Controls **incoming traffic**.

```text
Source IP
    ↓
Port
    ↓
EC2 Instance
```

Example:

```text
Source: 10.10.10.10
Port:   22
Protocol: TCP
```

The notes describe inbound rules as determining **from which IP address and through which port traffic can enter**. 

### 4.1.2 Outbound Rules — Egress

Controls **outgoing traffic**.

```text
EC2 Instance
    ↓
Port
    ↓
Destination IP
```

The notes describe outbound rules as determining **where and through which port traffic can leave**. 

---

# 5. EC2 Instance Creation Components

Before launching an EC2 instance, the major selections are:

## 5.1 AMI — Amazon Machine Image

AMI is used as a **template for server creation**. 

Example:

```text
AMI
 │
 └── EC2 Instance
```

The notes use Red Hat Linux as the example operating system.

---

## 5.2 Instance Type / Instance Class

Determines the compute resources allocated to the server, including:

* CPU
* Memory

Higher-capacity instance types generally increase cost. 

Reference in the notes:

[AWS EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/t4/?utm_source=chatgpt.com)

---

## 5.3 Key Pair

A **key pair** is created for SSH-based access to the Linux server.

Example:

```text
Local Computer
      │
      │ SSH + Private Key
      ▼
AWS EC2 Linux Server
```

The notes refer to the downloaded `.pem` file as the SSH key. 

---

## 5.4 Storage

AWS provides different storage options.

The notes specifically reference **EBS volume types**. 

[AWS EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html?utm_source=chatgpt.com)

---

# 6. AWS Account Setup

## 6.1 AWS Account Registration

The notes contain the following sequence:

1. Open AWS signup page.
2. Provide root-user email address.
3. Provide AWS account name.
4. Verify email address.
5. Open email received from AWS.
6. Click the verification link.
7. Reset/set the password.
8. Provide credit-card details.
9. Log in again. 

---

## 6.2 MFA Setup

The notes recommend setting up MFA using a mobile authenticator application.

Sequence:

```text
AWS Account
    │
    ▼
Enable MFA
    │
    ▼
QR Code
    │
    ▼
Authenticator App
    │
    ▼
OTP
```

The notes specifically mention Google Duo. 

---

# 7. AWS Console — EC2 Instance Creation

## 7.1 Region Selection

Select:

```text
Mumbai
ap-south-1
```



---

## 7.2 Navigate to EC2

```text
AWS Console
   │
   └── EC2
       │
       └── Instances
           │
           └── Launch Instance
```



---

## 7.3 Instance Configuration

The notes specify:

| Configuration       | Value in Notes        |
| ------------------- | --------------------- |
| Server Name         | `pgsql-aug26`         |
| OS                  | Red Hat Linux         |
| Version             | Red Hat 10            |
| Instance Type       | `t2.micro`            |
| Key Pair            | `pgsql-aug26`         |
| VPC                 | Default               |
| Subnet              | Default               |
| Security Group      | New                   |
| Storage             | Additional EBS volume |
| Additional Storage  | Not more than 30 GB   |
| Number of Instances | 1                     |



---

# 8. Connecting to the Linux EC2 Instance

## 8.1 Windows

The notes indicate using **Command Prompt (`cmd`)** from Windows. 

---

## 8.2 macOS — Navigate to Downloads

```bash
cd Downloads
```



---

## 8.3 Select EC2 Instance

AWS Console:

```text
EC2
 ↓
Instances
 ↓
Select Instance
 ↓
Connect
 ↓
SSH Client
```



---

## 8.4 Set Private-Key Permissions — macOS

```bash
chmod 400 "pgsql-aug26.pem"
```

The notes state that this permission step is needed for Mac, while Windows does not require it in the same way. 

---

## 8.5 SSH Connection

```bash
ssh -i "pgsql-aug26.pem" ec2-user@<EC2-DNS-name>
```

The notes show an example EC2 public DNS name. 

---

# 9. First SSH Connection

During the first connection, SSH asks whether the host should be trusted.

Example:

```text
Are you sure you want to continue connecting
(yes/no/[fingerprint])?
```

Enter:

```text
yes
```

The host is then added to the local known-hosts list. 

---

# 10. Verify Linux Server Connection

Once connected:

## 10.1 Check Operating System

```bash
uname
```

Expected:

```text
Linux
```



---

## 10.2 Check Current User

```bash
whoami
```

Example:

```text
ec2-user
```



---

## 10.3 Check Hostname

```bash
hostname
```

Example:

```text
ip-172-31-0-51.ap-south-1.compute.internal
```



---

## 10.4 Check Server IP Address

```bash
hostname -i
```

Example:

```text
172.31.0.51
```



---

# 11. Linux File System / Mount Points

## 11.1 Drives vs Mount Points

Unlike Windows, Linux does not normally represent storage using drive letters such as:

```text
C:
D:
E:
```

Instead, Linux uses **mount points / file systems**.

The root mount point is:

```text
/
```

The notes compare `/` to the Windows `C:` drive. 

---

# 12. Check File Systems

Use:

```bash
df -h
```

This displays:

* File system
* Size
* Used space
* Available space
* Utilization percentage
* Mount point

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda3      9.8G  1.9G  7.9G  20% /
```



---

# 13. Check Disks and Partitions

Use:

```bash
lsblk
```

Example from the notes:

```text
xvda       10G  disk
├─xvda1     1M  part
├─xvda2   200M  part  /boot/efi
└─xvda3   9.8G  part  /

xvdb       10G  disk
```



This demonstrates the difference between:

```text
Disk
  ↓
Partition
  ↓
File System
  ↓
Mount Point
```

---

# 14. Linux Users and Privileges

## 14.1 Normal User

The notes state that a normal user cannot execute many administrative commands. 

---

## 14.2 Root User

`root` is the Linux administrative/superuser account and has extensive privileges.



---

## 14.3 Switch to Root

From `ec2-user`:

```bash
sudo su -
```

Result:

```text
[ec2-user@ip-172-31-0-51 ~]$ sudo su -
[root@ip-172-31-0-51 ~]#
```



Verify:

```bash
whoami
```

Output:

```text
root
```



---

# 15. Linux CPU Information

## 15.1 Check CPU Configuration

Command:

```bash
lscpu
```

The notes show:

```text
Architecture: x86_64
CPU(s):       1
```



---

# 16. Linux Memory Information

## 16.1 Check RAM

Command:

```bash
free -h
```

Example from the notes:

```text
Mem:
total       951Mi
used        288Mi
free        594Mi
buff/cache  185Mi
available   662Mi

Swap:
0B
```



---

# 17. Linux Server Uptime

## 17.1 Check Uptime

Command:

```bash
uptime
```

Example:

```text
14:38:51 up 20 min, 1 user,
load average: 0.00, 0.00, 0.00
```



This provides information about:

* Current time
* Server uptime
* Number of logged-in users
* Load average

---

# 18. Linux Processes

## 18.1 List Running Processes

Command:

```bash
ps -ef
```

Example process information includes:

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



---

## 18.2 Process Column Definitions

| Column | Meaning                          |
| ------ | -------------------------------- |
| UID    | Username / user ID               |
| PID    | Process ID                       |
| PPID   | Parent Process ID                |
| C      | CPU utilization indicator        |
| STIME  | Process start time               |
| TTY    | Terminal associated with process |
| TIME   | CPU time consumed                |
| CMD    | Command being executed           |

The notes explicitly define UID, PID, PPID, STIME, TIME and CMD. 

---

# 19. Linux CPU Utilization

## 19.1 Real-Time CPU Monitoring

Command:

```bash
top
```

The example shows:

```text
Tasks: 105 total
1 running
104 sleeping
0 stopped
0 zombie
```

CPU:

```text
%Cpu(s):
0.0 us
0.0 sy
0.0 ni
100.0 id
0.0 wa
0.0 hi
0.0 si
0.0 st
```



---

## 19.2 Memory Information from `top`

```text
MiB Mem:
951.2 total
593.8 free
288.9 used
185.8 buff/cache
662.3 avail Mem
```

Swap:

```text
0.0 total
0.0 free
0.0 used
```



---

# 20. `top` Process-Level Information

The process section of `top` displays:

| Column  | Purpose              |
| ------- | -------------------- |
| PID     | Process ID           |
| USER    | Process owner        |
| PR      | Priority             |
| NI      | Nice value           |
| VIRT    | Virtual memory       |
| RES     | Resident memory      |
| SHR     | Shared memory        |
| S       | Process state        |
| %CPU    | CPU utilization      |
| %MEM    | Memory utilization   |
| TIME+   | CPU time             |
| COMMAND | Process/command name |

The notes provide examples of these columns. 

---

# 21. Understanding CPU Idle %

The example shows:

```text
%Cpu(s):
0.0 us
0.0 sy
0.0 ni
100.0 id
```

The notes interpret:

```text
100.0 idle = 100% CPU available / idle
```



---

# Overall Learning Structure

Your original notes are best organized into **six major learning modules**:

```text
LINUX SETUP & LINUX BASICS
│
├── 1. Lab / Environment Setup
│
├── 2. AWS Cloud Fundamentals
│   ├── EC2
│   ├── Region
│   ├── Availability Zones
│   ├── VPC
│   ├── Subnets
│   ├── Security Groups
│   ├── AMI
│   ├── Instance Type
│   ├── Key Pair
│   └── EBS Storage
│
├── 3. AWS Account & EC2 Setup
│   ├── Account Registration
│   ├── MFA
│   ├── Region Selection
│   ├── EC2 Launch
│   └── Instance Configuration
│
├── 4. Linux Server Access
│   ├── SSH
│   ├── Key Permissions
│   ├── uname
│   ├── whoami
│   ├── hostname
│   └── hostname -i
│
├── 5. Linux Storage & Users
│   ├── Mount Points
│   ├── df -h
│   ├── lsblk
│   ├── Normal User
│   ├── root
│   └── sudo su -
│
└── 6. Linux Server Administration Basics
    ├── CPU → lscpu
    ├── Memory → free -h
    ├── Uptime → uptime
    ├── Processes → ps -ef
    └── CPU/Memory Monitoring → top
```

This segregation separates the **AWS concepts**, **AWS provisioning**, **Linux connectivity**, **Linux storage**, **Linux user administration**, and **Linux performance/process commands** instead of mixing them together.
