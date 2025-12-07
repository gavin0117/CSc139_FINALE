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

### Why Concurrency?
- **Parallelism**: Use multiple CPUs simultaneously
- **Avoid blocking**: One thread waits for I/O, another runs
- **Better structure**: Separate concerns (e.g., UI thread + worker thread)

### Thread vs Process
- **Process**: Own address space, heavy context switch
- **Thread**: Share address space, light context switch
- Threads share: code, heap, globals
- Threads private: stack, registers

### The Problem: Race Conditions
- **Race condition**: Outcome depends on execution timing
- Occurs when multiple threads access shared data without synchronization
- **Critical section**: Code that accesses shared resource

### Visual: Thread Memory Layout
```
Process Address Space:
┌──────────────┐
│    Code      │ ← Shared by all threads
├──────────────┤
│    Heap      │ ← Shared
├──────────────┤
│  Thread 1    │ ← Private stack
│   Stack      │
├──────────────┤
│  Thread 2    │ ← Private stack
│   Stack      │
└──────────────┘
```

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

