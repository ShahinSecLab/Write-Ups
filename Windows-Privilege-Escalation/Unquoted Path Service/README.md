# Unquoted Path Service

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:** Privilege Escalation  
**Difficulty:** Easy  
**Tools:** msfvenom, Metasploit, winPEAS, accesschk.exe, certutil

# Table of Contents

- [What is Unquoted Path Service?](#what-is-unquoted-path-service)
- [Why This Attack Works](#why-this-attack-works)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Finding the Unquoted Path Service](#step-1--finding-the-unquoted-path-service)
- [Step 2 — Finding a Writable Folder in the Path](#step-2--finding-a-writable-folder-in-the-path)
- [Step 3 — Generating a New Payload and Downloading it to the Victim](#step-3--generating-a-new-payload-and-downloading-it-to-the-victim)
- [Step 4 — Copying the Payload and Starting the Service](#step-4--copying-the-payload-and-starting-the-service)
- [Step 5 — Getting a SYSTEM Shell](#step-5--getting-a-system-shell)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [What I Achieved](#what-i-achieved)

## What is Unquoted Path Service?

Unquoted Path Service is a Windows privilege escalation technique. When a service binary path has spaces in it but no quotes around the full path, Windows does not know exactly where the path ends. So it tries to find the executable by checking multiple locations one by one. If I can drop a malicious file in one of those locations before the real binary, Windows runs mine instead — and since the service runs as SYSTEM, I get a SYSTEM shell.

## Why This Attack Works

Windows handles unquoted paths with spaces in a specific way. Take this path for example:

```bash
C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
```

Windows does not read this as one full path. It breaks it up at every space and tries each combination:

```bash
C:\Program.exe
C:\Program Files\Unquoted.exe
C:\Program Files\Unquoted Path Service\Common.exe
C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
```

It stops at the first one it finds. So if I drop a file called Common.exe inside C:\Program Files\Unquoted Path Service\, Windows picks it up and runs it as SYSTEM before ever reaching the real binary.

## Lab Setup

```
|       Component      |         Details          |
|----------------------|--------------------------|
| **Attacker Machine** | Kali Linux               |
| **Attacker IP**      | 192.168.5.128            |
| **Victim Machine**   | Windows 10               |
| **Victim IP**        | 192.168.5.129            |
| **Network**          | VMware Host-Only Network |
| **Domain**           | WORKGROUP                |
```

## What I Understood During the Process

While working through this attack I realized that:

- A missing pair of quotes around a service path can lead to full SYSTEM access
- The attack only works if I can write to one of the folders Windows checks
- winPEAS finds these misconfigurations automatically and flags them clearly
- This is one of the most common privilege escalation paths found in real Windows environments
- Fixing it is as simple as adding quotes around the binary path

## Attack Flow

```
winPEAS flagged unquotedsvc service — unquoted path with spaces detected
                        ↓
Checked service config — runs as LocalSystem (SYSTEM)
                        ↓
Checked folder permissions along the binary path
                        ↓
Found C:\Program Files\Unquoted Path Service\ is writable by normal users
                        ↓
Generated malicious payload rev.exe on Kali with msfvenom
                        ↓
Hosted payload over HTTP with Python HTTP server
                        ↓
Downloaded rev.exe to victim using certutil
                        ↓
Copied rev.exe to writable folder as Common.exe
                        ↓
Started Metasploit listener on port 4444
                        ↓
Started unquotedsvc service
                        ↓
Windows found Common.exe first and ran it as SYSTEM
                        ↓
Metasploit caught the shell
                        ↓
whoami → nt authority\system
```