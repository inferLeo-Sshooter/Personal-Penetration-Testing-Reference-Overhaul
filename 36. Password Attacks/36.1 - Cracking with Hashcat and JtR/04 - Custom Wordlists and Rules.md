# Custom Wordlists and Rules

## Table of Contents

- [1. Why bother building your own wordlist?](#1-why-bother-building-your-own-wordlist)
- [2. The predictable patterns people fall into](#2-the-predictable-patterns-people-fall-into)
- [3. Turning one guess into many: Hashcat rule files](#3-turning-one-guess-into-many-hashcat-rule-files)
- [4. CeWL](#4-cewl)
- [5. Username Anarchy](#5-username-anarchy)
- [6. CUPP](#6-cupp)


---

## 1. Why bother building your own wordlist?

Standard wordlists (like `rockyou.txt`) work great against simple, lazy passwords. But once a system enforces a **password policy** — minimum length, uppercase letters, numbers, special characters — those generic lists stop working as well.

Here's the thing though: enforcing complexity rules doesn't actually stop people from being predictable. It just changes *how* they're predictable. Most people don't invent a truly random password — they take something easy to remember and bolt on the minimum required extras.

---

## 2. The predictable patterns people fall into

People tend to base passwords on things connected to their life or workplace: the company name, their pet, their favorite sports team, a hobby, etc. A bit of basic **OSINT** (publicly-gathered info about a person or company) can reveal a lot of these personal details, making guesses far more targeted than blind brute-forcing.

On top of that, when a policy forces extra complexity, people usually bolt it on using the same few tricks:

| What they do | Example |
|---------------|---------|
| Capitalize the first letter | `Password` |
| Add numbers | `Password123` |
| Add the current year | `Password2022` |
| Add the current month | `Password02` |
| End with an exclamation mark | `Password2022!` |
| Swap letters for symbols | `P@ssw0rd2022!` |

Statistically, most real-world passwords stay under 10 characters. So a common pattern looks like: *one familiar word + a year/month + one special character* — just long enough to meet the policy, just simple enough to remember. And since companies often force periodic password changes, people frequently just tweak one small piece each time (swap the month, bump a digit) rather than picking something new.

---

## 3. Turning one guess into many: Hashcat rule files

Rather than typing out every possible variation of a word by hand, we can write a **rule file** that tells Hashcat how to automatically mutate a base word in different ways.

### Basic rule syntax

| Symbol | What it does |
|--------|----------------|
| `:` | leave the word unchanged |
| `l` | make everything lowercase |
| `u` | make everything UPPERCASE |
| `c` | capitalize just the first letter |
| `sXY` | replace every `X` with `Y` (e.g. `so0` replaces `o` with `0`) |
| `$!` | add `!` to the end |

Each line in the rule file is one transformation recipe. You can also **combine** multiple functions on a single line to stack effects.

[official Hashcat rule-based attack documentation](https://hashcat.net/wiki/doku.php?id=rule_based_attack)

### Example rule file

```bash
$ cat custom.rule
:
c
so0
c so0
sa@
c sa@
c sa@ so0
$!
$! c
$! so0
$! sa@
$! c so0
$! c sa@
$! so0 sa@
$! c so0 sa@
```

### Applying it to a word

Starting with a tiny input list containing just one word:

```bash
$ cat password.list
password
```

Run Hashcat in "rule preview" mode (`--stdout` just prints the mutated results instead of cracking anything) to apply every rule in `custom.rule` to it:

```bash
$ hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

One input word → 15 mutated variants:

```
password
Password
passw0rd
Passw0rd
p@ssword
P@ssword
P@ssw0rd
password!
Password!
passw0rd!
p@ssword!
Passw0rd!
P@ssword!
p@ssw0rd!
P@ssw0rd!
```

This is effectively how real attackers expand a short, obvious guess into a whole family of realistic password candidates.

> **Tip:** Both Hashcat and John the Ripper ship with ready-made rule files — `best64.rule` is one of the most popular, since it bundles the transformations that statistically crack the most passwords in practice.

### The bigger picture

Password cracking (and wordlist-building) is fundamentally **educated guessing**. You improve your odds by narrowing the guesswork using context: the target's industry, company name, region, known password policy, or any other detail that hints at what words they might lean on. (The obvious exception: if a plaintext password list has already leaked somewhere, there's no guessing involved at all.)

---

## 4. CeWL

**CeWL** is a tool that scans a website's text and pulls out all the words it finds — building a custom wordlist made of terms that are actually relevant to that specific company or organization (product names, jargon, staff names, etc.) rather than generic dictionary words.

```bash
$ cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
```

What each flag does:

| Flag | Meaning |
|------|---------|
| `-d 4` | crawl up to 4 links deep from the starting page |
| `-m 6` | only keep words that are at least 6 characters long |
| `--lowercase` | store every word in lowercase |
| `-w inlane.wordlist` | save the output to this file |

Check how many words it found:

```bash
$ wc -l inlane.wordlist
326
```

From here, this company-specific wordlist can be combined with mutation rules (like the `custom.rule` example above) to produce a wordlist that's much more likely to contain an actual employee's password than a generic public list would.

---

## 5. Username Anarchy

Manual username generation for names like "Jane Smith" quickly becomes complex. Basic combinations like `jane`, `janesmith`, or `j.smith` only scratch the surface of potential variations.

Users often incorporate personal elements creating variations like `janemarie`, `smithj87`, `jane_the_gardener`, leetspeak versions (`j4n3`, `5m1th`), or interest-based usernames (`potterheadjane`, `smith_beatles_fan`).

`Username Anarchy` automates this process, generating comprehensive username lists using various patterns and substitutions:

```bash
$ ./username-anarchy -l
```
```
Plugin name             Example
--------------------------------------------------------------------------------
first                   anna
firstlast               annakey
first.last              anna.key
firstlast[8]            annakey
first[4]last[4]         annakey
firstl                  annak
f.last                  a.key
flast                   akey
lfirst                  kanna
l.first                 k.anna
lastf                   keya
last                    key
last.f                  key.a
last.first              key.anna
FLast                   AKey
first1                  anna0,anna1,anna2
fl                      ak
fmlast                  abkey
firstmiddlelast         annaboomkey
fml                     abk
FL                      AK
FirstLast               AnnaKey
First.Last              Anna.Key
Last                    Key
```

Running it against a real target name generates a full list:
```bash
$ ./username-anarchy Jane Smith > jane_smith_usernames.txt
```

Upon inspecting `jane_smith_usernames.txt`, you'll encounter a diverse array of usernames, encompassing:
- Basic combinations: `janesmith`, `smithjane`, `jane.smith`, `j.smith`, etc.
- Initials: `js, j.s, or s.j`, etc.
- and more.

This comprehensive list, tailored to the target's name, is valuable in a brute-force attack.

| Command | Description |
|---------|-------------|
| `username-anarchy Jane Smith` | Generate possible usernames for "Jane Smith" |
| `username-anarchy -i names.txt` | Use a file (names.txt) with names for input. Can handle space, CSV, or TAB delimited names |
| `username-anarchy -a --country us` | Automatically generate usernames using common names from the US dataset |
| `username-anarchy -l` | List available username format plugins |
| `username-anarchy -f format1,format2` | Use specific format plugins for username generation (comma-separated) |
| `username-anarchy -@ example.com` | Append @example.com as a suffix to each username |
| `username-anarchy --case-insensitive` | Generate usernames in case-insensitive (lowercase) format |

---

## 6. CUPP

`CUPP` (Common User Passwords Profiler) creates personalized password wordlists using gathered intelligence about targets. After generating usernames with Username Anarchy, CUPP complements this with targeted password lists.

CUPP's effectiveness depends on the quality of information provided. Intelligence sources include:
- **Social Media**: Birthdays, pet names, interests, relationships (Facebook, Twitter, Instagram, LinkedIn)
- **Company Websites**: Professional details and bios
- **Public Records**: Address, family members, property ownership
- **News Articles/Blogs**: Interests, achievements, affiliations

### Example: profiling "Jane Smith"

Say OSINT research turned up this profile:

| Detail |	Value|
|-|-|
|Name / Nickname|	Jane Smith / "Janey"|
|Birthdate|	December 11, 1990|
|Partner|	Jim ("Jimbo"), born Dec 12, 1990|
|Pet|	Spot|
|Company|	AHI|
|Interests|	Hackers, Pizza, Golf, Horses|
|Favorite color|	Blue|

`CUPP` mixes and mutates all of this into a big batch of password candidates, such as:
- Original/Capitalized: `jane`, `Jane`
- Reversed: `enaj`, `enaJ`
- Birthdate variations: `jane1994`, `smith2708`
- Concatenations: `janesmith`, `smithjane`
- Special characters: `jane!`, `smith@`
- Numbers: `jane123`, `smith2024`
- Leetspeak: `j4n3`, `5m1th`
- Combined mutations: `Jane1994!`, `smith2708@`

This personalized approach significantly increases password-cracking success rates compared to generic dictionaries.

Invoke `CUPP` in interactive mode, `CUPP` will guide you through a series of questions about your target, enter the following as prompted:

```bash
$ cupp -i
```

| Command | Description |
|---------|-------------|
| `cupp -i` | Generate wordlist based on personal information (interactive mode) |
| `cupp -w profiles.txt` | Generate a wordlist from a predefined profile file |
| `cupp -l` | Download popular password lists like rockyou.txt |

### Filtering results to match a password policy

We now have a generated `username.txt` list and `jane.txt` password list, but real systems usually enforce rules about what counts as a valid password. Say Jane's company (AHI) requires:
- Minimum Length: 6 characters
- Must Include:
    + At least one uppercase letter
    + At least one lowercase letter
    + At least one number
    + At least two special characters (from the set `!@#$%^&*`)

Instead of manually checking thousands of entries, we can chain `grep` filters to keep only the candidates that actually satisfy every rule:

```bash
$ grep -E '^.{6,}$' jane.txt \
  | grep -E '[A-Z]' \
  | grep -E '[a-z]' \
  | grep -E '[0-9]' \
  | grep -E '([!@#$%^&*].*){2,}' > jane-filtered.txt
```

What each piece checks:

|**Filter**	|**Requirement enforced**|
|-|-|
|`^.{6,}$`	|at least 6 characters long|
|`[A-Z]`	|contains an uppercase letter|
|`[a-z]`	|contains a lowercase letter|
|`[0-9]`	|contains a digit|
|`([!@#$%^&*].*){2,}`	|contains at least 2 special characters from the set|

This trims the list dramatically — in this example, from roughly 46,000 candidate passwords down to about 7,900 that actually meet the policy, all saved to `jane-filtered.txt`. The result is a much smaller, much higher-quality list to test, instead of wasting time on candidates that could never have been valid passwords in the first place.

| Policy Requirement | Grep Regex Pattern | Explanation |
|--------------------|-------------------|-------------|
| **Minimum Length** (e.g., 8 characters) | `grep -E '^.{8,}$' wordlist.txt` | `^` matches the start of the line, `.` matches any character, `{8,}` matches 8 or more occurrences, `$` matches the end of the line |
| **At Least One Uppercase Letter** | `grep -E '[A-Z]' wordlist.txt` | `[A-Z]` matches any uppercase letter |
| **At Least One Lowercase Letter** | `grep -E '[a-z]' wordlist.txt` | `[a-z]` matches any lowercase letter |
| **At Least One Digit** | `grep -E '[0-9]' wordlist.txt` | `[0-9]` matches any digit |
| **At Least One Special Character** | `grep -E '[!@#$%^&*()_+-=[]{};'"'"':"\,.<>/?]' wordlist.txt` | `[!@#$%^&*()_+-=[]{};':"\,.<>/?]` matches any special character (symbol) |
| **No Consecutive Repeated Characters** | `grep -E '(.)\1' wordlist.txt` | `(.)` captures any character, `\1` matches the previously captured character. This pattern will match any line with consecutive repeated characters. Use `grep -v` to invert the match |
| **Exclude Common Patterns** (e.g., "password") | `grep -v -i 'password' wordlist.txt` | `-v` inverts the match, `-i` makes the search case-insensitive. This pattern will exclude any line containing "password" (or "Password", "PASSWORD", etc.) |
| **Exclude Dictionary Words** | `grep -v -f dictionary.txt wordlist.txt` | `-f` reads patterns from a file. dictionary.txt should contain a list of common dictionary words, one per line |
| **Combination of Requirements** | `grep -E '^.{8,}$' wordlist.txt \| grep -E '[A-Z]'` | This command filters a wordlist to meet multiple password policy requirements. It first ensures that each word has a minimum length of 8 characters, and then pipes the result into a second grep command to match only words that contain at least one uppercase letter |

---

## Quick Recap

| Tool/Technique | Purpose |
|------------------|---------|
| Common password patterns table | Predict how users likely satisfy policy requirements |
| Hashcat rule files | Auto-generate many password variants from one base word |
| `best64.rule` | Pre-built ruleset of statistically effective transformations |
| CeWL | Scrape a target website to build a context-relevant wordlist |
| Username Anarchy | Generate likely username formats |
| CUPP | Generate a personalized password candidates from OSINT details about the target |
| `grep` filter chain | A trimmed-down, policy-compliant password list |

**Key idea:** Complexity *requirements* don't equal complexity in *practice* — people satisfy policies in predictable ways, and combining a relevant wordlist (ideally built from real context about the target) with smart mutation rules is far more effective than guessing blindly.
