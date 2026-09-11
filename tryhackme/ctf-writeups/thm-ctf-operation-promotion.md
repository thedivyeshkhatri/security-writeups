# RecruitCorp: SQLi to Root via Pattern-Guessed Password — TryHackMe

**Date:** 2026-09-11  
**Category:** Web, Full Chain  
**Skills used:** Nmap, SQL injection (auth bypass), command injection, hash cracking, password pattern recognition, sudo/GTFOBins privilege escalation  

> **TL;DR:** SQLi bypassed an admin login on the first attempt; the admin panel's built-in
> employee lookup surfaced a hidden service account; a command injection in that service's
> health-check endpoint gave RCE as `www-data`; a leaked password hash resisted cracking until I
> recognized a **site-specific naming pattern** and guessed it directly; and a dangerously broad
> `sudo` grant on `find` gave a one-line root shell via a classic **GTFOBins** technique.

## Overview
The chain: **Nmap recon → robots.txt disclosure → SQLi login bypass → admin-panel enumeration →
command injection (RCE) → credential file exposure → pattern-based password recovery →
sudo/GTFOBins to root.** This one is worth writing up in detail because I got genuinely stuck
partway through, and the way I got unstuck is as useful a lesson as the exploit chain itself.

## Stage 1 — Recon
Nmap found four open ports: **22** (SSH), **80** (HTTP), **139/445** (SMB). Two findings stood
out:
- **SMB2 signing was enabled but not required** — a relay-attack surface (not exploited in this
  engagement, but worth noting: signing should be *required*, not just supported).
- **`robots.txt` disallowed `/admin/`** — a classic case of trying to hide a path via robots.txt
  instead of actually restricting access to it. That's not access control; it's a map for an
  attacker.

## Stage 2 — SQL Injection Authentication Bypass
Visiting `/admin/` revealed a login form. It fell on the **first attempt** to the classic
bypass:

```
Username: admin' OR 1=1 -- 
```

Logged in as **admin** immediately — no iteration needed, meaning there was no input
sanitization on the login query at all.

## Stage 3 — Enumerating Employees via the Admin Lookup Feature
The dashboard included a built-in employee lookup: an input box to enter an ID and view that
employee's record. Since I was already authenticated as **admin**, this was a legitimate
feature working as intended, not an authorization bypass — the panel is *meant* to let admins
look up any employee.

Browsing IDs, entering **`7`** returned a record for an account called **`sysmaint`**, role
**`system`**, with a note attached:

> *"Service account for `/admin/sysmaint-checks/ping.php`. Do not disable."*

That note is effectively a signpost to the next vulnerability — the real finding here isn't the
lookup itself, but what it revealed.

## Stage 4 — Command Injection (RCE)
Visiting `/admin/sysmaint-checks/ping.php` showed its usage: `?host=<target>`, a tool that pings
a supplied host. Appending a chained command confirmed injection:

```
?host=127.0.0.1;whoami
```

The response included **`www-data`**, confirming **command injection** and giving me code
execution as the web server user (same vulnerability class as my Command Injection writeup —
here via a `;` shell operator rather than the newline trick from a previous engagement).

## Stage 5 — Credential File Exposure
From the RCE shell, `/var/www/html/config/db.conf` contained a **password hash for `jford`**.

## Stage 6 — Getting Stuck, and Recognizing a Pattern
The hash didn't crack against standard wordlists. At this point I was stuck, so I talked through
the problem with ChatGPT, which suggested building a **custom wordlist with `crunch`** targeting
a site-specific theme — for example, if the site referenced a season/year like "Spring2026,"
generating variations around it:

```bash
crunch 10 10 -t Spring20%% -o custom_wordlist.txt
```

That prompted me to recall that the website itself displayed **"Spring 2026."** Combining that
with a pattern I've noticed repeatedly across TryHackMe labs and CTFs — **passwords very often
end in `!`** — my first guess was `Spring2026!` / `spring2026!`. It worked immediately.

*(Worth being honest about: this isn't a real-world technique — it's a lab/CTF-specific pattern
I've picked up from experience, not a general password-cracking method. If it hadn't worked, the
next step was generating a proper `crunch` wordlist around "spring," "2026," "jford," and common
variants like `1loveSpring2026`, then password-spraying it.)*

With `jford`'s credentials, I found the **first flag** in the home directory.

## Stage 7 — Privilege Escalation via `sudo find` (GTFOBins)
Checking what `jford` could run as another user:

```
$ sudo -l
Matching Defaults entries for jford on recruitcorp:
    env_reset, mail_badpass, secure_path=..., use_pty

User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```

An unrestricted `NOPASSWD` grant on `find` is a well-known **GTFOBins** privilege-escalation
primitive, because `find` supports `-exec`:

```bash
sudo /usr/bin/find . -exec /bin/bash -p \;
```

- `sudo /usr/bin/find` — runs `find` with root privileges.
- `.` — starting search path (any valid path works).
- `-exec /bin/bash -p` — tells `find` to execute a new Bash shell; the **`-p` flag is critical**,
  forcing Bash to keep its elevated (privileged) permissions instead of dropping them, as it
  normally would when the effective and real UID differ.
- `\;` — terminates the `-exec` sequence.

This spawned a **root shell** immediately.

## The Fix (Defensive View)
- **SMB2 signing (Stage 1):** set signing to *required*, not just enabled, to close the relay
  surface even though it wasn't the path used here.
- **robots.txt disclosure (Stage 1):** never rely on robots.txt to hide sensitive paths — it's
  a courtesy for crawlers, not access control. Enforce real authentication.
- **SQLi (Stage 2):** parameterized queries on every auth path (see my SQLi writeup).
- **Sensitive service-account notes exposed via legitimate features (Stage 3):** not a
  vulnerability in the lookup itself, but a reminder that internal notes ("Do not disable",
  endpoint paths) shouldn't sit in fields visible through routine admin tooling — that
  information is exactly what an attacker who's already gained admin access will read first.
- **Command injection (Stage 4):** never build shell commands from user input; use safe APIs
  instead (see my Command Injection writeup).
- **Predictable/weak passwords (Stage 6):** the deeper issue isn't that the hash was crackable —
  it's that the password followed a **guessable, human pattern** tied to public site content.
  Enforce passwords with real entropy, independent of anything displayed publicly, and use a
  strong hashing algorithm (bcrypt/Argon2) to slow offline attempts regardless.
- **Sudo misconfiguration (Stage 7):** never grant blanket `NOPASSWD` on GTFOBins-listed
  binaries (`find`, `vim`, `less`, `awk`, etc.). If `find` access is genuinely needed, restrict
  it with exact allowed arguments, or use a wrapper script instead of the raw binary.

## What I Learned
The most valuable part of this engagement wasn't a technique — it was **getting stuck and
working through it methodically** rather than brute-forcing blindly. Talking through the
approach (including with an AI assistant for wordlist ideas) led me to notice something I'd
already half-registered subconsciously: **lab passwords in these environments tend to follow
human, thematic patterns** (a date/season shown on the site, a trailing `!`) rather than being
truly random. That's a CTF/lab-specific pattern, not a real-world credential-cracking method,
but recognizing it saved a lot of brute-force time here. On the privesc side, `sudo find` was a
good reminder to always run `sudo -l` early and check unfamiliar binaries against **GTFOBins**
before assuming a dead end.
