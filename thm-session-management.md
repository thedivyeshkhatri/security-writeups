# Session Management — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-20  
**Category:** Web, Authentication  
**Skills used:** Session management concepts, cookie & token analysis, HTTP request inspection  

## Overview
We don't send a username and password with every request. Instead, once authenticated, we're
given a **session**. The web application uses this session to keep our state, track our actions,
and decide whether a user is allowed to do what they're trying to do. Session management is
about making sure all of that happens correctly and securely.

## Session Management Lifecycle
The lifecycle has four stages:

1. **Session Creation** — on many web applications a session is created when you first visit,
   and once you provide valid credentials, your session value is sent with each new request.
2. **Session Tracking** — that session value is submitted with every subsequent request,
   letting the app track a user's actions even though HTTP itself is **stateless**. On each
   request, the app does a server-side lookup on the session to determine who it belongs to and
   what permissions they have.
3. **Session Expiry** — every session has a lifetime. If it expires and you submit an old
   session value, the app should reject it.
4. **Session Termination** — when the user logs out, the app should terminate the session.
   This is similar to expiry but distinct: even if the session's lifetime is still valid, an
   explicit logout must invalidate it.

## The IAAA Model
Understanding session vulnerabilities starts with the IAAA model:
- **Identification** — establishing who the user claims to be (usually submitting a username).
- **Authentication** — proving the user is who they claim to be (e.g. the password, MFA).
- **Authorisation** — ensuring that specific user has permission to perform the requested action.
- **Accountability** — creating a record of the actions users perform (logging/auditing).

## Cookie-Based Session Management
When an application wants to start tracking a user, its response includes a **`Set-Cookie`**
header. The browser interprets this and stores the cookie. The key point is that **the browser
decides when to send a cookie** — based on the cookie's domain and attributes, it attaches the
cookie automatically, with no client-side JavaScript involved.

## Token-Based Session Management
Token-based management is newer and relies on **client-side code** instead of the browser's
automatic cookie handling. After authentication, the application returns a **token** in the
response body; client-side JavaScript then stores it (commonly in the browser's **local
storage**) and manually attaches it to future requests.

## Cookie vs Token: The Security Trade-off
These two approaches fail in *opposite* ways — worth understanding together:

- **Cookies are attached automatically** by the browser, which is exactly why cookie sessions
  are vulnerable to **CSRF** (see my CSRF writeup) — a forged cross-site request rides along
  with the cookie. But a cookie marked **`HttpOnly`** can't be read by JavaScript, so it
  survives an **XSS** attack.
- **Tokens in local storage are attached manually** by JavaScript, so they're **not**
  vulnerable to CSRF — but any **XSS** can read local storage directly (the same
  `document`-reading technique as my XSS cookie-theft payload would steal the token).

So neither is "more secure" by default; the right defenses differ for each.

## The Fix (Defensive View)
Secure session management comes down to a few essentials:
- **Unpredictable session IDs** — long, random values so they can't be guessed or brute-forced.
- **Cookie attributes:** `HttpOnly` (blocks JS access → defends against XSS token theft),
  `Secure` (only sent over HTTPS → defends against sniffing), and `SameSite` (limits
  cross-site sending → defends against CSRF).
- **Proper expiry and termination** — enforce lifetimes *and* invalidate the session
  server-side on logout, so a stolen or old session can't be reused.
- **Always transmit sessions over HTTPS**, so the session value can't be sniffed in transit.

## What I Learned
Learned how sessions solve HTTP's statelessness, the four stages of the session lifecycle, and
the IAAA model as a framework for reasoning about access. The most valuable insight was seeing
that **cookie-based and token-based sessions have opposite weaknesses** — cookies lean toward
CSRF risk, tokens toward XSS risk — which connected this module directly to the CSRF and XSS
work I'd done earlier and made the purpose of flags like `HttpOnly`, `Secure`, and `SameSite`
click into place.
