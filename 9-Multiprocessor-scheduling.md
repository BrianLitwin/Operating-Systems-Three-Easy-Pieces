How should the OS schedule jobs on multiple CPUs? 
What new problems arise? 
Do the same techniques work as on a single CPU, or are new techniques required? 

To understand the new issues surrounding multiprocessor scheduling, we have to understand 
a new and fundamental difference between single-CPU hardware and multi-CPU hardware. 
This difference centers around the use of hardware caches (e.g., Figure 10.1), and exactly how
data is shared across multiple processors. In a system with a single CPU, there are a hierarchy of hardware
caches that in general help the processor run programs faster. Caches
are small, fast memories that (in general) hold copies of popular data that
is found in the main memory of the system. Main memory, in contrast,
holds all of the data, but access to this larger memory is slower. By keeping frequently accessed data in a cache, the system can make the large,
slow memory appear to be a fast one.

As an example, consider a program that issues an explicit load instruction to fetch a value from memory, 
and a simple system with only a single CPU; the CPU has a small cache (say 64 KB) and a large main memory.
The first time a program issues this load, the data resides in main memory, 
and thus takes a long time to fetch (perhaps in the tens of nanoseconds, or even hundreds). 
The processor, anticipating that the data may be reused, puts a copy of the loaded data into the CPU cache. 
If the program later fetches this same data item again, the CPU first checks for it in the
cache; if it finds it there, the data is fetched much more quickly (say, just
a few nanoseconds), and thus the program runs faster. 

Caches are thus based on the notion of locality, of which there are
two kinds: temporal locality and spatial locality. The idea behind temporal 
locality is that when a piece of data is accessed, it is likely to be
accessed again in the near future; imagine variables or even instructions
themselves being accessed over and over again in a loop. The idea behind spatial locality 
is that if a program accesses a data item at address
x, it is likely to access data items near x as well; here, think of a program
streaming through an array, or instructions being executed one after the
other. Because locality of these types exist in many programs, hardware
systems can make good guesses about which data to put in a cache and
thus work well.

Now for the tricky part: what happens when you have multiple processors in a single system, with a single shared main memory?

As it turns out, caching with multiple CPUs is much more complicated. Imagine, for example, that a program running on CPU 1 reads
a data item (with value D) at address A; because the data is not in the
cache on CPU 1, the system fetches it from main memory, and gets the
value D. The program then modifies the value at address A, just updating its cache with the new value D; writing the data through all the way
to main memory is slow, so the system will (usually) do that later. Then
assume the OS decides to stop running the program and move it to CPU
2. The program then re-reads the value at address A; there is no such data
CPU 2’s cache, and thus the system fetches the value from main memory,
and gets the old value D instead of the correct value D.

This general problem is called the problem of cache coherence, and
there is a vast research literature that describes many different subtleties
involved with solving the problem

The basic solution is provided by the hardware: by monitoring memory accesses, 
hardware can ensure that basically the “right thing” happens and that the view 
of a single shared memory is preserved. One way
to do this on a bus-based system (as described above) is to use an old
technique known as bus snooping [G83]; each cache pays attention to
memory updates by observing the bus that connects them to main memory. When a CPU then sees an update for a data item it holds in its cache,
it will notice the change and either invalidate its copy (i.e., remove it
from its own cache) or update it (i.e., put the new value into its cache
too). Write-back caches, as hinted at above, make this more complicated
(because the write to main memory isn’t visible until later), but you can
imagine how the basic scheme might work.

Given that the caches do all of this work to provide coherence, do programs (or the OS itself) 
have to worry about anything when they access shared data? The answer, unfortunately, is yes. 

When accessing (and in particular, updating) shared data items or
structures across CPUs, mutual exclusion primitives (such as locks) should
likely be used to guarantee correctness (other approaches, such as building lock-free data structures, 
are complex and only used on occasion; see the chapter on deadlock in the piece on concurrency for details). 
For example, assume we have a shared queue being accessed on multiple CPUs concurrently. Without locks, 
adding or removing elements from the queue concurrently will not work as expected, even with the underlying 
coherence protocols; one needs locks to atomically update the data structure to its new state.

The solution, of course, is to make such routines correct via locking. In this case, allocating a simple mutex (e.g., pthread mutex tm;) and then adding a lock(&m) at the beginning of the routine and
an unlock(&m) at the end will solve the problem, ensuring that the code
will execute as desired. Unfortunately, as we will see, such an approach is
not without problems, in particular with regards to performance. Specifically, as the number of CPUs grows, access to a synchronized shared data structure becomes quite slow. 

One final issue arises in building a multiprocessor cache scheduler,
known as cache affinity [TTG95]. This notion is simple: a process, when
run on a particular CPU, builds up a fair bit of state in the caches (and
TLBs) of the CPU. The next time the process runs, it is often advantageous to 
run it on the same CPU, as it will run faster if some of its state
is already present in the caches on that CPU. If, instead, one runs a 
process on a different CPU each time, the performance of the process will be
worse, as it will have to reload the state each time it runs (note it will run
correctly on a different CPU thanks to the cache coherence protocols of
the hardware). Thus, a multiprocessor scheduler should consider cache
affinity when making its scheduling decisions, perhaps preferring to keep
a process on the same CPU if at all possible.

The most basic approach is to simply reuse the basic framework for single 
processor scheduling, by putting all jobs that need to be scheduled into 
a single queue; we call this singlequeue multiprocessor scheduling or SQMS for short. This approach
has the advantage of simplicity; it does not require much work to take an
existing policy that picks the best job to run next and adapt it to work on
more than one CPU (where it might pick the best two jobs to run, if there
are two CPUs, for example). However, SQMS has obvious shortcomings. 
The first problem is a lack of scalability. To ensure the scheduler works 
correctly on multiple CPUs, the developers will have inserted some form of 
locking into the code, as described above. Locks ensure that when SQMS 
code accesses the single queue (say, to find the next job to run), the proper outcome arises.
Locks, unfortunately, can greatly reduce performance, particularly as
the number of CPUs in the systems grows [A91]. As contention for such
a single lock increases, the system spends more and more time in lock
overhead and less time doing the work the system should be doing (note:
it would be great to include a real measurement of this in here someday).

The second main problem with SQMS is cache affinity. 

Because of the problems caused in single-queue schedulers, some systems opt for multiple queues, e.g., one per CPU. 
We call this approach multi-queue multiprocessor scheduling (or MQMS).

In MQMS, our basic scheduling framework consists of multiple scheduling queues. 
Each queue will likely follow a particular scheduling discipline, such as round robin, though of course any algorithm can be used. When a job enters the system, it is placed on exactly one scheduling
queue, according to some heuristic (e.g., random, or picking one with
fewer jobs than others). Then it is scheduled essentially independently,
thus avoiding the problems of information sharing and synchronization
found in the single-queue approach.

MQMS has a distinct advantage of SQMS in that it should be inherently more scalable. As the number of CPUs grows, so too does the number of queues, and thus lock and cache contention should not become a
central problem. In addition, MQMS intrinsically provides cache affinity;
jobs stay on the same CPU and thus reap the advantage of reusing cached
contents therein.

But, if you’ve been paying attention, you might see that we have a new
problem, which is fundamental in the multi-queue based approach: load
imbalance.

How should a multi-queue multiprocessor scheduler handle load imbalance, 
so as to better achieve its desired scheduling goals?

The obvious answer to this query is to move jobs around, a technique
which we (once again) refer to as migration. By migrating a job from one
CPU to another, true load balance can be achieved. If there are two jobs on
one cpu and none on the other, you can  move one to un-used cpu. If there are
two jobs on a cpu and one on the other cpu, you can perform continuous migration 
ie switching which cpu has the single job. Of course, many other possible migration patterns exist. 

But now for the tricky part: how should the system decide to enact such a migration?

One basic approach is to use a technique known as work stealing
[FLR98]. With a work-stealing approach, a (source) queue that is low
on jobs will occasionally peek at another (target) queue, to see how full
it is. If the target queue is (notably) more full than the source queue, the
source will “steal” one or more jobs from the target to help balance load.

Of course, there is a natural tension in such an approach. If you look
around at other queues too often, you will suffer from high overhead
and have trouble scaling, which was the entire purpose of implementing
the multiple queue scheduling in the first place! If, on the other hand,
you don’t look at other queues very often, you are in danger of suffering
from severe load imbalances. Finding the right threshold remains, as is
common in system policy design, a black art.

 Linux Multiprocessor Schedulers
 
Interestingly, in the Linux community, no common solution has approached to building a multiprocessor scheduler. Over time, three different schedulers arose: the O(1) scheduler, the Completely Fair Scheduler (CFS), and the BF Scheduler (BFS)2
. See Meehean’s dissertation for an excellent overview of the strengths and weaknesses of said schedulers
[M11]; here we just summarize a few of the basics.

Both O(1) and CFS use multiple queues, whereas BFS uses a single
queue, showing that both approaches can be successful. Of course, there
are many other details which separate these schedulers. For example, the
O(1) scheduler is a priority-based scheduler (similar to the MLFQ discussed before), 
changing a process’s priority over time and then scheduling those with highest priority 
in order to meet various scheduling objectives; interactivity is a particular focus. 
CFS, in contrast, is a deterministic
proportional-share approach (more like Stride scheduling, as discussed
earlier). BFS, the only single-queue approach among the three, is also
proportional-share, but based on a more complicated scheme known as
Earliest Eligible Virtual Deadline First (EEVDF) [SA96]. 
