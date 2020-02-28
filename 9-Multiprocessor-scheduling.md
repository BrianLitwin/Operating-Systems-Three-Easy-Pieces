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


