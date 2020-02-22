
The fundamental problem MLFQ tries to address is two-fold. First, it
would like to optimize turnaround time, which, as we saw in the previous
note, is done by running shorter jobs first; unfortunately, the OS doesn’t
generally know how long a job will run for, exactly the knowledge that
algorithms like SJF (or STCF) require. Second, MLFQ would like to make
a system feel responsive to interactive users (i.e., users sitting and staring
at the screen, waiting for a process to finish), and thus minimize response
time; unfortunately, algorithms like Round Robin reduce response time
but are terrible for turnaround time. Thus, our problem: given that we
in general do not know anything about a process, how can we build a
scheduler to achieve these goals? How can the scheduler learn, as the
system runs, the characteristics of the jobs it is running, and thus make
better scheduling decisions?

Rule 1: If Priority(A) > Priority(B), A runs (B doesn’t).
• Rule 2: If Priority(A) = Priority(B), A & B run in round-robin fashion using the time slice (quantum length) of the given queue.
• Rule 3: When a job enters the system, it is placed at the highest
priority (the topmost queue).
• Rule 4: Once a job uses up its time allotment at a given level (regardless of how many times it has given up the CPU), its priority is
reduced (i.e., it moves down one queue).
• Rule 5: After some time period S, move all the jobs in the system
to the topmost queue.
