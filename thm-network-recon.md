# Network Reconnaissance — TryHackMe

**Date:** 2026-06-28
**Category:** Network, Fundamentals  
**Skills used:** Passive & active reconnaissance, protocol analysis  

> **TL;DR:** Foundational module on how attackers gather information about a target network —
> passive vs active recon — and the common protocols involved, including why insecure
> protocols (HTTP, FTP, Telnet) are risky and how their secure counterparts (HTTPS/TLS, SSH)
> defend against sniffing and man-in-the-middle attacks. This was the start of my learning, so
> everything here is at a foundational level.

## Overview
Reconnaissance is the first phase of an assessment: gathering information about a target before
any exploitation. The more an attacker (or a defender testing their own systems) knows about a
network's hosts, open ports, and running services, the more focused everything that follows
becomes.

## Passive vs Active Recon
- **Passive reconnaissance** — gathering information *without directly interacting* with the
  target, so there's nothing for the target to log. Sources include public records, DNS data,
  search engines, and other open-source intelligence (OSINT). Low risk of detection.
- **Active reconnaissance** — directly interacting with the target to pull information, e.g.
  port scanning or banner grabbing to see which services are running. More detailed, but it
  generates traffic the target can detect and log.

The trade-off is the key idea: passive is quiet but limited; active is richer but noisy.

## Common Protocols & Services
Each service typically listens on a well-known port, which is why recon starts with finding
open ports and identifying what's behind them:

- **HTTP** — web traffic. Plaintext, so anything sent (including form data) can be read if
  intercepted. The secure version is **HTTPS**, which wraps HTTP in **TLS**.
- **FTP** — file transfer. Sends credentials and data in plaintext. Secure alternatives:
  **FTPS** or **SFTP** (over SSH).
- **SMTP** — sending email between servers.
- **POP3** — retrieving email, typically downloading and removing it from the server.
- **IMAP** — retrieving email while keeping it synced on the server across devices.

The recurring theme: many classic protocols were designed **without encryption**, so their
contents are exposed on the wire unless a secure layer is added.

## Attacks & Secure Protocols
This is where the "why it matters" clicks — insecure protocols enable specific attacks, and
secure protocols exist to stop them:

- **Sniffing** — capturing network traffic as it passes. On a plaintext protocol, a sniffer
  can read credentials and data directly.
- **Man-in-the-Middle (MITM)** — an attacker positions themselves between two parties,
  intercepting (and possibly altering) traffic while both sides believe they're talking
  directly.
- **TLS (Transport Layer Security)** — encrypts traffic so a sniffer sees only ciphertext, and
  authenticates the server (via certificates) so a MITM can't silently impersonate it. This is
  what turns HTTP into HTTPS.
- **SSH (Secure Shell)** — encrypted remote administration, the secure replacement for
  plaintext protocols like Telnet. Provides an encrypted channel plus server and (optionally)
  key-based client authentication.
- **Password attacks** — attempts to gain access by guessing credentials, e.g. **brute force**
  (trying many combinations) or **dictionary attacks** (trying a wordlist of likely passwords).
  These are why strong, unique passwords, rate limiting, and multi-factor authentication matter.

## The Fix (Defensive View)
The through-line for defense across this whole module:
- **Use encrypted protocols everywhere** — HTTPS/TLS over HTTP, SSH over Telnet, SFTP/FTPS over
  FTP — so sniffing yields nothing and MITM is detectable via certificate validation.
- **Reduce the attack surface** — close unused ports and disable services you don't need, so
  there's less for active recon to find.
- **Defend authentication** — strong password policies, rate limiting/lockouts, and MFA blunt
  password attacks.
- **Monitor for active recon** — port scans and repeated login attempts are visible in logs;
  detecting them early is a defensive win.

## What I Learned
As the first module in my learning, this gave me the mental model everything else builds on:
you can't attack (or defend) what you haven't mapped. The most useful takeaway was pairing each
insecure protocol with its secure counterpart — understanding *why* HTTPS, SSH, and SFTP exist
made the point of encryption concrete rather than abstract. It also framed recon as the
foundation of the whole process: the quality of everything downstream depends on how well you
understood the target first.
