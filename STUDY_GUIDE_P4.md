# OS Final Study Guide - Part 4

## Chapter 32: Common Concurrency Problems

**THE CRUX: How To Handle Common Concurrency Bugs**
Concurrency bugs tend to come in a variety of common patterns. Knowing which ones to look out for is the first step to writing more robust, correct concurrent code.

### Key Insight from Real-World Study (Lu et al.)
- Study analyzed 105 concurrency bugs from MySQL, Apache, Mozilla, and OpenOffice
- **74 were non-deadlock bugs, only 31 were deadlock bugs**
- Most concurrency bugs are NOT deadlocks—race conditions are far more common
- 97% of non-deadlock bugs are either atomicity or order violations

---

### Practice Questions

#### 1. What are the two major types of non-deadlock concurrency bugs, and what percentage of non-deadlock bugs do they represent?

**Answer:**
The two major types are:

1. **Atomicity Violation Bugs**: When the desired serializability among multiple memory accesses is violated (i.e., a code region is intended to be atomic, but the atomicity is not enforced during execution).

2. **Order Violation Bugs**: When the desired order between two (groups of) memory accesses is flipped (i.e., A should always be executed before B, but the order is not enforced during execution).

Together, these two types represent **97%** of all non-deadlock bugs found in the Lu et al. study. This means if you focus on preventing these two patterns, you can prevent the vast majority of concurrency bugs.

#### 2. Identify the bug in this MySQL code and explain how to fix it:

```c
Thread 1::
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}

Thread 2::
thd->proc_info = NULL;
```

**Answer:**
This is an **atomicity violation bug**.

**The Problem**: Thread 1 checks if `proc_info` is non-NULL, but before it can call `fputs()`, Thread 2 could run and set `proc_info` to NULL. When Thread 1 resumes and calls `fputs()`, it will crash with a NULL pointer dereference.

**The Fix**: Add locks around the shared variable accesses:
```c
pthread_mutex_t proc_info_lock = PTHREAD_MUTEX_INITIALIZER;

Thread 1::
pthread_mutex_lock(&proc_info_lock);
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}
pthread_mutex_unlock(&proc_info_lock);

Thread 2::
pthread_mutex_lock(&proc_info_lock);
thd->proc_info = NULL;
pthread_mutex_unlock(&proc_info_lock);
```

The lock ensures that when either thread accesses `proc_info`, no other thread can interfere.

#### 3. What are the four conditions that must hold for deadlock to occur? How can you prevent deadlock by breaking each condition?

**Answer:**
The four conditions for deadlock (Coffman et al.):

1. **Mutual exclusion**: Threads claim exclusive control of resources (e.g., locks)
   - *Prevention*: Use lock-free data structures via compare-and-swap instructions

2. **Hold-and-wait**: Threads hold resources while waiting for additional ones
   - *Prevention*: Acquire all locks at once, atomically (e.g., use a global prevention lock before acquiring any other locks)

3. **No preemption**: Resources cannot be forcibly removed from threads
   - *Prevention*: Use `pthread_mutex_trylock()` to allow backing out of lock acquisition gracefully

4. **Circular wait**: A circular chain exists where each thread holds resources needed by the next
   - *Prevention*: Enforce **total or partial lock ordering** (e.g., always acquire L1 before L2)

**Most practical approach**: Prevent circular wait by establishing a lock ordering convention.

#### 4. Explain the total lock ordering prevention strategy. How can you implement it when a function must acquire two locks passed as parameters?

**Answer:**
**Total lock ordering** means establishing a global order for all lock acquisitions. For example, if locks L1 and L2 exist, always acquire L1 before L2. This prevents circular wait and thus prevents deadlock.

**Challenge**: What if a function `do_something(mutex_t *m1, mutex_t *m2)` is called with locks in different orders?
- One thread calls `do_something(L1, L2)`
- Another calls `do_something(L2, L1)`

**Solution**: Use lock addresses to enforce ordering:
```c
if (m1 > m2) {  // grab in high-to-low address order
    pthread_mutex_lock(m1);
    pthread_mutex_lock(m2);
} else {
    pthread_mutex_lock(m2);
    pthread_mutex_lock(m1);
}
// Code assumes m1 != m2
```

By always acquiring locks in the same address order (high-to-low or low-to-high), you guarantee consistent ordering regardless of parameter order.

#### 5. How does the trylock approach prevent deadlock? What new problem does it introduce?

**Answer:**
The **trylock approach** uses `pthread_mutex_trylock()` which either:
- Grabs the lock and returns success, OR
- Returns an error if the lock is held (without blocking)

**How it prevents deadlock**:
```c
top:
pthread_mutex_lock(L1);
if (pthread_mutex_trylock(L2) != 0) {
    pthread_mutex_unlock(L1);  // Release L1
    goto top;                   // Try again
}
```

If we can't get L2 while holding L1, we release L1 and try again. This breaks the hold-and-wait condition because we don't hold L1 while waiting for L2.

**New problem - Livelock**: Two threads could repeatedly attempt this sequence and keep failing. Both systems are running (not deadlocked), but no progress is made.

**Solution to livelock**: Add a random delay before looping back, decreasing the odds of repeated interference.

#### 6. Identify the bug in this code and explain how to fix it using condition variables:

```c
Thread 1::
void init() {
    mThread = PR_CreateThread(mMain, ...);
}

Thread 2::
void mMain(...) {
    mState = mThread->State;
}
```

**Answer:**
This is an **order violation bug**. Thread 2 assumes `mThread` has been initialized, but if Thread 2 runs first, `mThread` will be NULL, causing a NULL pointer dereference.

**Fix using condition variables**:
```c
pthread_mutex_t mtLock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t mtCond = PTHREAD_COND_INITIALIZER;
int mtInit = 0;

Thread 1::
void init() {
    mThread = PR_CreateThread(mMain, ...);

    // Signal that thread has been created
    pthread_mutex_lock(&mtLock);
    mtInit = 1;
    pthread_cond_signal(&mtCond);
    pthread_mutex_unlock(&mtLock);
}

Thread 2::
void mMain(...) {
    // Wait for thread to be initialized
    pthread_mutex_lock(&mtLock);
    while (mtInit == 0)
        pthread_cond_wait(&mtCond, &mtLock);
    pthread_mutex_unlock(&mtLock);

    mState = mThread->State;
}
```

Thread 2 now waits for Thread 1 to signal that initialization is complete before accessing `mThread`.

#### 7. How do lock-free data structures prevent deadlock? Provide an example using compare-and-swap.

**Answer:**
Lock-free data structures prevent deadlock by avoiding locks entirely, thus breaking the **mutual exclusion** condition. They use atomic hardware instructions like **compare-and-swap (CAS)**.

**Example - Atomic increment**:
```c
int CompareAndSwap(int *address, int expected, int new) {
    if (*address == expected) {
        *address = new;
        return 1; // success
    }
    return 0; // failure
}

void AtomicIncrement(int *value, int amount) {
    do {
        int old = *value;
    } while (CompareAndSwap(value, old, old + amount) == 0);
}
```

Instead of locking, this repeatedly tries to update the value. It reads the old value, then uses CAS to atomically update it only if it hasn't changed. If another thread modified it, CAS fails and we retry.

**Lock-free list insertion**:
```c
void insert(int value) {
    node_t *n = malloc(sizeof(node_t));
    n->value = value;
    do {
        n->next = head;
    } while (CompareAndSwap(&head, n->next, n) == 0);
}
```

No locks are acquired, so no deadlock can arise (though livelock is still possible).

**Trade-off**: Lock-free structures are complex to implement correctly and don't work for all scenarios, but they provide excellent performance and deadlock immunity when applicable.

---

## Chapters 36-41: I/O and File Systems

**Note**: Coverage was minimal in class. Focus on basic mechanisms only.

### Key Concepts Summary

#### I/O Devices (Chapter 36)
- **Canonical Device Structure**:
  - **Status register**: Check device state
  - **Command register**: Tell device what to do
  - **Data register**: Pass data to/from device

- **Interrupts vs Polling**:
  - **Polling**: CPU repeatedly checks status register (wastes CPU cycles, good for fast devices)
  - **Interrupts**: Device notifies CPU when done (better for slow devices, avoids wasting CPU)
  - Hybrid approach: Poll for a bit, then use interrupts if device is slow

#### Hard Disk Drives (Chapter 37)
- **Disk Geometry Terms**:
  - **Platter**: Circular hard surface for magnetic storage
  - **Surface**: Each platter has 2 surfaces (top and bottom)
  - **Track**: Concentric circle of sectors on a surface
  - **Cylinder**: All tracks at the same radius across all surfaces
  - **Sector**: Smallest unit that can be read/written (usually 512 bytes or 4KB)
  - **Spindle**: What platters rotate around

- **Disk Scheduling Algorithms**:
  - **FIFO (FCFS)**: First-In-First-Out, simple but no optimization
  - **SSTF (Shortest Seek Time First)**: Service request with shortest seek time (can cause starvation)
  - **SCAN (Elevator)**: Move in one direction servicing requests, then reverse (fair, reasonable performance)
  - **C-SCAN**: Like SCAN but only services in one direction, returns to start without servicing
  - **LOOK/C-LOOK**: Like SCAN/C-SCAN but only go as far as last request in each direction

#### Fast File System - FFS (Chapter 41)
- **FFS Mechanisms** (basics only):
  - **Cylinder groups**: Organize disk into groups of cylinders to keep related data close
  - **Goal**: Keep files in same directory together in same cylinder group
  - **Block size**: Use larger blocks (4KB or 8KB) for better performance
  - **Fragments**: Use sub-blocks for small files to avoid waste
  - **Locality**: Place inode and data blocks close together to reduce seek time

---

### Practice Questions (Chapters 36-41)

#### 1. What are the three basic registers in a canonical I/O device and their purposes?

**Answer:**
1. **Status register**: Allows OS to check current status of device (e.g., is it busy? ready? error?)
2. **Command register**: OS writes commands here to tell device what operation to perform
3. **Data register**: OS can read/write data to/from the device through this register

These registers provide the basic interface between OS and hardware device.

#### 2. When would you use polling vs interrupts for I/O?

**Answer:**
**Use polling when**:
- Device is very fast (response time is predictable and short)
- You need to check status very frequently anyway
- Interrupt overhead would be higher than just checking status

**Use interrupts when**:
- Device is slow (disk, network)
- You don't want to waste CPU cycles checking status repeatedly
- You want CPU to do other work while waiting for I/O

**Best approach**: Hybrid - poll for a short time, then switch to interrupts if device takes longer than expected.

#### 3. What are the key disk geometry terms you need to know?

**Answer:**
- **Platter**: Physical disk surface (hard drives have multiple platters)
- **Track**: Concentric ring of data on a disk surface
- **Sector**: Smallest addressable unit on disk (512B or 4KB)
- **Cylinder**: Collection of tracks at same position on all platters
- **Spindle**: Central axis that platters rotate around
- **Head**: Reads/writes data from disk surface

**Important**: Understanding these helps you reason about seek time (moving head to right track), rotational delay (waiting for sector to rotate under head), and transfer time (reading data).

#### 4. Compare FIFO, SSTF, and SCAN disk scheduling algorithms.

**Answer:**
**FIFO (First-In-First-Out)**:
- Service requests in order received
- Simple, fair
- Poor performance (lots of seeking)

**SSTF (Shortest Seek Time First)**:
- Always service request closest to current head position
- Better performance than FIFO
- **Problem**: Starvation - requests far away may never get serviced

**SCAN (Elevator)**:
- Move head in one direction, service all requests in path
- When reach end, reverse direction
- Fair (no starvation) and good performance
- Like elevator in building

**Best for most cases**: SCAN provides good balance of fairness and performance.

#### 5. What are the basic FFS mechanisms for improving file system performance?

**Answer:**
FFS (Fast File System) introduced several key mechanisms:

1. **Cylinder groups**: Divide disk into groups, keep related data together in same group
   - Files in same directory stored in same cylinder group
   - Reduces seek time for related operations

2. **Larger block sizes**: Use 4KB or 8KB blocks instead of 512B
   - Improves throughput
   - Better for large files

3. **Fragments**: Sub-divide blocks for small files
   - Avoids wasting space when files don't fill whole blocks

4. **Locality optimizations**:
   - Keep inode and data blocks close together
   - Keep directory and its files close together
   - Reduces seek time

**Goal**: Group related data together to minimize disk head movement and improve performance.

---

## Summary

**Chapter 32 - Most Important Points**:
- Most concurrency bugs are NOT deadlocks (74 vs 31 in study)
- 97% of non-deadlock bugs are atomicity or order violations
- Fix atomicity violations with locks around grouped operations
- Fix order violations with condition variables
- Four conditions for deadlock: mutual exclusion, hold-and-wait, no preemption, circular wait
- Best prevention: enforce total lock ordering

**Chapters 36-41 - Key Basics**:
- Know I/O device registers: status, command, data
- Interrupts vs polling trade-offs
- Disk geometry: platter, track, sector, cylinder
- Disk scheduling: FIFO, SSTF, SCAN
- FFS: cylinder groups for locality, larger blocks, keep related data together
