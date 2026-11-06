# Cracking Protected Archives

## Table of Contents
- [Why archives matter here](#why-archives-matter-here)
- [Cracking ZIP Files](#cracking-zip-files)
- [Cracking OpenSSL-Encrypted GZIP Files](#cracking-openssl-encrypted-gzip-files)
- [Cracking BitLocker-Encrypted Drives](#cracking-bitlocker-encrypted-drives)
  - [Extracting the hash](#extracting-the-hash)
  - [Cracking it with Hashcat](#cracking-it-with-hashcat)
  - [Mounting the Drive After Cracking](#mounting-the-drive-after-cracking)

---

## Why archives matter here

Beyond single files, we often run into **password-protected archives** — ZIPs, RARs, and similar compressed bundles. Think of a real-world scenario: a company needs to send a client a bundle of deliverables (an Excel sheet, a PDF, a Word doc, a slideshow). Sending each file separately gets messy fast, especially across a big organization juggling many projects at once — files get lost, naming gets confusing. So instead, people commonly zip everything into one organized, compressed archive.

There are a huge number of archive formats out there — `.tar`, `.gz`, `.rar`, `.zip`, `.7z`, `.vmdk/.vmx`, `.kdbx`, `.deb`, and more. You can even pull a live list instead of memorizing them:

```bash
$ curl -s https://fileinfo.com/filetypes/compressed | html2text | awk '{print tolower($1)}' | grep "\." | tee -a compressed_ext.txt
```
```
.mint
.zhelp
.b6z
.fzpz
.zst
.apz
.ufs.uzip
.vrpackage
.sfg
.gzip
.xapk
.rar
.pkg.tar.xz
<SNIP>
```

Not every archive format supports built-in password protection — some rely on separate encryption tools layered on top (TAR files, for example, are often encrypted afterward with `openssl` or `gpg`). Because there are so many formats and encryption methods, this guide walks through a few representative examples rather than every possible combination.

---

## Cracking ZIP Files

ZIP is extremely common, especially in Windows environments. The process mirrors what we've already seen for other file types — extract a crackable hash, then run it through John.

```bash
$ zip2john ZIP.zip > zip.hash
$ cat zip.hash
```
```
ZIP.zip/customers.csv:$pkzip2$1*2*2*0*2a*1e*490e7510*0*42*0*...
```

Then crack it the usual way:

```bash
$ john --wordlist=rockyou.txt zip.hash
```
```
Loaded 1 password hash (PKZIP [32/64])
1234             (ZIP.zip/customers.csv)
1g 0:00:00:00 DONE
Session completed
```

```bash
$ john zip.hash --show
ZIP.zip/customers.csv:1234:customers.csv:ZIP.zip::ZIP.zip
1 password hash cracked, 0 left
```

---

## Cracking OpenSSL-Encrypted GZIP Files

Sometimes a file doesn't *look* encrypted from its extension alone — especially if that format doesn't natively support passwords (like plain GZIP). The `file` command can reveal what's actually going on under the hood:

```bash
$ file GZIP.gzip
GZIP.gzip: openssl enc'd data with salted password
```

That confirms it's been encrypted using `openssl` rather than GZIP's own (nonexistent) password feature.

Because OpenSSL-encrypted files don't produce a clean "hash" the way ZIP or SSH keys do, the more reliable approach is to literally **try decrypting the file with every password in a list**, and see which one actually works:

```bash
$ for i in $(cat password_list.txt); do
    openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz
  done
```

Most attempts will fail loudly with errors like:
```
gzip: stdin: not in gzip format
tar: Child returned status 1
```
— these are expected and safe to ignore. If the right password is in the list, the loop will eventually succeed silently, and a new extracted file will just appear:

```bash
$ ls
customers.csv  GZIP.gzip  password_list.txt
```

---

## Cracking BitLocker-Encrypted Drives

**BitLocker** is Microsoft's built-in full-disk encryption feature (available since Windows Vista), using AES with 128-bit or 256-bit keys. If someone forgets their password or PIN, Windows lets them fall back on a **48-digit recovery key** generated during setup. In corporate settings, employees sometimes use BitLocker-encrypted virtual drives on company laptops to keep personal files private.

### Extracting the hash

The `bitlocker2john` tool can pull out up to four possible hashes from a BitLocker-protected drive — two relate to the actual password, two relate to the 48-digit recovery key. Since the recovery key is long and randomly generated, it's essentially impossible to brute-force, so in practice attention focuses on the **password hash** instead:

```bash
$ bitlocker2john -i Backup.vhd > backup.hashes
$ grep "bitlocker\$0" backup.hashes > backup.hash
$ cat backup.hash
```
```
$bitlocker$0$16$02b329c0453b9273f2fc1b927443b5fe$1048576$12$...
```

### Cracking it with Hashcat

BitLocker hashes use Hashcat mode `22100`. Because BitLocker uses strong AES encryption, cracking can be noticeably slower than cracking something like a raw MD5 hash:

```bash
$ hashcat -a 0 -m 22100 '$bitlocker$0$16$02b329c0...8ec54f' /usr/share/wordlists/rockyou.txt
```

```
Status...........: Cracked
Hash.Mode........: 22100 (BitLocker)
Speed.#1.........: 25 H/s
Recovered........: 1/1 (100.00%)
$bitlocker$0$...:1234qwer
```

Notice the cracking speed here — just **25 hashes per second**, compared to the millions/sec we saw with plain MD5 earlier. That's the cost of strong encryption: even with the right tool and wordlist, it takes real computation time per guess.

---

### Mounting the Drive After Cracking

Once you have the password, you still need to actually mount and browse the decrypted drive.

#### On Windows

Just double-click the `.vhd` file (Windows will initially complain since it's encrypted). After it mounts, double-click the BitLocker volume and enter the password when prompted.

#### On Linux or macOS

Use a tool called **dislocker**:

```bash
$ sudo apt-get install dislocker
```

Set up two mount points:
```bash
$ sudo mkdir -p /media/bitlocker
$ sudo mkdir -p /media/bitlockermount
```

Attach the virtual disk as a loop device, decrypt it, then mount the decrypted contents:
```bash
$ sudo losetup -f -P Backup.vhd
$ sudo dislocker /dev/loop0p2 -u1234qwer -- /media/bitlocker
$ sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
```

Now you can browse the files normally:
```bash
$ cd /media/bitlockermount/
$ ls -la
```

When finished, unmount cleanly:
```bash
$ sudo umount /media/bitlockermount
$ sudo umount /media/bitlocker
```

---

## Quick Recap

| Archive type | How to get a crackable hash | Crack with |
|--------------|-------------------------------|------------|
| ZIP | `zip2john` | John the Ripper |
| OpenSSL-encrypted GZIP | No clean hash — try decrypting directly in a loop | `openssl` + wordlist |
| BitLocker drive | `bitlocker2john` (focus on the `$bitlocker$0$...` hash) | Hashcat (mode `22100`) |

**Key idea:** Not every protected format works the same way — some give you a clean hash to crack offline (ZIP, BitLocker), while others require brute-force decrypting the file directly (OpenSSL-encrypted archives). Strong encryption like AES also means cracking speed can drop dramatically, so success often depends as much on wordlist quality as raw computing power.
