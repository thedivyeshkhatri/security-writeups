# Security Writeups

Documented walkthroughs of the labs, rooms, and challenges I complete as I
learn offensive security. Each writeup covers what I did, what I learned,
and how I'd apply it in a real engagement.

All work is performed in authorized, legal environments only — practice
platforms, labs, and in-scope programs.

## Challenges & CTFs

End-to-end engagements against a target — reconnaissance through to
full compromise, chaining multiple vulnerabilities.

- [AD Delegation Abuse](tryhackme/ctf-writeups/thm-ctf-proxy.md) — SMB enumeration to Domain Admin via NTLM coercion, BloodHound pathfinding, and Kerberos constrained delegation (S4U2Self/S4U2Proxy) abuse
- [AD RBCD Abuse](tryhackme/ctf-writeups/thm-ctf-forward.md) — KeePass credential extraction, password spraying, and Resource-Based Constrained Delegation abuse to Domain Admin
- [Web-to-Root Chain](tryhackme/ctf-writeups/thm-ctf-domino.md) — hardcoded key recovery, broken password reset, IDOR, JWT secret forgery, stored XSS, RFI-to-RCE, and cronjob privesc to root
- [SQLi to Root](tryhackme/ctf-writeups/thm-ctf-silent-monitoring.md) — SQL injection auth bypass, command injection via newline smuggling, and an offline KeePass crack to recover root
- [Node App to Domain Admin](tryhackme/ctf-writeups/thm-ctf-dead-drop.md) — SQLi to Node.js RCE, credentials leaked via a decompiled APK reused on the domain, escalating to Domain Admin via direct AddMember abuse
- [SQLi to Root](tryhackme/ctf-writeups/thm-ctf-operation-promotion.md) — SQLi, admin-panel enumeration, command injection, pattern-based password recovery after getting stuck, and sudo/GTFOBins privesc to root
- [OTP Bypass to RCE](tryhackme/ctf-writeups/thm-ctf-interceptor.md) — backup file disclosure, an OTP verification logic flaw (renaming the field, not guessing the code), and command injection with filter evasion
- [SSRF to Root](tryhackme/ctf-writeups/thm-ctf-operation-coldstart.md) — anonymous FTP source disclosure, SSRF via URL-validation bypass, and Tar Wildcard Injection for root
- [File Disclosure → SQL Injection Chain](tryhackme/ctf-writeups/thm-ctf-recruit.md) — leaked app source via a file-read flaw to gain a foothold, then used UNION-based SQLi to recover admin credentials and take over the portal
- [Support Operations Platform](tryhackme/ctf-writeups/thm-ctf-support.md) — a five-stage chain from weak-password foothold to command-injection RCE
- [Linux PrivEsc Chain](tryhackme/ctf-writeups/thm-ctf-jump.md) — lateral movement through five users to root via cron poisoning, PATH hijack, sudo helper abuse, and a GTFOBins less escape
- [Windows PrivEsc Chain](tryhackme/ctf-writeups/thm-ctf-windows-jump.md) — escalating from anonymous SMB access to SYSTEM via a service-binary hijack, a registry AutoLogon credential, and a writable SYSTEM scheduled task

## Reconnaissance & Tooling

- [Network Reconnaissance](tryhackme/learning-notes/thm-network-recon.md) — passive vs active recon, common protocols, and why encrypted protocols (TLS, SSH) defend against sniffing and MITM
- [Nmap](tryhackme/learning-notes/thm-nmap.md) — host discovery, basic and advanced port scanning, and service/OS fingerprinting
- [Burp Suite](tryhackme/learning-notes/thm-burp-suite.md) — using Burp for intercepting and modifying web requests, plus credential attacks and macros


## Web Application Vulnerabilities

- [SQL Injection](tryhackme/learning-notes/thm-sqli.md) — exploiting error-based and UNION-based SQLi, enumerating databases via information_schema
- [Cross-Site Scripting (XSS)](tryhackme/learning-notes/thm-xss.md) — crafting context-specific XSS payloads, bypassing filters, and escalating from alert() to cookie theft
- [Cross-Site Request Forgery (CSRF)](tryhackme/learning-notes/thm-csrf.md) — exploiting missing and weak (base64-encoded) CSRF tokens to take over accounts and escalate privileges
- [Server-Side Request Forgery (SSRF)](tryhackme/learning-notes/thm-ssrf.md) — bypassing a deny-list defense with a path-traversal trick to reach a restricted internal endpoint
- [Insecure Direct Object Reference (IDOR)](tryhackme/learning-notes/thm-idor.md) — exploiting a missing authorization check on an API endpoint to access other users' profiles
- [File Inclusion](tryhackme/learning-notes/thm-file-inclusion.md) — path traversal and LFI with null-byte injection, plus RFI escalated to remote code execution
- [Command Injection](tryhackme/learning-notes/thm-command-injection.md) — verbose and blind OS command injection, shell operators, and time-based detection
- [API Pentesting](tryhackme/learning-notes/thm-api-pentesting.md) — BOLA, JWT attacks (weak secrets, none-algorithm), excessive data exposure, and mass assignment
- [Broken Authentication](tryhackme/learning-notes/thm-broken-auth.md) — username enumeration, brute force, a password-reset hijack via HTTP Parameter Pollution, and cookie manipulation
- [Session Management](tryhackme/learning-notes/thm-session-management.md) — session lifecycle, the IAAA model, and the opposite CSRF/XSS weaknesses of cookie vs token sessions
