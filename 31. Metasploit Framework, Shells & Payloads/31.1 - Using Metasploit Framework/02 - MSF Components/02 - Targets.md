# Metasploit Framework: Targets

---

## Table of Contents

1. [What a "target" is](#1-what-a-target-is)
2. [`show targets`: in vs. out of a module](#2-show-targets-in-vs-out-of-a-module)
3. [Worked example: a one-target exploit](#3-worked-example-a-one-target-exploit)
4. [Worked example: a multi-target exploit](#4-worked-example-a-multi-target-exploit)
5. [Reading `info` before you do anything else](#5-reading-info-before-you-do-anything-else)
6. [Choosing Automatic vs. a specific target](#6-choosing-automatic-vs-a-specific-target)
7. [What makes targets different from each other](#7-what-makes-targets-different-from-each-other)
8. [Cheat sheet](#8-cheat-sheet)

---

## 1. What a "target" is

A **target** in Metasploit is a specific, named version of an operating system (and sometimes a specific application running on it) that a given exploit module has been built and tuned to work against.

Think of an exploit module as a lockpick, and the **target** setting as telling the lockpick exactly which lock model it's dealing with. The same general picking technique might need a slightly different motion depending on whether it's an old brass padlock or a modern deadbolt — same idea, different internals.

**Why this matters:** many exploits rely on very specific internal details of the OS/application (like memory addresses) that **change between versions**. Selecting the right target tells Metasploit exactly which set of those details to use.

---

## 2. `show targets`: in vs. out of a module

The `show targets` command behaves differently depending on **where** you run it.

### Outside any module (at the root menu)

```
msf6 > show targets

[-] No exploit module selected.
```

Makes sense: there's nothing to show targets *for* yet. You have to `use` a module first.

### Inside a selected exploit module

Once you've picked a module, `show targets` (or just `targets`) lists every OS/version combination that module knows how to handle.

---

## 3. Worked example: a one-target exploit

Picking back up the **MS17-010 (`ms17_010_psexec`)** example from previous section:

```
msf6 exploit(windows/smb/ms17_010_psexec) > options

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   RHOSTS     10.10.10.40      yes       The target host(s)
   RPORT      445              yes       The Target port (TCP)
   SHARE      ADMIN$           yes       The share to connect to
   ...

Payload options (windows/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address
   LPORT  4444              yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Automatic
```

Notice the **Exploit target** section at the bottom only lists **one** entry: `0 Automatic`. This particular exploit doesn't need you to pick a specific OS build — it works the same way regardless, so Metasploit only offers the automatic option.

> **Takeaway:** not every exploit needs manual target selection. Some are "one size fits all."

---

## 4. Worked example: a multi-target exploit

Now compare that to an exploit that genuinely needs to know which exact environment it's facing: **MS12-063**, a Use-After-Free vulnerability in Internet Explorer.

### Step 1: always start with `info`

Before touching a new module's options, the **first thing to do** is read its description with `info`. This is good practice for two reasons:

1. **You understand what the module actually does** before running it.
2. **You audit it for safety** — Metasploit modules can leave artifacts or have side effects, so knowing exactly what you're about to run protects both you and your client.

```
msf6 exploit(windows/browser/ie_execcommand_uaf) > info

       Name: MS12-063 Microsoft Internet Explorer execCommand Use-After-Free Vulnerability
     Module: exploit/windows/browser/ie_execcommand_uaf
   Platform: Windows
 Privileged: No
       Rank: Good
  Disclosed: 2012-09-14

Available targets:
  Id  Name
  --  ----
  0   Automatic
  1   IE 7 on Windows XP SP3
  2   IE 8 on Windows XP SP3
  3   IE 7 on Windows Vista
  4   IE 8 on Windows Vista
  5   IE 8 on Windows 7
  6   IE 9 on Windows 7

Check supported:
  No

Basic options:
  Name       Current Setting  Required  Description
  ----       ---------------  --------  -----------
  OBFUSCATE  false            no        Enable JavaScript obfuscation
  SRVHOST    0.0.0.0          yes       The local host to listen on
  SRVPORT    8080             yes       The local port to listen on
  SSL        false            no        Negotiate SSL for incoming connections
  URIPATH                     no        The URI to use for this exploit

Description:
  This module exploits a vulnerability found in Microsoft Internet
  Explorer (MSIE). When rendering an HTML page, the CMshtmlEd object
  gets deleted in an unexpected manner, but the same memory is reused
  again later in the CMshtmlEd::Exec() function, leading to a
  use-after-free condition...

References:
  https://cvedetails.com/cve/CVE-2012-4969/
  https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2012/MS12-063
```

### Reading this in plain English

- **It's a browser exploit, not a network-service exploit.** Instead of connecting directly to the target, it sets up a malicious web page (`SRVHOST`/`SRVPORT`) and waits for the victim's Internet Explorer to visit it.
- **"Use-after-free"** means the program frees up a chunk of memory, then accidentally uses it again later — attackers can exploit that gap to run their own code.
- Under **Available targets**, there are now **seven** entries, not one. This exploit genuinely behaves differently depending on the exact IE version and Windows version combination.
- **Check supported: No** means this particular module does **not** support the `check` command — you can't safely "test fire" it without actually attempting exploitation.

### Step 2: confirm with `options` and `show targets`

```
msf6 exploit(windows/browser/ie_execcommand_uaf) > options

Module options (exploit/windows/browser/ie_execcommand_uaf):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   OBFUSCATE  false            no        Enable JavaScript obfuscation
   SRVHOST    0.0.0.0          yes       The local host to listen on
   SRVPORT    8080             yes       The local port to listen on
   SSL        false            no        Negotiate SSL for incoming connections
   SSLCert                     no        Path to a custom SSL certificate
   URIPATH                     no        The URI to use for this exploit

Exploit target:

   Id  Name
   --  ----
   0   Automatic


msf6 exploit(windows/browser/ie_execcommand_uaf) > show targets

Exploit targets:

   Id  Name
   --  ----
   0   Automatic
   1   IE 7 on Windows XP SP3
   2   IE 8 on Windows XP SP3
   3   IE 7 on Windows Vista
   4   IE 8 on Windows Vista
   5   IE 8 on Windows 7
   6   IE 9 on Windows 7
```

### Step 3: pick a target

If you already know (from recon, user-agent sniffing, OSINT, etc.) exactly what the victim is running — say, **IE 9 on Windows 7** — you can lock the exploit onto that specific target using `set target <id>`:

```
msf6 exploit(windows/browser/ie_execcommand_uaf) > set target 6

target => 6
```

---

## 5. Reading `info` before you do anything else

Worth repeating as its own rule, since the source stresses it so heavily:

> **Best practice: run `info` on a module before you ever `run` or `exploit` it.**

Think of it like reading a medicine's label before taking it — the module might have side effects (artifacts left behind, specific requirements, unexpected behavior) that you want to know about *before* they surprise you in front of a client.

---

## 6. Choosing Automatic vs. a specific target

| Setting | What happens | When to use it |
|---|---|---|
| **`target 0` (Automatic)** | Metasploit runs its own service/fingerprint detection against the target first, then picks the matching exploit variant for you. | You're not 100% sure of the exact version, or you want Metasploit to handle detection. |
| **`target <specific id>`** | You tell Metasploit exactly which OS/app version to assume, skipping detection. | You already know the exact version (from recon, banners, fingerprinting) and want to save time or improve reliability. |

**Trade-off to keep in mind:** Automatic is more convenient but can be slower and, for some exploits, less reliable than manually specifying the right target if you already know it.

---

## 7. What makes targets different from each other

Different targets for the *same* exploit can vary by:

- **Service pack** (e.g. Windows XP SP2 vs. SP3)
- **OS version** (XP vs. Vista vs. 7)
- **Language/localization version** of the OS or application
- **Software version** (e.g. IE 7 vs. IE 8 vs. IE 9)

### Why this matters technically: return addresses

Many exploits (especially memory-corruption ones like this Use-After-Free example) depend on a **return address** — a specific memory location the exploit needs to redirect execution to. This address can shift because of:

- A different language pack
- A different software version
- Addresses being shifted by security hooks or mitigations

Common types of return addresses include:

- `jmp esp` — jump to wherever the stack pointer currently points
- A jump to a specific CPU register
- `pop/pop/ret` — a specific three-instruction sequence used to redirect execution

> This is a deep topic on its own. For now, just understand: **the "target" setting exists because the exact bytes an exploit needs to use can differ between versions, and picking the wrong one means the exploit won't work (or could crash the target).**

### How targets actually get identified (preview)

To correctly identify a target from scratch (when building or adapting an exploit), you'd typically:

1. **Obtain a copy of the target binaries** (the actual vulnerable software/OS files).
2. **Use `msfpescan`** to locate a suitable return address inside those binaries.

---

## 8. Cheat sheet

| Command | Where to run it | What it does |
|---|---|---|
| `show targets` | Root menu | Reports that no module is selected |
| `show targets` / `targets` | Inside a module | Lists all OS/version combinations this exploit supports |
| `info` | Inside a module | Full description, references, targets, and options — **run this first** |
| `options` | Inside a module | Shows current settings and what's required |
| `set target <id>` | Inside a module | Locks the exploit to one specific target instead of Automatic |

### Decision flow

```
Select exploit module
        │
        ▼
Run `info`  ───────────────►  Understand what it does, read the description
        │
        ▼
Run `show targets`
        │
        ├── Only "0 Automatic" listed  →  nothing more to decide, move on
        │
        └── Multiple targets listed
                   │
                   ▼
         Do you know the exact target version?
                   │
          ┌────────┴────────┐
          Yes                No
          │                  │
   set target <id>     leave as target 0 (Automatic)
```

### Big-picture takeaways

1. **A "target" = a specific OS/app version** an exploit is tuned for.
2. **`show targets` needs a module selected first** — it errors out at the root menu.
3. **Not every exploit needs manual targeting.** Some only offer `0 Automatic` because they work universally.
4. **Always run `info` before running anything new** — it's your safety and understanding check.
5. **"Automatic" triggers live detection**; a specific target skips detection and assumes you're right.
6. **Different targets exist because return addresses and other low-level details shift** between service packs, OS versions, language packs, and software versions.
7. **Target/return-address identification** (binaries + `msfpescan`) is exploit-development territory, covered later.

---
