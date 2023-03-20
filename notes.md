# Operating System notes

Operating system notes based on NYU’s graduate level OS course.

These notes are not complete and only exist to refresh my memory about certain things that I tend to forget. For full material, check the course slides.


# OS Basics



* Bios starts
    * Checks RAM
    * Determines the boot device
    * Finds the active partition
    * Secondary boot loader is then loaded from the active partition
        * Loads the OS
* Operating systems differ based on what they will be used for.
    * Home PCs
    * Mainframes
    * Server side
    * Real time OS
    * Shared OS
    * iOS / Android
    * etc.
* CPUs contain
    * General registers
        * Hold the key variables and temp results.
    * Special registers
        * Program counter - Contains the memory address of the next instruction
        * Stack pointer - points to the top of the current stack
        * PSW (Program Status Word) contains the condition code bits which are set by comparison instructions, the CPU priority, the mode (user or kernel) and various other control bits.
* Caches
    * Cache invalidation is a hard problem to solve
        * When there are multiple CPUs each having their own caches, when some CPU writes the value, others must be either invalidated or get the ownership
        * There are 2 modes, shared and exclusive.
        * Check scenarios in slides.
* Process
    * Processes contain
        * Address space
        * Process table entries (state, registers, open files, resources held, threads and their states.)
    * OS maintains a process tree.
    * Processes do address virtualization
    * Addresses in OS are protected. (Read only / write only / executable)
    * Kernel has its own address space as well.
* Kernel mode v/s user mode
    * How to switch?
        * Traps - system calls - Synchronous
        * Interrupts - Triggered by hardware event / device - Asynchronous
        * Exception - Fault conditions - Synchronous
        * RFI (Return from interrupt) Kernel to user
    * Special hardware register contains address for the code related to the kernel entry called __entry.
    * __entry is written in assembly language.
    * IS the only way to enter the kernel.
    * Anything that has to do with more resources or something similar to that requires us to enter the kernel mode. Like spawning new thread (fork()) exit() etc.
* Syscall implementation
    * Params passed from registers, following ABI rules
    * Kernel has  a syscall table, that maps the sys call to the code to be executed.
    * Stack change from user stack to kernel stack happens.
* Fun fact : Windows has no process hierarchy.
* Services that can be provided at user level(because they only read unprotected data) \
– Read time of the day
    * Services that need to be provided at kernel level \
– System calls: file open, close, read and write \
– Control the CPU so that users won’t stuck by running while ( 1 ) ; \
– Protection:
        * Keep user programs from crashing OS
        * Keep user programs from crashing each other
* OS expects executable files to have info like code locations / size and data locations and size.
    * It also requires a symbol table.
        * Things defined and where they are in memory
        * Things not defined but used and where they are used.
        * Symbol can be name of function / variable etc.
* Linking resolved cross - files references while linking.
* Loader loads the binary from hard disk and makes a process out of it.. Also figures out stuff that needs to be done if we are using dynamically linked libraries.
* **Question** : When is extern keyword needed really? During linking?


# Concurrency and Deadlocks



* The idea of critical region where only one thread can execute at a time
* Think of possible erroneous cases where there are read-modify-write type code.
* Mutual exclusion can be made possible trivially using busy waiting.
    * But it is bad.
* Software solutions:
    * Peterson’s solution. Uses busy waiting
* Hardware solutions:
    * TSL instruction (Test and Set Lock)
    * XCHG instruction - Logically same as TSL. AKA cmp and swp instruction at some places
* Reservation system on the cache - How XCHG are implemented using cache coherency!!
    * NOT atomic in a strict sense, but is a good way to re-use what we studied in cache coherency, the dynamics of shared and exclusive owning of cachelines.
* Priority Inversion problem:
    * Low priority holds the lock. High priority busy waits and loops to get the lock.
        * Low priority is not scheduled. Hence will not release the lock.
        * Everyone loses.
    * Solution: Temporarily increase the priority of the low prio process so that it is able to release the lock. This is a complex solution as this involves the scheduler as well.
* Lock contention
    * If LC is low then TSL is a good enough solution. Linux uses TSL at many places.
    * IF LC high we need something like semaphores
* Semaphores
    * P : Wait / Down / Acquiring the semaphore
    * V: Signal / Up / Releasing the semaphore
    * P and V functions must be atomic!
        * We do that using locks!
            * These locks are busy wait locks, but since the lock time for P and V is small, it is okay.
    * At a high level, Semaphores still use the busy lock technique, but they involve the scheduler into the locking mechanism and therefore are able to reduce the lock time to be very less and hence are efficient. This is why it does not makes sense to use Semaphores for CS which are already pretty small, just simply using TSL / XCHG / load-store technique is better.
    * Two types:
        * Mutex semaphores
        * Counting semaphores
    * Pthread library gives the functionality of mutexes
        * Pthread_mutex_init – Simple mutex
        * Pthread_cond_init – Analogous of Semaphores. Linux does not contain Semaphores / doesnt calls it semaphores.
    * Mutex_trylock – fails if we cannot achieve. Can do something else!
    * Pthreads use a weird combination of condition variables and mutex variables.
        * Waiting on both the mutex as well as the condition.
            * As long as you don’t have the condition, release the mutex. As soon as you get the condition, re-acquire the mutex.
                * See the code example in the slides
    * Problems with semaphores:
        * If a thread dies, the token is lost.
        * Scheduling is expensive.
        * Its complex to use them correctly.
            * Chances of deadlocks
    * In order to prevent deadlocks, locks must be acquired in the same order always.
        * SMP scheduler example. How load balancing is done by imposing a total ordering over the locks and always acquiring the locks in the order. We may have to let go of some lock(s) that we have already acquired in order to ensure ordering of acquisition. See code example in slides.
* Other unix / linux locking mechanisms:
    * flock() – for files
    * System V semaphores – heavy weight semaphores - the ones that we know.
    * futex() – Lightweight locks. Uncontested cases are identified in userspace and if no one is contesting, do an atomic cmpxchg and acquire the lock. Otherwise if race condition is detected, go into the kernel space.
        * Very useful in databases - Where there is mostly no contention, but we need this for correctness.
    * mq_open() etc…. – message queues lockings
* Check how Dining Philosophers problem can be solved using semaphores
    * Basically we try to eat, set our state to hungry and check if both forks are available.
        * If they are, we down the sema and eat. If not we down it and wait.
    * When we are done eating we set our own state to thinking and test left and right if they want to eat.
    * Total semas used : s[N] one for each philosopher and s_mutex for checking the states.
* Multiple readers and writers problem
    * Many readers can read at the same time.
        * Only one writer can write and if there is one writer, there can be no other writer or reader on the database.
    * Simple application of semaphores.
    * One sema for ReadCount var
    * One sema for the database.
* Avoiding locks : Read Copy Update dynamic
    * Much more complex.
* Monitors: Higher level sync primitive – Supported by all popular languages
    * Only one thread can be in a monitor
    * Compiler achieves this mutual exclusion
    * Monitor looks and behaves like a class.
    * Monitor is a programming language construct like a class or a for-loop.
    * **Question** : What happens when A signals B but has some code left to run? Alternatively, what does it mean that a single thread runs in the monitor? Across all procedures, is there only a single thread at a time in the monitor?
    * **Question** : Monitors are supposed to be inefficient?
    * **Question** : Monitors do not require a counting semaphore type of lock? Because anyways there can only be one thread active in the monitor at a time?
