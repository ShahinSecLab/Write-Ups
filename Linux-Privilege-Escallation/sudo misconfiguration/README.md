# Sudo Misconfiguration

**Sudo **Misconfiguration <br>
**Date:** June 2026 <br>
**Author:** ShahinSecLab <br>
**Category:** Privilege Escalation <br>
**Difficulty:** Easy <br>
**Tools:** SSH, sudo, find, GTFOBins 


## Table of Contents

- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [What I Needed Before Starting](#what-i-needed-before-starting)
- [What I Understood During the Process](#what-i-understood-during-the-process)
- [Attack Flow](#attack-flow)
- [Step 1 — Connecting to the Target and Checking Sudo Permissions](#step-1--connecting-to-the-target-and-checking-sudo-permissions)
- [Step 2 — Picking a Binary and Checking GTFOBins](#step-2--picking-a-binary-and-checking-gtfobins)
- [Step 3 — Abusing find with Sudo to Get a Root Shell](#step-3--abusing-find-with-sudo-to-get-a-root-shell)
- [Step 4 — Confirming Full Root Access](#step-4--confirming-full-root-access)
- [How Defenders Can Catch This](#how-defenders-can-catch-this)
- [How to Prevent It](#how-to-prevent-it)
- [What I Achieved](#what-i-achieved)

