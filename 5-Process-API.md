

- Each process has a name; in most systems, that name is a number
known as a process ID (PID).
- The fork() system call is used in UNIX systems to create a new process. The creator is called the parent; the newly created process is
called the child. fork() creates a new process by duplicating the calling process. The new process, referred to as the child, is an exact duplicate of the calling process, 
except for a few exceptions. 
- The wait() system call allows a parent to wait for its child to complete execution.
- The exec() family of system calls allows a child to break free from
its similarity to its parent and execute an entirely new program.
- A UNIX shell commonly uses fork(), wait(), and exec() to
launch user commands; the separation of fork and exec enables features like input/output redirection, pipes, and other cool features,
all without changing anything about the programs being run.
- Process control is available in the form of signals, which can cause
jobs to stop, continue, or even terminate.
- Which processes can be controlled by a particular person is encapsulated in the notion of a user; the operating system allows multiple
users onto the system, and ensures users can only control their own
processes.
- A superuser can control all processes (and indeed do many other
things); this role should be assumed infrequently and with caution
for security reasons.
