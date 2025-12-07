# OS Final Study Guide - Part 3

## Chapter 30: Condition Variables

### The Crux: How To Wait For A Condition
In multi-threaded programs, it is often useful for a thread to wait for some condition to become true before proceeding. The simple approach, of just spinning until the condition becomes true, is grossly inefficient and wastes CPU cycles, and in some cases, can be incorrect. Thus, how should a thread wait for a condition?

### Key Concepts

**What is a Condition Variable?**
- A **condition variable** is an explicit queue that threads can put themselves on when some state of execution (i.e., some condition) is not as desired (by waiting on the condition)
- Some other thread, when it changes said state, can then wake one (or more) of those waiting threads and thus allow them to continue (by signaling on the condition)
- Two operations: `wait()` and `signal()`

**POSIX Calls:**
```c
pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);
pthread_cond_signal(pthread_cond_t *c);
```

**How wait() works:**
1. Assumes the mutex is locked when `wait()` is called
2. Releases the lock and puts the calling thread to sleep (atomically)
3. When the thread wakes up (after being signaled), it re-acquires the lock before returning

**Mesa vs Hoare Semantics:**
- **Mesa semantics**: Signaling a thread only wakes them up; it's a hint that the state has changed, but no guarantee the state will still be as desired when the woken thread runs. (Most systems use this)
- **Hoare semantics**: Provides stronger guarantee that woken thread will run immediately upon being woken (harder to build)

### The Producer/Consumer (Bounded Buffer) Problem

**Problem Description:**
- One or more producer threads generate data items and place them in a buffer
- One or more consumer threads grab items from the buffer and consume them
- The bounded buffer is a shared resource requiring synchronized access

**Real-world examples:**
- Multi-threaded web server: producers put HTTP requests into work queue, consumers process them
- UNIX pipes: `grep foo file.txt | wc -l` - grep is producer, wc is consumer

### Common Mistakes and Solutions

**Mistake 1: No State Variable**
```c
void thr_exit() {
    Pthread_mutex_lock(&m);
    Pthread_cond_signal(&c);  // Signal with no state variable
    Pthread_mutex_unlock(&m);
}
```
**Problem**: If child runs first and signals before parent waits, parent will wait forever.
**Solution**: Always use a state variable (e.g., `done`) to record what threads care about.

**Mistake 2: No Lock When Signaling/Waiting**
```c
void thr_exit() {
    done = 1;
    Pthread_cond_signal(&c);  // No lock held!
}
```
**Problem**: Race condition - parent might check `done==0`, then get interrupted; child sets `done=1` and signals (but no one waiting); parent then waits forever.
**Solution**: Always hold the lock when calling signal or wait.

**Mistake 3: Using if Instead of while**
```c
if (count == 0)  // Wrong! Should be while
    Pthread_cond_wait(&cond, &mutex);
```
**Problem**: With multiple consumers, a woken thread might find the condition false again (another consumer "stole" the data).
**Solution**: **Always use while loops, not if statements** when checking conditions.

**Mistake 4: Single Condition Variable**
With one CV and multiple producers/consumers, a consumer might accidentally wake another consumer (not a producer), causing all threads to sleep.
**Solution**: Use two condition variables - one for producers (empty) and one for consumers (fill).

### The Correct Producer/Consumer Solution

**State and Synchronization:**
```c
int buffer[MAX];
int fill_ptr = 0;
int use_ptr = 0;
int count = 0;

cond_t empty, fill;
mutex_t mutex;
```

**Producer Code:**
```c
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == MAX)                    // Wait if buffer full
            Pthread_cond_wait(&empty, &mutex);  // Wait on 'empty'
        put(i);
        Pthread_cond_signal(&fill);             // Signal 'fill'
        Pthread_mutex_unlock(&mutex);
    }
}
```

**Consumer Code:**
```c
void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == 0)                      // Wait if buffer empty
            Pthread_cond_wait(&fill, &mutex);   // Wait on 'fill'
        int tmp = get();
        Pthread_cond_signal(&empty);            // Signal 'empty'
        Pthread_mutex_unlock(&mutex);
        printf("%d\n", tmp);
    }
}
```

**Why This Works:**
- Producers wait on `empty` and signal `fill`
- Consumers wait on `fill` and signal `empty`
- A consumer can never accidentally wake another consumer
- A producer can never accidentally wake another producer
- Using `while` ensures condition is re-checked after waking

### Covering Conditions

**Problem**: When you don't know which specific waiting thread should be woken.

**Example**: Memory allocator where threads wait for different amounts of memory:
- Thread Ta waits for 100 bytes
- Thread Tb waits for 10 bytes
- Thread Tc frees 50 bytes - which thread to wake?

**Solution**: Use `pthread_cond_broadcast()` instead of `pthread_cond_signal()`
- Wakes up **all** waiting threads
- Each thread re-checks its condition
- Threads that can't proceed go back to sleep
- Guarantees correct thread(s) will wake up
- **Cost**: May needlessly wake threads that will just go back to sleep

**When to use broadcast:**
- When you can't determine which specific thread should wake
- Called a "covering condition" because it conservatively covers all cases

### Important Rules and Tips

**TIP: Always Hold The Lock While Signaling**
Although not always strictly necessary, it's simplest and safest to hold the lock when calling signal. It's **mandatory** to hold the lock when calling wait.

**TIP: Use While (Not If) For Conditions**
- Always use while loops when checking conditions
- Handles Mesa semantics correctly
- Handles spurious wakeups (implementation detail where threads wake without signal)

**TIP: Always Use State Variables**
- The sleeping, waking, and locking are all built around the state variable
- State variable records what threads care about
- Condition variables enable efficient waiting, but state drives the logic

---

## Practice Questions for Chapter 30

### Question 1: Understanding Condition Variables
**Q**: Explain why the `wait()` call takes both a condition variable and a mutex as parameters. What would happen if wait() didn't release the lock?

**A**: The `wait()` call takes both a condition variable and a mutex because it needs to:
1. Release the lock atomically while putting the thread to sleep - this prevents race conditions
2. Re-acquire the lock before returning to the caller

If `wait()` didn't release the lock, we'd have deadlock: the waiting thread would hold the lock while sleeping, preventing any other thread from acquiring the lock to change the condition and signal the waiting thread. The atomicity of releasing the lock and sleeping is crucial to prevent the race where a signal happens between checking the condition and going to sleep.

### Question 2: Mesa vs Hoare Semantics
**Q**: What is the difference between Mesa and Hoare semantics for condition variables? Which do most systems use and why?

**A**:
- **Mesa semantics**: Signaling a thread only wakes it up (a hint that state changed), but provides no guarantee that the state will still be as desired when the woken thread actually runs. The woken thread goes to the ready queue.
- **Hoare semantics**: Provides a stronger guarantee that the woken thread will run immediately upon being woken, so the condition is guaranteed to still be true.

Most systems use **Mesa semantics** because:
1. It's easier to implement efficiently
2. It's more flexible for the scheduler
3. The downside (needing to re-check conditions) is easily handled by using while loops instead of if statements

### Question 3: Producer/Consumer Race Condition Analysis
**Q**: In the broken producer/consumer code with a single condition variable and while loops, explain the race condition that occurs when two consumers (Tc1 and Tc2) and one producer (Tp) are running. Walk through the scenario where all three threads end up sleeping.

**A**: Here's the step-by-step scenario:
1. Two consumers Tc1 and Tc2 run first, find buffer empty (count==0), and both go to sleep on the single condition variable
2. Producer Tp runs, produces one item (count=1), and signals the condition variable, waking Tc1 (Tc1 moves to ready, Tc2 still sleeping)
3. Producer tries to produce again but buffer is full, so Tp waits on the condition (now Tp sleeping, Tc1 ready, Tc2 sleeping)
4. Consumer Tc1 wakes up, consumes the item (count=0), and signals the condition
5. **THE PROBLEM**: The signal might wake Tc2 instead of Tp!
6. Tc2 wakes up, checks count==0, finds buffer empty, goes back to sleep
7. Now ALL three threads are asleep (Tc1 finished its loop and sleeps, Tc2 sleeping, Tp sleeping)

The problem: Consumer Tc1 should have woken the producer Tp (because it emptied the buffer), but it accidentally woke consumer Tc2. The solution is using two condition variables: `empty` (for producers) and `fill` (for consumers).

### Question 4: Why While, Not If?
**Q**: Explain with a specific example why you must use `while` instead of `if` when checking conditions before calling `wait()`. What problem occurs with `if`?

**A**: Using `if` creates a problem when multiple threads of the same type (e.g., multiple consumers) are waiting.

**Example scenario with if:**
```c
if (count == 0)  // WRONG - using if
    pthread_cond_wait(&fill, &mutex);
int tmp = get();
```

1. Consumer Tc1 checks count==0, waits and sleeps
2. Producer creates item (count=1), signals Tc1
3. Before Tc1 runs, Consumer Tc2 runs, checks count==1, consumes item (count=0)
4. Now Tc1 wakes up, **does not re-check the condition** (because it was an if, not while), tries to call get()
5. **CRASH or ASSERTION FAILURE** - buffer is empty but Tc1 tries to get()!

With `while`, step 4 changes:
4. Tc1 wakes up, re-checks while(count==0) in the while loop, sees it's true, goes back to sleep

The while loop ensures the condition is **always** re-checked after waking up, handling Mesa semantics correctly.

### Question 5: Covering Conditions
**Q**: What is a covering condition? Describe the memory allocator scenario where covering conditions are needed, and explain the tradeoff involved.

**A**: A **covering condition** is when you use `pthread_cond_broadcast()` to wake all waiting threads because you don't know which specific thread(s) should be woken.

**Memory Allocator Scenario:**
```c
// Thread Ta calls: allocate(100) - waits because only 0 bytes free
// Thread Tb calls: allocate(10)  - waits because only 0 bytes free
// Thread Tc calls: free(50)      - which thread to wake?
```

The problem: Tc freed 50 bytes, which is enough for Tb (needs 10) but not Ta (needs 100). If Tc calls `signal()` and accidentally wakes Ta, Ta will check, see insufficient memory, and go back to sleep. Tb stays sleeping even though it could proceed. All threads might end up sleeping.

**Solution**: Use `pthread_cond_broadcast()` in free():
```c
void free(void *ptr, int size) {
    Pthread_mutex_lock(&m);
    bytesLeft += size;
    Pthread_cond_broadcast(&c);  // Wake ALL waiting threads
    Pthread_mutex_unlock(&m);
}
```

**Tradeoff:**
- **Benefit**: Guarantees that any thread that should wake up will wake up (conservative/safe)
- **Cost**: May wake many threads unnecessarily - they'll wake, check condition, find it false, and go back to sleep (performance overhead)

This is called a "covering" condition because it covers all cases where a thread needs to wake up.

### Question 6: Lock and Signal Rule
**Q**: The chapter provides a tip: "Always hold the lock while signaling." Why is this recommended? What could go wrong if you don't hold the lock while signaling?

**A**: While not always strictly necessary, holding the lock while signaling is recommended because:

1. **Simplicity and Safety**: It's a simple rule that always works correctly
2. **Prevents certain race conditions**: In some cases, not holding the lock can cause problems

**Example of problem without lock during signal:**
```c
// Thread T1
done = 1;  // Without lock!
pthread_cond_signal(&c);
```

Between setting `done=1` and calling signal, another thread could:
- Grab the lock
- Check `done==1`
- Skip waiting
- Unlock
- Do its work

Then when the signal happens, no one is waiting. While this might work in some cases, the sequencing becomes unpredictable.

**The correct pattern:**
```c
Pthread_mutex_lock(&m);
done = 1;
Pthread_cond_signal(&c);  // Signal while holding lock
Pthread_mutex_unlock(&m);
```

For `wait()`, holding the lock is **mandatory** (not just recommended):
- `wait()` requires the lock to be held when called
- It releases the lock while sleeping
- It re-acquires the lock before returning

### Question 7: Analyzing Bounded Buffer Code
**Q**: Given this producer code, identify what's wrong and explain why:

```c
void *producer(void *arg) {
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        if (count == MAX)
            Pthread_cond_wait(&empty, &mutex);
        put(i);
        Pthread_mutex_unlock(&mutex);
        Pthread_cond_signal(&fill);  // Signal outside lock
    }
}
```

**A**: There are TWO problems:

**Problem 1: Using `if` instead of `while`**
- Should be `while (count == MAX)` not `if`
- With `if`, if the producer is woken but another producer filled the buffer first, this producer won't re-check and will call `put()` on a full buffer
- This violates the bounded buffer constraint

**Problem 2: Signaling outside the lock**
- `Pthread_cond_signal(&fill)` is called after releasing the lock
- While this might work in some cases, it's not the recommended pattern
- Better practice: signal while holding the lock for simplicity and safety

**Corrected code:**
```c
void *producer(void *arg) {
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == MAX)  // Changed to while
            Pthread_cond_wait(&empty, &mutex);
        put(i);
        Pthread_cond_signal(&fill);  // Moved inside lock
        Pthread_mutex_unlock(&mutex);
    }
}
```

---

## Chapter 31: Semaphores

### The Crux: How To Use Semaphores
How can we use semaphores instead of locks and condition variables? What is the definition of a semaphore? What is a binary semaphore? Is it straightforward to build a semaphore out of locks and condition variables? To build locks and condition variables out of semaphores?

### Key Concepts

**What is a Semaphore?**
- A **semaphore** is an object with an integer value that we can manipulate with two routines: `sem_wait()` and `sem_post()`
- Invented by Dijkstra as a single primitive for all things related to synchronization
- Can be used as **both locks and condition variables**

**POSIX Semaphore Calls:**
```c
#include <semaphore.h>
sem_t s;
sem_init(&s, 0, 1);  // Initialize to value 1

int sem_wait(sem_t *s) {
    decrement the value of semaphore s by one
    wait if value of semaphore s is negative
}

int sem_post(sem_t *s) {
    increment the value of semaphore s by one
    if there are one or more threads waiting, wake one
}
```

**Important Properties:**
1. `sem_wait()` will either return right away (if value ≥ 0) or block the caller
2. `sem_post()` does not wait - it simply increments and wakes a waiting thread if any
3. When negative, the value equals the number of waiting threads (Dijkstra's invariant)
4. The operations are performed **atomically**

### Binary Semaphores (Locks)

**Using a semaphore as a lock:**
```c
sem_t m;
sem_init(&m, 0, 1);  // Initialize to 1 for lock

sem_wait(&m);
// critical section here
sem_post(&m);
```

**Initial value should be 1** because:
- First thread calls `sem_wait()`: decrements to 0, continues (lock acquired)
- Second thread calls `sem_wait()`: decrements to -1, blocks (lock held by first thread)
- First thread calls `sem_post()`: increments to 0, wakes second thread

**Why called binary semaphore:**
- Only two states: held (0 or negative) and not held (1)
- Acts exactly like a lock

### Semaphores For Ordering

**Parent waiting for child example:**
```c
sem_t s;

void *child(void *arg) {
    printf("child\n");
    sem_post(&s);  // signal here: child is done
    return NULL;
}

int main(int argc, char *argv[]) {
    sem_init(&s, 0, 0);  // Initialize to 0 for ordering!
    printf("parent: begin\n");
    pthread_t c;
    Pthread_create(&c, NULL, child, NULL);
    sem_wait(&s);  // wait here for child
    printf("parent: end\n");
    return 0;
}
```

**Initial value should be 0** because:
- **Case 1** (Parent runs first): Parent calls `sem_wait()`, decrements to -1, sleeps. Child runs, calls `sem_post()`, increments to 0, wakes parent.
- **Case 2** (Child runs first): Child calls `sem_post()`, increments to 1. Parent calls `sem_wait()`, decrements to 0, continues without sleeping.

**Rule for semaphore initialization:**
Think about the number of resources you're willing to give away immediately:
- Lock: 1 (one thread can acquire it)
- Ordering: 0 (nothing to give away until event happens)

### Producer/Consumer Problem with Semaphores

**Correct Solution:**
```c
int buffer[MAX];
int fill = 0;
int use = 0;

sem_t empty;  // Tracks empty slots
sem_t full;   // Tracks full slots
sem_t mutex;  // Protects buffer manipulation

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty);      // Wait for empty slot
        sem_wait(&mutex);      // Acquire lock
        put(i);
        sem_post(&mutex);      // Release lock
        sem_post(&full);       // Signal full slot
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&full);       // Wait for full slot
        sem_wait(&mutex);      // Acquire lock
        int tmp = get();
        sem_post(&mutex);      // Release lock
        sem_post(&empty);      // Signal empty slot
        printf("%d\n", tmp);
    }
}

int main() {
    sem_init(&empty, 0, MAX);  // MAX empty slots initially
    sem_init(&full, 0, 0);     // 0 full slots initially
    sem_init(&mutex, 0, 1);    // Binary semaphore for lock
    // ...
}
```

**Why this works:**
- `empty` and `full` track available resources (empty/full buffer slots)
- `mutex` provides mutual exclusion for buffer manipulation
- **Order matters!** Must wait for resource (empty/full) before acquiring mutex
  - Otherwise: Deadlock! (Holding mutex while waiting for resource)

**Common Mistake - Deadlock:**
```c
void *producer(void *arg) {
    sem_wait(&mutex);   // WRONG ORDER!
    sem_wait(&empty);   // Can deadlock here
    put(i);
    sem_post(&mutex);
    sem_post(&full);
}
```
If all buffers full and producer holds mutex, it blocks waiting for `empty`. Consumers can't get mutex to empty buffers. Deadlock!

### Reader-Writer Locks (31.5) ⚠️ EXAM FOCUS

**Problem:** Allow multiple concurrent readers OR one exclusive writer.

**Complete Implementation (THIS CODE WILL BE ON THE EXAM):**
```c
typedef struct _rwlock_t {
    sem_t lock;       // binary semaphore (basic lock)
    sem_t writelock;  // allow ONE writer/MANY readers
    int   readers;    // #readers in critical section
} rwlock_t;

void rwlock_init(rwlock_t *rw) {
    rw->readers = 0;
    sem_init(&rw->lock, 0, 1);
    sem_init(&rw->writelock, 0, 1);
}

void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers++;
    if (rw->readers == 1)  // first reader gets writelock
        sem_wait(&rw->writelock);
    sem_post(&rw->lock);
}

void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers--;
    if (rw->readers == 0)  // last reader lets it go
        sem_post(&rw->writelock);
    sem_post(&rw->lock);
}

void rwlock_acquire_writelock(rwlock_t *rw) {
    sem_wait(&rw->writelock);
}

void rwlock_release_writelock(rwlock_t *rw) {
    sem_post(&rw->writelock);
}
```

**How it works:**

1. **First reader** (readers goes 0→1):
   - Acquires `lock`
   - Increments `readers` to 1
   - Acquires `writelock` (blocks any writers)
   - Releases `lock`

2. **Additional readers** (readers > 1):
   - Acquire `lock`
   - Increment `readers`
   - Skip `writelock` (already held by first reader)
   - Release `lock`
   - Now multiple readers are in critical section concurrently

3. **Last reader** (readers goes N→0):
   - Acquires `lock`
   - Decrements `readers` to 0
   - Releases `writelock` (allows writers)
   - Releases `lock`

4. **Writer**:
   - Simply acquires `writelock` (blocks if any readers)
   - Exclusive access
   - Releases `writelock` when done

**Fairness Issue:**
- Readers can **starve writers**
- As long as readers keep arriving, writer never gets `writelock`
- More sophisticated solutions exist to prevent starvation

### The Dining Philosophers Problem

**Setup:**
- 5 philosophers sitting around table
- 5 forks (one between each pair)
- Each philosopher thinks, then eats (needs both left and right fork)
- Challenge: No deadlock, no starvation, high concurrency

**Helper functions:**
```c
int left(int p)  { return p; }
int right(int p) { return (p + 1) % 5; }
```

**Broken Solution (Deadlock):**
```c
sem_t forks[5];  // Each initialized to 1

void get_forks(int p) {
    sem_wait(&forks[left(p)]);   // Grab left fork
    sem_wait(&forks[right(p)]);  // Grab right fork
}

void put_forks(int p) {
    sem_post(&forks[left(p)]);
    sem_post(&forks[right(p)]);
}
```

**Why it deadlocks:**
If all philosophers grab their left fork simultaneously, each waits for right fork forever. Classic circular wait.

**Working Solution (Breaking the Dependency):**
```c
void get_forks(int p) {
    if (p == 4) {  // Last philosopher gets forks in opposite order
        sem_wait(&forks[right(p)]);
        sem_wait(&forks[left(p)]);
    } else {
        sem_wait(&forks[left(p)]);
        sem_wait(&forks[right(p)]);
    }
}
```

**Why this works:**
Philosopher 4 tries right-then-left, breaking the circular dependency. No situation where all philosophers hold one fork and wait for another.

### Thread Throttling

**Problem:** Prevent "too many" threads from executing a resource-intensive section simultaneously.

**Solution:** Use semaphore initialized to maximum allowed concurrent threads.

```c
sem_t throttle;
sem_init(&throttle, 0, MAX_CONCURRENT);  // e.g., 10

// In each thread:
sem_wait(&throttle);      // Acquire slot
// memory-intensive work here
sem_post(&throttle);      // Release slot
```

**Use case:** If 100 threads all enter memory-intensive region, system thrashes. Throttling limits to (e.g.) 10 concurrent threads in that region.

### Implementing Semaphores (Zemaphores)

**Building semaphores from locks and condition variables:**
```c
typedef struct __Zem_t {
    int value;
    pthread_cond_t cond;
    pthread_mutex_t lock;
} Zem_t;

void Zem_init(Zem_t *s, int value) {
    s->value = value;
    Cond_init(&s->cond);
    Mutex_init(&s->lock);
}

void Zem_wait(Zem_t *s) {
    Mutex_lock(&s->lock);
    while (s->value <= 0)
        Cond_wait(&s->cond, &s->lock);
    s->value--;
    Mutex_unlock(&s->lock);
}

void Zem_post(Zem_t *s) {
    Mutex_lock(&s->lock);
    s->value++;
    Cond_signal(&s->cond);
    Mutex_unlock(&s->lock);
}
```

**Note:** Building condition variables from semaphores is much trickier and error-prone!

---

## Practice Questions for Chapter 31

### Question 1: Binary Semaphore as Lock
**Q**: A semaphore initialized to 1 is used as a lock. Two threads (T0 and T1) try to acquire it. Walk through the semaphore value and thread states as T0 acquires, then T1 tries to acquire, then T0 releases.

**A**:

| Semaphore Value | Thread T0 Action | T0 State | Thread T1 Action | T1 State |
|-----------------|------------------|----------|------------------|----------|
| 1 | - | Ready | - | Ready |
| 1 | call sem_wait() | Run | - | Ready |
| 0 | decrement, continue | Run | - | Ready |
| 0 | (in critical section) | Run | - | Ready |
| 0 | - | Run | call sem_wait() | Run |
| -1 | - | Run | decrement | Run |
| -1 | - | Run | (value < 0) → sleep | **Sleep** |
| -1 | call sem_post() | Run | - | Sleep |
| 0 | increment | Run | - | Sleep |
| 0 | wake(T1) | Run | - | **Ready** |
| 0 | - | Ready | sem_wait() returns | Run |

Key points:
- Initial value 1 allows first thread through
- Second thread decrements to -1 and blocks
- When value is negative, it equals number of waiting threads
- Post increments to 0 and wakes T1

### Question 2: Semaphore for Ordering
**Q**: Explain why a semaphore used for ordering (like parent waiting for child) is initialized to 0, not 1. What would go wrong if initialized to 1?

**A**:

**Initialized to 0 (CORRECT):**
- Parent calls `sem_wait()`: decrements to -1, blocks until child signals
- Child calls `sem_post()`: increments to 0, wakes parent
- Enforces ordering: parent waits for child to finish

**Initialized to 1 (WRONG):**
```c
sem_init(&s, 0, 1);  // WRONG!

// Parent
sem_wait(&s);  // Decrements to 0, continues immediately (NO WAITING!)
printf("parent: end\n");  // Prints before child runs!

// Child
sem_post(&s);  // Increments to 1, but parent already done
```

Problem: Parent doesn't wait! It decrements from 1 to 0 and continues. The parent/child ordering is lost.

**Rule:** For ordering, initialize to the number of resources available at start. For parent/child, there are 0 "completed child" resources initially, so initialize to 0.

### Question 3: Producer/Consumer Deadlock Analysis
**Q**: In the producer/consumer code, explain why this ordering causes deadlock:
```c
void *producer(void *arg) {
    sem_wait(&mutex);   // Line 1
    sem_wait(&empty);   // Line 2
    put(i);
    sem_post(&mutex);
    sem_post(&full);
}
```

**A**:

**Deadlock scenario:**
1. Buffer is full (empty = 0, full = MAX)
2. Producer P1 calls `sem_wait(&mutex)` (Line 1)
   - Acquires mutex (mutex = 0)
3. Producer P1 calls `sem_wait(&empty)` (Line 2)
   - No empty slots! Decrements empty to -1
   - **P1 blocks while holding mutex**
4. Consumer C1 tries to consume:
   - Calls `sem_wait(&mutex)`
   - **Mutex already held by P1**
   - **C1 blocks waiting for mutex**

**Result:**
- P1 holds mutex, waiting for empty slot
- C1 could provide empty slot, but blocked waiting for mutex
- Classic deadlock: circular dependency

**Solution:** Wait for resources BEFORE acquiring mutex:
```c
sem_wait(&empty);   // First: ensure resource available
sem_wait(&mutex);   // Then: acquire lock to use it
```

### Question 4: Reader-Writer Lock Analysis ⚠️ CRITICAL
**Q**: Given the reader-writer lock code from section 31.5, trace through what happens when:
1. Reader R1 acquires read lock
2. Reader R2 acquires read lock (while R1 still holds it)
3. Writer W1 tries to acquire write lock (while R1 and R2 hold read locks)
4. R1 releases read lock
5. R2 releases read lock
6. W1 acquires write lock

**A**:

**Step 1: R1 acquires read lock**
```c
sem_wait(&rw->lock);        // R1 acquires lock, lock = 0
rw->readers++;              // readers = 1
if (rw->readers == 1)       // TRUE
    sem_wait(&rw->writelock);  // R1 acquires writelock, writelock = 0
sem_post(&rw->lock);        // R1 releases lock, lock = 1
```
- State: readers = 1, writelock = 0 (held by readers), lock = 1

**Step 2: R2 acquires read lock**
```c
sem_wait(&rw->lock);        // R2 acquires lock, lock = 0
rw->readers++;              // readers = 2
if (rw->readers == 1)       // FALSE (readers = 2)
    // SKIP acquiring writelock (already held!)
sem_post(&rw->lock);        // R2 releases lock, lock = 1
```
- State: readers = 2, writelock = 0 (still held), lock = 1
- **R1 and R2 both in critical section concurrently!**

**Step 3: W1 tries to acquire write lock**
```c
sem_wait(&rw->writelock);   // writelock = 0 (held by readers)
                            // W1 BLOCKS! Decrements to -1
```
- State: readers = 2, writelock = -1, W1 sleeping

**Step 4: R1 releases read lock**
```c
sem_wait(&rw->lock);        // R1 acquires lock, lock = 0
rw->readers--;              // readers = 1
if (rw->readers == 0)       // FALSE (readers = 1)
    // SKIP releasing writelock
sem_post(&rw->lock);        // R1 releases lock, lock = 1
```
- State: readers = 1, writelock = -1, W1 still sleeping

**Step 5: R2 releases read lock**
```c
sem_wait(&rw->lock);        // R2 acquires lock, lock = 0
rw->readers--;              // readers = 0
if (rw->readers == 0)       // TRUE
    sem_post(&rw->writelock);  // Increments to 0, WAKES W1!
sem_post(&rw->lock);        // R2 releases lock, lock = 1
```
- State: readers = 0, writelock = 0, W1 wakes up

**Step 6: W1 acquires write lock**
```c
// W1 returns from sem_wait(&rw->writelock)
// W1 now has exclusive access
```
- State: readers = 0, writelock = 0 (held by W1)

**Key Insights:**
- First reader acquires `writelock` to block writers
- Subsequent readers just increment counter (no writelock needed)
- Last reader releases `writelock` to allow writers
- Writer simply waits for/holds `writelock` for exclusive access

### Question 5: Dining Philosophers Deadlock
**Q**: In the broken dining philosophers solution, all 5 philosophers grab their left fork simultaneously. Show the state of each fork semaphore and explain why this is deadlock.

**A**:

**Initial state:**
All fork semaphores initialized to 1:
```
forks[0] = 1, forks[1] = 1, forks[2] = 1, forks[3] = 1, forks[4] = 1
```

**All philosophers grab left fork simultaneously:**
```c
// Philosopher 0: left(0) = 0
sem_wait(&forks[0]);  // forks[0] = 0

// Philosopher 1: left(1) = 1
sem_wait(&forks[1]);  // forks[1] = 0

// Philosopher 2: left(2) = 2
sem_wait(&forks[2]);  // forks[2] = 0

// Philosopher 3: left(3) = 3
sem_wait(&forks[3]);  // forks[3] = 0

// Philosopher 4: left(4) = 4
sem_wait(&forks[4]);  // forks[4] = 0
```

**Current state:**
```
forks[0] = 0 (held by P0), forks[1] = 0 (held by P1),
forks[2] = 0 (held by P2), forks[3] = 0 (held by P3),
forks[4] = 0 (held by P4)
```

**Now each tries to grab right fork:**
```c
// Philosopher 0: right(0) = 1
sem_wait(&forks[1]);  // forks[1] = -1, P0 BLOCKS (held by P1)

// Philosopher 1: right(1) = 2
sem_wait(&forks[2]);  // forks[2] = -1, P1 BLOCKS (held by P2)

// Philosopher 2: right(2) = 3
sem_wait(&forks[3]);  // forks[3] = -1, P2 BLOCKS (held by P3)

// Philosopher 3: right(3) = 4
sem_wait(&forks[4]);  // forks[4] = -1, P3 BLOCKS (held by P4)

// Philosopher 4: right(4) = 0
sem_wait(&forks[0]);  // forks[0] = -1, P4 BLOCKS (held by P0)
```

**Deadlock - Circular wait:**
```
P0 holds fork[0], waits for fork[1] (held by P1)
P1 holds fork[1], waits for fork[2] (held by P2)
P2 holds fork[2], waits for fork[3] (held by P3)
P3 holds fork[3], waits for fork[4] (held by P4)
P4 holds fork[4], waits for fork[0] (held by P0)
```

All philosophers stuck in circular dependency!

**Solution:** Make P4 grab forks in opposite order (right then left), breaking the cycle.

### Question 6: Semaphore vs Condition Variable
**Q**: You're implementing a producer/consumer queue. Compare using:
- (A) Two semaphores (`empty` and `full`) with a mutex
- (B) Two condition variables (`empty` and `full`) with a mutex

What are the key differences in the code structure?

**A**:

**Approach A: Semaphores**
```c
sem_t empty, full, mutex;
sem_init(&empty, 0, MAX);  // Tracks resource count
sem_init(&full, 0, 0);
sem_init(&mutex, 0, 1);

void *producer(void *arg) {
    sem_wait(&empty);    // Wait for resource
    sem_wait(&mutex);    // Then lock
    put(i);
    sem_post(&mutex);
    sem_post(&full);     // Signal resource available
}
```

**Approach B: Condition Variables**
```c
cond_t empty, full;
mutex_t mutex;
int count = 0;  // NEED STATE VARIABLE!

void *producer(void *arg) {
    Pthread_mutex_lock(&mutex);     // Lock FIRST
    while (count == MAX)            // MUST use while loop
        Pthread_cond_wait(&empty, &mutex);  // Wait on condition
    put(i);
    count++;
    Pthread_cond_signal(&full);
    Pthread_mutex_unlock(&mutex);
}
```

**Key Differences:**

1. **State tracking:**
   - Semaphores: Value tracks resource count automatically
   - CVs: Need explicit state variable (`count`)

2. **Lock ordering:**
   - Semaphores: Wait for resource BEFORE acquiring mutex
   - CVs: Acquire mutex FIRST, then check condition

3. **Condition checking:**
   - Semaphores: Implicit (value >= 0)
   - CVs: Explicit while loop (MUST use while, not if)

4. **Simplicity:**
   - Semaphores: More concise, fewer lines
   - CVs: More explicit, clearer intent

5. **Flexibility:**
   - CVs: More flexible (can broadcast, complex predicates)
   - Semaphores: Simpler, but less expressive

### Question 7: Thread Throttling
**Q**: You have 100 threads, each needing to allocate 100MB of memory temporarily. Your system has 2GB of RAM. If all threads allocate simultaneously, the system will thrash. Write code using semaphores to limit to 10 concurrent allocations.

**A**:

```c
#include <semaphore.h>

#define NUM_THREADS 100
#define MAX_CONCURRENT_ALLOCS 10
#define ALLOC_SIZE_MB 100

sem_t throttle;

void *worker_thread(void *arg) {
    int id = (int)arg;

    // Phase 1: Work that doesn't need memory
    printf("Thread %d: doing work without memory\n", id);
    do_some_work();

    // Phase 2: Need to allocate memory - throttle here
    sem_wait(&throttle);  // Acquire one of 10 slots

    printf("Thread %d: acquired throttle slot\n", id);
    void *memory = malloc(ALLOC_SIZE_MB * 1024 * 1024);

    // Memory-intensive work
    printf("Thread %d: doing memory-intensive work\n", id);
    process_with_memory(memory);

    free(memory);
    sem_post(&throttle);  // Release slot
    printf("Thread %d: released throttle slot\n", id);

    // Phase 3: More work without memory
    do_more_work();

    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];

    // Initialize throttle semaphore to 10
    // This allows UP TO 10 threads in memory-intensive section
    sem_init(&throttle, 0, MAX_CONCURRENT_ALLOCS);

    // Create 100 threads
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, worker_thread, (void*)i);
    }

    // Wait for all
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    sem_destroy(&throttle);
    return 0;
}
```

**How it works:**
- Semaphore initialized to 10 (max concurrent allocations)
- First 10 threads: `sem_wait()` succeeds, value goes 10→9→8...→0
- Thread 11: `sem_wait()` blocks (value would go negative)
- When thread finishes and calls `sem_post()`, value goes 0→1, thread 11 wakes
- **Result:** Never more than 10 threads with allocated memory
- System uses max 10 × 100MB = 1GB (safe, no thrashing)

**Benefits:**
- Simple admission control
- Prevents resource exhaustion
- Allows other 90 threads to continue non-memory-intensive work

