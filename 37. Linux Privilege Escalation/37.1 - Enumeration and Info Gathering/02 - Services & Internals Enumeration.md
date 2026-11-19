# Linux Services & Internals Enumeration

## Table of Contents

1. [Why Go Deeper Than the Environment](#1-why-go-deeper-than-the-environment)
2. [Network Interfaces and the Hosts File](#2-network-interfaces-and-the-hosts-file)
   - 2.1 [Network Interfaces](#21-network-interfaces)
   - 2.2 [The /etc/hosts File](#22-the-etchosts-file)
3. [Who's Using This Box?](#3-whos-using-this-box)
   - 3.1 [Last Login Times](#31-last-login-times)
   - 3.2 [Who's Logged In Right Now](#32-whos-logged-in-right-now)
4. [Bash History and Hidden History Files](#4-bash-history-and-hidden-history-files)
5. [Cron Jobs — A Quick Revisit](#5-cron-jobs--a-quick-revisit)
6. [The /proc Filesystem](#6-the-proc-filesystem)
7. [Installed Packages and Binaries](#7-installed-packages-and-binaries)
   - 7.1 [Listing Installed Packages](#71-listing-installed-packages)
   - 7.2 [Checking the Sudo Version](#72-checking-the-sudo-version)
   - 7.3 [Binaries Without Packages](#73-binaries-without-packages)
8. [Cross-Checking Against GTFOBins](#8-cross-checking-against-gtfobins)
9. [Tracing System Calls with strace](#9-tracing-system-calls-with-strace)
10. [Configuration Files and Scripts](#10-configuration-files-and-scripts)
11. [Matching Running Processes to Users](#11-matching-running-processes-to-users)
12. [Decision Flow: Internals Enumeration](#12-decision-flow-internals-enumeration)
13. [Cheat Sheet](#13-cheat-sheet)
14. [Big-Picture Takeaways](#14-big-picture-takeaways)

---

## 1. Why Go Deeper Than the Environment

Previously, the focus was on getting the lay of the land — the operating system, the kernel, users, groups, and how all of that ties into file and folder permissions. This next phase goes one level deeper: the **internals** of the host, meaning how the operating system is actually configured and how its integrated background processes work.

Think of the earlier stage as mapping out the building — which rooms exist, who has keys to what. This stage is more like reviewing the building's security logs, its intercom system, and its maintenance schedule: who came and went, what automated systems are quietly running in the background, and what tools are sitting in the supply closet that could be repurposed.

Specifically, this phase is about answering questions like:

- What services and applications are installed, and which ones are actually running?
- What sockets (network communication endpoints) are in use?
- What users, admins, and groups exist?
- Who is currently logged in, and who has recently logged in?
- Are any password policies enforced?
- Is this host joined to an Active Directory domain?
- Is there anything useful in history, log, or backup files?
- Which files have been modified recently, and is there a pattern suggesting a cron job worth hijacking?
- What's the current IP addressing information, and is there anything interesting in `/etc/hosts`?
- Are there any interesting network connections to other internal or external systems?
- What useful tools are already installed (Netcat, Perl, Python, Ruby, Nmap, tcpdump, gcc, etc.)?
- Can bash history for any user reveal something interesting, like a password typed on the command line?
- Are there any cron jobs that could be hijacked?

Everything gathered here feeds directly into the specific escalation techniques covered later — this is still reconnaissance, just a layer deeper.

---

## 2. Network Interfaces and the Hosts File

### 2.1 Network Interfaces

```bash
$ ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo

2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b9:ed:2a brd ff:ff:ff:ff:ff:ff
    inet 10.129.203.168/16 brd 10.129.255.255 scope global dynamic ens192
```

- `ip a` ("ip address") lists every network interface the system has, along with its assigned IP address(es).
- `lo` is the **loopback interface** — a virtual interface a machine uses to talk to itself (always `127.0.0.1`), not useful for reaching anything else.
- `ens192` is a real network interface with an actual routable address (`10.129.203.168`). If a host turns out to have *multiple* real interfaces, each one reaching a different subnet, that's a strong signal the box could be used to **pivot** — using it as a stepping stone to reach a network segment that wasn't directly reachable from your attacking machine before.
- Note: on some systems `ip a` won't work if the `net-tools` package isn't installed — `ifconfig` is the older, equivalent command to fall back on.

### 2.2 The /etc/hosts File

```bash
$ cat /etc/hosts

127.0.0.1 localhost
127.0.1.1 nixlpe02
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

- `/etc/hosts` is a simple local lookup table mapping hostnames to IP addresses, checked *before* the system ever asks a DNS server. Think of it as a personal contacts list a phone checks before dialing directory assistance.
- It's worth a glance every time — custom entries here (mapping a nickname to an internal server, for example) can reveal other hosts on the network that aren't otherwise obvious.

---

## 3. Who's Using This Box?

### 3.1 Last Login Times

```bash
$ lastlog

Username         Port     From             Latest
root                                       **Never logged in**
daemon                                     **Never logged in**
...SNIP...
mickey           pts/1    10.10.14.15      Tue Aug  2 19:33:16
lxd                                        **Never logged in**
administrator.evilcorp                     **Never logged in**
backupsvc                                  **Never logged in**
goffy.d          pts/0    127.0.0.1        Tue Aug  2 19:32:29
logger                                     **Never logged in**
shared                                     **Never logged in**
john             pts/0    10.10.14.15      Wed Aug  3 13:37:22
```

- `lastlog` shows the most recent login time (and source IP) for every account on the system, including accounts that have never logged in at all.
- Checking this is useful for building a sense of *how actively used* this machine is. A busy, frequently-logged-into server is more likely to have messy leftover files, forgotten cron jobs, or loose command histories than a box nobody touches — more activity generally means more opportunities for a misconfiguration to slip through.

### 3.2 Who's Logged In Right Now

```bash
$ w

 12:27:21 up 1 day, 16:55,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
pluto    pts/0    10.10.14.16      Tue19   40:54m  0.02s  0.02s -bash
```

- `w` shows who is currently logged in, from where, how long they've been idle, and what command they're running. (`who` is a simpler alternative, and on some systems `finger` provides similar detail.)
- Here, `pluto` is logged into the system at the same time — worth being aware of, since that user might notice unusual activity, or their active session might itself be worth investigating further.

---

## 4. Bash History and Hidden History Files

Checking a user's shell history matters because people frequently type things they shouldn't directly on the command line — passwords passed as arguments, git operations, cron job setup, and so on. Reviewing this gives real insight into what the server is used for and can point directly at an escalation path.

```bash
$ history

    1  id
    2  cd /home/cliff.moore
    3  exit
    4  touch backup.sh
    5  tail /var/log/apache2/error.log
    6  ssh ec2-user@dmz02.evilcorp.local
    7  history
```

- Even a short, ordinary-looking history can be a lead — here, an SSH connection to `dmz02.evilcorp.local` as a different user (`ec2-user`) hints at a relationship with another internal host, worth following up on.

Beyond the standard `.bash_history`, some monitoring scripts or programs create their *own* special history files to track user activity:

```bash
$ find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null

-rw------- 1 john john 387 Nov 27 14:02 /home/john/.bash_history
```

- This `find` command searches for any file ending in `_hist` or `_history`, which catches not just the standard bash history but any custom logging files following that naming pattern.

---

## 5. Cron Jobs — A Quick Revisit

Cron jobs are Linux's equivalent of Windows scheduled tasks — automated commands or scripts that run on a schedule, often for maintenance or backups. Combined with another weakness, like a relative file path or overly loose permissions, a cron job can become a privilege escalation path, especially when it runs as root.

```bash
$ ls -la /etc/cron.daily/

total 48
drwxr-xr-x  2 root root 4096 Aug  2 17:36 .
drwxr-xr-x 96 root root 4096 Aug  2 19:34 ..
-rwxr-xr-x  1 root root  376 Dec  4  2019 apport
-rwxr-xr-x  1 root root 1478 Apr  9  2020 apt-compat
-rwxr-xr-x  1 root root  355 Dec 29  2017 bsdmainutils
-rwxr-xr-x  1 root root 1187 Sep  5  2019 dpkg
-rwxr-xr-x  1 root root  377 Jan 21  2019 logrotate
-rwxr-xr-x  1 root root 1123 Feb 25  2020 man-db
-rw-r--r--  1 root root  102 Feb 13  2020 .placeholder
-rwxr-xr-x  1 root root 4574 Jul 18  2019 popularity-contest
-rwxr-xr-x  1 root root  214 Apr  2  2020 update-notifier-common
```

- Same idea covered earlier: all these scripts are owned by root and run automatically, so anything unusual (an unexpected script name, a much more recent modification date than its neighbors) is worth a closer look.

---

## 6. The /proc Filesystem

**`/proc`** (short for "process filesystem") is a special, *virtual* filesystem on Linux — meaning it isn't real data sitting on a disk, but a live view generated on the fly by the kernel. It's the main way to look at (and sometimes change) information about running processes, hardware, and kernel settings — things like process state, memory usage, scheduling, and device information.

Think of `/proc` like a hospital's real-time patient monitoring display: the numbers on the screen aren't "stored" anywhere permanent, they're a live readout generated continuously from what's actually happening right now.

```bash
$ find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"

...SNIP...
root@10.129.14.200
ssh
root@10.129.14.200
sshd:
john
[priv]
/usr/bin/ssh-agent
-D
-a
/run/user/1000/keyring/.ssh
sshd:
john@pts/2
```

- Every running process has its own folder under `/proc` (named by its process ID), and each one contains a `cmdline` file showing exactly how that process was launched, including every argument passed to it.
- `tr " " "\n"` just converts spaces into line breaks, making the normally-jammed-together output easier to read.
- This example reveals an active SSH connection out to `root@10.129.14.200` — process command lines can sometimes leak credentials or connection details passed directly as arguments, so this is worth checking any time a process might still be running.

---

## 7. Installed Packages and Binaries

### 7.1 Listing Installed Packages

Older systems are more likely to have packages with at least one known vulnerability sitting installed — but even a fairly current distribution can still have older software bundled in that carries real vulnerabilities. The first step toward checking for this is simply building a list of what's installed.

```bash
$ apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list

Listing...
accountsservice-ubuntu-schemas 0.0.7+17.10.20170922-0ubuntu1
accountsservice 0.6.55-0ubuntu12~20.04.5
acl 2.2.53-6
acpi-support 0.143
acpid 2.0.32-1ubuntu1
adduser 3.118ubuntu2
...SNIP...
apparmor 2.13.3-7ubuntu5.1
apport 2.20.11-0ubuntu27.24
...SNIP...
```

- `apt list --installed` prints every package installed via `apt` (Ubuntu/Debian's package manager), each shown with its version.
- The rest of the pipeline cleans the output down to just `package_name version` pairs and saves the result into a file (`installed_pkgs.list`) for later reference — this list gets reused in Section 8 below.

### 7.2 Checking the Sudo Version

```bash
$ sudo -V

Sudo version 1.8.31
Sudoers policy plugin version 1.8.31
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.31
```

- `sudo -V` ("version") prints the exact sudo version in use. Sudo itself has had several serious, publicly documented vulnerabilities over the years (certain versions allow bypassing restrictions entirely), so this version number is always worth checking against known CVEs.

### 7.3 Binaries Without Packages

Occasionally a program isn't installed as a tracked package at all — it's just a standalone compiled binary dropped onto the system, runnable directly without any installation step.

```bash
$ ls -l /bin /usr/bin/ /usr/sbin/

lrwxrwxrwx 1 root root     7 Oct 27 11:14 /bin -> usr/bin

/usr/bin/:
total 175160
-rwxr-xr-x 1 root root       31248 May 19  2020  aa-enabled
-rwxr-xr-x 1 root root       35344 May 19  2020  aa-exec
...SNIP...

/usr/sbin/:
total 32500
-rwxr-xr-x 1 root root      3068 Mai 19  2020 aa-remove-unknown
...SNIP...
```

- Note `/bin -> usr/bin` is a **symlink** (a shortcut pointing to another location) — on modern Ubuntu, `/bin` and `/usr/bin` are effectively the same folder.
- Listing these folders directly catches anything that might not show up in a package manager's list, which is exactly why the next step cross-references this broader universe of binaries.

---

## 8. Cross-Checking Against GTFOBins

**GTFOBins** is a curated, publicly maintained list of common Unix binaries that can be abused to break out of restricted shells or escalate privileges — because many of them have legitimate features (like "read a file" or "run a shell command") that can be repurposed well beyond their intended use.

```bash
$ for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i";fi; done

Check GTFO for: ab
Check GTFO for: apt
Check GTFO for: ar
Check GTFO for: as
Check GTFO for: bash
Check GTFO for: busybox
Check GTFO for: curl
Check GTFO for: dd
...SNIP...
```

- This one-liner pulls the full current list of GTFOBins-tracked binaries straight from their API (`curl ... | jq ...` fetches and parses the JSON list of executable names), then checks each one against the `installed_pkgs.list` file built in Section 7.1.
- Every match printed is a binary present on this system that has at least one documented GTFOBins abuse technique — not a guarantee it's exploitable *here* (it still depends on what privileges you have and how the binary can actually be invoked), but a strong, specific list of leads worth checking one by one.

---

## 9. Tracing System Calls with strace

**`strace`** is a diagnostic tool that tracks and displays every system call (a request a program makes to the kernel — opening a file, sending network data, etc.) and signal a program makes while it runs. It lets you follow exactly how a program touches system resources, handles signals, and sends or receives data — including, sometimes, catching a program sending a password or token to a remote host in plain sight.

```bash
$ strace ping -c1 10.129.112.20

execve("/usr/bin/ping", ["ping", "-c1", "10.129.112.20"], 0x7ffdc8b96cc0 /* 80 vars */) = 0
access("/etc/suid-debug", F_OK)         = -1 ENOENT (No such file or directory)
...SNIP...
socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP) = 3
...SNIP...
connect(5, {sa_family=AF_INET, sin_port=htons(1025), sin_addr=inet_addr("10.129.112.20")}, 16) = 0
...SNIP...
sendto(3, "\10\0\31\303\0\0\0\1eX\327c..."..., 64, 0, {sa_family=AF_INET, ... sin_addr=inet_addr("10.129.112.20")}, 16) = 64
...SNIP...
write(1, "64 bytes from 10.129.112.20: icmp_se"..., 57) = 57
write(1, "1 packets transmitted, 1 receive"..., 60) = 60
exit_group(0)                           = ?
+++ exited with 0 +++
```

- `execve(...)` shows the exact command and arguments used to launch the program.
- `socket(...)` and `connect(...)` show the program opening a network connection — here, a standard ICMP (ping) socket connecting out to the target IP.
- `sendto(...)` / `write(...)` lines show data actually being sent or written — in a program handling authentication, this is exactly where a password or token could leak into view.
- The output can be large; piping it to a file (`strace -o output.txt <command>`) makes it much easier to search through afterward.

---

## 10. Configuration Files and Scripts

As long as an administrator hasn't tightened default permissions, a regular user can read almost every configuration file on a Linux system. These files reveal exactly how a service is set up, and sometimes contain sensitive information outright — keys, credentials, or paths to files in folders you otherwise can't browse. Importantly: **if a specific file has read permission for everyone, you can still read that individual file even if you can't list or enter the folder it sits in.**

```bash
$ find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null

-rw-r--r-- 1 root root 448 Nov 28 12:31 /run/tmpfiles.d/static-nodes.conf
-rw-r--r-- 1 root root 71 Nov 28 12:31 /run/NetworkManager/resolv.conf
-rw-r--r-- 1 systemd-resolve systemd-resolve 736 Nov 28 12:31 /run/systemd/resolve/stub-resolv.conf
...SNIP...
```

Scripts are worth the same attention as config files, for a similar reason — even without loose permissions letting you *modify* one, just reading a script can reveal internal processes and logic that's genuinely useful to know about.

```bash
$ find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"

/home/john/automation.sh
/etc/wpa_supplicant/action_wpa.sh
/etc/wpa_supplicant/ifupdown.sh
/etc/init.d/keyboard-setup.sh
/etc/init.d/console-setup.sh
...SNIP...
```

- `grep -v "src\|snap\|share"` filters *out* paths containing those three words, cutting down on noisy results from source trees, Snap packages, and shared library folders so the output stays focused on scripts more likely to be custom or administratively relevant.
- `/home/john/automation.sh` immediately stands out here — a script sitting in a user's own home directory, rather than a standard system location, is far more likely to be something an administrator wrote themselves, and therefore worth reading closely.

---

## 11. Matching Running Processes to Users

Looking at the live process list can reveal exactly which scripts or binaries are actively in use, and by which user. If a script was written by an administrator, placed somewhere in their own `$PATH`, and never had its permissions locked down properly, it may be possible to run it directly — without even needing root access to the directory it lives in.

```bash
$ ps aux | grep root

root           1  2.0  0.2 168196 11364 ?        Ss   12:31   0:01 /sbin/init splash
root         378  0.5  0.4  62648 17212 ?        S<s  12:31   0:00 /lib/systemd/systemd-journald
root         755  0.0  0.1 248088  7448 ?        Ssl  12:31   0:00 /usr/bin/vmtoolsd
root         778  0.0  0.0  18052  2992 ?        Ss   12:31   0:00 /usr/sbin/cron -f
root         779  0.0  0.2  37204  8964 ?        Ss   12:31   0:00 /usr/sbin/cupsd -l
root         784  0.4  0.5 273512 21680 ?        Ssl  12:31   0:00 /usr/sbin/NetworkManager --no-daemon
root         792  0.1  0.5  48244 20540 ?        Ss   12:31   0:00 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
root         889  0.1  0.5 126676 22888 ?        Ssl  12:31   0:00 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
root        1257  0.0  0.4 293736 16316 ?        Ssl  12:31   0:00 /usr/lib/packagekit/packagekitd
```

- `ps aux` lists every process on the system; piping into `grep root` narrows it down to anything running as root — worth scanning for anything custom, unusual, or tied to a script in a writable location.
- This single snapshot ties together almost everything covered so far: a running `cron` daemon (Section 5), Python scripts invoked directly with full paths (useful to cross-check against Section 10's scripts list), and background services that may be worth researching for known vulnerabilities (Section 7).

This gives a solid overall picture of the target system, setting up the next stage: digging into the specific permissions on each of these individual pieces.

---

## 12. Decision Flow: Internals Enumeration

```
        ┌─────────────────────────────────────┐
        │  Environment mapped — now dig into  │
        │           the internals             │
        └─────────────────────┬───────────────┘
                              │
      ┌───────────────────────┼────────────────────────┐
      │                       │                        │
      ▼                       ▼                        ▼
┌────────────────┐   ┌─────────────────────┐   ┌───────────────────────┐
│  Who's here?   │   │  What's installed   │   │  What's running right │
│                │   │  / present?         │   │  now?                 │
│ lastlog, w     │   │ apt list --installed│   │ ps aux | grep root    │
│ history files  │   │ ls /bin /usr/bin    │   │ find /proc -name      │
│                │   │ sudo -V             │   │   cmdline             │
└───────┬────────┘   └──────────┬──────────┘   └───────────┬───────────┘
        │                       │                          │
        ▼                       ▼                          ▼
 Found creds in          Cross-check against          Spot a process tied
 history or a live        GTFOBins + known CVEs        to a writable script
 proc cmdline?            for sudo/packages?            or custom binary?
        │                       │                           │
        └───────────────────────┴─────────────┬─────────────┘
                                              │
                                              ▼
                             ┌────────────────────────────────────┐
                             │  Read configs (*.conf) and scripts │
                             │  (*.sh) for logic, secrets, and    │
                             │  hints about how services are wired│
                             │  together — even without write     │
                             │  access to modify them yet         │
                             └────────────────────────────────────┘
```

---

## 13. Cheat Sheet

| Goal | Command |
|---|---|
| Network interfaces / IPs | `ip a` (or `ifconfig` if `net-tools` is installed) |
| Local hostname lookups | `cat /etc/hosts` |
| Last login time per user | `lastlog` |
| Who's logged in right now | `w` (or `who`, `finger`) |
| Current session's command history | `history` |
| Find hidden/custom history files | `find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null` |
| List daily cron jobs | `ls -la /etc/cron.daily/` |
| Dump a running process's launch command | `find /proc -name cmdline -exec cat {} \; 2>/dev/null \| tr " " "\n"` |
| List installed packages (apt-based) | `apt list --installed \| tr "/" " " \| cut -d" " -f1,3 \| sed 's/[0-9]://g'` |
| Check sudo version | `sudo -V` |
| List binaries directly | `ls -l /bin /usr/bin/ /usr/sbin/` |
| Cross-check installed binaries against GTFOBins | `for i in $(curl -s https://gtfobins.org/api.json \| jq -r '.executables \| keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i"; fi; done` |
| Trace a program's system calls | `strace <command>` |
| Find world-readable config files | `find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null` |
| Find shell scripts (noise filtered out) | `find / -type f -name "*.sh" 2>/dev/null \| grep -v "src\|snap\|share"` |
| Running processes owned by root | `ps aux \| grep root` |

---

## 14. Big-Picture Takeaways

1. **Internals enumeration builds on, rather than replaces, environment enumeration.** Users, groups, and file permissions from the earlier stage directly inform what's worth investigating here — who's logged in, what they've run, and what's running as root.
2. **Live state matters as much as static configuration.** Files like `/etc/passwd` tell you what *could* happen; `ps aux`, `/proc`, and `w` tell you what's *actually* happening right now — and the two together paint a much fuller picture than either alone.
3. **Command history and process arguments are a recurring leak point.** Whether it's a password typed into `history`, or one visible in a live process's `cmdline` under `/proc`, people and scripts alike tend to pass secrets as plain arguments more often than you'd expect.
4. **Tools like GTFOBins turn a long package list into a short, actionable one.** Knowing *that* a binary exists is one thing; cross-referencing it against a maintained database of known abuse techniques turns raw enumeration data into concrete next steps.
5. **Readable doesn't require full folder access.** A world-readable config file can be read directly by path even if you can't browse the directory it lives in — worth remembering any time a `find` search turns up a file in an otherwise-inaccessible-looking location.
6. **Different IP addresses and hostnames across examples reflect different lab sessions, not one continuous host.** As with the earlier environment enumeration notes, treat each code block here as an independent illustration of a command's typical output.
7. **This is still reconnaissance, not exploitation.** The goal at this stage is building a complete, accurate picture of services, processes, users, and configuration — the specific privilege escalation *techniques* that act on these findings come next.
