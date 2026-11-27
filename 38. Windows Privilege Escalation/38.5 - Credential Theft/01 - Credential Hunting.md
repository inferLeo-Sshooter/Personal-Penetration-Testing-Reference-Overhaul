# Credential Hunting on Windows

## Table of Contents
1. [Big Idea: Why Credential Hunting Matters](#1-big-idea-why-credential-hunting-matters)
2. [Application Configuration Files](#2-application-configuration-files)
3. [Dictionary Files (Chrome, etc.)](#3-dictionary-files-chrome-etc)
4. [Unattended Installation Files](#4-unattended-installation-files)
5. [PowerShell Command History](#5-powershell-command-history)
6. [PowerShell Stored Credentials (DPAPI)](#6-powershell-stored-credentials-dpapi)
7. [Network Shares and Other Loose Files](#7-network-shares-and-other-loose-files)
8. [Manually Searching the File System](#8-manually-searching-the-file-system)
9. [Sticky Notes App Passwords](#9-sticky-notes-app-passwords)
10. [Other Files Worth Checking](#10-other-files-worth-checking)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Big-Picture Takeaways](#12-big-picture-takeaways)

---

## 1. Big Idea: Why Credential Hunting Matters

During a penetration test (a simulated, authorized hacking engagement meant to find security weaknesses before real attackers do), one of the most valuable things you can stumble across is a **credential** — a username and password, or some other secret like an encryption key. Credentials found while poking around a compromised machine can:

- Hand you **local administrator** access on that machine.
- Give you a **foothold** (an initial point of access) into a company's Active Directory environment — the central system Windows networks use to manage user accounts, computers, and permissions.
- Let you **escalate privileges**, meaning move from a low-permission account to a high-permission one, inside that same domain.

Think of a compromised computer like a house you've just been let into through a side door. Credential hunting is the process of checking under doormats, inside drawers, and behind picture frames for spare keys that open other doors in the house (or even the house next door).

There are many places these "spare keys" get left behind — some obvious, some easy to overlook. This document walks through the most common hiding spots.

---

## 2. Application Configuration Files

### Why This Happens
Even though it's against best security practice, many applications save passwords in **plaintext** (readable, unencrypted text) inside their configuration files. If you've gained **command execution** (the ability to run commands) as a low-privilege user, you might find credentials for an admin account sitting in one of these files.

### Searching for Them
[findstr](https://ss64.com/nt/findstr.html) is a built-in Windows command-line tool that searches file contents for a specific word or pattern — similar to Ctrl+F, but run against many files at once from the command line.

```
PS C:\evilcorp> findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

| Flag | What it does |
|------|--------------|
| `/S` | Search subdirectories too, not just the current folder |
| `/I` | Case-insensitive search (matches "Password", "PASSWORD", etc.) |
| `/M` | Only print the file name of matches, not every matching line |
| `/C:"..."` | Treat the text in quotes as one exact literal string to search for |

### A Specific Target: IIS Web Config
**IIS** (Internet Information Services) is Microsoft's web server software. Its sites often store sensitive settings — sometimes including credentials — in a file called `web.config`. For the default website, this typically lives at:

```
C:\inetpub\wwwroot\web.config
```

Because a server can host multiple sites, there may be several copies of this file in different folders, so it's worth searching recursively (through all subfolders) rather than checking just one location.

---

## 3. Dictionary Files (Chrome, etc.)

### The Concept
Word processors, email clients, and browsers often underline words they don't recognize ("spell-check squiggles"). If a user types a password into one of these text boxes and it gets underlined as a typo, some people add the word to their **custom dictionary** just to stop the red squiggle — without realizing that dictionary is just a plain text file sitting on disk.

It's a bit like writing your front door code on a sticky note just to stop forgetting it — convenient for you, but also readable by anyone who finds the note.

### Example: Chrome's Custom Dictionary
`gc` is a PowerShell shorthand (alias) for `Get-Content`, which reads a file's contents. `Select-String` works like `findstr` — it searches for a pattern.

```
PS C:\evilcorp> gc 'C:\Users\john\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password

Password1234!
```

Here, the command reads Chrome's dictionary file and filters the output to only show lines containing the word "password" — and finds an actual password sitting right there.

---

## 4. Unattended Installation Files

### What These Are
An **unattended installation** is a Windows setup process that runs without a person manually clicking through prompts — all the install-time answers (computer name, accounts to create, auto-login settings) are pre-written into an answer file, usually named `unattend.xml`. Passwords in this file are stored either in **plaintext** or **Base64** (a simple text-encoding scheme — not real encryption, easily reversible).

### Example `unattend.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="specialize">
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <AutoLogon>
                <Password>
                    <Value>local_4dmin_p@ss</Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <LogonCount>2</LogonCount>
                <Username>Administrator</Username>
            </AutoLogon>
            <ComputerName>*</ComputerName>
        </component>
    </settings>
```

Reading it line by line:
- `<AutoLogon>` block configures Windows to log a user in automatically without a password prompt.
- `<Password><Value>...</Value><PlainText>true</PlainText></Password>` — the actual password, flagged as plaintext (not encoded).
- `<Username>Administrator</Username>` — tells you which account that password belongs to.

### Why Copies Still Exist
Microsoft's setup process is *supposed* to delete this file automatically once installation finishes. In practice, system administrators building custom Windows images often save extra copies in other folders while they're testing — and forget to clean them up.

---

## 5. PowerShell Command History

### Background
Since PowerShell 5.0 (which ships by default starting with Windows 10), PowerShell automatically logs every command you type into a history file, using a feature called **PSReadLine**. This is meant to be a convenience (press the up arrow to recall old commands) — but it also means any password accidentally typed on the command line gets permanently saved to disk.

### Finding the History File's Location
```
PS C:\evilcorp> (Get-PSReadLineOption).HistorySavePath

C:\Users\john\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
This command asks PowerShell's own settings object where it's configured to save history, confirming the file path.

### Reading It
```
PS C:\evilcorp> gc (Get-PSReadLineOption).HistorySavePath

dir
cd Temp
md backups
cp c:\inetpub\wwwroot\* .\backups\
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://www.powershellgallery.com/packages/MrAToolbox/1.0.1/Content/Get-IISSite.ps1'))
. .\Get-IISsite.ps1
Get-IISsite -Server WEB02 -web "Default Web Site"
wevtutil qe Application "/q:*[Application [(EventID=3005)]]" /f:text /rd:true /u:WEB02\administrator /p:5erv3rAdmin! /r:WEB02
```
Reading through this, the last line is the goldmine: an admin ran `wevtutil` (a Windows tool for querying the Event Log) and passed credentials directly on the command line with `/u:` (username) and `/p:` (password) — `WEB02\administrator` and `5erv3rAdmin!` — both now sitting in plain text in the history file.

### Checking Every User on the Box at Once
This one-liner loops through every folder under `C:\Users` and tries to read each user's history file — useful as a post-exploitation step (something done right after gaining deeper access) to recheck accounts you couldn't read before.

```
PS C:\evilcorp> foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}

dir
cd Temp
md backups
cp c:\inetpub\wwwroot\* .\backups\
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://www.powershellgallery.com/packages/MrAToolbox/1.0.1/Content/Get-IISSite.ps1'))
. .\Get-IISsite.ps1
Get-IISsite -Server WEB02 -web "Default Web Site"
wevtutil qe Application "/q:*[Application [(EventID=3005)]]" /f:text /rd:true /u:WEB02\administrator /p:5erv3rAdmin! /r:WEB02
```
- `ls C:\users` — lists every folder under `C:\Users` (i.e., every user profile).
- `-ErrorAction SilentlyContinue` — if PowerShell can't read a particular user's file (e.g., permission denied), it skips silently instead of throwing an error and stopping.

---

## 6. PowerShell Stored Credentials (DPAPI)

### The Concept
Admins and script writers often don't want to hardcode a plaintext password inside a script, so PowerShell lets them save a credential to an encrypted file instead, using `Export-Clixml`. The encryption is handled by **DPAPI** (Data Protection API), a built-in Windows feature. The important catch: DPAPI-protected data can normally only be decrypted **by the same Windows user account, on the same computer**, where it was originally encrypted — think of it like a locker that only opens with the specific badge of the person who locked it, in that specific building.

### Example Script (`Connect-VC.ps1`)

The following script `Connect-VC.ps1`, which a sysadmin has created to connect to a vCenter server easily.
```powershell
# Connect-VC.ps1
# Get-Credential | Export-Clixml -Path 'C:\scripts\pass.xml'
$encryptedPassword = Import-Clixml -Path 'C:\scripts\pass.xml'
$decryptedPassword = $encryptedPassword.GetNetworkCredential().Password
Connect-VIServer -Server 'VC-01' -User 'bob_adm' -Password $decryptedPassword
```
- The commented-out first line shows how the credential was originally created and saved (`Export-Clixml`).
- `Import-Clixml` loads the encrypted credential object back in.
- `.GetNetworkCredential().Password` pulls the actual decrypted password back out — but only works under the right user/machine combination.
- The script then uses that password to connect to a vCenter server (VMware's management platform).

### Decrypting It Yourself
If you have command execution as that same user, you can run the same decryption steps the script does:

```
PS C:\evilcorp> $credential = Import-Clixml -Path 'C:\scripts\pass.xml'
PS C:\evilcorp> $credential.GetNetworkCredential().username

bob


PS C:\evilcorp> $credential.GetNetworkCredential().password

Str0ng3ncryptedP@ss!
```

---

## 7. Network Shares and Other Loose Files

Beyond the local machine, network **share drives** (folders shared across the network, like a company-wide "shared drive" everyone can access) often hide credentials too. In Active Directory environments, a tool called [Snaffler](https://github.com/SnaffCon/Snaffler) can automatically crawl these shares looking for interesting file types, such as:

| Extension | What it usually is |
|-----------|---------------------|
| `.kdbx` | KeePass password database file |
| `.vmdk` / `.vhdx` | Virtual hard disk files (could be mounted to extract local admin password hashes) |
| `.ppk` | PuTTY private key (used for SSH authentication) |

Other common finds on shares include SSH private keys, Excel/Word files, OneNote notebooks, or a plain old `passwords.txt`.

### The "Personal Folder" Trap
Many companies give each employee a personal folder on a shared drive — e.g., a folder called `bjones` for a user named Bob Jones, on a share called `users` on a server called `FILE01`. If permissions are configured loosely (for example, if **all Domain Users** — meaning every single employee account in the company — have read access to *everyone's* personal folder, not just their own), people unknowingly expose sensitive personal files to the whole organization, assuming the folder is private just because it has their name on it.

---

## 8. Manually Searching the File System

Automated scripts are useful, but it's valuable to know how to search by hand too — you might catch something a script's rule set doesn't cover.

We can search the file system or share drive(s) manually using the following commands from this [cheatsheet](https://swisskyrepo.github.io/InternalAllTheThings/redteam/escalation/windows-privilege-escalation/).

### `findstr` Examples

```
C:\evilcorp> cd c:\Users\john\Documents & findstr /SI /M "password" *.xml *.ini *.txt

stuff.txt
```
`/SI` combines recursive search with case-insensitivity; `/M` prints just the matching filename.

```
C:\evilcorp> findstr /si password *.xml *.ini *.txt *.config

stuff.txt:password: l#-x9r11_2_GL!
```
Without `/M`, it shows the filename **and** the matching line itself.

```
C:\evilcorp> findstr /spin "password" *.*

stuff.txt:1:password: l#-x9r11_2_GL!
```
Adding `/N` (bundled into `/spin`) also prints the line number where the match occurred.

### PowerShell Equivalent
```
PS C:\evilcorp> select-string -Path C:\Users\john\Documents\*.txt -Pattern password

stuff.txt:1:password: l#-x9r11_2_GL!
```
`Select-String` is PowerShell's native version of `findstr` — same idea, different tool.

### Searching by File Extension/Name Instead of Content

```
C:\evilcorp> dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*

c:\inetpub\wwwroot\web.config
```
```
C:\evilcorp> where /R C:\ *.config

c:\inetpub\wwwroot\web.config
```
```
PS C:\evilcorp> Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore

    Directory: C:\inetpub\wwwroot

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         5/25/2021   9:59 AM            329 web.config
```
These three commands all do roughly the same job through different tools: find every file on the drive whose **name** matches a pattern (rather than searching file contents), which is faster when you're just hunting for "files that are likely to contain creds" by type.

### Content Search vs. Name Search — Decision Flow

```
                 Looking for credentials on disk?
                              |
                 ------------------------------
                 |                            |
      Know roughly WHAT            Know roughly WHAT KIND
      the text says?               of file it'd be in?
      (e.g. "password")            (e.g. .config, .rdp, .vnc)
                 |                            |
         Search FILE CONTENTS         Search FILE NAMES / EXTENSIONS
         (findstr, Select-String)     (dir /s, where /R, Get-ChildItem -Include)
                 |                            |
                 ------------------------------
                              |
                  Found a candidate file?
                              |
                 Open & manually confirm it's
                   a real, usable credential
```

---

## 9. Sticky Notes App Passwords

### The Trap
The built-in Windows **Sticky Notes** app feels like a scrap of paper on your screen, so people jot down passwords and other sensitive info in it without realizing it's actually backed by a real database file on disk: `plum.sqlite` (a **SQLite** database — a lightweight, file-based database format).

Path:
```
C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

### Spotting the Files
```
PS C:\evilcorp> ls
 
    Directory: C:\Users\john\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState
 
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         5/25/2021  11:59 AM          20480 15cbbc93e90a4d56bf8d9a29305b8981.storage.session
-a----         5/25/2021  11:59 AM            982 Ecs.dat
-a----         5/25/2021  11:59 AM           4096 plum.sqlite
-a----         5/25/2021  11:59 AM          32768 plum.sqlite-shm
-a----         5/25/2021  12:00 PM         197792 plum.sqlite-wal
```
There are actually three related files (`plum.sqlite`, `-shm`, `-wal`) — SQLite uses the extra two as temporary helper files for performance and crash-recovery, but they can also contain recently-written data not yet saved into the main file, so it's worth grabbing all three.

### Three Ways to Read the Data

**Option A — GUI tool:** Copy the three files to your own machine, open them in [DB Browser for SQLite](https://sqlitebrowser.org/dl/) (a free graphical database viewer), and run the query `select Text from Note;` to see note contents.

**Option B — PowerShell with the PSSQLite module:**

This can also be done with PowerShell using the [PSSQLite module](https://github.com/RamblingCookieMonster/PSSQLite). First, import the module, point to a data source (in this case, the SQLite database file used by the StickNotes app), and finally query the `Note` table and look for any interesting data.
```
PS C:\evilcorp> Set-ExecutionPolicy Bypass -Scope Process

Execution Policy Change
The execution policy helps protect you from scripts that you do not trust. Changing the execution policy might expose
you to the security risks described in the about_Execution_Policies help topic at
https:/go.microsoft.com/fwlink/?LinkID=135170. Do you want to change the execution policy?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): A

PS C:\evilcorp> cd .\PSSQLite\
PS C:\evilcorp> Import-Module .\PSSQLite.psd1
PS C:\evilcorp> $db = 'C:\Users\john\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite'
PS C:\evilcorp> Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
 
Text
----
\id=de368df0-6939-4579-8d38-0fda521c9bc4 vCenter
\id=e4adae4c-a40b-48b4-93a5-900247852f96
\id=1a44a631-6fff-4961-a4df-27898e9e1e65 root:Vc3nt3R_adm1n!
\id=c450fc5f-dc51-4412-b4ac-321fd41c522a Thycotic demo tomorrow at 10am
```
- `Set-ExecutionPolicy Bypass -Scope Process` temporarily allows running unsigned scripts just for this session (Windows normally blocks unsigned scripts by default as a safety measure).
- `Import-Module` loads the PSSQLite module, which teaches PowerShell how to talk to SQLite database files.
- `Invoke-SqliteQuery` runs an actual SQL query (`SELECT Text FROM Note`) against the database — same idea as the GUI query in Option A, just from the command line.
- The result shows a note containing a real credential: `root:Vc3nt3R_adm1n!`.

**Option C — `strings` as a fallback:**
`strings` is a tool that pulls out all human-readable text from a binary file, ignoring everything else. It's less precise (you get the database's internal structure mixed in with your data) but works when you don't have database tools handy.

```
$  strings plum.sqlite-wal

CREATE TABLE "Note" (
"Text" varchar ,
...
"Id" varchar primary key not null ,
...
\id=011f29a4-e37f-451d-967e-c42b818473c2 vCenter
\id=34910533-ddcf-4ac4-b8ed-3d1f10be9e61 alright*
\id=ffaea2ff-b4fc-4a14-a431-998dc833208c root:Vc3nt3R_adm1n!ManagedPosition=Yellow93b49900-6530-42e0-b35c-2663989ae4b3af907b1b-1eef-4d29-b238-3ea74f7ffe5c
```
Buried in the raw output is the same credential (`root:Vc3nt3R_adm1n!`), just surrounded by database formatting noise rather than neatly in a table.

---

## 10. Other Files Worth Checking

A grab-bag of additional legacy and system files that have historically held credentials or sensitive data:

```
%SYSTEMDRIVE%\pagefile.sys
%WINDIR%\debug\NetSetup.log
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software, %WINDIR%\repair\security
%WINDIR%\iis6.log
%WINDIR%\system32\config\AppEvent.Evt
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
%WINDIR%\system32\CCM\logs\*.log
%USERPROFILE%\ntuser.dat
%USERPROFILE%\LocalS~1\Tempor~1\Content.IE5\index.dat
%WINDIR%\System32\drivers\etc\hosts
C:\ProgramData\Configs\*
C:\Program Files\Windows PowerShell\*
```
`%SYSTEMDRIVE%`, `%WINDIR%`, and `%USERPROFILE%` are **environment variables** — placeholders Windows fills in automatically (usually `C:`, `C:\Windows`, and `C:\Users\<currentuser>` respectively).

Automated privilege-escalation enumeration scripts (tools that automatically scan a system for security weaknesses) typically already check most of these spots. Still, it's worth knowing how to search manually, both to catch anything the scripts miss and so you can extend or customize those scripts yourself when needed.

---

## 11. Cheat Sheet

| Goal | Command |
|------|---------|
| Search file contents for "password" (several extensions) | `findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml` |
| Search file contents, PowerShell | `select-string -Path C:\path\*.txt -Pattern password` |
| Find PowerShell history file path | `(Get-PSReadLineOption).HistorySavePath` |
| Read PowerShell history | `gc (Get-PSReadLineOption).HistorySavePath` |
| Read every user's PowerShell history | `foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}` |
| Decrypt a saved PS credential (same user/machine only) | `(Import-Clixml -Path 'C:\path\pass.xml').GetNetworkCredential().password` |
| Find files by extension (CMD) | `dir /S /B *pass*.txt == *.config*` or `where /R C:\ *.config` |
| Find files by extension (PowerShell) | `Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred` |
| Query Sticky Notes database | `Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note"` |
| Pull readable text from a binary/db file | `strings plum.sqlite-wal` |

---

## 12. Big-Picture Takeaways

1. Credentials get left behind almost everywhere on a Windows system — config files, command history, spell-check dictionaries, note-taking apps, and old installation leftovers — because convenience usually wins out over security in day-to-day use.
2. Two broad search strategies exist: searching **file contents** for keywords like "password" (`findstr`, `Select-String`) and searching for **file types/names** known to often contain secrets (`.config`, `.rdp`, `.vnc`, `.kdbx`, etc.).
3. Some credentials are encrypted (like DPAPI-protected PowerShell credentials), but that encryption is often tied to a specific user account and machine — so gaining code execution as that exact user can still let you unlock it.
4. Apps that don't look like "databases" — Sticky Notes, browser dictionaries — often are databases under the hood, and are easy to overlook for that reason.
5. Network shares and personal user folders are common dumping grounds for credentials, especially when folder permissions are set too loosely (e.g., every employee can read every other employee's personal folder).
6. Automated enumeration scripts cover most of these spots already, but understanding the manual techniques lets you catch what scripts miss and customize your own tooling.
