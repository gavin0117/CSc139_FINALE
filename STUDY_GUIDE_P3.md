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

