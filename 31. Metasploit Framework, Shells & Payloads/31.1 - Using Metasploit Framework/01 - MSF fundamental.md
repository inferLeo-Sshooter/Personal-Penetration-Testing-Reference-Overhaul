# Metasploit Framework

---

## Table of Contents

1. [Should we use automated tools at all?](#1-should-we-use-automated-tools-at-all)
2. [Discipline: how to use tools wisely](#2-discipline-how-to-use-tools-wisely)
3. [What is Metasploit?](#3-what-is-metasploit)
4. [Metasploit Pro vs. Framework](#4-metasploit-pro-vs-framework)
5. [What is msfconsole?](#5-what-is-msfconsole)
6. [Architecture: where everything lives](#6-architecture-where-everything-lives)
7. [Launching msfconsole](#7-launching-msfconsole)
8. [The MSF engagement structure](#8-the-msf-engagement-structure)
9. [A first walkthrough](#9-a-first-walkthrough)
10. [Cheat sheet and key takeaways](#10-cheat-sheet-and-key-takeaways)

---

## 1. Should we use automated tools at all?

The security community is split on this.

**The skeptics say:** automated tools let the tester skip the hard part. If a tool does the work, the tester gets no chance to show real skill, and the job looks too easy to deserve credit.

**The supporters say:** tools help beginners learn because they give a friendlier way into the huge world of vulnerabilities. They also save time, which frees you up for the harder, more interesting parts of an assessment.

The source sides with the supporters, but it admits tools have real downsides:

| Downside | What it means |
|---|---|
| **Comfort zone** | It becomes hard to leave the tool and learn new skills. |
| **Security risk** | Publicly available tools can be used by bad actors too, including people with almost no skill. |
| **Tunnel vision** | "If the tool can't do it, I can't either." You stop thinking beyond what the tool offers. |

**Analogy:** think of a calculator. It makes math faster, but if you never learn how multiplication works, you get stuck the moment the calculator can't help.

---

## 2. Discipline: how to use tools wisely

Security technology changes constantly, so you need good habits. The source gives three reminders:

1. **You will never have enough time.** Every environment contains many technologies, so a perfectly complete assessment is unrealistic. Tackle the issues with the **highest impact** first.
2. **Clients don't care about bragging rights.** They want the work done well, in the time they paid for. You are not competing with other testers. You are working within the client's budget and expectations.
3. **Impress yourself, not the community.** Your job is to *validate vulnerabilities*, not to *validate your ego*. Chasing online praise tends to make your work stale and generic.

### The rules of the road

- **Know your tools inside and out.** Some tools are unpredictable. They can leave traces on the target or even open holes on *your own* attacking machine.
- **Read the documentation.** Understanding what a tool actually does helps you avoid surprises, angry clients, and legal trouble.
- **Don't let a tool be your whole method.** It should support your methodology, not replace it.
- **Use the time you save to go deeper.** Study how security mechanisms really work.

---

## 3. What is Metasploit?

**Metasploit** is a **Ruby-based, modular penetration testing platform**. It lets you:

- Write exploit code
- Test exploit code
- Run exploit code

You can use your own code, or use the large library of **modules** that others have already built and tested.

### Key idea: modules

A **module** is a ready-made, tested piece of functionality, such as an exploit for a specific vulnerable service. Metasploit packages them so you can reach many attack vectors for many platforms through one consistent interface.

### The typical flow

```
Pick a vulnerable target
        │
        ▼
Pick an EXPLOIT built for that vulnerability   ← gets you through the door
        │
        ▼
Pick a PAYLOAD                                  ← what runs once you're in
        │
        ▼
Get access (a "session") to the system
        │
        ▼
Post-exploitation (explore, escalate, move around)
```

**Analogy:**
- The **exploit** is the lockpick that opens the door.
- The **payload** is what you do after you step inside, like installing a remote control so you can keep access.
- A **session** is that open connection to the target.

> Metasploit isn't a tool that does everything. It's more like a Swiss Army knife: it has just enough tools to handle the most common **unpatched** vulnerabilities.

---

## 4. Metasploit Pro vs. Framework

Metasploit comes in two versions.

| | **Framework** (free, open source) | **Pro** (commercial) |
|---|---|---|
| Console (`msfconsole`) | Yes | Yes (its own console too) |
| GUI / web interface | No | Yes |
| Task Chains (automated workflows) | No | Yes |
| Social engineering tools / Phishing wizard | No | Yes |
| Vulnerability validation | No | Yes |
| Quick Start wizards | No | Yes |
| Nexpose (vuln scanner) integration | No | Yes |

### What Pro adds, grouped by purpose

- **Infiltrate:** manual exploitation, antivirus and IDS/IPS evasion, proxy and VPN pivoting, post-exploitation, session clean-up, credential reuse, social engineering, payload generator, web app testing, persistent sessions.
- **Collect data:** importing and scanning data, discovery scans, meta-modules, Nexpose scan integration, Project Sonar integration.
- **Remediate and report:** bruteforce, task chains, session rerun, task replay, team collaboration, credential management, evidence collection, reporting, data export, backup and restore.

> **Note:** For this module (and most learning), the free **Framework** is all you need.

---

## 5. What is msfconsole?

`msfconsole` is the most popular way to use Metasploit. It's a single, centralized, text-based console where you can reach almost everything the framework offers.

**Why people like it:**

- It's the **only supported way** to access most Metasploit features.
- It's the **most stable** interface and has the **most features**.
- It has full **readline support**: tab completion, command history, and so on.
- You can run **external commands** (like `ping` or `nmap`) without leaving the console.

It looks intimidating at first, but once you learn the command syntax, it becomes fast and powerful.

### Sessions and jobs

You can juggle many targets at once with **sessions** (open connections to targets) and **jobs** (background tasks). Think of them like **browser tabs**: you can switch between them without losing your place.

---

## 6. Architecture: where everything lives

Knowing the folder layout helps you later when you want to **add new modules or write your own**.

On ParrotOS (and similar distros like Kali), the base directory is:

```
/usr/share/metasploit-framework
```

### Folder overview

| Folder | Purpose |
|---|---|
| `data`, `lib` | The core that makes `msfconsole` work |
| `documentation` | Technical docs for the project |
| `modules` | All the modules, split by category (see below) |
| `plugins` | Optional add-ons you load when needed |
| `scripts` | Meterpreter and other helper scripts |
| `tools` | Command-line utilities that can be called from the console |

### Module categories

```bash
$ ls /usr/share/metasploit-framework/modules

auxiliary  encoders  evasion  exploits  nops  payloads  post
```

| Category | What it's for (simple version) |
|---|---|
| `auxiliary` | Helper modules: scanners, fuzzers, brute-forcers. They don't give you a shell. |
| `encoders` | Re-package payloads so they work in different conditions (formatting, bad characters). |
| `evasion` | Help payloads avoid detection by security software. |
| `exploits` | Code that takes advantage of a specific vulnerability. |
| `nops` | "No-operation" filler instructions used in some exploit techniques. |
| `payloads` | The code that runs on the target after a successful exploit. |
| `post` | Post-exploitation modules: things you do *after* you have access. |

### Plugins

Plugins add extra features and can be loaded manually or automatically.

```bash
$ ls /usr/share/metasploit-framework/plugins/

aggregator.rb      ips_filter.rb  openvas.rb           sounds.rb
alias.rb           komand.rb      pcap_log.rb          sqlmap.rb
auto_add_route.rb  lab.rb         request.rb           thread.rb
beholder.rb        libnotify.rb   rssfeed.rb           token_adduser.rb
db_credcollect.rb  msfd.rb        sample.rb            token_hunter.rb
db_tracker.rb      msgrpc.rb      session_notifier.rb  wiki.rb
event_tester.rb    nessus.rb      session_tagger.rb    wmap.rb
ffautoregen.rb     nexpose.rb     socket_logger.rb
```

Examples: `nessus.rb` and `openvas.rb` connect to vulnerability scanners, `sqlmap.rb` connects to SQLmap, and `session_notifier.rb` alerts you when a new session opens.

Loading a plugin inside msfconsole looks like this:

```
msf6 > load nessus
```

### Scripts

```bash
$ ls /usr/share/metasploit-framework/scripts/

meterpreter  ps  resource  shell
```

- `meterpreter`: helper scripts for Meterpreter sessions (Meterpreter is Metasploit's advanced payload).
- `resource`: **resource scripts**, which are text files of msfconsole commands that run automatically.

A tiny resource script example (`setup.rc`):

```
use auxiliary/scanner/portscan/tcp
set RHOSTS 10.10.10.5
set PORTS 22,80,443
run
```

Run it on startup with:

```bash
$ msfconsole -r setup.rc
```

### Tools

```bash
$ ls /usr/share/metasploit-framework/tools/

context  docs     hardware  modules   payloads
dev      exploit  memdump   password  recon
```

Small utilities grouped by purpose (for example `password` and `recon`).

---

## 7. Launching msfconsole

Type this in your terminal:

```bash
$ msfconsole
```

You'll see a random splash-art banner, then the prompt. To skip the banner, use **quiet mode**:

```bash
$ msfconsole -q

msf6 >
```

### Useful startup options

| Command | What it does |
|---|---|
| `msfconsole -q` | Start without the banner |
| `msfconsole -r file.rc` | Run a resource script on startup |
| `msfconsole -x "command; command"` | Run commands on startup |

### First commands to try

```
msf6 > help          # list all available commands
msf6 > version       # show your Metasploit version
msf6 > exit          # leave the console
```

### Keep your tools sharp: updating

Before any engagement, make sure your modules are current.

- **Old way:** run `msfupdate` in the OS terminal (outside msfconsole).
- **Current way:** use the system package manager:

```bash
$ sudo apt update && sudo apt install metasploit-framework
```

---

## 8. The MSF engagement structure

Metasploit work can be organized into **five phases**:

```
1. Enumeration  →  2. Preparation  →  3. Exploitation  →  4. Privilege Escalation  →  5. Post-Exploitation
```

| Phase | Plain-English meaning |
|---|---|
| **Enumeration** | Find out what's there: hosts, open ports, services, versions. |
| **Preparation** | Pick and configure the right modules and settings for what you found. |
| **Exploitation** | Run the exploit and (hopefully) gain access. |
| **Privilege Escalation** | You're in, but with low rights. Try to become admin/root. |
| **Post-Exploitation** | Gather evidence, move to other systems, maintain access, clean up. |

Each phase has subcategories (for example, *Service Validation* and *Vulnerability Research*). Getting comfortable with this structure makes it easier to find the right Metasploit feature at the right time.

> **Tip from the source:** don't just read, experiment. Try every feature you can in a lab and analyze the results yourself. That's how you actually learn a tool.

---

## 9. A first walkthrough

Here is a safe, typical beginner workflow, run against **your own lab machine**. We'll use an *auxiliary scanner* so nothing is exploited yet.

### Step 1: Start the console

```bash
$ msfconsole -q
```

### Step 2: Search for a module

```
msf6 > search type:auxiliary portscan
```

You'll get a list like:

```
   #  Name                                Rank    Description
   -  ----                                ----    -----------
   0  auxiliary/scanner/portscan/tcp      normal  TCP Port Scanner
   1  auxiliary/scanner/portscan/syn      normal  TCP SYN Port Scanner
```

### Step 3: Select the module

```
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) >
```

Notice the prompt changes to show the active module.

### Step 4: Look at the options

```
msf6 auxiliary(scanner/portscan/tcp) > show options
```

Key things to read:

- **Name:** the setting
- **Current Setting:** its present value
- **Required:** `yes` means you must set it
- **Description:** what it does

### Step 5: Set the options

```
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 192.168.56.101
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 1-1000
```

> **RHOSTS** = Remote Host(s), the target. **LHOST** = Local Host, *your* machine. You'll see both constantly.

### Step 6: Run it

```
msf6 auxiliary(scanner/portscan/tcp) > run
```

Example output:

```
[+] 192.168.56.101:         - 192.168.56.101:22 - TCP OPEN
[+] 192.168.56.101:         - 192.168.56.101:80 - TCP OPEN
[*] 192.168.56.101:         - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

### Reading the output symbols

| Symbol | Meaning |
|---|---|
| `[*]` | Information |
| `[+]` | Success |
| `[-]` | Failure or error |
| `[!]` | Warning |

### How an exploit module differs (concept only)

When you move on to exploitation, the flow is the same, with one extra choice: a **payload**.

```
msf6 > use exploit/<platform>/<service>/<module_name>
msf6 exploit(...) > show options
msf6 exploit(...) > set RHOSTS <target IP>
msf6 exploit(...) > set LHOST <your IP>
msf6 exploit(...) > show payloads
msf6 exploit(...) > set PAYLOAD <chosen payload>
msf6 exploit(...) > check        # if supported: is the target actually vulnerable?
msf6 exploit(...) > run
```

**Payload names follow a pattern:** `platform / architecture / payload type / connection method`

```
windows/x64/meterpreter/reverse_tcp
   │      │       │           │
   │      │       │           └─ target connects BACK to you
   │      │       └───────────── Meterpreter: an advanced, flexible shell
   │      └───────────────────── 64-bit system
   └──────────────────────────── Windows target
```

- **Reverse** payload: the *target* connects back to *you*. This is the common case, since firewalls usually allow outbound traffic.
- **Bind** payload: the target opens a port and *you* connect to it.

### Managing sessions

Once you have access, you can manage it like browser tabs:

```
msf6 > sessions -l         # list active sessions
msf6 > sessions -i 1       # interact with session 1
msf6 > background          # send current session to the background (inside a session)
msf6 > jobs                # list background jobs
```

---

## 10. Cheat sheet and key takeaways

### Essential msfconsole commands

| Command | Purpose |
|---|---|
| `help` | List commands |
| `search <term>` | Find modules (e.g. `search type:exploit smb`) |
| `info` | Detailed info about the selected module |
| `use <module>` | Select a module |
| `show options` | Show the module's settings |
| `show payloads` | Show compatible payloads |
| `set <OPTION> <value>` | Set an option for this module |
| `setg <OPTION> <value>` | Set an option **globally** |
| `unset <OPTION>` | Clear an option |
| `check` | Test whether a target is vulnerable (if supported) |
| `run` / `exploit` | Execute the module |
| `back` | Leave the current module |
| `sessions` | Manage sessions |
| `jobs` | Manage background jobs |
| `load <plugin>` | Load a plugin |
| `db_nmap <args>` | Run Nmap and store results in Metasploit's database |
| `exit` | Quit |

### Big-picture takeaways

1. **Tools are helpers, not crutches.** Understand what's happening under the hood.
2. **Metasploit = modules + a console.** Exploits get you in, payloads decide what happens next, and auxiliary and post modules support everything around that.
3. **`msfconsole` is the main interface.** Learn its syntax and it will speed up your work a lot.
4. **Know the folder layout.** It matters when you add or write your own modules.
5. **Follow the five phases:** Enumeration → Preparation → Exploitation → Privilege Escalation → Post-Exploitation.
6. **Keep it updated, keep it legal.** Update your modules regularly and always have authorization.
7. **Experiment in a lab.** Hands-on practice beats reading every time.

---
