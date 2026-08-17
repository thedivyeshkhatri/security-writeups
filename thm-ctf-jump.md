# Internal Automation Pipeline — TryHackMe

**Date:** 2026-08-14  
**Difficulty:** Medium / Hard  
**Category:** Linux Privilege Escalation — lateral movement through a multi-user pipeline  
**Skills used:** anonymous FTP, pspy, cron/script poisoning, SUID binaries, PATH hijacking, sudo rule abuse, GTFOBins shell escape  

> ⚖️ Performed in an authorized, in-scope lab environment (TryHackMe). All flags, passwords, and IP addresses recovered during this exercise have been redacted.

---

## Overview

The target ran a misconfigured internal automation pipeline on Linux, where each stage (recon, dev backups, monitoring, deployment) was operated by a separate user and trusted the output of the stage before it. By abusing those trust boundaries, it was possible to move laterally through five identities:

```
anonymous → recon_user → dev_user → monitor_user → ops_user → root
```

Each hop used a different privilege-escalation technique, making this a broad tour of common Linux privesc: an auto-processing upload directory, a group-writable cron script, a PATH hijack, a sudo-invoked helper script, and finally a sudo binary shell-escape.

---

## Reconnaissance

An `nmap` scan showed SSH (22) and FTP (21) among the open services. "Anonymous access" in the brief pointed directly at FTP, which accepted the `anonymous` login. The FTP share contained a `README` describing a "recon pipeline": files placed in an `incoming/` directory are *processed automatically on arrival*, with invalid formats ignored.

That single line described the first trust boundary: a process automatically executing files that an anonymous user can upload.

---

## Hop 1 — anonymous → recon_user

**Mechanism: auto-executed upload directory.**

A shell script was uploaded into the FTP `incoming/` directory. The pipeline's watcher executed it automatically as `recon_user`, and the script's reverse shell connected back to a listener, yielding a shell as `recon_user`.

*Root cause:* untrusted, user-supplied files are executed automatically without validation or sandboxing.

---

## Hop 2 — recon_user → dev_user

**Mechanism: group-writable cron script + SUID binary.**

Enumeration as `recon_user` revealed group membership in `dev_user` (and `devops`). The pipeline's dev-backup stage was a script owned by `dev_user` but **group-writable**:

```
-rwxrwxr-x 1 dev_user dev_user  /opt/dev/backup.sh
```

Because the current user was in the `dev_user` group, the script could be edited. `pspy` (run without root) confirmed the trigger: an hourly cron executed `backup.sh` **as dev_user**.

A payload was appended that, when run as dev_user, created a SUID copy of the shell:

```
cp /bin/bash /tmp/devbash; chmod 4755 /tmp/devbash
```

After the cron fired, running the SUID shell with privileges preserved granted dev_user access:

```
/tmp/devbash -p     # euid=dev_user
```

*Root cause:* a privileged scheduled job executes a script that a lower-privileged group can modify. *Note on technique:* the SUID bit causes a program to run with the file owner's privileges; `bash -p` prevents bash from dropping those privileges on startup.

---

## Hop 3 — dev_user → monitor_user

**Mechanism: PATH hijack of a command called by a privileged script.**

`pspy` revealed a `healthcheck` script running **as monitor_user every ~5 seconds**, which called `ps` by its bare name (not an absolute path):

```
UID=1003  /bin/bash /usr/local/bin/healthcheck
UID=1003  ps aux
```

A planted file `/opt/dev/bin/ps` existed and was group-writable by `dev_user`. Because `healthcheck` called `ps` via `PATH` and `/opt/dev/bin` preceded the system paths in monitor_user's environment, replacing that file controlled what ran as monitor_user.

As `recon_user` this could not be exploited (the file could not be made executable — not the owner). After becoming `dev_user` (the owner), the file was made executable and replaced with a reverse-shell payload. Within one 5-second cycle, `healthcheck` executed it and a shell returned **as monitor_user**.

*Root cause:* a privileged script invokes an external command by relative name while a writable directory sits ahead of the system paths in `PATH`.

---

## Hop 4 — monitor_user → ops_user

**Mechanism: sudo-invoked script calling a writable helper.**

`sudo -l` as monitor_user showed:

```
(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

`deploy.sh` (runnable as ops_user without a password) called a helper by relative path:

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

The helper `/opt/app/deploy_helper.sh` was **owned by monitor_user** — the current user — so it could be freely rewritten. Its contents were replaced with a payload, and the sudo rule was used to run `deploy.sh` as ops_user, which executed the modified helper as ops_user, granting ops_user access.

*Root cause:* a sudo-permitted script executes a helper file that the calling (lower-privileged) user can modify.

*Note on technique:* a SUID shell produces a real/effective UID mismatch (`uid=monitor_user`, `euid=ops_user`), and `sudo` evaluates the **real** UID. A real-UID shell for ops_user was obtained before checking `sudo -l` again, so that ops_user's own sudo rights were shown rather than the previous user's.

---

## Hop 5 — ops_user → root

**Mechanism: sudo binary shell-escape (GTFOBins).**

`sudo -l` as ops_user revealed:

```
(root) NOPASSWD: /usr/bin/less
```

The pager `less` can spawn a shell from within its interface. Run as root, that shell is a root shell:

```
sudo /usr/bin/less /etc/profile
# inside less:
!/bin/bash
# → uid=0(root)
```

*Root cause:* a binary capable of spawning a subshell is permitted via passwordless sudo. This is a classic GTFOBins entry — many common binaries (`less`, `vi`, `find`, `awk`, `nmap`, `python`, …) can break out to a shell when run via sudo or with SUID.

---

## The Chain

```
anonymous ──(auto-run upload)──► recon_user
          ──(group-writable cron + SUID bash)──► dev_user
          ──(PATH hijack of ps in healthcheck)──► monitor_user
          ──(sudo deploy.sh → writable helper)──► ops_user
          ──(sudo less → !/bin/bash)──► root
```

Every hop shares one root cause: **a privileged stage trusts something a lower-privileged stage controls** — an uploaded file, a group-writable script, a `PATH`-resolved command, a writable helper, a sudo-permitted binary. No single misconfiguration was catastrophic alone; chained, they walked an anonymous user to root.

---

## Remediation (Defensive View)

- **Auto-processing uploads:** never execute user-supplied files; validate, sandbox, and run processing under a minimally-privileged, isolated account.
- **Group-writable privileged scripts:** scripts executed by scheduled jobs must not be writable by lower-privileged groups. Restrict ownership and permissions; audit `find / -perm -g=w` on anything run by cron.
- **PATH hijacking:** privileged scripts should call binaries by absolute path (`/usr/bin/ps`) and set a controlled `PATH`; no writable directory should precede system paths.
- **Sudo helper scripts:** a script granted via sudo must not call other scripts/files that the invoking user can modify, and should use absolute paths with a clean environment.
- **Sudo binary escapes:** never grant passwordless sudo on binaries that can spawn a shell or read/write arbitrary files (consult GTFOBins before writing any sudo rule). Grant the narrowest possible command.

---

## What I Learned

- **The same enumeration loop solves every hop:** on landing as a new user, check `sudo -l`, group membership, cron/scheduled jobs, SUID binaries, and any file the *next* user's process trusts. Repeating that loop methodically carried the whole chain.
- **`pspy` is invaluable for finding triggers without root** — it surfaced the hourly backup cron and the 5-second healthcheck that were the pivots for hops 2 and 3.
- **Understand real vs. effective UID.** A SUID shell leaves the real UID unchanged, and tools like `sudo` key off the real UID — which briefly showed the wrong user's sudo rights until a real-UID shell was obtained.
- **Know when a technique genuinely doesn't apply.** The `ps` hijack could not be completed as recon_user (couldn't set the execute bit); recognizing that and returning to it *after* becoming the file's owner avoided wasted effort.
- **`sudo -l` → GTFOBins is a reflex worth building.** Any binary in a sudo rule or with SUID should be checked against GTFOBins immediately; the final root hop was a textbook `less` escape.
