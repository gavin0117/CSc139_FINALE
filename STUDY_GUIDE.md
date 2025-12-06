# Operating Systems Final Exam Study Guide

## Chapter 4: The Process Abstraction

### Key Terms & Definitions
- **Process**: A running program; the OS abstraction for execution
- **Program**: Static code and data on disk
- **Address Space**: Memory the process can access (code, stack, heap)
- **Process API**: Create, destroy, wait, control, status operations
- **Machine State**: What a program can read/update (memory, registers, PC, stack pointer, frame pointer)

### Process States
1. **Running**: Executing on CPU
2. **Ready**: Ready to run but OS chose not to run it
3. **Blocked**: Not ready (e.g., waiting for I/O)

### State Transitions
- Ready → Running: **Scheduled**
- Running → Ready: **Descheduled**
- Running → Blocked: **I/O initiated**
- Blocked → Ready: **I/O completion**

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
- **exec()**: Transforms calling process into a different program
- **wait()**: Parent waits for child to complete
- **kill()**: Send signals to processes
- **signal()**: Set up signal handlers

### fork() Behavior
- Returns **0** to child process
- Returns **child's PID** to parent
- Returns **-1** on failure
- Child gets copy of parent's address space

### exec() Family
- Loads new program into current process
- Does NOT create new process
- Does NOT return on success (old program is gone)
- Common variants: execl(), execle(), execv(), execvp()

### Practice Questions

1. What does fork() return to the parent process? To the child?

2. After fork(), how many processes are running? What is the relationship between them?

3. Why would you call fork() followed by exec()?

4. Write the typical pattern: parent forks a child, child runs `/bin/ls`, parent waits for completion.

5. What happens if you call exec() without fork() first?

6. If fork() fails, what value does it return and why might it fail?

7. What is the purpose of wait() or waitpid()?

8. If a parent doesn't call wait() on its child, what problem can occur?

9. What happens to open file descriptors after fork()?

10. Explain the output when this code runs:
    ```c
    printf("A");
    fork();
    printf("B");
    ```

11. What is a zombie process?

12. What is an orphan process?

13. How does the shell use fork() and exec() to run commands?

14. Can a child process affect the parent's variables after fork()?

15. What signal does kill() send by default?

---

## Chapter 6: Limited Direct Execution

### Core Concept
**Direct Execution**: Run program directly on CPU (fast but problematic)
**Limited Direct Execution**: OS restricts what programs can do via hardware support

### Two Main Problems
1. **Restricted Operations**: How to prevent user programs from doing whatever they want?
2. **Switching Between Processes**: How does OS regain control of CPU?

### User Mode vs Kernel Mode
- **User Mode**: Restricted; cannot issue I/O requests or privileged instructions
- **Kernel Mode**: Full privileges; OS runs in this mode
- **System Call**: User program requests OS to perform privileged operation
- **Trap**: Enters kernel mode (raises privilege)
- **Return-from-trap**: Returns to user mode (lowers privilege)

### System Call Mechanism
1. Program issues system call
2. **Trap** into kernel (save registers, switch to kernel mode)
3. OS handles request via **trap handler**
4. OS executes **return-from-trap** (restore registers, switch to user mode)
5. Program continues

### Timer Interrupt
- Hardware timer interrupts CPU periodically
- Gives OS chance to regain control
- OS can decide to switch to different process (context switch)

### Context Switch
- OS saves register state of current process (PCB)
- OS restores register state of next process (PCB)
- Switch to new process's kernel stack
- Return-from-trap returns to new process

### Practice Questions

1. What does "Limited Direct Execution" mean?

2. Why does direct execution need to be "limited"?

3. What are the advantages of unlimited direct execution? What are the problems?

4. What is the difference between user mode and kernel mode?

5. How does a user program perform a privileged operation (like reading from disk)?

6. What happens during a trap instruction?

7. What happens during return-from-trap?

8. How does the OS set up trap handlers? When does this happen?

9. Why is a timer interrupt necessary for the OS?

10. Without a timer interrupt, what could a malicious program do?

11. Describe the complete flow when a program makes a system call to read a file.

12. What is a context switch and when does it occur?

13. What information must be saved during a context switch?

14. How does the OS enforce limited direct execution?

15. What is the trap table and when is it initialized?

16. If a program runs a tight loop with no system calls, how does the OS regain control?

17. What privilege level does user code run at? What about OS code?

18. Why can't user programs directly access hardware?

19. Explain the cooperative vs non-cooperative approach to regaining CPU control.

20. What happens on boot with respect to trap handlers and timer interrupts?

---

## Chapter 7 & 8: Scheduling

### Key Scheduling Algorithms

**FIFO (First In First Out)**: Run jobs in order of arrival
**SJF (Shortest Job First)**: Run shortest job first (non-preemptive)
**STCF (Shortest Time-to-Completion First)**: Preemptive SJF
**Round Robin (RR)**: Time slice each job in turns
**MLFQ (Multi-Level Feedback Queue)**: Multiple queues with different priorities

### Key Metrics
- **Turnaround Time**: T_completion - T_arrival
- **Response Time**: T_first_run - T_arrival
- **Average**: Sum of all times / number of jobs

### Visual: Process Scheduling Timeline
```
Jobs: A(100ms), B(10ms), C(10ms) all arrive at t=0

FIFO:     |----A----|B|C|
          0        100 110 120
Avg Turnaround: (100+110+120)/3 = 110ms

SJF:      |B|C|----A----|
          0 10 20      120
Avg Turnaround: (10+20+120)/3 = 50ms

Round Robin (time slice=10):
|A|B|C|A|A|A|A|A|A|A|A|A|
Avg Response: (0+10+20)/3 = 10ms
```

### Practice Questions with Answers

**Q1: Given jobs A(50ms), B(30ms), C(20ms) arriving at t=0, calculate average turnaround time for SJF.**

**Answer:**
- Run order: C, B, A
- C completes at 20ms (turnaround = 20)
- B completes at 50ms (turnaround = 50)
- A completes at 100ms (turnaround = 100)
- Average = (20+50+100)/3 = **56.67ms**

**Q2: What problem does STCF solve that SJF cannot handle?**

**Answer:** STCF is preemptive, so when a shorter job arrives, it can preempt the currently running job. SJF is non-preemptive and must wait for the current job to finish, leading to convoy effect when short jobs arrive after a long job has started.

**Q3: Jobs A(100ms) arrives at t=0, B(10ms) arrives at t=10. Calculate average turnaround for FIFO vs STCF.**

**Answer:**
- **FIFO:** A finishes at 100, B finishes at 110
  - Avg turnaround = ((100-0)+(110-10))/2 = (100+100)/2 = **100ms**
- **STCF:** B preempts A at t=10, finishes at t=20, A finishes at t=120
  - Avg turnaround = ((120-0)+(20-10))/2 = (120+10)/2 = **65ms**

**Q4: Why is Round Robin bad for turnaround time but good for response time?**

**Answer:** RR gives every job a quick time slice, so all jobs start running quickly (good response time). However, jobs take longer to complete because they're constantly being switched out, increasing turnaround time. The overhead of context switching also adds to completion time.

**Q5: In MLFQ, why do we periodically boost all jobs to the highest priority queue?**

**Answer:** To prevent starvation of long-running jobs and to handle jobs that change behavior (e.g., from CPU-bound to interactive). Without boosting, a job that used up its time slices early would stay in low-priority queues forever, even if it becomes interactive later.

**Q6: Three jobs arrive: A at t=0 (runtime=30), B at t=5 (runtime=20), C at t=10 (runtime=10). Using RR with time slice=10, when does each job complete?**

**Answer:**
```
Timeline:
0-10: A runs (20 left)
10-20: B runs (10 left)
20-30: C runs (done at 30)
30-40: A runs (10 left)
40-50: B runs (done at 50)
50-60: A runs (done at 60)
```
- C completes at **30ms**
- B completes at **50ms**
- A completes at **60ms**

**Q7: What is the convoy effect and which algorithm suffers from it?**

**Answer:** Convoy effect occurs when short jobs get stuck behind a long job, like cars stuck behind a slow truck. **FIFO** suffers from this because if a long job arrives first, all subsequent short jobs must wait for it to complete, even though running the short jobs first would minimize average turnaround time.

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

1. **No Abstraction**: One program directly uses physical memory
2. **Time Sharing**: OS switches processes in/out of memory (slow)
3. **Address Space**: Each process has its own virtual memory view

### Address Space Structure
```
Virtual Memory Layout:
0KB   ┌──────────────┐
      │   Code       │  (Program instructions)
      ├──────────────┤
      │   Heap       │  (malloc data, grows DOWN→)
      │      ↓       │
      │              │
      │      ↑       │
      │   Stack      │  (local vars, grows UP←)
16KB  └──────────────┘
```

### Key Concepts
- **Virtual Address**: What program sees (0KB to 16KB)
- **Physical Address**: Actual location in RAM
- **Address Translation**: Hardware+OS maps virtual → physical
- **Isolation**: Each process thinks it owns all memory (0 to max)

### Practice Questions with Answers

**Q1: Why do we need address spaces instead of letting programs use physical memory directly?**

**Answer:**
- **Isolation**: Prevents processes from accessing each other's memory (security/stability)
- **Ease of use**: Program doesn't need to know where in physical RAM it will run
- **Flexibility**: OS can move process in memory, swap to disk, without program knowing
- **Protection**: OS can enforce read-only code sections, prevent stack overflows into heap

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

**Q4: In early time-sharing systems without address spaces, how did the OS switch between processes?**

**Answer:** The OS would save the entire process memory to disk, load the next process's memory from disk into RAM, then run it. This was extremely slow due to I/O overhead, making multiprogramming inefficient.

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

---

## Chapter 15: Address Translation

### Base and Bounds (Dynamic Relocation)

**Hardware Registers (per-CPU):**
- **Base register**: Start of physical memory for process
- **Bounds register**: Size of address space

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
- Sets base/bounds registers during context switch
- Handles exceptions (e.g., kills process on seg fault)
- Manages free memory (which physical regions are free)

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

### Problem with Base & Bounds
- Wastes memory (internal fragmentation)
- If heap and stack are small, large gap between them is unused but still allocated

### Segmentation = Generalized Base & Bounds
- Separate base & bounds for each **segment** (code, heap, stack)
- Each segment can be placed independently in physical memory

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

