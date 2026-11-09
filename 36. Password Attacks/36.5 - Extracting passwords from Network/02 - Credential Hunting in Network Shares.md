# Credential Hunting in Network Shares

## Table of Contents

- [Hunting from Windows](#hunting-from-windows)
  - [Snaffler](#snaffler)
  - [PowerHuntShares](#powerhuntshares)
- [Hunting from Linux](#hunting-from-linux)
  - [MANSPIDER](#manspider)
  - [NetExec](#NetExec)

---

## Why shared drives are such a common weak point

Almost every company has shared network folders — team drives, department shares, that one "IT" folder everyone has access to. They're genuinely useful for collaboration, but they're also where people quietly dump sensitive stuff they probably shouldn't: config files, deployment notes, old scripts with hardcoded passwords, spreadsheets someone made "just to keep track" of logins. For an attacker, a scan through these shares can be a surprisingly efficient way to find real, usable credentials.

---

## What to look for (quick refresher)

This builds on the same patterns covered in earlier sections — a few quick reminders tailored to shares specifically:

- **Keywords inside files:** `passw`, `user`, `token`, `key`, `secret`
- **File extensions commonly tied to credentials:** `.ini`, `.cfg`, `.env`, `.xlsx`, `.ps1`, `.bat`
- **"Interesting" filenames:** anything with `config`, `user`, `passw`, `cred`, or `initial` in the name
- **Domain-specific strings:** if you're targeting `EVILCORP.LOCAL`, searching for the literal string `EVILCORP\` can surface domain-account references
- **Localize your keywords:** if the target is a German company, searching for "Benutzer" (user) will likely turn up more than searching for "user" in English
- **Be strategic about which shares you scan:** scanning ten shares with thousands of files each takes real time. An IT department's share is a much higher-value target than a shared photo archive.

Before reaching for specialized tools, even a basic PowerShell one-liner can go a long way:
```powershell
Get-ChildItem -Recurse -Include *.ext \\Server\Share | Select-String -Pattern "password"
```

For anything beyond quick manual checks, though, purpose-built tools save a lot of time. Let's walk through the main ones — **Snaffler**, **PowerHuntShares**, **MANSPIDER**, and **NetExec**.

---

## Hunting from Windows

### Snaffler

[Snaffler](https://github.com/SnaffCon/Snaffler) is a C# tool that, when run from a domain-joined machine, automatically discovers every network share it can reach and scans them for files that look "interesting" — pattern-matching against things like password strings, sensitive filenames, and more.

```cmd
C:\Users\Public> Snaffler.exe -s
```

```
[Share] {Green}<\\DC01.evilcorp.local\IT>(R)
[Share] {Green}<\\DC01.evilcorp.local\SYSVOL>(R) Logon server share

[File] {Red}<KeepPassOrKeyInCode|...>(\\DC01.evilcorp.local\ADMIN$\Panther\unattend.xml)
<AdministratorPassword>*SENSITIVE*DATA*DELETED*</AdministratorPassword>
```

Notice it found `unattend.xml` — the exact same Windows deployment config file flagged as a high-value target in the earlier Windows credential-hunting guide. Snaffler color-codes its findings by how interesting they look (red = high priority), which helps triage results quickly.

**Useful options to narrow the search:**
- `-u` — pulls a list of real usernames from Active Directory and specifically searches for references to them across files (useful for spotting personal notes or credential dumps tied to real people)
- `-i` / `-n` — explicitly include or exclude specific shares from the scan, so you can focus effort where it matters

> **Reality check:** all of these tools generate a *lot* of output. Expect plenty of false positives — manual review of the flagged results is still a necessary step, not optional.

### PowerHuntShares

[PowerHuntShares](https://github.com/NetSPI/PowerHuntShares) is a PowerShell script alternative that — unlike Snaffler — doesn't require running from a domain-joined machine. Its standout feature is producing a clean **HTML report** at the end, making it much easier to browse results than scrolling through raw console output.

<img width="1035" height="571" alt="image" src="https://github.com/user-attachments/assets/d3c74692-5f48-42f4-96c4-0f5d6f6fe53f" />

```powershell
PS C:\Users\Public\PowerHuntShares> Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public
```

What it automates behind the scenes:
- Figures out the current domain
- Enumerates every domain computer
- Checks which ones are reachable (ping + port 445 open)
- Pulls the list of SMB shares from each
- Checks share **permissions** — specifically flagging shares with excessive or risky access levels
- Builds a timeline of when files were last written/accessed
- Generates the final HTML summary + detailed CSV files

```
[*] Getting a list of SMB shares from 1 computers
[*] - 11 SMB shares were found.
[*] Getting share permissions from 11 SMB shares
```

> **Heads-up:** in large environments, this can take **hours** to run — it's thorough, but not something you'd use for a quick spot-check.

---

## Hunting from Linux

### MANSPIDER

If you're working from a Linux attack box (or just don't have a domain-joined Windows machine handy), [MANSPIDER](https://github.com/blacklanternsecurity/MANSPIDER) lets you scan SMB shares remotely for files matching a given keyword. It's recommended to run it via its official Docker container to avoid dependency headaches.

```bash
$ docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider \
    10.129.234.121 -c 'passw' -u 'mendres' -p 'evilcorp2025!'
```

```
[+] Searching by file content: "passw"
[+] Matching files will be downloaded to /root/.manspider/loot
[+] 10.129.234.121: Successful login as "mendres"
```

- `-c 'passw'` — search file **contents** for this string
- `-u` / `-p` — the credentials used to authenticate to the share

Any matching files get automatically downloaded locally for review.

### NetExec

We've used **NetExec** throughout this module for authentication testing — it also doubles as a share-spidering tool via its `--spider` option.

```bash
$ nxc smb 10.129.234.121 -u mendres -p 'evilcorp2025!' --spider IT --content --pattern "passw"
```

```
[+] evilcorp.local\mendres:evilcorp2025!
[*] Started spidering
[*] Spidering .
```

- `--spider IT` — target a specific share (here, the `IT` share — exactly the kind of high-value target mentioned earlier)
- `--content --pattern "passw"` — search file contents for the given string, rather than just listing filenames

---

## Quick Recap

| Tool | Platform | Best for |
|------|-------------|-------------|
| Snaffler | Windows (domain-joined) | Fast, pattern-based scanning with color-coded risk flags |
| PowerHuntShares | Windows (no domain-join required) | Deep share permission auditing + readable HTML report |
| MANSPIDER | Linux (via Docker) | Remote content-keyword scanning of SMB shares |
| NetExec `--spider` | Linux | Quick content search, reusing a tool you're likely already running |

**Key idea:** Network shares are an easy target precisely because they're meant to be convenient — and convenience and security are often at odds. Automated tools can scan thousands of files across dozens of shares far faster than any human could, but the real skill is in choosing *which* shares to prioritize (IT and admin shares over generic company folders) and manually verifying the matches these tools surface, since high recall almost always comes with a fair number of false positives.
