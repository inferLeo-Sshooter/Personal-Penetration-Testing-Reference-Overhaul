# Insecure Deserialization Cheatsheet

Quick-reference only — assumes you already know the fundamentals (serialize/deserialize, why it's dangerous, magic methods, gadget chains).

---

## 1. Spotting It

| Language | Format | Signature | Key funcs/interfaces |
|---|---|---|---|
| PHP | Readable string | `O:<len>:"ClassName":<n>:{...}`, markers `s:` `b:` `i:` | `serialize()` / `unserialize()` |
| Java | Binary | Hex `ac ed` / Base64 `rO0` | `Serializable`, `readObject()` |
| Ruby | "Marshaled" | — | `Marshal.dump` / `Marshal.load` |

**Where to look:** cookies, hidden fields, URL/query params, API bodies — anywhere user-controlled data flows back into the app. Burp Pro auto-flags serialized objects.

**Source-code checklist:** grep for `unserialize()`, `readObject()`, `Marshal.load`, classes implementing `Serializable`.

---

## 2. Attack Escalation Ladder

1. **Modify attribute value** → e.g. `isAdmin;b:0` → `b:1`. Re-serialize, resubmit.
2. **Modify data type** → exploit PHP loose `==`: send `password;i:0` to bypass a string check (works if PHP ≤7 non-numeric-string quirk, or always via "leading number" trick `5 == "5 of something"` even in PHP 8).
3. **Abuse legit functionality** → point an attacker-controlled field (e.g. `image_location`) at a sensitive path so a normal feature (delete pic) acts on it.
4. **Trigger magic methods** → `__wakeup()` / `__destruct()` (PHP), custom `readObject()` (Java) fire *automatically* during deserialization — no need to wait for app logic to touch the field.
5. **Inject arbitrary object** → swap the expected class for any other serializable class on the site; its magic method fires even if the app rejects the "wrong type" afterward.
6. **Gadget chain → RCE** → chain existing methods (kick-off gadget → ... → sink gadget) to reach command execution.

⚠️ **Always fix length/type markers after editing** (`s:6:"carlos"` → `s:3:"bob"`), or payload is rejected as corrupt. Use Hackvertor (Burp extension) for binary formats to avoid manual offset math.

---

## 3. Finding a Gadget Manually (source-code available)

1. Enumerate all serializable classes.
2. Look for magic methods: `__wakeup`, `__destruct`, `readObject`.
3. Check if that method does something dangerous with attacker-controlled data (file ops, exec, DB calls).
4. Craft serialized object of that class w/ malicious attributes.
5. Fix length markers, inject in place of expected object.

**Leaking source without access:** try `GET /path/File.php~` (tilde) — some servers return backup/source instead of executing it.

---

## 4. Pre-Built Gadget Chain Tools

### ysoserial (Java)
```bash
# Java ≤15
java -jar ysoserial-all.jar CommonsCollections4 'cmd' | base64

# Java 16+ (needs module-access flags)
java --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
     --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
     --add-opens=java.base/java.net=ALL-UNNAMED \
     --add-opens=java.base/java.util=ALL-UNNAMED \
     -jar ysoserial-all.jar [payload] '[cmd]'
```
Swap into session cookie (URL-encode) → send.

**Detection-only chains** (no RCE needed):
- `URLDNS` — forces DNS lookup to your Collaborator; any Java version, no vulnerable lib needed.
- `JRMPClient` — forces TCP connection attempt; compare local vs. firewalled-IP timing to confirm blind deserialization.

### PHPGGC (PHP)
```bash
./phpggc Symfony/RCE4 exec 'cmd' | base64
```
Typical signed-cookie workflow: tamper cookie → read leaked error/debug info (e.g. `/cgi-bin/phpinfo.php`) → grab `SECRET_KEY` → generate payload → **re-sign** cookie with leaked key (HMAC-SHA1) → swap in → send.

### Documented (non-tool) chains — e.g. Ruby
No automated tool for the framework? Search for a published write-up/PoC script, adapt the command, run it (e.g. via a throwaway Docker container matching the target Ruby version), swap the marshaled output into the cookie.

---

## 5. Core Mental Model

```
attacker data → kick-off gadget (magic method) → gadget → gadget → SINK (exec/file-write/etc.)
```
Attacker never writes the chain — it already exists in app/library code. They only control the *input* to the first gadget. Patching one known chain (e.g. one ysoserial payload) doesn't fix the root cause — the real vuln is deserializing untrusted data at all.

---

## 6. Golden Rule / Fixes

- Don't deserialize untrusted data — full stop, if avoidable.
- If unavoidable: sign/encrypt + verify integrity before deserializing, and use a strict class allow-list.
- Never trust deserialized values for auth/authz decisions without independent verification.
