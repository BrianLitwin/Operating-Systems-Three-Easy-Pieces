Problem: Although there are only a few physical CPUs available, how can the
OS provide the illusion of a nearly-endless supply of said CPUs?

The OS creates this illusion by virtualizing the CPU. By running one
process, then stopping it and running another, and so forth, the OS can
promote the illusion that many virtual CPUs exist when in fact there is
only one physical CPU (or a few). This basic technique, known as time
sharing of the CPU, allows users to run as many concurrent processes as
they would like; the potential cost is performance, as each will run more
slowly if the CPU(s) must be shared.

To implement virtualization of the CPU, and to implement it well, the
OS will need both some low-level machinery and some high-level intelligence. 
We call the low-level machinery mechanisms; mechanisms
are low-level methods or protocols that implement a needed piece of
functionality. For example, we’ll learn later how to implement a context
switch, which gives the OS the ability to stop running one program and
start running another on a given CPU; this time-sharing mechanism is
employed by all modern OSes.
On top of these mechanisms resides some of the intelligence in the
OS, in the form of policies. Policies are algorithms for making some
kind of decision within the OS. For example, given a number of possible 
programs to run on a CPU, which program should the OS run? A
scheduling policy in the OS will make this decision, likely using 
historical information (e.g., which program has run more over the last minute?),
workload knowledge (e.g., what types of programs are run), and performance metrics 
(e.g., is the system optimizing for interactive performance,
or throughput?) to make its decision.

Time sharing is a basic technique used by an OS to share a resource. By
allowing the resource to be used for a little while by one entity, and then
a little while by another, and so forth, the resource in question (e.g., the
CPU, or a network link) can be shared by many. The counterpart of time
sharing is space sharing, where a resource is divided (in space) among
those who wish to use it. For example, disk space is naturally a spaceshared resource; 
once a block is assigned to a file, it is normally not assigned to another file until 
the user deletes the original file.

The abstraction provided by the OS of a running program is something
we will call a process. As we said above, a process is simply a running
program; at any instant in time, we can summarize a process by taking an
inventory of the different pieces of the system it accesses or affects during
the course of its execution.
To understand what constitutes a process, we thus have to understand
its machine state: what a program can read or update when it is running.
At any given time, what parts of the machine are important to the execution 
of this program?
One obvious component of machine state that comprises a process is
its memory. Instructions lie in memory; the data that the running program 
reads and writes sits in memory as well. Thus the memory that the
process can address (called its address space) is part of the process.
Also part of the process’s machine state are registers; many instructions
explicitly read or update registers and thus clearly they are important to
the execution of the process.
Note that there are some particularly special registers that form part
of this machine state. For example, the program counter (PC) (sometimes
called the instruction pointer or IP) tells us which instruction of the 
program will execute next; similarly a stack pointer and associated frame
pointer are used to manage the stack for function parameters, local variables, 
and return addresses.
Finally, programs often access persistent storage devices too. Such I/O
information might include a list of the files the process currently has open.

In many operating systems, a common design paradigm is to separate
high-level policies from their low-level mechanisms [L+75]. You can
think of the mechanism as providing the answer to a how question about
a system; for example, how does an operating system perform a context
switch? The policy provides the answer to a which question; for example,
which process should the operating system run right now? Separating the
two allows one easily to change policies without having to rethink the
mechanism and is thus a form of modularity, a general software design
principle.

These APIs, in some form, are available on any modern operating system.
• Create: An operating system must include some method to create new processes. 
When you type a command into the shell, or
double-click on an application icon, the OS is invoked to create a
new process to run the program you have indicated.
• Destroy: As there is an interface for process creation, systems also
provide an interface to destroy processes forcefully. Of course, many
processes will run and just exit by themselves when complete; when
they don’t, however, the user may wish to kill them, and thus an 
interface to halt a runaway process is quite useful.
• Wait: Sometimes it is useful to wait for a process to stop running;
thus some kind of waiting interface is often provided.
• Miscellaneous Control: Other than killing or waiting for a process,
there are sometimes other controls that are possible. For example,
most operating systems provide some kind of method to suspend a
process (stop it from running for a while) and then resume it (continue it running).
• Status: There are usually interfaces to get some status information
about a process as well, such as how long it has run for, or what
state it is in.

**Process Creation 

The first thing that the OS must do to run a program is to load its code
and any static data (e.g., initialized variables) into memory, into the address space of the process. 
Programs initially reside on disk (or, in some
modern systems, flash-based SSDs) in some kind of executable format;
thus, the process of loading a program and static data into memory requires the OS to read those bytes from disk 
and place them in memory. 

