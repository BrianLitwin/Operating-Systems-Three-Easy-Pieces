The two main challenges in time sharing, and thus virtualizing, the CPU are performance and control. 
The performance challenge is implementing time sharing without adding excessive overhead to the system. 
The control challenge is maintaining sufficient control over the CPU. 

**Limited Direct Execution** 

To make a program run as fast as one might expect, not surprisingly
OS developers came up with a technique, which we call limited direct
execution. The “direct execution” part of the idea is simple: just run the
program directly on the CPU. Thus, when the OS wishes to start a program running, 
it creates a process entry for it in a process list, allocates
some memory for it, loads the program code into memory (from disk), 
locates its entry point (i.e., the main() routine or something similar), jumps
to it, and starts running the user’s code. It looks like this: 

OS 
Create entry for process list
Allocate memory for program
Load program into memory (from disk) 
Set up stack with argc/argv
Clear registers
Execute call main()

Program 
Run main()
Execute return from main

OS 
Free memory of process
Remove from process list

Direct execution has the advantage of being fast, but there still needs to be a notion of control: 
what if the process wishes to perform some kind of restricted operation, such
as issuing an I/O request to a disk, or gaining access to more system
resources such as CPU or memory?

Code that runs in user mode is restricted in what it
can do. For example, when running in user mode, a process can’t issue
I/O requests; doing so would result in the processor raising an exception;
the OS would then likely kill the process.

In contrast to user mode is kernel mode, which the operating system
(or kernel) runs in. In this mode, code that runs can do what it likes, 
including privileged operations such as issuing I/O requests and executing
all types of restricted instructions.

What should a user process do when it wishes to perform some kind of privileged operation,
such as reading from disk? To enable this, virtually all modern hardware provides the ability 
for user programs to perform a system call. system calls
allow the kernel to carefully expose certain key pieces of functionality to
user programs, such as accessing the file system, creating and destroying processes, communicating 
with other processes, and allocating more memory. 

To execute a system call, a program must execute a special trap instruction. This instruction 
simultaneously jumps into the kernel and raises the privilege level to kernel mode; once in the 
kernel, the system can now perform whatever privileged operations are needed (if allowed), and thus do
the required work for the calling process. When finished, the OS calls a
special return-from-trap instruction, which, as you might expect, returns
into the calling user program while simultaneously reducing the privilege level back to user mode.

The hardware needs to be a bit careful when executing a trap, in that it
must make sure to save enough of the caller’s registers in order to be able
to return correctly when the OS issues the return-from-trap instruction.
On x86, for example, the processor will push the program counter, flags,
and a few other registers onto a per-process kernel stack; the return-fromtrap 
will pop these values off the stack and resume execution of the usermode program 
(see the Intel systems manuals [I11] for details). Other
hardware systems use different conventions, but the basic concepts are
similar across platforms.

There is one important detail left out of this discussion: how does the
trap know which code to run inside the OS? Clearly, the calling process
can’t specify an address to jump to (as you would when making a procedure call)
; doing so would allow programs to jump anywhere into the
kernel which clearly is a Very Bad Idea. Thus the kernel must carefully
control what code executes upon a trap. The kernel does so by setting up 
a trap table at boot time.

The kernel does so by setting up a trap table at boot time. When the
machine boots up, it does so in privileged (kernel) mode, and thus is free
to configure machine hardware as need be. One of the first things the OS
thus does is to tell the hardware what code to run when certain exceptional events occur.
For example, what code should run when a harddisk interrupt takes place, when a keyboard 
interrupt occurs, or when a program makes a system call? The OS informs the hardware of the
locations of these trap handlers, usually with some kind of special instruction. 
Once the hardware is informed, it remembers the location of
these handlers until the machine is next rebooted, and thus the hardware
knows what to do (i.e., what code to jump to) when system calls and other
exceptional events take place. 

To specify the exact system call, a system-call number is usually assigned to each system call. 
The user code is thus responsible for placing the desired system-call number in 
a register or at a specified location on the stack; the OS, when handling the system 
call inside the trap handler, examines this number, ensures it is valid, and, if it is, 
executes the corresponding code. This level of indirection serves as a form of protection;
user code cannot specify an exact address to jump to, but rather must
request a particular service via number.

There are two phases in the limited direct execution (LDE) protocol.
In the first (at boot time), the kernel initializes the trap table, and the
CPU remembers its location for subsequent use. The kernel does so via a
privileged instruction.

In the second (when running a process), the kernel sets up a few things
(e.g., allocating a node on the process list, allocating memory) before using a return-from-trap instruction to start the execution of the process;
this switches the CPU to user mode and begins running the process.
When the process wishes to issue a system call, it traps back into the OS,
which handles it and once again returns control via a return-from-trap
to the process. The process then completes its work, and returns from
main(); this usually will return into some stub code which will properly
exit the program (say, by calling the exit() system call, which traps into
the OS). At this point, the OS cleans up and we are done. 

**Switching between processes**

Switching between processes should be simple, right? The
OS should just decide to stop one process and start another. What’s the
big deal? But it actually is a little bit tricky: specifically, if a process is
running on the CPU, this by definition means the OS is not running. If
the OS is not running, how can it do anything at all? (hint: it can’t) While
this sounds almost philosophical, it is a real problem: there is clearly no
way for the OS to take an action if it is not running on the CPU. Thus we
arrive at the crux of the problem.

The answer turns out to be simple and was discovered by a number
of people building computer systems many years ago: a timer interrupt
[M+63]. A timer device can be programmed to raise an interrupt every
so many milliseconds; when the interrupt is raised, the currently running
process is halted, and a pre-configured interrupt handler in the OS runs.
At this point, the OS has regained control of the CPU, and thus can do
what it pleases: stop the current process, and start a different one

As we discussed before with system calls, the OS must inform the
hardware of which code to run when the timer interrupt occurs; thus,
at boot time, the OS does exactly that. Second, also during the boot
sequence, the OS must start the timer, which is of course a privileged
operation. Once the timer has begun, the OS can thus feel safe in that
control will eventually be returned to it, and thus the OS is free to run
user programs. The timer can also be turned off (also a privileged operation), 
something we will discuss later when we understand concurrency in more detail.

Note that the hardware has some responsibility when an interrupt occurs, 
in particular to save enough of the state of the program that was
running when the interrupt occurred such that a subsequent return-from
trap instruction will be able to resume the running program correctly.
This set of actions is quite similar to the behavior of the hardware during
an explicit system-call trap into the kernel, with various registers thus
getting saved (e.g., onto a kernel stack) and thus easily restored by the
return-from-trap instruction.

**Saving and Restoring Context**
Now that the OS has regained control, whether cooperatively via a system call, 
or more forcefully via a timer interrupt, a decision has to be
made: whether to continue running the currently-running process, or
switch to a different one. This decision is made by a part of the operating
system known as the scheduler.

If the decision is made to switch, the OS then executes a low-level
piece of code which we refer to as a context switch. A context switch is
conceptually simple: all the OS has to do is save a few register values
for the currently-executing process (onto its kernel stack, for example)
and restore a few for the soon-to-be-executing process (from its kernel
stack). By doing so, the OS thus ensures that when the return-from-trap
instruction is finally executed, instead of returning to the process that was
running, the system resumes execution of another process.

To save the context of the currently-running process, the OS will execute some 
low-level assembly code to save the general purpose registers, PC, and the kernel 
stack pointer of the currently-running process, and then 
restore said registers, PC, and switch to the kernel stack for the
soon-to-be-executing process. By switching stacks, the kernel enters the
call to the switch code in the context of one process (the one that was 
interrupted) and returns in the context of another (the soon-to-be-executing
one). When the OS then finally executes a return-from-trap instruction,
the soon-to-be-executing process becomes the currently-running process.
And thus the context switch is complete.

A timeline of the entire process is shown in Figure 6.3. In this example,
Process A is running and then is interrupted by the timer interrupt. The
hardware saves its registers (onto its kernel stack) and enters the kernel
(switching to kernel mode). In the timer interrupt handler, the OS decides
to switch from running Process A to Process B. At that point, it calls the
switch() routine, which carefully saves current register values (into the
process structure of A), restores the registers of Process B (from its process
structure entry), and then switches contexts, specifically by changing the
stack pointer to use B’s kernel stack (and not A’s). Finally, the OS returnsfrom-trap, 
which restores B’s registers and starts running it.

Note that there are two types of register saves/restores that happen
during this protocol. The first is when the timer interrupt occurs; in this
case, the user registers of the running process are implicitly saved by the
hardware, using the kernel stack of that process. The second is when the
OS decides to switch from A to B; in this case, the kernel registers are explicitly saved by the software (i.e., the OS), but this time into memory in
the process structure of the process. The latter action moves the system
from running as if it just trapped into the kernel from A to as if it just
trapped into the kernel from B.

