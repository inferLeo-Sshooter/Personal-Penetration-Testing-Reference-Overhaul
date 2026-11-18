# Linux Privilege Escalation — Environment Enumeration Guide

## Table of Contents

1. [What Is Privilege Escalation](#1-what-is-privilege-escalation)
2. [Why Enumeration Comes First](#2-why-enumeration-comes-first)
3. [First Moves: Quick Situational Awareness](#3-first-moves-quick-situational-awareness)
4. [System-Level Information](#4-system-level-information)
   - 4.1 [OS Version](#41-os-version)
   - 4.2 [Kernel Version](#42-kernel-version)
   - 4.3 [Running Services and Processes](#43-running-services-and-processes)
   - 4.4 [Installed Packages and Versions](#44-installed-packages-and-versions)
5. [Orienting Yourself on the Box](#5-orienting-yourself-on-the-box)
   - 5.1 [PATH Variable](#51-path-variable)
   - 5.2 [Environment Variables](#52-environment-variables)
   - 5.3 [CPU Info](#53-cpu-info)
   - 5.4 [Available Login Shells](#54-available-login-shells)
   - 5.5 [Security Defenses in Place](#55-security-defenses-in-place)
6. [Users and Groups](#6-users-and-groups)
   - 6.1 [/etc/passwd Breakdown](#61-etcpasswd-breakdown)
   - 6.2 [Readable Shadow File / Hashes in /etc/passwd](#62-readable-shadow-file--hashes-in-etcpasswd)
   - 6.3 [Password Hash Types](#63-password-hash-types)
   - 6.4 [Users With Login Shells](#64-users-with-login-shells)
   - 6.5 [Groups and getent](#65-groups-and-getent)
7. [User-Level Information and Home Directories](#7-user-level-information-and-home-directories)
   - 7.1 [Home Directories Overview](#71-home-directories-overview)
   - 7.2 [Inspecting a User's Directory](#72-inspecting-a-users-directory)
   - 7.3 [SSH Keys](#73-ssh-keys)
   - 7.4 [Bash History](#74-bash-history)
   - 7.5 [Sudo Privileges](#75-sudo-privileges)
8. [Files and Filesystem Clues](#8-files-and-filesystem-clues)
   - 8.1 [Configuration Files](#81-configuration-files)
   - 8.2 [Cron Jobs](#82-cron-jobs)
   - 8.3 [SETUID and SETGID Binaries](#83-setuid-and-setgid-binaries)
   - 8.4 [Writable Directories and Files](#84-writable-directories-and-files)
9. [Drives, Mounts, and Network Info](#9-drives-mounts-and-network-info)
   - 9.1 [Block Devices (lsblk)](#91-block-devices-lsblk)
   - 9.2 [Printers](#92-printers)
   - 9.3 [fstab and Credential Hunting](#93-fstab-and-credential-hunting)
   - 9.4 [Mounted Filesystems (df -h)](#94-mounted-filesystems-df--h)
   - 9.5 [Unmounted Filesystems](#95-unmounted-filesystems)
   - 9.6 [Routing Table](#96-routing-table)
   - 9.7 [DNS Config](#97-dns-config)
   - 9.8 [ARP Table](#98-arp-table)
10. [Hidden Files, Hidden Directories, and Temp Folders](#10-hidden-files-hidden-directories-and-temp-folders)
11. [Decision Flow: Where Do I Look First?](#11-decision-flow-where-do-i-look-first)
12. [Cheat Sheet](#12-cheat-sheet)
13. [Big-Picture Takeaways](#13-big-picture-takeaways)

---

## 1. What Is Privilege Escalation?

On a Linux computer, the **root** account is like the building superintendent who has a master key to every room — it can read, change, or delete anything on the system.

In a security assessment (a controlled, authorized test of a lab or client environment), you often start out with a **low-privileged shell** — basically a regular tenant's keycard that only opens a few doors. **Privilege escalation** is the process of working your way up from that limited access to the superintendent's master key (root).

Getting root matters because it lets you:
- Capture network traffic on the machine
- Read sensitive files
- Use this machine as a stepping stone to reach other machines in the network
- If the Linux box is joined to a Windows **Active Directory** (a centralized system for managing users/computers on a network) domain, grab its **NTLM hash** (a Windows password representation) to start attacking the AD environment

---

## 2. Why Enumeration Comes First

**Enumeration** means systematically gathering information about a system before trying to act on it. Think of it like a locksmith casing a building before trying any locks — checking which doors are unlocked, which windows are weak, which keys are lying around in plain sight. Or, picture a new hire being handed a badge that only opens a few doors in a huge office building — before trying to sneak into the server room, it makes sense to first walk the halls, check which doors are unlocked, see which rooms have been left unattended with notes stuck to monitors, and figure out who else works there and what they do.

There are helper tools that automate a lot of this walk-through, such as [LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) and [LinEnum](https://github.com/rebootuser/LinEnum) (both are scripts that scan a Linux box and flag anything interesting). They're useful, but they don't replace understanding *why* a given piece of information matters and *how* to go find it by hand — a script might miss something, might not be allowed to run, or might simply not be available on the box you land on.

When you get initial shell access to a host (your first foothold — basically the lowest rung of the ladder), there's a specific checklist of things worth checking every time, covered one by one below.

---

## 3. First Moves: Quick Situational Awareness

Imagine you've just gained access to a Linux host by exploiting an unrestricted file upload flaw during an external penetration test. You've got a reverse shell and ideally some persistence (a way to get back in later if your shell drops). Now what?

The very first question to answer is: **what operating system am I actually on?** Landing on CentOS or Red Hat Enterprise Linux calls for slightly different enumeration than landing on a Debian-based system like Ubuntu. Landing on something more obscure — FreeBSD, Solaris, HP's proprietary HP-UX, or IBM's AIX — means the commands themselves may be completely different, sometimes requiring you to look up a command reference. The underlying *principles*, though, stay the same regardless of the specific flavor of Linux (or Unix) you land on.

There are plenty of enumeration cheat sheets out there, and several of them overlap in what information they're hunting for, even if they get there via different commands. Understanding *what* each command is doing and *why* matters more than memorizing the exact syntax — if a command doesn't work on a given box, knowing what you're actually looking for lets you find another way to get the same answer.

A handful of basic commands are worth running immediately, just to orient yourself:

| Command | What It Tells You |
|---|---|
| `whoami` | What user you're currently running as |
| `id` | What groups your user belongs to |
| `hostname` | The server's name — sometimes naming conventions hint at the server's role or the organization's internal naming scheme |
| `ifconfig` or `ip a` | What subnet (network segment) you landed in, and whether the host has additional network interfaces reaching into other subnets |
| `sudo -l` | Whether your user can run anything with `sudo` — i.e., as another user, or as root — without needing a password. This can sometimes be the easiest possible win: if it allows something broad, a simple `sudo su` (switch user to root) drops you straight into a root shell |

**Tip for reporting:** taking screenshots of the above output is useful for a client report — it provides clear evidence of successful remote code execution (RCE) and clearly identifies exactly which system was affected.

### Who Else Is Logged In

Knowing who else is logged into the system, and what they're actively doing, can reveal lateral movement opportunities or hint at privilege escalation paths that wouldn't be obvious otherwise.

**Listing terminal-attached processes (i.e., what logged-in users are actually running):**
```bash
$ ps au

USER            PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root            1256  0.0  0.1  65832  3364 tty1     Ss   23:26   0:00 /bin/login --
goffy.d     1322  0.0  0.1  22600  5160 tty1     S    23:26   0:00 -bash
shared          1367  0.0  0.1  22568  5116 pts/0    Ss   23:27   0:00 -bash
root            1384  0.0  0.1  52700  3812 tty1     S    23:29   0:00 sudo su
root            1385  0.0  0.1  52284  3448 tty1     S    23:29   0:00 su
root            1386  0.0  0.1  21224  3764 tty1     S+   23:29   0:00 bash
shared          1397  0.0  0.1  37364  3428 pts/0    R+   23:30   0:00 ps au
```
- The `TTY` column shows which terminal session each process belongs to — `tty1` suggests a direct/console login, while `pts/0` suggests a remote (e.g., SSH) session.
- Notice the root user here recently ran `sudo su` and `su` — both commands used to switch to another user's identity, a strong signal that someone with root access was recently active on this box.

---

## 4. System-Level Information

### 4.1 OS Version

Knowing the specific Linux distribution — Ubuntu, Debian, FreeBSD, Fedora, SUSE, Red Hat, CentOS, and so on — narrows down which tools and techniques are likely to be available or relevant. It also identifies the exact OS version, which matters because **publicly known exploits** are often tied to specific versions.

```bash
$ cat /etc/os-release

NAME="Ubuntu"
VERSION="20.04.4 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.4 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

- `cat /etc/os-release` prints a standard file that almost every modern Linux distribution ships, describing exactly which distro and version is installed.
- Here the target is Ubuntu 20.04.4 LTS, codenamed "Focal Fossa."
- Whatever version you find, it's worth checking whether it's still receiving security updates or has gone **end of life** (EOL — meaning the vendor no longer patches it, making it a much softer target). Ubuntu publishes its release/support schedule publicly, and "Focal Fossa" isn't due to reach EOL until April 2030. In other words: if a target is still inside its support window and has clearly been kept patched, you're less likely to stumble on a well-known kernel bug — but it's always still worth checking.

### 4.2 Kernel Version

The kernel is the core engine of the operating system. Like the OS version, specific kernel versions sometimes have known, publicly documented vulnerabilities with working exploits already published.

```bash
$ uname -a

Linux nixlpe02 5.4.0-122-generic #138-Ubuntu SMP Wed Jun 22 15:00:31 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

- `uname -a` ("unix name, all info") prints the hostname, kernel version, build details, and CPU architecture in one shot. (`cat /proc/version` is an alternative way to get similar information.)
- This tells you the exact kernel build, which you can then cross-reference against databases of known kernel exploits.

> **Caution:** kernel exploits are a double-edged sword. Running one can just as easily cause system instability or an outright crash as it can grant elevated access. On a production system (a live, real-world system actually being used by a business) — especially during a client engagement — this isn't a risk to take lightly. Fully understanding what a given kernel exploit actually does, and what could go wrong, is essential before ever running one.

### 4.3 Running Services and Processes

Anything running as **root** is worth a close look, since a flaw in that service inherits root's privileges automatically. Well-known services like Nagios, Exim, Samba, and ProFTPd have all had publicly documented flaws in the past, often with ready-made proof-of-concept exploits — for instance, **CVE-2016-9566**, a local privilege escalation flaw affecting Nagios Core versions before 4.2.4 (a "CVE" is a standardized ID number assigned to a publicly tracked vulnerability).

**Listing processes running as root:**
```bash
$ ps aux | grep root

root         1  1.3  0.1  37656  5664 ?        Ss   23:26   0:01 /sbin/init
root         2  0.0  0.0      0     0 ?        S    23:26   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    23:26   0:00 [ksoftirqd/0]
root         4  0.0  0.0      0     0 ?        S    23:26   0:00 [kworker/0:0]
root         5  0.0  0.0      0     0 ?        S<   23:26   0:00 [kworker/0:0H]
root         6  0.0  0.0      0     0 ?        S    23:26   0:00 [kworker/u8:0]
root         7  0.0  0.0      0     0 ?        S    23:26   0:00 [rcu_sched]
root         8  0.0  0.0      0     0 ?        S    23:26   0:00 [rcu_bh]
root         9  0.0  0.0      0     0 ?        S    23:26   0:00 [migration/0]

<SNIP>
```
- `ps aux` lists every running process on the system, in detail (`a` = all users' processes, `u` = show the owning user, `x` = include processes not attached to a terminal).
- Piping into `grep root` narrows the list down to anything running as the root user — worth scanning carefully for anything unusual, or any known-vulnerable service version.

### 4.4 Installed Packages and Versions

Just as with running services, outdated or vulnerable installed packages can offer an easy escalation path even if they're not actively running as a service right this moment. A concrete example: **Screen** (a terminal multiplexer, similar to `tmux`, that lets you open multiple virtual terminal "windows" inside a single session) — specifically version **4.05.00** — has a well-documented privilege escalation vulnerability that's straightforward to exploit once identified.

---

## 5. Orienting Yourself on the Box

### 5.1 PATH Variable

```bash
$ echo $PATH

/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

- `$PATH` is the list of folders Linux automatically searches, in order, whenever you type a command name without specifying its full location. For example, typing `id` works because the system checks each folder in `$PATH` until it finds an executable file named `id` — which in this case lives at `/usr/bin/id`.
- Think of `$PATH` like a list of cupboards a chef checks, in order, every time a recipe calls for "the big knife" — if someone secretly swaps in a different "big knife" in the first cupboard checked, the chef grabs that one without realizing it's not the usual tool. A misconfigured `$PATH` (for example, if a writable folder appears earlier in the list than it should) can be abused the same way, tricking a privileged process into running an attacker's version of a command instead of the real one. This is covered in more detail elsewhere, but it's worth noting down the moment you see it.

### 5.2 Environment Variables

```bash
$ env

SHELL=/bin/bash
PWD=/home/john
LOGNAME=john
XDG_SESSION_TYPE=tty
MOTD_SHOWN=pam
HOME=/home/john
LANG=en_US.UTF-8

<SNIP>
```

- `env` lists all environment variables currently set for your session — small named values the shell and other programs can read (current shell, working directory, username, language settings, and so on).
- Occasionally something sensitive — like a password someone set as a variable for convenience — turns up here by accident. Worth a quick scan every time.

### 5.3 CPU Info

```bash
$ lscpu 

Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   43 bits physical, 48 bits virtual
CPU(s):                          2
On-line CPU(s) list:             0,1
Thread(s) per core:              1
Core(s) per socket:              2
Socket(s):                       1
NUMA node(s):                    1
Vendor ID:                       AuthenticAMD
CPU family:                      23
Model:                           49
Model name:                      AMD EPYC 7302P 16-Core Processor
Stepping:                        0
CPU MHz:                         2994.375
BogoMIPS:                        5988.75
Hypervisor vendor:                VMware

<SNIP>
```

- `lscpu` ("list CPU") prints hardware details about the processor. The `Hypervisor vendor: VMware` line is a useful side note — it tells you this is a virtual machine running under VMware, rather than physical hardware.

### 5.4 Available Login Shells

```bash
$ cat /etc/shells

# /etc/shells: valid login shells
/bin/sh
/bin/bash
/usr/bin/bash
/bin/rbash
/usr/bin/rbash
/bin/dash
/usr/bin/dash
/usr/bin/tmux
/usr/bin/screen
```

- `/etc/shells` lists every program on the system that's registered as a valid login shell (the program that runs when a user logs in and starts typing commands).
- Both **Tmux** and **Screen** are available here — these are "terminal multiplexer" programs, meaning they let you split one terminal session into multiple independent virtual windows, keep sessions running in the background, and reattach to them later. Worth noting down since they can be handy for maintaining a session (and, per Section 4.4, specific Screen versions can themselves be a privilege escalation path).

### 5.5 Security Defenses in Place

It's worth checking what security protections exist on the host, since knowing what's in place (even without being able to see its exact configuration) helps avoid wasting time on techniques that are certain to get blocked or logged. Common things to look for:

| Defense | What It Generally Does |
|---|---|
| Exec Shield | Memory-protection feature that helps block certain exploit techniques |
| iptables | Linux's built-in firewall rule engine |
| AppArmor | Restricts what individual programs are allowed to do, even if run by a privileged user |
| SELinux | A stricter, mandatory access-control system (common on Red Hat-based distros) |
| Fail2ban | Watches logs and temporarily bans IPs after repeated failed login attempts |
| Snort | Network intrusion detection system — watches traffic for suspicious patterns |
| Uncomplicated Firewall (ufw) | A simplified front-end for managing `iptables` rules |

Often you won't have the privileges needed to inspect how these are actually configured, but just knowing *which ones exist at all* can help you decide where not to waste effort.

---

## 6. Users and Groups

Knowing which user accounts exist on a system matters because individual service accounts are frequently created specifically to limit what a given application or service is allowed to do — a deliberate security boundary. The reasoning: if a service is compromised while running as root, the attacker who broke in automatically inherits root's full privileges. Running things as a limited, dedicated user instead keeps the blast radius smaller if that one service gets compromised.

### 6.1 /etc/passwd Breakdown

Every user account on the system is listed in `/etc/passwd`, which is readable by **everyone** on the system by design. Each line follows this format:

```
username : password : UID : GID : user_ID_info : home_directory : shell
```

| Field | Meaning |
|---|---|
| Username | The account's login name |
| Password | Normally just an `x`, meaning the real password hash lives elsewhere, in `/etc/shadow` (which only root can read) |
| UID | User ID — a unique number identifying the account |
| GID | Group ID — the user's primary group |
| User ID info | Optional free-text comment field (full name, description, etc.) |
| Home directory | Where that user's personal files live |
| Shell | The program that runs when that user logs in |

```bash
$ cat /etc/passwd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
<...SNIP...>
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
mickey:x:1000:1000:mickey,,,:/home/mickey:/bin/bash
colord:x:111:118:colord colour management daemon,,,:/var/lib/colord:/bin/false
backupsvc:x:1001:1001::/home/backupsvc:
indie.joe:x:1002:1002::/home/indie.joe:
goffy.d:x:1003:1003::/home/goffy.d:
logger:x:1004:1004::/home/logger:
shared:x:1005:1005::/home/shared:
pluto:x:1006:1006::/home/pluto:
sysadm:$6$vdH7vuQIv6anIBWg$Ysk.UZzI7WxYUBYt8WRIWF0EzWlksOElDE0HLYinee38QI1A.0HW7WZCrUhZ9wwDz13bPpkTjNuRoUGYhwFE11:1007:1007::/home/sysadm:
```

- Accounts with low UIDs (`root`, `daemon`, `bin`, `sys`, etc.) are **system accounts** — created automatically by the OS or installed software, not meant for an actual human to log into interactively (notice most of them end in `/usr/sbin/nologin` or `/bin/false`, meaning login is blocked outright).
- Accounts from UID 1000 upward (`mickey`, `backupsvc`, `pluto`, etc.) are **regular human/service accounts**.
- Most entries here show an `x` in the password field — meaning the actual hash is stored separately in `/etc/shadow`, as expected. The `sysadm` line is the exception, and is covered next.

### 6.2 Readable Shadow File / Hashes in /etc/passwd

**`/etc/shadow`** is where Linux normally stores password hashes — and critically, unlike `/etc/passwd`, it's supposed to be readable only by root. If it turns out to be readable by the current user (due to misconfiguration), every user's password hash on the system becomes available for an **offline password cracking attack** (repeatedly guessing passwords, hashing each guess, and comparing it against the stolen hash, all without touching the live system and so without triggering any lockout or logging).

Occasionally, a password hash shows up directly inside **`/etc/passwd`** itself instead of just an `x` — a file that, unlike `/etc/shadow`, is readable by every user on the system by design. This isn't a common setup on general-purpose servers, but it does occasionally appear on embedded devices and routers.

Looking back at the `/etc/passwd` listing in Section 6.1: the `sysadm` line has an actual password hash (starting with `$6$`, indicating the SHA-512 hashing algorithm — see the table below) sitting directly in this normally-world-readable file. This is exactly the misconfiguration described above, and a prime candidate for an offline cracking attempt.

### 6.3 Password Hash Types

Different hashing algorithms produce hashes with recognizable prefixes. Being able to spot which algorithm produced a given hash at a glance helps you pick the right cracking tool/settings later:

| Algorithm | Hash Prefix |
|---|---|
| Salted MD5 | `$1$...` |
| SHA-256 | `$5$...` |
| SHA-512 | `$6$...` |
| BCrypt | `$2a$...` |
| Scrypt | `$7$...` |
| Argon2 | `$argon2i$...` |

### 6.4 Users With Login Shells

```bash
$ grep "sh$" /etc/passwd

root:x:0:0:root:/root:/bin/bash
mickey:x:1000:1000:mickey:/home/mickey:/bin/bash
indie.joe:x:1001:1001::/home/indie.joe:/bin/sh
administrator.evilcorp:x:1002:1002::/home/administrator.evilcorp:/bin/sh
backupsvc:x:1003:1003::/home/backupsvc:/bin/sh
goffy.d:x:1004:1004::/home/goffy.d:/bin/bash
logger:x:1005:1005::/home/logger:/bin/sh
shared:x:1006:1006::/home/shared:/bin/sh
pluto:x:1007:1007::/home/pluto:/bin/bash
john:x:1008:1008::/home/john:/bin/bash
```

- `grep "sh$"` searches `/etc/passwd` for any line ending in "sh" — a quick way to filter down to only accounts that actually have a usable login shell (`bash`, `sh`, etc.), skipping all the system accounts set to `nologin`.
- Once you know which shells are in use, it's worth checking their specific versions for known vulnerabilities — for example, outdated Bash versions like 4.1 are vulnerable to the well-known **Shellshock** exploit (a flaw that lets an attacker smuggle extra commands into Bash through specially crafted environment variables).

### 6.5 Groups and getent

Every user belongs to one or more **groups**, and group membership grants specific privileges. Think of it like building access badges: a folder meant only for the "developers" team requires a user's badge to be coded for the "developers" group before the door will open — being a regular employee badge-holder isn't enough on its own. Group definitions live in `/etc/group`.

```bash
$ cat /etc/group

root:x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
adm:x:4:syslog,john
tty:x:5:syslog
disk:x:6:
lp:x:7:
mail:x:8:
news:x:9:
uucp:x:10:
man:x:12:
proxy:x:13:
kmem:x:15:
dialout:x:20:
fax:x:21:
voice:x:22:
cdrom:x:24:john
floppy:x:25:
tape:x:26:
sudo:x:27:mickey,john
audio:x:29:pulse
dip:x:30:john
www-data:x:33:
...SNIP...
```

- Each line follows the format `group_name:password:GID:member_list`.

```bash
$ getent group sudo

sudo:x:27:mickey
```

- `getent group <name>` is a cleaner, more reliable way to pull the member list for one specific group — here, showing that `mickey` belongs to the `sudo` group, meaning that user can potentially run commands as root (subject to whatever sudo rules are configured — checked with `sudo -l`, covered in Section 7.5).

---

## 7. User-Level Information and Home Directories

### 7.1 Home Directories Overview

```bash
$ ls /home

backupsvc  indie.joe  goffy.d  logger  mickey  shared  pluto
```

- `ls /home` lists every user who has a personal home folder on this system. Each one is worth checking for sensitive data.
- Things specifically worth hunting for inside each home directory:
  - Readable `.bash_history` files — these log previously typed commands and can reveal credentials typed on the command line, or hints about what the server is used for.
  - Configuration files that might contain hardcoded credentials — sometimes even credentials that lead into the broader Active Directory environment.
  - **SSH keys** — these matter for three separate reasons: they can be used to maintain persistence (a reliable way back into this same box later), to escalate privileges (if the key belongs to a more privileged user), or to pivot and move laterally into other systems on the network (if the same key happens to also grant access elsewhere).
- A good minimum check: compare the hosts listed in the **ARP cache** (Section 9.8) against any usable SSH private keys you find — a key might unlock one of those other machines the host has recently been talking to.
- Finally: if you've found *any* passwords anywhere during enumeration, try them against every user account on the system. **Password reuse is extremely common**, so testing one found password broadly can pay off more than you'd expect.

### 7.2 Inspecting a User's Directory

```bash
$ ls -la /home/pluto/

total 32
drwxr-xr-x 3 pluto pluto 4096 Aug 30 23:37 .
drwxr-xr-x 9 root           root           4096 Aug 30 23:33 ..
-rw------- 1 pluto pluto   41 Aug 30 23:35 .bash_history
-rw-r--r-- 1 pluto pluto  220 Sep  1  2015 .bash_logout
-rw-r--r-- 1 pluto pluto 3771 Sep  1  2015 .bashrc
-rw-r--r-- 1 pluto pluto   97 Aug 30 23:37 config.json
-rw-r--r-- 1 pluto pluto  655 May 16  2017 .profile
drwx------ 2 pluto pluto 4096 Aug 30 23:35 .ssh
```

- `ls -la` lists all files (including hidden "dotfiles" starting with `.`) with full permission and ownership details.
- The permission string at the start of each line (e.g., `-rw-------`) shows who can read, write, or execute that file — `-rw-------` means only the owning user can read or write it, while `-rw-r--r--` means anyone on the system can at least read it.
- A `config.json` sitting right alongside standard shell config files is worth a closer look — application config files are a common place to find hardcoded credentials.

### 7.3 SSH Keys

If an SSH private key turns up for the current user, it could potentially be used to open a new, fully interactive SSH session on the same host (if SSH is exposed), or even be reused to authenticate to *other* systems in the network, if the same key happens to grant access elsewhere.

```bash
$ ls -l ~/.ssh

total 8
-rw------- 1 mickey mickey 1679 Aug 30 23:37 id_rsa
-rw-r--r-- 1 mickey mickey  393 Aug 30 23:37 id_rsa.pub
```
*`id_rsa` is the private key (and should normally be readable only by its owner, per the `-rw-------` permissions shown) — `id_rsa.pub` is the corresponding public key, which is safe to share and doesn't grant access on its own.*

> **Tip:** at minimum, checking the **ARP cache** (Section 9.8) and cross-referencing it against any usable private keys found can reveal promising lateral movement targets.

### 7.4 Bash History

A user's shell command history can reveal passwords typed directly on the command line, git repository interactions, cron job setup, and all sorts of other clues about what this particular server is used for and how it's typically administered.

```bash
$ history

    1  id
    2  cd /home/goffy.d
    3  exit
    4  touch backup.sh
    5  tail /var/log/apache2/error.log
    6  ssh ec2-user@dmz02.evilcorp.local
    7  history
```
*Even this short, fairly mundane-looking history reveals something useful: an SSH connection was made to another host (`dmz02.evilcorp.local`) as a different user (`ec2-user`) — a lead worth following up on, since it suggests this box has some relationship with that other server.*

### 7.5 Sudo Privileges

**Sudo** lets an authorized user run specific commands (or, in some configurations, *any* command) as another user — commonly root. Checking what the current user is actually permitted to run via sudo is an essential step, since a misconfigured sudo rule can be a direct path to root.

```bash
$ sudo -l

Matching Defaults entries for sysadm on NIX02:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User sysadm may run the following commands on NIX02:
    (root) NOPASSWD: /usr/sbin/tcpdump
```
- `sudo -l` ("list") shows exactly which commands the current user is allowed to run with elevated privileges, and as which user.
- The key detail here is **`NOPASSWD`** — meaning the `sysadm` user can run `tcpdump` as root *without being prompted for a password at all*.
- Whether this specific permission is actually useful for escalating privileges depends entirely on what that particular binary can be made to do (some binaries, even run as root, don't lead anywhere useful) — but a `NOPASSWD` entry is always worth investigating closely.

> **The simplest possible case:** if a user happens to have full, unrestricted sudo privileges — meaning they can run *any* command as root — a plain `sudo su` command (switching to the root user) immediately grants a full root session, no further exploitation needed at all.

---

## 8. Files and Filesystem Clues

Beyond home directories, it's worth searching the whole filesystem for "low-hanging fruit."

### 8.1 Configuration Files

Configuration files (commonly ending in `.conf` or `.config`) are worth searching broadly across the filesystem, since they frequently contain usernames, passwords, API keys, or other secrets needed for whatever service or application they configure.

### 8.2 Cron Jobs

**Cron jobs** are Linux's equivalent of Windows scheduled tasks — commands or scripts set to run automatically on a schedule, often for routine maintenance or backups. When a cron job combines with another misconfiguration — such as a script using a relative (rather than absolute) file path, or weak file permissions — it can become an avenue for privilege escalation, especially if that cron job runs as root.

```bash
$ ls -la /etc/cron.daily/

total 60
drwxr-xr-x  2 root root 4096 Aug 30 23:49 .
drwxr-xr-x 93 root root 4096 Aug 30 23:47 ..
-rwxr-xr-x  1 root root  376 Mar 31  2016 apport
-rwxr-xr-x  1 root root 1474 Sep 26  2017 apt-compat
-rwx--x--x  1 root root  379 Aug 30 23:49 backup
-rwxr-xr-x  1 root root  355 May 22  2012 bsdmainutils
-rwxr-xr-x  1 root root 1597 Nov 27  2015 dpkg
-rwxr-xr-x  1 root root  372 May  6  2015 logrotate
-rwxr-xr-x  1 root root 1293 Nov  6  2015 man-db
-rwxr-xr-x  1 root root  539 Jul 16  2014 mdadm
-rwxr-xr-x  1 root root  435 Nov 18  2014 mlocate
-rwxr-xr-x  1 root root  249 Nov 12  2015 passwd
-rw-r--r--  1 root root  102 Apr  5  2016 .placeholder
-rwxr-xr-x  1 root root 3449 Feb 26  2016 popularity-contest
-rwxr-xr-x  1 root root  214 May 24  2016 update-notifier-common
```
- `/etc/cron.daily/` holds scripts that run once a day, all owned by root in this listing — meaning whatever these scripts actually do, they run with root's privileges.
- The `backup` entry (modified far more recently than the others, on the same date as this enumeration) stands out as worth a closer look, since it's clearly been customized rather than left as a stock system script.

### 8.3 SETUID and SETGID Binaries

**SETUID** and **SETGID** are special file permission bits that let a program run with the privileges of the file's *owner* (SETUID) or *group* (SETGID), rather than the privileges of whoever actually launched it. This exists for entirely legitimate reasons — it lets an ordinary user run a specific command that needs elevated privileges for just that one task, without needing to grant that user full root access generally.

The catch: many common binaries with SETUID set have functionality that can be abused to spawn a root-level shell, if that functionality wasn't intentionally designed with this risk in mind (or if a specific, documented SETUID exploitation technique exists for that particular binary).

### 8.4 Writable Directories and Files

Knowing which directories are writable matters for two separate reasons:
- Practically, it tells you where you're actually able to drop tools or scripts you need to transfer onto the system
- Strategically, discovering that a cron job writes its output into a writable directory can hint at how frequently that job runs — and if the *script* that cron job executes is also writable, that's a direct path to getting your own code to run with that job's privileges (commonly root's)

**Finding writable directories:**
```bash
$ find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null

/dmz-backups
/tmp
/tmp/VMwareDnD
/tmp/.XIM-unix
/tmp/.Test-unix
/tmp/.X11-unix
/tmp/systemd-private-8a2c51fcbad240d09578916b47b0bb17-systemd-timesyncd.service-TIecv0/tmp
/tmp/.font-unix
/tmp/.ICE-unix
/proc
/dev/mqueue
/dev/shm
/var/tmp
/var/tmp/systemd-private-8a2c51fcbad240d09578916b47b0bb17-systemd-timesyncd.service-hm6Qdl/tmp
/var/crash
/run/lock
```
- `find / ... -prune ...` searches the entire filesystem starting from root (`/`), specifically excluding (`-prune`) the `/proc` virtual filesystem (which doesn't represent real files and would just add noise).
- `-type d -perm -o+w` filters for directories (`d`) where the "other" permission group (`o`, meaning anyone who isn't the owner or in the owning group) has write access (`+w`).
- `2>/dev/null` discards error messages (e.g., "permission denied" when trying to search folders the current user can't access), keeping the output clean.
- The standout result here is `/dmz-backups` — an unusual, non-standard writable directory that doesn't belong to the normal Linux filesystem layout, and therefore worth investigating closely.

**Finding writable files:**
```bash
$ find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

/etc/cron.daily/backup
/dmz-backups/backup.sh

<SNIP>

/home/backupsvc/backup.sh

<SNIP>
```
- Same logic as above, but `-type f` searches for files instead of directories.
- Finding `/etc/cron.daily/backup` writable here is a serious finding on its own, directly connecting back to the cron jobs in Section 8.2 — a world-writable script that root automatically executes daily is a textbook privilege escalation opportunity: modifying it to add a malicious command means that command runs as root the next time the cron job fires.

---

## 9. Drives, Mounts, and Network Info

### 9.1 Block Devices (lsblk)

```bash
$ lsblk

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   30G  0 disk 
├─sda1   8:1    0   29G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda3                      8:3    0   19G  0 part 
| └─ubuntu--vg-ubuntu--lv 253:0    0   18G  0 lvm  /
|
└─sda5   8:5    0  975M  0 part [SWAP]
sr0     11:0    1  848M  0 rom
```
- `lsblk` ("list block devices") shows all storage devices and partitions the system knows about, along with where each one is currently mounted (if at all).
- `MOUNTPOINT` being blank for a given partition means it exists but isn't currently attached anywhere in the filesystem — worth investigating whether it can be mounted and what it might contain.
- If an extra drive or an unmounted filesystem turns up here, it's worth trying to mount it (mounting requires root privileges), since it might contain sensitive files, documentation, or backups.
- Note here that this host's root filesystem (`/`) sits on an **LVM** volume (`ubuntu--vg-ubuntu--lv`) — LVM (Logical Volume Manager) is a flexible storage layer that lets Linux resize or combine disk partitions more easily than working with raw partitions directly.

### 9.2 Printers

The `lpstat` command can show information about printers attached to the system. If there are active or queued print jobs sitting around, they might expose sensitive documents that were sent to print.

### 9.3 fstab and Credential Hunting

```bash
$ cat /etc/fstab

# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-BdLsBLE4CvzJUgtkugkof4S0dZG7gWR8HCNOlRdLWoXVOba2tYUMzHfFQAP9ajul / ext4 defaults 0 0
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/20b1770d-a233-4780-900e-7c99bc974346 /boot ext4 defaults 0 0
```

- `/etc/fstab` ("filesystem table") lists filesystems the system knows how to automatically mount, and where.
- Worth checking: can you find any types of credentials by grepping this file for common keywords like `password`, `username`, or `credential`? Mount options occasionally include authentication details for network shares.

### 9.4 Mounted Filesystems (df -h)

```bash
$ df -h

Filesystem      Size  Used Avail Use% Mounted on
udev            1,9G     0  1,9G   0% /dev
tmpfs           389M  1,8M  388M   1% /run
/dev/sda5        20G  7,9G   11G  44% /
tmpfs           1,9G     0  1,9G   0% /dev/shm
tmpfs           5,0M  4,0K  5,0M   1% /run/lock
tmpfs           1,9G     0  1,9G   0% /sys/fs/cgroup
/dev/loop0      128K  128K     0 100% /snap/bare/5
/dev/loop1       62M   62M     0 100% /snap/core20/1611
/dev/loop2       92M   92M     0 100% /snap/gtk-common-themes/1535
/dev/loop4       55M   55M     0 100% /snap/snap-store/558
/dev/loop3      347M  347M     0 100% /snap/gnome-3-38-2004/115
/dev/loop5       47M   47M     0 100% /snap/snapd/16292
/dev/sda1       511M  4,0K  511M   1% /boot/efi
tmpfs           389M   24K  389M   1% /run/user/1000
/dev/sr0        3,6G  3,6G     0 100% /media/john/Ubuntu 20.04.5 LTS amd64
/dev/loop6       50M   50M     0 100% /snap/snapd/17576
/dev/loop7       64M   64M     0 100% /snap/core20/1695
/dev/loop8       46M   46M     0 100% /snap/snap-store/599
/dev/loop9      347M  347M     0 100% /snap/gnome-3-38-2004/119
```

- `df -h` ("disk free, human-readable") lists every currently mounted filesystem, its total size, how much is used, and where it's mounted.
- Note: this `df -h` example, the `lsblk` output in Section 9.1, and the `/etc/fstab` output above each come from a different illustrative session — treat every code block in this document as its own standalone example of a command and its typical output, rather than assuming every block describes the exact same machine.

### 9.5 Unmounted Filesystems

```bash
$ cat /etc/fstab | grep -v "#" | column -t

UUID=5bf16727-fcdf-4205-906c-0620aa4a058f  /          ext4  errors=remount-ro  0  1
UUID=BE56-AAE0                             /boot/efi  vfat  umask=0077         0  1
/swapfile                                  none       swap  sw                 0  0
```

- `grep -v "#"` filters *out* any comment lines (lines starting with `#`), and `column -t` neatly aligns the remaining output into columns for easier reading.
- An **unmounted filesystem** is one that exists but isn't currently attached to any folder, meaning it's not directly accessible — sometimes deliberately, to keep certain scripts, documents, or data away from a standard (non-root) user. If you can escalate to root, you gain the ability to mount and inspect these filesystems yourself. (See also Section 8.4's blank-`MOUNTPOINT` entries in `lsblk`, which flag unmounted partitions the same way.)

### 9.6 Routing Table

```bash
$ route

Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    0      0        0 ens192
10.129.0.0      0.0.0.0         255.255.0.0     U     0      0        0 ens192
```

- `route` (or equivalently `netstat -rn`) shows which networks this host can reach, and through which network interface. This is especially useful for spotting additional internal subnets beyond the one you initially landed in.

### 9.7 DNS Config

In a domain environment (a network managed by Windows Active Directory), it's worth checking `/etc/resolv.conf`. **If this Linux host is configured to use an internal DNS server, that server is often a useful starting point for querying information about the Active Directory environment.**

### 9.8 ARP Table

```bash
$ arp -a

_gateway (10.129.0.1) at 00:50:56:b9:b9:fc [ether] on ens192
```

- `arp -a` shows the **ARP cache** — a local record of which other devices on the same network segment this host has recently talked to, each one tied to its hardware (MAC) address. This can hint at what other machines are reachable and worth investigating, even when you have no other map of the network. As noted in Sections 7.1 and 7.3, this list is worth cross-referencing against any SSH keys you find.

---

## 10. Hidden Files, Hidden Directories, and Temp Folders

Many files and folders on Linux are kept "hidden" (any name starting with a `.`) simply to reduce clutter and prevent accidental edits to important configuration files — not as a security measure on their own. Even so, hidden files can contain sensitive information, so it's worth specifically searching for them, even in folders where you only have read access.

**All Hidden Files**

```bash
$ find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null | grep john

-rw-r--r-- 1 john john 3771 Nov 27 11:16 /home/john/.bashrc
-rw-rw-r-- 1 john john 180 Nov 27 11:36 /home/john/.wget-hsts
-rw------- 1 john john 387 Nov 27 14:02 /home/john/.bash_history
-rw-r--r-- 1 john john 807 Nov 27 11:16 /home/john/.profile
-rw-r--r-- 1 john john 0 Nov 27 11:31 /home/john/.sudo_as_admin_successful
-rw-r--r-- 1 john john 220 Nov 27 11:16 /home/john/.bash_logout
-rw-rw-r-- 1 john john 162 Nov 28 13:26 /home/john/.notes
```

- `find / -type f -name ".*"` searches the entire filesystem (`/`) for files (`-type f`) whose name starts with a dot. `-exec ls -l {} \;` runs a detailed listing on every match found, and `2>/dev/null` throws away "permission denied" error noise.
- The `.notes` file at the end stands out immediately — it's not a standard dotfile that every Linux account automatically has, so it's worth reading right away.

**All Hidden Directories**

```bash
$ find / -type d -name ".*" -ls 2>/dev/null

   684822      4 drwx------   3 john john     4096 Nov 28 12:32 /home/john/.gnupg
   790793      4 drwx------   2 john john     4096 Okt 27 11:31 /home/john/.ssh
   684804      4 drwx------  10 john john     4096 Okt 27 11:30 /home/htb-
```

- Same idea as above, but hunting for directories (`-type d`) instead of files.
- `.gnupg` is the config/key storage folder for GPG (GNU Privacy Guard, an encryption tool), and `.ssh` is the standard folder for SSH keys and configuration — both are always worth checking when found.
- *(Note: the third line is cut off exactly as it appeared in the source output — this is simply where the pasted material was truncated, not something to read into.)*

**Temporary Directories**

Linux has three default locations meant for temporary files, all readable by every user on the system, and all worth checking for leftover logs or script output:

| Location | Cleared When | Typical Retention |
|---|---|---|
| `/tmp` | Immediately on every reboot | Files auto-deleted after about 10 days even without a reboot |
| `/var/tmp` | Survives reboots | Files auto-deleted after about 30 days |
| `/dev/shm` | Shared memory-backed temp space | Cleared on reboot (acts similarly to `/tmp`) |

Because `/var/tmp` survives a reboot, programs that need to temporarily store data *across* a restart typically use it instead of `/tmp`.

```bash
$ ls -l /tmp /var/tmp /dev/shm

/dev/shm:
total 0

/tmp:
total 52
-rw------- 1 john john    0 Nov 28 12:32 config-err-v8LfEU
drwx------ 3 root        root        4096 Nov 28 12:37 snap.snap-store
drwx------ 2 john john 4096 Nov 28 12:32 ssh-OKlLKjlc98xh
<SNIP>
drwx------ 2 john john 4096 Nov 28 12:37 tracker-extract-files.1000
drwx------ 2 gdm         gdm         4096 Nov 28 12:31 tracker-extract-files.125

/var/tmp:
total 28
drwx------ 3 root root 4096 Nov 28 12:31 systemd-private-7b455e62ec09484b87eff41023c4ca53-colord.service-RrPcyi
drwx------ 3 root root 4096 Nov 28 12:31 systemd-private-7b455e62ec09484b87eff41023c4ca53-ModemManager.service-4Rej9e
...SNIP...
```

Note that the same three locations (`/tmp`, `/var/tmp`, `/dev/shm`) also show up in Section 8.4's writable-directories search — unsurprising, since they're world-writable by design.

---

## 11. Decision Flow: Where Do I Look First?

```
                    ┌─────────────────────────────────────┐
                    │ Just landed a low-privileged        │
                    │ shell — where to start enumerating? │
                    └──────────────────┬──────────────────┘
                                       │
         ┌─────────────────────────────┼──────────────────────────────┐
         │                             │                              │
   Quick wins first              Check what this user          Check the filesystem
   (misconfigurations)           is specifically allowed         for artifacts/clues
         │                        to do                               │
         ▼                              ▼                             ▼
┌─────────────────────┐      ┌────────────────────────┐      ┌───────────────────────┐
│ sudo -l             │      │ Does current user have │      │ /etc/passwd readable  │
│ → NOPASSWD entries? │      │ SSH keys, readable     │      │ hashes? /etc/shadow   │
│ → full sudo access? │      │ home dirs, or bash     │      │ readable at all?      │
└─────────────────────┘      │ history worth reading? │      └───────────────────────┘
         |                   └────────────────────────┘                |
         │                              │                              │
         ▼                              ▼                              ▼
┌───────────────────┐      ┌────────────────────────┐      ┌────────────────────────┐
│ SETUID/SETGID     │      │ Writable directories/  │      │ Cron jobs — especially │
│ binaries worth    │      │ files, especially ones │      │ ones combined with a   │
│ exploiting?       │      │ tied to a cron job     │      │ writable script        │
└───────────────────┘      └────────────────────────┘      └────────────────────────┘
         │                              │                              │
         └──────────────────────────────┴──────────────────────────────┘
                                        │
                                        ▼
                         ┌──────────────────────────────────────┐
                         │ Still stuck? Dig deeper: OS/kernel   │
                         │ version against public exploits,     │
                         │ running services/packages,           │
                         │ lsblk / fstab / df -h for extra or   │
                         │ unmounted drives, hidden files &     │
                         │ directories, writable temp folders   │
                         │ (exercise caution with kernel        │
                         │ exploits on live/production systems) │
                         └──────────────────────────────────────┘
```

---

## 12. Cheat Sheet

| Goal | Command |
|---|---|
| Current user | `whoami` |
| Current user's groups | `id` |
| Server name | `hostname` |
| Network interfaces/subnets | `ifconfig` or `ip a` |
| Check sudo permissions | `sudo -l` |
| List terminal-attached processes (who's logged in, doing what) | `ps au` |
| OS name/version | `cat /etc/os-release` |
| Kernel/build info | `uname -a` (or `cat /proc/version`) |
| List processes running as root | `ps aux \| grep root` |
| Current PATH | `echo $PATH` |
| Environment variables | `env` |
| CPU/hardware info | `lscpu` |
| Valid login shells | `cat /etc/shells` |
| List all users | `cat /etc/passwd` |
| Filter to accounts with a real shell | `grep "sh$" /etc/passwd` |
| List all groups | `cat /etc/group` |
| List members of one group | `getent group <groupname>` |
| List home directories | `ls /home` |
| Inspect a user's home directory in detail | `ls -la /home/username/` |
| Check for SSH keys | `ls -l ~/.ssh` |
| View command history | `history` |
| List daily cron jobs | `ls -la /etc/cron.daily/` |
| List block devices/drives | `lsblk` |
| Printer jobs | `lpstat` |
| Static mount config | `cat /etc/fstab` |
| Currently mounted filesystems | `df -h` |
| Unmounted filesystems (cleaned up) | `cat /etc/fstab \| grep -v "#" \| column -t` |
| Routing table | `route` or `netstat -rn` |
| DNS config | `cat /etc/resolv.conf` |
| ARP cache (recent hosts contacted) | `arp -a` |
| Find world-writable directories | `find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null` |
| Find world-writable files | `find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null` |
| Find all hidden files | `find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null` |
| Find all hidden directories | `find / -type d -name ".*" -ls 2>/dev/null` |
| Check temp directories | `ls -l /tmp /var/tmp /dev/shm` |

---

## 13. Big-Picture Takeaways

1. **Enumeration is the real skill — exploits are just the payoff.** Automated tools like LinPEAS and LinEnum speed things up, but knowing *why* each piece of information matters (and being able to go find it manually, or recognize something unusual when a script doesn't flag it) is what actually gets you to root.
2. **Misconfigurations are usually easier wins than exploits.** A `NOPASSWD` sudo entry, a world-writable cron script, or a readable `/etc/shadow` (or a stray hash in `/etc/passwd`) typically require far less risk and effort than hunting for and safely deploying a kernel exploit.
3. **Version numbers are leads, not verdicts.** An OS, kernel, service, or installed-package version isn't automatically exploitable just because it's old — but it's always worth checking against known public exploits, carefully, and with extra caution around kernel exploits specifically because of their potential to crash the system.
4. **Readable-by-everyone files are a goldmine.** `/etc/passwd`, `/etc/group`, `/etc/fstab`, and `/etc/shells` are all world-readable by design, and all can leak useful details — occasionally including an outright password hash sitting somewhere it shouldn't be.
5. **Small, mundane-looking details often connect to something bigger.** A single SSH command in someone's bash history, a slightly-too-recent file timestamp in a cron directory, or one unusual line in `/etc/passwd` can each be the thread that unravels a full escalation path once followed up on.
6. **Permissions tell a story, not just a technical detail.** The difference between `-rw-------` and `-rw-r--r--`, or between a directory with and without the "other write" bit set, is frequently the entire reason a particular escalation path exists at all.
7. **Password reuse is common — always test what you find.** Any password or hash uncovered anywhere on the system is worth trying against every other user account.
8. **Hidden doesn't mean secure.** Dotfiles and dot-directories are hidden from a normal directory listing purely for tidiness, not for protection — they're well worth specifically searching for.
9. **Different example outputs may come from different lab machines.** Not every code block in this write-up necessarily reflects one continuous session on one box — treat each as an illustration of a command's typical output rather than assuming perfect continuity across sections.
10. **This is a foundation, not a complete methodology.** Manual enumeration across users, processes, system info, files, permissions, and cron jobs builds the base picture that specific privilege-escalation techniques get applied against later.
