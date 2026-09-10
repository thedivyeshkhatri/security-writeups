# Internal Ops Panel: SQLi to Root — TryHackMe (Silent Monitoring)

**Date:** 2026-09-10  
**Category:** Web, Full Chain  
**Skills used:** Nmap, Gobuster, SQL injection (auth bypass), newline/command injection, credential hunting, offline KeePass brute force  

> **TL;DR:** Bypassed login on an internal ops panel with a classic SQLi payload, found a "ping
> an IP" tool that was vulnerable to **command injection via a smuggled newline (`%0a`)**,
> pulled plaintext SSH credentials from a config file in the working directory, then cracked an
> offline **KeePass** vault found on that host to recover the root password.

## Overview
The chain: **Nmap → Gobuster directory discovery → SQLi login bypass → command injection on an
internal tool → credential file exposure → SSH → offline KeePass crack → root.** Each stage was
a distinct vulnerability class, but the chain stayed short and direct — four hops from
unauthenticated to root.

## Stage 1 — Recon
Nmap found an unusual open port, **5050**, hosting a web application. Running **Gobuster**
against it turned up **`/internal`**, a login page not linked from the public site.

## Stage 2 — SQL Injection Authentication Bypass
The login form was vulnerable to a classic SQLi auth-bypass payload:

```
Username: admin' OR 1=1 -- 
```

This comments out the rest of the query's password check, so the query effectively becomes
"return a user where username is admin OR true" — logging me in as a **`netops`** operator
without valid credentials.

## Stage 3 — Command Injection via Newline Smuggling
Inside the panel, **`/internal/health`** had an input box that pinged a supplied IP address —
a classic ping-wrapper pattern that's often unsafely built from shell commands.

Using the browser's inspector, I appended a **URL-encoded newline (`%0a`)** followed by
`whoami` to the request:

```
ip=127.0.0.1%0awhoami
```

The injected `whoami` ran successfully, confirming **command injection** (the same
vulnerability class as my dedicated Command Injection writeup, here delivered via a smuggled
newline rather than a shell operator like `;` or `&&`).

## Stage 4 — Credential Discovery & SSH Pivot
With command execution, I explored the working directory and found **`secret.config`**,
containing plaintext credentials:

```
username: sysadmin
password: [REDACTED]
```

Storing credentials in a plaintext config file readable by the web process is its own finding,
independent of the command injection that exposed it. I used them to **SSH in as `sysadmin`**.

## Stage 5 — Offline KeePass Crack to Root
On the `sysadmin` host, I found **`infrastructure.kdbx`** — another KeePass vault. Rather than
guessing manually, I brute-forced it offline with **[bfkeepass.py](https://github.com/toneillcodes/brutalkeepass)**,
a KeePass brute-forcing tool. This recovered the master password, **`spring`** — a weak,
dictionary-guessable choice for a vault protecting infrastructure secrets.

Opening the database revealed the **root password**, completing the chain to full root access.

## The Fix (Defensive View)
- **SQLi auth bypass (Stage 2):** use parameterized queries everywhere, especially login logic
  — string-concatenated SQL in an auth check is one of the most damaging places this bug can
  live (see my SQL Injection writeup).
- **Command injection via newline (Stage 3):** never build shell commands from user input.
  Note that this bypass shows **blocklisting shell operators like `;`/`&&` is not enough** —
  a raw newline can terminate one command and start another just as effectively. Use
  parameterized/array-form execution instead (see my Command Injection writeup).
- **Plaintext credentials (Stage 4):** never store credentials in a config file in cleartext or
  in a location reachable by the web application's process; use a secrets manager or, at
  minimum, restrictive file permissions outside the web root.
- **Weak KeePass master password (Stage 5):** a password vault is only as strong as its master
  password — `spring` is trivially dictionary-guessable. Enforce a strong, unique master
  password and consider a key file or hardware key as a second factor.

## What I Learned
The most useful technical detail was in Stage 3: I'd been thinking of command injection purely
in terms of shell operators (`;`, `&&`, `|`), but a **raw newline** works just as well against a
naive `ping` wrapper, because it terminates the intended command line entirely rather than
chaining onto it. That's a filter-bypass angle worth remembering — blocking `;` and `&&` alone
doesn't stop newline-based injection. The chain also reinforced a theme from my other writeups:
weak secrets (a guessable KeePass password, a plaintext credential file) are often the actual
final barrier, even after the initial technical vulnerability is exploited.
