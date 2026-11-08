# Credential Hunting in Windows

## Table of Contents

- [Search Tool 1: Windows Search (built-in, GUI)](#search-tool-1-windows-search-built-in-gui)
- [Search Tool 2: LaZagne (automated credential harvester)](#search-tool-2-lazagne-automated-credential-harvester)
- [Search Tool 3: findstr (built-in, command-line text search)](#search-tool-3-findstr-built-in-command-line-text-search)
- [Adjusting your approach based on the machine](#adjusting-your-approach-based-on-the-machine)

---

## What is "credential hunting"?

Everything we've covered so far has targeted specific, well-known credential **stores** — SAM, LSASS, NTDS.dit, Credential Manager. But there's a much lower-tech category of attack that's often just as effective: **searching the filesystem and installed applications directly**, because people constantly leave passwords lying around in places they shouldn't.

### Setting the scene

Imagine you've gained RDP access to an **IT admin's Windows 10 workstation**. Before randomly poking around, it pays to think about the user: *what does an IT admin actually do all day, and what tools would require them to type or save passwords?* That context should drive where you look — admins often juggle credentials for VPNs, RDP sessions, databases, network devices, and internal tools, any of which might get saved somewhere "temporarily" and then forgotten.

---

## Useful search terms

A short list of keywords that tend to surface saved credentials across files, configs, and notes:

```
password        passphrase       key
username        user account     creds
users           passkey          configuration
dbcredential    dbpassword       pwd
login           credentials
```

---

## Search Tool 1: Windows Search (built-in, GUI)

If you have graphical access, don't overlook the OS's own built-in search — typing any of the keywords above into Windows Search will scan both file contents and application data for matches. It's free, it's already installed, and it's an easy first pass before reaching for anything heavier.

<img width="805" height="734" alt="image" src="https://github.com/user-attachments/assets/89bcb744-61e4-4881-881d-0558d60a76f6" />

---

## Search Tool 2: LaZagne (automated credential harvester)

**LaZagne** is a dedicated tool for pulling credentials out of the insecure ways many applications store them — especially browsers, which are a goldmine since so many people let Chrome/Edge/Firefox "remember" their passwords.

### What it covers

| Module | Targets |
|--------|------------|
| `browsers` | Chromium, Firefox, Edge, Opera |
| `chats` | Skype and other chat apps |
| `mails` | Outlook, Thunderbird |
| `memory` | KeePass and LSASS memory |
| `sysadmin` | Config files for tools like OpenVPN, WinSCP |
| `windows` | LSA secrets, Credential Manager, and other Windows-native stores |
| `wifi` | Saved Wi-Fi network passwords |

LaZagne alone supports credential extraction from **35 different browsers** on Windows — a huge surface area if the target has ever saved a single password in their browser.

> **Why browsers specifically matter so much:** browser-stored credentials are encrypted, but that encryption is routinely broken — standalone tools like [firefox_decrypt](https://github.com/unode/firefox_decrypt) and various [Chrome-password-decryptors](https://github.com/ohyicong/decrypt-chrome-passwords) exist specifically because this encryption is well understood and well-documented. It's often surprisingly easy to pull real, plaintext credentials straight out of a browser's saved password store.

### Running LaZagne on the target

1. Keep a standalone copy of `LaZagne.exe` ready on your attack machine.
2. Transfer it over — if you're using an RDP client like `xfreerdp`, you can typically just copy-paste the file directly into the session.
3. Run it:

```cmd
C:\Users\bob\Desktop> start LaZagne.exe all
```

The `all` flag tells it to run every module it has. (Add `-vv` for verbose output, so you can see exactly which software it's checking as it goes.)

### Example output

```
|====================================================================|
|                                                                    |
|                        The LaZagne Project                         |
|                                                                    |
|                          ! BANG BANG !                             |
|                                                                    |
|====================================================================|


########## User: bob ##########

------------------- Winscp passwords -----------------

[+] Password found !!!
URL: 10.129.202.51
Login: admin
Password: SteveisReallyCool123
Port: 22
```

Found entirely in plaintext — WinSCP (an SFTP/SCP client) had stored a saved server login that LaZagne simply recovered. This kind of result is common precisely because so many desktop applications store "remember me" credentials with weak or reversible protection.

---

## Search Tool 3: findstr (built-in, command-line text search)

For a quick, no-install option from the command line, `findstr` can grep through files for your keyword list:

```cmd
C:\> findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

What the flags mean:
- `/S` — search subdirectories too
- `/I` — case-insensitive match
- `/M` — only print filenames that matched (not every matching line)
- `/C:"password"` — the literal string to search for

This is a fast way to sweep a whole directory tree for config files, scripts, or notes that happen to contain the word "password" (or whatever term you swap in).

---

## Adjusting your approach based on the machine

What you search for — and where — should shift depending on **what kind of system you've landed on**. A Windows Server running business applications calls for a different hunting strategy than someone's everyday desktop. Sometimes you'll even stumble onto credentials just by browsing folder structures while your other tools are still running.

### Other high-value places to check

Beyond generic keyword searches, these specific spots are worth checking directly, especially in domain environments:

| Location | Why it's worth checking |
|----------|----------------------------|
| Group Policy objects in the **SYSVOL** share | Occasionally contain hardcoded passwords from policy scripts |
| Scripts stored in **SYSVOL** or IT shares | Login automation scripts sometimes embed plaintext credentials |
| `web.config` files on dev machines/IT shares | Web app configs often store database or service credentials |
| `unattend.xml` | Windows' automated-install config file, which can contain the local admin password set during OS deployment |
| **AD user/computer description fields** | Admins sometimes (badly) use the "description" field as an informal note, occasionally including a password |
| **KeePass databases** | If the master password can be guessed or cracked, this unlocks everything stored inside |
| Loose files like `pass.txt`, `passwords.docx`, `passwords.xlsx` | Found surprisingly often on user desktops, network shares, or SharePoint sites |

---

## Quick Recap

| Tool | Best for |
|------|-------------|
| Windows Search | Fast, no-setup keyword search across files/apps |
| LaZagne | Automated extraction from browsers, chat apps, mail clients, sysadmin tools, and Windows-native stores |
| `findstr` | Lightweight command-line keyword search across config/script files |
| Manual browsing | SYSVOL, `unattend.xml`, AD description fields, loose password files on shares |

**Key idea:** Not every credential needs to be cracked or dumped from a protected store — a huge number of real-world passwords are just sitting in plaintext somewhere, left behind in a config file, a saved app login, or a forgotten note, simply because applications and people alike are often careless about where credentials end up.
