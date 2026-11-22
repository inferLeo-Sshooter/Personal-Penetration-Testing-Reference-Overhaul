# Communication with Processes

## Table of Contents
1. [Why Process Communication Matters for Privilege Escalation](#1-why-process-communication-matters-for-privilege-escalation)
2. [Access Tokens — Your Digital ID Badge](#2-access-tokens--your-digital-id-badge)
3. [Enumerating Network Services (`netstat -ano`)](#3-enumerating-network-services-netstat--ano)
4. [Named Pipes — The Other Way Processes Talk](#4-named-pipes--the-other-way-processes-talk)
5. [Listing Named Pipes (PipeList & PowerShell)](#5-listing-named-pipes-pipelist--powershell)
6. [Checking Named Pipe Permissions with AccessChk](#6-checking-named-pipe-permissions-with-accesschk)
7. [Real-World Example: The WindscribeService Named Pipe](#7-real-world-example-the-windscribeservice-named-pipe)
8. [Cheat Sheet](#8-cheat-sheet)
9. [Big-Picture Takeaways](#9-big-picture-takeaways)

---

## 1. Why Process Communication Matters for Privilege Escalation

One of the richest hunting grounds for privilege escalation is simply **looking at what's running on the machine and how those programs talk to each other and the network.**

Here's a very common real-world scenario: you find a web server like IIS or XAMPP running on the box. You upload a small malicious web script (an "aspx" or "php" shell), and visiting that page gives you a shell running **as the user account the web server itself runs under**. That account usually isn't an administrator — but it often carries a special permission called `SeImpersonate`. That one permission alone is frequently enough to escalate straight to **SYSTEM**, using well-known tools nicknamed "Potato" attacks (Rogue Potato, Juicy Potato, Lonely Potato, etc.).

**Analogy:** Think of the web server account as a junior employee who isn't allowed into the executive suite directly — but happens to be holding a master keycard they're not supposed to have. The "Potato" tools are essentially tricks that get a more senior-looking process to hand its "elevator ride" to that keycard holder.

---

## 2. Access Tokens — Your Digital ID Badge

In Windows, every process and every thread carries something called an **access token**. This token is the operating system's record of *who you are* and *what you're allowed to do*.

**Analogy:** An access token is like an employee ID badge. When you log in (authenticate) and Windows checks your password against its security database, you're handed a badge. From that point on, every time you try to open a door (access a resource) or interact with another process, a copy of that badge gets shown to prove your identity and privilege level — without you having to re-type your password every time.

This concept matters a lot in privilege escalation because many attacks are really about **stealing or impersonating someone else's badge** rather than directly "hacking" anything.

---

## 3. Enumerating Network Services (`netstat -ano`)

The most common way outside users or other processes interact with a running program is over a **network socket** — think DNS, HTTP, SMB, and so on. The `netstat` command shows every active and listening connection, along with the **PID** (Process ID) responsible for it.

```
C:\evilcorp> netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:21             0.0.0.0:0              LISTENING       3812
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       836
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       936
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       5044
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       528
  [... several more high-numbered "ephemeral" ports omitted — these are
       normal, temporary, OS-assigned ports and rarely interesting ...]

  TCP    10.129.43.8:139        0.0.0.0:0              LISTENING       4
  TCP    10.129.43.8:3389       10.10.14.3:63191       ESTABLISHED     936
  TCP    10.129.43.8:49671      40.67.251.132:443      ESTABLISHED     1260
  [... more outbound HTTPS (443) connections to external IPs omitted —
       typically normal Windows/update/telemetry traffic ...]

  TCP    127.0.0.1:14147        0.0.0.0:0              LISTENING       3812

<SNIP — IPv6 equivalents of the same ports, plus some UDP listeners, omitted>
```

**The key enumeration trick:** look specifically for entries listening on a **loopback address** — `127.0.0.1` (IPv4) or `::1` (IPv6) — that are *not* also listening on the machine's real IP address or on the broadcast address (`0.0.0.0` / `[::]`).

**Why loopback-only services are dangerous (to the defenders, useful to you):** Software developers often assume "if it's only listening on localhost, nobody outside can reach it, so I don't need to secure it properly." That assumption only holds if nobody can get a shell *on* the box — which is exactly the situation you're in during privilege escalation. A locally-exposed, insecure admin interface is a gift.

**Analogy:** It's the difference between leaving your front door unlocked versus leaving the door to your home office unlocked, reasoning "well, you'd already have to be inside the house to reach that door anyway." True — but once an intruder *is* inside the house, that second unlocked door is now wide open too.

In the example above, the standout is `127.0.0.1:14147`, owned by PID `3812` — this turns out to be **FileZilla Server's administrative interface**. Connecting to this port could let you extract stored FTP passwords, or even create a new FTP share rooted at `C:\` running as the FileZilla Server service account (which may itself be running as Administrator).

---

## 4. Named Pipes — The Other Way Processes Talk

Besides network sockets, Windows processes can also talk to each other using **named pipes**. A pipe is essentially a small, temporary "file" that lives in memory — data gets written into one end, read out the other end, and then cleared.

**Analogy:** Think of a named pipe like a pneumatic tube system in an old-fashioned department store — one person puts a message capsule in at one station, and it shoots through to come out at another station, where it's picked up and the tube is then empty again, ready for the next message.

### Real-world example: how Cobalt Strike uses named pipes

Cobalt Strike (a widely-used red-team/attack-simulation tool) uses named pipes for nearly every command it runs (with the exception of something called "BOF"). The flow looks like this:

1. The "beacon" (the implant running on the compromised machine) creates a named pipe, e.g. `\\.\pipe\msagent_12`.
2. The beacon starts a new process and injects the command to run into that process, directing its output into the pipe.
3. The attacker's server reads whatever was written into the pipe and displays it.

**Why go through all this trouble?** If the command being run gets flagged by antivirus or crashes outright, only that *separate* process is affected — not the beacon itself, which keeps running. It's a form of isolation/insurance.

**A tradecraft note on detection:** attackers will often rename their pipes to masquerade as something innocent-looking — for example, naming the pipe something starting with `mojo` (associated with Google Chrome) instead of the Cobalt Strike default `msagent`. One memorable real-world catch: a `mojo`-prefixed pipe was found on a machine that didn't even have Chrome installed — a dead giveaway. In that particular case it turned out to be the client's own internal red team already active on the same box, not a genuine attacker — a slightly awkward (but educational) moment, since it meant an external consultant spotted the internal red team before the internal blue (defensive) team did.

### Technical details of named pipes

- There are two types: **named pipes** (have an actual name, e.g. `\\.\PipeName\ExampleNamedPipeServer`) and **anonymous pipes** (no name).
- Named pipes use a **client-server model**: whichever process *creates* the pipe is the "server," and whichever process connects to talk to it is the "client."
- Communication can be:
  - **Half-duplex** — one-way only; the client can write to the server, but not the reverse.
  - **Duplex** — two-way; both sides can write and read.
- A single named pipe can have **multiple simultaneous instances** — each new client connection gets its own independent data buffer, even though all instances share the same pipe name.

---

## 5. Listing Named Pipes (PipeList & PowerShell)

### Using PipeList (from the Sysinternals Suite)

We can use the tool [PipeList](https://learn.microsoft.com/en-us/sysinternals/downloads/pipelist) from the Sysinternals Suite to enumerate instances of named pipes.
```
C:\evilcorp> pipelist.exe /accepteula

PipeList v1.02 - Lists open named pipes
Copyright (C) 2005-2016 Mark Russinovich
Sysinternals - www.sysinternals.com

Pipe Name                                    Instances       Max Instances
---------                                    ---------       -------------
InitShutdown                                      3               -1
lsass                                             4               -1
ntsvcs                                            3               -1
scerpc                                            3               -1
epmapper                                          3               -1
eventlog                                          3               -1
spoolss                                           3               -1
wkssvc                                            4               -1
vmware-usbarbpipe                                 5               -1
srvsvc                                            4               -1
vmware-authdpipe                                  1                1

<...more standard Windows service pipes and per-socket "Winsock2\CatalogChangeListener" entries omitted — these are normal background plumbing...>
```

| Column | Meaning |
|---|---|
| `Instances` | How many active connections currently exist for this pipe |
| `Max Instances` | The maximum allowed instances; `-1` means unlimited |

### Using PowerShell to list Named Pipes (`gci` / `Get-ChildItem`)

```
PS C:\evilcorp>  gci \\.\pipe\

    Directory: \\.\pipe

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------       12/31/1600   4:00 PM              3 InitShutdown
------       12/31/1600   4:00 PM              4 lsass
------       12/31/1600   4:00 PM              3 ntsvcs
------       12/31/1600   4:00 PM              3 scerpc

    Directory: \\.\pipe\Winsock2

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------       12/31/1600   4:00 PM              1 Winsock2\CatalogChangeListener-34c-0

    Directory: \\.\pipe

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------       12/31/1600   4:00 PM              3 epmapper

<SNIP>
```
Same information as PipeList, just using a built-in PowerShell cmdlet instead of a downloaded tool — handy if you can't bring external binaries onto the box.

---

## 6. Checking Named Pipe Permissions with AccessChk

Just finding pipes isn't enough — you need to know **who's allowed to read from, write to, or fully control each one.** This is governed by a pipe's **DACL** (Discretionary Access Control List) — essentially a permissions list attached to the resource.

**Analogy:** If named pipes are pneumatic tubes, the DACL is the sign-in sheet at each tube station saying exactly who's allowed to drop a message in, who's allowed to take one out, and who's allowed to do both.

The [AccessChk](https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk) tool (also from Sysinternals) reads this out for you. Let's check the pipe used by `lsass` (Local Security Authority Subsystem Service — a core Windows process that handles authentication and holds sensitive credential material in memory):

```
C:\evilcorp> accesschk.exe /accepteula \\.\Pipe\lsass -v

Accesschk v6.12 - Reports effective permissions for securable objects
Copyright (C) 2006-2017 Mark Russinovich
Sysinternals - www.sysinternals.com

\\.\Pipe\lsass
  Untrusted Mandatory Level [No-Write-Up]
  RW Everyone
        FILE_READ_ATTRIBUTES
        FILE_READ_DATA
        FILE_WRITE_ATTRIBUTES
        FILE_WRITE_DATA
        SYNCHRONIZE
        READ_CONTROL
  RW NT AUTHORITY\ANONYMOUS LOGON
        FILE_READ_ATTRIBUTES
        FILE_READ_DATA
        FILE_WRITE_ATTRIBUTES
        FILE_WRITE_DATA
        SYNCHRONIZE
        READ_CONTROL
  RW APPLICATION PACKAGE AUTHORITY\Your Windows credentials
        FILE_READ_ATTRIBUTES
        FILE_READ_DATA
        FILE_WRITE_ATTRIBUTES
        FILE_WRITE_DATA
        SYNCHRONIZE
        READ_CONTROL
  RW BUILTIN\Administrators
        FILE_ALL_ACCESS
```

**How to read this — an important nuance:** at first glance it might look like `Everyone`, `ANONYMOUS LOGON`, and the `APPLICATION PACKAGE AUTHORITY` group all have broad read/write (`RW`) access, which could look alarming. But look closely at exactly *which* rights each group has:

- `Everyone`, `ANONYMOUS LOGON`, and `APPLICATION PACKAGE AUTHORITY` are limited to specific, narrower rights like reading/writing basic file *attributes* and *data* on the pipe object itself.
- Only `BUILTIN\Administrators` has `FILE_ALL_ACCESS` — meaning genuinely complete, unrestricted control over the pipe.

So the statement "only administrators have full access" specifically means **full (`FILE_ALL_ACCESS`) control** — not that other groups have zero access at all. This is a normal, expected, and safe configuration for something as sensitive as the `lsass` pipe.

You can also review **every** named pipe's permissions at once with:
```
C:\evilcorp> accesschk.exe /accepteula \pipe\*
```

---

## 7. Real-World Example: The WindscribeService Named Pipe

Here's a concrete case study showing why checking pipe permissions matters: the [WindscribeService Named Pipe Privilege Escalation](https://www.exploit-db.com/exploits/48021) (a real, documented vulnerability in the Windscribe VPN client's Windows service).

**Step 1 — Hunt for over-permissioned pipes.** You can search broadly for any named pipe that grants write access using:
```
C:\evilcorp> accesschk.exe -w \pipe\* -v
```

**Step 2 — Confirm the finding.** In this case, the `WindscribeService` pipe turned out to allow the `Everyone` group (i.e., **any authenticated user on the machine**) complete control:

```
C:\evilcorp> accesschk.exe -accepteula -w \pipe\WindscribeService -v

Accesschk v6.13 - Reports effective permissions for securable objects
Copyright (C) 2006-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

\\.\Pipe\WindscribeService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW Everyone
        FILE_ALL_ACCESS
```

This is a **misconfiguration**, not a safe default — unlike the `lsass` example above, here the `Everyone` group has `FILE_ALL_ACCESS`, the same unrestricted level normally reserved for Administrators.

**Why this is dangerous:** the `WindscribeService` runs as a privileged account (that's the whole point of a VPN service needing deep system access). If literally any logged-in user can write arbitrary data into that service's communication pipe with full control, they can potentially manipulate what the privileged service does — turning a low-privilege foothold into a path to **SYSTEM**.

**Analogy:** Imagine a bank vault (the privileged service) that only communicates with the outside world through a single pneumatic tube — and that tube's intake slot has no lock on it at all, letting any visitor to the building drop instructions straight into the vault's internal mail system.

---

## 8. Cheat Sheet

| Goal | Command |
|---|---|
| View all active/listening network connections + owning PID | `netstat -ano` |
| List named pipes (Sysinternals tool) | `pipelist.exe /accepteula` |
| List named pipes (built-in PowerShell) | `gci \\.\pipe\` |
| Check permissions on one specific named pipe | `accesschk.exe /accepteula \\.\Pipe\<name> -v` |
| Check permissions on *all* named pipes at once | `accesschk.exe /accepteula \pipe\*` |
| Find pipes that allow write access to anyone | `accesschk.exe -w \pipe\* -v` |

### Decision Flow: "Is This Named Pipe Exploitable?"

```
          ┌────────────────────────────────┐
          │ List named pipes (pipelist /   │
          │ gci \\.\pipe\)                 │
          └───────────────┬────────────────┘
                          │
          Is the pipe tied to 3rd-party / non-default
          software (e.g. a VPN client, backup agent)?
                          │
            ┌─────────────┴─────────────┐
            │ Yes                       │ No (core Windows pipe)
            ▼                           ▼
   Check permissions with         Still worth checking, but
   accesschk.exe -v               less likely to be misconfigured
            │                     (Windows core pipes are
            ▼                      usually locked down tightly)
   Does "Everyone" or a low-
   privilege group have
   FILE_ALL_ACCESS (full control)?
            │
    ┌───────┴────────┐
    │ Yes            │ No — only specific,
    ▼                │ limited rights (e.g. just
 Likely exploitable  │ FILE_READ_DATA) granted
 → investigate the   ▼
 service's behavior  Probably safe / expected —
 for a privesc path  move on to next enumeration step
```

---

## 9. Big-Picture Takeaways

1. Processes and how they communicate — whether over the network or through named pipes — are one of the richest sources of privilege-escalation opportunities.
2. Access tokens are Windows' way of tracking "who you are and what you're allowed to do" for every process and thread; many escalation techniques are fundamentally about hijacking or impersonating someone else's token.
3. When scanning `netstat` output, pay special attention to services listening **only** on `127.0.0.1`/`::1` — developers often leave these less secured, assuming (wrongly, once you have a shell) that nobody outside can reach them.
4. Named pipes are an alternate communication channel to network sockets, used heavily by legitimate Windows services *and* by attacker tools like Cobalt Strike.
5. A pipe's DACL (permissions list) tells you exactly who can read, write, or fully control it — having *some* access listed for a broad group (like `Everyone`) isn't automatically dangerous; what matters is whether that access includes full control (`FILE_ALL_ACCESS`).
6. Third-party services (VPN clients, backup tools, etc.) are generally far more likely to have misconfigured pipe permissions than core, heavily-scrutinized Windows components like `lsass`.
7. A single over-permissioned named pipe tied to a privileged service — as in the WindscribeService case — can be enough on its own to escalate a low-privilege foothold all the way to SYSTEM.
8. Small details (like a named pipe prefixed `mojo` on a machine with no Chrome installed) can reveal the presence of other actors — including, awkwardly, an organization's own internal red team.
