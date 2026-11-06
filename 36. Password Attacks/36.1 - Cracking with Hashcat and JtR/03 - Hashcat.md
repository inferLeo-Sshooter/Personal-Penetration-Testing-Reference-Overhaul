# Hashcat Basics

## Table of contents
- [1. What is it](#1-what-is-it)
- [2. Identifying Hash types](#2-Identifying-Hash-types)
- [3. Attack modes](#3-Attack-modes)
  - [Dictionary attack](#Dictionary-attack)
  - [Mask attack](#Mask-attack)

---

## 1. What is it?

**Hashcat** is another extremely popular password-cracking tool, available on Linux, Windows, and macOS. It used to be closed-source (2009–2015) but is now fully open-source. Its biggest strength is **GPU acceleration** — graphics cards are much faster than CPUs at the repetitive math hashing requires, so Hashcat can test way more guesses per second than CPU-only tools. Like John the Ripper, it supports several different "attack modes" for different situations.

### Basic command structure

```bash
$ hashcat -a 0 -m 0 <hashes> [wordlist, rule, mask, ...]
```

- `-a` → which **attack mode** to use (dictionary, mask, etc.)
- `-m` → which **hash type** you're targeting (MD5, SHA-256, etc.)
- `<hashes>` → either one hash, or a file full of hashes of the same type
- everything after that → extra inputs depending on the mode (wordlist file, rule file, mask pattern...)

---

## 2. Identifying Hash Type

Hashcat assigns a numeric **ID** to every hash algorithm it supports (hundreds of them). You can list them all with:

```bash
$ hashcat --help
```

```
...SNIP...

- [ Hash modes ] -

      # | Name                                                       | Category
  ======+============================================================+======================================
    900 | MD4                                                        | Raw Hash
      0 | MD5                                                        | Raw Hash
    100 | SHA1                                                       | Raw Hash
   1300 | SHA2-224                                                   | Raw Hash
   1400 | SHA2-256                                                   | Raw Hash
  10800 | SHA2-384                                                   | Raw Hash
   1700 | SHA2-512                                                   | Raw Hash
  17300 | SHA3-224                                                   | Raw Hash
  17400 | SHA3-256                                                   | Raw Hash
  17500 | SHA3-384                                                   | Raw Hash
  17600 | SHA3-512                                                   | Raw Hash
   6000 | RIPEMD-160                                                 | Raw Hash
    600 | BLAKE2b-512                                                | Raw Hash
  11700 | GOST R 34.11-2012 (Streebog) 256-bit, big-endian           | Raw Hash
  11800 | GOST R 34.11-2012 (Streebog) 512-bit, big-endian           | Raw Hash
   6900 | GOST R 34.11-94                                            | Raw Hash
  17010 | GPG (AES-128/AES-256 (SHA-1($pass)))                       | Raw Hash
   5100 | Half MD5                                                   | Raw Hash
  17700 | Keccak-224                                                 | Raw Hash
  17800 | Keccak-256                                                 | Raw Hash
  17900 | Keccak-384                                                 | Raw Hash
  18000 | Keccak-512                                                 | Raw Hash
   6100 | Whirlpool                                                  | Raw Hash
  10100 | SipHash                                                    | Raw Hash
     70 | md5(utf16le($pass))                                        | Raw Hash
    170 | sha1(utf16le($pass))                                       | Raw Hash
   1470 | sha256(utf16le($pass))                                     | Raw Hash
...SNIP...
```

If you're not sure what type of hash you're looking at, two easy options:

1. Check Hashcat's own online [example-hash list]((https://hashcat.net/wiki/doku.php?id=example_hashes)), which pairs sample hashes with their correct mode number.
2. Run it through `hashID`, which guesses for you by specifying the `-m` argument:

```bash
$ hashid -m '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'
```
```
Analyzing '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'
[+] MD5 Crypt [Hashcat Mode: 500]
[+] Cisco-IOS(MD5) [Hashcat Mode: 500]
[+] FreeBSD MD5 [Hashcat Mode: 500]
```

---

## 3. Attack modes

Hashcat has many different attack mode, including `dictionary`, `mask`, `combinator`, and `association`.

### Dictionary Attack

Dictionary Attack (`-a 0`) works exactly like you'd expect — Hashcat tries every word in a wordlist file against the hash until one matches (or the list runs out).

**Example:** Say we pulled this MD5 hash from a database:
```
e3e3ec5831ad5e7288241960e5d4fdb8
```

We know it's MD5 (mode `0`), so we run it against the classic `rockyou.txt` wordlist:

```bash
$ hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 /usr/share/wordlists/rockyou.txt
```

```
Status...........: Cracked
Hash.Target......: e3e3ec5831ad5e7288241960e5d4fdb8
Speed.#1.........: 1706.6 kH/s
Recovered........: 1/1 (100.00%)
Candidates.#1....: 010292 -> spongebob9
```

Cracked — turned out to be a variation of "spongebob9."

#### Boosting a dictionary attack with rules

Sometimes the wordlist alone isn't enough — the real password might be a *slightly modified* version of a dictionary word (like `Password1!` instead of `password`). That's where **rule files** come in: they apply mutations like capitalizing letters, appending numbers, or swapping letters for "leet speak" symbols (`a` → `@`, etc.), effectively multiplying your wordlist's coverage.

Rule files that ship with Hashcat live here:

```bash
$ ls -l /usr/share/hashcat/rules
```
```
total 2852
-rw-r--r-- 1 root root 309439 Apr 24  2024 Incisive-leetspeak.rule
-rw-r--r-- 1 root root  35802 Apr 24  2024 InsidePro-HashManager.rule
-rw-r--r-- 1 root root  20580 Apr 24  2024 InsidePro-PasswordsPro.rule
-rw-r--r-- 1 root root  64068 Apr 24  2024 T0XlC-insert_00-99_1950-2050_toprules_0_F.rule
-rw-r--r-- 1 root root   2027 Apr 24  2024 T0XlC-insert_space_and_special_0_F.rule
...SNIP...
```

**Example:** A second leaked hash `1b0556a75770563578569ae21392630c` didn't crack with rockyou.txt alone. So we add the `best64.rule` ruleset (64 common password tweaks) using `-r`:

```bash
$ hashcat -a 0 -m 0 1b0556a75770563578569ae21392630c \
    /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule
```

```
Status...........: Cracked
Speed.#1.........: 13624.4 kH/s
Recovered........: 1/1 (100.00%)
Candidates.#1....: slimshady -> drousd
```

The rules let Hashcat try tweaked versions of each dictionary word, not just the exact word — and this time it worked.

---

### Mask attack

Mask attack (`-a 3`) is a **smarter brute-force**. Instead of blindly trying every possible character in every position, you tell Hashcat exactly what *pattern* the password probably follows — e.g., "8 characters long, first one capital, rest lowercase, ending in a digit." This massively shrinks the number of guesses needed compared to a pure brute-force sweep.

### How masks work

Each position in the password is represented by a symbol standing for a character set:

| Symbol | Characters it covers |
|--------|----------------------|
| `?l` | lowercase letters (a–z) |
| `?u` | uppercase letters (A–Z) |
| `?d` | digits (0–9) |
| `?h` | lowercase hex (0–9, a–f) |
| `?H` | uppercase hex (0–9, A–F) |
| `?s` | symbols/punctuation: «space»!"#$%&'()*+,-./:;<=>?@[]^_`{ |
| `?a` | all of the above combined (`?l?u?d?s`) |
| `?b` | every possible byte value (`0x00 - 0xff`) |

You can also define your own custom sets with `-1`, `-2`, `-3`, `-4` and reference them as `?1`, `?2`, etc.

**Example:** Suppose we suspect a password pattern of: 1 uppercase letter → 4 lowercase letters → 1 digit → 1 symbol. That becomes the mask:

```
?u?l?l?l?l?d?s
```

Running it:

```bash
$ hashcat -a 3 -m 0 1e293d6912d074c0fd15844d803400dd '?u?l?l?l?l?d?s'
```

```
Status...........: Cracked
Guess.Mask.......: ?u?l?l?l?l?d?s [7]
Speed.#1.........: 101.6 MH/s
Recovered........: 1/1 (100.00%)
Candidates.#1....: Uayvf7- -> Dikqn5!
```

Cracked, and much faster than a generic brute-force would've been — because we told Hashcat exactly where to focus its guessing instead of making it check every conceivable character combination everywhere.

---

## Quick Recap

| Mode | Flag | How it works | Best for |
|------|------|----------------|----------|
| Dictionary | `-a 0` | Tries words from a wordlist (optionally mutated with rules) | Common/reused passwords |
| Mask | `-a 3` | Smart brute-force following a known pattern | Passwords with a known structure/length |

**Key idea:** Hashcat's power comes from combining raw GPU speed with *smart targeting* — rules narrow dictionary attacks, and masks narrow brute-force attacks, so you're not wasting time testing guesses that don't fit what you already know about the password.
