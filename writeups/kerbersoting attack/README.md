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