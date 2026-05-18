# LLMNR Poisoning Attack

**Date:** May 2026  
**Author:** ShahinSecLab  
**Category:** Network Attack / Credential Capture  
**Difficulty:** Easy  
**Tools:** Responder, Hashcat  

---

## Table of Contents
1. [What is LLMNR?](#what-is-llmnr)
2. [How the Attack Works](#how-the-attack-works)
3. [Lab Setup](#lab-setup)
4. [Attack Steps](#attack-steps)
5. [Defense & Mitigation](#defense--mitigation)
6. [Key Takeaways](#key-takeaways)
7. [References](#references)

---

## What is LLMNR?

LLMNR stands for **Link-Local Multicast Name Resolution.**

Windows uses this when it cannot find a computer name through DNS. When DNS fails, Windows shouts to the whole network:

> *"Hey, does anyone know where \\fileserver is?"*

The problem is — any machine on the network can reply. So an attacker can say *"Yeah, that's me!"* and the victim sends their password hash without knowing.

---

## Protocol Order (Windows Name Resolution)

- **DNS** → tries first  
- **mDNS** → tries second  
- **LLMNR** → tries third (can be abused)  
- **NBT-NS** → tries last (can also be abused)  

## How the Attack Works

## LLMNR Poisoning Flow
Victim Network Attacker
| | |
|--- DNS request ----------->| |
|<-- DNS: "I don't know" ----| |
| | |
|--- LLMNR broadcast ------->| |
| "Who knows fileserver?" |<-- Responder listening ---|
| | |
|<-- "I know! I am him!" ----|---------------------------|
| | |
|--- NTLMv2 Hash ----------->|-------------------------->|
| (credentials sent) | |
Attacker captures
NTLMv2 hash
