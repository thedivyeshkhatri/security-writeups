# Server-Side Request Forgery (SSRF) — TryHackMe

**Date:** 2026-07-18  
**Category:** Web, Server-Side  
**Skills used:** SSRF exploitation, HTTP request manipulation, deny-list bypass  

> **TL;DR:** Identified an SSRF in an avatar-selection feature and bypassed its deny-list
> defense using a path-traversal trick (`x/../private`) to reach a restricted internal
> endpoint and read its base64-encoded contents.

## Overview
Server-Side Request Forgery (SSRF) is a vulnerability where an attacker can make the
**server-side** application send an HTTP request to a destination of the attacker's choosing.
Because the request originates from the server, it can reach internal systems the attacker
couldn't otherwise touch — including cloud metadata services and internal admin panels.

## Types of SSRF
- **Regular SSRF** — the response is returned to the attacker (visible).
- **Blind SSRF** — the response is not visible; success is inferred from side effects or an
  out-of-band callback.

## Finding an SSRF
Common signals that a request may be SSRF-able:
- A **full URL in a query parameter** in the address bar — almost always worth testing.
- A full URL in the page source, e.g. `http://server.website.thm/xyz`.
- A **partial URL or path** in a parameter, e.g. `https://website.thm/form?server=api` or
  `https://website.thm/form?dst=forms/contact` — the server is likely fetching that value.

## Bypassing Common SSRF Defenses
- **Deny lists** — block specific addresses/patterns, allow everything else. Bypass by
  disguising the target: convert the IP to **decimal or octal**, or use shorthand,
  wildcards, DNS-based tricks, or IPv6 representations of the blocked address.
- **Allow lists** — block everything unless it matches an approved entry. Stronger than a
  deny list, but implementation flaws still allow bypasses, e.g.:
  - **Subdomain matching:** `https://website.thm.attackers-domain.thm`
  - **URL credential abuse:** `https://website.thm@attacker.com/`
- **Open redirects** — an endpoint that forwards to a URL from a parameter, e.g.
  `https://website.thm/link?url=https://tryhackme.com`. If the app only allows URLs starting
  with `https://website.thm/`, chain the open redirect to reach a forbidden internal target:
  `https://website.thm/link?url=http://169.254.169.254/latest/meta-data/`
  *(`169.254.169.254` is the **cloud metadata endpoint** — a link-local address that can leak
  cloud credentials, making it the highest-value SSRF target.)*

## Hands-On: TryHackMe Lab
During content discovery, two endpoints stood out:
- **`/private`** — returns an error that its contents can't be viewed from your IP address.
- **`/customers/new-account`** — lets customers select an avatar for their profile.

Each avatar had a **radio button whose `value` attribute held the path to an image file**.
When the avatar updated, it was rendered using the **data URI scheme**, with the image content
base64-encoded in the `src` attribute.

Inspecting the radio button, I saw the `value` field could be edited. I changed it to
`private` and clicked *Update Avatar* — which returned the error **"URL cannot start with
private,"** revealing a **deny-list** defense.

To bypass it, instead of `private` I used **`x/../private`**. This doesn't *start* with
`private`, so it passes the deny list, but the server resolves it by entering directory `x`,
climbing back up one level with `../`, and landing on `/private`. The restricted contents were
then returned in the rendered, base64-encoded data — which I decoded to complete the lab.

## The Fix (Defensive View)
The root cause is the server fetching a user-controlled destination without proper validation.
My lab bypass shows exactly why deny lists fail. Defenses, strongest first:

- **Allow-list the destination** — permit only known, specific hosts/URLs, and validate the
  **fully resolved** address (after parsing and DNS resolution), not the raw string. This
  defeats path tricks like `x/../private` and IP-encoding bypasses.
- **Block requests to internal/link-local ranges** (127.0.0.0/8, 169.254.0.0/16, RFC1918)
  *after* resolution, protecting the cloud metadata endpoint.
- **Disable unneeded URL schemes** (`file://`, `gopher://`, etc.) and don't follow redirects
  to non-allow-listed hosts.
- **Require authentication on internal services** (like the metadata endpoint — e.g. IMDSv2),
  so a single SSRF isn't enough.

## What I Learned
Learned why SSRF works and the difference between regular and blind SSRF. Got hands-on with
finding SSRF from URL/path parameters, understood the common defenses and their weaknesses,
and — most usefully — defeated a real deny list with a path-traversal bypass. The key insight:
deny lists are fragile because there are countless ways to encode the same destination;
allow-listing the *resolved* address is the only reliable fix.
