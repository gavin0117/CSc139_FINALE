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

