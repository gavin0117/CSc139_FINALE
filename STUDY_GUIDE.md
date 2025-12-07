# Operating Systems Final Exam Study Guide

## Chapter 4: The Process Abstraction

### Key Terms & Definitions
- **Process**: A running program; the OS abstraction for execution
- **Program**: Lifeless bytes on disk (instructions + static data) waiting to run
- **Address Space**: Memory the process can address (code, stack, heap)
- **Machine State**: What a program can read/update when running:
  - **Memory** (address space)
  - **Registers** (PC/instruction pointer, stack pointer, frame pointer, general-purpose)
  - **I/O information** (list of open files)
- **Time Sharing**: OS shares CPU by running one process, stopping it, running another
- **PCB (Process Control Block)**: Data structure storing process information

### Process Creation Steps
1. **Load** code and static data from disk into memory (eager or lazy)
2. **Allocate** run-time stack (with argc/argv for main())
3. **Allocate** heap (small initially, grows via malloc())
4. **I/O setup**: Open 3 file descriptors (stdin, stdout, stderr)
5. **Jump to main()**: Start program execution

### Process States
1. **Running**: Executing instructions on CPU
2. **Ready**: Ready to run but OS chose not to run it
3. **Blocked**: Not ready until event occurs (e.g., waiting for I/O)
4. **Embryo** (initial): Being created
5. **Zombie** (final): Exited but not cleaned up; parent can check return code

### State Transitions
- Ready → Running: **Scheduled**
- Running → Ready: **Descheduled**
- Running → Blocked: **I/O: initiate**
- Blocked → Ready: **I/O: done**

### Practice Questions

1. What is the difference between a program and a process?

2. Draw the process state diagram showing Running, Ready, and Blocked states with transitions.

3. A process issues a disk read request. What state transition occurs?

4. What information must the OS track for each process? (Hint: think PCB)

5. Why can a process move from Running to Ready state?

6. What registers are critical to save during a context switch?

7. If a process is blocked waiting for keyboard input, can it transition directly to Running state? Why or why not?

8. What data structure does the OS use to track all processes?

9. Explain what happens when a timer interrupt occurs while a process is running.

10. A process completes an I/O operation. What state is it in before completion? After completion?

---

## Chapter 5: The Process API

### Key System Calls
- **fork()**: Creates a new child process (copy of parent)
  - Child doesn't start at main(), comes to life as if it called fork() itself
  - Returns **0** to child, **child's PID** to parent, **-1** on failure
  - Child gets copy of parent's address space and file descriptors

- **exec()**: Transforms calling process into a different program
  - Loads code and static data, overwrites current code segment
  - Re-initializes heap and stack
  - Does NOT create new process
  - Does NOT return on success (old program is completely replaced)
  - 6 variants on Linux: execl(), execlp(), execle(), execv(), execvp(), execvpe()

- **wait()**: Parent waits for child to complete
  - Makes output deterministic (prevents nondeterministic scheduling)
  - Prevents zombie processes

- **Nondeterminism**: CPU scheduler determines which process runs, making output order unpredictable without wait()

### File Descriptors and Redirection
- File descriptors remain **open across fork()** - child inherits parent's open files
- File descriptors remain **open across exec()** - enables I/O redirection
- UNIX searches for free file descriptors starting at **zero**
- Shell uses this to redirect stdin/stdout before exec()

### Separation of fork() and exec()
- Allows shell to set up environment between fork and exec
- Enables: I/O redirection, pipes, background jobs
- Shell can manipulate child's file descriptors before running new program

### Practice Questions

1. **What does fork() return to the parent process? To the child? On failure?**
   - **Answer**: Parent receives child's PID, child receives 0, failure returns -1

2. **After fork(), does the child start executing at main()? Where does it start?**
   - **Answer**: No, child comes to life as if it had called fork() itself, returning from fork() with value 0

3. **What happens to the heap, stack, and code segment when exec() is called successfully?**
   - **Answer**: Code segment is overwritten with new program code, heap and stack are re-initialized. The old program is completely replaced.

4. **Why do file descriptors remain open across exec()? How does this enable I/O redirection?**
   - **Answer**: Keeping file descriptors open allows the shell to redirect stdin/stdout before exec(). Shell closes stdin, opens file (gets fd 0), then exec() - new program inherits redirected I/O.

5. **Explain why the separation of fork() and exec() is useful for shells.**
   - **Answer**: Between fork() and exec(), the shell can modify the child's environment: set up pipes, redirect I/O, change working directory, etc., before running the new program.

6. **What is nondeterminism in process execution? How does wait() solve it?**
   - **Answer**: Nondeterminism means the scheduler decides which process runs when, making output order unpredictable. wait() forces parent to wait until child completes, making execution deterministic.

7. **What happens if you call exec() without fork() first?**
   - **Answer**: The calling process is replaced by the new program. Useful in shells, but means you lose the original program entirely.

---

## Chapter 6: Limited Direct Execution

### Core Concept
**Direct Execution**: Run program directly on CPU (fast but problematic)
**Limited Direct Execution**: OS restricts what programs can do via hardware support

The **two central challenges**: (1) **Performance** - implement virtualization without excessive overhead, and (2) **Control** - run processes efficiently while retaining control over the CPU.

### Two Main Problems
1. **Problem #1: Restricted Operations** - How to prevent user programs from doing whatever they want while still running efficiently?
2. **Problem #2: Switching Between Processes** - How does OS regain control of CPU?

### User Mode vs Kernel Mode (Protected Control Transfer)
- **User Mode**: Restricted; cannot issue I/O requests or privileged instructions. Process would raise exception if it tried.
- **Kernel Mode**: Full privileges; OS runs in this mode. Can issue I/O, execute all restricted instructions.
- **Trap**: Special instruction that **simultaneously** jumps into kernel **and** raises privilege level to kernel mode
- **Return-from-trap**: Special instruction that **simultaneously** returns to user program **and** reduces privilege level back to user mode
- **Hardware saves registers** onto **per-process kernel stack** during trap

### Trap Table and System Calls
- **Trap table** set up by OS **at boot time** (when machine boots in privileged/kernel mode)
- OS informs hardware where trap handlers are located (syscall handler, timer handler, etc.)
- Hardware remembers these locations until machine is rebooted
- User code **cannot specify exact address** to jump to (security risk!)
- Instead, **system-call number** is placed in register or stack
- OS examines this number and executes corresponding code
- This level of indirection provides **protection**

### Two Phases of Limited Direct Execution Protocol

**Phase 1 @ Boot (kernel mode)**:
- Initialize trap table (tell hardware where syscall handler, timer handler are)
- Start timer interrupt

**Phase 2 @ Run (kernel mode → user mode → kernel mode)**:
- OS creates process entry, allocates memory, loads program, sets up user stack with argv
- OS fills kernel stack with reg/PC
- OS executes **return-from-trap** to start process in user mode
- Process runs in user mode
- Process traps back into OS (via system call or timer interrupt)
- OS handles trap, then return-from-trap back to process

### Regaining Control: Cooperative vs Non-Cooperative

**Cooperative Approach** (old Mac OS, Xerox Alto):
- OS **waits** for system calls or illegal operations
- Process voluntarily gives up CPU via system calls or yield()
- Problem: Infinite loop → must reboot machine!

**Non-Cooperative Approach** (modern systems):
- **Timer interrupt** - hardware timer programmed to raise interrupt every X milliseconds
- When interrupt raised, currently running process is halted
- Hardware saves registers, OS interrupt handler runs
- OS regains control and can switch to different process
- This is a **privileged operation** (starting/stopping timer)

### Context Switch - TWO Types of Register Saves

A context switch involves **two different types** of register saves/restores:

1. **When timer interrupt occurs (hardware-driven)**:
   - **User registers** of running process are **implicitly saved by hardware**
   - Saved onto that process's **kernel stack**
   - Hardware does this automatically

2. **When OS decides to switch from A to B (software-driven)**:
   - **Kernel registers** are **explicitly saved by software (OS)**
   - Saved into **process structure** in memory (not stack)
   - OS saves registers, PC, kernel stack pointer of process A
   - OS restores registers, PC, kernel stack pointer of process B
   - **By switching stacks**, kernel enters switch() in context of A, returns in context of B
   - Return-from-trap then restores B and starts running it

### Practice Questions

1. **What are the two main problems that Limited Direct Execution solves? What is the solution to each?**
   - **Answer**: (1) Restricted Operations - solved using user mode/kernel mode and trap mechanism. (2) Switching Between Processes - solved using timer interrupt to regain control.

2. **Explain what happens during a trap instruction. What two things does it do simultaneously?**
   - **Answer**: Trap simultaneously (1) jumps into the kernel at a pre-specified location in the trap table, and (2) raises the privilege level from user mode to kernel mode. Hardware also saves registers onto the per-process kernel stack.

3. **Why can't user code specify an exact address to jump to when making a system call? What mechanism is used instead?**
   - **Answer**: Allowing exact addresses would let malicious code jump anywhere in kernel (e.g., past permission checks), taking over the machine. Instead, user code places a system-call number in a register, and the OS uses the trap table to find the corresponding handler. This indirection provides protection.

4. **Describe the two phases of the Limited Direct Execution protocol (boot time vs run time).**
   - **Answer**: Boot time (kernel mode): OS initializes trap table, tells hardware where handlers are, starts timer. Run time: OS sets up process and uses return-from-trap to start it in user mode. Process runs, then traps back via syscall or timer interrupt. OS handles it and returns control.

5. **What is the difference between cooperative and non-cooperative approaches to regaining CPU control? Why is the non-cooperative approach necessary?**
   - **Answer**: Cooperative: OS waits for process to make system call or illegal operation. Problem: infinite loop means reboot. Non-cooperative: Timer interrupt forcibly interrupts process every X milliseconds, giving OS control. This prevents rogue processes from taking over the machine.

6. **During a context switch, there are TWO different types of register saves/restores. What are they and who performs each?**
   - **Answer**: (1) When timer interrupt occurs: hardware implicitly saves user registers of running process onto its kernel stack. (2) When OS switches from process A to B: software (OS) explicitly saves kernel registers into process structure of A, then restores kernel registers from process structure of B.

7. **How does switching kernel stacks enable a context switch? Explain the flow.**
   - **Answer**: OS calls switch() routine while in context of process A (on A's kernel stack). switch() saves A's registers to A's process structure, restores B's registers from B's process structure, and switches to B's kernel stack. Now executing on B's stack. When switch() returns, it returns in context of B. Return-from-trap then resumes B in user mode.

---

## Chapter 7 & 8: Scheduling

### Key Metrics
- **Turnaround Time**: T_turnaround = T_completion - T_arrival (performance metric)
- **Response Time**: T_response = T_firstrun - T_arrival (interactivity metric)
- **Trade-off**: Performance vs Fairness often at odds in scheduling

### Basic Scheduling Algorithms

**FIFO (First In First Out)**:
- Run jobs in order of arrival
- Simple, easy to implement
- **Problem**: **Convoy effect** - short jobs stuck behind long job (like cars behind slow truck)

**SJF (Shortest Job First)**:
- Run shortest job first (non-preemptive)
- Optimal for turnaround time when all jobs arrive simultaneously
- **Problem**: Still suffers convoy effect if short jobs arrive after long job starts

**STCF (Shortest Time-to-Completion First / Preemptive SJF)**:
- Preempts currently running job when shorter job arrives
- Any time new job enters, scheduler picks job with least time remaining
- Optimal for turnaround time
- **Problem**: Terrible for response time

**Round Robin (RR)**:
- Runs each job for time slice (scheduling quantum), then switches
- Time slice must be multiple of timer-interrupt period
- **Trade-off**: Shorter time slice = better response time BUT more context-switch overhead
- **Amortization**: Longer time slices amortize context-switch cost (e.g., 1ms context switch + 100ms time slice = <1% overhead)
- Excellent for response time
- **Problem**: Terrible for turnaround time (stretches out every job)

### Incorporating I/O
- When job performs I/O, it's **blocked** waiting for completion
- Scheduler should schedule another job during I/O
- Treat each CPU burst as independent job
- **Overlap**: CPU used by one process while waiting for I/O of another → better utilization

### MLFQ (Multi-Level Feedback Queue)

**Goal**: Schedule without perfect knowledge - optimize both turnaround time AND response time

**Core Concept**: **Learn from history** to predict future behavior
- Multiple queues, each with different priority level
- Job on single queue at any time
- MLFQ varies priority based on **observed behavior**
  - Job repeatedly gives up CPU (interactive) → keep priority high
  - Job uses CPU intensively → reduce priority

**Five Rules of MLFQ**:
1. If Priority(A) > Priority(B), A runs (B doesn't)
2. If Priority(A) = Priority(B), A & B run in RR
3. When job enters system, placed at **highest priority**
4. Once job uses up **time allotment** at given level (regardless of how many times it gave up CPU), priority reduced
5. After time period **S**, move **all** jobs to topmost queue (**priority boost**)

**Allotment**: Amount of time job can spend at given priority level before scheduler reduces priority

**Three Problems MLFQ Solves**:
1. **Starvation**: Too many interactive jobs → long-running jobs never run
   - **Solution**: Rule 5 (priority boost) guarantees all jobs make progress
2. **Gaming**: Smart user issues I/O just before allotment expires to stay at high priority
   - **Solution**: Rule 4 - better accounting tracks total time used at level
3. **Changing behavior**: CPU-bound job becomes interactive
   - **Solution**: Rule 5 allows job to regain high priority

**Tuning MLFQ** ("voo-doo constants" - Ousterhout's Law):
- How many queues?
- Time slice per queue?
- Allotment size?
- Boost period S?
- **Varying time slices**: High-priority queues get short slices (10ms), low-priority get longer slices (100s ms)

**Real Implementations**:
- Solaris: 60 queues, 20ms-few hundred ms slices, boost every ~1 second
- BSD UNIX, Windows NT: use MLFQ variants

### Practice Questions with Answers

1. **What is the convoy effect? Give an example showing how SJF avoids it.**
   - **Answer**: Convoy effect is when short jobs get stuck behind a long job, severely increasing average turnaround time. Example: Jobs A(100ms), B(10ms), C(10ms) arrive at t=0. FIFO: Avg turnaround = (100+110+120)/3 = 110ms (B and C wait for A). SJF runs B, C, A: Avg turnaround = (10+20+120)/3 = 50ms (more than 2× better).

2. **Explain the fundamental trade-off between turnaround time and response time. Which schedulers optimize each?**
   - **Answer**: Turnaround time (job completion) vs response time (interactivity) are at odds. To minimize turnaround, run shortest jobs first (unfair) - SJF/STCF optimize this. To minimize response, give all jobs quick turns (fair but stretches completion) - Round Robin optimizes this. Any fair scheduler hurts turnaround; any unfair scheduler hurts response.

3. **Jobs A(100ms) arrives at t=0, B(10ms) and C(10ms) arrive at t=10. Calculate average turnaround for STCF.**
   - **Answer**:
     - t=0-10: A runs (90ms left)
     - t=10: B, C arrive and preempt A (shorter remaining time)
     - t=10-20: B runs, completes (turnaround = 20-10 = 10ms)
     - t=20-30: C runs, completes (turnaround = 30-10 = 20ms)
     - t=30-130: A finishes (turnaround = 130-0 = 130ms)
     - **Average = (130+10+20)/3 = 53.33ms**

4. **In Round Robin, why does choosing time slice length involve a trade-off? What is amortization?**
   - **Answer**: Shorter time slice = better response time (jobs run sooner) but more context-switch overhead wastes CPU. Longer time slice = amortizes context-switch cost across more work but hurts responsiveness. **Amortization**: Spreading fixed cost (context switch) over more work. Example: 1ms switch + 100ms slice = <1% overhead; 1ms switch + 10ms slice = 10% overhead.

5. **How does MLFQ learn from history? Describe how it treats a new job vs a CPU-bound job.**
   - **Answer**: MLFQ observes job behavior and adjusts priority accordingly. **New job**: Placed at highest priority (Rule 3), assuming it might be short/interactive. If it uses full time slices, priority drops (Rule 4), revealing it as CPU-bound. Eventually settles at low priority. **CPU-bound**: Stays at low priority with long time slices. This way MLFQ approximates SJF without knowing job lengths in advance.

6. **Explain how a malicious user could game early MLFQ (without Rule 4 better accounting). How does Rule 4 prevent this?**
   - **Answer**: **Gaming**: Issue I/O just before time slice expires (e.g., at 99% of slice). Old rules let job stay at high priority because it "voluntarily" gave up CPU. Repeat to monopolize CPU at high priority. **Rule 4 fix**: Tracks total time used at priority level. Whether job uses allotment in one burst or many small pieces doesn't matter - once allotment exhausted, priority drops. Can't game the system.

7. **Why does MLFQ need Rule 5 (priority boost)? What is a "voo-doo constant" and give an example from MLFQ.**
   - **Answer**: **Rule 5 prevents**: (1) Starvation - ensures long-running jobs get some CPU time even with many interactive jobs, (2) Handles behavior changes - CPU-bound job that becomes interactive gets another chance at high priority. **Voo-doo constant**: Parameter requiring "black magic" to set correctly (Ousterhout's Law). Example: boost period **S** - too high → starvation; too low → interactive jobs don't get fair CPU. No perfect formula, requires experimentation/tuning.

---

## Chapter 9: Stride Scheduling (Basics)

### Core Concept
**Proportional-share scheduling**: Each job gets a proportion of CPU time based on its **tickets**
- More tickets = more CPU time
- **Stride** = large number / tickets (e.g., 10000 / tickets)
- Lower stride = runs more often

### Algorithm
1. Each process has a **pass value** (starts at 0)
2. Pick process with lowest pass value
3. Run it for a time slice
4. Increment its pass value by its stride
5. Repeat

### Example Calculation
```
Process A: 100 tickets, stride = 10000/100 = 100
Process B: 50 tickets,  stride = 10000/50 = 200
Process C: 250 tickets, stride = 10000/250 = 40

Initial: A=0, B=0, C=0
Step 1: Run C (lowest), C=40
Step 2: Run C, C=80
Step 3: Run C, C=120
Step 4: Run A, A=100
Step 5: Run A, A=200
Step 6: Run C, C=160
```

### Practice Questions with Answers

**Q1: Three processes have tickets: A=200, B=100, C=100. What percentage of CPU does each get?**

**Answer:**
- Total tickets = 200+100+100 = 400
- A gets: 200/400 = **50%**
- B gets: 100/400 = **25%**
- C gets: 100/400 = **25%**

**Q2: Given stride=10000/tickets, calculate strides for A(100 tickets), B(200 tickets), C(50 tickets). Which runs most frequently?**

**Answer:**
- A: 10000/100 = **100**
- B: 10000/200 = **50**
- C: 10000/50 = **200**
- **B runs most frequently** (smallest stride means pass value increases slowest, so it gets selected more often)

**Q3: Why is stride scheduling deterministic while lottery scheduling is randomized?**

**Answer:** Stride scheduling uses a deterministic algorithm (always pick lowest pass value) to achieve proportional sharing. Lottery scheduling randomly picks a ticket, which only achieves proportional sharing on average over many iterations. Stride provides more precise guarantees.

**Q4: Two processes: A(100 tickets, stride=100), B(50 tickets, stride=200). Both start with pass=0. Show first 5 scheduling decisions.**

**Answer:**
```
Initial: A=0, B=0
1. Tie→pick A: A=100, B=0
2. Run B: A=100, B=200
3. Run A: A=200, B=200
4. Tie→pick A: A=300, B=200
5. Run B: A=300, B=400
```
Result: A runs 3 times, B runs 2 times (A has 2× tickets, runs ~2× more)

**Q5: What problem occurs when a new process joins in stride scheduling?**

**Answer:** The new process starts with pass=0, which is likely much lower than existing processes' pass values. This causes the new process to monopolize the CPU until its pass value catches up. Solution: set new process's pass to minimum of existing pass values.

**Q6: Calculate: If A has 75 tickets and B has 25 tickets, after 100 time slices, approximately how many does each get?**

**Answer:**
- A: 75/100 = 75% → approximately **75 time slices**
- B: 25/100 = 25% → approximately **25 time slices**
- Stride scheduling guarantees this proportion deterministically

---

## Chapter 13: Address Spaces

### Evolution of Memory Models

1. **Early Systems (No Abstraction)**:
   - OS at physical address 0 (just a library)
   - One program at physical address 64KB, uses rest of memory
   - No protection, direct physical memory access

2. **Multiprogramming**:
   - Multiple processes **ready to run** at a given time
   - OS switches between them (e.g., when one does I/O)
   - **Goal**: Increase CPU utilization/efficiency

3. **Time Sharing**:
   - Multiple users wanting **interactivity** and timely response
   - Processes **stay in memory** (not swapped to disk each switch)
   - **Problem**: Protection becomes critical - need to prevent processes from accessing each other's memory

4. **Address Space Abstraction**:
   - Each process has its own **virtual memory view**
   - Program thinks it starts at address 0 and has large address space
   - Reality: loaded at arbitrary physical address(es)

### The Crux of Memory Virtualization

**How can the OS build this abstraction of a private, potentially large address space for multiple running processes (all sharing memory) on top of a single, physical memory?**

### Address Space Structure
```
Virtual Memory Layout:
0KB   ┌──────────────┐
      │   Code       │  (Program instructions - static)
      ├──────────────┤
      │   Heap       │  (malloc data, grows DOWN→)
      │      ↓       │
      │              │
      │      ↑       │
      │   Stack      │  (local vars, grows UP←)
16KB  └──────────────┘
```

### Three Goals of Virtual Memory

1. **Transparency**:
   - VM should be **invisible** to the running program
   - Program behaves as if it has its own private physical memory
   - OS and hardware do all the work behind the scenes

2. **Efficiency**:
   - Make virtualization efficient in **time** (not making programs run much slower)
   - Make virtualization efficient in **space** (not using too much memory for structures)
   - Requires hardware support (e.g., TLBs for fast translation)

3. **Protection/Isolation**:
   - Protect processes from one another
   - Protect OS from processes
   - Process cannot access or affect memory outside its address space
   - Each process runs in its own isolated cocoon

### Key Concepts
- **Virtual Address**: What program sees (0KB to 16KB) - **every address you see as a programmer is virtual**
- **Physical Address**: Actual location in RAM where data really lives
- **Address Translation**: Hardware+OS maps virtual → physical
- **Isolation**: Each process thinks it owns all memory (0 to max)

### Practice Questions with Answers

**Q1: What are the three goals of a virtual memory system and why is each important?**

**Answer:**
1. **Transparency**: The OS should implement VM in a way that is **invisible** to the running program. The program shouldn't be aware that memory is virtualized; it behaves as if it has its own private physical memory.

2. **Efficiency**: The OS should make virtualization efficient in both **time** (not making programs run much more slowly) and **space** (not using too much memory for VM structures). Time-efficient virtualization requires hardware support like TLBs.

3. **Protection**: The OS must protect processes from one another and protect itself from processes. When a process performs a load, store, or instruction fetch, it should not be able to access memory outside its address space. This enables **isolation** - each process runs safely, isolated from other faulty or malicious processes.

**Q2: A process has virtual addresses 0-16KB. Its code is at physical address 32KB. What physical address corresponds to virtual address 4KB?**

**Answer:**
- Base physical address = 32KB
- Virtual address = 4KB
- Physical address = 32KB + 4KB = **36KB**
- This assumes simple base-and-bounds relocation

**Q3: What are the three segments typically in an address space and why are stack and heap at opposite ends?**

**Answer:**
- **Code**: Program instructions (static size)
- **Heap**: Dynamic allocation (malloc), grows downward
- **Stack**: Function calls, local variables, grows upward
- They're at opposite ends so both can grow without immediately colliding. This maximizes usable space before needing more memory.

**Q4: Explain the difference between multiprogramming and time sharing. Why did time sharing make protection critical?**

**Answer:**
- **Multiprogramming**: Multiple processes are **ready to run** at a given time. The OS switches between them (e.g., when one performs I/O) to increase **CPU utilization**. This was about efficiency.

- **Time Sharing**: Multiple users **concurrently** using a machine, each expecting **timely, interactive** response. Processes **stay in memory** while switching (not swapped to disk each time), making switching fast.

- **Why protection became critical**: With time sharing, multiple programs **reside concurrently in memory**. Without protection, one process could read or write another process's memory, causing security/stability issues. This necessitated memory isolation via address spaces.

**Q5: What does the OS need to set up when creating a process's address space?**

**Answer:**
- Allocate physical memory for code, heap, and stack
- Load program code and static data from disk
- Set up page table or segmentation registers for address translation
- Initialize heap and stack regions
- Mark memory regions with permissions (code=read/execute, heap/stack=read/write)

**Q6: Draw the address space for a process where code=4KB, heap currently=2KB, stack currently=2KB, total space=16KB.**

**Answer:**
```
0KB   ┌──────────────┐
      │   Code (4KB) │
4KB   ├──────────────┤
      │   Heap (2KB) │
6KB   ├──────────────┤
      │              │
      │   (free)     │
      │              │
14KB  ├──────────────┤
      │  Stack (2KB) │
16KB  └──────────────┘
```

**Q7: You write a C program that prints a pointer to a heap-allocated variable and prints the address of main(). Are these virtual or physical addresses? Where do the values actually live in physical memory?**

**Answer:**
- **Every address you see as a programmer is virtual**. When you print a pointer in a C program, you're printing a virtual address.
- The addresses printed (e.g., 0x1095afe50 for code, 0x1096008c0 for heap) are virtual addresses that give the illusion of where things are laid out.
- **Only the OS and hardware know the real physical addresses**. The program might think it starts at virtual address 0, but it's actually loaded at some arbitrary physical address like 320KB.
- The OS, with hardware support, translates every virtual address the program uses into the corresponding physical address to fetch/store the actual data.

---

## Chapter 14: Memory API

### The Crux: How To Allocate And Manage Memory

**In UNIX/C programs, understanding how to allocate and manage memory is critical in building robust and reliable software. What interfaces are commonly used? What mistakes should be avoided?**

### Two Types of Memory

**1. Stack Memory (Automatic Memory)**:
- Allocations and deallocations managed **implicitly** by the compiler
- Declaration: `int x;` inside a function
- **Compiler handles everything**: Makes space on call, frees on return
- **Limitation**: Data disappears when function returns
- **Use**: Local variables, function parameters

**2. Heap Memory**:
- Allocations and deallocations **explicitly** handled by programmer
- Uses `malloc()` to allocate, `free()` to deallocate
- **Heavy responsibility**, cause of many bugs
- **Advantage**: Long-lived memory that persists beyond function call
- **Use**: Dynamic data structures, memory needed across function calls

### The malloc() Call

**Signature**: `void *malloc(size_t size);`

**Usage**:
```c
#include <stdlib.h>

int *x = (int *) malloc(sizeof(int));  // Allocate integer on heap
```

**Key points**:
- Pass size in bytes using `sizeof()` operator
- Returns pointer to allocated memory (or NULL on failure)
- Returns `void *` (generic pointer), typically cast to appropriate type
- **Library call**, not system call (built on top of brk/sbrk/mmap)
- **Always check for NULL** to handle allocation failures

**Important idioms**:
- Integers: `int *x = malloc(sizeof(int));`
- Arrays: `int *arr = malloc(10 * sizeof(int));`
- Strings: `char *s = malloc(strlen(src) + 1);` (the +1 is for null terminator!)
- Structs: `struct node *n = malloc(sizeof(struct node));`

**Common mistake with sizeof()**:
```c
int *x = malloc(10 * sizeof(int));  // Array of 10 integers
printf("%zu\n", sizeof(x));  // Prints 4 or 8 (size of pointer!)
                             // NOT 40 (size of allocated memory)
```

### The free() Call

**Signature**: `void free(void *ptr);`

**Usage**:
```c
int *x = malloc(10 * sizeof(int));
// ... use x ...
free(x);  // Free the memory
```

**Key points**:
- Takes **only** the pointer returned by malloc() (no size argument)
- Size tracked internally by memory allocation library
- **Must** free memory when done (or memory leak occurs)
- Knowing **when**, **how**, and **if** to free is the hard part

### Common Errors (The Deadly Seven)

**1. Forgetting to Allocate Memory**
```c
char *src = "hello";
char *dst;  // OOPS! Not allocated
strcpy(dst, src);  // Segmentation fault!
```
**Fix**: `char *dst = malloc(strlen(src) + 1);`

**2. Not Allocating Enough Memory (Buffer Overflow)**
```c
char *src = "hello";
char *dst = malloc(strlen(src));  // TOO SMALL! (missing +1)
strcpy(dst, src);  // Writes past allocated space
```
- Can cause security vulnerabilities
- May appear to work but is incorrect
- **Always allocate enough**: `malloc(strlen(src) + 1)` for strings

**3. Forgetting to Initialize Allocated Memory**
```c
int *arr = malloc(10 * sizeof(int));
// Forgot to initialize!
printf("%d\n", arr[0]);  // Uninitialized read - unknown value
```
- Results in **uninitialized reads**
- **Fix**: Initialize after allocating, or use `calloc()` which zeros memory

**4. Forgetting to Free Memory (Memory Leak)**
```c
void func() {
    int *x = malloc(sizeof(int));
    // ... use x ...
    // OOPS! Forgot to call free(x)
}  // Memory leaked when function returns
```
- In **long-running programs** (servers, OS): eventually runs out of memory
- In **short programs**: OS reclaims all memory when process exits
- **Best practice**: Always free what you allocate (develop good habits!)

**5. Freeing Memory Before Done (Dangling Pointer)**
```c
int *x = malloc(sizeof(int));
free(x);
*x = 10;  // OOPS! Using freed memory (dangling pointer)
```
- Can crash or corrupt memory
- Free might let malloc() reuse that memory for something else

**6. Freeing Memory Repeatedly (Double Free)**
```c
int *x = malloc(sizeof(int));
free(x);
free(x);  // OOPS! Double free
```
- Result is **undefined** (crashes common)
- Memory allocation library gets confused

**7. Calling free() Incorrectly (Invalid Free)**
```c
int *x = malloc(10 * sizeof(int));
free(x + 5);  // OOPS! Not the pointer malloc returned!
```
- **Must** pass exact pointer returned by malloc()
- Passing any other value → undefined behavior

### Underlying OS Support

**malloc() and free() are library calls**, not system calls. The memory allocation library manages space within your virtual address space, built on top of system calls:

**brk / sbrk**:
- Change location of program's **break** (end of heap)
- `brk(addr)`: Set break to new address
- `sbrk(increment)`: Increment break by given amount
- **Never call directly** - used internally by malloc library

**mmap()**:
- Create **anonymous memory region** (not associated with file)
- Can be treated like heap and managed
- Provides alternative to brk/sbrk for memory allocation
- Understanding basic role sufficient (not detailed parameters)

**Two levels of memory management**:
1. **OS level**: Gives memory to processes, reclaims when process exits
2. **Library level**: malloc/free manage heap within process

### Other Useful Calls

**calloc()**:
- Allocates memory **and zeroes it** before returning
- Prevents uninitialized read errors
- `int *arr = calloc(10, sizeof(int));` // 10 integers, all set to 0

**realloc()**:
- Resize previously allocated memory
- Makes new larger region, copies old data, returns new pointer
- `arr = realloc(arr, 20 * sizeof(int));` // Grow from 10 to 20 elements

### Practice Questions with Answers

**Q1: What is the difference between stack memory and heap memory? Give an example of when you would use each.**

**Answer:**
**Stack memory (automatic)**:
- Managed **implicitly** by compiler
- Allocated on function call, freed on return
- Example: `int x;` inside function
- **Use when**: Local variables, temporary data that doesn't need to persist beyond function

**Heap memory**:
- Managed **explicitly** by programmer with malloc()/free()
- Persists until explicitly freed
- Example: `int *x = malloc(sizeof(int));`
- **Use when**: Data must persist beyond function call, size unknown at compile time, large data structures

**Key difference**: Stack data **disappears** when function returns; heap data **persists** until freed.

**Q2: What is wrong with this code? How would you fix it?**
```c
char *src = "hello";
char *dst = malloc(strlen(src));
strcpy(dst, src);
```

**Answer:**
**Problem**: **Buffer overflow** - allocated only 5 bytes but "hello" needs 6 bytes (5 characters + 1 null terminator).

**Fix**:
```c
char *dst = malloc(strlen(src) + 1);  // +1 for null terminator!
strcpy(dst, src);
```

The `+1` is critical for the null terminator `'\0'` that marks the end of C strings. Without it, strcpy writes one byte past the allocated space.

**Q3: Explain the memory leak in this code. Is it a problem if the program runs for 1 second? What if it runs for 1 year?**
```c
void process_data() {
    int *data = malloc(1000 * sizeof(int));
    // ... process data ...
    // Forgot to call free(data)
}

int main() {
    while (1) {
        process_data();  // Called repeatedly
    }
}
```

**Answer:**
**Memory leak**: Each call to `process_data()` allocates 4KB (1000 × 4 bytes) but never frees it. The allocated memory is lost when the function returns.

**Short program (1 second)**: Not a major problem. When the process exits, the **OS reclaims all memory** (both leaked and properly managed). No memory is truly "lost" to the system.

**Long-running program (1 year)**: **Huge problem!** This server/daemon will continuously leak 4KB per call. Eventually runs out of memory and crashes. This is why **long-running programs** (web servers, databases, operating systems) must be extremely careful about memory leaks.

**Fix**: Add `free(data);` before returning from `process_data()`.

**Q4: What happens when you call free() on a pointer that was not returned by malloc()? What about calling free() twice on the same pointer?**

**Answer:**
**Invalid free** (pointer not from malloc):
```c
int x = 5;
free(&x);  // WRONG! x is on stack, not heap
```
- **Undefined behavior** - can crash, corrupt memory allocator state
- Must **only** pass pointers returned by malloc() to free()

**Double free**:
```c
int *x = malloc(sizeof(int));
free(x);
free(x);  // WRONG! Already freed
```
- **Undefined behavior** - memory allocator gets confused
- Common outcome: **crash**
- Allocator might try to add same memory to free list twice

**Q5: Why do we use sizeof(type) instead of hardcoding sizes like 4 or 8 when calling malloc()?**

**Answer:**
**Portability**: Type sizes vary across architectures
- 32-bit system: `sizeof(int)` = 4, `sizeof(int *)` = 4
- 64-bit system: `sizeof(int)` = 4, `sizeof(int *)` = 8

**Example problem**:
```c
int *x = malloc(4);  // BAD! Hardcoded size
```
- Works on 32-bit, but pointer is 8 bytes on 64-bit → too small!

**Correct approach**:
```c
int *x = malloc(sizeof(int));  // GOOD! Always correct size
```

Using `sizeof()` makes code **portable** across different architectures and resilient to type changes.

**Q6: What is a dangling pointer and how can it cause problems?**

**Answer:**
**Dangling pointer**: A pointer to memory that has been freed

**Example**:
```c
int *x = malloc(sizeof(int));
*x = 42;
free(x);  // Memory freed
printf("%d\n", *x);  // Dangling pointer dereference!
```

**Problems**:
1. **Undefined behavior** - might print 42, might print garbage, might crash
2. **Use-after-free**: If malloc() reuses that memory for something else, you're now reading/writing someone else's data
3. **Security vulnerabilities**: Attackers can exploit use-after-free bugs

**Prevention**:
- Set pointer to NULL after freeing: `free(x); x = NULL;`
- Don't use pointers after freeing the memory they point to

**Q7: Explain the relationship between malloc()/free() and the OS. Are they system calls?**

**Answer:**
**malloc()/free() are NOT system calls** - they are **library calls** (part of C standard library).

**How it works**:
1. **Library level**: malloc()/free() manage the **heap** within your process's virtual address space
   - Track which regions are allocated/free
   - Satisfy allocation requests from existing heap space when possible

2. **OS level**: When library needs more memory, it calls OS system calls:
   - **brk/sbrk**: Grow the heap by moving the program break
   - **mmap()**: Create new memory regions

3. **When process exits**: OS reclaims **all** process memory (code, stack, heap) regardless of whether you called free()

**Why this matters**:
- malloc/free are **fast** (no system call overhead most of the time)
- OS system calls only needed occasionally when heap grows
- Two-level management: library handles small allocations efficiently, OS handles process-level memory

---

## Chapter 15: Address Translation

### The Crux: How to Efficiently and Flexibly Virtualize Memory

**How can we build an efficient virtualization of memory? How do we provide the flexibility needed by applications? How do we maintain control over which memory locations an application can access? How do we do all of this efficiently?**

### Hardware-Based Address Translation

- **Address translation**: Hardware transforms each memory access (instruction fetch, load, store)
- Changes **virtual address** (from program) → **physical address** (where data actually is)
- Happens on **every memory reference** performed by hardware
- OS sets up hardware correctly, then hardware does translation with no OS intervention
- **Interposition**: Hardware interposes on each memory access to translate addresses transparently

### Base and Bounds (Dynamic Relocation)

**Introduced in late 1950s time-sharing machines**

**Assumptions** (simplified):
1. Address space placed **contiguously** in physical memory
2. Address space size **< physical memory** size
3. All address spaces are **exactly the same size**

**Hardware Registers (per-CPU, part of MMU):**
- **Base register**: Start of physical memory for process
- **Bounds register** (also called limit): Size of address space

**Translation Formula:**
```
physical_address = virtual_address + base
if (virtual_address >= bounds) → EXCEPTION (segmentation fault)
```

### Visual: Address Translation
```
Virtual Address Space (Process view):
0KB   ┌──────────┐
      │   Code   │
      │   Heap   │
      │   Stack  │
16KB  └──────────┘

              ↓ Translation (base=32KB)

Physical Memory (actual RAM):
0KB   ┌──────────┐
      │  OS      │
32KB  ├──────────┤  ← Base register points here
      │ Process  │
      │  Code    │
      │  Heap    │
      │  Stack   │
48KB  └──────────┘  ← Base + Bounds
```

### Hardware vs OS Responsibilities

**Hardware (MMU):**
- Performs translation on every memory access
- Checks bounds on every access
- Raises exception if out of bounds

**OS:**
- **On process creation**: Find space for address space using **free list** (data structure tracking free physical memory ranges)
- **On process termination**: Reclaim memory, add back to free list
- **On context switch**: Save/restore base and bounds registers to/from PCB (Process Control Block)
- **Exception handling**: Terminate processes that access out-of-bounds memory

### Key Problem: Internal Fragmentation

**Base and bounds wastes memory** due to **internal fragmentation**:
- Process allocated from base to base+bounds (e.g., 32KB to 48KB = 16KB)
- But only **code, heap, stack** are actually used
- **Large gap between heap and stack** is wasted but still allocated
- This unused space **inside** the allocated unit is "fragmented" → internal fragmentation
- Solution: **Segmentation** (next chapter) - separate base/bounds for each segment

### Practice Questions with Answers

**Q1: Base=32KB, Bounds=16KB. Translate virtual addresses: (a) 0KB, (b) 4KB, (c) 20KB**

**Answer:**
- (a) 0KB: Physical = 0 + 32KB = **32KB** ✓
- (b) 4KB: Physical = 4KB + 32KB = **36KB** ✓
- (c) 20KB: 20KB >= 16KB bounds → **EXCEPTION (out of bounds)** ✗

**Q2: A process tries to access virtual address 8000. Base=16384, Bounds=16384. What happens?**

**Answer:**
- Check bounds: 8000 < 16384 ✓ (within bounds)
- Physical address = 8000 + 16384 = **24384**
- Access succeeds at physical address 24384

**Q3: What happens during a context switch with respect to base and bounds registers?**

**Answer:**
1. OS saves current process's base and bounds to its PCB
2. OS loads next process's base and bounds from its PCB into hardware registers
3. Now all address translations use new process's base/bounds
4. This happens in **kernel mode** (privileged operation)

**Q4: Why must base and bounds registers be privileged (kernel-mode only)?**

**Answer:** If user programs could modify base/bounds, they could change their address translation to access any physical memory, including other processes' memory or OS memory. This would break isolation and security.

**Q5: In x86, what is the role of registers like eax, ebx, esp?**

**Answer:**
- **eax, ebx, ecx, edx**: General-purpose registers for computation
- **esp**: Stack pointer (points to top of stack)
- **ebp**: Base pointer (stack frame base)
- **eip**: Instruction pointer (program counter)
These contain virtual addresses that get translated by the MMU.

**Q6: Given 16-bit virtual addresses and base=40000, what is the maximum physical address this process can access?**

**Answer:**
- 16-bit address space = 2^16 = 65536 bytes = 64KB
- Max virtual address = 64KB - 1
- Max physical address = (64KB - 1) + 40000 = **105535** (or ~103KB)

**Q7: Why is this called "dynamic relocation" instead of "static relocation"?**

**Answer:**
- **Static relocation**: Rewrite program's addresses when loaded (done once by loader)
- **Dynamic relocation**: Translate addresses at runtime using hardware registers
- Dynamic is better because the process can be moved in physical memory (just change base register) without modifying the program itself.

---

## Chapter 16: Segmentation

### The Crux: How to Support a Large Address Space

**How do we support a large address space with (potentially) a lot of free space between the stack and the heap?** A 32-bit address space is 4GB, but a typical program only uses megabytes. Base and bounds would waste physical memory on the unused gap.

### Problem with Base & Bounds: Internal Fragmentation
- Wastes memory (internal fragmentation)
- If heap and stack are small, large gap between them is unused but still allocated
- Entire address space must be contiguous in physical memory
- Not flexible enough for **sparse address spaces**

### Segmentation = Generalized Base & Bounds

**Old idea from early 1960s**: Instead of one base/bounds pair for entire address space, have base/bounds pair **per logical segment**.

- **Segment**: Contiguous portion of address space of particular length
- Typical segments: **code, heap, stack**
- Each segment can be placed **independently** in physical memory
- Only **used** memory is allocated → avoids internal fragmentation between heap and stack

### Segment Registers (per-process)
```
Segment   Base    Bounds   Grows
Code      32KB    2KB      —
Heap      34KB    3KB      Positive
Stack     28KB    2KB      Negative (!)
```

### Address Translation with Segmentation
```
1. Extract segment from virtual address (top 2 bits)
2. Extract offset within segment (remaining bits)
3. Check: offset < bounds? (account for growth direction)
4. Physical address = base[segment] + offset
```

### Visual: Segmentation
```
Virtual Address Space:          Physical Memory:
┌─────────────┐                ┌─────────────┐ 0KB
│ Code  (2KB) │ ───────┐      │   OS        │
├─────────────┤        │      ├─────────────┤ 16KB
│ Heap  (3KB) │ ────┐  └─────→│ Code  (2KB) │ 32KB
├─────────────┤     │         ├─────────────┤ 34KB
│   (free)    │     └────────→│ Heap  (3KB) │
├─────────────┤               ├─────────────┤ 37KB
│ Stack (2KB) │ ──────┐       │   (free)    │
└─────────────┘       │       ├─────────────┤ 26KB
                      └──────→│ Stack (2KB) │ 28KB
                              └─────────────┘
```

### Which Segment? Two Approaches

**Explicit Approach** (VAX/VMS):
- Use **top bits** of virtual address to select segment
- Example: 14-bit address → top 2 bits select segment (00=code, 01=heap, 11=stack), bottom 12 bits = offset
- Hardware: `Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT`
- Limitation: Each segment limited to maximum size (e.g., 4KB with 2-bit selector in 16KB space)

**Implicit Approach**:
- Hardware determines segment by **how address was formed**
- From PC (instruction fetch) → code segment
- From stack/base pointer → stack segment
- Other → heap segment

### Code Sharing and Protection Bits

**Protection bits** per segment:
- **Read**, **Write**, **Execute** permissions
- Code segment: **Read-Execute** (not writable)
- Heap/Stack: **Read-Write** (not executable)

**Code sharing**:
- Set code segment to read-only
- Same physical code segment can be **shared across multiple processes**
- Each process thinks it has private memory, OS secretly shares read-only code
- Saves memory while preserving isolation

### External Fragmentation: The KEY Problem

**External fragmentation**: Physical memory becomes full of little holes of free space, making it difficult to allocate new segments or grow existing ones.

Example:
- Physical memory has 24KB free in three non-contiguous chunks (8KB, 8KB, 8KB)
- Process requests 20KB segment
- **Cannot satisfy** request even though total free space (24KB) > request (20KB)
- Free space is **fragmented** into pieces too small

**Solutions** (none perfect - "if 1000 solutions exist, no great one does"):
1. **Compaction**: Rearrange segments to create contiguous free space
   - Stop processes, copy data, update segment registers
   - **Expensive**: memory-intensive, uses CPU time
2. **Free-list algorithms**: Best-fit, worst-fit, first-fit, buddy algorithm
   - Try to minimize fragmentation
   - External fragmentation still exists

### OS Responsibilities with Segmentation

1. **Context switch**: Save/restore all segment registers (base + bounds for each segment) to/from PCB
2. **Segment growth**: Handle `malloc()`/`sbrk()` system calls to grow heap
   - Update segment size register if space available
   - May reject if no physical memory available
3. **Managing free space**: Find physical memory for segments using free list
4. **Exception handling**: Handle segmentation faults (out-of-bounds access)

### Per-Process vs Common to All
- **Per-process**: Segment registers (base/bounds), page tables
- **Common**: OS code, trap handlers, free memory lists, CPU

### Practice Questions with Answers

**Q1: Virtual address is 14-bits: [1 0|0 1 0 0 0 0 0 0 0 0 0 0]. Segment bits are top 2. What segment and offset?**

**Answer:**
- Top 2 bits: **10** = segment 2 (stack, if 00=code, 01=heap, 10=stack)
- Remaining 12 bits = 01000000000 = **1024** (offset)
- So: Stack segment, offset 1024

**Q2: Code base=32KB, bounds=2KB. Heap base=34KB, bounds=3KB. Translate virtual address in heap segment with offset 1KB.**

**Answer:**
- Segment: Heap
- Check bounds: 1KB < 3KB ✓
- Physical = 34KB + 1KB = **35KB**

**Q3: Stack grows negative. Stack base=28KB, bounds=2KB, max segment size=4KB. Virtual offset=3KB. What's the physical address?**

**Answer:**
- Negative growth: actual offset = max_size - virtual_offset = 4KB - 3KB = 1KB
- Check: 1KB < 2KB bounds ✓
- Physical = 28KB - 1KB = **27KB**
(Or: base points to bottom, so base - (max_segment - offset))

**Q4: What happens on a context switch with segmentation?**

**Answer:**
1. OS saves current process's segment registers (3 base + 3 bounds) to PCB
2. OS restores next process's segment registers from its PCB into hardware
3. All address translations now use new process's segments
4. This is privileged (kernel mode only)

**Q5: Why is segmentation better than simple base-and-bounds for memory efficiency?**

**Answer:** With base-and-bounds, the entire address space (including the large free gap between heap and stack) must be contiguous in physical memory. Segmentation places each segment separately, so the gap doesn't consume physical memory. Only actually used segments (code, heap, stack) take physical space.

**Q6: What data structures must be per-process vs shared among all processes?**

**Answer:**
- **Per-process**: Segment registers, page tables, PCB, register state, address space
- **Shared/Global**: OS code, physical memory allocator, device drivers, CPU hardware, trap table

**Q7: Explain external fragmentation in segmentation. Why is it a problem and what are potential solutions?**

**Answer:**
- **External fragmentation**: Physical memory gets full of small holes of free space between segments
- **Problem**: May have enough total free space (e.g., 24KB) but not contiguous
  - Cannot satisfy allocation request (e.g., 20KB) even though total free > requested
  - Free space is in non-contiguous pieces (e.g., three 8KB chunks)
- **Solutions**:
  1. **Compaction**: Rearrange segments to create large contiguous free region (expensive - requires copying, updating registers)
  2. **Better allocation algorithms**: Best-fit, worst-fit, first-fit (minimize but don't eliminate fragmentation)
  3. **Ultimate solution** (next chapters): Avoid variable-sized allocations entirely → use **paging**

---

## Chapter 17: Free Space Management

### The Crux: How to Manage Free Space

**How should free space on the heap be managed, when satisfying variable-sized requests? What strategies can be used to minimize fragmentation? What are the time and space overheads of alternate approaches?**

### Problem: External Fragmentation
- Free space chopped into little pieces of **varying sizes**
- Can't satisfy large allocation even if total free space is sufficient
- Applies to **variable-sized units** (like malloc/free with segments)
- Fixed-size units (like paging) **do not** suffer from external fragmentation

### Low-Level Mechanisms

**Splitting**: When requested size < free block size
- Allocate requested amount
- Return remaining space to free list

**Coalescing**: When freeing memory
- Look at addresses of adjacent blocks
- Merge newly freed space with adjacent free chunks
- Returns larger free block to list

**Tracking allocated region size**: Use **header**
- Located **immediately before** the handed-out memory chunk
- Contains at minimum: **size** of allocated region
- When `free(ptr)` called, library examines header at `ptr - sizeof(header)` to find size
- **Magic number** often included for integrity checking

**Embedding free list in free space itself**:
- Free list pointers stored **inside** the free chunks themselves
- When chunk is free, use it to store `next` pointer
- When allocated, data overwrites the pointers (they're not needed)
- **Zero overhead** when chunk is allocated

### Basic Allocation Strategies

**Best Fit**: Search list, find smallest block that fits
- Minimizes wasted space
- **Exhaustive search** → slow

**Worst Fit**: Find largest block, return requested amount
- Leaves bigger leftover chunks (may be more useful)
- **Exhaustive search** → slow
- Still high fragmentation

**First Fit**: Find first block that fits
- **Fast** (stops at first match)
- Pollutes beginning of free list with small objects

**Next Fit**: Like first fit, but start from **last allocation point**
- Spreads searches more uniformly
- Similar performance to first fit

### Advanced Approaches

**Segregated Lists**:
- Keep **separate lists** for popular-sized objects
- Example: one list for 8-byte allocations, another for 16-byte, etc.
- Request for N bytes → check N-byte-specific list first
- If empty, fall back to general allocator
- **Benefits**: Fast allocation for common sizes, reduces fragmentation
- **Slab allocator** is a modern segregated list approach

**Slab Allocator** (Jeff Bonwick, Solaris kernel):
- Allocates **object caches** for kernel objects (locks, file-system inodes, etc.)
- When kernel boots: creates object caches (pools) for each kernel object type
- Requests go to specific object cache
- When cache runs low, requests **slabs** of memory from general allocator
- **Benefits**:
  - No fragmentation (objects same size within cache)
  - Fast allocation/free (free objects immediately reusable)
  - Pre-initialized objects (constructor called once)

**Buddy Allocation** (Binary Buddy Allocator):
- Memory divided using **binary tree** structure
- Free memory starts as one big block (e.g., 64KB)
- When request comes in, recursively divide block in **half** until block just big enough
- Example: 7KB request from 64KB
  - Divide 64→32→16→8 (8KB buddy blocks)
  - Allocate one 8KB, mark buddy as free
- **Coalescing**: When freeing, check if **buddy** is free → merge back
  - Buddy address: flip bit in address at recursion level
- **Benefits**: Fast coalescing (buddy address easily calculated)
- **Drawbacks**: Internal fragmentation (7KB request wastes ~1KB)

### Visual Examples

**Splitting** example:
```
Request 10 bytes from 30-byte block:
Before: [30 bytes free]
After:  [10 allocated][20 free]
```

**Coalescing** example:
```
Free block A, then B (adjacent):
Before: [A: used][B: used][C: free]
After A freed: [A: free][B: used][C: free]
After B freed: [A+B+C: free] ← coalesced!
```

### Visual: Free List with Headers
```
Heap Memory:
┌────────┬─────────┬────────┬─────────┐
│ Header │  Data   │ Header │  Free   │
│ size=8 │ (8 B)   │ size=16│ (16 B)  │
│ alloc=1│         │ alloc=0│         │
└────────┴─────────┴────────┴─────────┘
```

### Practice Questions with Answers

**Q1: Free blocks: 20B, 35B, 50B. Request 30B. What block is chosen by Best Fit? First Fit? Worst Fit?**

**Answer:**
- **Best Fit**: 35B (smallest that fits)
- **First Fit**: 35B (first in list that fits)
- **Worst Fit**: 50B (largest block)

**Q2: Heap has blocks: [100B free][50B used][60B free][80B free]. After coalescing, what does it look like?**

**Answer:**
```
Before: [100 free][50 used][60 free][80 free]
After:  [100 free][50 used][140 free]
          ↑ (not merged - separated by used block)
                          ↑ (60+80 merged!)
```
Can't merge 100 with 60 because 50-byte used block separates them.

**Q3: A memory allocator has a free list of [10B, 25B, 40B]. Request 12B using First Fit. Show result after split.**

**Answer:**
- First Fit finds 25B (first block ≥12B)
- Split: allocate 12B, remainder = 25-12 = 13B
- **Result**: [10B free][12B used][13B free][40B free]

**Q4: Why is external fragmentation a problem even when total free space is large?**

**Answer:** External fragmentation creates many small, non-contiguous free blocks. Even if their total size is 100KB, you can't satisfy a 50KB request if the largest contiguous block is only 10KB. Memory is fragmented into unusable pieces.

**Q5: What is the purpose of the header in each allocated block, and how does it enable free()?**

**Answer:** The header stores metadata:
- **Size** of the block (needed for free() to know how much to free)
- **Allocation status** (allocated or free)
- **Magic number** for integrity checking
- Located **immediately before** the returned pointer
When user calls `free(ptr)`, the library examines the header at `ptr - sizeof(header)` to determine how much memory to return to the free list.

**Q6: Explain how segregated lists improve allocation performance. What is the slab allocator?**

**Answer:**
**Segregated lists**:
- Maintain **separate free lists** for popular object sizes (e.g., 8B list, 16B list, 64B list)
- Allocation for N bytes → check N-byte list first (fast, O(1) if available)
- If empty, request from general allocator
- **Benefits**: Reduces fragmentation, fast allocation for common sizes

**Slab allocator** (Bonwick, Solaris):
- Specialized segregated list for **kernel objects** (inodes, locks, process descriptors)
- Pre-allocates **object caches** (slabs) for each type
- Objects pre-initialized with constructor
- **Benefits**: Zero fragmentation within cache, very fast alloc/free, no initialization overhead

**Q7: A buddy allocator has 64KB free. Request 7KB. Show the splitting process and calculate internal fragmentation.**

**Answer:**
Buddy allocation uses **binary splitting**:
1. Start: 64KB free
2. Need ≥7KB → split: 64KB → 32KB + 32KB
3. Need ≥7KB → split left: 32KB → 16KB + 16KB
4. Need ≥7KB → split left: 16KB → 8KB + 8KB
5. Allocate one 8KB block (just fits 7KB request)

**Result**:
- Allocated: 8KB
- Used: 7KB
- **Internal fragmentation**: 8KB - 7KB = **1KB wasted**

Free structure: [8KB allocated][8KB free buddy][16KB free][32KB free]

---

## Chapter 18: Introduction to Paging

### The Crux: How to Virtualize Memory with Pages

**How can we virtualize memory with pages, so as to avoid the problems of segmentation? What are the basic techniques? How do we make those techniques work well, with minimal space and time overheads?**

### Core Concept

**Paging** - second approach to space management (from **Atlas system**, early 1960s):
- Chop space into **fixed-sized pieces** (not variable-sized like segmentation)
- Divide address space into fixed-size **pages** (virtual memory)
- Divide physical memory into fixed-size **page frames** (physical memory)
- Page size = Frame size (typically 4KB)
- Each frame can contain exactly one virtual page

### Advantages Over Segmentation

1. **No external fragmentation**:
   - Fixed-size units mean any page can fit in any free frame
   - No need to worry about fragmentation of free space

2. **Flexibility**:
   - Support sparse address spaces easily
   - No assumptions about heap/stack growth direction
   - Process can use address space however it wants

### Page Table

**Per-process data structure** that stores address translations:
- Maps virtual page numbers (VPN) to physical frame numbers (PFN)
- **One page table per process** in the system
- **Linear page table** = simple array structure
  - OS indexes array by VPN
  - Looks up page table entry (PTE) at that index
  - Extracts PFN from the PTE

**Where stored**:
- **In memory** (managed by OS), NOT in MMU hardware
- Too big for special on-chip hardware
- Example: 32-bit address space, 4KB pages
  - 20-bit VPN (2²⁰ = ~1 million entries)
  - 4 bytes per PTE
  - **4MB per page table!**
  - 100 processes = 400MB just for page tables

**Page-table base register (PTBR)**:
- Hardware register containing physical address of page table start
- Used to locate page table for address translation

### Page Table Entry (PTE) Contents

Each PTE contains several bits:

1. **Valid bit**: Is this translation valid?
   - Invalid pages generate trap to OS (likely terminate process)
   - Supports sparse address spaces (mark unused pages invalid)

2. **Protection bits**: Read/Write/Execute permissions
   - Can page be read from? Written to? Executed?
   - Violation generates trap to OS

3. **Present bit**: Is page in physical memory or swapped to disk?
   - Used for swapping (covered in later chapters)

4. **Dirty bit**: Has page been modified since loaded into memory?
   - Used for page replacement decisions

5. **Reference/Accessed bit**: Has page been accessed?
   - Tracks popular pages for replacement policies

6. **PFN (Physical Frame Number)**: The actual translation
   - Also called PPN (Physical Page Number)

### Address Translation Process

**Virtual address breakdown**:
```
Virtual Address = [VPN | Offset]
```
- VPN bits = log₂(# virtual pages)
- Offset bits = log₂(page size)

**Translation steps**:
1. Extract VPN from virtual address: `VPN = (VA & VPN_MASK) >> SHIFT`
2. Find PTE address: `PTEAddr = PTBR + (VPN × sizeof(PTE))`
3. Fetch PTE from memory: `PTE = Memory[PTEAddr]`
4. Check valid bit and protection bits
5. Extract PFN from PTE
6. Form physical address: `PA = (PFN << SHIFT) | offset`

**Note**: Offset stays the same (not translated)

### The Two Problems with Paging

1. **Too Slow**:
   - Every memory access requires **TWO** memory accesses:
     - One to fetch PTE from page table
     - One to fetch actual data
   - Can slow down process by **factor of 2 or more**
   - Solution: TLB (Chapter 19)

2. **Too Much Memory**:
   - Page tables consume large amounts of memory
   - 32-bit address space with 4KB pages = 4MB per process
   - Wasted on page table instead of useful application data
   - Solution: Advanced page table structures (Chapter 20)

### Visual: Address Translation Example

**Example from textbook** (64-byte address space, 16-byte pages):
```
Virtual address 21 = binary 010101

VPN (top 2 bits)  offset (bottom 4 bits)
      01      |        0101
      ↓       |          ↓
   VPN = 1    |    offset = 5 (5th byte of page)

Page table lookup: VPN 1 → PFN 7 (binary 111)

Physical address:
      111     |        0101
   PFN = 7    |    offset = 5
   = 1110101 binary = 117 decimal
```

### Calculations
- **# Virtual Pages** = Virtual Address Space Size / Page Size
- **# Physical Frames** = Physical Memory Size / Page Size
- **VPN bits** = log₂(# virtual pages)
- **PFN bits** = log₂(# physical frames)
- **Offset bits** = log₂(page size)
- **Page table size** = # Virtual Pages × sizeof(PTE)

### Practice Questions with Answers

**Q1: What are the two main advantages of paging over segmentation?**

**Answer:**
1. **No external fragmentation**: Fixed-size units (pages/frames) mean any free frame can hold any page. No fragmentation of free space into unusable holes.
2. **Flexibility**: Supports sparse address spaces easily. No assumptions about how process uses memory (heap/stack growth direction). Mark unused pages as invalid without allocating physical memory.

**Q2: 32-bit virtual address space, 4KB pages. Calculate: (a) how many virtual pages, (b) VPN bits, (c) offset bits, (d) linear page table size.**

**Answer:**
- Page size = 4KB = 2¹² bytes
- (a) # virtual pages = 2³² / 2¹² = **2²⁰ pages** (~1 million)
- (b) VPN bits = log₂(2²⁰) = **20 bits**
- (c) Offset bits = log₂(2¹²) = **12 bits**
- (d) Page table size = 2²⁰ entries × 4 bytes/PTE = **4MB**

**Q3: Page table for process: VPN 0→PFN 4, VPN 1→PFN 7, VPN 2→PFN 1. Page size=1KB. Translate virtual address 1200.**

**Answer:**
- Page size = 1KB = 1024 bytes
- VPN = 1200 / 1024 = **1**, Offset = 1200 % 1024 = **176**
- Page table lookup: VPN 1 → PFN 7
- Physical address = (PFN × page size) + offset = (7 × 1024) + 176 = **7344**

**Q4: List and explain three bits found in a typical page table entry (PTE).**

**Answer:**
1. **Valid bit**: Indicates if translation is valid. If 0, page is not in use → accessing it traps to OS (segmentation fault). Allows sparse address spaces.
2. **Protection bits (R/W/X)**: Specifies whether page can be read, written, or executed. Prevents code from being written to, data from being executed.
3. **Dirty bit**: Tracks if page has been modified since loaded. Used for page replacement—clean pages don't need to be written back to disk.
(Others: Present bit for swapping, Reference/Accessed bit for replacement policies)

**Q5: Why does paging cause the system to run "too slowly"? Explain with a specific example.**

**Answer:**
Every memory access requires **TWO** memory accesses:
1. Access page table in memory to fetch PTE (get PFN for translation)
2. Access actual data at translated physical address

Example: `mov 0x1000, %eax`
- Access 1: Fetch PTE for VPN of 0x1000 from page table in memory
- Access 2: Fetch data at physical address from translated result
This **doubles** the memory accesses, slowing system by factor of 2 or more. TLB (next chapter) solves this.

**Q6: 64-byte virtual address space, 16-byte pages. Virtual address is 21. Show the translation to physical address if VPN 1 maps to PFN 7.**

**Answer:**
- 64-byte space / 16-byte pages = 4 pages → VPN = 2 bits
- Offset within 16-byte page → 4 bits
- Virtual address 21 = binary 010101
  - VPN = top 2 bits = **01** (VPN 1)
  - Offset = bottom 4 bits = **0101** (5 decimal)
- Page table: VPN 1 → PFN 7 (binary 111)
- Physical address = 1110101 binary = **117 decimal**

**Q7: Why are page tables stored in memory instead of in special hardware registers in the MMU?**

**Answer:** Page tables are **too large** to fit in hardware. Example: 32-bit address space with 4KB pages requires 2²⁰ entries (~1 million). With 4 bytes per PTE, that's 4MB per process. With 100 processes, that would be 400MB of on-chip storage—completely impractical. Instead, page tables reside in memory, and hardware uses a **page-table base register (PTBR)** to locate them.

---

## Chapter 19: Paging - Faster Translations (TLBs)

### The Crux: How to Speed Up Address Translation

**How can we speed up address translation, and generally avoid the extra memory reference that paging seems to require? What hardware support is required? What OS involvement is needed?**

### What is a TLB?

**TLB** = **Translation-Lookaside Buffer** (historical name from 1964, John Couleur)
- Better name: **Address-translation cache**
- **Hardware cache** of popular virtual-to-physical address translations
- Part of chip's **Memory Management Unit (MMU)**
- Small, fast, on-chip memory
- **Makes virtual memory possible** - without TLB, paging would be too slow

### TLB Basic Operation

**On each virtual memory reference**:
1. Hardware checks TLB for translation
2. **TLB Hit**: Translation found → use it (fast, no memory access needed)
3. **TLB Miss**: Translation not found → access page table in memory, update TLB, retry instruction

**Control flow**:
```
VPN = extract from virtual address
if (VPN in TLB):  // TLB Hit
    PFN = TLB[VPN].PFN
    check protection bits
    PhysAddr = (PFN << SHIFT) | offset
    access memory
else:  // TLB Miss
    PTE = PageTable[VPN]  // Extra memory access!
    check valid bit, protection bits
    TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
    retry instruction  // Now will be TLB hit
```

### Spatial and Temporal Locality

**Spatial locality**: Elements close in space accessed together
- Array access: `a[0]`, `a[1]`, `a[2]` all on same page
- First access to page = TLB miss, subsequent accesses = TLB hit
- Example: 10-element array spread across 3 pages
  - Access pattern: **miss, hit, hit, miss, hit, hit, hit, miss, hit, hit**
  - Hit rate: **70%** even on first pass!

**Temporal locality**: Recently accessed items accessed again soon
- Second pass through same array: **all hits** (if TLB large enough)
- Loop variables, repeatedly executed code

### Who Handles TLB Miss?

**Hardware-managed TLB** (CISC, Intel x86):
- Hardware knows page table location (CR3 register)
- Hardware knows page table format (multi-level)
- On miss: Hardware automatically walks page table, updates TLB
- **Advantage**: Fast
- **Disadvantage**: Inflexible (page table format fixed by hardware)

**Software-managed TLB** (RISC, MIPS, SPARC):
- On miss: Hardware raises **TLB_MISS exception**
- Traps to OS handler in kernel mode
- OS walks page table (any format OS wants!)
- OS uses **privileged instructions** to update TLB
- Return-from-trap **retries** instruction → TLB hit
- **Advantage**: Flexibility (OS can use any page table structure)
- **Disadvantage**: Trap overhead

**Important details**:
1. **Return-from-trap must retry** the instruction that caused the trap (not next instruction)
2. **Avoid infinite TLB miss loops**:
   - Keep TLB miss handler in **physical memory** (unmapped)
   - Use **wired translations** (permanently in TLB for handler code)

### TLB Contents

**Typical TLB**: 32, 64, or 128 entries

**Each entry**:
```
| VPN | PFN | Valid | Protection | Dirty | ASID | G | other |
```

- **VPN**: Virtual Page Number (tag for lookup)
- **PFN**: Physical Frame Number (translation)
- **Valid bit**: Is this entry valid?
- **Protection bits**: Read/Write/Execute permissions
- **Dirty bit**: Has page been modified?
- **ASID**: Address Space Identifier (process ID, ~8 bits)
- **G (Global)**: Globally shared across processes

**Fully associative**:
- Translation can be in **any TLB entry**
- Hardware searches **all entries in parallel**
- Expensive but necessary for small TLBs

### Context Switch Problem

**Problem**: TLB contains translations only valid for current process
- Process P1: VPN 10 → PFN 100
- Process P2: VPN 10 → PFN 170
- If both in TLB, which is which?

**Solution 1: Flush TLB on context switch**
- Set all valid bits to 0
- **Advantage**: Simple, works
- **Disadvantage**: Next process incurs TLB misses for all pages → expensive if frequent context switches

**Solution 2: Address Space Identifier (ASID)**
- Add ASID field to each TLB entry
- Hardware checks: `(VPN == entry.VPN) AND (ASID == entry.ASID)`
- OS sets ASID register on context switch
- **Advantage**: Processes can share TLB without confusion
- **Disadvantage**: What if > 2^8 processes? (must flush some entries)

**TLB with ASID**:
```
VPN | PFN | Valid | Prot | ASID
 10 | 100 |   1   | rwx  |  1    ← Process 1
 10 | 170 |   1   | rwx  |  2    ← Process 2 (no conflict!)
```

### TLB Replacement Policy

**When TLB full and new entry needed, which to evict?**

**LRU (Least Recently Used)**:
- Evict entry not used for longest time
- Exploits temporal locality
- **Problem**: Pathological case with loop over n+1 pages in TLB of size n → 100% misses

**Random**:
- Evict random entry
- Simple, avoids corner cases
- Generally performs reasonably well

### TLB Coverage and Performance

**TLB coverage** = # entries × page size
- Example: 64 entries × 4KB = 256KB coverage

**Exceeding TLB coverage**:
- Working set > TLB coverage → many TLB misses → severe performance penalty
- **Solution**: Use **larger pages** for key data structures
  - Maps more memory with same # of TLB entries
  - Common in databases (large, randomly accessed data)

**Culler's Law**: "RAM isn't always RAM"
- Random memory access can be much slower if exceeds TLB coverage
- Access cost varies depending on TLB hits/misses

### Practice Questions with Answers

**Q1: Why is a TLB necessary? What problem would exist without it?**

**Answer:** Without a TLB, **every** memory access would require **two** memory accesses:
1. Access page table in memory to get translation
2. Access actual data at translated address
This would make programs run **2x slower or worse**. The TLB caches translations so most accesses only need one memory reference (on TLB hit), making paging practical.

**Q2: A TLB has 64 entries. A program accesses a 10-element integer array (4 bytes each) with 16-byte pages. The array starts at virtual address 100. Trace the TLB hits/misses for accessing all elements sequentially.**

**Answer:**
Array: `a[0]` at 100, `a[1]` at 104, ..., `a[9]` at 136
- Page size = 16 bytes → 4 integers per page
- `a[0], a[1], a[2], a[3]` on VPN 6 (bytes 96-111)
- `a[4], a[5], a[6], a[7]` on VPN 7 (bytes 112-127)
- `a[8], a[9]` on VPN 8 (bytes 128-143)

Access pattern:
- `a[0]`: **Miss** (load VPN 6)
- `a[1], a[2], a[3]`: **Hit, Hit, Hit** (VPN 6 in TLB)
- `a[4]`: **Miss** (load VPN 7)
- `a[5], a[6], a[7]`: **Hit, Hit, Hit**
- `a[8]`: **Miss** (load VPN 8)
- `a[9]`: **Hit**

**Result**: 3 misses, 7 hits → **70% hit rate** (spatial locality!)

**Q3: What is the difference between hardware-managed and software-managed TLBs? Give an example architecture of each.**

**Answer:**
**Hardware-managed** (Intel x86, CISC):
- Hardware walks page table on TLB miss
- Hardware must know page table format and location
- Faster (no trap overhead)
- Less flexible (page table structure fixed)

**Software-managed** (MIPS, SPARC, RISC):
- TLB miss raises exception to OS
- OS trap handler walks page table, updates TLB
- Slower (trap overhead)
- More flexible (OS can use any page table structure)

**Q4: Explain how Address Space Identifiers (ASIDs) solve the context switch problem. Why is flushing the TLB on every context switch expensive?**

**Answer:**
**Without ASID**: Two processes with same VPN would conflict in TLB. Must flush entire TLB on context switch (set all valid bits to 0).
- **Expensive**: Next process incurs TLB miss on every page access until TLB repopulated
- Frequent context switches → many TLB misses → poor performance

**With ASID**: Each TLB entry tagged with process ID (ASID field)
- Hardware checks: match VPN **and** ASID
- Processes can share TLB without confusion
- No flush needed → much faster context switches

**Q5: A system has 64-entry TLB with 4KB pages. Calculate TLB coverage. If a program's working set is 512KB, what will likely happen to performance?**

**Answer:**
- **TLB coverage** = 64 entries × 4KB = **256KB**
- Working set (512KB) **exceeds** TLB coverage
- Program will experience **many TLB misses** (exceeding TLB coverage)
- **Severe performance penalty** - constant thrashing of TLB
- **Solution**: Use larger pages (e.g., 2MB pages → 128MB coverage)

**Q6: On a TLB hit, how many memory accesses are required? On a TLB miss?**

**Answer:**
- **TLB hit**: **1 memory access** (just the data itself; translation in TLB on-chip)
- **TLB miss** (hardware-managed): **2 memory accesses**:
  1. Access page table in memory to get PFN
  2. Access actual data at translated address
- **TLB miss** (software-managed): **2+ memory accesses**:
  1. Access page table (may involve multiple levels)
  2. Access actual data
  Plus trap overhead

**Q7: Explain the difference between temporal locality and spatial locality. How does each improve TLB performance?**

**Answer:**
**Spatial locality**: Memory locations **close together** accessed near in time
- Example: Array elements `a[0], a[1], a[2]`
- **TLB benefit**: Elements on same page share one translation → one TLB miss covers multiple accesses

**Temporal locality**: Same memory location accessed **repeatedly over time**
- Example: Loop variable, loop instructions
- **TLB benefit**: Once translation in TLB, subsequent accesses to same page are hits

Both reduce TLB misses, making paging efficient in practice.

---

## Chapter 20: Paging - Smaller Tables

### The Crux: How To Make Page Tables Smaller?

**Simple array-based page tables (linear page tables) are too big, taking up far too much memory on typical systems. How can we make page tables smaller? What are the key ideas? What inefficiencies arise as a result of these new data structures?**

### The Problem: Page Tables Are Too Big

With 32-bit address space and 4KB pages:
- Virtual address space = 2^32 bytes = **4GB**
- Page size = 4KB = 2^12 bytes
- Number of pages = 4GB / 4KB = **2^20 = 1 million pages**
- Each PTE = 4 bytes
- **Page table size = 4MB per process**

With 100 processes: **400MB** just for page tables! Most of that space is wasted (invalid pages).

### Solution 1: Bigger Pages

**Idea**: Use larger pages (e.g., 16KB instead of 4KB)
- Fewer pages → smaller page table
- Example: 16KB pages → 2^18 = 256K pages → 1MB page table

**Problem**: **Internal fragmentation**
- Large pages waste space inside each page
- Application uses small portion of page → rest wasted
- **Not commonly used** (except for specific use cases like large databases)

### Solution 2: Hybrid Approach (Paging + Segmentation)

**Idea**: Combine paging and segmentation (from Multics system)
- One page table **per segment** (code, heap, stack)
- Base register points to page table for that segment
- Bounds register indicates end of page table

**Advantages**:
- Unallocated pages between stack and heap don't need page table entries
- Saves memory compared to linear page table

**Disadvantages**:
- Still have **segmentation problems** (external fragmentation)
- Not flexible if sparse address space (large but sparsely used heap)
- **Not widely used** in modern systems

### Solution 3: Multi-Level Page Tables (Most Important!)

**The key idea**: Turn linear page table into **tree structure**
- Chop up page table into **page-sized units**
- If entire page of PTEs invalid → don't allocate that page
- Use **page directory** to track which pages of page table are allocated

**Two-level page table structure**:

```
Page Directory (points to pages of page table)
    ↓
Page Table Pages (contains actual PTEs)
    ↓
Physical Memory Pages
```

**Advantages**:
1. **Only allocate page table space proportional to amount of address space in use**
   - Sparse address spaces use much less memory
   - Growing address space easier (just allocate new page table pages)
2. **Each page table page fits in a page** → can be swapped if memory tight
3. **Widely used** in modern systems (x86, ARM, etc.)

**Disadvantages**:
1. **Time-space trade-off**: Saves space but costs time
   - On TLB miss: must load **two** PTEs from memory (directory entry + actual PTE)
   - More complex to manage
2. **More memory lookups** on TLB miss:
   - Without TLB: 1 load for PTE + 1 for data = **2 loads**
   - With 2-level: 1 for directory + 1 for PTE + 1 for data = **3 loads**
   - TLB critically important!

### Multi-Level Page Table Example (x86)

**32-bit address breakdown** (two-level):
```
| VPN (20 bits)     | Offset (12 bits) |
| PDI (10) | PTI (10) | Offset (12)     |
```

- **PDI** (Page Directory Index): Index into page directory (2^10 = 1024 entries)
- **PTI** (Page Table Index): Index into page table (2^10 = 1024 entries)
- **Offset**: Byte within page (4KB = 2^12)

**Address translation with two-level table**:
```
1. Extract PDI from virtual address
2. PDEAddr = PDBR + (PDI × sizeof(PDE))  // Page Directory Base Register
3. PDE = Memory[PDEAddr]
4. if (PDE.Valid == False): raise exception
5. PTEAddr = (PDE.PFN << SHIFT) + (PTI × sizeof(PTE))
6. PTE = Memory[PTEAddr]
7. if (PTE.Valid == False): raise exception
8. PhysAddr = (PTE.PFN << SHIFT) + Offset
```

**Three-level and beyond**:
- 64-bit address spaces require **more levels** (x86-64 uses 4 levels)
- More levels = even more memory lookups on TLB miss
- **TLB becomes even more critical** for performance

### Solution 4: Inverted Page Tables

**Idea**: Instead of one entry per virtual page, have **one entry per physical page**
- Page table size determined by **physical memory**, not virtual
- Example: 1GB physical memory, 4KB pages → 256K entries (only 1MB for page table!)

**Structure**:
```
Physical Frame | VPN | ASID
      0        | ... | ...
      1        | ... | ...
     ...
```

**Translation**:
- Search page table for matching (VPN, ASID)
- Index in table = physical frame number
- **Problem**: Linear scan is slow → use **hash table** to speed up lookup

**Advantages**: Very **compact** (size based on physical memory)

**Disadvantages**: Reverse lookup is **slower** (even with hash table)

### Swapping Page Tables to Disk

Since page table pages fit in pages, kernel can **swap** them to disk if memory tight
- Page directory entries have **present bit**
- If present bit = 0, page table page on disk
- **Page fault** when accessing → OS loads page table page from disk

Allows system to support very large address spaces even when memory constrained.

### Practice Questions with Answers

**Q1: Why are simple linear page tables impractical for modern systems? Calculate the page table size for a 32-bit address space with 4KB pages and 4-byte PTEs.**

**Answer:**
- Address space = 2^32 = 4GB
- Page size = 4KB = 2^12
- Number of pages = 4GB / 4KB = 2^20 = 1,048,576 pages
- PTE size = 4 bytes
- **Page table size = 1,048,576 × 4 = 4,194,304 bytes = 4MB per process**

With 100 processes: **400MB** just for page tables! Most entries are invalid (wasted space). Linear page tables don't scale.

**Q2: Explain the time-space trade-off in multi-level page tables. Why does this make TLBs even more important?**

**Answer:**
**Space saving**: Multi-level tables only allocate page table space for used portions of address space (sparse allocation). Sparse address spaces use much less memory than linear tables.

**Time cost**: On TLB miss, need **multiple memory accesses**:
- Linear table: 1 PTE load + 1 data = **2 memory accesses**
- Two-level: 1 page directory + 1 PTE + 1 data = **3 memory accesses**
- Four-level (x86-64): **5 memory accesses** total!

**Why TLB critical**: On **TLB hit**, only **1 memory access** (for data). The TLB eliminates extra page table lookups, making the time cost negligible in practice. Without TLB, multi-level tables would be unacceptably slow.

**Q3: For a two-level page table on x86 (4KB pages), how is a 32-bit virtual address broken down? Show the calculation for translating VPN to page directory and page table indices.**

**Answer:**
**Address breakdown**:
```
| Page Directory Index | Page Table Index | Offset |
|      10 bits        |     10 bits      | 12 bits|
```

- **Offset**: 12 bits → 2^12 = 4KB page
- **PTI** (Page Table Index): 10 bits → 2^10 = 1024 PTEs per page table page
- **PDI** (Page Directory Index): 10 bits → 2^10 = 1024 page directory entries

**Example**: Virtual address = `0x00403004`
- Binary: `00 0000 0100 | 00 0000 0011 | 0000 0000 0100`
- PDI = `0000000100` = **4**
- PTI = `0000000011` = **3**
- Offset = `000000000100` = **4**

**Translation**:
1. Page directory entry 4 → gives PFN of page table page
2. Page table entry 3 in that page → gives PFN of actual data page
3. Offset 4 within that page → final physical address

**Q4: How does a multi-level page table save memory compared to a linear page table? Give a concrete example.**

**Answer:**
**Linear page table**: Must allocate **all** PTEs, even for invalid pages
- 32-bit address, 4KB pages → **4MB page table** (all 1M entries)

**Multi-level page table**: Only allocate page table pages for **valid** regions

**Example**: Process uses:
- Code: 1MB (at bottom of address space)
- Heap: 2MB (growing upward)
- Stack: 1MB (at top of address space)
- **Total**: 4MB of 4GB address space used

**Linear**: 4MB for page table

**Two-level**:
- Page directory: 1 page = **4KB**
- Code region: 1MB / 4KB = 256 pages → needs 256 PTEs → **1 page table page (4KB)**
- Heap: 2MB → 512 PTEs → **2 page table pages (8KB)**
- Stack: 1MB → 256 PTEs → **1 page table page (4KB)**
- **Total**: 4KB + 4KB + 8KB + 4KB = **20KB** (200x smaller!)

**Q5: What is an inverted page table? What are its advantages and disadvantages?**

**Answer:**
**Inverted page table**: **One entry per physical frame** (not per virtual page)
- Size determined by **physical memory**, not virtual address space
- Entry contains: (VPN, ASID) for process using that frame

**Example**: 1GB physical memory, 4KB pages
- Frames = 1GB / 4KB = 256K frames
- Page table = 256K entries × 8 bytes = **2MB** (vs 400MB for 100 linear page tables)

**Advantages**:
- **Very compact** - grows with physical memory, not virtual
- Fixed size regardless of number of processes

**Disadvantages**:
- **Slow translation** - must search table for matching (VPN, ASID)
- Requires hash table to speed up lookups
- Still slower than multi-level forward page tables with TLB

**Used in**: IBM PowerPC, older systems

**Q6: Why is using bigger pages (e.g., 16KB instead of 4KB) not a widely adopted solution for reducing page table size?**

**Answer:**
**How it helps**: Bigger pages → fewer pages → smaller page table
- 4KB pages: 1M pages → 4MB table
- 16KB pages: 256K pages → 1MB table

**Why not used**:
- **Internal fragmentation**: Application allocates 16KB page but only uses 4KB → **12KB wasted**
- Small applications waste significant memory
- **Trade-off gets worse**: 2MB pages (x86 "huge pages") → massive waste for small allocations

**Where it IS used**:
- Large databases, scientific computing (huge datasets)
- Increase TLB coverage for large working sets
- **Optional/selective**, not default for all processes

**Multi-level page tables** are preferred: save space without internal fragmentation.

**Q7: Describe the address translation process for a two-level page table. What happens on a TLB miss with this structure?**

**Answer:**
**On TLB hit** (common case):
1. TLB contains (VPN → PFN) mapping
2. PhysAddr = (PFN << 12) | Offset
3. Access memory once → **1 memory access total**

**On TLB miss** (two-level table):
1. **Extract PDI** (page directory index) from VPN
2. **Load page directory entry**: `PDE = Memory[PDBR + PDI × 4]` → **Memory access #1**
3. Check PDE.Valid (if 0 → page fault)
4. **Extract PTI** (page table index) from VPN
5. **Load page table entry**: `PTE = Memory[(PDE.PFN << 12) + PTI × 4]` → **Memory access #2**
6. Check PTE.Valid (if 0 → page fault)
7. **Access data**: `Data = Memory[(PTE.PFN << 12) + Offset]` → **Memory access #3**
8. **Update TLB** with (VPN → PFN) mapping
9. **Total on TLB miss: 3 memory accesses** (vs 1 on TLB hit)

This is why **TLB hit rate is critical** for performance!

---

## Chapters 26/27: Introduction to Concurrency

### The Crux: How To Support Synchronization

**What support do we need from the hardware in order to build useful synchronization primitives? What support do we need from the OS? How can we build these primitives correctly and efficiently? How can programs use them to get the desired results?**

### Thread Abstraction

**Classic process**: Single point of execution (one PC fetching and executing instructions)

**Multi-threaded process**: More than one point of execution (**multiple PCs**, each being fetched and executed from)

**Key insight**: Each thread is like a separate process, **except** they share the same address space and thus can access the same data.

### Thread State

**Each thread has**:
- **Program Counter (PC)**: Tracks where thread is fetching instructions
- **Private set of registers** for computation
- **Own stack** (thread-local storage for local variables, parameters, return values)

**Thread Control Block (TCB)**: Stores state of each thread (similar to PCB for processes)

### Context Switch Between Threads

**Similar to process context switch**:
- Save register state of T1 to its TCB
- Restore register state of T2 from its TCB

**Major difference**: **Address space remains the same**
- No need to switch page tables (unlike process context switch)
- This is why thread switching is **much faster** than process switching

### Multi-Threaded Address Space

**Single-threaded process**: One stack at bottom of address space

**Multi-threaded process**: **One stack per thread** spread throughout address space

```
Single-Threaded:              Multi-Threaded (2 threads):
0KB  ┌─────────────┐         0KB  ┌─────────────┐
     │ Program Code│              │ Program Code│
1KB  ├─────────────┤         1KB  ├─────────────┤
     │    Heap     │              │    Heap     │
2KB  │      ↓      │         2KB  ├─────────────┤
     │             │              │   (free)    │
     │             │              │             │
     │      ↑      │              ├─────────────┤
15KB │   Stack     │              │  Stack (2)  │  ← Thread 2 stack
16KB └─────────────┘              ├─────────────┤
                                  │   (free)    │
                             15KB ├─────────────┤
                                  │  Stack (1)  │  ← Thread 1 stack
                             16KB └─────────────┘
```

**Thread-local storage**: Stack-allocated variables, parameters, return values placed in the stack of the relevant thread

### Why Use Threads?

**Reason 1: Parallelism**
- Use multiple CPUs to speed up programs
- Example: Adding two large arrays
  - Single CPU: Process sequentially
  - Multiple CPUs: Each CPU processes a portion → **much faster**
- **Parallelization**: Transforming single-threaded program to use multiple CPUs
- Thread per CPU is natural way to make programs faster on modern hardware

**Reason 2: Avoid Blocking Due to Slow I/O**
- Instead of waiting for I/O (disk, network, page fault), do something else
- While one thread waits (blocked for I/O), CPU scheduler switches to other ready threads
- **Overlap I/O with computation** within single program
- Like multiprogramming for processes, but within one program
- Why modern servers (web servers, databases) use threads extensively

**Why not use processes?**
- Could use multiple processes instead
- But threads **share address space** → easy to share data
- Threads are natural choice when sharing data structures needed
- Processes better for logically separate tasks with little sharing

### The Problem: Shared Data and Race Conditions

**Example**: Two threads each increment shared counter 10 million times
```c
static volatile int counter = 0;

void *mythread(void *arg) {
    for (int i = 0; i < 10000000; i++) {
        counter = counter + 1;  // Critical section
    }
}
```

**Expected result**: 20,000,000

**Actual result**: Sometimes 19,345,221, sometimes 19,221,041, sometimes correct!

**Why?** The single C statement `counter = counter + 1` compiles to **three assembly instructions**:

```assembly
mov 0x8049a1c, %eax   # Load counter into register
add $0x1, %eax        # Increment register
mov %eax, 0x8049a1c   # Store register back to counter
```

**The race condition**:
1. Thread 1 loads counter (50) into eax
2. Thread 1 increments eax to 51
3. **INTERRUPT** - Thread 1 paused, Thread 2 runs
4. Thread 2 loads counter (still 50!) into eax
5. Thread 2 increments eax to 51
6. Thread 2 stores 51 to counter
7. **INTERRUPT** - Thread 1 resumes
8. Thread 1 stores eax (51) to counter
9. **Result**: counter = 51 (should be 52!)

### Key Concurrency Terms (Dijkstra)

**Critical Section**:
- Piece of code that accesses shared resource (variable, data structure, etc.)
- **Must not** be concurrently executed by more than one thread
- Example: The three instructions updating counter

**Race Condition (Data Race)**:
- Results depend on **timing** of code execution
- Multiple threads enter critical section at roughly same time
- Both attempt to update shared data → surprising outcome

**Indeterminate Program**:
- Contains one or more race conditions
- Output **varies** from run to run depending on which threads ran when
- Not **deterministic** (what we expect from computers!)

**Mutual Exclusion**:
- Property that guarantees only **one thread** executing in critical section
- If one thread in critical section, others **prevented** from entering
- What we need to solve the race condition problem

**All these terms coined by Edsger Dijkstra** (Turing Award winner, 1968 paper "Cooperating Sequential Processes")

### The Wish For Atomicity

**Atomicity**: Execute as a unit, "**all or nothing**"
- Either all actions in group occur, or none occur
- No in-between state visible
- Also called a **transaction** (from database systems)

**What we want**: Execute the three-instruction sequence atomically:
```assembly
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```

**Hardware solution**: Could have powerful atomic instruction like:
```assembly
memory-add 0x8049a1c, $0x1  # Atomic increment of memory location
```

Hardware guarantees: When interrupt occurs, instruction has either not run at all, or has run to completion. No in-between state.

**Problem**: Can't have atomic instruction for everything (imagine "atomic update of B-tree"!)

**Real solution**:
- Hardware provides a few useful **atomic instructions**
- OS provides help
- Build general **synchronization primitives** on top
- Use these primitives to build correct multi-threaded code

### One More Problem: Waiting For Another

Besides atomicity, threads often need to **wait** for each other:
- One thread must wait for another to complete some action before continuing
- Example: Process performs disk I/O and is put to sleep; when I/O completes, process needs to be woken up

Coming chapters cover:
1. **Synchronization primitives** for atomicity (locks)
2. **Mechanisms for sleeping/waking** interaction (condition variables)

### Why in OS Class?

**History**: OS was the **first concurrent program**
- Interrupts can occur at any time
- OS must update kernel data structures (page tables, process lists, file system structures, bitmaps, inodes)
- These are **critical sections** that must be protected
- Techniques created for OS, later used by application programmers

**Example**: Two processes call write() to append to same file
- Both must: allocate new block, update inode, change file size
- Interrupt can occur anytime → critical sections must be synchronized
- Virtually every kernel data structure must be carefully accessed with proper synchronization

### Practice Questions with Answers

**Q1: What is the difference between a process and a thread?**

**Answer:**
- **Process**: Separate address space, own page table, heavy context switch, isolated
- **Thread**: Shares address space with other threads in same process, own stack/registers, light context switch, can directly access shared memory
- Switching threads in same process doesn't require changing page tables (fast)

**Q2: Two threads execute count++ (where count=0 initially). Each runs once. What are possible final values?**

**Answer:**
count++ compiles to: load→increment→store

Possible interleavings:
```
T1: load(0) → T1: inc(1) → T1: store(1) → T2: load(1) → T2: inc(2) → T2: store(2) = 2
T1: load(0) → T2: load(0) → T1: inc(1) → T1: store(1) → T2: inc(1) → T2: store(1) = 1
```
**Possible values: 1 or 2** (2 is correct, 1 is the race condition bug)

**Q3: What is a critical section?**

**Answer:** A critical section is a piece of code that accesses a shared resource (variable, data structure, file, etc.) that must not be concurrently accessed by multiple threads. Only one thread should execute the critical section at a time to avoid race conditions.

**Q4: Why are threads lighter weight than processes?**

**Answer:**
- Threads share the same address space (no need to switch page tables)
- Less state to save/restore during context switch
- Communication between threads is easier (shared memory vs IPC)
- Creating a thread is faster than forking a process (no copying address space)

**Q5: What parts of memory do threads share vs have private?**

**Answer:**
- **Shared**: Code, heap (malloc'd data), global variables, open file descriptors
- **Private**: Stack, registers (PC, SP, etc.), thread-local storage

**Q6: Name the three basic threading API functions.**

**Answer:**
- **pthread_create()**: Create a new thread
- **pthread_join()**: Wait for a thread to complete
- **pthread_exit()**: Terminate calling thread (or just return from thread function)

---

## Chapters 28/29: Locks

### The Crux: How to Build a Lock
**Question:** How can we build an efficient lock? Efficient locks provide mutual exclusion at low cost, and also might attain a few other properties. What hardware support is needed? What OS support?

### Lock Basics
```c
lock_t mutex; // lock variable (available/free or acquired/held)
lock(&mutex);  // acquire lock
// critical section - only one thread here at a time
unlock(&mutex); // release lock
```

A lock is just a **variable** that holds state: **available** (unlocked/free) or **acquired** (locked/held). Exactly one thread holds the lock and is presumably in a critical section.

**POSIX**: Uses `pthread_mutex_t` (mutex = mutual exclusion)
```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
Pthread_mutex_lock(&lock);
balance = balance + 1;
Pthread_mutex_unlock(&lock);
```

### Lock Evaluation Criteria

**1. Mutual Exclusion**: Does the lock work? Does it prevent multiple threads from entering critical section?

**2. Fairness**: Does each thread contending for the lock get a fair shot at acquiring it once it's free? Or can threads starve, never obtaining the lock?

**3. Performance**: Time overheads added by using the lock
- **No contention**: Single thread running, grabs and releases lock - overhead?
- **Single CPU, contention**: Multiple threads contending on single CPU - performance concerns?
- **Multiple CPUs, contention**: Threads on each CPU contending - how does it scale?

### Early Solutions

**Controlling Interrupts** (doesn't work on multiprocessors!)
```c
void lock() { DisableInterrupts(); }
void unlock() { EnableInterrupts(); }
```
- **Positives**: Simple
- **Negatives**:
  - Requires privileged operation (trust issues - greedy program could monopolize processor)
  - Doesn't work on multiprocessors (threads on other CPUs can still enter critical section)
  - Can lose interrupts (disk I/O completion, etc.)
- **Use**: OS itself uses interrupt masking for its own data structures (trust is not an issue)

**Failed Attempt: Just Loads/Stores** (Figure 28.1)
```c
typedef struct { int flag; } lock_t;
void lock(lock_t *lock) {
    while (lock->flag == 1) ; // TEST the flag
    lock->flag = 1;            // SET it!
}
```
**Problem**: Race condition! Two threads can both see flag==0, both exit while loop, both set flag=1, both enter critical section. **Mutual exclusion violated!**

### Atomic Instructions (Hardware Support)

**Test-and-Set** (a.k.a. atomic exchange) - Burroughs B5000, early 1960's
```c
int TestAndSet(int *ptr, int new) {
    int old = *ptr;  // fetch old value
    *ptr = new;      // store 'new' into ptr
    return old;      // return the old value
}
```
- Entire operation is **atomic** (executed as single instruction)
- On SPARC: `ldstub`, on x86: `xchg`
- Returns 0 if lock was free (enabling you to acquire it)

**Using Test-and-Set for Spin Lock** (Figure 28.3):
```c
void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1)
        ; // spin-wait (do nothing)
}
void unlock(lock_t *lock) {
    lock->flag = 0;
}
```
**Evaluation**: ✓ Correct (mutual exclusion), ✗ No fairness (can spin forever), ✗ Poor performance (single CPU wastes entire time slice spinning)

**Compare-and-Swap** (more powerful than test-and-set)
```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected)
        *ptr = new;
    return actual;  // caller knows if it succeeded
}
```
- Tests if value at ptr equals expected; if so, update to new
- Used similarly to test-and-set for locks:
```c
void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1)
        ; // spin
}
```

**Load-Linked and Store-Conditional** (MIPS, Alpha, PowerPC, ARM)
```c
int LoadLinked(int *ptr) {
    return *ptr;
}
int StoreConditional(int *ptr, int value) {
    if (no update to *ptr since LL to this addr) {
        *ptr = value;
        return 1; // success!
    } else {
        return 0; // failed to update
    }
}
```
**Using LL/SC** (Figure 28.6):
```c
void lock(lock_t *lock) {
    while (1) {
        while (LoadLinked(&lock->flag) == 1)
            ; // spin until it's zero
        if (StoreConditional(&lock->flag, 1) == 1)
            return; // success: acquired lock
        // otherwise: try again
    }
}
```

**Fetch-and-Add** (enables ticket locks - Mellor-Crummey & Scott)
```c
int FetchAndAdd(int *ptr) {
    int old = *ptr;
    *ptr = old + 1;
    return old;
}
```
**Ticket Lock** (Figure 28.7) - **ensures fairness!**
```c
typedef struct { int ticket; int turn; } lock_t;

void lock(lock_t *lock) {
    int myturn = FetchAndAdd(&lock->ticket);
    while (lock->turn != myturn)
        ; // spin
}
void unlock(lock_t *lock) {
    lock->turn = lock->turn + 1;
}
```
**Key difference**: Ensures progress for all threads (FIFO ordering). Once assigned ticket, you WILL be scheduled eventually. Test-and-set could spin forever.

### Spin Locks: Problem and Solutions

**Problem**: Spinning wastes CPU cycles
- Single CPU: If lock holder preempted, N-1 other threads spin for entire time slice each. Huge waste!
- Multiple CPUs: Works reasonably well if #threads ≈ #CPUs and critical sections are short

**Solution 1: Just Yield** (Figure 28.8)
```c
void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1)
        yield(); // give up CPU
}
```
- `yield()` moves caller from running→ready, deschedules itself
- Better than spinning, but with many threads (e.g., 100), all 99 others run-and-yield before lock holder runs again
- Still doesn't address starvation

**Solution 2: Queues - Sleeping Instead of Spinning** (Figure 28.9)
Use OS primitives: `park()` (put thread to sleep) and `unpark(threadID)` (wake specific thread)

```c
typedef struct {
    int flag;
    int guard;  // spin lock around flag and queue
    queue_t *q; // queue of waiting threads
} lock_t;

void lock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ; // acquire guard lock by spinning
    if (m->flag == 0) {
        m->flag = 1; // lock is acquired
        m->guard = 0;
    } else {
        queue_add(m->q, gettid());
        m->guard = 0;
        park(); // sleep
    }
}
void unlock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ; // acquire guard lock
    if (queue_empty(m->q))
        m->flag = 0; // let go of lock
    else
        unpark(queue_remove(m->q)); // wake one thread
    m->guard = 0;
}
```
**Key idea**: Guard lock protects flag and queue (spins only briefly). When can't get lock, add to queue and sleep. Woken thread gets lock directly (flag not set to 0).

**Wakeup/waiting race**: Solved by `setpark()` - call before releasing guard, so if unpark happens before park, subsequent park returns immediately.

**Linux futex** (fast userspace mutex):
- Combines ideas above
- Two calls: `futex_wait(address, expected)` and `futex_wake(address)`
- Single integer tracks both lock status (high bit) and number of waiters (other bits)
- Fast path (no contention): just atomic bit test-and-set
- Slow path: kernel manages wait queue

**Two-Phase Locks**:
- Phase 1: Spin for a while (hoping lock released soon)
- Phase 2: If not acquired, sleep until woken
- Hybrid approach: combines advantages of spinning (low latency if lock released quickly) and sleeping (doesn't waste CPU if lock held longer)

### Priority Inversion Problem

**Problem**: High-priority thread T3 waiting for lock held by low-priority T1. Medium-priority T2 starts and runs, preventing T1 from running. T3 (highest priority!) can't run because waiting for T1.

**Real-world**: Occurred on Mars Pathfinder!

**Solutions**:
- Avoid spin locks (use sleep locks)
- **Priority inheritance**: Temporarily boost lower thread's priority when higher-priority thread waits for it
- Ensure all threads have same priority

---

## Chapter 29: Lock-based Concurrent Data Structures

### The Crux: How to Add Locks to Data Structures
**Question:** When given a particular data structure, how should we add locks to it, in order to make it work correctly? Further, how do we add locks such that the data structure yields high performance, enabling many threads to access the structure at once, i.e., **concurrently**?

### Design Principles

**1. Start Simple**: Add a single big lock (monitor-like approach)
- Locks acquired when calling routine, released when returning
- If it's not too slow, you're done! (Avoid premature optimization - Knuth's Law)

**2. Enable More Concurrency**: Only if performance is a problem
- Use fine-grained locking strategies
- Be careful: more concurrency ≠ necessarily faster (overhead matters)

**3. Be Wary of Control Flow and Locks**:
- Functions that begin by acquiring lock must release before returning
- Error paths are error-prone (e.g., malloc fails - must unlock before return)
- Minimize lock/unlock points to reduce bugs

### Concurrent Counters

**Simple Counter** (Figure 29.2) - not scalable:
```c
typedef struct { int value; pthread_mutex_t lock; } counter_t;

void increment(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value++;
    Pthread_mutex_unlock(&c->lock);
}
```
**Problem**: With multiple threads, terrible performance (2 threads: 5+ seconds vs 0.03 for 1 thread, for 1M increments each)

**Approximate Counter** (Figure 29.4) - **scalable!**
- Uses **local counters** (one per CPU) and one **global counter**
- Each has own lock
- Thread updates local counter (no contention across CPUs)
- Periodically transfer local→global when local reaches threshold S
```c
typedef struct {
    int global;                  // global count
    pthread_mutex_t glock;       // global lock
    int local[NUMCPUS];         // per-CPU count
    pthread_mutex_t llock[NUMCPUS]; // per-CPU locks
    int threshold;               // update frequency
} counter_t;

void update(counter_t *c, int threadID, int amt) {
    int cpu = threadID % NUMCPUS;
    pthread_mutex_lock(&c->llock[cpu]);
    c->local[cpu] += amt;
    if (c->local[cpu] >= c->threshold) {
        pthread_mutex_lock(&c->glock);
        c->global += c->local[cpu];
        pthread_mutex_unlock(&c->glock);
        c->local[cpu] = 0;
    }
    pthread_mutex_unlock(&c->llock[cpu]);
}
```
**Tradeoff**: Threshold S controls accuracy vs performance
- Small S: more accurate, less scalable (behaves like simple counter)
- Large S: less accurate (global can lag by #CPUs × S), highly scalable
- Example: S=1024 gives excellent performance (4 threads almost as fast as 1 thread)

### Concurrent Linked Lists

**Basic Approach** (Figure 29.7): Single lock for entire list
```c
typedef struct {
    node_t *head;
    pthread_mutex_t lock;
} list_t;

int List_Insert(list_t *L, int key) {
    pthread_mutex_lock(&L->lock);
    node_t *new = malloc(sizeof(node_t));
    if (new == NULL) {
        perror("malloc");
        pthread_mutex_unlock(&L->lock); // Must unlock on error path!
        return -1;
    }
    new->key = key;
    new->next = L->head;
    L->head = new;
    pthread_mutex_unlock(&L->lock);
    return 0;
}
```

**Improved Version** (Figure 29.8): Lock only critical section
```c
int List_Insert(list_t *L, int key) {
    node_t *new = malloc(sizeof(node_t)); // OUTSIDE lock
    if (new == NULL) {
        perror("malloc");
        return -1; // No unlock needed!
    }
    new->key = key;
    pthread_mutex_lock(&L->lock);  // Lock only critical section
    new->next = L->head;
    L->head = new;
    pthread_mutex_unlock(&L->lock);
    return 0;
}
```
**Benefit**: Reduces lock/unlock points, malloc doesn't need lock (thread-safe)

**Hand-over-hand Locking** (a.k.a. lock coupling):
- Instead of one lock for entire list, lock per node
- When traversing: grab next node's lock, then release current node's lock
- Enables high concurrency in list operations
- **Problem**: Hard to make faster than simple single lock (overhead of acquiring/releasing many locks)

### Concurrent Queues

**Michael and Scott Queue** (Figure 29.9): Two locks for better concurrency
```c
typedef struct {
    node_t *head;
    node_t *tail;
    pthread_mutex_t head_lock, tail_lock;
} queue_t;

void Queue_Enqueue(queue_t *q, int value) {
    node_t *tmp = malloc(sizeof(node_t));
    tmp->value = value;
    tmp->next = NULL;
    pthread_mutex_lock(&q->tail_lock);  // Only tail lock
    q->tail->next = tmp;
    q->tail = tmp;
    pthread_mutex_unlock(&q->tail_lock);
}

int Queue_Dequeue(queue_t *q, int *value) {
    pthread_mutex_lock(&q->head_lock);  // Only head lock
    node_t *tmp = q->head;
    node_t *new_head = tmp->next;
    if (new_head == NULL) {
        pthread_mutex_unlock(&q->head_lock);
        return -1; // queue empty
    }
    *value = new_head->value;
    q->head = new_head;
    pthread_mutex_unlock(&q->head_lock);
    free(tmp);
    return 0;
}
```
**Key ideas**:
- Separate locks enable concurrency of enqueue and dequeue
- Dummy node enables separation of head/tail operations
- Common case: enqueue only uses tail_lock, dequeue only uses head_lock

### Concurrent Hash Tables

**Simple and Effective** (Figure 29.10):
```c
#define BUCKETS (101)
typedef struct {
    list_t lists[BUCKETS];
} hash_t;

int Hash_Insert(hash_t *H, int key) {
    return List_Insert(&H->lists[key % BUCKETS], key);
}
```
**Why it scales magnificently**:
- **Lock per hash bucket** instead of single lock for entire structure
- Many concurrent operations can proceed (different buckets accessed concurrently)
- Simple concurrent list used for each bucket

**Performance**: Concurrent hash table with 4 threads scales almost perfectly. Simple linked list with single lock does not scale at all.

**Key Lesson**: Instead of single lock (coarse-grained), use multiple locks for different parts of structure (fine-grained) to enable more concurrency.

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
