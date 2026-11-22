# Initial Enumeration for Windows Privilege Escalation

## Table of Contents
1. [Why We Escalate Privileges](#1-why-we-escalate-privileges)
2. [The Five Possible "Prizes" of Privilege Escalation](#2-the-five-possible-prizes-of-privilege-escalation)
3. [Key Data Points to Enumerate First](#3-key-data-points-to-enumerate-first)
4. [Looking at Running Processes (`tasklist /svc`)](#4-looking-at-running-processes-tasklist-svc)
5. [Environment Variables & the `PATH` Trap (`set`)](#5-environment-variables--the-path-trap-set)
6. [Detailed System Info (`systeminfo`)](#6-detailed-system-info-systeminfo)
7. [Checking Patches Two Different Ways (`wmic qfe` vs `Get-HotFix`)](#7-checking-patches-two-different-ways-wmic-qfe-vs-get-hotfix)
8. [Installed Programs (`wmic product` / `Get-WmiObject`)](#8-installed-programs-wmic-product--get-wmiobject)
9. [Network Connections (`netstat -ano`)](#9-network-connections-netstat--ano)
10. [Users, Groups, and Privileges](#10-users-groups-and-privileges)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Big-Picture Takeaways](#12-big-picture-takeaways)

---

## 1. Why We Escalate Privileges

When you first land on a Windows machine during an assessment, you usually only have a low-privilege user account. **Privilege escalation** means finding a way to gain more powerful access on that machine. Why bother? Because a fully compromised host can unlock:

- Sensitive files or file shares,
- The ability to capture network traffic (which often leaks more credentials),
- Credentials that let you move even further — potentially all the way up to **Domain Admin** in an Active Directory (AD) environment.

**Analogy:** Think of the low-privilege shell as being handed a visitor's badge that only opens the lobby door. Privilege escalation is finding a way to get a badge that opens the server room, the executive offices, or even the master key cabinet.

---

## 2. The Five Possible "Prizes" of Privilege Escalation

Depending on the system and what you find, escalating privileges can land you one of these levels of access:

| Target Account | What It Is |
|---|---|
| **NT AUTHORITY\SYSTEM** (`LocalSystem`) | The most powerful local account — even more powerful than a local administrator. Most Windows services run as this account. |
| **Built-in local Administrator** | Often disabled by security-conscious organizations, but frequently left on — and often the *same password* is reused across many machines. |
| **Another local Administrators group member** | Any account in this group gets the same power as the built-in administrator. |
| **A standard domain user who's in the local Administrators group** | A regular AD user account that happens to have admin rights on this one machine. |
| **A Domain Admin who's in the local Administrators group** | The jackpot — this account is powerful across the *entire* Active Directory domain, not just this one machine. |

This [Windows commands reference](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/windows-commands) is very handy for performing manual enumeration tasks.

---

## 3. Key Data Points to Enumerate First

The core idea: **enumeration is the foundation of privilege escalation.** Before you try any exploit or technique, gather information about the box. Automated scripts can do a lot of this for you, but it's essential to know how to do it manually too — you might land on a machine where you can't download tools (due to network restrictions, no internet access, or security products blocking you).

The three big things to understand early on:

- **OS name** (e.g. Windows 10, Windows Server 2016) — tells you what tools are likely available (like which PowerShell version) and whether public exploits exist for that OS.
- **Version/patch level** — specific Windows versions may have known public exploits. ⚠️ **Caution:** exploits targeting the OS itself can crash or destabilize a system — be very careful running these against anything resembling a production machine, and understand the exploit fully first.
- **Running services** — especially ones running as `NT AUTHORITY\SYSTEM` or another privileged account. A vulnerable or misconfigured service running with high privileges is often an easy win, because exploiting it inherits that service's privilege level.

---

## 4. Looking at Running Processes (`tasklist /svc`)

This command lists every running process (program) alongside the Windows *service(s)* tied to it, if any.

```
C:\evilcorp> tasklist /svc

Image Name                     PID Services
========================= ======== ============================================
System Idle Process              0 N/A
System                           4 N/A
smss.exe                       316 N/A
csrss.exe                      424 N/A
winlogon.exe                   612 N/A
services.exe                   664 N/A
lsass.exe                      672 KeyIso, SamSs, VaultSvc
svchost.exe                    776 BrokerInfrastructure, DcomLaunch, LSM,
                                   PlugPlay, Power, SystemEventsBroker
[... many more standard Windows background processes omitted — these are
     normal OS plumbing, not worth dwelling on during enumeration ...]

svchost.exe                   1988 W3SVC, WAS
svchost.exe                   1996 ftpsvc
FileZilla Server.exe          1140 FileZilla Server
inetinfo.exe                  1164 IISADMIN
MsMpEng.exe                   2136 WinDefend

[... more entries omitted ...]

TrustedInstaller.exe          1120 TrustedInstaller
tasklist.exe                  3596 N/A
```

**What matters here isn't memorizing every line — it's spotting what's *unusual*.**

| Normal "background noise" processes (safe to skim past) | Why they're not interesting |
|---|---|
| `smss.exe`, `csrss.exe`, `winlogon.exe`, `services.exe`, `svchost.exe`, `lsass.exe` | Standard, always-present core Windows processes. Learning to recognize these quickly helps you *filter them out* so the unusual stuff jumps out. |

| Process that stands out here | Why it matters |
|---|---|
| `FileZilla Server.exe` | A third-party FTP server — not a default Windows component. Worth checking its version for known vulnerabilities or checking if it allows anonymous FTP access (which can leak sensitive files). |
| `MsMpEng.exe` (Windows Defender's engine) | Confirms Defender is actually running as a process — useful for understanding what protections you'll need to work around. |

**Analogy:** Scanning a process list is like scanning a crowd for someone out of place — you don't study every ordinary person walking by, you train your eye to notice the one person wearing a shirt that doesn't match the dress code.

---

## 5. Environment Variables & the `PATH` Trap (`set`)

Environment variables are small pieces of configuration the operating system keeps handy — things like where to find files, where temp data goes, and what folders to search when running a command. You can list them with `set`.

```
C:\evilcorp> set

APPDATA=C:\Users\Administrator\AppData\Roaming
COMPUTERNAME=WINLPE-SRV01
ComSpec=C:\Windows\system32\cmd.exe
HOMEDRIVE=C:
HOMEPATH=\Users\Administrator
LOCALAPPDATA=C:\Users\Administrator\AppData\Local
LOGONSERVER=\\WINLPE-SRV01
OS=Windows_NT
Path=C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Users\Administrator\AppData\Local\Microsoft\WindowsApps;
PROCESSOR_ARCHITECTURE=AMD64
SystemDrive=C:
SystemRoot=C:\Windows
TEMP=C:\Users\ADMINI~1\AppData\Local\Temp\1
USERDOMAIN=WINLPE-SRV01
USERNAME=Administrator
USERPROFILE=C:\Users\Administrator
windir=C:\Windows
...<SNIP>...
```

### Why `PATH` deserves special attention

`PATH` is the list of folders Windows searches through, in order, to find a program when you type its name without a full file path.

**Analogy:** Think of `PATH` like a list of rooms a librarian checks, in order, when you ask for a book by title without telling them which shelf. If someone sneaks a fake book with the same title into the *first* room on the list, the librarian hands you the fake before ever reaching the real shelf.

Key facts:
- Windows checks your **Current Working Directory (CWD)** *first*, then walks through the folders listed in `PATH`, left to right.
- It's common to see things like Python or Java added to `PATH` — useful, but also potentially risky.
- **If a folder earlier in `PATH` is writable by your (low-privilege) user**, it opens the door to something called a **DLL injection/hijack** — effectively planting a malicious file with the name of something trusted so it gets picked up *before* the real, safe version.
- The position matters: a writable custom folder placed **before** `C:\Windows\System32` in the `PATH` is far more dangerous than one placed after it, because it gets checked — and trusted — first.

In the example above, nothing looks out of place: the standard system folders come first, nothing unusual is prepended.

### Other useful bits from `set`

- **`HOMEDRIVE` / `HOMEPATH`** — In many companies, the "home drive" is actually a network file share. Browsing that share might reveal other accessible folders — sometimes including things like an "IT directory" with a spreadsheet full of passwords (a very real, very common finding).
- **Roaming profiles** — Some organizations let users log into different machines and see the same desktop/files everywhere, which happens via these shared profile folders. One consequence: if a user plants a malicious file in `USERPROFILE\AppData\Microsoft\Windows\Start Menu\Programs\Startup`, it will **automatically run** the next time that user logs into *any* machine using that roaming profile — a sneaky persistence trick.

---

## 6. Detailed System Info (`systeminfo`)

This single command gives a big-picture snapshot: exact OS version, hardware, patch history, and more.

```
C:\evilcorp> systeminfo

Host Name:                 WINLPE-SRV01
OS Name:                   Microsoft Windows Server 2016 Standard
OS Version:                10.0.14393 N/A Build 14393
OS Configuration:          Standalone Server
Original Install Date:     3/24/2021, 3:46:32 PM
System Boot Time:          3/25/2021, 9:24:36 AM
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
Processor(s):              3 Processor(s) Installed.
Total Physical Memory:     6,143 MB
Domain:                    WORKGROUP
Hotfix(s):                 3 Hotfix(s) Installed.
                           [01]: KB3199986
                           [02]: KB5001078
                           [03]: KB4103723
Network Card(s):           2 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 IP address(es): 10.129.43.8 (+ IPv6 addresses)
                           [02]: vmxnet3 Ethernet Adapter
                                 IP address(es): 192.168.20.56 (+ IPv6 addresses)
Hyper-V Requirements:      A hypervisor has been detected. Features required
                           for Hyper-V will not be displayed.

...<SNIP>...
```


**How to put this to use:**

| Field | How to use it |
|---|---|
| `OS Name` / `OS Version` | Search for known public exploits matching this exact build. |
| `Hotfix(s)` (KBs installed) | Google each KB number to figure out roughly *when* this machine was last patched — a box with an old patch level is a strong hint that privilege escalation via a known exploit is realistic. |
| `System Boot Time` | If the system hasn't rebooted in 6+ months, it's a strong sign it also hasn't been patched in a while (some patches only apply after a reboot). |
| `Network Card(s)` | Confirms whether the machine is dual-homed (connected to more than one network) — but note: in most real enterprises, multi-network access is usually granted through firewall rules, *not* a second physical cable, so don't assume a second NIC always means an extra physical network to pivot into. |

> **Heads-up:** `systeminfo` doesn't always show hotfixes — it's possible for administrators to hide this information from non-administrator users. If that happens, you'll need the WMI/PowerShell methods in the next section instead.

---

## 7. Checking Patches Two Different Ways (`wmic qfe` vs `Get-HotFix`)

If `systeminfo` won't show patch history, you have two backup methods that query Windows' patch database (WMI, "Windows Management Instrumentation") directly.

**Method 1 — classic `wmic` command:**
```
C:\evilcorp> wmic qfe

Caption                                     CSName        Description      HotFixID   InstalledOn
http://support.microsoft.com/?kbid=3199986  WINLPE-SRV01  Update           KB3199986
https://support.microsoft.com/help/5001078  WINLPE-SRV01  Security Update  KB5001078  3/25/2021
http://support.microsoft.com/?kbid=4103723  WINLPE-SRV01  Security Update  KB4103723  3/25/2021
```

**Method 2 — PowerShell equivalent:**
```
PS C:\evilcorp> Get-HotFix | ft -AutoSize

Source       Description     HotFixID  InstalledBy                InstalledOn
------       -----------     --------  -----------                -----------
WINLPE-SRV01 Update          KB3199986 NT AUTHORITY\SYSTEM        11/21/2016 12:00:00 AM
WINLPE-SRV01 Update          KB4054590 WINLPE-SRV01\Administrator 3/30/2021 12:00:00 AM
WINLPE-SRV01 Security Update KB5001078 NT AUTHORITY\SYSTEM        3/25/2021 12:00:00 AM
WINLPE-SRV01 Security Update KB3200970 WINLPE-SRV01\Administrator 4/13/2021 12:00:00 AM
```

- **Worth noticing:** the two commands don't return exactly the same list of patches! `wmic qfe` lists `KB3199986`, `KB5001078`, `KB4103723`, while `Get-HotFix` lists `KB3199986`, `KB4054590`, `KB5001078`, `KB3200970`. This isn't a mistake in either tool — `wmic` and `Get-HotFix` can genuinely pull from slightly different underlying data sources and can return different results on the same machine.
- **Practical lesson:** don't trust just one enumeration method for patch levels — cross-check with more than one command (and ideally also `systeminfo`) before concluding a box is or isn't patched.

---

## 8. Installed Programs (`wmic product` / `Get-WmiObject`)

Knowing what third-party software is installed can point you toward:
- Exploits for specific vulnerable software versions,
- Stored credentials (tools like **LaZagne** can extract saved passwords/credentials from apps like FileZilla or PuTTY),
- Software running as a service that might itself be vulnerable or misconfigured.

**Classic command:**
```
C:\evilcorp> wmic product get name

Name
Microsoft Visual C++ 2019 X64 Additional Runtime - 14.24.28127
Java 8 Update 231 (64-bit)
VMware Tools
Java Auto Updater
<SNIP — list continues>
```

**PowerShell equivalent (also shows version numbers):**
```
PS C:\evilcorp> Get-WmiObject -Class Win32_Product | select Name, Version

Name                                                     Version
----                                                     -------
SQL Server 2016 Database Engine Shared                   13.2.5026.0
Microsoft ODBC Driver 13 for SQL Server                   13.2.5026.0
Java 8 Update 231 (64-bit)                                8.0.2310.11
<SNIP — list continues>
```

**Standout finding here:** the presence of a full SQL Server 2016 installation — a big, feature-rich piece of software that's historically been a frequent source of privilege-escalation paths (e.g., SQL Server service accounts with excess privileges). This is exactly the kind of "non-standard" software worth digging into further.

---

## 9. Network Connections (`netstat -ano`)

This command shows active and listening network connections — which ports are open, and which process (by PID) owns each one.

```
PS C:\evilcorp> netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:21             0.0.0.0:0              LISTENING       1096
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       840
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:1433           0.0.0.0:0              LISTENING       3520
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       968
```

**Analogy:** Think of this as a list of every door and window in the house (ports) that's currently unlocked, along with which "resident" (process/PID) is responsible for each one.

| Port | Commonly Associated With |
|---|---|
| 21 | FTP |
| 80 | HTTP (web server) |
| 135 | RPC (Remote Procedure Call) |
| 445 | SMB (file sharing) |
| 1433 | Microsoft SQL Server |
| 3389 | RDP (Remote Desktop) |

**Why this matters for privesc specifically:** some vulnerable services are only listening on `127.0.0.1` (localhost) — invisible to the outside network, but reachable *from inside the box once you have a shell there*. These "local-only" services can be some of the easiest privilege-escalation wins, because nobody outside expects them to be attackable at all.

---

## 10. Users, Groups, and Privileges

People, not just software, are often the weakest link. Good user/group enumeration might reveal a well-patched, well-secured machine... with an admin's home folder left wide open containing a plaintext `logins.xlsx` file — an instant win.

### 10.1 Who's Logged In Right Now?

```
C:\evilcorp> query user

 USERNAME              SESSIONNAME        ID  STATE   IDLE TIME  LOGON TIME
>administrator         rdp-tcp#2           1  Active          .  3/25/2021 9:27 AM
```
Shows who's currently logged on, whether they're active or idle, and since when. Useful for figuring out if someone else is actively using the box right now (important for staying stealthy on an evasive engagement) — and in some cases, a foothold for attacking that user directly.

### 10.2 Who Am I? (`echo %USERNAME%`)

```
C:\evilcorp> echo %USERNAME%

john
```
Always check this first. Sometimes you'll be surprised to already be `SYSTEM` or an equivalent powerful account without even trying.

### 10.3 What Can I Do? (`whoami /priv`)

```
C:\evilcorp> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```
This lists special Windows privileges your current account holds, separate from group membership. Some of these privileges (like `SeImpersonatePrivilege`, not shown in this particular example) can be abused directly to escalate to SYSTEM using well-known tools such as **Juicy Potato** — especially common when you're running as a service account.

### 10.4 What Groups Am I In? (`whoami /groups`)

```
C:\evilcorp> whoami /groups

GROUP INFORMATION
-----------------

Group Name                             Type             SID
====================================== ================ ============
Everyone                               Well-known group S-1-1-0
BUILTIN\Remote Desktop Users           Alias            S-1-5-32-555
BUILTIN\Users                          Alias            S-1-5-32-545
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192

...<SNIP>...
```

Group membership can grant inherited rights you wouldn't otherwise have — and in an AD environment, it's worth checking whether any of your groups carry weight across the whole domain, not just this machine.

### 10.5 All Local User Accounts (`net user`)

```
C:\evilcorp> net user

User accounts for \\WINLPE-SRV01

-------------------------------------------------------------------------------
Administrator            DefaultAccount           Guest
helpdesk                 john                     jordan
sarah                    secsvc
The command completed successfully.
```
**Why check this:** if you captured credentials for a user called `bob` elsewhere and later see a `bob_adm` account on this box, that's a strong hint to try credential reuse. It's also worth checking whether you can browse into other users' profile folders — sometimes scripts with hardcoded passwords or SSH keys are sitting in a Desktop or Downloads folder.

### 10.6 All Local Groups (`net localgroup`)

```
C:\evilcorp> net localgroup

Aliases for \\WINLPE-SRV01

-------------------------------------------------------------------------------
*Administrators
*Backup Operators
*Guests
*Hyper-V Administrators
*IIS_IUSRS
*Remote Desktop Users
*Remote Management Users
*Users
The command completed successfully.

...<SNIP>...
```

Unusual or custom groups here can hint at what the machine is used for, and may even reveal a misconfiguration — like every regular domain user accidentally being added to the local Administrators group.

### 10.7 Who's in a Specific Group? (`net localgroup administrators`)

```
C:\evilcorp> net localgroup administrators

Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
helpdesk
sarah
secsvc
The command completed successfully.
```
Occasionally, a group's "Comment" field itself contains leftover notes or even a password — worth a quick glance, even if it's a long shot. Here, we can see `helpdesk`, `sarah`, and `secsvc` are all full local administrators alongside the built-in `Administrator` account — any one of those three accounts would be a privilege-escalation win if their credentials can be obtained.

### 10.8 Password Policy (`net accounts`)

```
C:\evilcorp> net accounts

Force user logoff how long after time expires?:       Never
Minimum password age (days):                          0
Maximum password age (days):                          42
Minimum password length:                              0
Lockout threshold:                                    Never
Lockout duration (minutes):                           30
Computer role:                                        SERVER
The command completed successfully.
```

**Why this matters:** a `Minimum password length` of `0` and a `Lockout threshold` of `Never` means there's effectively **no limit on password guessing attempts** and **no enforced minimum password strength** — a very weak policy that makes brute-forcing or password-spraying attacks against local accounts far more viable.

---

## 11. Cheat Sheet

Many cheat sheets are available to help us, such as [this one](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md). 

| Goal | Command |
|---|---|
| List running processes + their services | `tasklist /svc` |
| List all environment variables (check `PATH`!) | `set` |
| Full system/OS/hardware/patch overview | `systeminfo` |
| List patches (method 1) | `wmic qfe` |
| List patches (method 2, PowerShell) | `Get-HotFix \| ft -AutoSize` |
| List installed software (method 1) | `wmic product get name` |
| List installed software + versions (PowerShell) | `Get-WmiObject -Class Win32_Product \| select Name, Version` |
| Show listening ports / active connections | `netstat -ano` |
| Who's currently logged in | `query user` |
| Who am I | `echo %USERNAME%` |
| What special privileges do I have | `whoami /priv` |
| What groups am I in | `whoami /groups` |
| List all local users | `net user` |
| List all local groups | `net localgroup` |
| List members of a specific group | `net localgroup <groupname>` |
| Check password policy | `net accounts` |

### Decision Flow: "Where Do I Look First After Getting a Shell?"

```
                ┌──────────────────────────┐
                │   Got a low-priv shell   │
                └─────────────┬────────────┘
                              │
                 Check: whoami /priv, whoami /groups
                              │
              ┌───────────────┴───────────────┐
              │                               │
   Dangerous privilege found?        Already in Administrators
   (e.g. SeImpersonatePrivilege)     or similarly powerful group?
              │                                │
              ▼                                ▼
   Try privilege-specific tool         You may already have what
   (e.g. Juicy Potato) → SYSTEM        you need — confirm access
              │                                │
              └───────────────┬────────────────┘
                              │ Neither of the above?
                              ▼
              Broaden the search:
       ┌───────────┬───────────┬────────────┬─────────────┐
       ▼           ▼           ▼            ▼             ▼
  tasklist /svc   set (PATH)  systeminfo   net user /     netstat
  (odd services)  (writable   (patch gaps) localgroup     -ano
                   folders)                (creds reuse,  (local-only
                                            misconfigs)    services)
```

---

## 12. Big-Picture Takeaways

1. Privilege escalation isn't always about reaching full SYSTEM access — sometimes a local admin account, or even a domain user who happens to be a local admin, is enough to meet your goal.
2. Thorough manual enumeration matters because you won't always be able to run automated tools — network restrictions, no internet, or security products can block them.
3. Learn to recognize "normal" Windows processes and services quickly, so unusual or third-party software (like a stray FTP server) jumps out immediately.
4. The `PATH` environment variable is a frequently-overlooked attack surface — a writable folder placed early in `PATH` can be a direct route to code execution as another user.
5. Never trust a single enumeration command for critical facts like patch level — `systeminfo`, `wmic qfe`, and `Get-HotFix` can each surface a slightly different picture, and administrators can sometimes hide hotfix info entirely.
6. A long-uptime machine (no reboot in months) is a strong hint it also hasn't been patched recently.
7. Installed third-party software (databases, FTP clients, dev tools) is a goldmine — both for known exploits and for stored credentials tools like LaZagne can extract.
8. Users and their habits (weak password policies, reused credentials, unprotected home directories) are frequently the easiest way in — don't focus only on technical misconfigurations.
9. Always check your current user, privileges, and group memberships *immediately* after landing a shell — you may already have more access than you think.
