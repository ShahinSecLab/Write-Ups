# ShahinSecLab

Penetration testing portfolio. Web application security, Active Directory attacks, privilege escalation, and network testing. Every writeup here comes from hands-on work in my own lab or on legal practice platforms.

> Items marked `[TODO]` are planned and not done yet. I update this page as each one is finished.

---

## About Me

- Name: Shahin Alam
- Based in Bangladesh
- Focus: web application security, Active Directory attacks, network penetration testing
- GitHub: [ShahinSecLab](https://github.com/ShahinSecLab)
- Contact: [shahinalam12341@gmail.com](mailto:shahinalam12341@gmail.com) | LinkedIn: [Shahin Alam](https://www.linkedin.com/in/shahin-alam-50a98b3a9/)
- Certifications: `[TODO: add any, or write "Working towards ___"]`
- Platforms: Hack The Box `[TODO: profile link]` | TryHackMe `[TODO: profile link]` | PortSwigger Academy `[TODO: profile link]`

## Tools I Use

Burp Suite (Proxy, Repeater, Intruder), Metasploit, Impacket, CrackMapExec / NetExec, Responder, mitm6, Hashcat, Nmap, Kali Linux.

`[TODO: add BloodHound, Certipy, ffuf, sqlmap, etc. once used in a writeup]`

---

## Featured Work

| Project | What it is | Link |
|---|---|---|
| WordPress ACF Frontend Display Exploit | Exploit work on real software, not a lab | [wordpress-acf-frontend-display-exploit](./wordpress-acf-frontend-display-exploit) |
| Full Active Directory attack chain | `[TODO: foothold to Domain Admin in one story]` | `[TODO]` |
| Sample Pentest Report | `[TODO: PDF with exec summary, findings, CVSS, remediation]` | `[TODO]` |

---

## Currently Learning

- TryHackMe: **AI Security** learning path, in progress (currently on Prompt Security)
- I will add writeups here as I finish rooms from this path.

---

## Home Lab

A VMware Workstation Pro 17 lab built from scratch.

| Machine | Role |
|---|---|
| Windows Server 2019 | Domain Controller (`READTEAMBD.local`) |
| Windows 10 | Victim workstations |
| Kali Linux | Attacker machine |
| Metasploitable 2 | Vulnerable Linux target |

`[TODO: add a network diagram and a short setup writeup]`

---

## Writeup Index

### 1. Active Directory

Folder: [Active-Directory](./Active-Directory)

| # | Topic | Status |
|---|---|---|
| 01 | LLMNR Poisoning | Done |
| 02 | SMB Relay | Done |
| 03 | IPv6 Attack with mitm6 | Done |
| 04 | Pass the Hash | Done |
| 05 | Token Impersonation | Done |
| 06 | Kerberoasting | Done |
| 07 | NTDS Dump | Done |
| 08 | Golden Ticket | Done |
| 09 | AS-REP Roasting | `[TODO]` |
| 10 | BloodHound Enumeration | `[TODO]` |
| 11 | DCSync | `[TODO]` |
| 12 | Delegation Attacks (Unconstrained / Constrained / RBCD) | `[TODO]` |
| 13 | ADCS Abuse (ESC1 and more) | `[TODO]` |
| 14 | Lateral Movement | `[TODO]` |
| 15 | Silver Ticket | `[TODO]` |
| 16 | Full chain: Foothold to Domain Admin | `[TODO]` |

### 2. Web Application Security

Folder: [PortSwigger-Labs](./PortSwigger-Labs)

**Authentication** (done)
- 2FA broken logic
- Broken brute-force protection (IP block)
- Broken brute-force protection (multiple credentials per request)
- Brute-forcing a stay-logged-in cookie
- Offline password cracking
- Password brute-force via password change
- Password reset broken logic
- Password reset poisoning via middleware
- Username enumeration via response timing
- Username enumeration via account lock
- Username enumeration via subtly different responses

**SQL Injection** (done)
- UNION attack, retrieving data
- UNION attack, single column
- Blind SQLi with conditional responses
- Blind SQLi with conditional errors
- Blind SQLi with time delays and info retrieval
- Blind SQLi with out-of-band interaction

`[TODO: SQLi login bypass, listing database contents (Oracle / non-Oracle), determining number of columns]`

**Cross-Site Scripting**
- Reflected XSS, HTML context with custom tags (done)
- Reflected XSS, HTML context with most tags and attributes blocked (done)
- Reflected XSS into HTML tag attribute (done)
- Reflected XSS with event handlers and href attributes blocked (done)
- Reflected XSS with some SVG markup allowed (done)
- Stored XSS `[TODO]`
- DOM-based XSS `[TODO]`

**Other web topics (done)**

| Topic | Status |
|---|---|
| CSRF | Done `[TODO: add folder link]` |
| SSRF | Done `[TODO: add folder link]` |
| Access Control / IDOR | Done `[TODO: add folder link]` |
| File Upload Vulnerabilities | Done `[TODO: add folder link]` |
| OS Command Injection | Done `[TODO: add folder link]` |

**Not started yet**

| Topic | Status |
|---|---|
| XXE Injection | `[TODO]` |
| Path Traversal | `[TODO]` |
| JWT Attacks | `[TODO]` |
| Insecure Deserialization | `[TODO]` |
| Business Logic Flaws | `[TODO]` |
| Race Conditions | `[TODO]` |

### 3. Privilege Escalation

**Linux** ([Linux-Privilege-Escalation](./Linux-Privilege-Escalation))
- Kernel exploit (Dirty COW)
- Misconfigured file permissions (writable `/etc/passwd`)
- Sudo misconfiguration
- SUID / SGID
- `[TODO: cron jobs, capabilities, PATH hijacking]`

**Windows** ([Windows-Privilege-Escalation](./Windows-Privilege-Escalation))
- Unquoted service path
- DLL hijacking
- File permission on service
- Insecure service configuration
- Scheduled task manipulation
- `[TODO: AlwaysInstallElevated, token privileges (SeImpersonate / Potato attacks)]`

### 4. Network Penetration Testing

- Metasploitable: vsftpd 2.3.4 backdoor ([Metasploitable/vsftpd-2.3.4-backdoor](./Metasploitable/vsftpd-2.3.4-backdoor))
- `[TODO: full network pentest of the lab, with scanning, enumeration, exploitation, post-exploitation]`

### 5. Full Machine Writeups

Recon to root, start to finish. This is the biggest gap right now.

| Machine | Platform | Difficulty | Status |
|---|---|---|---|
| `[TODO]` | Hack The Box / TryHackMe / Proving Grounds | Easy | `[TODO]` |
| `[TODO]` | | Medium | `[TODO]` |
| `[TODO]` | | Medium | `[TODO]` |

### 6. Research / CVE / Bug Bounty

| Item | Details | Link |
|---|---|---|
| WordPress ACF Frontend Display Exploit | Real software, exploit and writeup | [View](./wordpress-acf-frontend-display-exploit) |
| `[TODO: CVE ID]` | `[TODO]` | `[TODO]` |
| `[TODO: bug bounty finding]` | `[TODO]` | `[TODO]` |

### 7. Reports

| Report | Scope | Link |
|---|---|---|
| Sample internal pentest report | READTEAMBD.local lab | `[TODO]` |
| Sample web application pentest report | `[TODO: DVWA / Juice Shop / own lab]` | `[TODO]` |

Each report will have: executive summary, scope, methodology, findings with CVSS scores, evidence, remediation, and an appendix.

---

## How Each Writeup Is Structured

Every writeup follows the same format:

1. Table of Contents
2. Introduction
3. Why This Attack Works
4. Lab Setup (table)
5. Prerequisites
6. Attack Flow (ASCII diagram)
7. Step-by-Step Walkthrough (with flag breakdown tables)
8. Detection
9. Prevention
10. MITRE ATT&CK mapping `[TODO: add to old writeups]`
11. References
12. Key Takeaways

---

## Repository Layout

```
ShahinSecLab/
├── README.md
├── Active-Directory/
├── PortSwigger-Labs/
│   ├── Authentication/
│   ├── SQL-Injection/
│   └── XSS/
├── Linux-Privilege-Escalation/
├── Windows-Privilege-Escalation/
├── Metasploitable/
├── Full-Machines/              [TODO]
├── Research-CVE/               [TODO: move wordpress-acf-frontend-display-exploit here]
├── Reports/                    [TODO]
└── Home-Lab/                   [TODO]
```

---

## Disclaimer

Everything in this repository was done in my own lab or on platforms that allow testing. Nothing here is meant for use against systems without written permission.