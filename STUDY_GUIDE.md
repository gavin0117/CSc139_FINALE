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

