# Reflected XSS with Some SVG Markup Allowed

## Table of Contents

* [Introduction](#introduction)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Prerequisites](#prerequisites)
* [Attack Flow](#attack-flow)
* [Step-by-Step Walkthrough](#step-by-step-walkthrough)
* [Payload Breakdown](#payload-breakdown)
* [Tools Used](#tools-used)
* [Detection](#detection)
* [Prevention](#prevention)
* [References](#references)
* [Key Takeaways](#key-takeaways)

## Introduction

This lab is a PortSwigger Web Security Academy challenge that shows how a tag filter can be bypassed by finding an allowed element that still supports an event handler. The search feature reflects user input into the page, and a filter blocks most common XSS tags and attributes, but leaves some SVG-related tags untouched. The goal is to find those gaps and build a working payload from them.

## Why This Attack Works

The application relies on a blacklist-style filter to stop script injection. It blocks obvious tags like `<script>` and common event handlers like `onerror` or `onload`, but the filter list is not complete. SVG has its own set of animation elements and events that are far less commonly checked. Because `<animatetransform>` and the `onbegin` event are missed by the filter, an attacker can build a payload using only allowed tags and attributes, and still get JavaScript execution once the browser starts the animation.

This is a classic example of blacklist filtering failing against a large, unusual attack surface. Covering every dangerous tag and event manually is close to impossible once SVG elements are added into the mix.

## Lab Setup

| Item | Detail |
|---|---|
| Lab name | Reflected XSS with some SVG markup allowed |
| Platform | PortSwigger Web Security Academy |
| Vulnerability type | Reflected Cross-Site Scripting (XSS) |
| Injection point | `search` GET parameter |
| Filter type | Tag and attribute blacklist |
| Goal | Trigger `alert(1)` in the browser |

## Prerequisites

* Burp Suite Community Edition (Proxy, Repeater, Intruder)
* Burp's built-in browser
* Basic understanding of SVG elements and event attributes
* PortSwigger XSS Cheat Sheet (used to pull tag and event payload lists)

## Attack Flow

```
[Search Input] 
      |
      v
[Try normal payload: <img src=1 onerror=alert(1)>]
      |
      v
[Blocked - 400 Bad Request, "Tag is not allowed"]
      |
      v
[Send request to Burp Intruder]
      |
      v
[Fuzz tag list from XSS Cheat Sheet]
      |
      v
[Allowed tags found: svg, animatetransform, title, image]
      |
      v
[Fuzz event list inside <svg><animatetransform>]
      |
      v
[Allowed event found: onbegin]
      |
      v
[Build final payload with allowed tag + allowed event]
      |
      v
[alert(1) fires - Lab Solved]
```

## Step-by-Step Walkthrough

### Step 1: Confirm the filter is active

A standard XSS payload is sent through the search parameter:

```
<img src=1 onerror=alert(1)>
```

The server responds with a 400 status and the message "Tag is not allowed," confirming a server-side tag filter is in place before the page even renders.

### Step 2: Fuzz for allowed tags

The request is sent to Burp Intruder. The search value is set to:

```
<>
```

A payload position is placed between the brackets:

```
<§§>
```

The full tag list is copied from the PortSwigger XSS Cheat Sheet and pasted into the Intruder payload list. The attack is run against all of them.

### Step 3: Review tag results

Most tags return 400, but four come back with a 200 response:

* `svg`
* `animatetransform`
* `title`
* `image`

These are the tags the filter fails to catch.

### Step 4: Fuzz for allowed event attributes

Since a bare tag alone does not execute JavaScript, an event handler is needed. The search value is changed to:

```
<svg><animatetransform%20=1>
```

A payload position is placed right before the `=` sign:

```
<svg><animatetransform%20§§=1>
```

The event list from the XSS Cheat Sheet is pasted in, replacing the earlier tag list, and the attack is run again.

### Step 5: Review event results

Every event returns 400 except one:

* `onbegin` → 200

### Step 6: Build and confirm the final payload

Combining the allowed tag, the allowed event, and a valid `attributeName` so the animation is treated as legitimate by the browser:

```
<svg><animatetransform onbegin=alert(1) attributeName=transform>
```

Visiting the URL with this payload in the search parameter triggers the `alert(1)` popup, confirming the lab is solved.

## Payload Breakdown

| Field | Value | Description |
|---|---|---|
| `<svg>` | container tag | Passes the filter, allows nested SVG elements |
| `<animatetransform>` | SVG animation element | One of the few tags not blocked by the filter |
| `onbegin` | event attribute | Fires as soon as the animation starts, bypassing common event blacklists like `onerror`/`onload` |
| `attributeName=transform` | required attribute | Makes the animation valid so the browser actually starts it and fires `onbegin` |
| `alert(1)` | JavaScript payload | Confirms code execution in the browser |

## Tools Used

| Tool | Purpose |
|---|---|
| Burp Suite Community Edition | Intercepting and modifying requests |
| Burp Repeater | Manually testing individual payloads |
| Burp Intruder | Bulk fuzzing tags and event attributes |
| PortSwigger XSS Cheat Sheet | Source of tag and event payload lists |

## Detection

* Monitor for unusual SVG-related parameters in URL query strings, especially combinations of `<svg>`, `animate`, and `on`-prefixed attributes
* Log and alert on 400 responses tied to tag filtering, since a burst of these from one source usually means automated fuzzing is happening
* Watch for encoded angle brackets (`%3C`, `%3E`) paired with SVG element names in request logs

## Prevention

* Do not rely on blacklist filtering for tags and attributes; use a strict whitelist of allowed HTML elements and strip everything else
* Apply context-aware output encoding so any reflected value is treated as text, not as markup, regardless of what tag it resembles
* Set a strong Content Security Policy (CSP) that blocks inline event handlers and restricts script execution sources
* Sanitize SVG input using a dedicated library that understands the full list of SVG animation elements and event attributes, rather than a generic HTML sanitizer

## References

* [PortSwigger Web Security Academy - Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting)
* [PortSwigger XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

## Key Takeaways

Blacklist-based filters break down once the attack surface grows beyond the obvious tags and events. SVG brings in a large set of animation elements and event attributes that most filters never account for, and `<animatetransform>` with `onbegin` is a good example of that gap. Systematic fuzzing with Burp Intruder, rather than guessing payloads one by one, is what makes finding these gaps realistic in a large filter list.