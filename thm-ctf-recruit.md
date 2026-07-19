# Recruit — TryHackMe

**Date:** 2026-07-19
**Difficulty:** Easy / Medium
**Category:** Web Application — Information Disclosure, SQL Injection, Privilege Escalation
**Skills used:** nmap, gobuster, Burp Suite, manual SQL injection, source-code review

> ⚖️ Performed in an authorized, in-scope lab environment (TryHackMe). All credentials, hashes, and flags recovered during this exercise have been redacted from this writeup.

---

## Overview

Recruit is a PHP recruitment portal that lets HR staff manage candidate applications and administrators oversee hiring. The goal was to assess it as an attacker would: map the app, abuse exposed functionality, gain a foothold, and escalate to administrator.

The box fell to a **chain** of two flaws. A CV-fetching endpoint allowed reading arbitrary source files inside the web root, which disclosed a hardcoded HR credential and the application's login logic. Logged in as HR, the candidate search feature was found to be SQL-injectable, which allowed reading the administrator's password directly from the database. Neither flaw alone was game-over; chained, they resulted in full administrative compromise.

---

## Reconnaissance

### Port scan

```
nmap -sV -sC -p- <target>
```

| Port | Service | Notes |
|------|---------|-------|
| 22/tcp | OpenSSH 8.2p1 (Ubuntu) | Potential destination if credentials were recovered |
| 53/tcp | ISC BIND 9.16.1 | DNS exposed — unusual; noted, not pursued |
| 80/tcp | Apache 2.4.41, PHP app | Primary target; page title "Recruit" |

Two things worth recording from the scan output:

- The application is **PHP-based** (`PHPSESSID` cookie, `.php` routes).
- nmap flagged that the **`PHPSESSID` cookie is missing the `HttpOnly` flag** — a low-severity finding recorded below.

### Content discovery

```
gobuster dir -u http://<target> -w .../common.txt -x bak,txt -t 20
```

Reading the **status codes** rather than just the names surfaced the interesting entries:

| Path | Code | Why it matters |
|------|------|----------------|
| `/sitemap.xml` | 200 | Freely lists the app's routes — read first |
| `/phpmyadmin` | 301 | Database admin panel exposed |
| `/mail` | 301 | Mail directory on a recruitment portal |
| `/index.php` | 200 | The login page / main app |
| `/assets`, `/javascript` | 301 | Static directories |

The `.ht*` and `server-status` **403** entries are Apache defaults and were disregarded as noise.

### Reading the sitemap

`sitemap.xml` disclosed the full route list, including two not yet seen:

- `api.php` — the CV-retrieval API documentation.
- `dashboard.php` — filed under **"Authenticated Pages"** → the escalation target.

Every URL used the hostname **`recruit.thm`**, indicating virtual-host routing, so this was added to `/etc/hosts` before continuing. The sitemap also contained a comment block hinting at internal documentation/logs and role-restricted data — an early confirmation that access control and roles would be central.

---

## Finding the Vulnerability

### The CV-fetch endpoint

The site's "Access API" page documented an endpoint:

```
/file.php?cv=<URL>
```

described as fetching candidate CVs from external HTTP/HTTPS URLs, with a note that "requests targeting restricted locations may be blocked."

Testing revealed the implementation did **not** match the docs. The **behaviour of each refusal** told the story:

| Input | Response | What it revealed |
|-------|----------|------------------|
| external `http://` URL | `Only local files are allowed` | A scheme check — remote URLs rejected |
| `file:///etc/passwd` | `Access denied` | A *different*, path-based check fired |
| `file:///var/www/html/config.php` | file contents returned | The app's own files were **not** blocked |

Two different error strings meant two different checks. The `file://` scheme was accepted; only certain *targets* were denied. The key insight: **the filter was protecting system files like `/etc/passwd`, but not the application's own source.** A developer writing a blocklist thinks about `/etc/passwd`, not their own `config.php`.

### Confirming the filter logic

Because the file-read worked, `file.php`'s own source could be read, which removed all guesswork:

```php
if (strpos($cv, 'file://') !== 0) { die('Only local files are allowed'); }
$filePath = str_replace('file://', '', $cv);
$realPath = realpath($filePath);                       // resolves ../ and symlinks
$allowedBase = '/var/www/html';
if ($realPath === false || strpos($realPath, $allowedBase) !== 0) {
    die('Access denied');
}
echo file_get_contents($realPath);
```

This confirmed:

- The scheme must be `file://`.
- `realpath()` runs **before** the base-directory check, so path-traversal (`../`) and encoding tricks are correctly defeated — the path is canonicalized first.
- Only files that resolve to a path starting with `/var/www/html` are served.

An important negative result: `index.php` includes `/var/www/db.php` (the database credentials, one level **outside** the web root). Because `realpath()` resolves `/var/www/html/../db.php` to `/var/www/db.php`, which fails the base-directory check, **`db.php` was provably unreachable through this endpoint.** Reading the filter's source made it possible to stop attacking a genuinely closed door and pivot instead of wasting time.

### What the source disclosed

Reading `config.php` and `index.php` gave up two things:

1. A **hardcoded HR password** stored directly in `config.php`, commented as "temporary."
2. The complete **login logic** in `index.php`:

```php
if ($username === "hr" && $password === $HR_PASSWORD) { ...hr session... }

if ($username === 'admin') {
    // SELECT password FROM users WHERE username = ?  (prepared statement)
    if ($dbPassword && $password === $dbPassword) { ...admin session... }
}
```

This revealed the HR username (`hr`), that the **admin password lives in the database** (`users` table, `password` column, plaintext), and — critically — that the **login form itself uses a prepared statement and is therefore NOT SQL-injectable.** That saved effort: there was no point attacking the login.

---

## Exploitation

### 1. Foothold — HR login

Logging in at `/` as `hr` with the password recovered from `config.php` granted an HR session and access to `dashboard.php`, which displayed a candidate table (ID, Name, Position, Status) with a search box. The HR flag was returned here.

> Flag: `THM{...redacted...}`

### 2. Escalation — SQL injection in the candidate search

The search request was intercepted in Burp:

```
GET /dashboard.php?search=<input>
```

Appending a single quote (`search=Alice'`) returned a visible database error:

```
SQL Error: You have an error in your SQL syntax ... near '%'' at line 1
```

This confirmed injection **and** leaked the query shape: the input is placed inside a `LIKE '%...%'` clause — meaning, unlike the login, this query concatenates user input rather than parameterizing it.

Knowing the results table renders **four columns**, a UNION-based payload was used to pull data from the `users` table into the visible output:

```
search=xyzz' UNION SELECT 1,username,password,4 FROM users-- -
```

- `xyzz` — matches no candidate, so only the injected row appears.
- `UNION SELECT 1,...,4` — four values to match the four columns.
- `-- -` — comments out the trailing `%'` the app appends.

This returned the administrator's username and plaintext password in the candidate table.

> Recovered admin credentials — **redacted.** (In a real engagement these would be delivered to the client in a restricted appendix and rotated immediately.)

<img width="655" height="180" alt="passwordcracked" src="https://github.com/user-attachments/assets/0e573e07-1b2d-4d49-b197-7e6c4dbfb650" />

### 3. Administrator access

Logging in at `/` as `admin` with the recovered password granted an administrator session, completing the objective.

> Flag: `THM{...redacted...}`

---

## The Fix (Defensive View)

**1. Information disclosure via `file.php` (High).**
Root cause: a user-supplied value is passed to `file_get_contents()` with only a blocklist-style guard. Even though traversal was handled well, exposing a file-read primitive to users is dangerous by design.
*Remediation:* don't let users specify filesystem paths. Map CV requests to opaque IDs resolved server-side, serve only from a dedicated, non-source directory, and use an allowlist of permitted files rather than a blocklist of forbidden ones.

**2. Hardcoded credentials in source (High).**
Root cause: `$HR_PASSWORD` stored in a web-served config file; the "temporary" comment is a classic "we'll fix it later" that never gets fixed.
*Remediation:* store secrets in environment variables or a secrets manager, never in web-root source. Rotate the exposed credential. Ensure source files can never be read as plaintext.

**3. SQL injection in candidate search (Critical).**
Root cause: the search query concatenates input into a `LIKE` clause. Notably, the **login** on the same app *correctly* used a prepared statement — the vulnerability is an **inconsistency**, not ignorance.
*Remediation:* use parameterized queries for **every** database call, without exception. The safe login pattern already in the codebase should be applied to the search query. This is the central lesson of the box: security must be applied consistently, or one missed query undoes the rest.

**4. `PHPSESSID` missing `HttpOnly` (Low / Informational).**
The session cookie is readable by JavaScript, increasing session-theft impact if any XSS exists.
*Remediation:* set `HttpOnly` (and `Secure`, `SameSite`) on session cookies.

---

## The Attack Chain

```
file.php file-read  ──►  config.php  ──►  HR password  ──►  HR dashboard
                     └──►  index.php  ──►  login logic + "admin pw is in DB, plaintext"
                                                                │
HR dashboard search  ──►  SQL injection  ──►  UNION read of users table  ──►  admin password  ──►  ADMIN
```

Each finding's payoff was **information that located the next finding**. Individually the file-read and SQLi are serious; chained, they produce full administrative takeover — which is why the overall risk is Critical even though the entry point looked minor.

---

## What I Learned

- **Read every refusal as evidence.** Two different error strings meant two different filters; that distinction is what pointed me at reading the app's own source instead of hammering `/etc/passwd`.
- **When you have file-read, read the code that makes decisions.** Reading `file.php` proved which paths were reachable and which weren't, and reading `index.php` told me the login was safe *before* I wasted time on it.
- **Attack the blocklist's blind spot, not its wall.** The filter guarded system files, not the app's own config — that reframe was the whole foothold.
- **Consistency is the vulnerability.** The same app parameterized its login but not its search. Real bugs often live in the one code path a developer forgot, not in a total absence of security.
- **Redact secrets by habit.** Proving the access matters; publishing the password does not.

Next time I'd stand up a callback listener earlier to characterize the fetcher before assuming its behaviour, and I'd note the column count of any results table the moment I see it, since it's needed for UNION injection later.
