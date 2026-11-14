# Shells & Payloads: Web Shells

---

## Table of Contents

1. [Why web shells matter more than ever](#1-why-web-shells-matter-more-than-ever)
2. [What a web shell actually is](#2-what-a-web-shell-actually-is)
3. [How you get a web shell onto a server](#3-how-you-get-a-web-shell-onto-a-server)
4. [Web shell languages and their matching server stacks](#4-web-shell-languages-and-their-matching-server-stacks)
5. [Laudanum: a ready-made web shell collection](#5-laudanum-a-ready-made-web-shell-collection)
6. [Antak: a PowerShell-based ASPX web shell](#6-antak-a-powershell-based-aspx-web-shell)
7. [PHP web shells](#7-php-web-shells)
8. [Web shells vs. reverse shells: a critical comparison](#8-web-shells-vs-reverse-shells-a-critical-comparison)
9. [Operational security: using web shells responsibly](#9-operational-security-using-web-shells-responsibly)
10. [Cheat sheet](#10-cheat-sheet)

---

## 1. Why web shells matter more than ever

In the past, external penetration tests often found easy wins through exposed network services — open SMB shares, vulnerable FTP servers, unpatched RDP. Today, perimeter networks are much more locked down. Those classic entry points now show up primarily on **internal** tests.

What *is* consistently exposed externally? **Web applications.** Almost everything is moving to the browser — entertainment, finance, communication, enterprise tools. This creates a massive and ever-growing attack surface.

**The most common ways to get a foothold during an external test today:**

| Method | Description |
|---|---|
| **Web application attacks** | File upload abuse, SQL injection, RFI/LFI, command injection |
| **Password spraying** | Trying common passwords against RDS, VPN portals, Citrix, OWA, cloud logins |
| **Social engineering** | Phishing, pretexting, malicious attachments |

Of these, web application attacks are the ones most likely to lead directly to a **web shell** — a persistent foothold right inside the web server.

---

## 2. What a web shell actually is

A **web shell** is a script hosted on a web server that lets you send OS commands through your browser and see the output returned as a web page.

**Analogy:** imagine you've picked a lock and stepped inside a house. A regular reverse shell is like pulling out a walkie-talkie to talk to your team. A web shell is more like finding the house has an internal intercom system — you use the building's own infrastructure to communicate, rather than your own device.

```
Normal shell (reverse/bind):
  Attacker machine  ←──TCP connection──→  Target OS

Web shell:
  Attacker browser
       │
       │  HTTP request: ?cmd=whoami
       ▼
  Web shell script on server
       │
       ▼
  Web server passes to OS → executes command
       │
       ▼
  Output returned as HTTP response → displayed in browser
```

**Two important implications of this model:**

1. **No direct network connection required** between attacker and target shell. The HTTP/S channel is the conduit — it's the same port the web server already has open (80/443).
2. **The web server process runs the commands**, so you inherit whatever user the web server runs as (commonly `www-data`, `apache`, `IIS APPPOOL\DefaultAppPool`, etc.).

---

## 3. How you get a web shell onto a server

A web shell only works if you can **upload the script file** to a location the web server will execute it from. Here are the most common upload vectors:

| Upload vector | How it works |
|---|---|
| **Unrestricted file upload forms** | Public-facing forms that allow uploading `.php`, `.asp`, `.aspx`, `.jsp` without validation |
| **User profile picture upload** | Self-registration pages that accept image uploads — bypass client-side checks and upload a script instead |
| **Application deployment interfaces** | Tomcat, Axis2, WebLogic — allow WAR file deployment as part of their core functionality; a WAR file can contain JSP web shells |
| **Misconfigured FTP** | FTP service that allows anonymous write access to the server's webroot |
| **CMS plugin/theme upload** | WordPress, Joomla, Drupal — plugin or theme installers that accept `.zip` packages can be abused |
| **SQL injection + file write** | In some configurations, SQL injection can write files directly to disk via `INTO OUTFILE` |

> **Core principle:** any path that lets you place a script file inside a directory the web server will execute is a potential web shell entry point. Finding one is the whole challenge — once you have it, the rest follows.

---

## 4. Web shell languages and their matching server stacks

A web shell has to be written in a language the **target web server can execute**. If the server runs PHP, your `.asp` file does nothing. Always match the shell language to the server stack.

| Server technology | Shell language | Common extension |
|---|---|---|
| Apache + PHP | PHP | `.php`, `.php5`, `.phtml` |
| Microsoft IIS | ASP / ASPX | `.asp`, `.aspx` |
| Apache Tomcat / JBoss | JSP | `.jsp`, `.jspx` |
| ColdFusion | CFML | `.cfm` |
| Node.js | JavaScript | Varies |
| Python-based servers | Python | Varies |

**How to identify the target's stack:** Nmap version banners, HTTP response headers (`X-Powered-By`, `Server`), file extensions on the site, error messages, and technology detection tools like `whatweb`.

---

## 5. Laudanum: a ready-made web shell collection

**Laudanum** is a pre-built collection of web shell scripts covering a wide range of server languages. Think of it as a toolbox where someone has already done the boilerplate coding — you just configure and deploy.

### Where it lives

```bash
/usr/share/laudanum
```

Pre-installed on both Parrot OS and Kali Linux.

### What's inside

Languages covered include: `asp`, `aspx`, `jsp`, `php`, and others.

### How to use it

```
1. Browse to /usr/share/laudanum
2. Find the file matching your target's language (e.g. php/php-reverse-shell.php)
3. For shell files specifically: open the file and insert your attacker IP address
4. Read the file's comments — some files need extra configuration
5. Upload the modified file to the target via your chosen upload vector
6. Navigate to the uploaded file's URL in a browser to trigger execution
```

> **Always read the file's contents and comments before deploying.** Laudanum files vary — some are simple command-execution web shells, others initiate a reverse callback. The comment block at the top of each file explains what it does and what needs to be configured.

**Key distinction:** for shell-type Laudanum files, you must **edit in your attacker IP** before uploading — unlike purely browser-based web shells that don't need a callback listener.

### Example

**Insert your IP in this line:**
```
void Page_Load(object sender, System.EventArgs e) {

        // Check for an IP in the range we want
        string[] allowedIps = new string[] {"::1","192.168.0.1", "127.0.0.1", "YOUR_IP"};
```

**Web shell trigger:**
<img width="1028" height="727" alt="Annotation 2026-11-14 124903" src="https://github.com/user-attachments/assets/02bbdf25-f8c2-4c10-b835-7e27ead06bd1" />

---

## 6. Antak: a PowerShell-based ASPX web shell

**Antak** is an ASPX-based web shell built into the **Nishang** project (Nishang is an Offensive PowerShell toolkit). It's specifically designed for Windows servers running IIS with the ASP.NET framework.

### Why Antak stands out

- Runs on the **ASP.NET framework** via ASPX — meaning it works on most modern Windows/IIS web servers.
- Uses **PowerShell** as its backend shell interpreter — not just basic cmd.exe commands.
- Has a **UI themed to look like PowerShell**, making navigation familiar.
- Since PowerShell gives access to the full .NET ecosystem, Antak is significantly more powerful than a simple command-runner shell.

### Quick ASPX primer

**ASPX (Active Server Pages Extended)** is a web page format for Microsoft's ASP.NET framework. When a browser requests an `.aspx` page, IIS compiles and executes the server-side code and returns HTML output. Antak exploits this: an Antak `.aspx` file on a Windows IIS server executes PowerShell commands server-side and returns the output as a rendered web page.

```
Browser submits PowerShell command via Antak's web form
        │
        ▼
IIS compiles and runs the ASPX → executes PowerShell backend
        │
        ▼
PowerShell runs the command on the Windows OS
        │
        ▼
Output returned as HTML → browser displays it
```

### Where it lives

```bash
/usr/share/nishang/Antak-WebShell
```

### Usage pattern

Same as Laudanum:
1. Copy the Antak `.aspx` file.
2. Edit in your attacker IP if you want a reverse callback.
3. Upload to the target IIS server via your upload vector.
4. Visit the file's URL — the PowerShell-themed interface appears.

### Example

**Make sure you set credentials for access to the web shell:**

<img width="1779" height="636" alt="image" src="https://github.com/user-attachments/assets/50f7c6a3-f207-41c3-affa-10aba2155d45" />

<img width="1716" height="162" alt="image" src="https://github.com/user-attachments/assets/d50cabcd-ea2e-4101-9c50-666087b198e0" />

**Access and trigger the web shell:**

<img width="1915" height="389" alt="image" src="https://github.com/user-attachments/assets/74a2dadc-b9e4-456a-87d4-9f4f4e530ed2" />

<img width="1920" height="898" alt="image" src="https://github.com/user-attachments/assets/36a6c5a0-0a57-4760-bf59-096e9d151f4b" />

<img width="1208" height="783" alt="image" src="https://github.com/user-attachments/assets/6e2412d0-7075-4a49-b6da-b3c5b4bb391e" />

---

## 7. PHP web shells

PHP is the most widely deployed server-side language on the internet (WordPress, Drupal, Joomla, and countless custom applications run on it), making PHP web shells the most commonly used type.

A minimal PHP web shell can be as small as one line:

```php
<?php system($_GET['cmd']); ?>
```

**How this works:**
- `$_GET['cmd']` reads the `cmd` parameter from the URL query string.
- `system()` runs that value as an OS command and prints the output.
- Visiting `http://target.com/shell.php?cmd=whoami` runs `whoami` on the server and displays the result.

### A commonly used, more functional PHP shell

**WhiteWinterWolf's PHP Web Shell** is a well-known single-file PHP web shell that provides a full browser-based interface for executing commands, browsing the filesystem, uploading/downloading files, and more. It can be downloaded or the source code can be copied directly into a `.php` file.

> **Important:** a plain PHP web shell like this, uploaded without encoding, will often be caught by web application firewalls (WAF) or server-side AV scanning. The more functional a web shell is, the larger and more detectable its signature tends to be.

**Web shell example:**
<img width="1217" height="450" alt="image" src="https://github.com/user-attachments/assets/3b982411-5fc2-441d-81d3-6a3b5b7cd20a" />

---

## 8. Web shells vs. reverse shells: a critical comparison

Web shells are valuable — but they come with significant limitations compared to a proper reverse shell. Understanding this helps you decide when to treat the web shell as a stepping stone vs. the final access method.

| Factor | Web Shell | Reverse Shell |
|---|---|---|
| **Persistence mechanism** | Exists as a file on disk — can be deleted by the server | Runs in memory (Meterpreter) or as a network connection |
| **Interactivity** | Limited — browser-based, no real-time interaction | Full interactive terminal |
| **Chaining commands** | May not work (`whoami && hostname` might fail) | Full shell features, all commands chain normally |
| **File navigation** | Clunky — no `cd` persistence between requests | Full filesystem navigation |
| **Stability** | Can be deleted by the web app after a period | Persistent as long as the connection holds |
| **Upgrade path** | Use web shell to deliver a reverse shell payload | Already at this stage |
| **Detection footprint** | File on disk — obvious artifact | In-memory payloads leave less disk evidence |

**The standard workflow:**

```
Web shell (initial foothold)
        │
        ▼
Use web shell to execute a reverse shell payload
        │
        ▼
Catch the reverse shell on your listener
        │
        ▼
Delete the web shell file  ← clean up the artifact
        │
        ▼
Full interactive reverse/Meterpreter session
```

> **Web shells are often just a stepping stone, not the destination.** Use them to get a more stable, interactive connection established — then remove the file.

---

## 9. Operational security: using web shells responsibly

Operational security (OpSec) is about minimizing the evidence you leave behind. Web shells make this harder because they're files on disk, visible to anyone who looks. Best practices:

### During the engagement

- **Document everything:** record every file name, upload path, timestamp, and delivery method attempted.
- **Hash your files:** calculate and record `sha1sum` or `md5sum` of every payload/web shell file you upload. This provides proof of what you placed on the system and when.
- **Upgrade and delete:** as soon as you use the web shell to get a reverse shell, **delete the web shell file** from the server.

### Why cleanup matters from both sides

| From the attacker's perspective | From the defender's perspective |
|---|---|
| A leftover file is evidence of your presence, potentially causing legal complications after the engagement period ends | Defenders watching for file creation events or scanning for known web shell signatures (Laudanum, Antak filenames, PHP shell patterns) can detect and kill an attack mid-engagement |

> **The same artifact that's a liability for an attacker is a detection opportunity for a defender.** Both sides should understand this dynamic.

### Evasion-aware considerations

For **black-box evasive assessments** (where you're specifically testing whether the client can detect a live threat):

- Modify web shell files rather than using them verbatim — file content signatures are well-known for popular shells like Laudanum and Antak.
- Rename files to something inconspicuous (e.g. `config_cache.php` rather than `shell.php`).
- Upload to less-monitored directories if possible.
- Minimize dwell time — use the web shell, establish your reverse shell, delete it.

---

## 10. Cheat sheet

### Web shell quick reference

| Shell | Language | File | Location | Use when |
|---|---|---|---|---|
| Laudanum (various) | PHP, ASPX, JSP, ASP | Multiple | `/usr/share/laudanum` | Broad coverage, pick the matching language |
| Antak | ASPX (PowerShell) | `antak.aspx` | `/usr/share/nishang/Antak-WebShell` | Windows IIS target, want PowerShell access |
| WhiteWinterWolf | PHP | `webshell.php` | Download from GitHub | PHP targets, feature-rich browser interface |
| Minimal PHP | PHP | Manual (one line) | Craft inline | Quick, tiny — less detectable |

### Minimal PHP web shell

```php
<?php system($_GET['cmd']); ?>
```

Usage: `http://target.com/shell.php?cmd=whoami`

### Language-to-stack matching

| Target technology | Shell language |
|---|---|
| Apache/Nginx + PHP | `.php` |
| Microsoft IIS | `.asp` or `.aspx` |
| Apache Tomcat | `.jsp` |

### Upgrade path: web shell → reverse shell

```
1. Upload web shell to target
2. Use browser to run a reverse shell one-liner via the web shell
3. Catch the callback on your nc/msfconsole listener
4. Use the reverse shell to delete the web shell file
5. Continue assessment from the stable reverse shell
```

### OpSec checklist

```
□ Record every filename, path, and hash of uploaded files
□ Document what worked and what didn't
□ Upgrade to reverse shell as soon as possible
□ Delete the web shell file after establishing the reverse shell
□ For evasive engagements: rename and modify files to avoid known signatures
```

### Big-picture takeaways

1. **Web applications are now the primary external attack surface** — SMB/RDP are mostly internal; the web is what's exposed externally.
2. **A web shell is a script on the server that accepts commands through HTTP** — no direct network connection needed beyond port 80/443.
3. **Web shell language must match the server's technology** — PHP for PHP stacks, ASPX for IIS/ASP.NET, JSP for Tomcat.
4. **Laudanum and Antak are pre-built, ready-to-deploy collections** — available by default on Parrot/Kali. Edit attacker IP before deploying shell-type files.
5. **Antak is specifically optimized for Windows IIS** — it uses PowerShell as the backend, making it far more capable than a simple command runner on Windows targets.
6. **Web shells are stepping stones, not destinations** — they're unstable, leave disk artifacts, and have limited interactivity. Use them to deliver a reverse shell, then delete them.
7. **Cleanup is both a legal/professional obligation and an OpSec requirement** — leftover files are evidence after an engagement, and detectable artifacts for defenders.
8. **Document everything:** filenames, hashes, upload paths, timestamps. This protects you and strengthens the final report.

---
