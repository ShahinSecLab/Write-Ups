# IPv6 Attack with mitm6

## Table of Contents
1. [Overview](#overview)
2. [Attack Theory](#attack-theory)
3. [Lab Environment](#lab-environment)
4. [Tools Required](#tools-required)
5. [Attack Walkthrough](#attack-walkthrough)
   - [Step 1 — Start mitm6](#step-1--start-mitm6)
   - [Step 2 — Launch NTLM Relay](#step-2--launch-ntlm-relay)
   - [Step 3 — Trigger Authentication from Victim](#step-3--trigger-authentication-from-victim)
   - [Step 4 — Relay Succeeds & Loot Retrieved](#step-4--relay-succeeds--loot-retrieved)
   - [Step 5 — Browse SMB Shares on Victim](#step-5--browse-smb-shares-on-victim)
   - [Step 6 — Explore Loot Folder](#step-6--explore-loot-folder)
   - [Step 7 — Credential & Domain Dump](#step-7--credential--domain-dump)
6. [Understanding the Output](#understanding-the-output)
7. [Mitigations & Defenses](#mitigations--defenses)
8. [Key Takeaways](#key-takeaways)
9. [References](#references)

## Overview

`mitm6` is a tool that exploits a fundamental design decision in Windows networking: **IPv6 is preferred over IPv4 by default on modern Windows systems**.

In many corporate environments, IPv6 is enabled but not properly configured. In most cases, **no legitimate DHCPv6 or DNSv6 server exists**, which creates a security gap. Attackers can abuse this gap to position themselves as the **authoritative DNS server** for an entire network segment.

When combined with Impacket’s `ntlmrelayx`, this technique becomes a powerful Active Directory attack chain:

- Capture NTLM authentication from Windows machines on the network
- Relay captured authentication to services such as **LDAP/LDAPS** on a Domain Controller
- Create new domain users or extract Active Directory data
- Relay to SMB services to access file shares or execute commands
- Ultimately escalate privileges and potentially achieve **Domain Admin access without cracking passwords**

This attack technique was originally researched and published by security researchers at :contentReference[oaicite:0]{index=0}, including :contentReference[oaicite:1]{index=1}. It remains one of the most impactful internal network attack methods against Active Directory environments.

## Attack Theory
Why IPv6?
Windows machines periodically broadcast DHCPv6 Solicit messages looking for an IPv6 gateway and DNS server. By default, these go unanswered on most enterprise networks. mitm6 responds to these broadcasts, assigning the attacker's link-local IPv6 address as the DNS server.

```
Victim  → DNS: "Where is wpad.corp.local?"
mitm6   → DNS: "It's at attacker's IP"
Victim  → HTTP: GET http://attacker/wpad.dat
ntlmrelayx → "Authenticate with NTLM!"
Victim  → [Sends NTLM handshake]
ntlmrelayx → [Relays to DC's LDAP]
DC      → [Authenticated!]
```

## Full Kill Chain
```
DHCPv6 Spoofing
      ↓
DNS Hijack (WPAD / internal names)
      ↓
NTLM Authentication Capture
      ↓
NTLM Relay → LDAP / SMB on Domain Controller
      ↓
Create Domain User / Dump AD / Execute Commands
      ↓
Domain Compromise
```