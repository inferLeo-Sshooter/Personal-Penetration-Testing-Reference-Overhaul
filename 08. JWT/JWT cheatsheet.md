# JWT Attacks Cheatsheet

Quick-reference only — assumes you already know the fundamentals (header.payload.signature structure, stateless vs stateful auth, why signature trust = everything).

---

## 1. Signature Verification Bypasses

| Attack | How | Tell |
|---|---|---|
| **Arbitrary signature accepted** | Server calls `decode()` instead of `verify()` (e.g. Node `jsonwebtoken`) — signature never actually checked | Edit payload (e.g. `sub: administrator`), re-send unsigned/mismatched sig — if accepted, confirmed |
| **`alg: none` accepted** | Set header `alg` to `none`, strip signature, keep trailing dot | Try case variants to bypass blacklist filters: `none`, `None`, `NONE`, `nOnE` |

```
# unsigned token shape (note trailing dot, no third segment)
eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9.
```

---

## 2. Brute-Forcing the Secret (HS256/384/512 only)

Symmetric algs → same key signs and verifies → guessable/default secrets = forgeable tokens.

```bash
hashcat -a 0 -m 16500 <jwt> <wordlist>
```
Use [wallarm/jwt-secrets](https://github.com/wallarm/jwt-secrets) wordlist. Once cracked, resign any payload with the secret.

---

## 3. Header Parameter Injection

All three share the same root cause: **server trusts a self-declared/attacker-influenced key source instead of a pre-approved allowlist.**

### `jwk` — embed the key directly
Attacker generates own RSA keypair → signs forged payload with private key → embeds matching public key in `jwk` header.
- Burp JWT Editor: generate RSA key → edit payload → **Attack → Embedded JWK** (auto-signs + embeds).
- Sometimes must also match `kid` header to the embedded `jwk`'s `kid`.

### `jku` — point to a hosted key set
Same idea, but `jku` points to a URL the server fetches for a JWK Set. Host your own key set (with matching `kid`) on an attacker server → server fetches it → trusts it.
```json
{"alg":"RS256","kid":"...","jku":"https://attacker.com/jwks.json"}
```
Domain-restriction bypass: same category as SSRF filter bypass (`trusted-server.com.evil.com`, parser/redirect/DNS tricks).

### `kid` — key lookup injection
`kid` value is used to *look up* a key (file, DB) rather than supply one directly — the lookup mechanism itself is the vuln.
- **Directory traversal to a file:** `"kid": "../../../../dev/null"` → server reads empty string as key → sign your HS256 token with secret `""` → matches.
- **SQL injection:** if `kid` is used unsanitized in a query (`SELECT key FROM keys WHERE kid='<kid>'`) → standard SQLi.

### Other header fields (secondary attack surface — need a signature bypass first)
- `cty` (Content Type) — set to `text/xml` or `application/x-java-serialized-object` to make the parser reinterpret the payload → potential XXE or insecure deserialization.
- `x5c` (X.509 cert chain) — same trust issue as `jwk` but via certificate format; also a historically buggy parser (see CVE-2017-2800, CVE-2018-2633).

---

## 4. Algorithm Confusion (RS256 → HS256)

**Idea:** trick a server expecting asymmetric `RS256` into verifying with symmetric `HS256`, using the server's own (public, non-secret) RSA public key as the HMAC secret.

**Root cause:** generic `verify(token, key)` library methods pick behavior based on the attacker-controlled `alg` header instead of being pinned to the algorithm the developer intended.

### Steps
1. **Get the public key** — check `/jwks.json` or `/.well-known/jwks.json`. If unavailable, derive it mathematically (below).
2. **Convert to exact byte-matching format** — server's internal copy is likely PEM/X.509; your signing key must match byte-for-byte (including newlines). In Burp JWT Editor: import JWK as RSA key → **Copy Public Key as PEM** → Base64-encode it in Decoder → paste as the `k` value of a new **Symmetric Key**.
3. **Edit payload**, set `alg: HS256`.
4. **Sign with HS256** using that repackaged public-key-as-secret.

### If the public key isn't exposed
Derive it from two valid JWTs signed by the same RSA key (number theory on RSA signatures):
```bash
docker run --rm -it portswigger/sig2n <token1> <token2>
```
Produces multiple candidate keys (X.509 + PKCS1 PEM) + pre-forged JWTs for each — try each candidate against the server (wrong ones just get rejected, no harm) until one is accepted.

---

## 5. Tooling

**jwt_tool** — analysis + automated forging:
```bash
git clone https://github.com/ticarpi/jwt_tool
pip3 install -r requirements.txt
python3 jwt_tool.py <jwt>                                    # decode/analyze
python3 jwt_tool.py -X a -pc isAdmin -pv true -I -t <jwt>     # forge alg:none variants + inject claim
```
`-X a` = alg:none exploit (auto-generates case-variant bypasses). `-pc`/`-pv` = set a payload claim/value. `-I` = sign/inject.

---

## 6. Testing Order (run through in sequence)

1. Decode header/payload — check `alg`, presence of `jwk`/`jku`/`kid`/`cty`/`x5c`.
2. Try `decode()`-only bypass: tamper payload, resend as-is.
3. Try `alg: none` (+ case variants), strip signature, keep trailing dot.
4. If HS256: hashcat against wordlist.
5. If `jwk` present or acceptable: embed own key + re-sign (Burp: Attack → Embedded JWK).
6. If `jku` present or acceptable: host own JWK Set, point `jku` at it.
7. If `kid` present: test directory traversal (`/dev/null` trick for empty-secret HS256) and SQLi.
8. If RS256 and none of the above work: attempt algorithm confusion — fetch/derive public key, reformat, sign with HS256.
9. Cross-check any forged token's claims for privilege escalation potential (role, isAdmin, sub/user id) before declaring success.
