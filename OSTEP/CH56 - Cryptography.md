# CH56 - Protecting Information With Cryptography

## Big Picture
- The OS controls only **its own hardware**. Data on other machines, removed disks, or in transit is outside its control.
- Idea: make data **unintelligible** to attackers → protects **secrecy**, and (used right) **integrity** and **authentication**.
- Cryptography is powerful but **expensive** and **easy to misuse** → use selectively, with vetted tools.

---

## 1. Basics
```
C = E(P, K)      // encryption: plaintext P, key K → ciphertext C
P = D(C, K)      // decryption
```
- Requirements: deterministic; recovering P from C **without K** must be computationally infeasible.
- Example of a weak cipher: **Caesar cipher** (shift letters).
- **Ciphers are public; security depends entirely on keeping the key secret** (open design).

> ⚠️ **Don't develop your own ciphers** — experts' ciphers get broken too. Use standards.

## 2. Symmetric Cryptography
- Same key encrypts and decrypts. Recommended: **AES**.
- Tampered ciphertext decrypts to garbage; add a hash of the plaintext to detect tampering.
- Implies **authentication**: if only you and Remzi know K, a message decrypting correctly came from Remzi.
- Problem: **key distribution** to millions of parties.

## 3. Public Key (Asymmetric) Cryptography
- Key pair: **private key** (secret) + **public key** (widely shared).
| Goal | How |
|---|---|
| **Authentication / signing** | Encrypt with **private** key → anyone verifies with public key (e.g., software updates) |
| **Secrecy** | Encrypt with recipient's **public** key → only the private key holder decrypts |
| Both | Alice signs with her private key, then encrypts with Bob's public key |
- ❌ **Hundreds of times slower** than symmetric → use sparingly.
- ❌ Key distribution still hard: must be sure a public key really belongs to Alice (→ certificates, CH57).

## 4. Cryptographic Hashes
```
S = H(P)     // no key
```
- Properties: infeasible to find **collisions**; any input change → unpredictable output change; can't infer input from hash.
- Integrity: hash the data, **encrypt the hash** (else attacker recomputes it), send both.
- Other uses: **salted password** storage, file change detection (**Tripwire**), **proof of work** (spam, blockchains).
- Recommended: **SHA-3**. SHA-1 broken in 2017.

## 5. Cracking Cryptography
- Standard algorithms have no known practical attacks → systems fail via **software bugs** or **key mismanagement**:
  - Keys shared in widely distributed software; sent in plaintext; chosen from a small space.
  - **Heartbleed**: OpenSSL bug leaked keys from server memory (a quarter to half of HTTPS sites).
- **Brute force**: AES ≥ 128-bit keys → ~2^127 guesses on average. Restrict choices to 32 → trivial.
- Examples: **WEP** keys recoverable in about a minute (→ WPA, WPA2, WPA3); early **Netscape** seeded keys from time + PID → guessable in ~30 s; ~4.5 million embedded devices sharing the same private key.

> 💡 **Choose keys randomly** (entropy gathering, Linux `/dev/random`). **Perfect forward secrecy**: compromising one key doesn't reveal others. Public keys are 2K–4K bits since they can't be fully random (mathematically related pair).

## 6. Cryptography and the OS
- If the key is on the same machine, the OS can use it:
  - Don't trust the OS → crypto can't help (OS copies the key).
  - Trust the OS → no need to hide data from it.
- Still useful with partial/no trust: **hashed passwords**, **data in transit**, **at-rest** data accessed outside the OS. Keep keys off the machine and decrypt only briefly.

## 7. At-Rest Data Encryption
- Granularity: records, blocks, files, whole file systems; done in apps, libraries, FS, or drivers.
- **Full Disk Encryption (FDE)**: BitLocker, FileVault; passphrase at boot; encrypt on write, decrypt on read; overhead ~few %, hardware FDE at full speed.

| FDE protects against | FDE does NOT protect against |
|---|---|
| Stolen / resold drive (another OS can't read raw blocks) | Legit users with bad permissions, app bugs (buffer overflow, SQL injection), dishonest admins, OS flaws |
- Other uses: archives, encrypt before cloud upload, encrypted email, **password vaults** (key kept in RAM briefly — usability vs security, like single sign-on).

## 8. Cryptographic Capabilities
- Make capabilities **unforgeable** so users can hold them: trusted entity creates an encrypted/signed structure listing allowed access; resource manager verifies without an ACL.
- Symmetric (creator and checker share a key) or public key (sign with private key, verify with public).
- Can carry expiration times, identity, be bound to an IP address. Must be long enough that guessing is infeasible.

## 9. Summary
| Category | Keys |
|---|---|
| Symmetric ciphers | One shared secret key |
| Asymmetric ciphers | Public/private pair |
| Cryptographic hashes | No key, not reversible |
- Store keys in as **few places**, for as **short a time**, available to as **few parties** as possible.
