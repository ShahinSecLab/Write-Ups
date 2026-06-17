# Insecure Service Configuration

Date: June 2026
Author: ShahinSecLab
Category: Privilege Escalation
Difficulty: Easy
Tools: msfvenom, Metasploit, winPEAS, accesschk.exe, certutil

## Table of Contents

- [Introduction](#Introduction)
- [Why this attack works](#why-this-attack-works)
- [Attack flow](#attack-flow)
- [What I needed before starting](#what-i-needed-before-starting)
- [What I understood during the process](#what-i-understood-during-the-process)

- [Step 1 — Generating a Malicious Payload with msfvenom](#step-1--generating-a-malicious-payload-with-msfvenom)
- [Step 2 — Setting Up Metasploit Listener and HTTP Server](#step-2--setting-up-metasploit-listener-and-http-server)
- [Step 3 — Downloading the Payload on the Victim Machine](#step-3--downloading-the-payload-on-the-victim-machine)
- [Step 4 — Transferring the Payload to PrivEsc Folder](#step-4--transferring-the-payload-to-privesc-folder)
- [Step 5 — Running the Payload and Getting a Meterpreter Shell](#step-5--running-the-payload-and-getting-a-meterpreter-shell)
- [Step 6 — Enumerating the Victim and Uploading winPEAS](#step-6--enumerating-the-victim-and-uploading-winpeas)
- [Step 7 — Running winPEAS to Find Privilege Escalation Paths](#step-7--running-winpeas-to-find-privilege-escalation-paths)
- [Step 8 — Verifying Service Permissions with accesschk.exe](#step-8--verifying-service-permissions-with-accesschkexe)
- [Step 9 — Checking the Service Configuration](#step-9--checking-the-service-configuration)
- [Step 10 — Generating a New Payload and Starting a Second Listener](#step-10--generating-a-new-payload-and-starting-a-second-listener)
- [Step 11 — Uploading the New Payload and Changing the Service Binary Path](#step-11--uploading-the-new-payload-and-changing-the-service-binary-path)
- [Step 12 — Starting the Service and Getting a SYSTEM Shell](#step-12--starting-the-service-and-getting-a-system-shell)

- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [What I Achieved](#what-i-achieved)

## Introduction

Insecure Service Configuration is a local privilege escalation technique. When a Windows service is set up with weak permissions, a low privilege user can modify how that service runs. If the service runs as SYSTEM, I can abuse those weak permissions to run my own code as SYSTEM and take full control of the machine — without any exploit or CVE.

## Why This Attack Works

Windows services often run as SYSTEM — the highest privilege account on the machine. If the service permissions are weak enough for a normal user to modify, I can change the binary path to point to my own malicious executable. When the service restarts, it runs my payload as SYSTEM.

## Attack Flow

```
Generated malicious payload (reverse.exe) on Kali with msfvenom
                              ↓
Started Metasploit listener on port 4444
                              ↓
Hosted payload over HTTP with Python HTTP server
                              ↓
Downloaded reverse.exe on victim machine via browser
                              ↓
Transferred payload to C:\PrivEsc using certutil
                              ↓
Ran reverse.exe on victim machine
                              ↓
Got Meterpreter shell on Kali as low privilege user (MSEDGEWIN10\user)
                              ↓
Uploaded winPEAS and ran it to find privilege escalation paths
                              ↓
winPEAS flagged daclsvc service as vulnerable
                              ↓
Verified with accesschk.exe — found SERVICE_CHANGE_CONFIG permission
                              ↓
Checked service config — daclsvc runs as LocalSystem (SYSTEM)
                              ↓
Generated second payload (privesc.exe) on port 9001
                              ↓
Uploaded privesc.exe to C:\PrivEsc on victim
                              ↓
Changed daclsvc binary path to C:\PrivEsc\privesc.exe
                              ↓
Started daclsvc service
                              ↓
Metasploit caught SYSTEM shell on port 9001
                              ↓
whoami → nt authority\system
```