# Linux Authentication Process

## Table of content

- [Passwd file](#Passwd-file)
- [Shadow file](#Shadow-file)
- [Opasswd file](#Opasswd-file)
- [Cracking Linux credentials](#Cracking-Linux-credentials)

---

## The system behind the scenes: PAM

Linux handles logins through a flexible framework called **PAM** (Pluggable Authentication Modules). You don't usually see it directly, but it's working behind almost every authentication-related action — including something as simple as running `passwd` to change your own password.

The core module most relevant to us is `pam_unix.so` (sometimes `pam_unix2.so`), typically found at:
```
/usr/lib/x86_64-linux-gnu/security/
```

This module is responsible for reading and writing to the two files that actually store account and password info: `/etc/passwd` and `/etc/shadow`. (PAM also supports plenty of other authentication backends — LDAP, Kerberos, etc. — but those two files are our focus here.)

---

## Passwd file

`/etc/passwd` — **the user directory (world-readable!)**

This file lists basic info for **every user on the system**, and critically — anyone can read it, no special permissions needed. Each line is one user, with fields separated by colons:

```
john:x:1000:1000:,,,:/home/john:/bin/bash
```

| Field | Value | Meaning |
|-------|----------|------------|
| Username | `john` | |
| Password | `x` | Indicates the real hash lives elsewhere (see below) |
| User ID | `1000` | |
| Group ID | `1000` | |
| GECOS | `,,,` | Extra info (full name, phone, etc. — usually blank) |
| Home directory | `/home/john` | |
| Default shell | `/bin/bash` | |

### Why that "Password" field matters

On virtually every modern system, this field just contains the letter `x` — a flag meaning "the actual hash is stored in `/etc/shadow` instead, not here." That's by design: since `/etc/passwd` is readable by everyone, storing real hashes in it would let any user on the box try to crack every other user's password.

On **very old** systems, though, the real hash might still be sitting directly in this world-readable file — an easy win if you ever encounter it.

### A rarer but serious misconfiguration

If `/etc/passwd` is ever **writable** by a low-privileged user (a permissions mistake, sometimes caused by an inexperienced admin granting broad write access to `/etc` for an application and forgetting to scope it down), you can literally delete the password field for `root` entirely:

```bash
$ head -n 1 /etc/passwd
root::0:0:root:/root:/bin/bash
```

With the password field left empty, `root` requires **no password at all**:

```bash
$ su
root#
```

Rare, but devastating when it happens — always worth checking file permissions on `/etc/passwd` just in case.

---

## Shadow file

`/etc/shadow` — **where the real hashes live**

Because exposing password hashes to every user on the box is risky, Linux moved actual hash storage into `/etc/shadow` — a file **only readable by root/admin users**. If a user listed in `/etc/passwd` has no matching entry here, that account is effectively invalid and can't log in.

Format (9 colon-separated fields):

```
john:$y$j9T$3QSBB6CbHEu...SNIP...f8Ms:18955:0:99999:7:::
```

| Field | Example | Meaning |
|-------|------------|------------|
| Username | `john` | |
| Password hash | `$y$j9T$...` | The actual hashed password |
| Last changed | `18955` | Days since Jan 1, 1970 |
| Minimum age | `0` | Days before password can be changed again |
| Maximum age | `99999` | Days before password must be changed |
| Warning period | `7` | Days of warning before expiration |
| Inactivity period | *(blank)* | Grace period after expiration |
| Expiration date | *(blank)* | Account expiration date |
| Reserved | *(blank)* | Unused |

### Special password field values

- `!` or `*` → this account **can't log in with a Unix password at all** (but might still authenticate via Kerberos, SSH keys, etc.)
- *Empty* → **no password required** to log in

### Decoding the hash format itself

The password field follows a structured pattern:
```
$<id>$<salt>$<hashed>
```

The `<id>` tells you exactly which algorithm was used:

| ID | Algorithm |
|----|--------------|
| `1` | MD5 |
| `2a` | Blowfish |
| `5` | SHA-256 |
| `6` | SHA-512 |
| `sha1` | SHA1crypt |
| `y` | Yescrypt |
| `gy` | Gost-yescrypt |
| `7` | Scrypt |

Most modern distros (including current Debian) default to **yescrypt**, which is deliberately slow and resistant to cracking. But older systems might still be using weaker algorithms like MD5 — worth checking the ID before you assume cracking will be slow or fast.

---

## Opasswd file

`/etc/security/opasswd` — **old password history**

PAM can be configured to stop users from reusing recent passwords, and when it does, it keeps a record of their **previous** password hashes in:
```
/etc/security/opasswd
```

This file requires root access to read (unless misconfigured):

```bash
$ sudo cat /etc/security/opasswd
sam:1000:2:$1$HjFAfYTG$qNDkF0zJ3v8ylCOrKB0kt0,$1$kcUjWZJX$E9uMSmiQeRh4pAAgzuvkq1
```

Each comma-separated entry is one of that user's old password hashes. A couple of things worth noting:

- **Check the algorithm ID on each one** — older entries are often MD5 (`$1$`), which is *vastly* easier to crack than something like SHA-512.
- **Old passwords can reveal patterns.** People rarely pick a completely unrelated password every time they're forced to change it — cracking an old, weaker hash can reveal a pattern (`Summer2023!` → `Summer2024!`) that helps you guess their *current* password even if it uses a stronger hash you can't crack directly.

---

## Cracking Linux credentials

Once you have root access and can read `/etc/shadow`, the actual cracking workflow is simple. John the Ripper includes a handy helper tool called **unshadow** that merges `/etc/passwd` and `/etc/shadow` into a single file formatted for cracking tools.

```bash
$ sudo cp /etc/passwd /tmp/passwd.bak
$ sudo cp /etc/shadow /tmp/shadow.bak
$ unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
```

From here, you can crack with either John the Ripper directly, or feed the merged file to Hashcat:

```bash
$ hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```

(Mode `1800` corresponds to SHA-512 crypt — adjust the `-m` value to match whatever algorithm ID you spotted in the shadow file.)

> **Fun connection back to earlier material:** this exact scenario — a combined username + hash file, with real account info attached — is precisely what John the Ripper's **Single Crack mode** (covered earlier) was built for, since it can leverage the username/GECOS info to generate smarter guesses than a blind dictionary attack alone.

---

## Quick Recap

| File | Contains | Readable by |
|------|-------------|----------------|
| `/etc/passwd` | Usernames, UID/GID, shell, home dir | Everyone |
| `/etc/shadow` | Actual password hashes + aging policy | Root/admin only |
| `/etc/security/opasswd` | Previous password hashes (for reuse prevention) | Root/admin only |

**Key idea:** Linux deliberately splits "who exists" (world-readable `/etc/passwd`) from "what their password actually is" (root-only `/etc/shadow`) specifically to prevent exactly the kind of offline hash-grabbing we've covered throughout this guide. Root access collapses that separation entirely — once you can read `/etc/shadow`, the rest is the same crack-the-hash workflow you've already seen with Windows hashes, just pointed at a different file format.
