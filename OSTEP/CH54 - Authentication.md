# CH54 - Authentication

## Big Picture
- Most access decisions depend on **who is asking**.
- Crux: securely identify processes, give new processes the right identity, prevent malicious identity changes.
- Users are authenticated by **what you know**, **what you have**, **what you are**.

---

## 1. Terms
| Term | Meaning |
|---|---|
| **Principal** | Security-meaningful entity making requests (user, group, program) |
| **Agent** | The process acting on behalf of a principal |
| **Object** | The requested resource (file, IPC channel...) |
| **Credential** | OS-kept data recording an access decision for reuse (e.g. page tables) |
- OSes usually trust authentication for a **process's whole lifetime** → must be right the first time.

## 2. Attaching Identities to Processes
- **Child inherits parent's identity** (`fork()` copies the identity from parent's PCB).
- **Login bootstrap**: OS authenticates the user, then sets the owner of the first process (shell / window manager); everything after inherits it.

## 3. What You Know: Passwords
- Since **CTSS** (MIT, early 1960s). Security depends on: others not knowing it, not guessing it, user remembering it without writing it down.
- **Store hashes, not passwords**: hash the attempt, compare. Use a well-studied cryptographic hash (SHA-3).
- Guessing: n-bit password → ~2^(n−1) guesses on average (8 bits → 256 possibilities, ~128 guesses).
- **Dictionary attacks**: try likely passwords first ("123456", names, words) — cracked ~90% of a typical site's passwords.
- Online defenses: lock out after failures, or better, **slow down** checks after a few wrong guesses.

### Salting (Morris & Thompson, 1979)
```
stored = (salt, H(salt || password))     // random 32/64-bit salt per user
```
- Precomputed dictionary tables become infeasible: each salt needs its own table (32-bit salt → 2^32 versions per word).

> 💡 **Avoid storing secrets**: only if needed, hashed if possible, in as few places as possible (watch logs, backups, editor temp files). Never embed secrets in executables.

> 💡 **Password vaults**: encrypted store unlocked by one master password; doesn't fix weak passwords.

### Linux Login
1. Privileged login process prompts for username (echoed), password (not echoed).
2. Look up username in the password file → UID, GID, shell, home dir, **salted hash**.
3. Hash salt + typed password, compare.
4. Match → `fork()`, set UID/GID, `chdir` home, `exec` the shell.
- Doesn't say whether username or password was wrong → doesn't leak valid usernames.

## 4. What You Have
- ATM cards, **USB hardware tokens** ("dongles"), **display tokens** showing a code to type in.
- Codes must **change frequently** — a static code turns "have" into "know".
- Weakness: lost/stolen device → combine with a password (**multi-factor**).
- **SMS codes**: academically weak (lost phones, redirected texts), practically OK; NIST (2016 draft) deprecated SMS for 2FA in some cases.

## 5. What You Are: Biometrics
- Fingerprints, faces, eyes, typing patterns. Matching is **approximate** (features + tolerance).
| Error | Meaning |
|---|---|
| **False negative** | Legitimate user rejected |
| **False positive** | Impostor accepted |
- Tuning trades one for the other; **crossover error rate** (where they're equal) = common accuracy measure.
- Phone → favor fewer false negatives; bank vault → fewer false positives.
- Needs special hardware; **remote biometric data can be fabricated** → scan on hardware attached to the verifying machine. (Gummy fingers can fool fingerprint readers.)

## 6. Authenticating Non-Humans
- Web servers, IoT devices → create **system users** (e.g. `webserver`) and run processes under them.
- Ways to start such processes: password for the account; privileged user creates processes for another user; process changes its own owner; temporary identity changes.
- **`sudo`**: trusted admins run commands as another identity using **their own** credentials: `sudo -u webserver apache2`.
- **Groups** are principals too (Linux, Windows) — a second path to resource access (benefit and danger).
- Other factors: **location**, **behavior** (typing rhythm).

## 7. Summary
- Identity chain starts at login; children inherit; secure mechanisms change identity when needed.
- **Multi-factor authentication** = evidence from more than one category.
