# Reflected XSS into HTML Tag Attribute

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

---

## Introduction

This lab contains a reflected Cross-Site Scripting (XSS) vulnerability in an HTML tag attribute.

The search input is reflected inside the `value` attribute of an `<input>` element.

The main goal is to escape from the attribute and create an event handler that executes JavaScript.

---

## Why This Attack Works

The application places the user's input directly inside an HTML attribute.

The vulnerable HTML looks similar to:

`<input type=text name=search value="USER_INPUT">`

The important part is:

`value="USER_INPUT"`

Since the input is inside double quotes, if the application does not properly encode the `"` character, an attacker can close the existing attribute and add another attribute.

In this lab, the `onfocus` event handler can be used to execute JavaScript.

---

## Lab Setup

The lab was provided by PortSwigger Web Security Academy.

The vulnerable functionality is the search feature.

The search parameter is:

`search`

Example request:

`GET /?search=test12345`

---

## Prerequisites

Before attempting this lab, it is useful to understand:

* Basic HTML tags and attributes
* HTML attribute values
* Reflected XSS
* JavaScript event handlers
* Basic Burp Suite usage
* HTTP requests and responses

---

## Attack Flow

`Search input`

↓

`Input reflected in the response`

↓

`Identify the HTML context`

↓

`Input found inside the value attribute`

↓

`Test whether " can escape the attribute`

↓

`Quote is accepted`

↓

`Close the value attribute`

↓

`Add autofocus`

↓

`Add onfocus event handler`

↓

`Execute JavaScript`

↓

`Lab solved`

---

## Step-by-Step Walkthrough

### 1. Find where the input is reflected

I first entered a simple value:

`test12345`

After submitting the search, I intercepted the request using Burp Suite.

The request contained:

`GET /?search=test12345`

I then checked the response and searched for `test12345`.

The input appeared in the search result and also inside an input element.

The important reflection was:

`<input type=text placeholder='Search the blog' name=search value="test12345">`

This showed that my input was being placed inside the `value` attribute.

---

### 2. Identify the XSS context

The important part of the HTML was:

`value="test12345"`

So the input was inside an HTML attribute value.

The structure was:

`value="USER_INPUT"`

This is an HTML attribute context.

---

### 3. Test whether the quote can be closed

Before using an XSS payload, I tested whether the application filtered the double quote.

I searched for:

`TEST"TEST`

Then I checked the response.

The quote was not blocked or encoded.

This meant I could potentially close the existing `value` attribute.

---

### 4. Close the attribute and add a new attribute

Since the quote was accepted, I used:

`" autofocus onfocus=alert(123) x="`

The payload first closes the existing `value` attribute.

Then it adds `autofocus` and an `onfocus` event handler.

---

### 5. Trigger the event

The `autofocus` attribute makes the browser try to focus the input automatically.

When the input receives focus, the `onfocus` event is triggered.

The event contains:

`alert(123)`

This causes the JavaScript alert to appear.

---

### 6. Confirm the result

After submitting the payload, the JavaScript executed and the alert appeared.

This confirmed that the reflected XSS vulnerability could be exploited.

The lab was then marked as solved.

---

## Payload Breakdown

Payload:

`" autofocus onfocus=alert(123) x="`

### `"`

This closes the existing `value` attribute.

The original structure:

`value="USER_INPUT"`

becomes:

`value=""`

---

### `autofocus`

This is an HTML attribute that tells the browser to automatically focus the element when possible.

---

### `onfocus=alert(123)`

`onfocus` is an event handler.

When the element receives focus, the JavaScript inside the event handler is executed.

In this case:

`alert(123)`

runs.

---

### `x="`

The original HTML contains markup after the user input.

The `x="` part helps consume the remaining quote and keeps the resulting HTML structure valid.

The resulting structure is similar to:

`<input value="" autofocus onfocus=alert(123) x="">`

---

## Tools Used

* Burp Suite
* Burp Proxy
* Firefox
* PortSwigger Web Security Academy

---

## Detection

Developers and security testers can look for this vulnerability by checking whether user-controlled input is reflected inside HTML attributes.

A simple test is to submit a unique string such as:

`test12345`

Then inspect the HTTP response.

If the value appears inside an attribute such as:

`value="test12345"`

the exact output context should be checked.

Testing characters such as:

`"`

can also help determine whether the application properly encodes special characters.

---

## Prevention

The best way to prevent this type of XSS is to properly encode user-controlled data before placing it into HTML.

For HTML attribute values:

* Apply context-aware output encoding.
* Encode quotation marks and other special HTML characters.
* Use framework-provided automatic escaping.
* Avoid inserting untrusted input directly into HTML.
* Use a strong Content Security Policy as an additional security layer.
* Validate input where appropriate, but do not rely on input filtering as the main XSS defense.

For example, if user input needs to be placed inside an HTML attribute, the application should safely encode characters such as:

`"`

`<`

`>`

`&`

---

## References

* PortSwigger Web Security Academy — Cross-site scripting
* PortSwigger Web Security Academy — XSS contexts
* OWASP — Cross Site Scripting Prevention Cheat Sheet

---

## Key Takeaways

* Always identify where user input is reflected before testing XSS.
* The same payload does not work in every XSS context.
* HTML attribute context is different from normal HTML text context.
* A double quote can be important when the input is inside a quoted attribute.
* An event handler can create a JavaScript execution context.
* `autofocus` can help trigger a focus-based event without manually clicking the element.
* Proper context-aware output encoding is the main defense against XSS.
