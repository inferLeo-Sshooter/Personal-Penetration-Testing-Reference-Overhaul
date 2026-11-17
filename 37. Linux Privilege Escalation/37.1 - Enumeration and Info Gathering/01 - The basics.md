# Introduction to Linux Privilege Escalation

## Table of Contents
1. [What Is Privilege Escalation](#1-What-Is-Privilege-Escalation)
2. [Why Enumeration Comes First](#2-why-enumeration-comes-first)
3. [System-Level Information](#3-system-level-information)
   - 3.1 [OS Version](#31-os-version)
   - 3.2 [Kernel Version](#32-kernel-version)
   - 3.3 [Running Services and Processes](#33-running-services-and-processes)
   - 3.4 [Installed Packages and Versions](#34-installed-packages-and-versions)
4. [User-Level Information](#4-user-level-information)
   - 4.1 [Logged-In Users](#41-logged-in-users)
   - 4.2 [User Home Directories](#42-user-home-directories)
   - 4.3 [SSH Keys](#43-ssh-keys)
   - 4.4 [Bash History](#44-bash-history)
   - 4.5 [Sudo Privileges](#45-sudo-privileges)
5. [Files and Filesystem Clues](#5-files-and-filesystem-clues)
   - 5.1 [Configuration Files](#51-configuration-files)
   - 5.2 [Readable Shadow File / Hashes in /etc/passwd](#52-readable-shadow-file--hashes-in-etcpasswd)
   - 5.3 [Cron Jobs](#53-cron-jobs)
   - 5.4 [Unmounted Filesystems and Additional Drives](#54-unmounted-filesystems-and-additional-drives)
   - 5.5 [SETUID and SETGID Binaries](#55-setuid-and-setgid-binaries)
   - 5.6 [Writable Directories and Files](#56-writable-directories-and-files)
6. [Decision Flow: Where Do I Look First?](#6-decision-flow-where-do-i-look-first)
7. [Cheat Sheet](#7-cheat-sheet)
8. [Big-Picture Takeaways](#8-big-picture-takeaways)

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

**Enumeration** just means "systematically gathering information." Think of it like a locksmith casing a building before trying any locks — checking which doors are unlocked, which windows are weak, which keys are lying around in plain sight.
 
There are helper scripts (like [LinEnum](https://github.com/rebootuser/LinEnum)) that automate a lot of this, but it's important to know how to do it by hand too, since automated tools can miss things or might not be available.
 
When you first land on a box, there's a checklist of things worth looking at — covered one by one below.

---

## 3. System-Level Information

### 3.1 OS Version

Knowing the specific Linux distribution — Ubuntu, Debian, FreeBSD, Fedora, SUSE, Red Hat, CentOS, and so on — narrows down which tools and techniques are likely to be available or relevant. It also identifies the exact OS version, which matters because **publicly known exploits** are often tied to specific versions.

### 3.2 Kernel Version

The kernel is the core of the operating system, and like the OS version, specific kernel versions sometimes have known, publicly documented vulnerabilities with working exploits already published.

> **Important caution:** kernel exploits are a double-edged sword. Running one can just as easily cause system instability or an outright crash as it can grant elevated access. On a production system — especially during a client engagement — this isn't a risk to take lightly. Fully understanding what a given kernel exploit actually does, and what could go wrong, is essential before ever running one.

### 3.3 Running Services and Processes

Knowing what's actively running on a host — and critically, what's running **as root** — matters because a vulnerable or misconfigured service running with root privileges can be an easy path to escalation. Well-known services like Nagios, Exim, Samba, and ProFTPd have all had publicly documented flaws in the past, often with ready-made proof-of-concept exploits (for example, **CVE-2016-9566**, a local privilege escalation flaw affecting Nagios Core versions before 4.2.4).

**Listing processes running as root:**
```
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

### 3.4 Installed Packages and Versions

Just as with running services, outdated or vulnerable installed packages can offer an easy escalation path even if they're not actively running as a service right this moment. A concrete example: **Screen** (a terminal multiplexer, similar to `tmux`, that lets you open multiple virtual terminal "windows" inside a single session) — specifically version **4.05.00** — has a well-documented privilege escalation vulnerability that's straightforward to exploit once identified.

---

## 4. User-Level Information

### 4.1 Logged-In Users

Knowing who else is logged into the system, and what they're actively doing, can reveal lateral movement opportunities or hint at privilege escalation paths that wouldn't be obvious otherwise.

**Listing terminal-attached processes (i.e., what logged-in users are actually running):**
```
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

### 4.2 User Home Directories

Checking whether other users' home directories are accessible can turn up SSH keys usable to access other systems, or scripts and configuration files that happen to contain hardcoded credentials — sometimes even credentials that lead into the broader Active Directory environment.

**Listing available home directories:**
```
$ ls /home

backupsvc  indie.joe  goffy.d  logger  mickey  shared  pluto
```

**Inspecting one user's directory contents:**
```
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

### 4.3 SSH Keys

If an SSH private key turns up for the current user, it could potentially be used to open a new, fully interactive SSH session on the same host (if SSH is exposed), or even be reused to authenticate to *other* systems in the network, if the same key happens to grant access elsewhere.

```
$ ls -l ~/.ssh

total 8
-rw------- 1 mickey mickey 1679 Aug 30 23:37 id_rsa
-rw-r--r-- 1 mickey mickey  393 Aug 30 23:37 id_rsa.pub
```
*`id_rsa` is the private key (and should normally be readable only by its owner, per the `-rw-------` permissions shown) — `id_rsa.pub` is the corresponding public key, which is safe to share and doesn't grant access on its own.*

> **Tip:** at minimum, checking the **ARP cache** (a local record of which other devices on the network this host has recently communicated with) and cross-referencing it against any usable private keys found can reveal promising lateral movement targets.

### 4.4 Bash History

A user's shell command history can reveal passwords typed directly on the command line, git repository interactions, cron job setup, and all sorts of other clues about what this particular server is used for and how it's typically administered.

```
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

### 4.5 Sudo Privileges

**Sudo** lets an authorized user run specific commands (or, in some configurations, *any* command) as another user — commonly root. Checking what the current user is actually permitted to run via sudo is an essential step, since a misconfigured sudo rule can be a direct path to root.

```
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

## 5. Files and Filesystem Clues

### 5.1 Configuration Files

Configuration files (commonly ending in `.conf` or `.config`) are worth searching broadly across the filesystem, since they frequently contain usernames, passwords, API keys, or other secrets needed for whatever service or application they configure.

### 5.2 Readable Shadow File / Hashes in /etc/passwd

**`/etc/shadow`** is where Linux normally stores password hashes — and critically, unlike `/etc/passwd`, it's supposed to be readable only by root. If it turns out to be readable by the current user (due to misconfiguration), every user's password hash on the system becomes available for an **offline brute-force/cracking attack** — attempting to guess the original password by hashing many candidate passwords and comparing the results, without needing to interact with the live system at all.

Occasionally, password hashes show up directly inside **`/etc/passwd`** itself instead — a file that, unlike `/etc/shadow`, is readable by every user on the system by design. This configuration isn't common on modern general-purpose Linux systems, but does occasionally appear on embedded devices and routers.

```
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
- Each line follows the format `username:password_field:UID:GID:comment:home_directory:shell`.
- Most entries here show an `x` in the password field — meaning the actual hash is stored separately in `/etc/shadow`, as expected.
- The `sysadm` line is the exception: it has an actual password hash (starting with `$6$`, indicating the SHA-512 hashing algorithm) sitting directly in this normally-world-readable file — exactly the misconfiguration described above, and a prime candidate for an offline cracking attempt.

### 5.3 Cron Jobs

**Cron jobs** are Linux's equivalent of Windows scheduled tasks — commands or scripts set to run automatically on a schedule, often for routine maintenance or backups. When a cron job combines with another misconfiguration — such as a script using a relative (rather than absolute) file path, or weak file permissions — it can become an avenue for privilege escalation, especially if that cron job runs as root.

```
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

### 5.4 Unmounted Filesystems and Additional Drives

Discovering — and successfully mounting — an additional drive or filesystem that isn't currently in use can turn up sensitive files, stored credentials, or backups, any of which might directly enable privilege escalation.

```
$ lsblk

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   30G  0 disk 
├─sda1   8:1    0   29G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0  975M  0 part [SWAP]
sr0     11:0    1  848M  0 rom
```
- `lsblk` ("list block devices") shows all storage devices and partitions the system knows about, along with where each one is currently mounted (if at all).
- `MOUNTPOINT` being blank for a given partition means it exists but isn't currently attached anywhere in the filesystem — worth investigating whether it can be mounted and what it might contain.

### 5.5 SETUID and SETGID Binaries

**SETUID** and **SETGID** are special file permission bits that let a program run with the privileges of the file's *owner* (SETUID) or *group* (SETGID), rather than the privileges of whoever actually launched it. This exists for entirely legitimate reasons — it lets an ordinary user run a specific command that needs elevated privileges for just that one task, without needing to grant that user full root access generally.

The catch: many common binaries with SETUID set have functionality that can be abused to spawn a root-level shell, if that functionality wasn't intentionally designed with this risk in mind (or if a specific, documented SETUID exploitation technique exists for that particular binary).

### 5.6 Writable Directories and Files

Knowing which directories are writable matters for two separate reasons: 
- Practically, it tells you where you're actually able to drop tools or scripts you need to transfer onto the system
- Strategically, discovering that a cron job writes its output into a writable directory can hint at how frequently that job runs — and if the *script* that cron job executes is also writable, that's a direct path to getting your own code to run with that job's privileges (commonly root's).

**Finding writable directories:**
```
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
```
$ find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

/etc/cron.daily/backup
/dmz-backups/backup.sh

<SNIP>

/home/backupsvc/backup.sh

<SNIP>
```
- Same logic as above, but `-type f` searches for files instead of directories.
- Finding `/etc/cron.daily/backup` writable here is a serious finding on its own, directly connecting back to the cron jobs section — a world-writable script that root automatically executes daily is a textbook privilege escalation opportunity: modifying it to add a malicious command means that command runs as root the next time the cron job fires.

---

## 6. Decision Flow: Where Do I Look First?

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
                         │ Still stuck? Check OS/kernel         │
                         │ version against public exploits,     │
                         │ and running services/packages        │
                         │ for known vulnerabilities            │
                         │ (exercise caution with kernel        │
                         │ exploits on live/production systems) │
                         └──────────────────────────────────────┘
```

---

## 7. Cheat Sheet

| Goal | Command |
|---|---|
| List processes running as root | `ps aux \| grep root` |
| List terminal-attached processes (who's logged in, doing what) | `ps au` |
| List home directories | `ls /home` |
| Inspect a user's home directory in detail | `ls -la /home/username/` |
| Check for SSH keys | `ls -l ~/.ssh` |
| View command history | `history` |
| Check current user's sudo permissions | `sudo -l` |
| View /etc/passwd (check for embedded hashes) | `cat /etc/passwd` |
| List daily cron jobs | `ls -la /etc/cron.daily/` |
| List block devices/filesystems | `lsblk` |
| Find world-writable directories | `find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null` |
| Find world-writable files | `find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null` |

---

## 8. Big-Picture Takeaways

1. **Enumeration is the actual skill — exploits are just the payoff.** Scripts like LinEnum can speed up data collection, but knowing *why* each piece of information matters (and recognizing something unusual when a script doesn't flag it) is what actually leads to successful privilege escalation.
2. **Misconfigurations are usually easier wins than exploits.** A `NOPASSWD` sudo entry, a world-writable cron script, or a readable `/etc/shadow` typically require far less risk and effort than hunting for and safely deploying a kernel exploit.
3. **Small, mundane-looking details often connect to something bigger.** A single SSH command in someone's bash history, a slightly-too-recent file timestamp in a cron directory, or one unusual line in `/etc/passwd` can each be the thread that unravels a full escalation path once followed up on.
4. **Permissions tell a story, not just a technical detail.** The difference between `-rw-------` and `-rw-r--r--`, or between a directory with and without the "other write" bit set, is frequently the entire reason a particular escalation path exists at all.
5. **Caution matters even when you've found something exploitable.** Kernel exploits in particular can crash or destabilize a system — a real risk during a live engagement — so understanding exactly what a technique does before running it is as important as finding it in the first place.
6. **This is explicitly a starting point, not a complete methodology.** Manual enumeration across these categories — users, processes, files, permissions, cron jobs — builds the foundational picture that specific privilege escalation techniques (covered in later material) are then applied against.
