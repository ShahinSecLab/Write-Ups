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
- [Step 2: Start Responder](#step-2-start-responder)
- [Step 3: Create Target List](#step-3-create-target-list)
- [Step 4: Start ntlmrelayx](#step-4-start-ntlmrelayx)
- [Step 5: Trigger Authentication](#step-5-trigger-authentication)
- [Step 6: Successful Relay](#step-6-successful-relay)
- [Result](#result)
- [Prevention](#prevention)

## Introduction

SMB Relay is a network attack where an attacker captures a user's authentication request and forwards it to victim machine instead of cracking the password. If the target system accepts the authentication, the attacker can gain access using the victim's session.

# Lab Setup
```
| Machine  | Operating System                 | Role           |
| -------- | -------------------------------- | -------------- |
| Attacker | Kali Linux                       | Attack machine |
| Victim   | Windows 10                       | User machine   |
| Target   | Windows Server / Windows Machine | Target machine |
```

# Attack Flow

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

# Step 1: Disable SMB and HTTP in Responder

### Open the Responder configuration file:

```bash
sudo nano /etc/responder/Responder.conf
```
### Find these lines and change them:

```bash
SMB = Off
HTTP = Off
```

### Save and exit:

```bash
CTRL + X
Y
ENTER
```
<p align="center">
  <img src="/writeups/02-smb-relay/images/step1.png" width="600">
</p>

# Step 2: Start Responder