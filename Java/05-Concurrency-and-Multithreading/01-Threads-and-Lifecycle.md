# Threads and Lifecycle

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Concurrency allows multiple tasks to make progress during overlapping periods. Correct Java concurrency depends on coordination, visibility, atomicity, and ownership—not simply creating more threads. This chapter establishes thread behavior before moving to synchronization, the Java Memory Model, executors, and virtual threads.

## 1. Thread fundamentals

### Q1. What is a process versus a thread? *(Beginner)*

A process is an operating-system execution environment with resources and an address space. A thread is an execution path within a process and shares many process resources with other threads, including heap objects. Each Java platform thread has its own call stack and execution state.

Shared memory makes communication efficient but introduces races when threads access mutable state without a correct coordination protocol. Separate processes offer stronger isolation but require inter-process communication.

### Q2. What are the Java thread states? *(Intermediate)*

Thread.State includes NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, and TERMINATED. These are JVM-level states, not a perfect one-to-one mapping to every operating-system scheduling state.

RUNNABLE can include a thread that is executing or ready to execute. BLOCKED specifically refers to waiting to acquire an intrinsic monitor; waiting on a condition or sleeping is represented by other states.

### Q3. What is the difference between start() and run()? *(Beginner)*

start() arranges for a new thread of execution to invoke run(). Calling run() directly is an ordinary method call on the current thread and does not create a new thread.

A Thread instance can be started only once; attempting to start it again throws IllegalThreadStateException. For reusable task definitions, prefer Runnable or Callable submitted to an ExecutorService.

### Q4. What is the difference between Runnable and Callable? *(Intermediate)*

Runnable.run returns no result and cannot declare checked exceptions in its method signature. Callable<V>.call returns a value and may throw checked exceptions. ExecutorService can submit either, with Callable submissions typically returning a Future.

Use Callable when a task result or checked failure needs to be observed. Use Runnable for a task whose result is communicated through another explicit mechanism.

### Q5. What is daemon thread behavior? *(Intermediate)*

A daemon thread does not prevent the JVM from exiting when only daemon threads remain. Daemon status is set before start and is inherited by newly created threads according to the thread-creation rules.

Do not use daemon status as a substitute for graceful shutdown. Important writes, resource cleanup, and business work should be managed explicitly because daemon threads may stop when the process exits.

### Q6. What is the difference between sleep and wait? *(Intermediate)*

Thread.sleep pauses the current thread for a time and does not release monitors it owns. Object.wait requires the caller to own that object's monitor and releases that monitor while waiting, reacquiring it before returning.

wait is used with a condition protected by the monitor, and it should be called in a loop. sleep is a timing operation, not a coordination mechanism.

### Q7. What is the difference between join and yield? *(Intermediate)*

join waits for another thread to terminate, optionally with a timeout. yield is only a scheduler hint that the current thread is willing to yield execution; the scheduler may ignore it.

Use join or a higher-level completion primitive when one task depends on another. Do not rely on yield or sleep timing to establish correctness or memory visibility.

## 2. Interruption and cancellation

### Q8. What is thread interruption? *(Intermediate)*

Interruption is a cooperative signal that asks a thread to stop waiting or cancel work. Some blocking methods throw InterruptedException and clear the interrupt status when they do so; other APIs respond differently.

Interruption does not forcibly terminate a thread. Code must respond by propagating the exception, restoring the interrupt flag, or completing a defined cancellation path.

### Q9. Why should InterruptedException not be swallowed? *(Advanced)*

Swallowing interruption can prevent shutdown, cancellation, or task coordination from working. If a method cannot propagate InterruptedException, it commonly restores the status with Thread.currentThread().interrupt() and then exits or translates the failure appropriately.

Do not continue indefinitely as though nothing happened. The correct response depends on whether the operation can safely stop and whether cleanup or rollback is required.

### Q10. What does Thread.interrupted() do differently from isInterrupted()? *(Advanced)*

Thread.interrupted() is a static method that checks and clears the current thread's interrupt status. isInterrupted() checks a specific thread's status without clearing it.

Accidentally calling Thread.interrupted() in diagnostic or control code can consume a cancellation signal. Use the non-clearing check when the status must remain available to higher layers.

### Q11. How should a task be cancelled? *(Senior)*

Use cooperative cancellation: interrupt the task when appropriate, stop scheduling additional work, release resources, and make the task periodically observe cancellation. Future.cancel(true) requests interruption of the executing thread but cannot guarantee that arbitrary code stops.

Tasks must be written to respond safely to interruption and to avoid leaving shared state inconsistent. Define cancellation semantics for external side effects, because an interrupted request may already have committed remotely.

## 3. Thread creation and design

### Q12. Why prefer ExecutorService over manually creating threads for each task? *(Intermediate)*

Executors separate task submission from thread management and can reuse workers, bound concurrency, return task results, and support orderly shutdown. Creating a new platform thread for every small task can incur overhead and make resource limits hard to control.

Choose executor configuration based on workload. A thread pool with an unbounded queue can still accumulate unbounded pending work, so thread reuse alone does not provide backpressure.

### Q13. What is a race condition? *(Beginner)*

A race condition occurs when the result depends on timing or interleaving of concurrent operations in a way the program does not correctly control. A classic example is two threads incrementing a shared integer with a read-modify-write sequence.

Fix races with synchronization, atomics, confinement, immutability, or an algorithm designed for concurrent access. Adding sleeps is not a correctness fix.

### Q14. What is thread confinement? *(Advanced)*

Thread confinement means mutable state is owned by one thread and not accessed concurrently by others. Local variables and objects that do not escape a thread can often be used without locks.

Confinement can be simpler and faster than shared mutable state. Ensure references do not escape through callbacks, static fields, shared collections, or asynchronous tasks.

### Q15. What is safe publication? *(Advanced)*

Safe publication ensures another thread observes an object and its initialized state according to the Java Memory Model. Mechanisms include static initialization, volatile references, synchronized handoff, final-field guarantees under correct construction, and concurrent utilities.

Without a happens-before relationship, another thread may observe stale state or improperly published initialization. Do not assume that assigning a reference to a shared field automatically makes every associated mutation visible.

## 4. Production scenarios

### Q16. How do you diagnose a thread that appears stuck? *(Senior)*

Capture thread dumps and identify whether the thread is RUNNABLE, BLOCKED, WAITING, or TIMED_WAITING. Inspect stack frames, monitor ownership, lock chains, repeated dumps over time, and relevant I/O or executor state.

A single dump is a snapshot; repeated dumps help distinguish a stable deadlock from slow progress. Correlate thread names with requests and tasks, and use metrics for queue depth and task duration.

### Q17. Why are sleeps unreliable for thread coordination? *(Advanced)*

sleep only delays the current thread for at least an approximate interval; it does not guarantee another thread has reached a desired state and does not establish the required memory-visibility relationship. Scheduler delays can make timing assumptions fail under load.

Use latches, barriers, semaphores, futures, queues, or condition variables to express the actual coordination condition. Timeouts are useful for bounded waiting, not as proof that an event occurred.

### Q18. How should thread names and uncaught exceptions be managed? *(Senior)*

Use meaningful names or executor thread factories so dumps and logs identify the task domain. Configure uncaught-exception handling at suitable boundaries, but remember exceptions from tasks submitted to an ExecutorService may be captured in a Future rather than reaching the thread's uncaught handler.

Observe task failures explicitly through futures, structured logging, metrics, and error handling. A background task that fails silently can leave a service degraded without obvious symptoms.

## Official references

1. [Java SE 21 API — Thread](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html)
2. [Java SE 21 API — Runnable](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Runnable.html)
3. [Java SE 21 API — Callable](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Callable.html)
4. [Java SE 21 API — ExecutorService](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ExecutorService.html)
5. [Java Language Specification — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)

## Chapter checklist

- [ ] Distinguish thread states and start/run semantics.
- [ ] Handle interruption as cooperative cancellation.
- [ ] Explain races, confinement, and safe publication.
- [ ] Use executors and coordination primitives rather than timing tricks.
- [ ] Diagnose stuck threads with dumps and task-level observability.

## Worked examples: lifecycle, joining, and interruption

### Example 1. start() schedules a new thread; run() is an ordinary call

~~~java
Thread worker = new Thread(() -> System.out.println("worker"));
worker.run();   // runs on the current thread
worker.start(); // starts a separate thread
~~~

Calling run() directly does not create a new thread. The second call requests a new thread of execution. A Thread instance cannot be started more than once; attempting to restart it throws IllegalThreadStateException.

### Example 2. Use join to wait for completion

~~~java
Thread worker = new Thread(() -> performBoundedWork());
worker.start();
worker.join();
System.out.println("work completed");
~~~

join() waits for the target thread to terminate. In real code, decide how interruption of the waiting thread should be handled and whether a timeout is required. Do not rely on sleeps to coordinate completion; use lifecycle and synchronization primitives.

### Example 3. Interruption is a cooperative cancellation request

~~~java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        performOneBoundedStep();
    }
});
worker.start();
worker.interrupt();
~~~

Interruption does not forcibly terminate arbitrary code. Blocking methods may throw InterruptedException and clear the status; code must propagate, restore, or deliberately handle the request. A production task should define its cleanup and cancellation behavior.
