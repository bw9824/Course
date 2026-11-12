# CH57 - Distributed System Security

## Big Picture
- The OS controls only its own machine. Two core problems:
  1. **Remote partners** may not enforce our policies, or may be impersonators.
  2. **The network** is controlled by no one we trust → messages can be forged, copied, **replayed**, altered, destroyed, delayed.
- Main tool: **cryptography** — via vetted standard protocols (TLS, SSH), not home-made ones.

---

## 1. Role of Authentication
- We can't verify remote partners follow our policies → we **trust** them, backed by **detection and compensation**.
- Network authentication is based on what you know:
  - **Password** → good for **many clients → one server**.
  - **Private key** → good for **one server → many clients**.
- Passwords must be encrypted in transit.

> ⚠️ Secure networking doesn't protect against a **compromised partner** — an authenticated, encrypted attack is still an attack.

## 2. Public Key Authentication: Certificates
- Public key doesn't need secrecy, but you must know **whose** it is.
- **Certificate**: identity + public key + expiration + ..., **signed** by a trusted third party:
  - Signer hashes contents, encrypts the hash with its **private key**.
  - Verifier recomputes the hash and compares to the signature decrypted with the signer's **public key**.
- **Certificate Authority (CA)**: verifies identity before signing (e.g., Frobazz Inc. buys a cert from AcmeSign).
  - CA not involved in verification; verify once, then store the key.
- **Bootstrapping**: browsers ship with several hundred trusted CA keys. Standard: **X.509** (PGP is another tool).
- **Replay attacks**: defeated by unique info (nonces, sequence numbers) in each message.
- Public key ops are expensive → only to bootstrap, then switch to **symmetric session keys**.
- A cert only shows who owns the key; proving the sender **holds** the private key needs a separate challenge.

## 3. Password Authentication
- Send user ID + password encrypted (server's public key / TLS session key).
- Only authenticates **one message** → later messages need to be tied to it → TLS session.

## 4. SSL/TLS
- Crypto layer between application and socket, on both ends.
- **SSL is insecure — use TLS (1.3)**.
- Handshake:
  1. Negotiate cipher suite (asymmetric alg, key exchange, hash, bulk cipher) — e.g. RSA-2048 + Diffie-Hellman + SHA-3 + AES-256.
  2. Server presents its **certificate** (client auth optional).
  3. **Diffie-Hellman** creates a fresh **symmetric key**, signed with the server's private key.
  4. Bulk data encrypted with the symmetric key.
- New symmetric key **per connection**; asymmetric for authentication, symmetric for bulk data.
- Typically only the **server** is authenticated at the TLS level; client logs in later with a password over the encrypted channel.

### Diffie-Hellman Key Exchange
```
public: large prime n, primitive root g
Alice: secret x, sends X = g^x mod n
Bob:   secret y, sends Y = g^y mod n
key  = Y^x mod n = X^y mod n = g^(xy) mod n
```
- Eavesdropper must solve the **discrete logarithm** (infeasible for large n, ~600 digits).
- ❌ No authentication → **man-in-the-middle** (Mallory shares a key with each side). Fix: sign the exchange (certificates).

> 💡 On a TLS vulnerability, ask: **protocol** flaw (all implementations) or **implementation** flaw (e.g., Heartbleed — OpenSSL only)?

## 5. Other Authentication Approaches
| Approach | Idea |
|---|---|
| **Web cookies** | After login, server issues a cookie (a cryptographic capability only it can create); browser sends it automatically. Convenience vs misuse on a shared browser |
| **Challenge/response** | Server sends a fresh random challenge; client returns f(challenge, shared secret). Recorded responses are useless |
| **Authentication servers** | Trusted third party vouches online (**Kerberos**); can revoke immediately (CAs are the offline version) |

## 6. Higher-Level Tools
### HTTPS
- HTTP over TLS; browser checks server certificates against built-in CA keys.
- A valid cert ≠ good site, uncompromised key, or uncompromised CA.
- Downgrade to plain HTTP risk → **HSTS** (HTTP Strict Transport Security): insist on HTTPS.

### SSH
- Secure remote shell + general tool (X forwarding, TCP port forwarding).
- Server authenticated by public key; client by password, keys, or Kerberos.
- First connection: no known server key → MITM possible; SSH warns when keys change, but users usually click "yes".

> 💡 Folk wisdom: users ignore security warnings — don't rely on them.

## 7. Summary
- Authenticate and encrypt every message; X.509 certs for public keys, TLS for secure sockets, passwords for humans; symmetric crypto for bulk data; DH key exchange must be authenticated.
- Use proven tools; remote actions remain risky since the OS can't control other machines.
