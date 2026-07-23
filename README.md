# Security Writeups

Documented walkthroughs of the labs, rooms, and challenges I complete as I
learn offensive security. Each writeup covers what I did, what I learned,
and how I'd apply it in a real engagement.

All work is performed in authorized, legal environments only — practice
platforms, labs, and in-scope programs.

## Challenges & CTFs

End-to-end engagements against a target — reconnaissance through to
full compromise, chaining multiple vulnerabilities.

- [File Disclosure → SQL Injection Chain](thm-ctf-recruit.md) — leaked app source via a file-read flaw to gain a foothold, then used UNION-based SQLi to recover admin credentials and take over the portal
- [Support Operations Platform](thm-ctf-support.md) — a five-stage chain from weak-password foothold to command-injection RCE

## Reconnaissance & Tooling

- [Network Reconnaissance](thm-network-recon.md) — passive vs active recon, common protocols, and why encrypted protocols (TLS, SSH) defend against sniffing and MITM
- [Nmap](thm-nmap.md) — host discovery, basic and advanced port scanning, and service/OS fingerprinting
- [Burp Suite](thm-burp-suite.md) — using Burp for intercepting and modifying web requests, plus credential attacks and macros


## Web Application Vulnerabilities

- [SQL Injection](thm-sqli.md) — exploiting error-based and UNION-based SQLi, enumerating databases via information_schema
- [Cross-Site Scripting (XSS)](thm-xss.md) — crafting context-specific XSS payloads, bypassing filters, and escalating from alert() to cookie theft
- [Cross-Site Request Forgery (CSRF)](thm-csrf.md) — exploiting missing and weak (base64-encoded) CSRF tokens to take over accounts and escalate privileges
- [Server-Side Request Forgery (SSRF)](thm-ssrf.md) — bypassing a deny-list defense with a path-traversal trick to reach a restricted internal endpoint
- [Insecure Direct Object Reference (IDOR)](thm-idor.md) — exploiting a missing authorization check on an API endpoint to access other users' profiles
- [File Inclusion](thm-file-inclusion.md) — path traversal and LFI with null-byte injection, plus RFI escalated to remote code execution
- [Command Injection](thm-command-injection.md) — verbose and blind OS command injection, shell operators, and time-based detection
- [API Pentesting](thm-api-pentesting.md) — BOLA, JWT attacks (weak secrets, none-algorithm), excessive data exposure, and mass assignment
- [Broken Authentication](thm-broken-auth.md) — username enumeration, brute force, a password-reset hijack via HTTP Parameter Pollution, and cookie manipulation
- [Session Management](thm-session-management.md) — session lifecycle, the IAAA model, and the opposite CSRF/XSS weaknesses of cookie vs token sessions
