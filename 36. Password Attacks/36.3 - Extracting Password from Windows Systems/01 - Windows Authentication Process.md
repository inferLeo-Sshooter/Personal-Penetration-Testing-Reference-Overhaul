# Windows Authentication Process

## Table of Contents

- [1. Start here to navigate](#1-start-here-to-navigate)
- [2. How these pieces actually work together when someone logs in](#2-how-these-pieces-actually-work-together-when-someone-logs-in)
- [3. LSASS](#3-lsass)
- [4. SAM](#4-sam)
- [5. Credential Manager](#5-credential-manager)
- [6. NTDS.dit](#6-ntdsdit)

---

## 1. Start here to navigate

### The big picture

Windows essentially has **two parallel hash databases**:
- one for **local accounts** (`SAM database`)
- one for the **whole domain** (`NTDS.dit`)
- plus one **process** that holds decrypted secrets in memory (`LSASS`)
- plus a completely separate vault for app passwords (`Credential Manager`)

<img width="1192" height="726" alt="image" src="https://github.com/user-attachments/assets/d913d972-9403-4d33-9b35-9e4b7d23305c" />

### How the pieces connect

On the **local machine side** — every Windows box, whether domain-joined or not, has its own mini credential system:
| Component | Description | What it stores |
|-----------|-------------------|---------------|
| **SAM database** | local database | Hashed passwords for *local* accounts only |
| **SECURITY hive** | registry hive | LSA secrets, cached domain logons, local security policy |
| **LSASS** | a **running process**, not a file | Live, in-memory credentials for currently logged-in users |

On the **domain side** is a completely separate, much bigger version of the same idea (lives only on `Domain Controllers`):
| Component | Description | What it stores |
|-----------|-------------------|---------------|
| **NTDS.dit** | essentially “SAM, but for the entire Active Directory domain” | Hashed passwords for *every* account in the whole domain |

And then, there's **Credential Manager sits apart from both**. It’s not an authentication backend at all — it’s a **per-user vault for saved application and website credentials** (think saved RDP passwords, mapped drive credentials, or browser-saved logins integrated with Windows). 

It exists on every machine regardless of domain membership, and extracting from it gets you app-level passwords, not OS authentication secrets:
| Component | Description | What it stores |
|-----------|-------------------|---------------|
| **Credential Manager** | per-user vault | Saved passwords for apps, websites, and network shares |

---

## 2. How these pieces actually work together when someone logs in

### The cast of characters

Logging into Windows isn't one single step — it's a relay race between several components:

| Component | Role |
|-----------|------|
| **WinLogon** | The trusted process that manages the login screen, password changes, and locking/unlocking |
| **LogonUI** | The actual graphical window you type your password into |
| **Credential Providers** | Plugins that know how to collect credentials (password, PIN, fingerprint, smart card, etc.) |
| **LSASS** | The "gatekeeper" process that actually verifies whether your credentials are correct |
| **Authentication Packages** | Specialized DLLs that LSASS calls on to check specific credential types (local password, Kerberos ticket, etc.) |
| **SAM / Active Directory** | The actual database the authentication package checks your credentials against |

### How a basic login actually flows

1. You hit a key or move the mouse at the lock screen → **WinLogon** (always running in the background) notices and launches **LogonUI**, the visual prompt.
2. **LogonUI** doesn't collect your password itself — it hands that job off to a **credential provider**, which is just a small plugin (a COM object/DLL) that knows how to grab whatever kind of credential you're using.
3. Once your username and password are collected, **WinLogon** passes them along to **LSASS**.
4. **LSASS** doesn't check the password itself either — it delegates to the right **authentication package** for the situation. For a typical local, non-domain login, that's `Msv1_0.dll`.
5. That authentication package checks your credentials against either the **SAM database** (local account) or **Active Directory** (domain account), and reports back whether the login succeeds.

Think of it like a receptionist (WinLogon) who directs you to a form (LogonUI/credential provider), hands your completed form to a security officer (LSASS), who then calls the right specialist department (the authentication package) to actually verify your ID against the records (SAM or AD).

---

## 3. LSASS

**LSASS - Local Security Authority Subsystem Service** (`%SystemRoot%\System32\Lsass.exe`) is the process that sits at the center of all of this. Its job is to:
- Enforce local security policy
- Actually authenticate users
- Forward security audit events to the Event Log

It doesn't do the verification work alone — it loads different **authentication packages** (DLLs) depending on the situation:

| DLL | What it handles |
|-----|--------------------|
| `Lsasrv.dll` | The core LSA server — enforces policy and decides whether to use NTLM or Kerberos for a given login |
| `Msv1_0.dll` | Handles standard local machine logons |
| `Samsrv.dll` | Manages the SAM database and its local policies |
| `Kerberos.dll` | Handles Kerberos-based domain authentication |
| `Netlogon.dll` | Handles network-based logon requests |
| `Ntdsa.dll` | Manages the Active Directory database itself (`ntds.dit`) — **only loaded on Domain Controllers** |

Each time someone logs in interactively, a separate WinLogon instance is created for that session, and the credentials collected get passed through to LSALogonUser, which triggers the authentication check described above.

> **Why attackers care about LSASS specifically:** Unlike SAM (which stores hashes on disk), LSASS holds credentials **in memory, in a usable form**, for every currently logged-in user. That's why tools designed to dump LSASS memory are such a big deal — they skip the "crack a hash" step entirely and can sometimes recover usable credentials directly.

---

## 4. SAM

The **Security Account Manager (SAM)** is where local account passwords are stored — as hashes (LM or NTLM format), not plaintext. It lives at:
```
%SystemRoot%\system32\config\SAM
```
and is also mounted into the registry at `HKLM\SAM`. Reading it requires **SYSTEM-level privileges** — a regular user, even an administrator, can't just open it directly.

### Workgroup vs. domain

- If a machine is in a **workgroup** (not domain-joined), *all* its user accounts and their password hashes live locally in SAM.
- If a machine is **domain-joined**, domain accounts instead get verified against the Domain Controller's copy of Active Directory (`ntds.dit`) — SAM on that machine still exists, but mainly holds local-only accounts like the built-in Administrator.

### SYSKEY: an extra layer of protection

Starting with Windows NT 4.0, Microsoft added **SYSKEY**, which partially encrypts the SAM file on disk using a system-generated key. 

This means even if someone gets a copy of the raw SAM file, the hashes inside are encrypted using a key — called the **boot key** — that’s derived from data stored in the **SYSTEM registry hive**, not SAM itself.

---

## 5. Credential Manager

**Credential Manager** - a different kind of vault. It is a built-in Windows feature for remembering credentials used to access **other things** — websites, network shares, and applications — not for logging into Windows itself. Each user has their own private "Credential Locker," encrypted and stored at:

```powershell
C:\Users\[Username]\AppData\Local\Microsoft\[Vault/Credentials]\
```

Think of it as Windows' version of a browser's saved-password manager, but built into the OS and usable by any application, not just a browser. Because it's encrypted per-user, decrypting it generally requires being logged in as that user (or using specialized extraction techniques).

---

## 6. NTDS.dit

In a nutshell: SAM's domain-scale big sibling.

In domain environments — which is most corporate networks — logons get checked against a **Domain Controller** instead of (or in addition to) the local SAM. Every Domain Controller in the same Active Directory forest keeps a synchronized copy of a file called:

```
%SystemRoot%\NTDS\ntds.dit
```

(Except Read-Only Domain Controllers, which hold a restricted copy.)

This single file is essentially the master record for the *entire domain*, containing:
- User accounts (including password hashes)
- Group accounts
- Computer accounts
- Group Policy Objects

Compared to SAM, which only covers one machine's local accounts, NTDS.dit is the "keys to the kingdom" — compromising it means potentially having every domain credential at once.

---

## Putting it all together

| Question | Answer |
|----------|--------|
| Who handles the login screen? | WinLogon + LogonUI + Credential Providers |
| Who actually verifies credentials? | LSASS, via the appropriate authentication package |
| Where do local password hashes live? | SAM database |
| Where do domain-wide password hashes live? | NTDS.dit (Domain Controllers only) |
| Where do live, in-memory credentials live? | Inside the running LSASS process |
| Where do saved app/website passwords live? | Credential Manager (completely separate system) |

**Key idea:** Windows authentication isn't one mechanism — it's a coordinated handoff between a UI layer (WinLogon/LogonUI), a verification layer (LSASS + authentication packages), and a storage layer that scales from "one machine" (SAM) up to "the whole domain" (NTDS.dit), with Credential Manager sitting entirely off to the side as a vault for *other* applications' saved logins.
