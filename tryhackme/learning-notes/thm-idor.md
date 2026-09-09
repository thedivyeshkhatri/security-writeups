# Insecure Direct Object Reference (IDOR) — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-19  
**Category:** Web, Access Control  
**Skills used:** IDOR exploitation, HTTP request manipulation, parameter mining  

> **TL;DR:** Found an IDOR in the API endpoint `/api/v1/customer?id={user_id}` — changing the
> `id` to another user's value returned their profile with no authorization check, exposing
> other customers' data.

## Overview
Web applications rely on identifiers to distinguish objects: a user profile, an invoice, a
support ticket, or a private document each have some kind of reference (often a number or a
string) that the application uses internally to locate them. When an application lets the user
supply that reference and then retrieves the corresponding object **without checking whether
that user is permitted to access it**, the result is an Insecure Direct Object Reference
(IDOR).

IDOR is an **access control** vulnerability. It falls under **Broken Access Control
(A01:2025)** — the #1 risk in the OWASP Top 10, and its single most common pattern.

## Hands-On: TryHackMe Lab
Inspecting the lab site, I found a request to the endpoint:

```
GET /api/v1/customer?id={user_id}
```

…where `{user_id}` is your own account's numeric identifier. When I passed in a **different
ID**, the application returned that other customer's profile — **without checking whether I
was authorized to view it**. That missing server-side ownership check is the entire
vulnerability.

## The Fix (Defensive View)
The root cause is that the server trusts the client-supplied reference and skips authorization.
Fixes, most important first:

- **Server-side authorization on every object access** — the real fix. For each request,
  verify the *current* user actually owns or is permitted to access the requested object
  before returning it. This is what was missing in the lab.
- **Use indirect / unpredictable references** — map internal IDs to per-user, non-guessable
  tokens (e.g. UUIDs), so an attacker can't simply enumerate `id=1, 2, 3...`. This is
  defense-in-depth, **not** a replacement for the authorization check.
- **Deny by default** — resources should be inaccessible unless an explicit check grants access.
- **Log and alert** on access-control failures, since IDOR probing shows up as many
  denied/altered-ID requests.

## What I Learned
Learned what causes IDOR (missing server-side authorization), how to find it (inspecting
requests for user-controllable identifiers in URLs, parameters, and API paths), and where it
tends to live (numeric IDs in query strings and REST paths). I also learned about **parameter
mining** — probing for hidden or guessable parameters that reference objects. The key insight:
IDOR isn't a coding trick, it's a *missing check* — the server has to verify ownership on
every single object request, because unpredictable IDs alone won't save you.
