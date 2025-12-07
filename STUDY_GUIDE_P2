
## Chapters 28/29: Locks

### Lock Basics
```c
lock_t mutex;
lock(&mutex);
// critical section
unlock(&mutex);
```

### Properties of Locks
- **Mutual exclusion**: Only one thread in critical section
- **Fairness**: Each thread eventually gets the lock
- **Performance**: Low overhead

### Atomic Instructions

**Test-and-Set**: Atomically read old value and set to 1
```c
int TestAndSet(int *ptr) {
    int old = *ptr;
    *ptr = 1;
    return old;  // returns 0 if lock was free
}
```

**Fetch-and-Add**: Atomically increment and return old value
```c
int FetchAndAdd(int *ptr) {
    int old = *ptr;
    *ptr = old + 1;
    return old;
}
```

### Spin Lock vs Sleep Lock
- **Spin lock**: Waste CPU cycles waiting (busy-waiting)
- **Sleep lock**: Yield CPU, OS wakes when lock available (efficient)

### Practice Questions with Answers

**Q1: Analyze this code for race conditions:**
```c
int balance = 0;
void deposit(int amt) {
    balance = balance + amt;
}
// Two threads call deposit(10) simultaneously
```

**Answer:**
**Race condition exists!** balance = balance + amt is 3 operations:
1. load balance → register
2. add amt → register
3. store register → balance

Interleaving:
```
T1: load(0) → T2: load(0) → T1: add(10) → T1: store(10) → T2: add(10) → T2: store(10)
Final balance: 10 (WRONG! Should be 20)
```
**Fix:** Wrap critical section in lock/unlock.

**Q2: Does this lock implementation work correctly?**
```c
typedef struct { int flag; } lock_t;
void lock(lock_t *lock) {
    while (lock->flag == 1);  // spin
    lock->flag = 1;
}
void unlock(lock_t *lock) {
    lock->flag = 0;
}
```

**Answer:**
**NO! Race condition in lock() itself!** Two threads can both see flag==0, both exit while loop, both set flag=1, both enter critical section. Needs atomic test-and-set.

**Q3: Implement a simple spin lock using Test-and-Set.**

**Answer:**
```c
typedef struct { int flag; } lock_t;

void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag) == 1)
        ; // spin until we get 0 (lock was free)
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```

**Q4: What's the difference between Test-and-Set and Fetch-and-Add?**

**Answer:**
- **Test-and-Set**: Sets value to 1, returns old value. Used for simple spin locks (0=free, 1=held). Not fair (any waiting thread can grab it).
- **Fetch-and-Add**: Increments value, returns old value. Used for ticket locks (fair FIFO ordering). Each thread gets a ticket number.

**Q5: Analyze this concurrent linked list insert for race conditions:**
```c
void insert(int value) {
    node *n = malloc(sizeof(node));
    n->value = value;
    n->next = head;
    head = n;
}
```

**Answer:**
**Race condition!** If two threads execute simultaneously:
```
T1: n1->next = head (A)
T2: n2->next = head (A)  // Both see same head!
T1: head = n1
T2: head = n2  // Overwrites! n1 is lost
```
**Fix:** Protect entire critical section (lines 3-4) with lock.

**Q6: Why is spinning wasteful, and when is it acceptable?**

**Answer:**
- **Wasteful**: Thread consumes CPU cycles checking lock repeatedly instead of doing useful work
- **Acceptable when**:
  - Critical section is very short (lock held briefly)
  - Number of CPUs ≥ number of threads (spinning thread doesn't prevent lock holder from running)
  - Real-time systems where waking from sleep is too slow

**Q7: Explain how a ticket lock using Fetch-and-Add provides fairness.**

**Answer:**
```c
lock: ticket = FetchAndAdd(&lock->ticket);
      while (lock->turn != ticket);
unlock: lock->turn++;
```
Each thread gets a unique ticket number (FIFO). Lock serves threads in ticket order, preventing starvation. Unlike Test-and-Set where any thread can steal the lock.

---

## Chapter 30: Condition Variables

### Purpose
- Allow threads to **wait** for a condition to become true
- Avoids wasteful spinning to check a condition

### API
```c
pthread_cond_wait(cond_t *c, mutex_t *m);   // Release lock, sleep, reacquire lock
pthread_cond_signal(cond_t *c);             // Wake one waiting thread
pthread_cond_broadcast(cond_t *c);          // Wake all waiting threads
```

### Bounded Buffer (Producer/Consumer)
```c
int buffer[MAX], fill = 0, use = 0, count = 0;
cond_t empty, fill_cv;
mutex_t mutex;

void put(int value) {
    buffer[fill] = value;
    fill = (fill + 1) % MAX;
    count++;
}

int get() {
    int tmp = buffer[use];
    use = (use + 1) % MAX;
    count--;
    return tmp;
}
```

### Producer/Consumer Pattern
```c
// Producer
lock(&mutex);
while (count == MAX)
    wait(&empty, &mutex);  // buffer full, wait
put(item);
signal(&fill_cv);          // signal consumer
unlock(&mutex);

// Consumer
lock(&mutex);
while (count == 0)
    wait(&fill_cv, &mutex); // buffer empty, wait
item = get();
signal(&empty);             // signal producer
unlock(&mutex);
```

### Covering Condition
- Use `broadcast()` instead of `signal()` when multiple threads might satisfy different conditions

### Practice Questions with Answers

**Q1: Why must wait() be called with the lock held?**

**Answer:** To avoid lost wakeup problem. If we release lock before wait():
```
T1: check (count==0) → true
[T2: put() → signal()] ← runs before T1 waits!
T1: wait() ← misses signal, sleeps forever
```
wait() atomically releases lock and sleeps, preventing this race.

**Q2: Why use `while` instead of `if` when checking condition before wait()?**

**Answer:**
- **Spurious wakeups**: Thread can wake up without signal (OS artifact)
- **Mesa semantics**: Signaler doesn't guarantee condition still true when waiter runs (another thread might grab lock first and consume the resource)
Must recheck condition after waking up.

**Q3: Analyze this code for bugs:**
```c
lock(&m);
if (count == 0)
    wait(&c, &m);
item = get();
unlock(&m);
```

**Answer:**
**Bug: using `if` instead of `while`!**
- After wakeup, another thread might run first and consume item
- When this thread gets lock, count might be 0 again
- **Fix:** Change `if` to `while`

**Q4: How does pthread_cond_wait work internally?**

**Answer:**
1. Atomically release the mutex
2. Put thread to sleep (wait queue)
3. When signaled, wake up
4. Reacquire the mutex (may block here!)
5. Return to caller

**Q5: In bounded buffer, why do we need both `empty` and `fill_cv` condition variables?**

**Answer:**
- **empty**: Producers wait when buffer full
- **fill_cv**: Consumers wait when buffer empty

If we used one CV:
- Producer signals when adding item → might wake another producer (wrong!)
- Need separate CVs so producers wake consumers and vice versa

**Q6: What is a covering condition and when do you use broadcast()?**

**Answer:**
- **Covering condition**: A condition that might satisfy multiple different waiting threads
- Use `broadcast()` when you don't know which specific thread's condition is satisfied
- Example: Memory allocator with threads waiting for different-sized blocks. When freeing memory, broadcast to wake all; each checks if their requested size is now available.

**Q7: Fix this producer code:**
```c
lock(&m);
put(item);
signal(&c);
unlock(&m);
// Missing: check if buffer was full!
```

**Answer:**
```c
lock(&m);
while (count == MAX)
    wait(&empty, &m);  // Wait if full
put(item);
signal(&fill_cv);      // Wake consumer
unlock(&m);
```

---

## Chapter 31: Semaphores

### Definition
- Integer value with two operations: **wait()** (P) and **post()** (V)
- **sem_wait()**: Decrement; if negative, sleep
- **sem_post()**: Increment; if threads waiting, wake one

### As a Lock (Binary Semaphore)
```c
sem_t m;
sem_init(&m, 0, 1);  // Initial value = 1

sem_wait(&m);   // Lock (decrement to 0)
// critical section
sem_post(&m);   // Unlock (increment to 1)
```

### As a Condition Variable
```c
sem_t s;
sem_init(&s, 0, 0);  // Initial value = 0

// Thread 1
sem_wait(&s);  // Wait (blocks if 0)

// Thread 2
sem_post(&s);  // Signal (wakes waiter)
```

### Producer/Consumer with Semaphores
```c
sem_t empty, full, mutex;
sem_init(&empty, 0, MAX);  // # empty slots
sem_init(&full, 0, 0);     // # full slots
sem_init(&mutex, 0, 1);    // Lock

// Producer
sem_wait(&empty);  // Wait for empty slot
sem_wait(&mutex);  // Lock
put(item);
sem_post(&mutex);  // Unlock
sem_post(&full);   // Signal full slot

// Consumer
sem_wait(&full);   // Wait for full slot
sem_wait(&mutex);  // Lock
item = get();
sem_post(&mutex);  // Unlock
sem_post(&empty);  // Signal empty slot
```

### Dining Philosophers (Canonical Solution)
```c
// Prevent deadlock: last philosopher picks up forks in opposite order
void getforks(int p) {
    if (p == 4) {  // Last philosopher
        sem_wait(forks[right(p)]);  // Right first!
        sem_wait(forks[left(p)]);
    } else {
        sem_wait(forks[left(p)]);   // Left first
        sem_wait(forks[right(p)]);
    }
}
```

### Practice Questions with Answers

**Q1: What's the difference between a semaphore initialized to 1 vs 0?**

**Answer:**
- **sem_init(&s, 0, 1)**: Binary semaphore used as a **lock**. First wait() succeeds (1→0), subsequent waits block until post() (0→1).
- **sem_init(&s, 0, 0)**: Used as **condition variable** for ordering. wait() blocks until another thread posts (0→1).

**Q2: Can this code deadlock? Why?**
```c
// Thread 1
sem_wait(&s1);
sem_wait(&s2);

// Thread 2
sem_wait(&s2);
sem_wait(&s1);
```

**Answer:**
**YES! Classic deadlock:**
- T1 acquires s1, T2 acquires s2
- T1 waits for s2 (held by T2)
- T2 waits for s1 (held by T1)
- Both blocked forever
**Fix:** Both acquire in same order (s1 then s2)

**Q3: In producer/consumer, why must we wait(&empty) BEFORE wait(&mutex)?**

**Answer:**
If we reversed the order:
```c
sem_wait(&mutex);  // Lock first
sem_wait(&empty);  // Then wait for slot
```
Problem: If buffer is full, producer holds mutex while sleeping! Consumer can't run (needs mutex to empty buffer). **Deadlock!**
Must wait for resources before acquiring locks.

**Q4: Implement a rendezvous using semaphores (two threads must reach a point before either continues).**

**Answer:**
```c
sem_t s1, s2;
sem_init(&s1, 0, 0);
sem_init(&s2, 0, 0);

// Thread 1
statement1A;
sem_post(&s1);
sem_wait(&s2);
statement1B;

// Thread 2
statement2A;
sem_post(&s2);
sem_wait(&s1);
statement2B;
```
Ensures both 1A and 2A complete before either 1B or 2B starts.

**Q5: Why does the Dining Philosophers solution prevent deadlock?**

**Answer:**
Deadlock requires circular wait: P0 waits for P1's fork, P1 waits for P2's fork, ..., P4 waits for P0's fork (cycle).

By having one philosopher (P4) grab forks in opposite order:
- P4 grabs right fork first (same as P0's left fork)
- This breaks the cycle: can't have all philosophers holding one fork and waiting for the next

At least one philosopher can always make progress.

**Q6: Can semaphores replace both locks and condition variables?**

**Answer:**
**YES**, but it's often less clear:
- **Lock**: Binary semaphore (init=1)
- **Condition Variable**: Semaphore (init=0) with post() as signal, wait() as wait
- However, semaphores don't have the built-in lock reacquisition that cond_wait has, making some patterns more complex

**Q7: What happens if you call sem_post() without a corresponding sem_wait()?**

**Answer:**
Semaphore value increments beyond initial value. This can cause:
- Lock semaphore: Value > 1 means multiple threads can enter critical section (broken mutual exclusion!)
- Counting semaphore: May allow more resources than actually exist
Unlike condition variables (signal with no waiter is lost), semaphore posts accumulate.

---

## Chapter 32: Concurrency Bugs

### Non-Deadlock Bugs

**Atomicity Violation**: Failing to protect logically grouped accesses
```c
// Thread 1
if (ptr != NULL)
    // Thread 2 could run here and free ptr!
    foo = ptr->field;  // CRASH

// Fix: Lock around the entire check-and-use
```

**Order Violation**: Assuming operations happen in specific sequence
```c
// Thread 1
void init() { ptr = malloc(...); }

// Thread 2
void use() { ptr->field = 5; }  // Assumes init() ran first!

// Fix: Use condition variable for ordering
```

### Deadlock Conditions (All 4 Required)

1. **Mutual exclusion**: Threads claim exclusive control of resources
2. **Hold-and-wait**: Thread holds resources while waiting for more
3. **No preemption**: Resources can't be forcibly taken
4. **Circular wait**: Circular chain of threads waiting for each other

### Deadlock Prevention

**Total Ordering**: All threads acquire locks in same order
```c
// Always acquire L1 before L2
lock(L1); lock(L2); // Good
lock(L2); lock(L1); // BAD - can deadlock
```

**Trylock**: Avoid hold-and-wait
```c
top:
lock(L1);
if (trylock(L2) == -1) {
    unlock(L1);
    goto top;  // Retry
}
```

**Lock-free Data Structures**: Use compare-and-swap instead of locks

### Practice Questions with Answers

**Q1: Identify the bug type:**
```c
// Thread 1
if (list->head != NULL)
    value = list->head->data;

// Thread 2 (runs between the if and dereference)
free(list->head);
list->head = NULL;
```

**Answer:**
**Atomicity violation**. Thread 1 fails to protect the logically grouped operations (check and use). The check and dereference must be atomic.

**Fix:**
```c
lock(&m);
if (list->head != NULL)
    value = list->head->data;
unlock(&m);
```

**Q2: Identify the bug type:**
```c
// Thread 1
config = load_config();  // Slow

// Thread 2 (starts immediately)
use(config);  // config not initialized yet!
```

**Answer:**
**Order violation**. Thread 2 assumes Thread 1's initialization completed first, but there's no synchronization enforcing this order.

**Fix:**
```c
sem_t ready;
sem_init(&ready, 0, 0);

// Thread 1
config = load_config();
sem_post(&ready);

// Thread 2
sem_wait(&ready);
use(config);
```

**Q3: Why does this code deadlock?**
```c
// Thread 1
lock(A); lock(B);

// Thread 2
lock(B); lock(A);
```

**Answer:**
**Circular wait** (one of the 4 deadlock conditions):
- T1 holds A, waits for B
- T2 holds B, waits for A
- Circular dependency: T1→T2→T1

**Fix:** Both threads acquire in same order (A then B).

**Q4: Which deadlock condition does trylock() break?**

**Answer:**
**Hold-and-wait**. With trylock, if we can't get the second lock, we release the first lock (don't hold resources while waiting). This breaks the hold-and-wait condition, preventing deadlock.

**Q5: Real-world studies show most concurrency bugs are NOT deadlocks. What are the two main types?**

**Answer:**
1. **Atomicity violations** (~97% of non-deadlock bugs): Failing to protect grouped accesses
2. **Order violations** (~3% of non-deadlock bugs): Assuming specific execution order without enforcing it

Deadlocks are actually rare in practice compared to these.

**Q6: Explain how compare-and-swap enables lock-free data structures.**

**Answer:**
```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected)
        *ptr = new;
    return actual;
}
```
Atomically checks if value is expected, updates if so. This enables lock-free algorithms:
```c
do {
    old = head;
    new->next = old;
} while (CompareAndSwap(&head, old, new) != old);
```
Retry loop instead of locks. Avoids deadlock entirely (breaks mutual exclusion condition).

**Q7: A system has locks L1, L2, L3. Thread A: L1→L2, Thread B: L2→L3, Thread C: L3→L1. Can this deadlock?**

**Answer:**
**YES!** Circular wait exists:
- A holds L1, wants L2
- B holds L2, wants L3
- C holds L3, wants L1
- Cycle: A→B→C→A

**Fix:** Establish total order (e.g., always L1→L2→L3), so all threads acquire in same sequence.

---

## Chapters 36-41: I/O and File Systems

### Canonical I/O Device
```
Device Components:
┌─────────────────┐
│ Status Register │ ← Check if device ready/busy
├─────────────────┤
│Command Register │ ← Tell device what to do
├─────────────────┤
│  Data Register  │ ← Transfer data
└─────────────────┘
```

### Interrupts vs Polling

**Polling**: CPU repeatedly checks device status (wastes CPU cycles)
```c
while (STATUS == BUSY)
    ; // spin
```

**Interrupts**: Device notifies CPU when done (efficient, but overhead per interrupt)
- Use polling for fast devices
- Use interrupts for slow devices (disk, network)

### Disk Geometry
- **Platter**: Circular disk surface
- **Track**: Concentric circle on platter
- **Sector**: Smallest unit (512 bytes or 4KB)
- **Cylinder**: Same track across multiple platters
- **Seek**: Move disk head to track (slow, ms)
- **Rotation**: Wait for sector to rotate under head (medium)
- **Transfer**: Read/write data (fast, μs)

### Disk Scheduling Algorithms

**FCFS**: First Come First Served (fair, but slow)
**SSTF**: Shortest Seek Time First (greedy, can starve)
**SCAN** (Elevator): Sweep back and forth across disk
**C-SCAN**: Sweep one direction, jump back, repeat (more fair)

### FFS (Fast File System) Basics
- **Block groups**: Divide disk into groups
- **Locality**: Keep related data close (same directory in same group)
- **Large files**: Spread across groups to balance load

### Practice Questions with Answers

**Q1: When should you use interrupts vs polling?**

**Answer:**
- **Interrupts**: Slow devices (disk, network). CPU can do other work while waiting. Interrupt overhead is small compared to device latency.
- **Polling**: Fast devices (some SSDs, fast network cards). If device is almost always ready, polling wastes less time than interrupt overhead.
- **Rule of thumb**: If device time > cost of context switch, use interrupts.

**Q2: Disk queue has requests for tracks: 53, 183, 37, 122, 14. Current track: 53. What order for SSTF?**

**Answer:**
Start at 53:
- Closest: 37 (distance 16)
- From 37, closest: 14 (distance 23)
- From 14, closest: 122 (distance 108)
- From 122, closest: 183 (distance 61)

**Order: 53→37→14→122→183**

**Q3: What are the three main components of a canonical I/O device?**

**Answer:**
1. **Status register**: Reports device state (ready, busy, error)
2. **Command register**: CPU writes commands here (read, write, seek)
3. **Data register**: Transfer data between device and CPU/memory

**Q4: Calculate disk access time: seek=5ms, rotation=4ms average, transfer=0.1ms.**

**Answer:**
Total access time = Seek + Rotation + Transfer
= 5ms + 4ms + 0.1ms = **9.1ms**

(Seek is slowest component, which is why disk scheduling focuses on minimizing seeks)

**Q5: What is the elevator (SCAN) algorithm and why is it better than SSTF?**

**Answer:**
- **SCAN**: Move disk head in one direction, servicing all requests, then reverse direction
- **Better than SSTF** because SSTF can starve requests at disk edges (if requests keep coming for middle tracks)
- SCAN guarantees all requests eventually serviced (fairness)
- Like an elevator: services all floors while going up, then all while going down

**Q6: In FFS, why keep files from the same directory in the same block group?**

**Answer:**
- **Locality**: Files in same directory often accessed together (e.g., ls, compiling source files)
- Keeping them physically close minimizes seek time
- Improves performance for common access patterns (listing directory, reading related files)
