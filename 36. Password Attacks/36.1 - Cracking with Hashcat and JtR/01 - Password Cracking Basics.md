# Password Cracking Basics

## 1. Why passwords are hashed

Systems don't usually store your actual password. Instead, they run it through a **hash function** — a one-way math formula that turns any text into a fixed-length scrambled string. The idea is that even if someone steals the database, they shouldn't be able to read the real passwords.

Two popular hash functions are **MD5** and **SHA-256**. Example:

```bash
$ echo -n "Soccer06!" | md5sum
40291c1d19ee11a7df8495c4cccefdfa

$ echo -n "Soccer06!" | sha256sum
a025dc6fabb09c2b8bfe23b5944635f9b68433ebd9a1a09453dd4fee00766d9
```

Same input, always the same output — but you can't reverse the output back into the original text just by looking at it. That's what "one-way" means.

When someone tries to recover the original password from a hash, that's called **password cracking**. There are three main approaches, from fastest-but-limited to slow-but-guaranteed.

---

## 2. Rainbow Tables

A **rainbow table - ("lookup the answer")** is basically a giant pre-made phonebook: a list of common passwords paired with their hash values, calculated ahead of time.

| Password | MD5 Hash |
|----------|----------|
| 123456 | e10adc3949ba59abbe56e057f20f883e |
| password | 5f4dcc3b5aa765d61d8327deb882cf99 |
| iloveyou | f25a2fc72690b780b2a14e140ef6a9e0 |

If an attacker sees a hash that matches one already in their table, they instantly know the password — no guessing required. It's fast, but only works for passwords someone already bothered to pre-hash.

### The defense: salting

A **salt** is random extra data glued onto the password *before* hashing, so the output no longer matches anything in a pre-built table.

```bash
# Without salt
$ echo -n "Soccer06!" | md5sum
40291c1d19ee11a7df8495c4cccefdfa

# With salt "Th1sIsTh3S@lt_" added in front
$ echo -n "Th1sIsTh3S@lt_Soccer06!" | md5sum
90a10ba83c04e7996bc53373170b547
```

Totally different hash, even though the password is identical. The salt itself isn't secret — it's usually stored right alongside the hash — but it forces attackers to redo their rainbow table for every single salt value, which quickly becomes impractical. Even one random byte of salt multiplies the attacker's workload 256x.

---

## 3. Brute Force

**Brute Force — ("try literally everything")** means testing every possible combination of characters — `a`, `aa`, `ab`, ... `zzzz`, etc. — until one matches.

```
Sxejd → 2cdc813ef26e6d20c854adb107279338
Sxeje → 7703349a1f943f9da6d1dfcda51f3b63
Sxejf → db914f10854b97946046eabab2287178
...
```

- While **time-consuming** for long passwords, shorter passwords (**<9 characters**) are vulnerable even on consumer hardware.
- Rarely used due to duration and is typically replaced by more efficient `mask attacks`.


Speed also depends heavily on the hash type:

| Hash type | Rough guesses/second (consumer laptop) |
|-----------|------------------------------------------|
| MD5 | ~5,000,000/sec |
| DCC2 | ~10,000/sec |

Because brute force is so slow for strong passwords, it's rarely used directly — a smarter variant called a **mask attack** (brute-forcing only the *parts* of the password you're unsure about) is more common in practice.

---

## 4. Dictionary Attack — "try the likely ones first"

**Dictionary Attack — ("try the likely ones first")**: Instead of guessing randomly, this method tests a pre-made list of real, commonly-used passwords — because people tend to reuse the same weak passwords over and over.

```bash
$ head -10 /usr/share/wordlists/rockyou.txt
123456
12345
123456789
password
iloveyou
princess
1234567
rockyou
12345678
abc123
```

This is usually the most efficient method in practice, since it skips straight to the passwords people are actually likely to use.

> **Fun fact:** `rockyou.txt` is a famous wordlist of over 14 million real passwords leaked in a 2009 data breach of the RockYou website — the company had been storing user passwords with no encryption at all.
> Popular wordlists include `rockyou.txt`, `SecLists` collections and `Assetnote`.

---

## Quick Comparison

| Method | Speed | Success Rate | Best For |
|--------|-------|---------------|----------|
| Rainbow Table | ⚡ Very fast | Only works on known/un-salted hashes | Cracking leaked, unsalted hash dumps |
| Brute Force | 🐢 Slow (scales badly) | 100% eventually | Short/simple passwords |
| Dictionary | 🚀 Fast & practical | Depends on list quality | Everyday real-world cracking |

**Bottom line:** hashing alone isn't enough to protect passwords — salting is what makes rainbow tables useless, and using long, unique, non-dictionary passwords is what makes brute-force and dictionary attacks impractical.
