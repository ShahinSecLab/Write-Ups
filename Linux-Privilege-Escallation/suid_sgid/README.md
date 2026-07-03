
# SUID/SGID — Privilege Escalation

**Date:** June 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Medium <br>
**Tools:** SSH, find, searchsploit, wget, chmod

# Table of Contents

- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Connecting to the Target via SSH](#step-1--connecting-to-the-target-via-ssh)
- [Step 2 — Finding SUID Binaries](#step-2--finding-suid-binaries)
- [Step 3 — Searching for an Exploit with Searchsploit](#step-3--searching-for-an-exploit-with-searchsploit)
- [Step 4 — Downloading the Exploit to the Target](#step-4--downloading-the-exploit-to-the-target)
- [Step 5 — Running the Exploit and Getting Root](#step-5--running-the-exploit-and-getting-root)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [References](#references)
- [What I Achieved](#what-i-achieved)

## Introduction

SUID (Set User ID) is a special Linux file permission bit. When a binary has the SUID bit set, it runs as the owner of the file — not as the user who launched it. Most SUID binaries on a system are owned by root, which means they run as root regardless of who executes them. If one of those binaries is vulnerable or can be abused, a normal user can use it to get a root shell.

## Why This Attack Works

Linux uses file permission bits to control who can read, write, or execute files. The SUID bit is an extra permission that tells the system to run the file as its owner instead of the person running it. This is used legitimately by tools like `passwd` — which needs root access to modify /`etc/shadow` even when run by a normal user.

The problem comes in when a binary with the SUID bit set is either:

- Vulnerable to a known CVE — like Exim 4.84-3 in this case
- Abusable to spawn a shell — like `find`, `vim`, or `bash` with the SUID bit set

In either case, since the binary runs as root, whatever it does — including spawning a shell — happens as root.

## Lab Setup
```
| Component        | Details                                  |
|------------------|------------------------------------------|
| Attacker Machine | Kali Linux                               |
| Victim Machine   | Debian Linux                             |
| Victim IP        | 192.168.5.133                            |
| Access Method    | SSH with valid low-privilege credentials |
| Network          | VMware Host-Only Network                 |
```

## Tools Prepared on Kali Before Starting
```
|            Tool          |                Purpose                      |
|--------------------------|---------------------------------------------|
| `searchsploit`           | Find known exploits for discovered binaries |
| `python3 -m http.server` | Host the exploit file for download          |
| `wget`                   | Download the exploit onto the target        |
```

## What I Needed Before Starting
```
|             What                         |                       Why                           |
|------------------------------------------|-----------------------------------------------------|
| SSH credentials for a low-privilege user | Starting point for the attack                       |
| `find` command                           | To scan the system for SUID binaries                |
| `searchsploit` on Kali                   | To find a working exploit for the vulnerable binary |
| Python HTTP server                       | To host the exploit file for the target to download |
```

## What I Understood During the Process

While working through this attack I realized that:

- Finding SUID binaries should be one of the first things checked on any Linux machine after getting initial access
- Most SUID binaries on a system are there for legitimate reasons — the key is spotting the ones that are unusual or outdated
- exim-4.84-3 stood out straight away because it is a mail server binary and does not need to be SUID in most environments
- Searchsploit made finding the right exploit fast — I just copied the version number and searched
- The exploit worked straight out of the box without any modifications needed

## Attack Flow
```
Connected to the target over SSH with low privilege credentials
                        ↓
Ran find to locate all SUID binaries on the system
                        ↓
Spotted /usr/sbin/exim-4.84-3 as an unusual SUID binary
                        ↓
Searched for exploits with searchsploit exim 4.84-3
                        ↓
Found CVE-2016-1531 local privilege escalation exploit (39535.sh)
                        ↓
Downloaded the exploit to Kali with searchsploit -m 39535
                        ↓
Started Python HTTP server on Kali
                        ↓
Downloaded the exploit to the target with wget
                        ↓
Added execute permission with chmod +x
                        ↓
Ran the exploit with ./39535.sh
                        ↓
Got a root shell immediately
                        ↓
                whoami → root
```

## Step 1 — Connecting to the Target via SSH

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa user@192.168.5.133
```

- `-o HostKeyAlgorithms=+ssh-rsa` : Allows older RSA host key algorithm — needed for older Linux systems
- `-o PubkeyAcceptedAlgorithms=+ssh-rsa` : Allows older RSA public key algorithm for authentication
- `192.168.5.133` : Target IP
- `user`: User Name

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

I logged in as `user` — a normal low privilege account on the system.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>

## Step 2 — Finding SUID Binaries

```bash
user@debian:~$ find / -type f -perm -4000 2>/dev/null
```

### Flag Breakdown
```
|  Flag         |            Description                                           |
|---------------|------------------------------------------------------------------|
| `/`           | Starts searching from the root of the filesystem.                |
| `-type f`     | Searches for files only.                                         |
| `-perm -4000` | Finds files with the SUID bit set.                               |
| `2>/dev/null` | Redirects error messages to `/dev/null` to keep the output clean.|
```

**Output:**

```
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/sudoedit
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/local/bin/suid-so
/usr/local/bin/suid-env
/usr/local/bin/suid-env2
/usr/sbin/exim-4.84-3
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/pt_chown
/bin/ping6
/bin/ping
/bin/mount
/bin/su
/bin/umount
/sbin/mount.nfs
```
Most of these are normal system binaries that are supposed to have the SUID bit set. But a few stood out as unusual:

- `/usr/local/bin/suid-so`
- `/usr/local/bin/suid-env`
- `/usr/local/bin/suid-env2`
- `/usr/sbin/exim-4.84-3`

`/usr/sbin/exim-4.84-3` was the most interesting one — Exim is a mail transfer agent and this old version is known to be vulnerable to local privilege escalation through its SUID bit.