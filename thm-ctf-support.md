# Support Operations Platform — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-23  
**Difficulty:** Medium  
**Category:** Web Application — Broken Access Control, IDOR, LFI, Command Injection (RCE)  
**Skills used:** gobuster, ffuf/hydra (brute force), Burp Suite (Repeater), cookie tampering, PHP filter wrappers, OS command injection, reverse shells  

> ⚖️ Performed in an authorized, in-scope lab environment (TryHackMe). All credentials, hashes, and flag values recovered during this exercise have been redacted from this writeup.

---

## Overview

The Support Operations Platform is an internal PHP web application for IT/helpdesk teams, offering ticket management, internal APIs, and system diagnostics. The objective was to assess it as an attacker, escalate access, and achieve Remote Code Execution (RCE) on the server.

The platform fell to a **chain of five weaknesses**: a weak account password (brute-forced), a client-controlled privilege cookie, an IDOR in the user API, a Local File Inclusion in the theme selector, and — the endgame — OS command injection in a diagnostics feature that gave RCE as `www-data`. No single flaw was game-over; chained, they produced full server compromise.

---

## Reconnaissance

### Port scan

`nmap` identified SSH (22) and an Apache/PHP web application (80). The session cookie was issued without the `HttpOnly` flag (a low-severity finding recorded below).

### Content discovery

`gobuster` (with `-x php`) mapped the application:

| Path | Notes |
|------|-------|
| `/index.php` | Login page (email + password) |
| `/dashboard.php` | Authenticated area — redirects to login when unauthenticated |
| `/api.php` | Internal API (serves the IT Admin panel and user-lookup routes) — redirects to login when unauthenticated |
| `/config.php` | Returns empty over HTTP (source hidden; high-value target) |
| `/info.php` | Exposed `phpinfo()` — information disclosure |
| `/includes/`, `/skins/`, `/layout/`, `/js/` | Component and theming directories |

### phpinfo disclosure

The exposed `info.php` (`phpinfo()`) disclosed the server's full configuration and, critically for the RCE goal:

- Web root `/var/www/html`, running as `www-data`, PHP 8.3 on Ubuntu.
- **`disable_functions` was empty** — every command-execution function (`system`, `exec`, `shell_exec`, …) was available.
- `allow_url_fopen` was enabled.

Recording these early meant that once *any* code-execution primitive was found, exploitation would be trivial — no function-restriction bypass required.

---

## Finding the Vulnerabilities

### 1. Weak account password (initial foothold)

The login is email-based with a generic "Invalid credentials" message (no user enumeration via wording). A helpdesk contact address was disclosed on the login page. That account used a weak password and was recovered by brute force, granting access to the dashboard as a low-privileged helpdesk user.

### 2. Client-controlled privilege cookie (broken access control)

Inspecting the authenticated session revealed a cookie holding an MD5 hash. Cracking it revealed the value `false`. Replacing the cookie with the MD5 hash of `true` caused the application to render an **IT Admin panel** that is normally hidden — the app trusted a client-controlled cookie to make an authorization decision rather than enforcing it server-side.

### 3. IDOR in the user API

The IT Admin panel is served by `api.php`, which exposed a user-lookup endpoint of the form `GET /user/<id>` (routed through `api.php`) returning account details. Note that `api.php` had previously redirected to the login page for unauthenticated requests — but once a valid session was combined with the tampered admin cookie (finding 2), it served the API. The IDs were **sequential and unauthenticated at the object level**, so iterating them disclosed every user record:

- `/user/1` → the administrator account (`admin: true`, and notably `2FA: false`).

This leaked the administrator's email and confirmed the account had no second factor.

### 4. Local File Inclusion in the theme selector

The dashboard's theme selector loaded skins via a parameter:

```
GET /dashboard.php?skin=red
```

An invalid value rendered the page without a theme and no error, indicating the include was guarded by a file-existence check (and that `.php` was likely appended to the value). Path traversal against the parameter allowed reading application source outside the intended directory:

```
GET /dashboard.php?skin=../config
```

This caused the contents of `config.php` to be emitted **inside an HTML comment** in the page — not rendered on the visible page, but plainly readable in the raw HTML via "View Source". This disclosed a hardcoded master password. PHP filter wrappers (`php://filter/convert.base64-encode/resource=...`) can be used against this parameter to read further source files as base64 and decode them offline (useful for files whose PHP would otherwise execute).

### 5. OS command injection in diagnostics (RCE)

The admin area included a date/time diagnostic control that submitted a `sys` parameter, e.g. `sys=date`. The value was passed into a shell command without sanitisation, allowing command chaining:

```
sys=date; cat /var/www/db.php
```

This returned the contents of a file outside the web root — confirming arbitrary command execution as `www-data`.

---

## Exploitation

The findings were chained as follows:

1. **Foothold** — brute-forced the helpdesk account password and logged into the dashboard.
2. **Privilege escalation (UI)** — flipped the MD5 privilege cookie from `false` to `true`, exposing the IT Admin panel.
3. **Information disclosure** — used the IDOR (`/user/1`) to identify the administrator account and its email; used the LFI (`?skin=../config`) to include `config.php`, whose contents were emitted into an HTML comment and recovered by inspecting the page's raw HTML source, disclosing the master password.
4. **Administrator access** — logged in as the administrator account (no 2FA), obtaining the first objective flag.
5. **RCE** — abused command injection in the `sys` diagnostic parameter to execute commands. A PHP reverse shell was delivered and executed, returning an interactive shell as `www-data`.
6. **Objective** — from the shell, read `/home/ubuntu/user.txt`.

> Flags and recovered credentials are redacted.

---

## The Fix (Defensive View)

**Command Injection (Critical).** User input (`sys`) is passed into a shell command. *Remediation:* never pass user input to a shell. Use language-native functions for the intended operation, or if a subprocess is unavoidable, pass arguments as an array (no shell interpretation) and strictly allowlist permitted values.

**Local File Inclusion (High).** The `skin` parameter is used to build an include path. *Remediation:* map theme names to a fixed server-side allowlist (e.g. `['default','red','green','blue']`) and never incorporate user input into a filesystem path; reject anything not on the list.

**Broken Access Control — privilege cookie (High).** Authorization is decided by a client-controlled cookie. *Remediation:* store authorization state server-side (in the session), never in a client-editable value; derive privilege from the authenticated identity on every request.

**IDOR in user API (High).** Object references are direct, sequential, and unauthorized. *Remediation:* enforce an authorization check on every object access ("is this caller allowed to see this user?"), independent of whether the UI exposes the ID.

**Hardcoded secret in source (High).** A master password is stored in `config.php`, and the file inclusion caused it to be output inside an HTML comment where any user could read it via View Source. *Remediation:* move secrets out of source into environment variables or a secrets manager; rotate the exposed value; and never emit configuration or debug data into page output, including HTML comments ("hidden" markup is fully readable by clients).

**Information disclosure — exposed phpinfo (Medium).** `info.php` reveals the full server configuration to attackers. *Remediation:* remove `phpinfo()` pages from production.

**Weak password (Medium)** and **missing `HttpOnly` on the session cookie (Low)** round out the findings. *Remediation:* enforce strong password policy and rate-limit authentication; set `HttpOnly`, `Secure`, and `SameSite` on session cookies.

---

## The Attack Chain

```
Brute force  ──►  helpdesk login (foothold)
Cookie flip (false→true)  ──►  IT Admin panel unlocked
   ├─ IDOR /user/1        ──►  admin email + no 2FA
   └─ LFI ?skin=../config ──►  master password (config.php source)
              │
        Admin login  ──►  FLAG 1
              │
Command injection (sys=)  ──►  RCE as www-data  ──►  reverse shell  ──►  /home/ubuntu/user.txt (FLAG 2)
```

Each weakness handed the attacker the *information or access* needed for the next. The application's recurring root cause was **trusting client-controlled input** — a cookie, an object ID, a file name, a shell argument — at every layer.

---

## What I Learned

- **The vulnerable surface was post-authentication.** Extensive pre-auth fuzzing of the theme parameter found nothing because the LFI lived *behind* the login. Lesson: don't assume the intended bug is reachable before auth — get a session first, then re-examine every parameter.
- **Inspecting my own session paid off.** The privilege escalation came from reading and tampering with my own cookie, not from any external payload. Client-side trust is worth checking on every engagement.
- **"Hidden" is not "secure" — check underneath the UI.** This app repeatedly assumed that if the interface didn't show something, it was protected: a hidden admin panel gated only by a client cookie, sequential user IDs behind that panel, and the master password tucked inside an **HTML comment** (invisible on the rendered page, but plainly there in View Source). Each fell to simply looking beneath the surface — reading the cookie, iterating the IDs, viewing the raw HTML. Always read the page source, not just the rendered page.
- **Read source once you have LFI.** Reading `config.php` (and other files via PHP filter wrappers) converted guesswork into facts and directly supplied the admin credential.
- **`disable_functions` being empty told me RCE would be easy before I even found the injection point** — reconnaissance of server config shaped which exploitation path was worth pursuing.
- **Precise commands matter.** Building a working reverse shell required getting the delivery command exactly right; a plausible-looking one-liner that doesn't actually execute wastes time. Verify each step actually did what you think it did.
