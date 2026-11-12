# CH05 - Interlude: Process API

## Big Picture
- UNIX creates processes with a curious pair: **`fork()`** + **`exec()`**, plus **`wait()`** to wait for a child.
- Splitting create and run is what makes the **shell** (redirection, pipes) so easy to build.

---

## 1. fork()
```c
printf("hello (pid:%d)\n", (int) getpid());
int rc = fork();
if (rc < 0)       { fprintf(stderr, "fork failed\n"); exit(1); }
else if (rc == 0) { printf("child (pid:%d)\n", (int) getpid()); }
else              { printf("parent of %d (pid:%d)\n", rc, (int) getpid()); }
```
- Child = **almost exact copy** of the parent: own address space, registers, PC.
- Child starts as if **it** just returned from `fork()` (does not run from `main`).
- Return value: **parent gets the child's PID**, **child gets 0**, failure → negative.
- Output order is **non-deterministic** — depends on the CPU scheduler.

## 2. wait()
```c
} else {
    int rc_wait = wait(NULL);   // block until a child exits
    printf("parent of %d (rc_wait:%d)\n", rc, rc_wait);
}
```
- Parent waits for the child → output becomes **deterministic** (child always prints first).
- `waitpid()` waits for a specific child.

## 3. exec()
```c
char *myargs[3];
myargs[0] = strdup("wc");       // program
myargs[1] = strdup("p3.c");     // argument
myargs[2] = NULL;               // end of array
execvp(myargs[0], myargs);      // runs word count
printf("this shouldn't print out");
```
- Loads code + static data from the executable, **overwrites** the current code segment, re-initializes heap/stack, runs it.
- Does **not** create a new process — **transforms** the current one.
- Successful `exec()` **never returns**.
- Variants on Linux: `execl`, `execlp`, `execle`, `execv`, `execvp`, `execvpe`.

## 4. Why Separate fork and exec?
- Shell loop: read command → `fork()` → child `exec()`s the command → parent `wait()`s → prompt again.
- Between fork and exec the child can **alter its environment** (fds, etc.).

### Redirection: `wc p4.c > newfile.txt`
```c
rc = fork();
if (rc == 0) {
    close(STDOUT_FILENO);
    open("./p4.output", O_CREAT | O_WRONLY | O_TRUNC, S_IRWXU);  // gets fd 1
    execvp(myargs[0], myargs);
}
```
- UNIX picks the **lowest free fd** → new file becomes stdout.
- **Pipes** (`grep -o foo file | wc -l`): `pipe()` connects one process's output to another's input.

> 💡 "Lampson's Law": get it right — neither abstraction nor simplicity substitutes for getting it right. fork/exec is a simple yet powerful right design.

## 5. Process Control and Users
- **`kill()`** sends **signals**: SIGINT (Ctrl-C, interrupt), SIGTSTP (Ctrl-Z, stop; resume with `fg`).
- **`signal()`** lets a process catch signals and run handler code.
- **Users**: a user can only control **their own** processes. **Superuser (root)** can control all (`kill` any process) — use sparingly.

## 6. Useful Tools
| Tool | Use |
|---|---|
| `ps` | List processes |
| `top` | Live resource usage |
| `kill`, `killall` | Send signals |
| `man` | Read man pages! |

> ⚠️ `fork()` has critics (e.g. "A fork() in the road", HotOS'19): complex, slow with large address spaces, bad with threads. `posix_spawn()` is an alternative.
