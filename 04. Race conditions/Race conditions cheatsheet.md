# Race Conditions Cheat Sheet

> A **race condition** occurs when a site handles multiple requests simultaneously without proper safeguards, causing threads to collide on the same data and trigger unintended behavior. The brief collision window is called the **race window**.

---

## Table of Contents

- [1. Core Concepts](#1-core-concepts)
  - [1.1 Sub-State & Race Window](#11-sub-state--race-window)
  - [1.2 TOCTOU](#12-toctou)
- [2. Tooling](#2-tooling)
  - [2.1 Single-Packet Attack (HTTP/2)](#21-single-packet-attack-http2)
  - [2.2 Last-Byte Sync (HTTP/1 fallback)](#22-last-byte-sync-http1-fallback)
  - [2.3 Burp Repeater vs Turbo Intruder](#23-burp-repeater-vs-turbo-intruder)
- [3. Attack Types](#3-attack-types)
  - [3.1 Limit Overrun](#31-limit-overrun)
  - [3.2 Single-Endpoint Race](#32-single-endpoint-race)
  - [3.3 Multi-Endpoint Race](#33-multi-endpoint-race)
  - [3.4 Hidden Multi-Step Sequences](#34-hidden-multi-step-sequences)
  - [3.5 Partial Construction](#35-partial-construction)
- [4. Advanced Cases](#4-advanced-cases)
  - [4.1 Session-Based Locking](#41-session-based-locking)
  - [4.2 Time-Sensitive Attacks](#42-time-sensitive-attacks)
- [5. Methodology & Defense](#5-methodology--defense)
  - [5.1 Predict, Probe, Prove](#51-predict-probe-prove)
  - [5.2 Limit Overrun: Simpler Flow](#52-limit-overrun-simpler-flow)
  - [5.3 Alignment Troubleshooting](#53-alignment-troubleshooting)
  - [5.4 Prevention](#54-prevention)

---

## 1. Core Concepts

### 1.1 Sub-State & Race Window

A **sub-state** is a hidden, temporary state an app passes through *between* the start and end of processing a request. From the outside it looks like one action, but internally it's multiple steps.

```
User sees:   | POST /apply-coupon ---------> 200 OK |

Server does: | CHECK | APPLY | UPDATE |
                       ^-----^
                      sub-state
                    (race window lives here)
```

> **Sub-state** = *what* is temporarily vulnerable
> **Race window** = *when* it's vulnerable (typically milliseconds)

**Examples of sub-states:**
- MFA login: session valid, but MFA not yet enforced
- User registration: user exists in DB, but API key not initialized yet

---

### 1.2 TOCTOU

**Time-of-Check to Time-of-Use** — the app performs a security check, then acts on the result, but the state changes *in between*.

```
Request 1:  [CHECK ✓] ────────────────────> [USE] ──> [UPDATE ✗ too late]
Request 2:       [CHECK ✓] ──> [USE] ──> [UPDATE]
                      ^
                 race window
              (R1 hasn't updated yet)
```

**Example — one-time coupon:**
1. **Check:** Is the coupon unused? Yes.
2. **Use:** Apply discount.
3. **Update:** Mark coupon as used.

If two requests run concurrently, Request 2 passes the check before Request 1 finishes the update → coupon applied twice.

---

## 2. Tooling

**Quick decision tree:**
```
Target supports HTTP/2?
    YES --> Single-Packet Attack (Burp Repeater, auto)
    NO  --> Last-Byte Sync (Burp Repeater, auto fallback)

Attack needs 100+ requests or complex logic?
    YES --> Turbo Intruder
    NO  --> Burp Repeater is enough
```

### 2.1 Single-Packet Attack (HTTP/2)

Sends **20–30 requests inside one TCP packet** so they arrive and are processed virtually simultaneously. Eliminates **network jitter**.

```
Normal:         REQ1 --> [delay] --> REQ2 --> [delay] --> REQ3
                                                 race window missed

Single-packet:  [ REQ1 | REQ2 | REQ3 | ... | REQ30 ]  ← one TCP packet
                     all processed ~simultaneously
```

> **Preferred technique.** Use whenever the target supports HTTP/2.

---

### 2.2 Last-Byte Sync (HTTP/1 fallback)

HTTP/1 can't bundle requests, so Burp holds all requests with **everything except the last byte**, then fires all final bytes at once.

```
REQ1:  [headers....body] [last byte] ─┐
REQ2:  [headers....body] [last byte] ─┼──> fired together
REQ3:  [headers....body] [last byte] ─┘
```

Less reliable than single-packet, but works when HTTP/2 isn't available.

---

### 2.3 Burp Repeater vs Turbo Intruder

| | Burp Repeater | Turbo Intruder |
|---|---|---|
| **Best for** | Quick tests, confirming a race | Complex, large-scale attacks |
| **Setup** | No config needed | Requires Python script |
| **Request count** | Small groups | Hundreds+ |
| **Technique** | Auto (HTTP/2 → HTTP/1 fallback) | Manual `gates` |
| **Connection warming** | Manual | Built-in |
| **Use when** | Exploring and validating | Exploiting reliably at scale |

**Burp Repeater:** Add requests to a tab group → **"Send group in parallel"**

**Turbo Intruder:** Install via BApp Store → use `gates` to queue and release requests simultaneously. Use **connection warming** (throwaway request first) for cold back-end connections.

> **Rule of thumb:** Start with Repeater. Move to Turbo Intruder only when needed.

---

## 3. Attack Types

### 3.1 Limit Overrun

The most common race condition. Exploit the gap between **check** and **update** to exceed a business logic limit.

```
Request 1:  [CHECK ✓] ──────────────────────> [USE] ──> [UPDATE ✗ too late]
Request 2:       [CHECK ✓] ──> [USE] ──> [UPDATE]
                      ^
                 R1 hasn't updated yet —
                 both requests pass the check
```

**Common targets:**

| Target | Example |
|---|---|
| Discount codes | Redeem a one-time coupon twice |
| Gift cards | Apply the same gift card multiple times |
| Account balance | Withdraw more than available balance |
| Rate limits | Bypass brute-force protection |
| User interactions | Rate a product multiple times |
| CAPTCHA | Reuse a single CAPTCHA solution |

**Example — Burp Repeater (Community):**
1. Find `POST /cart/coupon` in Proxy history → send to Repeater
2. Confirm server-side session state (verify cart is session-keyed)
3. Group the tab, duplicate 19 times (20 total)
4. Send group in **sequence** first → confirm only first request succeeds
5. Remove code, send group **in parallel** → multiple should succeed

```
POST /cart/coupon HTTP/2
Cookie: session=<your-session>

csrf=<token>&coupon=PROMO20
```

**Example — Bypassing Rate Limits (Turbo Intruder):**
1. Send `POST /login` to Turbo Intruder
2. Use `examples/race-single-packet-attack.py` template
3. Queue one request per candidate password behind a single gate

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2)
    passwords = wordlists.clipboard
    for password in passwords:
        engine.queue(target.req, password, gate='1')
    engine.openGate('1')
```

---

### 3.2 Single-Endpoint Race

Send **parallel requests with different values to the same endpoint** to cause a state collision.

**Classic example — password reset token hijack:**
```
REQ1 [user=attacker] ──> write user_id=attacker ──────────────────>
REQ2 [user=victim]   ──────────> write user_id=victim
                                  ^
                             overwrites REQ1 before token is generated

Token generated AFTER collision:
  session.user_id = victim   ← from REQ2
  reset token     = abc123   ← sent to attacker's email (REQ1 triggered send)

Attacker resets victim's password using abc123.
```

> **Best targets:** Email-based endpoints — emails are sent in a background thread *after* the HTTP response, widening the race window.

**Example — Email Claim:**
1. `POST /my-account/change-email` → Repeater, group with two tabs
2. Tab 1: your exploit server address | Tab 2: `carlos@ginandjuice.shop`
3. Send **in parallel** — mismatch causes confirmation email for carlos to land in your inbox
4. Click the confirmation link → claim carlos's address → access admin panel

```
# Tab 1
csrf=<token>&email=attacker@exploit-server.net

# Tab 2
csrf=<token>&email=carlos@ginandjuice.shop
```

---

### 3.3 Multi-Endpoint Race

Send requests to **multiple different endpoints simultaneously** to exploit logic flaws across a multi-step workflow.

**Classic example — checkout bypass:**
```
Normal:  [add items] ──> [validate payment] ──> [confirm order]

Attack:
  [validate payment] ──────────────────────────> [confirm order]
       [add more items to basket] ──^
                              payment validated but
                              order not yet confirmed
```

**Aligning race windows:**
```
Requests not hitting the window together?
    │
    ├─> Connection setup delay?
    │       YES → Connection Warming
    │             Send 1-2 throwaway GET requests first
    │             to pre-establish the back-end connection
    │
    └─> Endpoints process at different speeds?
            YES → Abuse Rate/Resource Limits
                  Flood server with dummy requests to introduce
                  a delay, making timing viable for slower endpoints
```

**Example — Checkout Bypass:**
1. Send `POST /cart` and `POST /cart/checkout` to Repeater, group them
2. Baseline timing: send in sequence over a single connection
3. Add a `GET /` request at start to warm the connection, then remove it
4. Set `productId=1` (target item) in `POST /cart`
5. Add a gift card to cart, send both requests **in parallel**

```
# Request 1 — add item during checkout window
POST /cart
csrf=<token>&productId=1&redir=PRODUCT&quantity=1

# Request 2 — submit checkout
POST /cart/checkout
csrf=<token>
```

---

### 3.4 Hidden Multi-Step Sequences

A single HTTP request triggers **multiple hidden internal steps**. If two requests overlap during a transition, security logic can be skipped.

**Classic example — MFA bypass:**
```
Normal login (internal steps):
  [Step A] set user_id in session
  [Step B] enforce MFA flag

Attack:
  REQ1 (login)               ──> [Step A: user_id set] ──> [Step B: MFA enforced]
  REQ2 (sensitive endpoint)        ──────────^
                                   hits after Step A, before Step B
                                   → processed as fully authenticated, MFA bypassed
```

> The sub-state window can be microseconds wide. Vulnerability is invisible from outside — requires inferring internal steps.

---

### 3.5 Partial Construction

App creates an object across **multiple steps**, leaving a window where the object exists but is **not yet fully initialized**. Inject values matching the uninitialized state.

**Classic example — API key during registration:**
```
Normal registration:
  [Step 1] INSERT user record
  [Step 2] SET api_key          ← separate SQL statement

         ┌──── race window ────┐
         │   api_key = null    │  user exists, key not set
         └──────────┬──────────┘

Attacker sends: GET /api/user/info?user=victim&api-key[]=

Server compares:
  attacker input → [] (empty array / null equivalent)
  DB value       → null (uninitialized)
                   match → access granted
```

**Matching uninitialized values by framework:**

| Framework | Syntax | Server receives |
|---|---|---|
| PHP | `param[]` | empty array |
| Ruby on Rails | `param[key]` (no value) | `nil` object |
| Generic | `param=` | empty string |

---

## 4. Advanced Cases

### 4.1 Session-Based Locking

Some frameworks (e.g. PHP) process only **one request per session at a time**, queuing parallel requests and forcing sequential execution — making race conditions appear impossible.

```
Session-based locking:
REQ1 [session=abc] ──>[LOCK]──────────>[RELEASE]
REQ2 [session=abc]           ──>[LOCK]──>[RELEASE]   ← queued, waits for REQ1
REQ3 [session=abc]                       ──>[LOCK]──>[RELEASE]

Bypass — unique sessions per request:
REQ1 [session=aaa] ──>[LOCK]──>[RELEASE]
REQ2 [session=bbb] ──>[LOCK]──>[RELEASE]   ← no shared queue
REQ3 [session=ccc] ──>[LOCK]──>[RELEASE]      processed concurrently
```

> If requests process sequentially despite parallel send → suspect session-based locking → assign a unique session token to each request, then retest.

---

### 4.2 Time-Sensitive Attacks

Not a traditional race condition — exploits apps that generate tokens from **timestamps** instead of cryptographically secure randomness.

```
Vulnerable:  token = hash(current_timestamp)   ← same input = same output

Two requests arriving same millisecond → identical timestamp → identical tokens
```

**Classic example — password reset token collision:**
```
REQ1 (attacker's account) ──┐
                             ├──> arrive same millisecond
REQ2 (victim's account)   ──┘

  REQ1 token = hash(14:23:05.381) = a3f9...
  REQ2 token = hash(14:23:05.381) = a3f9...   ← identical

Attacker receives a3f9... → uses it to reset victim's password
```

**Key difference:**

| | Race Condition | Time-Sensitive Attack |
|---|---|---|
| **Goal** | Cause a state collision | Predict/duplicate a token |
| **Exploits** | Processing gap between steps | Predictable token generation |
| **Technique** | Parallel requests | Parallel requests |

**Example — PHP timestamp token (session-aware):**
1. `POST /forgot-password` → Repeater, duplicate into 2 tabs
2. Fetch a fresh session via `GET /forgot-password` (no cookie)
3. Replace session + CSRF in one tab with fresh values (two different sessions → two PHP threads)
4. Send pair **in parallel** until response times align
5. Change one tab to `username=carlos`, keep other as `username=wiener`
6. Send in parallel → matching timestamps → identical tokens
7. Copy your reset link, swap `username=wiener` to `username=carlos`, visit the modified URL

```
# Fresh session request (no cookie)
GET /forgot-password HTTP/2

# Request 1 — wiener's session
POST /forgot-password
Cookie: session=<wiener-session>
csrf=<wiener-csrf>&username=wiener

# Request 2 — fresh session
POST /forgot-password
Cookie: session=<new-session>
csrf=<new-csrf>&username=carlos

# Claim carlos's token
https://<lab>/forgot-password?username=carlos&token=<token-from-your-email>
```

---

## 5. Methodology & Defense

### 5.1 Predict, Probe, Prove

**Step 1 — Predict:** Focus on where collisions are *likely*.
```
Good candidates:
  ├─> Security-critical endpoints
  │     auth, MFA, password reset, financial transactions, privilege checks
  │
  └─> Endpoints where parallel requests touch the same record
        e.g. two reset requests writing to the same session
```
Ask: *If two requests hit this endpoint simultaneously, do they interact with the same data?*

**Step 2 — Probe:** Establish a baseline, then look for deviations.
```
1. Send requests sequentially → record normal response/state
2. Send same requests in parallel
   HTTP/2: single-packet attack | HTTP/1: last-byte sync
3. Compare:
   - Different HTTP response codes/bodies?
   - Unexpected email contents?
   - Unexpected state change?
   - Request succeeds when it normally shouldn't?
```
> Burp Suite: use **"Trigger race conditions"** custom action for one-click parallel send.

**Step 3 — Prove:** Confirm it's real and reproducible.
```
1. Remove superfluous requests → isolate the collision pair
2. Reproduce the effect consistently (one-time fluke ≠ proven vuln)
3. Map the impact → what does the resulting state actually allow?
```

---

### 5.2 Limit Overrun: Simpler Flow

```
1. IDENTIFY → single-use or rate-limited endpoint with security impact
2. EXECUTE  → send multiple parallel requests to the same endpoint
3. VERIFY   → did you exceed the intended limit?

Primary challenge: aligning the race window (often only milliseconds wide)
```

---

### 5.3 Alignment Troubleshooting

```
Requests not colliding?
    │
    ├─> Back-end connection delay?
    │       └─> Connection Warming
    │           Send a throwaway GET first to pre-establish connection
    │
    ├─> Endpoints processing at different speeds?
    │       └─> Abuse Rate/Resource Limits
    │           Flood with dummy requests to slow the faster endpoint
    │
    └─> Requests sequential despite parallel send?
            └─> Session-Based Locking
                Assign unique session token to each request
```

---

### 5.4 Prevention

Race conditions are caused by **non-atomic state changes**. Eliminate sub-states on sensitive endpoints.

| Problem | Fix |
|---|---|
| Check → use → update gap | Wrap in a single atomic DB transaction |
| Duplicate actions possible | Add DB uniqueness constraint |
| Mixed storage layers | Keep state in one consistent layer |
| Timestamp-based tokens | Use cryptographically secure random values |
| Session state mid-flight | Handle session consistency server-side |

> **Core principle:** State changes on sensitive endpoints must be ATOMIC — all steps succeed together, or none do. No sub-states = no race window = no collision.
