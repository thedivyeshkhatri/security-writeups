# Command Injection — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-23  
**Category:** Web, Server-Side  
**Skills used:** Command injection (verbose & blind), shell operators, time-based detection  

> **TL;DR:** Studied OS command injection — how unsanitised user input passed to system-command
> functions lets an attacker run arbitrary OS commands. Covered verbose vs blind injection,
> detection via time delays and output redirection, and confirmed injection in the lab with
> `; whoami`. Falls under **Injection (A05:2025)** in the OWASP Top 10.

## Overview
When user input is passed into a system command without proper checks, an attacker can inject
additional commands alongside the legitimate one. This is **command injection**, a vulnerability
that allows arbitrary OS-level commands to be executed through a vulnerable application. Like
SQL injection, it's an **Injection** flaw — the root cause is untrusted input being treated as
code rather than data.

## Discovering Command Injection
Command injection happens because many programming languages provide **built-in functions** that
run commands directly on the underlying OS:
- **PHP:** `exec()`, `system()`, `shell_exec()`, `passthru()`
- **Python:** the `subprocess` module (and `os.system()`)
- **Node.js:** `child_process.exec()`

These functions aren't harmful by themselves. They become dangerous when **user-supplied input
is passed into them without validation or sanitisation.**

## Exploiting Command Injection
Applications that build system commands from user input can often be manipulated into running
unintended operations. The key is **shell operators**, which chain commands together:
- `;` — run the next command regardless
- `&&` — run the next command only if the first succeeds
- `&` — run in background / separate
- `|` — pipe output of one command into the next

Two types:
- **Verbose command injection** — the application displays the injected command's output
  directly in its response, so results are immediately visible.
- **Blind command injection** — the command runs on the server but no output is returned,
  making it harder to detect.

## Detecting Blind Command Injection
Since output isn't visible, you confirm injection indirectly:
- **Time delays** — `ping` and `sleep` are ideal. Injecting a `ping` with a set count and
  seeing the response time increase *in proportion* is a strong indicator the command ran.
- **Output redirection** — force output into a readable file, e.g. inject
  `; whoami > /var/www/html/output.txt` to write into the web root, then browse to
  `http://target.thm/output.txt` to read the result.
- **Out-of-band** — use `curl` (or similar) to make the server call back to a server you
  control; if the request arrives, the injected command executed.

## Hands-On: TryHackMe Lab
The lab had a vulnerable input field. Injecting a chained command confirmed injection was
possible:

```bash
; whoami
```

The application processed the injected command, which confirmed **command injection**. From
there I navigated to the provided path to capture the flag.

*(If you can add it: was it verbose or blind? What did `whoami` return — and could you have run
something further, like reading a file or getting a shell? Noting the potential impact
strengthens the writeup.)*

## The Fix (Defensive View)
The root cause is user input reaching a system shell. Defenses, strongest first:
- **Avoid calling the shell with user input at all** — the real fix. Use language APIs that do
  the task directly instead of shelling out (e.g. a library call rather than `system()`).
- **Use parameterised/array-form execution** — pass arguments as a list (e.g. Python's
  `subprocess.run([...], shell=False)`) so input can't be interpreted as shell syntax. This is
  the command-injection equivalent of parameterised SQL queries.
- **Strict input validation (allow-lists)** — permit only expected characters/values; reject
  shell metacharacters (`; & | > $` etc.) as defense-in-depth.
- **Least privilege** — run the web server as a low-privilege user so a successful injection
  achieves less.

## What I Learned
Learned how command injection works and what causes it — unsanitised input reaching OS-command
functions — and the difference between **verbose** and **blind** injection. The most useful part
was learning to detect *blind* injection, where there's no visible output: time-based payloads
(`ping`/`sleep`) and output redirection turn an invisible bug into a confirmable one. It also
connected back to my SQLi writeup — both are Injection flaws with the same fix pattern: keep
untrusted input as data, never let it become executable code.
