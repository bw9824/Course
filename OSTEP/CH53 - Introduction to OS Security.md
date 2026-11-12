# CH53 - Introduction to Operating System Security

## Big Picture
- Almost all software runs on an OS → an insecure OS undermines every app above it ("building a house on sand").
- Crux: with many interacting processes, how does the OS ensure each one accesses **exactly** the resources it should, in **exactly** the permitted ways?
- Built from **goals** → precise **policies** → OS **mechanisms** that enforce them.

---

## 1. Why OS Security Is Hard
- Modern OSes are **huge and complex** → more code = more flaws; most security failures come from flaws.
- Must isolate many processes; **not all code deserves equal trust** (a script from an unknown website shouldn't wipe your disk).
- Apps rely on the security behavior of OS abstractions (file system, VM, scheduling).

## 2. What Are We Protecting?
- **Everything** — the OS controls nearly all hardware. A compromised OS can:
  - Read/alter any process's memory; read/write/delete any file.
  - Change scheduling, halt processes, send altered messages, enable/disable devices.
  - Give any process another's resources, revoke resources, **lie in system call results**.
- Processes can't protect themselves from a malicious OS; a flaw that lets a process manipulate the OS is almost as bad → **patch OS security flaws**.

> 💡 **Security enclaves** (TPM, newer hardware enclaves): hardware isolating some functions/keys even from the OS; useful in clouds. Less isolation than hoped, but still a barrier.

## 3. Security Goals and Policies
| Goal | Meaning | OS example |
|---|---|---|
| **Confidentiality** | Secrets stay hidden from unauthorized parties | A process can't read another's memory |
| **Integrity** | No unauthorized changes (incl. **authenticity**) | A user can't modify another user's file without permission |
| **Availability** | Authorized users can always use the service | One process can't monopolize the CPU |
- Also: **controlled sharing**; **non-repudiation** (can't deny an action later).
- **Policies**: precise, machine-enforceable rules ("users A and B may write file X; nobody else may").
- OS gives **general mechanisms**; they're only as good as the policy specified.

> 💡 Virtualization protects against **accidents** and **attacks**, but it works best when nobody actively tries to subvert it; legitimate sharing channels (file systems, IPC) can be abused.

## 4. Designing Secure Systems — Saltzer & Schroeder (1975)
| Principle | Meaning |
|---|---|
| **Economy of mechanism** | Keep it small and simple |
| **Fail-safe defaults** | Default to the restrictive/secure option |
| **Complete mediation** | Check **every** access, every time |
| **Open design** | Assume the attacker knows the design; rely only on keys being secret |
| **Separation of privilege** | Critical actions need multiple parties/credentials (e.g. 2FA) |
| **Least privilege** | Give only the privileges the job needs |
| **Least common mechanism** | Separate data structures per user/process (e.g. per-process page tables) |
| **Acceptability** | Users must be willing to use it, or they work around it |

- Real systems often skip complete mediation for performance → engineering trade-offs.

## 5. Basics of OS Security
- Core principle: keep processes **cleanly separated**; crossing the boundary requires OS help.
- Memory: hardware + page tables → a process can't even name memory outside its mappings.
- **System calls** are the protection point (user → kernel mode):
  1. Identify the calling process (PID / PCB → owner).
  2. **Access control** decides if it's authorized.
  3. Yes → OS performs it (or lets the process do it directly).
  4. No → return an error code.
- Devices go through drivers, reached via system calls → same mechanism protects hardware.

> 💡 **Weakest link**: attackers are smart and lazy — guessing "password" beats finding a buffer overflow. Humans are often the weakest link; least privilege limits damage when users are tricked.
