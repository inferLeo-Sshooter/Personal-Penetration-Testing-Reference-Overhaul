# Brute Forcing

## Table of Contents

1. [What is brute forcing?](#what-is-brute-forcing)
2. [How it works, step by step](#how-it-works-step-by-step)
   - [Dictionary attacks: exploiting predictable humans](#dictionary-attacks-exploiting-predictable-humans)
   - [Hybrid attacks: dictionary + brute force combined](#hybrid-attacks-dictionary--brute-force-combined)
   - [Credential stuffing: reusing stolen logins](#credential-stuffing-reusing-stolen-logins)
3. [Hydra: a fast login-cracking tool](#hydra-a-fast-login-cracking-tool)
4. [Medusa: another parallel login cracker](#medusa-another-parallel-login-cracker)
5. [Custom Wordlist](#Custom-Wordlist)

---

## What is brute forcing?

Think of a thief trying every key on a giant key ring until one fits. That's brute forcing: trying combination after combination of a password or key until something works.

Three things decide whether it succeeds:
- **How complex the password is** — longer and more varied passwords mean more combinations to try.
- **How much computing power the attacker has** — modern hardware can test billions of guesses per second.
- **What defenses are in place** — things like lockouts and CAPTCHAs slow attackers down.

## How it works, step by step

```
1. Start the tool
2. Generate a guess (based on rules: length, characters allowed)
3. Try that guess against the login/encrypted file
4. Did it work?
     - Yes → access granted, stop
     - No  → go back to step 2
```

## The different flavors of brute forcing

| Method | In plain terms | Example |
|---|---|---|
| Simple brute force | Try *everything* | All 4-letter combos: `aaaa`, `aaab`, `aaac`... |
| Dictionary attack | Try real words/known passwords | Try every line in `rockyou.txt` |
| Hybrid attack | Dictionary words + small tweaks | `Summer2023` → `Summer2024`, `Summer2023!` |
| Credential stuffing | Reuse leaked logins elsewhere | Email+password leaked from Site A, tried on Site B |
| Password spraying | Few common passwords, many accounts | Try `Password123` against 500 usernames |
| Rainbow table attack | Pre-computed hash lookups | Compare a stolen hash against a pre-built table instead of guessing live |
| Reverse brute force | One password, many usernames | Try a known-leaked password against every username on a site |
| Distributed brute force | Spread the work across machines | A botnet or cluster shares the guessing workload |

## Why pentesters use it

Ethical hackers use brute forcing when:
- Other methods (exploits, social engineering) didn't work.
- The target has weak password rules.
- They want to test one specific high-value account.

## The math behind it

```
Possible combinations = (size of character set) ^ (password length)
```

In other words, each extra character multiplies the guesswork, it doesn't just add to it.

| Password | Character set | Combinations |
|---|---|---|
| 6 lowercase letters | 26 | ~300 million |
| 8 lowercase letters | 26 | ~200 billion |
| 8 mixed-case letters | 52 | ~53 trillion |
| 12 chars, full keyboard | 94 | ~476 sextillion |

**Takeaway:** going from 6 to 8 characters, or adding uppercase/numbers/symbols, doesn't make cracking a little harder — it makes it *astronomically* harder.

## What makes a password weak or strong

**Strong password traits (per NIST guidance):**
- **Long** — 12+ characters is a good baseline.
- **Varied** — mixing character types helps, though length matters more these days.
- **Unique per site** — one leak shouldn't compromise everything.
- **Random** — not a word, name, or personal detail.

**Common weaknesses attackers exploit:**
- Short passwords (under 8 characters)
- Dictionary words or phrases
- Personal info (birthdays, pet names)
- Reused passwords across sites
- Predictable patterns like `qwerty`, `123456`, `p@ssw0rd`

## The problem with default credentials

Devices often ship with a built-in username and password that almost nobody changes. Attackers keep lists of these and try them first — often skipping brute forcing entirely.

```
Linksys Router    → admin / admin
Netgear Router    → admin / password
Cisco Router      → cisco / cisco
Hikvision DVR     → admin / 12345
Ubiquiti AP       → ubnt / ubnt
```

Even if the password gets changed, keeping the **default username** (like `admin` or `root`) still narrows the attacker's job down to just guessing the password.

---

## Dictionary attacks: exploiting predictable humans

Instead of trying every possible combination, a dictionary attack tries a curated list of likely passwords — real words, common passwords, leaked credentials. It works because people tend to pick memorable (and therefore guessable) passwords.

A **targeted** wordlist (e.g., gaming terms for a gaming site, or a company's name and employee names for a corporate target) beats a generic one.

### Brute force vs. dictionary attack

| | Dictionary Attack | Brute Force |
|---|---|---|
| Speed | Fast | Slow, especially on long passwords |
| Targeting | Can be customized to the victim | Not targeted at all |
| Best against | Weak/common passwords | Any password, given enough time |
| Fails against | Truly random passwords | Nothing — just takes forever |

### Common wordlists

| Wordlist | What it's for |
|---|---|
| `rockyou.txt` | Millions of real leaked passwords |
| `top-usernames-shortlist.txt` | Common usernames |
| `2023-200_most_used_passwords.txt` | The 200 most-used passwords |
| `default-passwords.txt` | Factory-default device credentials |

### Filtering a wordlist to match a password policy

Say a target requires: 8+ characters, 1 uppercase, 1 lowercase, 1 number. You can narrow a giant wordlist down to just the passwords that would even be *allowed*, using `grep`:

```bash
# Step 1: keep only passwords 8+ characters long
grep -E '^.{8,}$' wordlist.txt > step1.txt

# Step 2: keep only ones with an uppercase letter
grep -E '[A-Z]' step1.txt > step2.txt

# Step 3: keep only ones with a lowercase letter
grep -E '[a-z]' step2.txt > step3.txt

# Step 4: keep only ones with a number
grep -E '[0-9]' step3.txt > final.txt
```

Each step shrinks the list, so you end up testing far fewer — but far more *realistic* — guesses.

---

## Hybrid attacks: dictionary + brute force combined

People often reuse a base password and just tweak it slightly when forced to change it:

```
Summer2023  →  Summer2024  →  Summer2024!
```

A hybrid attack takes a dictionary word and automatically appends/modifies it (numbers, symbols, year increments) rather than guessing from scratch. It's faster than pure brute force and catches more passwords than a plain dictionary attack.

---

## Credential stuffing: reusing stolen logins

Many people reuse the same password everywhere. Credential stuffing takes a list of **username + password pairs** leaked from one breach and tries them on *other* sites.

```
1. Attacker gets leaked login list (from a breach, phishing, or malware)
2. Picks likely targets (email, banking, shopping sites)
3. Automates login attempts, mimicking normal user behavior
4. Any match = free access to that account
```

The fix is simple in theory, hard in practice: use a different password everywhere, and turn on multi-factor authentication.

---

## Hydra: a fast login-cracking tool

Hydra tries many username/password combos against a login service (SSH, FTP, web forms, databases, etc.), using multiple connections at once for speed.

### Basic syntax

```bash
hydra [login options] [password options] [attack options] [service options]
```

| Flag | Meaning |
|---|---|
| `-l` / `-L` | single username / file of usernames |
| `-p` / `-P` | single password / file of passwords |
| `-t` | number of parallel attempts |
| `-f` | stop after first success |
| `-s` | custom port |
| `-v` / `-V` | more detailed output |
| `service://server`| Specify service and server's address/hostname. (`hydra ssh://192.168.1.100`)|
| `/OPT`| Provide any additional options required by the target service. (`hydra http-get://example.com/login.php -m "POST:user=^USER^&pass=^PASS^"`)| 

### Hydra Services

|Hydra Service|	Description|	Example Command|
|-|-|-|
|`ftp`|	Used to brute-force login credentials for FTP services|	`hydra -l admin -P /path/to/password_list.txt ftp://192.168.1.100`|
|`ssh`|Targets SSH services to brute-force creds	|`hydra -l root -P /path/to/password_list.txt ssh://192.168.1.100`|
|`http-get/post`|Used to brute-force login credentials for HTTP web login forms using either GET or POST requests.	|`hydra -l admin -P /path/to/password_list.txt http-post-form "/login.php:user=^USER^&pass=^PASS^:F=incorrect"`|
|`smtp`|Attacks email servers by brute-forcing login credentials for SMTP	|`hydra -l admin -P /path/to/password_list.txt smtp://mail.server.com`|
|`pop3`| Targets email retrieval services to brute-force credentials for POP3 login	|`hydra -l user@example.com -P /path/to/password_list.txt pop3://mail.server.com`|
|`imap`|Used to brute-force credentials for IMAP services|`hydra -l user@example.com -P /path/to/password_list.txt imap://mail.server.com`|
|`mysql`|Brute-force login credentials for MySQL databases	|`hydra -l root -P /path/to/password_list.txt mysql://192.168.1.100`|
|`mssql`|Targets Microsoft SQL servers to brute-force database login credentials|`hydra -l sa -P /path/to/password_list.txt mssql://192.168.1.100`|
|`vnc`|	Brute-forces VNC services, used for remote desktop access.	|`hydra -P /path/to/password_list.txt vnc://192.168.1.100`|
|`rdp`|Targets Microsoft RDP services for remote login brute-forcing.	|`hydra -l admin -P /path/to/password_list.txt rdp://192.168.1.100`|


### Example: brute-forcing basic HTTP auth

```bash
hydra -l admin -P passwords.txt 127.0.0.1 http-get / -s 81
```
- This tries every password in `passwords.txt` against the user `admin` on port `81`.

### Example: attacking a custom login form with POST request

If a login form posts to `/login` with fields `username` and `password`, and shows "Invalid credentials" on failure:

```bash
hydra -L usernames.txt -P passwords.txt TARGET_IP \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials"
```

- `^USER^` and `^PASS^` get swapped out with each guess.
- `F=...` tells Hydra what text means "this attempt failed."
- You could use `S=302` or `S=Welcome` instead, to tell Hydra what text/status means "this attempt *succeeded*."

### Example: trying default SSH creds across many servers

```bash
hydra -l root -p toor -M targets.txt ssh
```
- Tries username `root`, password `toor`, against every IP listed in `targets.txt`.

### Example: RDP password Brute-Forcing

```bash
$ hydra -l administrator -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 192.168.1.100 rdp
```
- Use the username "`administrator`".
- Generate and test passwords ranging from 6 to 8 characters, using the specified character set.
- Target the RDP service on `192.168.1.100`.
- Employ the `rdp` module for the attack.

---

## Medusa: another parallel login cracker

Medusa works similarly to Hydra — fast, modular, supports many protocols.

### Basic syntax

```bash
medusa [target options] [credential options] -M module [module options]
```

| Flag | Meaning |
|---|---|
| `-h` / `-H` | single target / file of targets |
| `-u` / `-U` | single username / file of usernames |
| `-p` / `-P` | single password / file of passwords |
| `-M` | which module/protocol to use (ssh, ftp, http...) |
| `-m` | Additional parameters required by the chosen module (`-M http -m "POST /login.php HTTP/1.1\r\nContent-Length: 30\r\nContent-Type: application/x-www-form-urlencoded\r\n\r\nusername=^USER^&password=^PASS^" ...`) |
| `-t` | parallel attempts (`-t 4 ...`)|
| `-n` | custom port |
| `-f` or `-F` | Fast mode: Stop the attack after the first successful login is found, either on the current host (`-f`) or any host (`-F`) |
| `-v LEVEL` | Verbose output (up to 6), higher the level the more verbose the output (`-v 4 ...`) |

### Medusa Modules

Each module in Medusa is tailored to interact with specific authentication mechanisms, allowing it to send the appropriate requests and interpret responses for successful attacks. Below is a table of commonly used modules:

| Medusa Module | Description | Usage Example |
|---------------|------------------|-------------|
| FTP | Brute-forcing FTP login credentials | `medusa -M ftp -h 192.168.1.100 -u admin -P passwords.txt` |
| HTTP |  Brute-forcing login forms on web apps over HTTP (GET/POST) | `medusa -M http -h www.example.com -U users.txt -P passwords.txt -m DIR:/login.php -m FORM:username=^USER^&password=^PASS^` |
| IMAP |  Brute-forcing IMAP logins | `medusa -M imap -h mail.example.com -U users.txt -P passwords.txt` |
| MySQL | Brute-forcing MySQL database credentials | `medusa -M mysql -h 192.168.1.100 -u root -P passwords.txt` |
| POP3 | Brute-forcing POP3 logins | `medusa -M pop3 -h mail.example.com -U users.txt -P passwords.txt` |
| RDP |  Brute-forcing RDP logins | `medusa -M rdp -h 192.168.1.100 -u admin -P passwords.txt` |
| SSHv2 | Brute-forcing SSH logins | `medusa -M ssh -h 192.168.1.100 -u root -P passwords.txt` |
| Subversion (SVN) | Brute-forcing Subversion (SVN) repositories for version control. | `medusa -M svn -h 192.168.1.100 -u admin -P passwords.txt` |
| Telnet | Brute-forcing Telnet services | `medusa -M telnet -h 192.168.1.100 -u admin -P passwords.txt` |
| VNC | Brute-forcing VNC login credentials| `medusa -M vnc -h 192.168.1.100 -P passwords.txt` |
| Web Form | Brute-forcing login forms on websites using HTTP POST requests. | `medusa -M web-form -h www.example.com -U users.txt -P passwords.txt -m FORM:"username=^USER^&password=^PASS^:F=Invalid"` |


### Example: cracking SSH

```bash
medusa -h 192.168.0.100 -U usernames.txt -P passwords.txt -M ssh
```

### Example: checking for empty or default passwords

```bash
medusa -h 10.0.0.5 -U usernames.txt -e ns -M ssh
```
- `-e n` tries an empty password
- `-e s` tries the username as the password (e.g. user `admin` / password `admin`).

### Example: Targeting Multiple Web Servers with Basic HTTP Authentication

```bash
medusa -H web_servers.txt -U usernames.txt -P passwords.txt -M http -m GET
```
- Iterate through the list of web servers in `web_servers.txt`.
- Use the usernames and passwords provided.
- Employ the `http` module with the `GET` method to attempt logins.

---

## Custom Wordlist

For **Custom wordlist** and **rules filtering**, check section `36.1`

### Take away

Brute forcing works because:
- Weak/reused/default passwords are everywhere.
- Automation makes trying thousands of guesses fast and cheap.

It's defended against by:
- Long, random, unique passwords.
- Changing default usernames *and* passwords.
- Account lockouts, rate-limiting, CAPTCHAs.
- Multi-factor authentication.
