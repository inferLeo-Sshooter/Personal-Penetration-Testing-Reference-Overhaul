# Situational Awareness for Windows Privilege Escalation

## Table of Contents
1. [What "Situational Awareness" Means](#1-what-situational-awareness-means)
2. [Why Network Information Matters](#2-why-network-information-matters)
3. [Reading Interface, IP, and DNS Info (`ipconfig /all`)](#3-reading-interface-ip-and-dns-info-ipconfig-all)
4. [Reading the ARP Table (`arp -a`)](#4-reading-the-arp-table-arp--a)
5. [Reading the Routing Table (`route print`)](#5-reading-the-routing-table-route-print)
6. [Enumerating Security Protections](#6-enumerating-security-protections)
7. [Checking Windows Defender (`Get-MpComputerStatus`)](#7-checking-windows-defender-get-mpcomputerstatus)
8. [Checking AppLocker Rules](#8-checking-applocker-rules)
9. [Cheat Sheet](#9-cheat-sheet)
10. [Big-Picture Takeaways](#10-big-picture-takeaways)

---

## 1. What "Situational Awareness" Means

Before you try to do anything clever on a computer system — like escalating from a low-privilege user to an administrator — you first need to understand **where you are and what's around you**. This is called situational awareness.

**Analogy:** Think of landing on a new system like parachuting into unfamiliar terrain at night. Before you start marching toward your objective, you'd want to check your surroundings: What's nearby? Are there obstacles? Is anyone watching? Skipping this step means you're just guessing, and guessing gets you caught or stuck.

In practical terms, once you have a foothold on a Windows or Linux machine during a penetration test, you want to check things like:
- What other machines can this host reach? (possible pivot points)
- What security tools are running that might block or detect you?
- Which tools/techniques will actually work here?

Doing this enumeration first lets you act **proactively** (planning ahead) instead of **reactively** (stumbling into a wall and only then reacting).

---

## 2. Why Network Information Matters

Looking at a machine's network setup can reveal opportunities you wouldn't otherwise see.

- **Dual-homed hosts:** A machine is "dual-homed" when it's connected to two or more separate networks at once (often because it has multiple network cards/interfaces). **Analogy:** think of a building with a front door onto a public street and a back door into a private courtyard — a dual-homed machine is a bridge between two networks, and compromising it might let you walk from one network into another you couldn't reach directly before.
- **Routing tables** show you how traffic flows between networks — basically a "map" of where this computer can send data.
- **Domain information** (if the machine is part of an Active Directory environment) can reveal the IP addresses of domain controllers — important targets or waypoints in a Windows network.
- **ARP cache** (`arp -a`) shows which other devices this machine has recently talked to on the local network. This is useful for **lateral movement** (jumping from one compromised machine to another) — it might show you which servers an administrator regularly logs into via RDP (Remote Desktop) or WinRM (Windows Remote Management), hinting at where credentials you steal could be reused.

All of this network info might directly help you escalate privileges on the current box, or it might point you toward a completely different machine that's easier to attack.

---

## 3. Reading Interface, IP, and DNS Info (`ipconfig /all`)

This Windows command lists every network interface (physical or virtual) on the machine, along with its configuration.

```
C:\evilcorp> ipconfig /all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : WINLPE-SRV01
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No
   DNS Suffix Search List. . . . . . : .evilcorp

Ethernet adapter Ethernet1:

   Connection-specific DNS Suffix  . :
   Description . . . . . . . . . . . : vmxnet3 Ethernet Adapter
   Physical Address. . . . . . . . . : 00-50-56-B9-C5-4B
   DHCP Enabled. . . . . . . . . . . : No
   Autoconfiguration Enabled . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 192.168.20.56(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.20.1
   DNS Servers . . . . . . . . . . . : 8.8.8.8
   NetBIOS over Tcpip. . . . . . . . : Enabled

Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .evilcorp
   Description . . . . . . . . . . . : Intel(R) 82574L Gigabit Network Connection
   Physical Address. . . . . . . . . : 00-50-56-B9-90-94
   DHCP Enabled. . . . . . . . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 10.129.43.8(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:4ddf%4
                                       10.129.0.1
   DHCP Server . . . . . . . . . . . : 10.129.0.1
   DNS Servers . . . . . . . . . . . : 1.1.1.1
                                       8.8.8.8
   NetBIOS over Tcpip. . . . . . . . : Enabled

[... Tunnel adapters (ISATAP, Teredo) omitted — all show "Media disconnected",
     meaning these are inactive legacy IPv6 transition adapters, not in use here ...]
```

**What to notice here, line by line:**

| Field | What it tells you |
|---|---|
| `Host Name` | The machine's name on the network (`WINLPE-SRV01`) |
| `DNS Suffix Search List` | Confirms this box belongs to the `.evilcorp` domain |
| **Two Ethernet adapters** | This machine has **two separate network connections** — one on `192.168.20.0/24` and one on `10.129.0.0/16`. This is the dual-homed situation described above! |
| `DHCP Enabled` | Whether the IP address was assigned automatically (`Yes`) or set manually (`No`) |
| `Default Gateway` | The "exit door" this interface uses to reach other networks |
| `DNS Servers` | Which servers this machine asks to turn names into IP addresses |

Since this host has a leg in two different subnets (`192.168.20.x` and `10.129.x.x`), compromising it could be a stepping stone into the `192.168.20.0/24` network, which might not be reachable any other way.

---

## 4. Reading the ARP Table (`arp -a`)

**What ARP is:** ARP (Address Resolution Protocol) is how computers on the same local network match an IP address to a physical hardware address (MAC address). **Analogy:** think of IP addresses like apartment numbers and MAC addresses like the actual resident's name — ARP is the building directory that maps one to the other.

The ARP cache (`arp -a`) is essentially a short-term memory of which other devices on the local network this machine has recently exchanged traffic with.

```
C:\evilcorp> arp -a

Interface: 10.129.43.8 --- 0x4
  Internet Address      Physical Address      Type
  10.129.0.1            00-50-56-b9-4d-df     dynamic
  10.129.43.12          00-50-56-b9-da-ad     dynamic
  10.129.43.13          00-50-56-b9-5b-9f     dynamic
  10.129.255.255        ff-ff-ff-ff-ff-ff     static
  [... broadcast/multicast entries omitted — these are standard network
       housekeeping addresses, not real hosts ...]

Interface: 192.168.20.56 --- 0x9
  Internet Address      Physical Address      Type
  192.168.20.255        ff-ff-ff-ff-ff-ff     static
  [... multicast/broadcast entries omitted, same reason as above ...]
```

**Key takeaway:** On the `10.129.43.8` interface, this machine has recently talked to two other real hosts: `10.129.43.12` and `10.129.43.13`. Those are candidates worth investigating for lateral movement later — the current machine has a known relationship with them.

(Note: entries like `ff-ff-ff-ff-ff-ff` are broadcast addresses — messages meant for every device on the network, not a single specific host — and addresses starting with `224.` or `239.` are multicast groups. Neither represents an actual "other computer" of interest.)

---

## 5. Reading the Routing Table (`route print`)

A routing table tells the machine where to send traffic depending on its destination — like a signpost system for data.

```
C:\evilcorp> route print

===========================================================================
Interface List
  9...00 50 56 b9 c5 4b ......vmxnet3 Ethernet Adapter
  4...00 50 56 b9 90 94 ......Intel(R) 82574L Gigabit Network Connection
  1...........................Software Loopback Interface 1
  [... ISATAP / Teredo tunnel adapters omitted — inactive, as seen earlier ...]
===========================================================================

IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0       10.129.0.1      10.129.43.8     25
          0.0.0.0          0.0.0.0     192.168.20.1    192.168.20.56    271
       10.129.0.0      255.255.0.0         On-link       10.129.43.8    281
     192.168.20.0    255.255.255.0         On-link     192.168.20.56    271
        127.0.0.0        255.0.0.0         On-link         127.0.0.1    331
  [... remaining broadcast/multicast/link-local entries omitted for brevity ...]
===========================================================================
Persistent Routes:
  Network Address          Netmask  Gateway Address  Metric
          0.0.0.0          0.0.0.0     192.168.20.1  Default
===========================================================================
```

**How to read a routing table row:**

| Column | Meaning |
|---|---|
| `Network Destination` + `Netmask` | The range of addresses this rule applies to (`0.0.0.0` with mask `0.0.0.0` = "everywhere," i.e. the default route) |
| `Gateway` | Where to send traffic bound for that destination. `On-link` means "deliver directly, no gateway needed, it's on the same local network" |
| `Interface` | Which local network card is used |
| `Metric` | A priority number — **lower = preferred**. If two routes could apply, the one with the lower metric wins |

**Notice:** There are *two* default routes (`0.0.0.0/0`), one via `10.129.0.1` (metric 25) and one via `192.168.20.1` (metric 271). Since lower metric wins, this machine will normally send general internet-bound traffic out through `10.129.43.8` — but it's still perfectly capable of reaching the `192.168.20.0/24` network directly, confirming again that this box bridges two networks.

---

## 6. Enumerating Security Protections

Before trying any privilege-escalation technique, it's smart to find out what security software is watching you. Modern systems often run some form of:

- **Antivirus (AV)** — scans files/processes for known bad patterns.
- **EDR (Endpoint Detection and Response)** — a more advanced, modern layer that monitors *behavior* in real time, not just file signatures, and can alert security teams or automatically block suspicious actions.
- **Application whitelisting** — a system that only allows pre-approved programs or scripts to run. Microsoft's built-in version of this is called **AppLocker**.

**Analogy:** If AV is like a security guard checking IDs against a "wanted" poster, EDR is like a guard who also watches *how* you behave in the building — how you walk, where you go, what you touch — and raises an alarm if it looks suspicious, even if your ID checks out. AppLocker, meanwhile, is like a guest list at the door: if your name (or program) isn't on the approved list, you simply don't get in, no matter how innocent you look.

Why this matters: common system tools often used during attacks — like `net.exe` or `tasklist` — might be flagged or blocked outright. An EDR might also notice something odd like "an accountant's computer is running command-line tools it never uses," and raise a red flag. Enumerating these protections *before* acting lets you:
- Avoid tripping alarms,
- Know which tools you'll need to modify or avoid,
- Test your approach safely in a lab before trying it against a real target.

You can check AppLocker's configuration with the PowerShell cmdlet [Get-AppLockerPolicy](https://learn.microsoft.com/en-us/powershell/module/applocker/get-applockerpolicy?view=windowsserver2019-ps), which can show local rules, the policy actually being enforced ("effective"), and any domain-wide rules.

---

## 7. Checking Windows Defender (`Get-MpComputerStatus`)

This PowerShell command reports the live status of Windows Defender (Microsoft's built-in antivirus).

```
PS C:\evilcorp> Get-MpComputerStatus

AMEngineVersion                 : 1.1.17900.7
AMServiceEnabled                 : True
AntispywareEnabled               : True
AntivirusEnabled                 : True
BehaviorMonitorEnabled           : False
IoavProtectionEnabled            : False
NISEnabled                       : False
OnAccessProtectionEnabled        : False
RealTimeProtectionEnabled        : False
```

**How to read this:** Just because `AntivirusEnabled : True` doesn't mean you're fully covered by live protection. Notice:

| Setting | What it controls | Status here |
|---|---|---|
| `AntivirusEnabled` | Is Defender turned on at all | ✅ True |
| `RealTimeProtectionEnabled` | Does it scan files *as they're accessed/run*, live | ❌ False |
| `BehaviorMonitorEnabled` | Does it watch program *behavior* for suspicious activity | ❌ False |
| `OnAccessProtectionEnabled` | Does it scan files the moment they're opened | ❌ False |
| `IoavProtectionEnabled` | Does it scan files downloaded from the internet/email | ❌ False |

**Takeaway:** This is a crucial distinction — Defender is *installed and technically enabled*, but its real-time/behavioral detection features are all switched off. That's like having a smoke detector installed in the house but with the battery taken out: the hardware exists, but it won't actually alert anyone. This kind of configuration drastically reduces the chance of detection during enumeration or exploitation on this particular machine.

---

## 8. Checking AppLocker Rules

AppLocker rules define what is and isn't allowed to run, based on things like file path, publisher (digital signature), or file hash. You can see the rules currently in force with:

```
PS C:\evilcorp> Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

PublisherConditions : {*\*\*,0.0.0.0-*}
PublisherExceptions : {}
PathExceptions      : {}
HashExceptions      : {}
Id                  : a9e18c21-ff8f-43cf-b9fc-db40eed693ba
Name                : (Default Rule) All signed packaged apps
Description         : Allows members of the Everyone group to run packaged apps that are signed.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%PROGRAMFILES%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 921cc481-6e17-4653-8f75-050b80acca20
Name                : (Default Rule) All files located in the Program Files folder
Description         : Allows members of the Everyone group to run applications that are located in the Program Files
                      folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%WINDIR%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : a61c8b2c-a319-4cd0-9690-d2177cad7b51
Name                : (Default Rule) All files located in the Windows folder
Description         : Allows members of the Everyone group to run applications that are located in the Windows folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

...<SNIP>...
```

Trimmed/summarized view of the rules returned:

| Rule Name | Applies To | Who It Allows | Action |
|---|---|---|---|
| All signed packaged apps | Signed packaged apps (publisher-based) | Everyone | Allow |
| All files in Program Files | `%PROGRAMFILES%\*` | Everyone | Allow |
| All files in Windows folder | `%WINDIR%\*` | Everyone | Allow |
| All files (catch-all) | `*` (anything) | Local Administrators only | Allow |
| All signed Windows Installer files | Signed `.msi` files | Everyone | Allow |
| Installer files in `%systemdrive%\Windows\Installer` | That specific folder | Everyone | Allow |
| All Windows Installer files (catch-all) | `*.*` | Local Administrators only | Allow |
| Scripts in Program Files | `%PROGRAMFILES%\*` | Everyone | Allow |
| Scripts in Windows folder | `%WINDIR%\*` | Everyone | Allow |
| All scripts (catch-all) | `*` (anything) | Local Administrators only | Allow |

**How to read this table:** Regular users (the "Everyone" group) are only allowed to run things from specific trusted locations: the `Program Files` folder, the `Windows` folder, or properly signed installer packages. Anything outside of those locations is only allowed for Administrators. This is the "guest list" from the analogy in Section 6 — it restricts *where* programs can run from, regardless of what the program actually does.

You can test whether a *specific file* would be allowed to run for a *specific user* using `Test-AppLockerPolicy`:

```
PS C:\evilcorp> Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone

FilePath                    PolicyDecision MatchingRule
--------                    -------------- ------------
C:\Windows\System32\cmd.exe         Denied c:\windows\system32\cmd.exe
```

**Important nuance:** Even though `cmd.exe` lives inside `%WINDIR%\*` (which the table above says is allowed for "Everyone"), this specific test shows it's actually **Denied** for the `Everyone` group. This tells you that there's an additional, more specific rule (not shown in the default list above) explicitly blocking `cmd.exe` for regular users — a reminder that the *effective* policy can include extra deny rules beyond the default allow rules, and the only way to be sure is to test directly with `Test-AppLockerPolicy` rather than just reading the rule list.

---

## 9. Cheat Sheet

| Goal | Command |
|---|---|
| List all network interfaces, IPs, DNS config | `ipconfig /all` |
| View recently-contacted local hosts (MAC/IP pairs) | `arp -a` |
| View how traffic is routed between networks | `route print` |
| Check if a host is part of an AD domain with multiple NICs | Look for multiple adapters + different gateways in `ipconfig /all` |
| Check Windows Defender's real status | `Get-MpComputerStatus` |
| List AppLocker rules currently enforced | `Get-AppLockerPolicy -Effective \| select -ExpandProperty RuleCollections` |
| Check local-only AppLocker rules | `Get-AppLockerPolicy -Local` |
| Test if a specific file is blocked for a specific user | `Get-AppLockerPolicy -Local \| Test-AppLockerPolicy -path <file> -User <user>` |

### Decision Flow: "Is This Machine Worth Pivoting Through?"

```
                 ┌───────────────────────────┐
                 │  Run ipconfig /all        │
                 └──────────────┬────────────┘
                                │
                more than one active network adapter?
                                │
                 ┌──────────────┴──────────────┐
                 │ Yes                         │ No
                 ▼                              ▼
     Machine is dual-homed →          Single network only →
     check arp -a + route print       focus on local privesc
     for reachable extra networks      only (no pivot via this host)
                 │
                 ▼
     Found hosts on second network?
                 │
        ┌────────┴─────────┐
        │ Yes              │ No
        ▼                  ▼
  Note as pivot targets   Keep enumerating —
  for later lateral       dual-homing alone
  movement                doesn't guarantee access
```

### Decision Flow: "Will My Tool/Technique Even Run?"

```
        ┌────────────────────────────┐
        │ Check Get-MpComputerStatus │
        └────────────────┬───────────┘
                         │
          RealTimeProtectionEnabled = True?
                 ┌───────┴───────┐
                 │ Yes           │ No
                 ▼               ▼
     Expect live scanning —   Lower immediate detection
     consider evasion/        risk from Defender's
     obfuscation              live engine
                 │               │
                 └───────┬───────┘
                         ▼
           Check AppLocker (Get-AppLockerPolicy)
                         │
          Is your binary's path/publisher allowed
          for your current user group?
                 ┌───────┴───────┐
                 │ Yes           │ No
                 ▼               ▼
      Proceed, but verify     Find an allowed location/
   with Test-AppLockerPolicy  method, or a bypass technique
```

---

## 10. Big-Picture Takeaways

1. Always orient yourself on a new system before acting — understand the network layout and security tooling first, so you can plan instead of stumbling.
2. A "dual-homed" machine (multiple network interfaces on different subnets) can be a bridge into networks you couldn't otherwise reach — check `ipconfig /all` for multiple active adapters.
3. The ARP cache (`arp -a`) shows which nearby machines this host has recently talked to — useful hints for where to move next after escalating privileges.
4. The routing table (`route print`) reveals how traffic actually flows between networks, including which route is preferred (lowest metric wins).
5. A security product being "enabled" doesn't mean every protective feature is active — `Get-MpComputerStatus` can show Defender installed but with real-time/behavioral protections switched off.
6. AppLocker (or similar whitelisting tools) restricts *where* programs/scripts can run from — typically trusted folders for regular users, everything for administrators.
7. Reading the rule list isn't always enough — specific deny rules can override what looks like a blanket allow, so always confirm with a direct test like `Test-AppLockerPolicy` before relying on a technique.
8. The whole point of this enumeration phase is risk reduction: understanding the environment lets you choose tools and methods that are less likely to be blocked or detected.
