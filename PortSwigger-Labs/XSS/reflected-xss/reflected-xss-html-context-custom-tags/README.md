# Reflected XSS into HTML Context with All Tags Blocked Except Custom Ones

## Table of Contents

* [Introduction](#introduction)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Prerequisites](#prerequisites)
* [Attack Flow](#attack-flow)
* [Step-by-Step Walkthrough](#step-by-step-walkthrough)
* [Detection](#detection)
* [Prevention](#prevention)
* [References](#references)
* [Key Takeaways](#key-takeaways)

---

## Introduction

This lab has a search feature where my input gets reflected back in the page. Normally I would just throw a `<script>` or `<img onerror>` payload at it, but this app has a filter in front of it that blocks every known dangerous tag. So the usual tricks are dead on arrival.

The way around it is simple once I see it: the filter only knows about real tags. It has no idea what to do with a tag name it has never seen before. I can invent my own tag, and the browser will happily render it. From there I just need a way to run JavaScript without a real event like `onerror` or `onload`, since custom tags don't come with those built in.

## Why This Attack Works

The server-side filter works off a blocklist or an allowlist of known tag names. Things like `script`, `img`, `svg`, and `iframe` get stripped or rejected because the app already knows these are dangerous.

A made-up tag like `<xss>` is not on that list. The filter lets it through because it has no reason to flag something it does not recognize. The browser, on the other hand, does not reject unknown tags. It just treats them as generic HTML elements and renders them into the DOM.

The catch is that custom tags don't behave like real HTML elements. A real `<img>` tag fires `onerror` automatically if the image fails to load. A custom tag has no such built-in behavior, so an event has to be attached manually with `onfocus`.

But `onfocus` only fires when the element actually receives focus, and custom elements are not focusable by default the way an `<input>` or a link is. Adding `tabindex` to the tag makes it focusable. Once that's in place, all that's left is getting the browser to focus the element without needing a click. A URL fragment pointing at the element's id does exactly that: when a page loads with `#id-name` at the end of the URL, the browser scrolls to and focuses that element on its own.

Put together, this gives a full delivery chain with no interaction required from anyone who opens the link.

## Lab Setup

* Target: PortSwigger Web Security Academy lab environment
* Application: search feature with server-side output filtering
* Tools used: Burp Suite (Repeater), web browser

## Prerequisites

* Basic understanding of HTML tags and attributes
* Familiarity with how reflected input lands in a page
* Burp Suite for testing payloads before trying them in the browser

## Attack Flow

```
Search input submitted
        |
        v
Filter checks tag name against known/dangerous tags
        |
        v
Custom tag name (not on filter's list) passes through
        |
        v
Tag reflected into page as-is, with tabindex and onfocus set
        |
        v
URL fragment (#id) added to force auto-focus on page load
        |
        v
Browser focuses element automatically -> onfocus fires -> JavaScript runs
```

## Step-by-Step Walkthrough

**Step 1 — Find the injection point**

I opened the lab and used the search box on the homepage. Whatever I typed into it came back reflected somewhere in the response.

**Step 2 — Test the filter**

In Burp Repeater, I sent a normal payload first:

```
<script>alert(1)</script>
```

This got stripped or blocked. I tried a few other common tags like `<img src=x onerror=alert(1)>` and got the same result. The filter clearly knows the usual suspects.

**Step 3 — Try an unknown tag**

Next I sent a made-up tag name:

```
<xss>test</xss>
```

This came back untouched in the response. The filter had nothing to match it against, so it let it straight through.

**Step 4 — Build the real payload**

Since the input lands directly in the page body (not inside an attribute), there was no need to break out of anything with quotes or angle brackets. I could just place a new tag straight in:

```
<xss id=x onfocus=alert(document.cookie) tabindex=1>
```

Each piece here does a specific job:

* `<xss>` — a tag name the filter doesn't recognize, so it passes through
* `id=x` — gives the element something to target later from the URL
* `onfocus=alert(document.cookie)` — runs JavaScript the moment the element gets focus
* `tabindex=1` — makes the tag focusable, since custom tags aren't focusable on their own

**Step 5 — Submit and add the fragment**

I typed the payload into the search box and hit enter. The browser URL-encoded it automatically. Then I clicked into the address bar and added `#x` at the very end of the URL, matching the id I set in the payload.

**Step 6 — Load the final URL**

The full URL looked like this:

```
https://YOUR-LAB-ID.web-security-academy.net/?search=%3Cxss+id%3Dx+onfocus%3Dalert%28document.cookie%29+tabindex%3D1%3E#x
```

Loading this URL made the browser jump straight to the element with id `x` and focus it right away. That triggered `onfocus`, and the alert popped up showing the cookie value. Lab marked as solved.

## Detection

* Unknown or made-up HTML tag names showing up in a response, especially ones that were never part of the app's design
* Event handler attributes like `onfocus`, `onload`, `onerror` appearing in reflected input
* URLs containing a fragment (`#`) paired with suspicious query parameters
* WAF or filter logs showing repeated attempts with unrecognized tag names, which usually means someone is probing for gaps in the blocklist

## Prevention

* Stop relying on a blocklist of known tag names. New or unknown tags will always slip past this kind of check.
* Apply proper output encoding based on where the data lands. HTML body, attribute, and JavaScript contexts each need their own encoding rules.
* Use an allowlist approach for HTML if user input needs to support any formatting at all, and strip everything not explicitly permitted.
* Set a Content-Security-Policy header that blocks inline event handlers. Even if a tag gets through, the browser will refuse to run the inline JavaScript.

## References

* PortSwigger Web Security Academy — Reflected XSS labs
* OWASP Cross-Site Scripting (XSS) Prevention Cheat Sheet

## Key Takeaways

* Tag-based blocklists fail against custom or unknown tag names, since the filter can only block what it already knows about.
* Custom tags don't come with built-in events. `tabindex` plus `onfocus` is a reliable way to get code execution without needing a click.
* A URL fragment pointing at an element's id triggers auto-focus on page load, which means this attack needs zero interaction from the victim beyond opening a link.
* Where the input lands (inside an attribute vs. directly in the HTML body) changes whether a breakout sequence like `">` is needed. Always check the actual reflection point before assuming a payload structure.