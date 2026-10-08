# Brute-Forcing a Stay-Logged-In Cookie

**Category:** Authentication
**Difficulty:** Practitioner
**Status:** Solved ✅

---

## Table of Contents

* Introduction
* Why This Attack Works
* Lab Setup
* Prerequisites
* Attack Flow
* Step-by-Step Walkthrough
   * Step 1: Login Page Recon
   * Step 2: Capturing the Stay-Logged-In Cookie
   * Step 3: Decoding the Cookie Structure
   * Step 4: Setting Up Burp Intruder
   * Step 5: Configuring Payload Processing Rules
   * Step 6: Running the Attack and Spotting the Valid Cookie
   * Step 7: Logging In With the Forged Cookie
* Detection
* Prevention
* References
* Key Takeaways

---

## Introduction

This lab has a "Stay logged in" checkbox on the login page. Ticking it and logging in sets a cookie that keeps the session alive even after closing the browser. The problem is that this cookie value follows a predictable format, and the brute-force protection applied to the normal login form does not apply to this cookie mechanism. That opens the door to brute-forcing the cookie and taking over any account.

The goal was to:

Find how the stay-logged-in cookie is built.
Decode the cookie to confirm the username:hash format.
Brute-force the password through the cookie value.
Log in to Carlos's account.

## Why This Attack Works

A stay-logged-in cookie is usually built from a fixed formula, something like `base64(username:md5(password))`. Once that formula is known, a password list can be hashed with MD5, joined with the username, base64 encoded, and sent as the cookie. The server checks this cookie on every request, but there is no rate limiting or lockout on that check. The login form locks an account after a few wrong attempts, but the cookie path has no such protection. That gap is the core weakness.

## Lab Setup

| Field | Value |
|---|---|
| Target | PortSwigger Web Security Academy |
| Lab Name | Brute-forcing a stay-logged-in cookie |
| Target Username | carlos |
| Vulnerable Mechanism | "Stay logged in" persistent cookie |
| Tool Used | Burp Suite Community Edition (Proxy, Intruder) |
| Wordlist | PortSwigger candidate passwords list |

## Prerequisites

* Burp Suite running with browser proxy configured
* PortSwigger's candidate passwords list downloaded
* Basic understanding of Burp Intruder payload processing rules (Hash, Add Prefix, Encode)

## Attack Flow

```
[Login Page] --check "Stay logged in"--> [Valid Login]
       |
       v
[Cookie Set: stay-logged-in=base64(carlos:md5(password))]
       |
       v
[Intercept request with cookie in Burp]
       |
       v
[Send to Intruder, mark cookie value as payload position]
       |
       v
[Payload Processing: Hash(MD5) -> Add Prefix "carlos:" -> Encode Base64]
       |
       v
[Run attack against candidate passwords]
       |
       v
[Check response length/status for successful login]
       |
       v
[Replace cookie with winning value -> Access /my-account as carlos]
```

## Step-by-Step Walkthrough

### Step 1: Login Page Recon

First I checked the login page and found a "Stay logged in" checkbox. I ticked it and logged in with the lab's own test account to see what cookie gets set.

| Field | Value | Description |
|---|---|---|
| URL | /login | Login endpoint of the lab |
| Checkbox | Stay logged in | Enables persistent cookie |
| Result | New cookie set | `stay-logged-in` cookie appears after login |

### Step 2: Capturing the Stay-Logged-In Cookie

I checked Burp Proxy history and found a `Set-Cookie: stay-logged-in=...` header in the login response. The value is a long base64-looking string.

| Field | Value | Description |
|---|---|---|
| Cookie Name | stay-logged-in | Persistent session cookie |
| Raw Value | (base64 string) | Captured from Set-Cookie header |
| Transport | Cookie header on every request | Sent with each subsequent request |

### Step 3: Decoding the Cookie Structure

Decoding the cookie value from base64 reveals a `username:hash` format, where the hash part is the MD5 of the password. Once decoded, the structure comes out as:

```
base64("carlos:" + md5(password))
```

| Field | Value | Description |
|---|---|---|
| Decoded Format | username:md5(password) | Structure revealed by decoding |
| Username Confirmed | carlos | Target account for the lab |
| Hash Algorithm | MD5 | Used to hash the password portion |

### Step 4: Setting Up Burp Intruder

I sent the login request to Intruder and marked the entire `stay-logged-in` cookie value as the payload position, so only that value changes on each attempt.

| Field | Value | Description |
|---|---|---|
| Attack Type | Sniper | Single payload position attack |
| Payload Position | stay-logged-in cookie value | Full cookie value marked §payload§ |
| Payload Type | Simple list | Candidate passwords from wordlist |

### Step 5: Configuring Payload Processing Rules

This part carries the real work. Three rules go into the Payload Processing tab, and the order matters:

1. **Hash** → MD5 (turns the raw password into its MD5 digest)
2. **Add Prefix** → `carlos:` (joins the username and colon in front)
3. **Encode** → Base64-encode (converts the whole string into the final cookie format)

| Field | Value | Description |
|---|---|---|
| Rule 1 | Hash: MD5 | Converts raw password to MD5 digest |
| Rule 2 | Add Prefix: carlos: | Prepends username and colon |
| Rule 3 | Encode: Base64 | Produces final cookie-ready string |

### Step 6: Running the Attack and Spotting the Valid Cookie

After starting the attack, I checked response length and status code for each attempt. Most wrong passwords gave the same response length, but one entry stood out with a different length and a 302 redirect, which points to a successful login.

| Field | Value | Description |
|---|---|---|
| Indicator | Response length / status 302 | Marks the successful attempt |
| Result | One payload stands out | Different length than the rest |
| Password Found | Revealed in payload column | Original candidate password for that row |

### Step 7: Logging In With the Forged Cookie

Once the right password turned up, I built the same cookie value using the same processing chain and set it as the `stay-logged-in` cookie in the browser, then refreshed `/my-account`. The page loaded as carlos's account and the lab marked itself solved.

| Field | Value | Description |
|---|---|---|
| Final Step | Replace cookie in browser | Set the winning base64 value manually |
| Access Gained | /my-account as carlos | Confirms account takeover |
| Lab Status | Solved | Objective completed |

## Detection

* Many different values sent under the same cookie name in a short window (brute-force pattern)
* An unusual rate of failed authentication attempts arriving only through the cookie header, bypassing the login form
* A large number of distinct `stay-logged-in` cookie values coming from the same source IP

## Prevention

* Avoid predictable hashes inside persistent cookies; use a cryptographically random, server-side session token instead
* Apply the same rate limiting and account lockout to stay-logged-in cookie verification that the login form already has
* Replace MD5 with a salted, slow hashing algorithm (bcrypt, Argon2) to make both offline and online brute-force harder
* Keep no reversible or predictable user data inside the cookie value

## References

* PortSwigger Web Security Academy — Authentication vulnerabilities
* OWASP — Session Management Cheat Sheet

## Key Takeaways

* "Remember me" or "stay logged in" features often skip the security rules applied to the main login form, which creates a hidden attack surface
* Once the cookie structure is known (base64 plus a known hash algorithm), the whole authentication mechanism opens up to brute-forcing
* Chaining Burp Intruder's payload processing rules (Hash → Prefix → Encode) makes it possible to replicate a complex cookie format
* Rate limiting on the login form alone is not enough; every entry point into authentication needs the same coverage