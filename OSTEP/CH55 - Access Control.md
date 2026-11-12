# CH55 - Access Control

## Big Picture
- Security = (1) decide whether a request fits the policy (**access control**), (2) perform or block it.
- Running example: process owned by user X calls `open("/var/foo", read+write)` → trap → OS decides.
- Two general approaches: **ACLs** (guest list) and **capabilities** (keys/tickets).

---

## 1. Terms
| Term | Meaning |
|---|---|
| **Subject** | Who requests (user / process) |
| **Object** | What is accessed (file, device) |
| **Access** | Mode (read, write, execute) |
| **Authorization** | Deciding if subject may perform access on object |
| **Reference monitor** | Code making every access decision — must be correct and efficient |
- **Complete mediation** vs efficiency trade-off.
- Cut overhead by giving subjects objects that **inherently belong to them**: own address space (hardware PTE bits enforce R/W/X), virtualized devices (one grant, then direct use). But the OS must still protect the virtualization mechanism itself.

## 2. Access Control Lists (ACLs)
- **One ACL per object**, stored as file metadata (ideally in the **inode** — already read on access).
- `open()`: get process owner from PCB → read file's ACL → check user + requested mode.
- Problem: list length (most files: owner + a few friends; some: everyone).

### UNIX Nine Bits
```
-rwxr-x---    owner | group | other  ×  r w x
```
- Owner and group IDs in the inode; "other" = everyone else (no storage needed).
- Checking = simple bit logic, no search, no extra disk reads. ❌ Can't express complex sharing (Windows has richer ACLs).

| ACL pros | ACL cons |
|---|---|
| Easy to see **who** can access an object | Hard to find **everything** a principal can access |
| Easy to change access (edit one list) | Long lists / placement issues |
| Info lives with the file | Distributed: same username on two machines may not be the same principal |

> 💡 **Name spaces** across machines: ignore (separate spaces, like PIDs), central authority (AFS), or partition the space (Web, IPv4).

## 3. Capabilities
- Process holds a set of **unforgeable** tokens, each granting specific access (Dennis & van Horn; **Hydra** OS).
- Problem: capabilities are just bits → could be **forged**, freely **copied** (hard to **revoke**), passed via IPC.
- Fix: OS keeps capabilities in **protected kernel memory** (list off the PCB); transfers go through the OS. Alternative: **cryptographic capabilities** (long, unguessable) — for distributed systems.

| Capability pros | Capability cons |
|---|---|
| Easy to list what a principal can access | Hard to find everyone who can access one object |
| Cheap checks (in memory) | Forgery prevention is hard |
| Easy to give a child a **subset** of privileges | Less developed management |

### Real Systems: Hybrid (Linux)
- `open()` checks the **ACL once** → OS attaches a capability-like structure (fd with R/W rights) to the PCB.
- Later `read()`/`write()` only check that structure; `close()` removes it.
- Gets cheap capability checks + easy ACL policy management.

## 4. Mandatory vs Discretionary Access Control
| | Who decides |
|---|---|
| **DAC** (discretionary) | The **owner** decides who gets access |
| **MAC** (mandatory) | An **authority** sets rules owners can't override (military classification: Top Secret reader can't share a derived report) |
- Independent of ACLs vs capabilities. MAC can include DAC elements that **further restrict**, never loosen. (e.g., SELinux.)

## 5. Practicalities
- Defaults matter: creation mode bits, **`umask()`** masks out bits for every file a process creates. Users rarely change defaults.

### Android Model
- Each **app** runs under its own generated Linux UID → apps isolated from each other.
- **Permission labels** declared by developers, shown at install (least privilege) — like capabilities, a form of MAC. Weakness: users grant without understanding.

### RBAC (Role-Based Access Control, Ferraiolo & Kuhn 1992)
- Privileges assigned to **roles** (doctor, pharmacist); users assigned to roles.
- Change a role's permissions → everyone in it updated. Users may hold several roles but **activate one at a time** (manager switching to programmer role can't read reviews while in it); switching may need re-authentication.
- **Type enforcement**: finer rules attached to objects via a **security context**.
- Minimal RBAC on UNIX: ACLs + groups.

### Privilege Escalation
- **setuid** programs run with the **owner's** privileges (often root) during execution — must be carefully written; many security bugs come from them.
- **`sudo`**: designated users run specific commands as another identity after re-authenticating; secure only if configured carefully.

> ⚠️ **Privilege escalation is dangerous**: attackers who compromise a low-privilege process look for a path up — ultimate goal is **root**. Treat any path to root with great caution.

## 6. Summary
- A perfect mechanism only enforces the policy it's given; real failures usually come from **bad policies**, not faulty mechanisms.
