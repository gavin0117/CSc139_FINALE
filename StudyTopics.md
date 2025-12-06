# OS :: Final Study Guide \- FA25.1

### **Overview**

For this course the book itself is an excellent study guide and you should expect questions that tie very closely to how the ideas are presented in your book. Note that I omit the dialogue chapters. 

You will see many similar questions on the final to the questions on the module quizzes. Do not expect the exact same questions, but do expect that many questions may only have slight changes, e.g., the distractors re-arranged, or slightly reworded. The OA questions will leverage many of the same skills but you should also expect a very few OA questions on topics not seen in the module quizzes. 

**Chapter 2 :: Introduction**

* Much of the intro lecture was related to computer history. There will not be any specific questions that require you to know details dates or events regarding this material. The purpose was to frame the development of operating systems for you.  
* I'm not going to pull any punches on Chapter 2\.  It's short, it provides a good overview, so you should expect that questions can come from any of its 20 pages. If you haven't read them until now, then it's time to correct that.   
* 

### **CPU Virtualization**

**Chapter 4 :: The Process Abstraction**

* Know all of the terms and their definitions,  
* Know and understand the process states and the life of a process  
* Make sure that you can create the simple state diagram from memory, you may not have to, but I won't give it to you on the exam even if I ask you questions about the diagram.  
* Know the basic data structure concepts presented here, but there is no need to study the assembler code in detail. 

**Chapter 5 :: The Process API**

* Expect detailed questions about how to use the process API. In essence, anything on the three assignments is completely on topic for this exam. 

**Chapter 6 :: Limited Direct Execution**

* This is an important concept chapter that I spent significant time in lecture discussing. You must understand the idea in detail.  
  * What does it mean?  
  * Why does it need to be limited?  
  * What are the advantages and limitations of unlimited direct execution?  
  * How does the OS enforce limited direct execution?

**Chapter 7/8/9 :: Scheduling**

* Know all scheduling algorithms in Chapter 7 and 8  
* Be familiar with the basics of Stride Scheduling from Chapter 9, we did not discuss the CFS.   
* Be able to answer questions about which job will be scheduled next given an algorithm and a set of jobs  
* Be able to compute any metrics related to any of the algorithms in Chapter 7  
* The end of chapter questions will give you good insight on how to be prepared for questions related to this chapter.

**More generally with respect to the Process Abstraction**

* We did talk quite a bit about the mechanics of how the timer interrupt is key to the OS being able to take control of the processor. Make sure that you have a solid understanding of how and when the OS can take control and make changes.   
* Have a basic understanding of what goes on in bootup including some of the basic responsibilities of the operating system, e.g., setting up the interrupt handlers. 

### **Memory Virtualization**

**Chapter 13 :: Address Spaces**

* Make sure that you understand the different models of address spaces presented and be able to relate them to different requirements in the evolution of operating systems.  
* Why do we need something like paging or segmentation?  
* What involvement does the operating system have in setting up the address space for a process

**Chapter 14 :: Memory API**

This is a basic chapter that shouldn't really be very knew to you. That said, you will not be asked deep questions about calls such as mmap or sbrk. 

* Make sure that you fully understand the roll of malloc and free and where they allocate memory   
* You should understand the basic roll of the OS calls such as mmap or sbrk, but you are not required to understand how to use them in detail or know the parameters that you pass to them. 

**Chapter 15: Mechanism \- Address Translation**

* We spent a lot of time on this content in detail so it's really all on the table.   
* Make sure that you understand the relationship between virtual address and physical addresses and in which space the program runs and what happens in hardware generally whenever we are relying on address translation  
* Make sure that you understand what roll the hardware plays in address translation.   
* You don't need to memorize x86 assembler, but, you may be asked about assembler operations so you should be able to recognize and reason about simple assembler that loads values into memory or into registers.   
  * You should know the names of the x86 core registers. I'm not going to quiz you on this specifically, but if you don't know that eax is a general purpose register then you might not be able to understand some questions.

**Chapter 16 :: Segmentation**

* Make sure that you understand how base and bound work and that segmentation is essentially a generalized base and bounds  
* Make sure that you know which aspects of the system are per/process and which are common to all processes  
* Make sure that you know the basics of what happens on a context switch  
* Review but don't dwell on 16.4-16.6

**Chapter 17 :: Free Space Management**

* Because of the Memory Allocator Project., you should be very very comfortable with everything through section 17.3  
* Review but don't dwell on 17.4  
* Expect detailed questions on this because of the Memory Allocator Project.

**Chapters 18/19/20 :: Paging**

This is a important topic and one that students often have some trouble with the details. You will be asked some questions about the details in this section. I've grouped these chapters together. 

* You should be able to reason about the number of virtual pages and the number of physical pages as well as the number of bits in each address given a particular set of parameters.  
* Know what is executed in hardware and what is executed in software given RISC or CISC contexts  
* Know who is responsible for the code executed in software  
* Know the purpose of the TLB  
* Be able to compute the hit rate of the TLB given the parameters  
* Understand the motivations for each additional mechanism in the system  
  * What does the TLB contribute and why do we need it?  
  * What's the point of an inverted or a multi-level page table?  
* Know what memory MUST be allocated for a given page table structure.

In short, you should spend sufficient time with these chapters to make sure that you understand the concepts and detail involved. I don't just mean understand at a surface level, you should be able to reason about what benefits and costs a particular configuration provides.  

**Chapters 26/27 :: Introduction to Concurrency**

We didn't cover chapter 27 in class but your project should have introduced you to the details. 

* Review the motivations and problems of concurrency and the basics of the threading API

**Chapters 28/29 :: Locks**

Most of chapter 28 and 29 are on topic.

* We did not cover two-phase locks in much detail nor did we spend significant time evaluating locks.  
* Focus on the conceptual motivation for the different types of atomic instructions.  
  * Expect deeper questions related to:  
    * Test \-and-Set  
    * Fetch-and-add  
* Be able to answer questions on how spin locks work vs alternatives.   
* Be prepared to analyze small pieces of software for race conditions as we did in class.  
* Review the concurrent data structures in Chapter 29 that we covered in class. Pay attention to the basic principles. 

**Chapter 30 :: Condition Variables**

This is not much different from 29\.

* Make sure that you understand how the bounded buffer problem works  
* How do condition variables work with locks for bounded buffer  
* Be prepared to analyze small pieces of software for race conditions as we did in class.  
* Know what a covering condition is. 

**Chapter 31 :: Semaphores**

Again, we covered almost all of this. 

* Know how we can use semaphores for either locks or condition variables.   
* Be prepared to analyze small pieces of software for race conditions as we did in class.  
* Recognize the canonical solution given in the book to the Dining Philosopher's problem.

**Chapter 32 :: Concurrency Bugs**

We’ve already seen many of these in action.

* Know the two major non-deadlock bug types: atomicity violations (failing to protect logically grouped accesses) and order violations (assuming operations happen in a specific sequence).  
* Understand that real-world studies show most concurrency bugs are *not* deadlocks—race conditions are far more common.  
* Be able to identify and fix atomicity and order violations using locking and condition variables.  
* Learn the four conditions that make deadlock possible: mutual exclusion, hold-and-wait, no preemption, and circular wait.  
* Recognize strategies to prevent deadlock, including total lock ordering, trylock loops to break cycles, and lock-free data structures via compare-and-swap.

**Chapters 36 through 41 ::  I/O and File Systems**

We covered this minimally. We talked about disks  and basics of I/O towards the end. 

* Know the roles of the basic elements of an I/O canonical device, e.g., status register, command register  
* Be able to think about when interrupts vs polling might be appropriate. Don't spend much time on this, just understand the basic mechanisms.  
* Know the basic terms of disk geometry  
* Recognize the various disk scheduling algorithms  
* Understand the basics of FFS mechanisms. 

