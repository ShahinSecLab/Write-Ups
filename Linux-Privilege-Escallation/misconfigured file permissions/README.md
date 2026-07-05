# Misconfigured File Permissions — /etc/passwd

**Date:** June 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Easy <br>
**Tools:** SSH, openssl, nano, su 

## Table of Contents

- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Connecting to the Target via SSH](#step-1--connecting-to-the-target-via-ssh)
- [Step 2 — Checking `/etc/passwd` Permissions](#step-2--checking-etcpasswd-permissions)
- [Step 3 — Generating a Password Hash and Editing `/etc/passwd`](#step-3--generating-a-password-hash-and-editing-etcpasswd)
- [Step 4 — Logging in as Root with the New User](#step-4--logging-in-as-root-with-the-new-user)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [References](#references)
- [What I Achieved](#what-i-achieved)

## Introduction

Misconfigured File Permissions is one of the simplest privilege escalation techniques on Linux. If a critical system file like /etc/passwd is writable by normal users, I can add my own root level user directly to it. The next time I switch to that user, Linux gives me a full root shell — no exploit or CVE needed.

## Why This Attack Works

On a normal Linux system, `/etc/passwd` is readable by everyone but only writable by root. This file holds basic user account information — usernames, user IDs, group IDs, home directories, and shell paths.
The password field in `/etc/passwd` normally contains an `x` — which tells Linux to look at 
`/etc/shadow` for the real password hash. But if I replace that `x` with an actual password hash I generate myself, Linux will use my hash directly — completely skipping `/etc/shadow`.
Since I can write to the file, I can add a brand new user with UID 0 (root level) and a password I control. When I switch to that user, Linux sees UID 0 and gives me a full root shell.

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

## What I Needed Before Starting
```
|                    What                  |                      Why                             |
|------------------------------------------|------------------------------------------------------|
| SSH credentials for a low-privilege user | Starting point for the attack                        |
| Write access to `/etc/passwd`            | The misconfiguration that makes this attack possible |
| `openssl` on Kali                        | To generate a password hash                          |
| `nano` on the target                     | To edit `/etc/passwd`                                |
```

## What I Understood During the Process

While working through this attack I realized that:

- `/etc/passwd` being world writable is one of the most dangerous misconfigurations on any Linux system
- I did not need any exploit or CVE — just one line added to a file was enough to get root
- The `x` in the password field is the key — replacing it with a real hash bypasses `/etc/shadow` completely
- Setting UID to `0` in the new user entry gives root level access regardless of the username
- This kind of misconfiguration is easy to miss during system setup and very hard to detect without proper file integrity monitoring

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