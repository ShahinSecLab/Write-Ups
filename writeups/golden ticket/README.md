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

# Step 1 — Getting a Shell on the Domain Controller

First I logged into the Domain Controller using the Administrator hash I dumped earlier from the NTDS.dit file.