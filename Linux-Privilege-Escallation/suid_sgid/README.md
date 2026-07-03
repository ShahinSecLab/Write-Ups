
# SUID/SGID — Privilege Escalation

**Date:** June 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Medium <br>
**Tools:** SSH, find, searchsploit, wget, chmod

# Table of Contents

- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Connecting to the Target via SSH](#step-1--connecting-to-the-target-via-ssh)
- [Step 2 — Finding SUID Binaries](#step-2--finding-suid-binaries)
- [Step 3 — Searching for an Exploit with Searchsploit](#step-3--searching-for-an-exploit-with-searchsploit)
- [Step 4 — Downloading the Exploit to the Target](#step-4--downloading-the-exploit-to-the-target)
- [Step 5 — Running the Exploit and Getting Root](#step-5--running-the-exploit-and-getting-root)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [References](#references)
- [What I Achieved](#what-i-achieved)