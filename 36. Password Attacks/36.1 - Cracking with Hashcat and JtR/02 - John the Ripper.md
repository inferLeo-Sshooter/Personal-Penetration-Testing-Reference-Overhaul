# John the Ripper Basics

# Table of Contents

1. [What is it?](#1-what-is-it)
2. [Cracking modes](#2-cracking-modes)
   - [Single crack mode](#single-crack-mode)
   - [Wordlist mode](#wordlist-mode)
   - [Incremental mode](#incremental-mode)
3. [Identifying hash formats](#3-identifying-hash-formats)
4. [Cracking files](#4-cracking-files)

---

## 1. What is it?

**John the Ripper** (nicknamed "JtR" or just "john") is one of the most famous password-cracking tools used in penetration testing. It's open-source, first came out in 1996 for UNIX systems, and is still a go-to tool in the security industry today.

Most people use the **"jumbo" version**, which adds extra speed, multi-language wordlist support, and 64-bit compatibility — basically the upgraded edition. It also comes bundled with helper tools that convert odd file types into a format John can actually crack, and it's updated regularly to keep up with new hash types.

John has three main cracking "modes," each suited to a different situation.

---

## 2. Cracking modes

### Single Crack Mode

`Single crack mode` is a rule-based technique ideal for Linux credentials. **Guess based on what we know about the person**

Instead of guessing randomly, this mode builds password guesses out of **information already available about the user** — their username, home folder name, or full name — then applies common tweaks people make to passwords (adding numbers, capitalizing letters, etc.).

Example logic: if someone's name is *Bob Smith*, John might try `Smith1`, `BobSmith!`, `smith123`, and similar variations automatically.

**Example scenario:** Say we found this line in a leaked `passwd` file:

```
cat passwd

r0lf:$6$ues25dIanlctrWxg$nZHVz2z4kCy1760Ee28M1xtHdGoy0C2cYzZ8l2sVa1kIa8K9gAcdBP.GI6ng/qA4oaMrgElZ1Cb9OeXO4Fvy3/:0:0:Rolf Sebastian:/home/r0lf:/bin/bash
```

From this, John can pull out:
- Username: `r0lf`
- Real name: `Rolf Sebastian`
- Home directory: `/home/r0lf`

It uses those pieces to generate likely password guesses and test them against the hash:

```bash
$ john --single passwd

Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[...SNIP...]        (r0lf)     
1g 0:00:00:00 DONE 1/3 (2025-04-10 07:47) 12.50g/s 5400p/s 5400c/s 5400C/s NAITSABESFL0R..rSebastiannaitsabeSr
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

In this case — cracked. ✅ This mode works best against Linux-style password files where personal info is embedded nearby.

---

### Wordlist Mode

`Wordlist Mode` or **dictionary attack** — "try a list of known passwords" 

This is the classic dictionary attack: feed John a text file full of candidate passwords (one per line), and it tries each one against the target hash.

```bash
$ john --wordlist=<wordlist_file> <hash_file>
```

- You can supply multiple wordlists at once, separated by commas.
- You can also apply **rules** (`--rules`) that automatically mutate each word — e.g., turning `password` into `P@ssword1`, `password123`, `Password!`, etc. — multiplying your effective wordlist without writing it all out by hand.

---

### Incremental Mode

This is John's smart brute-force mode, but smarter than a naive "try everything alphabetically" approach. It uses a statistical model (built from training data on real passwords) to guess the *most likely* character combinations first, rather than wasting time on unlikely ones.

```bash
$ john --incremental <hash_file>
```

- It doesn't need a wordlist — it generates guesses on the fly.
- It's the most thorough mode (it will eventually try everything in the defined character set), but also the slowest and most resource-heavy.
- Default character sets and length limits come from John's config file (`john.conf`), but you can customize them:

```bash
$ grep '# Incremental modes' -A 100 /etc/john/john.conf
```

```ini
# Incremental modes

# This is for one-off uses (make your own custom.chr).
# A charset can now also be named directly from command-line, so no config
# entry needed: --incremental=whatever.chr
[Incremental:Custom]
File = $JOHN/custom.chr
MinLen = 0

# The theoretical CharCount is 211, we've got 196.
[Incremental:UTF8]
File = $JOHN/utf8.chr
MinLen = 0
CharCount = 196

# This is CP1252, a super-set of ISO-8859-1.
# The theoretical CharCount is 219, we've got 203.
[Incremental:Latin1]
File = $JOHN/latin1.chr
MinLen = 0
CharCount = 203

[Incremental:ASCII]
File = $JOHN/ascii.chr
MinLen = 0
MaxLen = 13
CharCount = 95

...SNIP...
```

Narrowing the character set or length range makes this mode run faster by cutting down the search space.

---

## 3. Identifying hash formats

Sometimes you'll find a hash and have no idea what algorithm made it, e.g.:

```
193069ceb0461e1d40d216e32c79c704
```

A couple of ways to identify it:
- Consult [JtR's sample hash documentation](https://openwall.info/wiki/john/sample-hashes) or [PentestMonkey's hash list](https://pentestmonkey.net/cheat-sheet/john-the-ripper-hash-formats) for format examples
- Use tools like [hashID](https://github.com/psypanda/hashID) to check hashes against built-in databases for potential format suggestions:

By adding the `-j` flag, `hashID` will, in addition to the hash format, list the corresponding JtR format:

```bash
$ hashid -j 193069ceb0461e1d40d216e32c79c704

Analyzing '193069ceb0461e1d40d216e32c79c704'
[+] MD2 [JtR Format: md2]
[+] MD5 [JtR Format: raw-md5]
[+] MD4 [JtR Format: raw-md4]
[+] Double MD5 
[+] LM [JtR Format: lm]
[+] RIPEMD-128 [JtR Format: ripemd-128]
[+] Haval-128 [JtR Format: haval-128-4]
[+] Tiger-128 
[+] Skein-256(128) 
[+] Skein-512(128) 
[+] Lotus Notes/Domino 5 [JtR Format: lotus5]
[+] Skype 
[+] Snefru-128 [JtR Format: snefru-128]
[+] NTLM [JtR Format: nt]
[+] Domain Cached Credentials [JtR Format: mscach]
[+] Domain Cached Credentials 2 [JtR Format: mscach2]
[+] DNSSEC(NSEC3) 
[+] RAdmin v2.x [JtR Format: radmin]
```

Notice it returns a *list* of possibilities, not one definite answer — many hash types produce outputs of the same length/format, so you often need outside context (where the hash came from, what system generated it) to narrow it down. In this example, it turned out to be RIPEMD-128.

Once you know the format, tell John which one to use with `--format`:

```bash
$ john --format=raw-md5 hash_file
```

John supports **hundreds** of hash formats — a few common ones:

| Hash format | Example command | Description |
|-------------|----------------|-------------|
| afs | `john --format=afs [...] <hash_file>` | AFS (Andrew File System) password hashes |
| bfegg | `john --format=bfegg [...] <hash_file>` | bfegg hashes used in Eggdrop IRC bots |
| bf | `john --format=bf [...] <hash_file>` | Blowfish-based crypt(3) hashes |
| bsdi | `john --format=bsdi [...] <hash_file>` | BSDi crypt(3) hashes |
| crypt(3) | `john --format=crypt [...] <hash_file>` | Traditional Unix crypt(3) hashes |
| des | `john --format=des [...] <hash_file>` | Traditional DES-based crypt(3) hashes |
| dmd5 | `john --format=dmd5 [...] <hash_file>` | DMD5 (Dragonfly BSD MD5) password hashes |
| dominosec | `john --format=dominosec [...] <hash_file>` | IBM Lotus Domino 6/7 password hashes |
| EPiServer SID hashes | `john --format=episerver [...] <hash_file>` | EPiServer SID (Security Identifier) password hashes |
| hdaa | `john --format=hdaa [...] <hash_file>` | hdaa password hashes used in Openwall GNU/Linux |
| hmac-md5 | `john --format=hmac-md5 [...] <hash_file>` | hmac-md5 password hashes |
| hmailserver | `john --format=hmailserver [...] <hash_file>` | hmailserver password hashes |
| ipb2 | `john --format=ipb2 [...] <hash_file>` | Invision Power Board 2 password hashes |
| krb4 | `john --format=krb4 [...] <hash_file>` | Kerberos 4 password hashes |
| krb5 | `john --format=krb5 [...] <hash_file>` | Kerberos 5 password hashes |
| LM | `john --format=LM [...] <hash_file>` | LM (Lan Manager) password hashes |
| lotus5 | `john --format=lotus5 [...] <hash_file>` | Lotus Notes/Domino 5 password hashes |
| mscash | `john --format=mscash [...] <hash_file>` | MS Cache password hashes |
| mscash2 | `john --format=mscash2 [...] <hash_file>` | MS Cache v2 password hashes |
| mschapv2 | `john --format=mschapv2 [...] <hash_file>` | MS CHAP v2 password hashes |
| mskrb5 | `john --format=mskrb5 [...] <hash_file>` | MS Kerberos 5 password hashes |
| mssql05 | `john --format=mssql05 [...] <hash_file>` | MS SQL 2005 password hashes |
| mssql | `john --format=mssql [...] <hash_file>` | MS SQL password hashes |
| mysql-fast | `john --format=mysql-fast [...] <hash_file>` | MySQL fast password hashes |
| mysql | `john --format=mysql [...] <hash_file>` | MySQL password hashes |
| mysql-sha1 | `john --format=mysql-sha1 [...] <hash_file>` | MySQL SHA1 password hashes |
| NETLM | `john --format=netlm [...] <hash_file>` | NETLM (NT LAN Manager) password hashes |
| NETLMv2 | `john --format=netlmv2 [...] <hash_file>` | NETLMv2 (NT LAN Manager version 2) password hashes |
| NETNTLM | `john --format=netntlm [...] <hash_file>` | NETNTLM (NT LAN Manager) password hashes |
| NETNTLMv2 | `john --format=netntlmv2 [...] <hash_file>` | NETNTLMv2 (NT LAN Manager version 2) password hashes |
| NEThalfLM | `john --format=nethalflm [...] <hash_file>` | NEThalfLM (NT LAN Manager) password hashes |
| md5ns | `john --format=md5ns [...] <hash_file>` | md5ns (MD5 namespace) password hashes |
| nsldap | `john --format=nsldap [...] <hash_file>` | nsldap (OpenLDAP SHA) password hashes |
| ssha | `john --format=ssha [...] <hash_file>` | ssha (Salted SHA) password hashes |
| NT | `john --format=nt [...] <hash_file>` | NT (Windows NT) password hashes |
| openssha | `john --format=openssha [...] <hash_file>` | OPENSSH private key password hashes |
| oracle11 | `john --format=oracle11 [...] <hash_file>` | Oracle 11 password hashes |
| oracle | `john --format=oracle [...] <hash_file>` | Oracle password hashes |
| pdf | `john --format=pdf [...] <hash_file>` | PDF (Portable Document Format) password hashes |
| phpass-md5 | `john --format=phpass-md5 [...] <hash_file>` | PHPass-MD5 (Portable PHP password hashing framework) password hashes |
| phps | `john --format=phps [...] <hash_file>` | PHPS password hashes |
| pix-md5 | `john --format=pix-md5 [...] <hash_file>` | Cisco PIX MD5 password hashes |
| po | `john --format=po [...] <hash_file>` | Po (Sybase SQL Anywhere) password hashes |
| rar | `john --format=rar [...] <hash_file>` | RAR (WinRAR) password hashes |
| raw-md4 | `john --format=raw-md4 [...] <hash_file>` | Raw MD4 password hashes |
| raw-md5 | `john --format=raw-md5 [...] <hash_file>` | Raw MD5 password hashes |
| raw-md5-unicode | `john --format=raw-md5-unicode [...] <hash_file>` | Raw MD5 Unicode password hashes |
| raw-sha1 | `john --format=raw-sha1 [...] <hash_file>` | Raw SHA1 password hashes |
| raw-sha224 | `john --format=raw-sha224 [...] <hash_file>` | Raw SHA224 password hashes |
| raw-sha256 | `john --format=raw-sha256 [...] <hash_file>` | Raw SHA256 password hashes |
| raw-sha384 | `john --format=raw-sha384 [...] <hash_file>` | Raw SHA384 password hashes |
| raw-sha512 | `john --format=raw-sha512 [...] <hash_file>` | Raw SHA512 password hashes |
| salted-sha | `john --format=salted-sha [...] <hash_file>` | Salted SHA password hashes |
| sapb | `john --format=sapb [...] <hash_file>` | SAP CODVN B (BCODE) password hashes |
| sapg | `john --format=sapg [...] <hash_file>` | SAP CODVN G (PASSCODE) password hashes |
| sha1-gen | `john --format=sha1-gen [...] <hash_file>` | Generic SHA1 password hashes |
| skey | `john --format=skey [...] <hash_file>` | S/Key (One-time password) hashes |
| ssh | `john --format=ssh [...] <hash_file>` | SSH (Secure Shell) password hashes |
| sybasease | `john --format=sybasease [...] <hash_file>` | Sybase ASE password hashes |
| xsha | `john --format=xsha [...] <hash_file>` | xsha (Extended SHA) password hashes |
| zip | `john --format=zip [...] <hash_file>` | ZIP (WinZip) password hashes |

---

## 4. Cracking files

John can also crack **password-protected files** — but first the file needs to be converted into a hash format John understands. That's what the "**2john**" tools are for (they come bundled with John).

General pattern:

```bash
$ <conversion_tool> <file_to_crack> > file.hash
$ john file.hash
```

For example, you found a password-protected `pdf` file, you can try cracking it with `john` as follow:
```bash
$ pdf2john locked_pdf_file > pdf_file.hash
$ john pdf_file.hash
```

Some examples:

| Tool | Description |
|------|-------------|
| pdf2john | Converts PDF documents for John |
| ssh2john | Converts SSH private keys for John |
| mscash2john | Converts MS Cash hashes for John |
| keychain2john | Converts OS X keychain files for John |
| rar2john | Converts RAR archives for John |
| pfx2john | Converts PKCS#12 files for John |
| truecrypt_volume2john | Converts TrueCrypt volumes for John |
| keepass2john | Converts KeePass databases for John |
| vncpcap2john | Converts VNC PCAP files for John |
| putty2john | Converts PuTTY private keys for John |
| zip2john | Converts ZIP archives for John |
| hccap2john | Converts WPA/WPA2 handshake captures for John |
| office2john | Converts MS Office documents for John |
| wpa2john | Converts WPA/WPA2 handshakes for John |
|...SNIP...|	...SNIP...|

You can see all the ones installed on a system with:

```bash
$ locate *2john*

/usr/bin/bitlocker2john
/usr/bin/dmg2john
/usr/bin/gpg2john
/usr/bin/hccap2john
/usr/bin/keepass2john
/usr/bin/putty2john
/usr/bin/racf2john
...SNIP...
```

---

## Quick Recap

| Mode | How it guesses | Best for |
|------|------------------|----------|
| Single Crack | Based on username/real name/home dir | Linux accounts with personal info nearby |
| Wordlist | Tries a pre-made password list (+ optional mutation rules) | Fast, practical everyday cracking |
| Incremental | Statistically-smart brute force, no wordlist needed | Thorough last-resort attempts |

**Key idea:** John isn't one single attack — it's a toolkit that lets you pick the smartest strategy for the situation, rather than always brute-forcing blindly.
