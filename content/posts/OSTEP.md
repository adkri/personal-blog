---
title: "Notes on Operating System: Three Easy Pieces"
date: 2019-01-27T16:04:56-04:00
draft: true

tags: ["books", "operating system"]
---


# CPU Virtualization



## Processes

a process is a running program 

the os virtualizes the cpu

by running one program, then stopping it and running another, the os promotes the illusion that many cpu's exist

this basic technique is called time sharing of the CPU, allows users to run as many concurrent processes as they like 



to implement virtualuzation of the cpu, os needs:

low-level machinery - mechanisms eg. time-sharing

high-level intelligence - policies eg. scheduling policy



what is the machine state of a process? it is what a program can read or update when it is running. 

one  component is memory, instructions lie in memory; data the process reads  and writes also in memory. the memory the process can address is its  address space

another are registers, many instructions read or update registers and thus important for execution



some particularly special registers form machine state

the  program counter(PC) or sometimes called the instruction pointer(IP)  tells us which instruction of program is currently executed.

stack pointer and frame pointer are used to manage the stack for function parameters, local variables and return addresses



programs often access persistent storage devices too. such IO information might include a list of files the process has open



## Process API

what must be included in any interface of an modern operating system 

- create: os must include some method to create new processes
- destroy: as there is interface to for process creation, also should be interface to destroy processes forcefully
- wait: sometimes it is useful to wait for a process to stop running
- misc control: other than killing or waiting, we need suspend and resume
- status: there are usually interfaces to get some status information about a process, how long it has run for, or what state it is in

Process Creation

how does a process get created from a program

the first thing a os must do is load its code and any static data into memory

programs initially reside on disk in some kind of executable format, thus, the process of loading a program and static data to memory requires the os to read those bytes from disk and place them into memory

in early os, the loading process is done eagerly, all at once before running the program; modern os perform the process lazily, by loading pieces of code or data as they are needed during execution 

once the code and static data are loaded, few other things 

some memory must be allocated for the program's run-time stack

c programs use the stack for local variables, function parameters and return addresses; the os allocates this memory and gives it to the process

os will initialize the stack with arguments; it will fill in the parameters to main i.e. argc and the argv array

os may also allocate memory for heap

in c programs heap is used for dynamically allocated data

request space by malloc() and free by free()

the heap is needed for data structures like lists, hash table, trees

the heap is small at first, as program runs heap grows

the os will also do some other init tasks, related to io

for example in unix, each process by default has three file descriptors, for standard input, output, error

- by loading the code and static data into memory

- by creating and initializing a stack 

- by doing other work for io setup

the os finally has set the stage for program execution

now last task to start the program running at entry point, main()

by jumping to main() routine, the os transfers control of the cpu to the newly created process, and program begins execution\

Process States

a process can be in one of three states:

- running: process is running on a processor. it is executing instructions
- ready: process is ready to run but for some reason the os has chosen not to run it at this moment 
- blocked: process has performed some kind of operation that makes it not ready. when process initiates an IO request to a disk, it becomes blocked and thus some other process can use the processor

a process can be moved between the ready and running states

once a process has been blocked the os keeps it in that stage until some event occurs; at that point the process moves to the ready state again

after it gets to ready state from blocked, it has to wait for os to switch from another running process to this one

the decisions is make by the **os scheduler** 

Data Structures

the os is a program and it has some key data structures that track various info

to track the state of each processes the os will keep a **process list/task list**

the os must also track blocked processes, when IO event completes and os should make sure to wake the correct process and ready it to run again

the **register context** will hold the contents of registers for a stopped process

when a process is stopped its registers will be saved to memory location; by restoring these registers the os can resume running the process 

sometimes a process will have other states: 

- initial, when it is created
- final, when it has exited but not yet cleaned up; also called zombie state; usually programs in unix return 0 when they accomplished task successfully, and non-zero otherwise
- when finished parent will make one final call, wait() to wait for the completion of the child, also indicate to os that it can clean up

each entry in process list is sometime found in a **process control block** that is a structure that contains info about a specific process



## Process API

process creation in UNIX systems 

UNIX presents one of the most intriguing way to create a new process with a pair of systems calls: *fork()* and *exec()*

a third routine *wait()* can be used by a process wishing to wait for a process it has created to complete

### The fork() system call

the fork() is used to create a new process; 

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

int main(int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if(rc < 0) { // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) { // child/ new process
        printf("hello, i am child (pid:%d)\n", (int) getpid());
    } else {
        printf("hello, i am parent of %d (pid:%d)\n", rc, (int) getpid());
    }
    return 0;
}
```

when this program is first started running, the process prints its **process identifier(PID)**

now the interesting part, the process calls the fork() system call, the process that is created is an exact copy of the calling process. the newly started process doesn't start running at main(); it just comes to life as if it had called fork() itself

the child isn't an exact copy. specifically, it now has its own copy of the address space, its own registers, its own PC, and the value it returns to the caller of fork() is different. 

the parent receives the PID of the new process

the child receives a return code of 0

this differentiation is useful, and it is easier to handle the two different cases

the output of the program is not deterministic. when the child process is created ; there are now two active processes in the system

we cannot tell which of them will run first



### the wait() system call

sometimes it is quite useful for a parent to wait for a child process to finish what is had been doing

this task is accomplished by the wait() system call/or its more complete sibling waitpid()

if we call wait() from parent, the system call won't return until the child had run and exited

thus even when the parent has run first, it politely waits for child to finish waiting



### the exec() system call

a final and important piece of process creation api is the exec() system call

this system call is important when you want to run a program different than the calling program

on linux, there are six variants of exec():

- execl
- execlp
- execle
- execv
- execvp
- execvpe

read man pages to learn more

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
	printf("hello world (pid:%d)\n", (int) getpid());
	int rc = fork();
	if (rc < 0) { // fork failed; exit
		fprintf(stderr, "fork failed\n");
		exit(1);
	} else if (rc == 0) { // child (new process)
		printf("hello, I am child (pid:%d)\n", (int) getpid());
		char *myargs[3];
		myargs[0] = strdup("wc");
        // program: "wc" (word count)
        myargs[1] = strdup("p3.c"); // argument: file to count
        myargs[2] = NULL;
        // marks end of array
        execvp(myargs[0], myargs); // runs word count
        printf("this shouldn’t print out");
    } else {
        // parent goes down this path (main)
        int rc_wait = wait(NULL);
        printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n",
        rc, rc_wait, (int) getpid());
    }
    return 0;
}
```



the fork() system call is strange; its partner in crime exec() is not so normal

what it does:

given the name of an executable and some arguments, it loads code and static data from that executable and overwrites its current code segment and current static data with it; the heap and the stack and other parts of memory space are reinitialized. then os simply runs that program, passing in any arguments as the argv of that process

it does not create a new process; rather it transforms the currently running process into a different running program

after the exec() in the child; it is almost as if the previous process never ran; a successful call to exec() never returns



### Why? Motivating the API

one big question is why would we build such an odd interface

the separation of fork() and exec() is essential in building a UNIX shell, because it lets the shell run code after the call to fork() but *before* the call to exec(); this code can alter the environment of the about-to-be-run program, and thus enables a variety of interesting features to be built

the shell is just a user program ; it shows a **prompt** and then waits for a command. in most shells it figures out where in the file system the executable resides, calls fork() to create a new child process to run the command, calls some variant of the exec() to run the command; then waits for command to complete by wait(). when wait() returns, it prints out a prompt again, ready for next command

the separation of fork() and exec() allows the shell to more useful things:

prompt > wc p3.c  >  newfile.txt

this above example, the output of the program is redirected into the output file. the way the shell accomplishes this task is quite simple: 

when the child is created, before calling exec() ; the shell closes standard output and opens the file newfile.txt; by doing so any output from new process wc are sent to the file instead of the screen

the reason this redirection works is due to an assumption about how the os manages file descriptors

specifically UNIX systems start looking for free file descriptors at zero. in this case, STDOUT_FILENO will be the first available one and thus get assigned when open() is called. subsequent writes by the child process to the standard output file descriptor, for example by printf() will then be routed transparently to the newly-opened file instead of screen

```cassandra
int main(int argc, char *argv[]) {
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) { // child: redirect standard output to a file
        close(STDOUT_FILENO);
        open("./p4.output", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);
        
        // now exec "wc"...
        char *myargs[3];
        myargs[0] = strdup("wc");
        myargs[1] = strdup("p4.c");
        myargs[2] = NULL;
        execvp(myargs[0], myargs);
    } else {
        // parent
        int rc_wait = wait(NULL);
    }
	return 0;
}
```

UNIX pipes are implemented in a similar way, but with pipe() system call

in this case, the output of one process is connected  to an inkernel pipe (i.e. queue) and the input of another process is connected to that same pipe; thus, the output of one process seamlessly is used as input to the next

long and useful chains of commands can be strung together

as a simple example consider looking for a word in a file, and then counting how many times said word occurs; with pipes and the utilities grep and wc

grep -o foo file | wc -l 

fork()/exec() combination is powerful way to create and manipulate processes

Note: read the man pages for whatever shell you are using 



### Process Control and Users

beyond fork(), exec() and wait(), there are a lot of other interfaces for interacting with processes in UNIX systems

for example, kill() system call is used to send signals to a process, including directives to pause, die and other useful imperatives

in most UNIX shells, certain keystroke combinations are configured to deliever a specific signal to the currently running process;

control-c sends a SIGINT(interrupt) to the process/terminating it

control-z sends a SIGTSTP(stop) signal thus pausing it in mid-execution; can resume it later with builtin *fg* command

the entire signals subsystem provides a rich infrastructure to deliever external events to processes, including ways to receive and process those signals within individual processes, and ways to send signals to individual processes as well as entire **process groups**

to use this form of communication, a process should use the signal() system call to catch() various signals; it ensures that when a particular signal is delievered to a process; it will suspend its normal execution and run a particular piece of code in response to signal

who sends signal to a process, and who cannot?

the systems we use have multiple people using them at the same time, if one of these people can arbitrarily send signals such as SIGINT, the usability and security of the system will be compromised

as a result, modern systems have strong conception of **user**

the user after entering a password to establish credentials, logs in to gain access to system resources; the user may then launch one or many processes and exercise full control over them

users can generally only control their own processes; it is job of the os to parcel out resources to each user to meet overall system goals



### Useful Tools

the *ps* command allows you to see which processes are running

the *top* command displays the processes of the system and how much CPU and other resources it is eating up

the command *kill* can be used to send signals to processes



- each process has a name; that name is a number knowns PID
- fork() system call is used in UNIX to create a new process; the creator the parent and newly created process the child
- the wait() system call allows a parent to wait for its child to complete execution
- the exec() family of system calls allows a child to break free from its similarity to its parent and execute a new program
- a UNIX shell commonly uses these 3 calls to launch user commands; the separation of fork and exec enables features like input/output redirection, pipes, and other cool features
- process control is available in form of signals, which can cause jobs to stop, continue or even terminate
- which processes can be controlled by a person is encapsulated in notion of user
- a superuser can control all processes





## Direct Execution

in order to virtualize the CPU, the os needs to share the physical CPU among many jobs running seemingly at the same time

the basic idea is simple: run one process for a little while, then run another one

by **time sharing** the CPU, virtualization is achieved

there are a few challenges

- performance
- control

the basic idea is Limited Direct Execution

direct execution -> run program directly on the CPU



### Problem 1: Restricted Operations

this is simple but has some problems. how does os stop it? how does it perform restricted operations?

the approach is a new processor mode, **user mode**

code in user mode is restricted in what it can do;can't IO

then os runs in **kernel mode**, code can do what it likes

so for process to do privileged operation, user programs perform **system call**

to execute a system call, program execute **trap** instruction

trap jumps into kernel; does IO operation

os calls **return-from-trap** instruction; back to user mode

the kernel sets up a **trap table** at boot time

sets up the hardware for what code to run when events occur

example, disk interrupt, keyboard interrupt, program makes system call

tells hardware where the trap handler code is

to specify syscall, a system-call number is assigned

the user code is thus responsible for placing the desired system-call number in register

the trap handler examines the number, executes corresponding code

two phases to the limited direct execution protocol

- at boot, kernel initializes trap table
- when running process, kernel sets up things before return-from-trap

### Problem 2: Switching Between Processes

a cooperative approach:wait for system calls

in this style the os trusts processes to behave

most processes transfer control to os by syscalls, to open file, create process

or when they do something illegal



a non-cooperative approach: the os takes control

a timer interrupt: runs every so many millis

when interrupt raised process halts, interrupt handler in os runs

now os has control of cpu



### Saving and  Restoring Context

after the timer interrupt, **scheduler** decides to keep running or make switch to another process

if to switch, the os does **context switch** 

in a context switch, os saves a few register values for current process and restore the registers for soon-to-execute process

the registers are saved to kernel stack

- The CPU should support at least two modes of execution: a re-
  stricted user mode and a privileged (non-restricted) kernel mode. 
- Typical user applications run in user mode, and use a system call
  to trap into the kernel to request operating system services.
- The trap instruction saves register state carefully, changes the hard-ware status to kernel mode, and jumps into the OS to a pre-specified destination: the trap table.
- When the OS finishes servicing a system call, it returns to the user
  program via another special return-from-trap instruction, which re-duces privilege and returns control to the instruction after the trap that jumped into the OS.
- The trap tables must be set up by the OS at boot time, and make
  sure that they cannot be readily modified by user programs. All
  of this is part of the limited direct execution protocol which runs
  programs efficiently but without loss of OS control.
- Once a program is running, the OS must use hardware mechanisms to ensure the user program does not run forever, namely the timer interrupt. This approach is a non-cooperative approach to CPU scheduling.
- Sometimes the OS, during a timer interrupt or system call, might
  wish to switch from running the current process to a different one,
  a low-level technique known as a context switch.



# Memory Virtualization



## Address Spaces

it is the running program's view of memory in the system

the address space of a process contains all of the memory state of the running program

the code, stack and heap are in the address space

the program code lives at top of the address space at 0; the heap at top after the code; and the stack at the bottom. this allows the stack and heap to grow 

the address space is an abstraction that the os provides to running program

the program shouldn't be aware of this virtualization

goals:

- transparency: it behaves as it has its own private physical memory
- efficiency: time and space
- protection: protect processes from one another and itself. isolation of address space



## Memory API

stack memory - allocations and deallocations are managed implicitly by the compiler or programmer; automatic memory

heap memory - allocations and deallocations are explicitly handled by the programmer



## Address Translation

technique used - hardware-based address translation

the hardware translates each memory access(eg., an instruction fetch, load, or store) changing the virtual address provided by instruction to a physical address

### Dynamic Relocation

base and bounds/dynamic resolution

we need two hardware registers - base register and bounds/limit register

when os decides where in physical memory program is located, sets base register to that value

now any memory reference is translated by the processor

physical address = virtual address + base

each reference generated by process is a virtual address

the hardware adds base register to this address and get physical address

**address translation: ** transforming a virtual address into physical address

this relocation of address happens at runtime, and we can move address spaces even after process has started running, referred as **dynamic resolution**

the processor will first check if memory reference is within bounds to make sure it is legal

the base and bounds are hardware structures kept on the chip 

this part of processor is **memory management unit(MMU)**

in one way the bounds register holds the size of address space; another way it holds the *physical address* of the end of memory space

### Hardware Support	

the os runs in kernel mode, access to entire machine

applications run in user mode, limited

a single bit, stored in some kind of **processor status word**, indicates which mode the CPU is running in

the hardware also provides the base and bounds registers, part of the MMU of cpu

the hardware should provide special instructions to modify the base and bounds registers, allow os to change them when processes run; this is privileged

the cpu must be able to generate exception when program tries to access memory illegally

the os out-of-bounds handler runs, decides how to handle, usually terminate the process

when os decides to stop running process, it must save values of   base and bounds registers to memory, in some per-process structure such as **process structure** or **process control block**

similarly, when os resumes a process, it must set values of base and bounds to correct values

if address space moves location, update the base register

os also must provide exception handlers



## Segmentation

with base and bounds, there is big chunk of "free" space in the middle of stack and heap

instead of having one base and bounds pair in our MMU, have a  base and bound pair per logical segment of the address space

three base and bounds register pairs for code, heap and stack

**external fragmentation** when physical memory has lots of holes of free memory

## Free Space Management

the generic data structure used to manage free space in the heap is some kind of **free list** 

this structure contains references to all free chunks of space in the managed region of memory

## Paging

chop up space into fixed-sized pieces - **paging**

instead of splitting up a process's address space into some number of variable-sized logical segments called **page**

we view physical memory as array of fixed-sized slots called **page frames**

each frame contains a single virtual-memory page

example, when os wishes to place our tiny 64byte address into eight-page memory, it finds four free pages; perhaps it keeps a free list of all free pages for this and grabs the first free pages off the list

to record where each virtual page of the address space is placed in memory, os keeps a per-process data structure known as a **page table**

the major role of page table is to store **address translations** for each of virtual pages of address space



### where are page tables stored?

page tables can get large

we store page tables in memory



## Paging: Faster Translations

going to memory for every instruction is slow

we can use **translation-lookaside buffer**

it is a part of the chip's MMU; simply a hardware cache of popular virtual-to-physical address translation. also called an **address-translation cache**

for each virtual memory reference, first check TLB

## Paging: Smaller Tables

problem: page tables are too big and consume too much memory

we usually have a page table for every process in the system

simpler solution: bigger pages

Problem will be big wastes within each page; internal fragmentation

hybrid approach: paging and segments

we use base/bounds to hold the physical address of the page table/end of the page table

problem here is it causes external fragmentation

### Multi-level Page Tables

first chop up the page table into page-sized chunks;

to track validity of page of a page table; use **page directory**



### Swapping the page tables to disk

if it gets too big for kernel virtual memory it goes into swap



## Swapping Mechanisms

we require additional level in memory hierarchy

### Swap Space

the first thing we need is reserve some space on disk for moving pages back and forth - **swap space**

the os will need to remember the **disk address** of a given page

The page fault - if page is not present page-fault handler; it is implemented in the hardware

if page is not present and has been swapped to disk, os will need to swap the page into memory inorder to service the page fault



## The Linux Virtual Memory System

### The Linux Address Space





# Concurrency

a new abstraction for a single running program: **thread**

a **multi-threaded ** program has more than one point of execution; each thread is like a separate process *except* they share the same address space

state of single thread has a program counter(PC); has its own private set of registers it uses; switching from one thread to another will need a **context switch**

the context switch is similar as between processes

with processes, we need process control block(PCB); now with threads we need thread control blocks(TCBs) to store the state of each thread of a process

major difference is that the address space remains the same

also with multi-threaded process, each thread runs independently and instead of single stack we have one stack per thread

