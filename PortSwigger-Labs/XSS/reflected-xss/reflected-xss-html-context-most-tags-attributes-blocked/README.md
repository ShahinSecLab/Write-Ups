# Reflected XSS with Some SVG Markup Allowed — WAF Bypass Using body/onresize

**Date:** August 2026<br>
**Author:** ShahinSecLab<br>
**Category:** Cross-Site Scripting (XSS)<br>
**Vulnerability:** Reflected XSS with WAF Bypass<br>
**Difficulty:** Practitioner<br>
**Platform:** PortSwigger Web Security Academy<br>
**Database:** N/A<br>
**Tools:** Burp Suite Community Edition, Firefox

## Table of Contents

* Introduction
* Why This Attack Works
* Lab Setup
* Prerequisites
* Attack Flow
* Step-by-Step Walkthrough
   * Step 1: Confirm the filter is blocking the standard payload
   * Step 2: Find which tags are allowed
   * Step 3: Find which event attributes are allowed
   * Step 4: Build the working exploit
   * Step 5: Deliver the exploit
* Detection
* Prevention
* References
* Key Takeaways

## Introduction

This lab has a WAF sitting in front of the search feature. A normal XSS payload like an img tag with onerror gets blocked right away. Instead of guessing payloads one by one, this writeup shows how to use Burp Intruder to test every tag and every event attribute from the XSS cheat sheet against the filter, find the ones that slip through, and combine them into a working exploit.

## Why This Attack Works

WAFs that rely on blacklists cannot realistically block every single tag and every single event attribute that exists in HTML. Some combination almost always gets missed. In this case the body tag itself is not blocked, and neither is the onresize event handler. The catch is that onresize only fires when the element actually gets resized, so a plain payload sitting on the page will never trigger on its own. Wrapping the payload inside an iframe and then shrinking that iframe with JavaScript forces a resize event, which fires the payload without needing any user interaction.

## Lab Setup

* Target: PortSwigger Web Security Academy lab (Reflected XSS with some SVG markup allowed)
* Browser: Burp's built-in Chromium browser
* Tooling: Burp Suite Community Edition (Proxy, Intruder, Repeater)
* Exploit delivery: PortSwigger exploit server

## Tools Used

* Burp Suite Community Edition — Proxy, Repeater, Intruder
* Firefox — manual testing and viewing reflected responses
* Burp's built-in Chromium browser — used for the search feature and sending requests to Intruder
* PortSwigger XSS Cheat Sheet — tag list and event attribute list for fuzzing
* PortSwigger Exploit Server — hosting and delivering the final payload

## Prerequisites

* Basic understanding of reflected XSS
* Familiarity with Burp Intruder payload positions (the § markers)
* Access to the PortSwigger XSS cheat sheet (Burp's built-in one under the Extensions/Learn tab, or the online version)

## Attack Flow

```
[Search feature] --> [Payload blocked, 400 response]
        |
        v
[Send request to Intruder] --> [Fuzz tag names using <§§>]
        |
        v
[body tag returns 200, rest return 400]
        |
        v
[Fuzz event attributes using <body%20§§=1>]
        |
        v
[onresize returns 200, rest return 400]
        |
        v
[Build payload: "><body onresize=print()>]
        |
        v
[Wrap in iframe, force resize via onload]
        |
        v
[Host on exploit server, deliver to victim]
        |
        v
[Victim's browser resizes iframe body --> onresize fires --> print() runs]
```

## Step-by-Step Walkthrough

### Step 1: Confirm the filter is blocking the standard payload

Try a normal payload in the search box:

```
<img src=1 onerror=print()>
```

The server responds with a 400, confirming there is a filter blocking known XSS patterns.

### Step 2: Find which tags are allowed

Send the search request to Burp Intruder. Change the search value to:

```
<>
```

Place the cursor between the two angle brackets and add a payload position, so it looks like:

```
<§§>
```

Open the XSS cheat sheet, copy the full list of tags, and paste it into the Intruder payload list. Run the attack.

| Field | Value | Description |
|---|---|---|
| Payload position | Between `<` and `>` | Tests each tag name from the cheat sheet in this spot |
| Payload set | List of tags | Every HTML tag known to be usable in XSS |
| Expected result | 200 for allowed tags | Anything not caught by the filter |

Most tags return 400. The body tag returns 200, meaning it passes through the filter untouched.

### Step 3: Find which event attributes are allowed

Go back to Intruder and change the search value to:

```
<body%20=1>
```

Place the cursor right before the `=` and add a payload position:

```
<body%20§§=1>
```

Clear the old payload list, copy the events list from the cheat sheet, paste it in, and run the attack.

| Field | Value | Description |
|---|---|---|
| Payload position | Before `=` in the body tag | Tests each event attribute name |
| Payload set | List of event handlers | onload, onerror, onresize, and so on |
| Expected result | 200 for allowed attribute | Anything the filter does not catch |

A good number of attributes return 200 instead of 400, including ones like ontouchcancel, onsuspend, onslotchange, onsearch, onsecuritypolicyviolation, onscrollsnapchanging, onscrollend, onratechange, onpromptdismiss, and onresize. So the filter is letting more through than expected.

Out of all the allowed ones, onresize is picked for the exploit because it can be forced to fire without any interaction from the victim. Attributes like onsearch or onscrollend need the victim to actually type in a search box or scroll something, and others like onratechange or onpromptaction need a specific element structure (video, popover) that does not fit cleanly into this injection point. onresize just needs the element's size to change, which can be triggered purely with JavaScript from an iframe wrapper, no victim action required.

### Step 4: Build the working exploit

Combining the body tag with onresize gives a payload that gets past the filter:

```
"><body onresize=print()>
```

The problem is onresize will not fire by itself since nothing on the page is resizing. To force it, this payload gets loaded inside an iframe, and the iframe's width gets changed with JavaScript right after it loads, which triggers a resize on everything inside it.

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload=this.style.width='100px'>
```

| Flag/Argument | Description |
|---|---|
| `src` | Points to the lab's search endpoint with the URL-encoded payload |
| `%22` | Encoded double quote, closes the existing search attribute |
| `%3E` / `%3C` | Encoded `>` and `<`, used to break out and inject the body tag |
| `onresize=print()` | Fires when the body element inside the iframe gets resized |
| `onload=this.style.width='100px'` | Runs as soon as the iframe loads, shrinking its width and forcing the resize |

### Step 5: Deliver the exploit

Paste the iframe code into the exploit server body, replace the lab ID placeholder with the actual lab ID, click Store, then click Deliver exploit to victim. The victim's browser loads the iframe, the width changes right after load, the resize fires on the injected body tag, and print() runs in the victim's session.

## Detection

* Logging requests where the search parameter contains raw or encoded angle brackets combined with a closing quote pattern like `">`
* Flagging responses where an unusual tag/attribute combination such as body plus an event handler appears in reflected output
* Monitoring for automated fuzzing traffic against a single parameter, since Intruder-style testing sends a large number of near-identical requests in a short window

## Prevention

* Use context-aware output encoding instead of a blacklist of tags and attributes, since blacklists will always miss some combination
* Apply a strict Content Security Policy that blocks inline event handlers entirely
* Validate and sanitize input against an allowlist of expected characters for the search field rather than trying to block specific payloads
* Encode output based on where it lands in the HTML (attribute, tag body, script context) rather than filtering the raw input alone

## References

PortSwigger Web Security Academy — Reflected XSS with some SVG markup allowed
https://portswigger.net/web-security/cross-site-scripting/contexts/lab-xss-in-body-onresize-event

PortSwigger XSS Cheat Sheet
https://portswigger.net/web-security/cross-site-scripting/cheat-sheet

## Key Takeaways

* Blacklist-based filters can be mapped out systematically instead of guessed, using Intruder with a full tag list and then a full event list
* body plus onresize is a filter bypass worth remembering since onresize is not as commonly blocked as onload or onerror
* onresize needs an actual resize to fire, which is why the payload alone is not enough and an iframe with a forced width change is needed to trigger it
* Building an exploit chain from small individual findings (one allowed tag, one allowed attribute) is often more effective than trying to find one payload that beats the filter in a single shot