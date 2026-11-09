# Credential Hunting in Network Traffic

## Table of Contents

- [Why network traffic is still worth checking](#why-network-traffic-is-still-worth-checking)
- [Wireshark — manually inspecting traffic](#wireshark--manually-inspecting-traffic)
- [Pcredz — automated credential extraction from traffic](#pcredz--automated-credential-extraction-from-traffic)

---

## Why network traffic is still worth checking

Most modern applications use **TLS** (the "S" in HTTPS) to encrypt data in transit, which rightly makes eavesdropping much harder than it used to be. But not everything gets this treatment — legacy systems, misconfigured services, and quick test deployments spun up without HTTPS are more common than you'd hope. Whenever **plaintext** protocols are in use, anything sent over them — including passwords — is potentially visible to anyone who can see the traffic.

### Common plaintext protocols and their secure replacements

| Insecure Protocol | Secure Version | What it's used for |
|---------------------|--------------------|------------------------|
| HTTP | HTTPS | Web pages/resources |
| FTP | FTPS / SFTP | File transfer |
| SNMP | SNMPv3 (encrypted) | Monitoring network devices |
| POP3 | POP3S | Retrieving email |
| IMAP | IMAPS | Managing email on the server |
| SMTP | SMTPS | Sending email |
| LDAP | LDAPS | Directory service queries (user credentials, roles) |
| RDP | RDP with TLS | Windows remote desktop |
| DNS | DNS over HTTPS (DoH) | Domain name resolution |
| SMB | SMB 3.0 with encryption | File/printer sharing |
| VNC | VNC with TLS/SSL | Remote graphical control |

If any of these show up in their **plaintext** form on a network you're assessing, it's worth taking a closer look — it may mean credentials are traveling in the clear.

---

## Wireshark — manually inspecting traffic

**Wireshark** is the standard tool for capturing and analyzing network traffic, and it comes pre-installed on most penetration testing Linux distros. Its real power is the filter engine, which lets you zero in on exactly the traffic you care about instead of scrolling through everything.

### Useful filters for credential hunting

| Filter | What it does |
|--------|-----------------|
| `ip.addr == 56.48.210.13` | Show packets involving a specific IP |
| `tcp.port == 80` | Show traffic on a specific port (HTTP here) |
| `http` | Show all HTTP traffic |
| `dns` | Show DNS lookups |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | Show TCP handshake attempts (SYN packets) — useful for spotting scanning activity |
| `icmp` | Show ping traffic |
| `http.request.method == "POST"` | Show HTTP POST requests — **this is a big one**, since login forms submitted over plain HTTP send their data via POST, meaning usernames/passwords may be sitting right there in plaintext |
| `tcp.stream eq 53` | Follow one specific conversation between two hosts |
| `eth.addr == 00:11:22:33:44:55` | Filter by MAC address |
| `ip.src == 192.168.24.3 && ip.dst == 56.48.210.3` | Show traffic specifically between two hosts |

The `http.request.method == "POST"` filter in particular is worth remembering — if you find a login form submitting over plain HTTP, right-clicking that packet and choosing "Follow → HTTP Stream" in Wireshark will often show you the submitted username and password fields in plain text.

- In Wireshark, it's possible to locate packets that contain specific bytes or strings. One way to do this is by using a display filter such as `http contains "passw"`. 
- Alternatively, you can navigate to `Edit > Find Packet` and enter the desired search query manually. For example, you might search for packets containing the string `"passw"`:

<img width="1028" height="657" alt="image" src="https://github.com/user-attachments/assets/f3f7b4d4-4553-46e7-a3ee-dff40c2ebf02" />

---

## Pcredz — automated credential extraction from traffic

Manually scrolling through packets works, but it's slow. [Pcredz](https://github.com/lgandx/PCredz) automates the whole process — point it at a packet capture (or a live interface) and it automatically pulls out anything resembling credentials, across a wide range of protocols.

### What it can extract

- Credit card numbers
- POP3 / SMTP / IMAP login credentials
- SNMP community strings
- FTP credentials
- HTTP NTLM/Basic auth headers and HTTP form submissions
- NTLMv1/v2 hashes (from SMB, LDAP, MSSQL, DCE-RPC, HTTP, and more)
- Kerberos pre-authentication hashes (AS-REQ)

### Installing it

You can either clone the repo and install dependencies manually, or use the Docker container Pcredz provides — both are documented in its README.

### Running it against a capture file

```bash
$ ./Pcredz -f demo.pcapng -t -v
```

```
CC number scanning activated

[1746131482.601354] protocol: udp 192.168.31.211:59022 > 192.168.31.238:161
Found SNMPv2 Community string: s3cr...SNIP...

[1746131482.658938] protocol: tcp 192.168.31.243:55707 > 192.168.31.211:21
FTP User: le...SNIP...
FTP Pass: qw...SNIP...

demo.pcapng parsed in: 1.82 seconds (File size 15.5 Mo).
```

In this run, Pcredz pulled both an **SNMP community string** (essentially a password used to monitor/manage network devices) and a full **FTP username/password pair** — all just from scanning through the capture file automatically, with zero manual packet inspection needed.

---

## Quick Recap

| Tool | Best for |
|------|-------------|
| Wireshark | Manual, targeted inspection — great for confirming a specific suspicious flow (e.g., a login POST request) |
| Pcredz | Fast, automated sweep of an entire capture for credentials across many protocols at once |

**Key idea:** Encryption adoption isn't universal — legacy protocols, forgotten test services, and misconfigured apps still regularly transmit credentials in plaintext. Capturing or reviewing network traffic (live or from a `.pcap` file) and either manually filtering with Wireshark or automatically scanning with Pcredz can surface real, usable credentials with very little effort whenever these gaps exist.
