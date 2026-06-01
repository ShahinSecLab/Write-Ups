# token impersonation attack

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Meterpreter, Incognito, Windows Access Tokens

## Table of Contents

- [Introduction](#introduction)
- [Lab Setup](#lab-setup)
- [Attack Flow](#attack-flow)
- [Step 1 - Get a Meterpreter Session](#step-1---get-a-meterpreter-session)
- [Step 2 - Load Incognito](#step-2---load-incognito)
- [Step 3 - List Available Tokens](#step-3---list-available-tokens)
- [Step 4 - Impersonate the Administrator Token](#step-4---impersonate-the-administrator-token)
- [Step 5 - Verify Access](#step-5---verify-access)
- [Why It Worked](#why-it-worked)
- [Detection](#detection)
- [Mitigation](#mitigation)
- [Key Takeaways](#key-takeaways)
- [References](#references)
- [Disclaimer](#disclaimer)

# Introduction

In this lab, I practiced a Windows Token Impersonation attack.

Windows creates an access token whenever a user logs in. The token contains information about that user's permissions and privileges.

If a privileged token is available on a compromised machine, it may be possible to impersonate that token and perform actions as that user.

The goal of this lab was to identify available tokens and impersonate a privileged account from an existing Meterpreter session.

# lab Setup

```
| Machine             | Role              |   Ip          |
| ------------------- | ----------------- |---------------|
| Kali Linux          | Attacker          | 192.168.5.128 |
| Windows 10          | Victim            | 192.168.5.135 |
| Windows Server 2019 | Domain Controller | 192.168.5.134 |
```

# Attack Flow

```
Gain Initial Access
        │
        ▼
List Available Tokens
        │
        ▼
Find Administrator Token
        │
        ▼
Impersonate Token
        │
        ▼
Gain Elevated Privileges
```

# Step 1 - Get a Meterpreter Session

### Starting Metasploit

### Step 1.1: Run the following command to start the Metasploit console:

```bash
msfconsole -q
```

**What it does:**

* `msfconsole` starts the Metasploit Framework console.
* The `-q` flag starts Metasploit in quiet mode.
* This hides the startup banner and extra messages, giving a cleaner terminal output.

**Output:**

```text
msf6 >
```
<p align="center">
  <img src="/writeups/token impersonation attack/images/step1 1.1.png" width="600">
</p>

The quiet mode does not change how Metasploit works. It only reduces the amount of information displayed when the console starts.


### Step 1.2: Now search for `psexec`

```bash
search psexec
```

It searches the Metasploit database for all available modules related to psexec.

**Output:**

```
Matching Modules
================

   #   Name                                         Disclosure Date  Rank       Check  Description
   -   ----                                         ---------------  ----       -----  -----------
   0   auxiliary/scanner/smb/impacket/dcomexec      2018-03-19       normal     No     DCOM Exec
   1   exploit/windows/smb/smb_relay                2001-03-31       excellent  Yes    MS08-068 Microsoft Windows SMB Relay Code Execution
   2     \_ action: CREATE_SMB_SESSION              .                .          .      Do not close the SMB connection after relaying, and instead create an SMB session
   3     \_ action: PSEXEC                          .                .          .      Use the SMB Connection to run the exploit/windows/psexec module against the relay target
   4     \_ target: Automatic                       .                .          .      .
   5     \_ target: PowerShell                      .                .          .      .
   6     \_ target: Native upload                   .                .          .      .
   7     \_ target: MOF upload                      .                .          .      .
   8     \_ target: Command                         .                .          .      .
   9   exploit/windows/smb/ms17_010_psexec          2017-03-14       normal     Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   10    \_ target: Automatic                       .                .          .      .
   11    \_ target: PowerShell                      .                .          .      .
   12    \_ target: Native upload                   .                .          .      .
   13    \_ target: MOF upload                      .                .          .      .
   14    \_ AKA: ETERNALSYNERGY                     .                .          .      .
   15    \_ AKA: ETERNALROMANCE                     .                .          .      .
   16    \_ AKA: ETERNALCHAMPION                    .                .          .      .
   17    \_ AKA: ETERNALBLUE                        .                .          .      .
   18  auxiliary/admin/smb/ms17_010_command         2017-03-14       normal     No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   19    \_ AKA: ETERNALSYNERGY                     .                .          .      .
   20    \_ AKA: ETERNALROMANCE                     .                .          .      .
   21    \_ AKA: ETERNALCHAMPION                    .                .          .      .
   22    \_ AKA: ETERNALBLUE                        .                .          .      .
   23  auxiliary/scanner/smb/psexec_loggedin_users  .                normal     No     Microsoft Windows Authenticated Logged In Users Enumeration
   24  exploit/windows/smb/psexec                   1999-01-01       manual     No     Microsoft Windows Authenticated User Code Execution
   25    \_ target: Automatic                       .                .          .      .
   26    \_ target: PowerShell                      .                .          .      .
   27    \_ target: Native upload                   .                .          .      .
   28    \_ target: MOF upload                      .                .          .      .
   29    \_ target: Command                         .                .          .      .
   30  auxiliary/admin/smb/psexec_ntdsgrab          .                normal     No     PsExec NTDS.dit And SYSTEM Hive Download Utility
   31  exploit/windows/local/current_user_psexec    1999-01-01       excellent  No     PsExec via Current User Token
   32  encoder/x86/service                          .                manual     No     Register Service
   33  auxiliary/scanner/smb/impacket/wmiexec       2018-03-19       normal     No     WMI Exec
   34  exploit/windows/smb/webexec                  2018-10-24       manual     No     WebExec Authenticated User Code Execution
   35    \_ target: Automatic                       .                .          .      .
   36    \_ target: Native upload                   .                .          .      .
   37  exploit/windows/local/wmi                    1999-01-01       excellent  No     Windows Management Instrumentation (WMI) Remote Command Execution
   ```

































# Key Takeaways

- Windows creates an access token when a user logs in.
- Access tokens contain the user's permissions.
- A privileged token can sometimes be reused without knowing the user's password.
- Token impersonation is a common post-exploitation technique.
- Monitoring privileged activity can help detect this behavior.

# Mitigation

- Follow the principle of least privilege
- Limit use of privileged accounts
- Review accounts with SeImpersonatePrivilege
- Enable Credential Guard
- Monitor suspicious process activity