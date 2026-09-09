# SQL Injection — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-13  
**Category:** Web, Database  
**Skills used:** SQL Injection (error-based, UNION-based), information_schema enumeration  

> **TL;DR:** Worked through SQL injection fundamentals — comments, UNION-based
> extraction, and database enumeration via `information_schema` — then got hands-on
> practice exploiting error-based and UNION-based SQLi in the TryHackMe AttackBox.

## Overview
SQL Injection (SQLi) is one of the most well-known web application vulnerabilities,
listed under OWASP's **A05:2025 – Injection** category. It happens when an attacker is
able to manipulate the SQL queries that a web application sends to its database —
because user input is treated as part of the query structure rather than as plain data.

*(Note: Injection dropped from #3 in the 2021 Top 10 to #5 in 2025. That reflects new
categories emerging, not a drop in severity — it still affects nearly every tested app.)*

## Key Concepts
- **Comments** (`--`, `#`) let you truncate the rest of the original query, so the
  application ignores whatever came after your injection.
- **Common operators:** `UNION` (append your own rows to a result set), `LIKE` with
  wildcards (pattern matching), and `LIMIT` (control how many rows return).
- **String functions:** `group_concat()` collapses many rows into a single value, and
  `concat()` joins values together — both useful when the app only displays one result.
- **information_schema:** every server has this built-in database, which holds metadata
  about every other database on the server. Two tables are especially valuable:
  - `information_schema.tables` — lists every table. The `table_schema` column holds the
    database name, and `table_name` holds the table name.
  - `information_schema.columns` — lists every column. The `table_name` and `column_name`
    columns let you discover the structure of any table.

## Types of SQL Injection
- **In-band SQLi** — the results of the injection are visible directly in the web
  application's response (this includes error-based and UNION-based).
- **Blind SQLi** — the application does not display any query results or error messages,
  so data is inferred through true/false conditions or timing.
- **Out-of-band SQLi** — the attacker manipulates the database into making an external
  network request, exfiltrating data through a separate channel.

## The Fix (Defensive View)
The root cause is that user input is mixed with query code. Prevention methods, strongest
first:

- **Parameterized queries / prepared statements** — the real fix. Input can never be
  parsed as SQL structure, which closes the vulnerability at its source.
- **Least privilege** on the database account, so even a successful injection reaches less.
- **Input validation** (allow-lists) and **escaping user input** — useful defense-in-depth,
  but not a substitute for parameterized queries.
- **Web Application Firewalls (WAFs)** — an extra detection layer, not a primary fix.

## What I Learned
Learned the different types of SQLi and understood how error-based and UNION-based
injection work. I also learned the methods of prevention and why parameterized queries are
the true fix rather than just input validation or a WAF. Getting hands-on with SQLi in the
TryHackMe AttackBox is what made it click.
