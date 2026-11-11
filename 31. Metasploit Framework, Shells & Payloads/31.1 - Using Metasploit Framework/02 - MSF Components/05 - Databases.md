# Metasploit Framework: Databases

---

## Table of Contents

1. [Why Metasploit needs a database](#1-why-metasploit-needs-a-database)
2. [Setting up PostgreSQL](#2-setting-up-postgresql)
3. [Initializing the Metasploit database](#3-initializing-the-metasploit-database)
4. [Troubleshooting init errors](#4-troubleshooting-init-errors)
5. [Connecting and checking status](#5-connecting-and-checking-status)
6. [Database help menu](#6-database-help-menu)
7. [Workspaces: organizing your assessments](#7-workspaces-organizing-your-assessments)
8. [Importing scan results](#8-importing-scan-results)
9. [Scanning directly from msfconsole with db_nmap](#9-scanning-directly-from-msfconsole-with-db_nmap)
10. [Backing up your data](#10-backing-up-your-data)
11. [Hosts, Services, Credentials, and Loot](#11-hosts-services-credentials-and-loot)
12. [Cheat sheet](#12-cheat-sheet)

---

## 1. Why Metasploit needs a database

Picture assessing a single complex machine, let alone an entire network: open ports, services, credentials found, vulnerabilities confirmed, loot collected. Keep that in your head (or in scattered notes) across dozens of hosts, and things get messy fast.

**Databases solve this.** msfconsole has built-in support for **PostgreSQL**, which gives you:

- Quick, direct access to all your scan results
- The ability to **import/export** data to and from third-party tools
- The ability to **auto-fill exploit module options** (like `RHOSTS`) straight from stored findings, instead of retyping IPs by hand

**Analogy:** think of it as the difference between keeping notes on sticky notes scattered around your desk versus keeping them in a searchable, organized spreadsheet that other tools can also read from and write to.

---

## 2. Setting up PostgreSQL

Before Metasploit's database features work, PostgreSQL (the underlying database engine) needs to be running.

### Check status

```bash
$ sudo service postgresql status

● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; disabled; vendor preset: disabled)
     Active: active (exited) since Fri 2022-05-06 14:51:30 BST; 3min 51s ago
    Process: 2147 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 2147 (code=exited, status=0/SUCCESS)
        CPU: 1ms
```

### Start it if it's not running

```bash
$ sudo systemctl start postgresql
```

---

## 3. Initializing the Metasploit database

Once PostgreSQL is running, Metasploit needs its **own** database created and configured inside it. The `msfdb init` command does this.

```bash
$ sudo msfdb init

[+] Starting database
[+] Creating database user 'msf'
[+] Creating databases 'msf'
[+] Creating databases 'msf_test'
[+] Creating configuration file '/usr/share/metasploit-framework/config/database.yml'
[+] Creating initial database schema
```

**What's happening here, step by step:**

1. A dedicated database **user** called `msf` is created.
2. Two databases are created: `msf` (your real working data) and `msf_test` (used for testing).
3. A **configuration file** is written so Metasploit knows how to connect automatically in future sessions.
4. The initial **schema** (the table structure the database needs) is created.

> This is what a clean, successful first-time initialization looks like. In practice, you may hit an error the first time — see the next section.

---

## 4. Troubleshooting init errors

Sometimes `msfdb init` fails partway through because Metasploit (or one of its dependencies) is out of date:

```bash
$ sudo msfdb init

[i] Database already started
[+] Creating database user 'msf'
[+] Creating databases 'msf'
[+] Creating databases 'msf_test'
[+] Creating configuration file '/usr/share/metasploit-framework/config/database.yml'
[+] Creating initial database schema
rake aborted!
NoMethodError: undefined method `without' for #<Bundler::Settings:0x000055dddcf8cba8>
Did you mean? with_options

<SNIP>
```

### How to fix it

1. **Update Metasploit** first:

   ```bash
   $ sudo apt update && sudo apt install metasploit-framework
   ```

2. **Try initializing again.** If the database was actually created despite the error, you'll now see:

   ```bash
   $ sudo msfdb init

   [i] Database already started
   [i] The database appears to be already configured, skipping initialization
   ```

3. **Double-check with `msfdb status`:**

   ```bash
   $ sudo msfdb status

   ● postgresql.service - PostgreSQL RDBMS
        Active: active (exited) since Mon 2022-05-09 15:19:57 BST; 35min ago

   COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
   postgres 2458 postgres    5u  IPv6  34336      0t0  TCP localhost:5432 (LISTEN)
   postgres 2458 postgres    6u  IPv4  34337      0t0  TCP localhost:5432 (LISTEN)

   [+] Detected configuration file (/usr/share/metasploit-framework/config/database.yml)
   ```

   If this shows an active PostgreSQL service and a detected configuration file, the database is actually fine despite the earlier error message.

### If the password won't reset / database state is stuck

Sometimes you need a harder reset. In that case:

```bash
$ msfdb reinit
$ cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
$ sudo service postgresql restart
$ msfconsole -q

msf6 > db_status

[*] Connected to msf. Connection type: PostgreSQL.
```

This reinitializes the database, re-copies the config file into Metasploit's user config directory (`~/.msf4/`), restarts PostgreSQL, and confirms the connection from inside `msfconsole`.

---

## 5. Connecting and checking status

Once everything's initialized, you can launch `msfconsole` and connect to the database **in one step** using `msfdb run`:

```bash
$ sudo msfdb run

[i] Database already started
       =[ metasploit v6.1.39-dev                          ]
+ -- --=[ 2214 exploits - 1171 auxiliary - 396 post       ]
+ -- --=[ 616 payloads - 45 encoders - 11 nops            ]
+ -- --=[ 9 evasion                                       ]

msf6>
```

This is generally the preferred way to start a Metasploit session that needs database access — it saves you a separate `db_connect` step.

### Verifying the connection from inside msfconsole

```
msf6 > db_status

[*] Connected to msf. Connection type: postgresql.
```

---

## 6. Database help menu

Metasploit has an integrated help reference just for database commands:

```
msf6 > help database

Database Backend Commands
=========================

    Command           Description
    -------           -----------
    db_connect        Connect to an existing database
    db_disconnect     Disconnect from the current database instance
    db_export         Export a file containing the contents of the database
    db_import         Import a scan result file (filetype will be auto-detected)
    db_nmap           Executes nmap and records the output automatically
...<SNIP>...
```

Keep this in mind whenever you forget a command's exact syntax — `help database` is faster than searching online.

---

## 7. Workspaces: organizing your assessments

**Workspaces** are to Metasploit's database what **folders** are to a filesystem. They let you keep different engagements (or different targets within an engagement) completely separate — by IP, subnet, network, or domain.

### Viewing the current workspace

```
msf6 > workspace

* default
```

The `*` marks which workspace is **currently active**. Fresh installs start with a single workspace called `default`.

### Creating and switching to a new workspace

```
msf6 > workspace -a Target_1

[*] Added workspace: Target_1
[*] Workspace: Target_1


msf6 > workspace Target_1

[*] Workspace: Target_1


msf6 > workspace

  default
* Target_1
```

Notice the `*` has now moved to `Target_1` — that's the active workspace. Anything you scan, import, or store from here on will land inside `Target_1`, not `default`.

### Full workspace command reference

```
msf6 > workspace -h

Usage:
    workspace                  List workspaces
    workspace -v               List workspaces verbosely
    workspace [name]           Switch workspace
    workspace -a [name] ...    Add workspace(s)
    workspace -d [name] ...    Delete workspace(s)
    workspace -D               Delete all workspaces
    workspace -r                Rename workspace
    workspace -h               Show this help information
```

**Analogy:** if you're juggling multiple client engagements, think of each workspace as a separate client folder — switching workspaces is like opening a different folder so you don't accidentally mix up one client's findings with another's.

---

## 8. Importing scan results

If you already have scan data from a tool like Nmap, you don't need to redo the scan — you can **import** the results directly into your active workspace.

> **Format tip:** `db_import` strongly prefers **XML** output (e.g. Nmap's `-oX` format) over plain text, since XML is structured and easy for Metasploit to parse reliably.

### Example: a stored Nmap scan (plain text, for reference)

```bash
$ cat Target.nmap

Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-17 20:54 UTC
Nmap scan report for 10.10.10.40
Host is up (0.017s latency).
Not shown: 991 closed ports
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
<SNIP>
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Importing the equivalent XML file

```
msf6 > db_import Target.xml

[*] Importing 'Nmap XML' data
[*] Import: Parsing with 'Nokogiri v1.10.9'
[*] Importing host 10.10.10.40
[*] Successfully imported ~/Target.xml
```

### Confirming the import worked

```
msf6 > hosts

Hosts
=====

address      mac  name  os_name  os_flavor  os_sp  purpose  info  comments
-------      ---  ----  -------  ---------  -----  -------  ----  --------
10.10.10.40             Unknown                    device


msf6 > services

Services
========

host         port   proto  name          state  info
----         ----   -----  ----          -----  ----
10.10.10.40  135    tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  139    tcp    netbios-ssn   open   Microsoft Windows netbios-ssn
10.10.10.40  445    tcp    microsoft-ds  open   Microsoft Windows 7 - 10 microsoft-ds workgroup: WORKGROUP
10.10.10.40  49152  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49153  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49154  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49155  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49156  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49157  tcp    msrpc         open   Microsoft Windows RPC
```

The imported data is now sitting in the database, queryable with `hosts` and `services` any time — no need to re-run or re-read the original scan file again.

---

## 9. Scanning directly from msfconsole with db_nmap

You don't have to leave msfconsole to run Nmap at all. The **`db_nmap`** command runs an actual Nmap scan and **automatically records the results into the current workspace** — no manual import step needed.

```
msf6 > db_nmap -sV -sS 10.10.10.8

[*] Nmap: Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-17 21:04 UTC
[*] Nmap: Nmap scan report for 10.10.10.8
[*] Nmap: Host is up (0.016s latency).
[*] Nmap: Not shown: 999 filtered ports
[*] Nmap: PORT   STATE SERVICE VERSION
[*] Nmap: 80/TCP open  http    HttpFileServer httpd 2.3
[*] Nmap: Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
[*] Nmap: Service detection performed. Please report any incorrect results at https://nmap.org/submit/
[*] Nmap: Nmap done: 1 IP address (1 host up) scanned in 11.12 seconds
```

`db_nmap` accepts the **same flags** as normal Nmap (`-sV` for version detection, `-sS` for a SYN scan, etc.) — it's really just Nmap with automatic database recording bolted on.

### Checking the combined results

Now both the imported host and the freshly scanned host show up together:

```
msf6 > hosts

Hosts
=====

address      mac  name  os_name  os_flavor  os_sp  purpose  info  comments
-------      ---  ----  -------  ---------  -----  -------  ----  --------
10.10.10.8              Unknown                    device
10.10.10.40             Unknown                    device


msf6 > services

Services
========

host         port   proto  name          state  info
----         ----   -----  ----          -----  ----
10.10.10.8   80     tcp    http          open   HttpFileServer httpd 2.3
10.10.10.40  135    tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  139    tcp    netbios-ssn   open   Microsoft Windows netbios-ssn
10.10.10.40  445    tcp    microsoft-ds  open   Microsoft Windows 7 - 10 microsoft-ds workgroup: WORKGROUP
10.10.10.40  49152  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49153  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49154  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49155  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49156  tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  49157  tcp    msrpc         open   Microsoft Windows RPC
```

**Key benefit:** this means every target you scan while connected to the database is **automatically** accumulated into one organized view — you never have to manually stitch together separate scan files later.

---

## 10. Backing up your data

Databases can fail or get corrupted — always back up before ending a session, in case PostgreSQL has issues later.

### Exporting

```
msf6 > db_export -h

Usage:
    db_export -f <format> [filename]
    Format can be one of: xml, pwdump
[-] No output file was specified


msf6 > db_export -f xml backup.xml

[*] Starting export of workspace default to backup.xml [ xml ]...
[*] Finished export of workspace default to backup.xml [ xml ]...
```

| Format | Use case |
|---|---|
| `xml` | General-purpose, structured export — can be re-imported later |
| `pwdump` | Specifically for exporting captured password data |

This export can later be brought back in with `db_import`, the same way you'd import an Nmap scan. It's good practice to export a backup at natural checkpoints during a long engagement, not just at the very end.

---

## 11. Hosts, Services, Credentials, and Loot

Beyond `workspace` and the import/export commands, four tables make up the bulk of what you'll interact with day to day.

### Hosts

The `hosts` table auto-populates from scans (Nmap, Nessus, Nexpose, etc.) with IP addresses, hostnames, and detected OS info. You can also **manually add** hosts, and customize how the table displays (add comments, edit fields, etc.).

```
msf6 > hosts -h

Usage: hosts [ options ] [addr1 addr2 ...]

OPTIONS:
  -a,--add          Add the hosts instead of searching
  -d,--delete       Delete the hosts instead of searching
...<SNIP>...
```

### Services

The `services` table works the same way, but for discovered **services** (ports, protocols, service names/versions) on those hosts.

```
msf6 > services -h

Usage: services [-h] [-u] [-a] [-r <proto>] [-p <port1,port2>] [-s <name1,name2>] [-o <filename>] [addr1 addr2 ...]

  -a,--add          Add the services instead of searching
  -d,--delete       Delete the services instead of searching
  -c <col1,col2>    Only show the given columns
  -h,--help         Show this help information
  -s <name>         Name of the service to add
  -p <port>         Search for a list of ports
  -r <protocol>     Protocol type of the service being added [tcp|udp]
  -u,--up           Only show services which are up
  -o <file>         Send output to a file in csv format
  -O <column>       Order rows by specified column number
  -R,--rhosts       Set RHOSTS from the results of the search
  -S,--search       Search string to filter by
  -U,--update       Update data for existing service

Available columns: created_at, info, name, port, proto, state, updated_at
```

> **Handy feature:** the `-R`/`--rhosts` flag lets you take the results of a `services` search and feed them straight back in as an `RHOSTS` value for a module — no manual copy-pasting of IP lists.

### Credentials

The `creds` command tracks any **usernames, passwords, hashes, or keys** you've discovered or captured during the assessment.

```
msf6 > creds -h

Usage - Listing credentials:
  creds [filter options] [address range]

Usage - Adding credentials:
  creds add uses the following named parameters.
    user      :  Public, usually a username
    password  :  Private, private_type Password.
    ntlm      :  Private, private_type NTLM Hash.
    Postgres  :  Private, private_type Postgres MD5
    ssh-key   :  Private, private_type SSH key, must be a file path.
    hash      :  Private, private_type Nonreplayable hash
    jtr       :  Private, private_type John the Ripper hash type.
    realm     :  Realm,
    realm-type:  Realm, realm_type (domain db2db sid pgdb rsync wildcard), defaults to domain.
```

**Example uses:**

```
# Add a username, password, and domain
creds add user:admin password:notpassword realm:workgroup

# Add just a username and password
creds add user:guest password:'guest password'

# Add an NTLM hash for a user
creds add user:admin ntlm:E2FC15074BF7751DD408E6B105741864:A1074A69B1BDE45403AB680504BBDD1A

# Add a user with an SSH private key
creds add user:sshadmin ssh-key:/path/to/id_rsa
```

Having credentials centrally stored means you can reuse them later (e.g. auto-filling `SMBUser`/`SMBPass` options) without digging back through notes.

### Loot

The `loot` command tracks files pulled from compromised systems — most commonly **hash dumps** (like `passwd`, `shadow`, or Windows SAM hashes), but it can hold any type of extracted artifact.

```
msf6 > loot -h

Usage: loot [options]
 Info: loot [-h] [addr1 addr2 ...] [-t <type1,type2>]
  Add: loot -f [fname] -i [info] -a [addr1 addr2 ...] -t [type]
  Del: loot -d [addr1 addr2 ...]

  -a,--add          Add loot to the list of addresses, instead of listing
  -d,--delete       Delete *all* loot matching host and type
  -f,--file         File with contents of the loot to add
  -i,--info         Info of the loot to add
  -t <type1,type2>  Search for a list of types
  -h,--help         Show this help information
  -S,--search       Search string to filter by
```

**Think of `loot` as your evidence locker** — it keeps a record of exactly what sensitive material was pulled from which host, which is invaluable both for continuing the attack chain (e.g. cracking a captured hash) and for writing the final report.

---

## 12. Cheat sheet

### Setup sequence

```bash
sudo service postgresql status      # check PostgreSQL is running
sudo systemctl start postgresql     # start it if needed
sudo msfdb init                     # create & configure the msf database
sudo msfdb run                      # launch msfconsole, auto-connected to the DB
```

### Database commands

| Command | Purpose |
|---|---|
| `db_status` | Confirm the current database connection |
| `db_connect` | Connect to an existing database |
| `db_disconnect` | Disconnect from the current database |
| `db_import <file>` | Import scan results (XML preferred) |
| `db_export -f <xml\|pwdump> <file>` | Export the current workspace's data |
| `db_nmap <nmap flags> <target>` | Run Nmap and auto-record results into the DB |

### Workspace commands

| Command | Purpose |
|---|---|
| `workspace` | List workspaces, `*` marks the active one |
| `workspace -a <name>` | Add a new workspace |
| `workspace <name>` | Switch to a workspace |
| `workspace -d <name>` | Delete a workspace |
| `workspace -D` | Delete all workspaces |
| `workspace -r` | Rename a workspace |

### Data tables

| Command | Tracks |
|---|---|
| `hosts` | Discovered hosts (IP, hostname, OS info) |
| `services` | Discovered services/ports on those hosts |
| `creds` | Captured or known credentials (passwords, hashes, keys) |
| `loot` | Extracted files/artifacts (hash dumps, etc.) |

### Big-picture takeaways

1. **Databases exist to keep complex engagements organized** — without one, tracking everything manually doesn't scale.
2. **PostgreSQL must be running first**, then `msfdb init` creates Metasploit's own database on top of it.
3. **`msfdb run`** is the convenient one-step way to launch msfconsole already connected to the database.
4. **Workspaces = folders.** Use a separate workspace per engagement/target to avoid mixing up data.
5. **`db_import` and `db_nmap`** are your two main ways to get scan data into the database — one for existing scan files, one for scanning live from inside msfconsole.
6. **Always back up with `db_export`** at reasonable checkpoints, not just at the end.
7. **`hosts`, `services`, `creds`, and `loot`** are the four core tables you'll query and add to throughout an assessment — and several options (like `set RHOSTS`, or module options) can pull directly from this stored data instead of being retyped.

---
