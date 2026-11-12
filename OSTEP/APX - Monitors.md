# APX - Monitors (Deprecated)

## Big Picture
- **Monitor** (Brinch Hansen, Hoare, 1970s): a language construct for concurrency — a class where **only one thread can be active inside at a time**.
- Locking is **automatic**: entering a method acquires the monitor's lock, leaving releases it.
- Waiting uses **condition variables** inside the monitor. Survives today in Java's `synchronized`.

---

## 1. Example
```cpp
monitor class account {
private:
    int balance = 0;
public:
    void deposit(int amount)  { balance = balance + amount; }
    void withdraw(int amount) { balance = balance - amount; }
};
```
- What the compiler effectively produces:
```cpp
class account {
private:
    int balance = 0;
    pthread_mutex_t monitor;
public:
    void deposit(int amount) {
        pthread_mutex_lock(&monitor);
        balance = balance + amount;
        pthread_mutex_unlock(&monitor);
    }
    // withdraw() likewise
};
```

## 2. Why Bother?
- Object-oriented programming was popular; monitors fit concurrency into OOP (each object protects its own data).
- Removes forgotten-lock bugs.

## 3. Condition Variables in Monitors (Producer/Consumer)
```cpp
monitor class BoundedBuffer {
private:
    int buffer[MAX];
    int fill, use;
    int fullEntries = 0;
    cond_t empty;
    cond_t full;
public:
    void produce(int element) {
        if (fullEntries == MAX)      // line P0
            wait(&empty);            // line P1
        buffer[fill] = element;      // line P2
        fill = (fill + 1) % MAX;     // line P3
        fullEntries++;               // line P4
        signal(&full);               // line P5
    }
    int consume() {
        if (fullEntries == 0)        // line C0
            wait(&full);             // line C1
        int tmp = buffer[use];       // line C2
        use = (use + 1) % MAX;       // line C3
        fullEntries--;               // line C4
        signal(&empty);              // line C5
        return tmp;                  // line C6
    }
};
```
- No explicit lock: the monitor provides it. `wait()` releases the monitor and sleeps.
- This `if`-based version is correct only under **Hoare semantics**.

## 4. Hoare vs Mesa Semantics
| | Hoare semantics | Mesa semantics |
|---|---|---|
| On `signal()` | Signaler **immediately** hands the monitor to the woken thread; it runs right away | Woken thread just moves to **ready**; signaler keeps running |
| Guarantee | Condition still true when the waiter runs | Condition may have changed before the waiter runs |
| Waiting code | `if` is OK | Must use **`while`** |
| Used by | Theory | Mesa (Xerox), and virtually every real system (pthreads, Java) |

- Mesa example bug: consumer C1 woken by the producer, but consumer C2 runs first and takes the item → C1 runs with an empty buffer → error. Fix: `while (fullEntries == 0) wait(&full);`.

## 5. Peeking Under the Hood: Implementing Monitors
- A C++ class with an explicit `pthread_mutex_t` locked/unlocked in every public method, plus `pthread_cond_t` members — exactly what we've done by hand in the locks/CV chapters.

## 6. Where Monitors Live Today: Java
```java
public synchronized void deposit(int amount) { balance += amount; }
```
- `synchronized` methods acquire the object's monitor lock. `wait()`, `notify()`, `notifyAll()` — but originally **only one condition variable per object** → `notifyAll()` often needed (covering-condition style). Later Java added explicit locks and multiple CVs (`java.util.concurrent`).

> 💡 Monitors are mostly of historical interest now ("deprecated" appendix), but they explain where Mesa semantics and the "always use while" rule came from.
