# Blind SQL Injection with Out-of-Band Interaction (Oracle XXE Technique)

**Date:** Aug 2026<br>
**Author:** ShahinSecLab<br>
**Category:** SQL Injection<br>
**Vulnerability:** Blind SQL Injection<br>
**Difficulty:** Easy<br>
**Platform:** PortSwigger Web Security Academy<br>
**Database:** PostgreSQL<br>
**Tools:** Burp Suite Community Edition, Firefox

# Table of Contents

* [Introduction](#introduction)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Prerequisites](#prerequisites)
* [Attack Flow](#attack-flow)
  * [Step 1: Intercept the Request](#step-1-intercept-the-request)
  * [Step 2: Confirm the Out-of-Band Interaction](#step-2-confirm-the-out-of-band-interaction)
  * [Step 3: Extract the Administrator Password](#step-3-extract-the-administrator-password)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Attack Flow
```
Browser --> Burp Proxy --> Intercept request
            |
            v
Modify TrackingId cookie
            |
            v
Inject UNION SELECT EXTRACTVALUE(...)
            |
            v
Oracle parses malicious XML inside EXTRACTVALUE
            |
            v
XML parser resolves external entity (XXE)
            |
            v
Oracle sends DNS/HTTP request to Collaborator
            |
            v
Burp Collaborator logs the interaction
            |
            v
Confirm injection, then exfiltrate data (e.g. password)
by embedding it in the subdomain
```

## Introduction

This writeup covers a blind SQL injection case where the application does not show any error, does not change its response, and does not delay its response either. In short, there is no visible feedback at all. This type of injection is often the hardest to confirm, because none of the usual blind techniques (boolean-based or time-based) give a clear signal.

To confirm and exploit this kind of injection, an out-of-band (OOB) channel is used instead. The database itself is forced to make a network call to a server controlled by the attacker. Burp Collaborator is used to catch that call. Once the channel is confirmed, the same channel is used to pull real data out of the database, such as a password, by hiding it inside the network request the database sends out.

The database backend in this lab is Oracle, and the injection is combined with an XML External Entity (XXE) trick through Oracle's EXTRACTVALUE() function.

## Why This Attack Works

The application takes the TrackingId cookie value and places it directly into a SQL query without proper sanitization or parameterization. Because the query result is never reflected back to the user in any form, normal blind techniques fail here.

Oracle has a function called EXTRACTVALUE(), which is meant to pull a value out of an XML document using an XPath expression. This function relies on Oracle's XML parser internally. If the XML data passed into it contains a malicious DOCTYPE with an external entity, the parser will try to resolve that entity by making an outbound network request, for example a DNS lookup or an HTTP request.

This behavior is not part of the SQL injection itself. It is a separate flaw (XXE) sitting inside a database function, and it gets triggered as a side effect of the injected query. Combining the two turns a completely blind, silent injection point into something that can be confirmed and abused externally.

## Lab Setup

| Component | Details |
|---|---|
| Attacker Machine | Kali Linux |
| Target | PortSwigger Web Security Academy lab (Blind SQL injection with out-of-band interaction)|
| Tool Used |Burp Suite (Proxy, Repeater), Burp Collaborator|
| Database | Oracle |
| Injection Point | `TrackingId` cookie |

## Tools Used

| Tool | Purpose |
|---|---|
| Burp Suite (Proxy) | Intercept requests and read/modify the `TrackingId` cookie |
| Burp Suite (Repeater) | Manually test injection payloads and read error vs. no-error responses |
| Burp Collaborator|  Receive DNS/HTTP requests from the database to confirm the SQL injection and capture exfiltrated data|
| Web Browser | Access the lab front page and login page |

## Prerequisites

- Basic understanding of SQL injection (UNION-based)
- Basic understanding of XML structure (DOCTYPE, ENTITY)
- Burp Suite configured as browser proxy
- Access to Burp Collaborator (built into Burp Suite Professional)







## How Defenders Can Catch This

| Signal | What to Look For |
|--------|-------------------|
| **Outbound DNS/HTTP from database server** | Database servers rarely need to make outbound calls to arbitrary internet hosts. Unexpected DNS or HTTP requests from the database server can indicate an out-of-band (OOB) attack. |
| **Unusual XML function usage** | Database query logs containing functions such as `EXTRACTVALUE()` or `XMLType`, especially when combined with user-controlled input. |
| **Long or malformed cookie values** | Cookie values (such as `TrackingId`) containing SQL keywords, XML tags, `DOCTYPE` declarations, or other unusual characters. |
| **Network egress monitoring** | Firewall, proxy, or DNS logs showing requests to unfamiliar or randomly generated subdomains, such as Burp Collaborator domains. |

## How to Prevent It

- Use parameterized queries (prepared statements) for all database interactions, never concatenate user input into SQL strings
- Disable external entity resolution in the database's XML parser where possible
- Block or restrict outbound network access from the database server, since a database should rarely need to reach the internet directly
- Apply strict input validation on cookie values before they are used in any backend logic
- Keep database software patched, since some of these XML functions have had specific security fixes over time

## References

- [PortSwigger Web Security Academy: Blind SQL injection with out-of-band interaction](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band)
- [PortSwigger SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [PortSwigger Web Security Academy: XML External Entity (XXE) injection](https://portswigger.net/web-security/xxe)

## Lessons Learned

- A silent, fully blind injection point can still be confirmed using an out-of-band channel when no other feedback is available
- Oracle's EXTRACTVALUE() function can be abused for XXE because it processes XML internally
- Data does not need to be shown in a response to be stolen; it can be smuggled out through a DNS lookup by embedding it in a subdomain
- Chaining two separate flaws (SQL injection and XXE) together can turn an unexploitable-looking bug into a full data exfiltration path