---
layout: post
title:  "Reading notes: Java Concurrency in Practice"
date:   2013-07-06 21:17:01
tags: concurrency
---

## Summary of Part I

* It's the mutable state, stupid.

* Make fileds final unless they need to be mutable.

* Immutable objects are automatically thread-safe.

* Encapsulation makes it practical to manage the complexity.

* Guard each mutable variable with a lock.

* Guard all variables in an invariant with the same lock.

* Hold locks for the duration of compound actions.

* A program that accesses a mutable variable from multiple threads without synchronization is a broken program.

* Don't rely on clever reasoning about why you don't need to synchronize.

* Include thread safety in the design process-or explicitly document that your class is not thread-safe.

* Document your synchronization policy.


## Summary of Part II

* Structuring applications around the execution of tasks can simplify development and facilitate concurrency. The Executor framework permits you to decouple task submission from execution policy and supports a rich variety of execution policies; whenever you find yourself creating threads to perform tasks, consider using an Executor instead. To maximize the benefit of decomposing an application into tasks, you must identify sensible task boundaries. In some applications, the obvious task boundaries work well, whereas in others some analysis may be required to uncover fine-grained exploitable parallelism.

* End-of-lifecycle issues for tasks, threads, services, and applications can add complexity to their design and implementation. Java does not provide a preemptive mechanism for cancelling activities or terminating threads. Insteds, it provides a cooperative interruption mechanism that can be used to facilitate cancellation, but it is up you to construct protocols for cancellation and use them consistently. Using FutureTask and the Executor framework simplifies building cancellable tasks and services.

* The Executor framework is a powerful and flexible framework for concurrently executing tasks. It offers a number of tuning options, such as policies for creating and tearing down threads, handling queued tasks, and what to do with execuss tasks, and provides several hooks for extending its behavior. As in most powerful frameworks, however, there are combinations of settings that do not work well together; some types of tasks require specific execution policies, and some combinations of tuning parameters may produce strange result.

* GUI frameworks are nearly always implemented as single-threaded subsystems in which all presentation-related code runs as tasks in an event thread. Because there is only a single event thread, long-running tasks can compromise responsiveness and so should be executed in background threads. Helper classes like SwingWorker or the BackgroundTask class built here, which provide support for cancellation, progress indication, and completion indication, can simplify the development of long-running tasks that hava both GUI and non-GUI components.

## Summary of Part III

* Liveness failures are a serious problem because there is no way to recover from them short of aborting the application. The most common form of liveness failure is lock-ordering deadlock. Avoiding lock ordering deadlock starts at design time: ensure that when threads acquire multiple locks, they do so in a consistent order. The best way to do this is by using open calls throughout your program. This greatly reduces the number of places where multiple locks are held at once, and makes it more obvious where those places are.

* Because one of the most common reasons to use threads is to exploit multiple processors, in discussing the performance of concurrent applications, we are usually more concerned with throughput or scalability than we are with raw service time. Amdahl's law tells us that the scalability of an application is driven by the proportion of code that must be executed serially. Since the primary source of serialization in Java programs is the exclusive resource lock, scalability can often be improved by spending less time holding locks, either by reducing lock granularity, reducing the duration for which locks are held, or replacing exclusive locks with nonexclusive or nonblocking alternatives.

* Testing concurrent programs for correctness can be extremely challenging because many of the possible failure modes of concurrent programs are low-probability events that are sensitive to timing, load, and other hard-to-reproduce conditions. Further, the testing infrastructure can introduce additional synchronization or timing constraints that can mask concurrency problems in the code being tested. Testing concurrent programs for performance can be equally challenging; Java programs are more difficult to test than programs written in statically compiled languages like C, because timing measurements can be affected by dynamic compilation, garbage collection, and adaptive optimization. To have the best chance of finding latent bugs before they occur in production, combine traditional testing techniques(being careful to avoid the pitfalls discussed here) with code reviews and automated analysis tools. Each of these techniques finds problems that the others are likely to miss.

## Summary of Part IV

* Explicit Locks offer an extended feature set compared to intrinsic locking, including greater flexibility in dealing with lock unavailability and greater control over queueing behavior. But ReentrantLock is not a blanket substitute for synchronized; use it only when you need features that synchronized lacks. Read-write locks allow multiple readers to access a guarded object concurrently, offering the potential for improved scalability when accessing read-mostly data structures.

* If you need to implement a state-dependent class - one whose methods must block if a state-based precondition does not hold - the best strategy is usually to build upon an existing library class such as Semaphore, BlockingQueue, or CountDownLatch, as in ValueLatch on page 187. However, sometimes existing library classes do not provide a sufficient foundation; in these cases, you can build your own synchronizers using intrinsic condition queues, explicit Condition objects, or AbstractQueuedSynchronizer. Intrinsic condition queues are tightly bound to intrinsic locking, since the mechanism for managing state dependence is necessarily tied to the mechanism for ensuring state consistency. Similarly, explicity Conditions are tightly bound to explicit Locks, and offer an extended feature set compared to intrinsic condition queues, including multiple wait sets per lock, intrerruptible or uninterrruptible condition waits, fair or nonfair queuing, and deadline-based waiting.

* The Java Memory Model specifies when the actions of one thread on memory are guaranteed to be visible to another. The specifics invole ensuring that operations are ordered by a partial ordering called happens-before, which is specified at the level of individual memory and synchronization operations. In the absence of sufficient synchronization, some very strange things can happen when threads access shared data. However, the higher-level rules offered in Chapters 2 and 3, such as @GuardedBy and safe publication, can be used to ensure thread safety without resorting to the low-level details of happens-before.
