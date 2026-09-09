# Broken Authentication — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-21  
**Category:** Web, Authentication  
**Skills used:** Username enumeration, credential brute force (ffuf), HTTP Parameter Pollution, cookie manipulation  

> **TL;DR:** Exploited three authentication flaws — enumerated valid usernames and brute-forced
> a login, hijacked a password-reset flow via **HTTP Parameter Pollution** (a POST `email`
> overriding the query-string value through PHP's `$_REQUEST`), and escalated to admin by
> editing a plaintext, unverified session cookie. Falls under **A07:2025 – Authentication
> Failures**.

## Overview
Authentication is the process of verifying the identity of a user making a request to a web
application. An **authentication bypass** is any attack that lets a user reach functionality
restricted to a given account without supplying the correct credentials for that account.

## Types of Authentication Bypass
Four techniques commonly appear in real-world testing:
- **Username enumeration** — building a list of accounts registered on the target by submitting
  candidate usernames to a form that responds *differently* for registered vs unregistered
  values (e.g. a signup or password-reset page).
- **Credential brute force** — taking that username list plus a dictionary of common passwords
  to find accounts whose passwords can be guessed.
- **Logic flaws** — when the intended flow of an authentication-related workflow (like account
  recovery) can be redirected using valid-looking input.
- **Cookie manipulation** — tampering with the session state the server returns to the client
  after login.

## Hands-On Lab

### Lab 1 — Enumeration + Brute Force
Using **ffuf**, I performed username enumeration and found **3 usernames** registered on the
site. I then ran a brute-force attack with a common-password list against those 3 accounts,
recovered the password for **1** of them, and logged in successfully.

### Lab 2 — Password-Reset Hijack via HTTP Parameter Pollution
The password-reset page used a two-step workflow. Step one accepts an **email**; if it matches
a known account, it advances to step two, which accepts the **username** tied to that email and
sends a reset link to the address on file.

The flaw is in how the two values are split across the request: the **email is in the URL query
string**, the **username is in the POST body**. Server-side, the app identifies the target
account from the query-string `email`, but composes the outbound reset message using PHP's
**`$_REQUEST`** superglobal.

`$_REQUEST` merges the query string, POST body, and cookies into one array, and when the same
key appears in more than one source, **the POST body takes precedence by default**. So placing a
*second* `email` parameter in the request body silently overrides the query-string value the app
used to locate the account — and the reset link is sent to the **attacker-chosen address**. This
is a classic **HTTP Parameter Pollution** bug.

### Lab 3 — Cookie Manipulation
A small lab where a **plaintext cookie** stored session state directly, in a form visible and
editable by anyone holding the cookie. Setting the admin flag manually escalated my privileges:

```bash
curl -H "Cookie: logged_in=true; admin=true" http://site.thm/cookie-test
```

A cookie accepted **without server-side verification** grants the session of whichever user the
cookie claims to be.

## The Fix (Defensive View)
Each lab maps to a concrete defense:
- **Enumeration:** return **identical responses** for valid and invalid usernames (same message,
  same timing) so attackers can't distinguish registered accounts.
- **Brute force:** enforce **rate limiting / lockouts**, and add **MFA** so a guessed password
  alone isn't enough.
- **Parameter pollution (Lab 2):** don't split a security decision across query string and body,
  and **never pass credentials or account identifiers in the URL query string** — use the POST
  body or headers. Avoid `$_REQUEST`; read explicitly from the intended source (`$_GET` /
  `$_POST`).
- **Cookie manipulation (Lab 3):** never trust client-editable state. Use **signed or
  server-side sessions** with unpredictable IDs, and verify authorization server-side on every
  request rather than trusting a flag in the cookie.

## What I Learned
This module tied together a lot of earlier work — brute force needed the enumeration output,
and the cookie lab connected straight back to my Session Management writeup (state that's
editable and unverified). The standout was **Lab 2**: understanding *why* `$_REQUEST` is
dangerous — that POST silently overrides the query string — turned an abstract "parameter
pollution" term into a concrete attack I could reason about and reproduce. It also lined up
exactly with OWASP's own guidance for A07 to keep identifiers out of URL query strings.
