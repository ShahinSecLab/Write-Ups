# Pass The Hash

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Responder, Hashcat 

## Table of Contents

- [What's the Point](#whats-the-point)
- [How NTLM Auth Actually Works](#how-ntlm-auth-actually-works)
- [So What is Pass the Hash](#so-what-is-pass-the-hash)
- [What You Need Before Starting](#what-you-need-before-starting)
- [Attack Walkthrough](#attack-walkthrough)
  - [Step 1: Get a Shell](#step-1-get-a-shell)
  - [Step 2: Pull the Hashes](#step-2-pull-the-hashes)
  - [Step 3: Use the Hash to Log In](#step-3-use-the-hash-to-log-in)
  - [Step 4: Move Around the Network](#step-4-move-around-the-network)
- [Tools](#tools)
- [How to Stop This Attack](#how-to-stop-this-attack)
- [How to Catch It Happening](#how-to-catch-it-happening)
- [References](#references)

## Overview

Pass the Hash is one of those attacks that sounds complicated but is actually pretty straightforward once you get it. The short version: Windows stores your password as a hash, and when you authenticate over the network, it uses that hash — not your actual password. So if you steal someone's hash, you can log in as them without ever knowing their password.
No cracking. No brute forcing. Just grab the hash and use it.
It's been around since the late 90s and it's still one of the go-to moves for moving sideways through a Windows domain. Once you're on one machine, PtH can get you to every other machine where that account has admin rights.