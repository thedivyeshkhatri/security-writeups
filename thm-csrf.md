# Cross-Site Request Forgery (CSRF) — TryHackMe

**Date:** 2026-07-15  
**Category:** Web  
**Skills used:** CSRF exploitation, HTTP request analysis, CSRF token inspection  

> **TL;DR:** Exploited two CSRF flaws in the TryHackMe lab — a form with no CSRF token
> (to change a victim's email) and a form with a weak, base64-encoded token (to escalate
> a user's role) — by hosting forged requests on a malicious page.

## Overview
CSRF is a web vulnerability where an attacker tricks the browser into sending a request to
a website where the user is already authenticated. Because the browser automatically
includes session cookies with every request, the web application assumes the request was
made intentionally by the user.

## How a CSRF Attack Works
1. The victim logs in to a legitimate web application, and their browser stores a session
   cookie.
2. The attacker tricks the victim into visiting a malicious webpage containing a forged
   request.
3. The victim's browser automatically sends that request to the target application, along
   with the stored session cookie.
4. Because the request carries a valid session cookie, the server treats it as a
   legitimate user action.

## Features Commonly Vulnerable to CSRF
- Change email address
- Update account password
- Edit profile details
- Alter payment information
- Submit a user-preference form

If these requests don't include any protection mechanism (such as a CSRF token), they may
be vulnerable to forged requests.

## Hands-On: TryHackMe Lab

**1. Form with no CSRF token — email takeover**  
Exploited an HTML form that had no CSRF token by creating a simple webpage that sends a POST
request to update the account's email address. When the victim opens the link, the server
receives the request along with the victim's session cookie, and the email address is
updated — handing account control to the attacker.

**2. Weak CSRF token — privilege escalation**  
Exploited a weak CSRF token to update a user's role within the company. The token was simply
the **base64 encoding of the role**, so it was trivial to forge. Using that knowledge, I
built a webpage displaying an image with a small JavaScript event handler attached. When the
victim moves their mouse over the image, the event fires, redirects to the server URL with
the forged request, and the role gets updated.

## The Fix (Defensive View)
The root cause is that the server trusts the session cookie alone to prove intent. Defenses,
strongest first:

- **Anti-CSRF tokens** — a unique, unpredictable, per-session (ideally per-request) token
  that the attacker's page can't guess. Note: the second lab shows why the token must be
  *random* — a base64-encoded value derived from known data (like the role) is not a real
  token.
- **SameSite cookies** (`SameSite=Lax` or `Strict`) — tells the browser not to send session
  cookies on cross-site requests, which neutralizes most CSRF at the browser level.
- **Re-authentication / confirmation** for sensitive actions (password/email changes,
  payments), so a single forged request isn't enough.
- **Checking Origin/Referer headers** as a secondary control.

## What I Learned
A repeatable methodology for finding CSRF:
- Focus on **state-changing requests** (the ones that modify data).
- Inspect requests for CSRF tokens — and check whether the token is actually
  unpredictable, or just encoded known data.
- Analyze the HTTP methods in use.
- Replay requests **outside** the application to see if they still succeed.
- Observe cookie behavior — is the session cookie sent cross-site?

The most valuable takeaway was the second lab: a "CSRF token" that's just base64-encoded
data provides no real protection, because anything reversible or predictable can be forged.
