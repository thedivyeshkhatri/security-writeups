# SSRF to Root via Tar Wildcard Injection — TryHackMe

**Date:** 2026-09-11  
**Category:** Web, Full Chain  
**Skills used:** Nmap, anonymous FTP enumeration, source code review, SSRF (URL-validation bypass), Tar Wildcard Injection, Linux privilege escalation  

> **TL;DR:** Anonymous FTP leaked the application's source code, revealing an SSRF-prone
> `/preview` endpoint whose host-validation logic could be bypassed to reach an internal-only
> `/admin/notes` page — leaking SSH credentials. From there, a root cron job running `tar` with
> a wildcard was vulnerable to **Tar Wildcard Injection**, giving a one-shot root command
> execution.

## Overview
The chain: **Nmap recon → anonymous FTP source disclosure → source review → SSRF via
URL-validation bypass → leaked SSH credentials → cron job discovery → Tar Wildcard Injection →
root.** The most interesting part is that the *first* privilege escalation vector (SSRF) was
found entirely by **reading the application's own source code**, which had been leaked through
a completely unrelated misconfiguration (anonymous FTP).

## Stage 1 — Recon
Nmap found **21** (FTP) and **80** (HTTP) open, and the FTP server allowed **anonymous login** —
a serious misconfiguration on its own, since it exposes anything on the server to anyone.

## Stage 2 — Source Code Disclosure via Anonymous FTP
Logging in as `anonymous`, I found and downloaded **`backup.tar.gz`**, containing three files:
`readme.md`, `requirements.txt`, and — critically — **`app.py`**, the application's full
source code.

## Stage 3 — Finding the SSRF in Source
Reading `app.py` revealed:
- An **`/admin`** endpoint, plus a sub-page **`/admin/notes`**.
- A **`/preview`** feature that fetches a user-supplied URL server-side — but first validates it
  by checking that the parsed host is **exactly `kestrel.thm`**, using Python's `urlparse`.

The bug: the app validates the URL with `urlparse`, but then passes the **original, raw string**
to `requests.get()`. That gap between *what gets validated* and *what actually gets requested* is
a classic SSRF-via-parser-confusion pattern — the check confirms the host looks right, but
doesn't guarantee the request that's actually made resolves to something equally restricted. In
this case, formatting the path with a **doubled slash** was enough to reach a page that direct,
external requests to `/admin` were otherwise blocked from — because the `/preview` feature
itself runs server-side, effectively bypassing whatever network-level restriction kept `/admin`
unreachable from outside.

## Stage 4 — Exploiting the SSRF to Reach `/admin/notes`
Crafted request:

```
/preview?url=http%3A%2F%2Fkestrel.thm%2F%2Fadmin%2Fnotes
```

…which decodes to `http://kestrel.thm//admin/notes`. This passed the `urlparse` host check
(netloc still resolves to `kestrel.thm`) while reaching the internal `/admin/notes` page via the
app's own server-side request — an **SSRF that turned an internal-only admin page into something
externally reachable**, without ever finding a direct hole in `/admin`'s own access control.

The page revealed SSH credentials:

```
user: webdev
pass: [REDACTED]
```

## Stage 5 — SSH Access & First Flag
SSH'd in as `webdev`. The **first flag** was sitting in the home directory.

## Stage 6 — Finding a Vulnerable Root Cron Job
Found a cron job, `voltlabs-backup`:

```cron
# Volt Labs staging backup - runs as root
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

Two things made this exploitable:
1. It runs as **root**, every minute.
2. It uses a **wildcard (`*`)** to select files to archive, and `webdev` had **write access to
   `/opt/backups`**.

## Stage 7 — Tar Wildcard Injection
When `tar` expands a shell wildcard like `*`, any filename in that directory that happens to
**look like a command-line option** (starting with `--`) gets interpreted as one, rather than
being archived as data. GNU tar has a `--checkpoint`/`--checkpoint-action` feature originally
meant for progress reporting during large archives — and `--checkpoint-action` can be set to
**`exec`**, running an arbitrary command mid-archive.

I dropped three files into `/opt/backups`:

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh exploit.sh'
echo 'echo "root:pass1234" | chpasswd' > exploit.sh
```

When the cron fired, `tar czf /var/backups/uploads.tgz *` expanded the wildcard to include the
two option-like filenames *as arguments*, effectively running:

```
tar czf /var/backups/uploads.tgz --checkpoint=1 --checkpoint-action=exec=sh exploit.sh ...
```

…which told `tar` to execute `exploit.sh` **as root** partway through the archive process. My
script changed the root password, and I logged in as root — capturing the **second flag**.

**Operational note:** since the cron re-ran every minute and would have kept resetting the root
password indefinitely, I removed `exploit.sh` immediately after confirming access. In a real
engagement I'd instead drop a script that plants a SUID `bash` copy in `/tmp` (e.g.
`cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash`) rather than repeatedly touching the
account's actual password — quieter, and doesn't risk locking out the legitimate owner or
tripping alerting on repeated password changes.

## The Fix (Defensive View)
- **Anonymous FTP (Stage 1–2):** disable anonymous access entirely, and never store application
  source or backups anywhere reachable by it.
- **SSRF via validation/request mismatch (Stage 3–4):** validate and use the **same** parsed,
  canonical representation of the URL for both the check and the actual request — never validate
  one string and request another. Additionally, don't rely on hostname allow-listing alone to
  protect an internal-only page; enforce that restriction at the network level too, so a
  server-side fetch can't silently bypass it (see my SSRF writeup for the same underlying
  lesson: validate the *resolved* target, not the input string).
- **Cron + wildcard + writable directory (Stage 6–7):** never run privileged cron jobs with
  wildcards over a directory writable by a lower-privileged user. If backups must include
  user-writable content, restrict the archive to explicit filenames, or run `tar` with
  `--no-recursion`/`--` guards, or simply ensure the backup directory isn't writable by anyone
  but root.

## What I Learned
Two techniques here I hadn't used before. First, the SSRF wasn't a simple "reach an internal
IP" bug — it was a **validation/request mismatch**, where the code checked one thing and acted
on another; that's a pattern worth specifically looking for in source review, not just testing
blindly from the outside. Second, **Tar Wildcard Injection** was new: understanding that `tar`'s
checkpoint-action feature can be triggered by filenames alone, with no direct command injection
needed, showed me that "wildcard + privileged cron + writable directory" is its own distinct
privesc primitive worth checking for specifically. The operational choice to avoid repeatedly
resetting the root password was a good reminder that even in a lab, practicing engagement
hygiene (minimal footprint, clean up after confirming access) is a habit worth building now.
