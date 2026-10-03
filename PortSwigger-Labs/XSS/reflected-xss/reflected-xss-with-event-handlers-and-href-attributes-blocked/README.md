# Reflected XSS with Event Handlers and Href Attributes Blocked

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

This lab reflects search input inside an anchor tag's `href` attribute. The filter here is tighter than a basic blocklist. It blocks every event handler attribute, so `onfocus`, `onclick`, `onerror`, none of them get through. It also blocks any `href` value that contains the string `javascript:`. So the two most obvious ways to get code execution are both closed off from the start.

Getting past this needs a different approach entirely, one that doesn't rely on an event handler or a static `javascript:` string at all.

## Why This Attack Works

The filter checks the HTML as it exists at the time the response is sent. It scans for `on` prefixed attributes and for `javascript:` inside `href` values, and strips or blocks anything matching those patterns.

SVG has a tag called `<animate>` that changes an attribute's value over time, this is normally used for animations. It is not an event handler, so the filter has no reason to touch it. More importantly, it doesn't set the `href` value directly in the HTML source. It sets that value at runtime, inside the browser, after the page has already loaded and passed through the filter's check.

This means the string `javascript:alert(1)` is never sitting in the raw HTML where the filter can see it. It only appears once the browser starts executing the SVG animation, which is after the filter has already done its job. By the time the dangerous value shows up, the check is already over.

## Lab Setup

* Target: PortSwigger Web Security Academy lab environment
* Application: search feature reflecting input inside an anchor `href` attribute
* Filter behavior: blocks all `on*` event handler attributes and any `href` value containing `javascript:`
* Tools used: Burp Suite (Repeater), web browser

## Prerequisites

* Basic understanding of HTML attributes and anchor tags
* Familiarity with SVG and how it differs from regular HTML elements
* Comfort testing payloads in Burp Repeater before trying them live

## Attack Flow

```
Search input submitted
        |
        v
Filter checks for on* event handlers -> blocked if found
        |
        v
Filter checks href value for "javascript:" -> blocked if found
        |
        v
SVG animate tag passes through (not an event handler, no javascript: in source)
        |
        v
Page loads, animate tag runs at runtime
        |
        v
href attribute value changed to javascript:alert(1) after the filter already checked it
        |
        v
Link clicked -> JavaScript executes
```

## Step-by-Step Walkthrough

**Step 1 — Find the injection point**

I used the search box and checked where the input landed in the response. It showed up inside an `href` attribute of an anchor tag.

**Step 2 — Test event handlers**

I sent a payload with `onfocus` in Burp Repeater, similar to what worked in an earlier lab:

```
"><a onfocus=alert(1) tabindex=1>click</a>
```

This got blocked. Any attribute starting with `on` was stripped or rejected outright.

**Step 3 — Test a javascript: href**

Next I tried setting the href directly to a javascript URL:

```
"><a href="javascript:alert(1)">click</a>
```

Blocked as well. The filter was scanning the `href` value for the string `javascript:` and rejecting it.

**Step 4 — Look for a way around static detection**

Since both direct routes were closed, I needed something that sets the dangerous value without it ever appearing in the raw HTML sent by the server. SVG's `<animate>` tag does exactly that, it modifies an attribute's value inside the browser after the page has loaded, so the filter never sees the final value.

**Step 5 — Build the payload**

```
"><svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a>
```

Breaking it down:

* `">` — breaks out of the current attribute and tag context
* `<svg>` — opens an SVG context, which is required for `<animate>` to work
* `<a>` — a fresh anchor tag with no href set yet
* `<animate attributeName=href values=javascript:alert(1) />` — changes the anchor's href to `javascript:alert(1)` at runtime, not in the static source
* `<text x=20 y=20>Click me</text>` — gives the link visible, clickable text inside the SVG

**Step 6 — Submit and trigger it**

I submitted this through the search box. The malicious href was not present anywhere in the static HTML, so the filter had nothing to catch. Once the page rendered, the animate tag kicked in and set the href to `javascript:alert(1)`. Clicking the "Click me" text ran the JavaScript and the alert popped up with the lab marked as solved.

## Detection

* SVG elements like `<animate>`, `<set>`, or `<animateTransform>` appearing in reflected input, especially combined with `attributeName=href` or `attributeName=src`
* Any reflected content involving SVG namespaces where none was expected on the page originally
* Anchor tags with no href in the initial HTML but a working, clickable link once rendered
* Logs showing repeated payload attempts that avoid `on*` prefixes and `javascript:` strings but still involve SVG tags

## Prevention

* Do not rely on filtering just event handler attributes and the `javascript:` string. Attackers can set dangerous values at runtime through mechanisms like SVG animation that the filter never sees in the static source.
* Sanitize using a proper HTML sanitization library that understands the full DOM and SVG specification, not simple string or regex matching.
* Apply a strict Content-Security-Policy that restricts script execution sources and blocks inline script execution paths, including javascript URLs.
* Avoid allowing SVG markup in user-controlled input unless it is strictly necessary, and if it is needed, strip or disable animation-related tags like `<animate>` and `<set>`.

## References

* PortSwigger Web Security Academy — Reflected XSS labs
* OWASP Cross-Site Scripting (XSS) Prevention Cheat Sheet

## Key Takeaways

* Filters that only check static HTML at response time can be bypassed by anything that changes the DOM after the page loads.
* SVG's `<animate>` tag can set attribute values like `href` at runtime without ever using an event handler or writing `javascript:` directly into the source.
* Blocking `on*` attributes and `javascript:` strings is not enough on its own. Real protection needs to account for what the browser does after the HTML has already been parsed.
* This kind of bypass shows why sanitization needs to understand the full range of HTML and SVG behavior, not just pattern match against known dangerous keywords.