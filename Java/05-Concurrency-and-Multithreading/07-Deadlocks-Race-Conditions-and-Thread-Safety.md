# Deadlocks, Race Conditions and Thread Safety

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Thread safety means an object's behavior remains correct when accessed concurrently according to its documented contract. It requires reasoning about state invariants, synchronization, visibility, atomicity, and lifecycle. This chapter focuses on diagnosing and designing around concurrency failures rather than merely defining terms.

## 1. Thread safety and races

### Q1. What does thread-safe mean? *(Beginner)*

A component is thread-safe when concurrent use preserves its stated invariants and contract. Thread safety may come from immutability, thread confinement, synchronization, atomic operations, or a design that avoids shared mutable state.

A class is not thread-safe simply because its fields are private or individual methods are synchronized. Multi-method workflows can still race unless the whole invariant is protected.

### Q2. What is a race condition versus a data race? *(Advanced)*

A data race is a JMM concept involving conflicting accesses without a happens-before ordering. A race condition is broader: program correctness depends on timing or interleaving. A program can have a race condition even when individual memory accesses are synchronized, such as check-then-act logic across separate critical sections.

Fix the invariant rather than only one visible data race. Atomic operations or locks should cover the full decision and state transition that must remain consistent.

### Q3. Why is incrementing a shared counter unsafe? *(Intermediate)*

count++ consists of reading, incrementing, and writing. Two threads can read the same old value and both write the same incremented value, losing an update.

Use AtomicInteger/AtomicLong for single-variable atomic updates or a lock when the counter participates in a larger invariant. Volatile alone does not make increment atomic.

### Q4. What is thread confinement? *(Intermediate)*

Thread confinement gives one thread exclusive ownership of mutable state. Local objects that never escape can be manipulated without synchronization because no other thread can race with them.

Confinement can be enforced by architecture, not just convention. Be careful when returning references, registering callbacks, submitting work to executors, or storing objects in static/shared state.

### Q5. What is immutability's role in thread safety? *(Intermediate)*

Immutable objects do not change observable state after construction, so multiple threads can share them without coordinating mutations. Correct construction and safe publication still matter, and objects reachable through fields must also be immutable or safely managed for deep immutability.

Prefer immutable value objects and copy-on-write state snapshots where practical. Immutability often reduces complexity more effectively than adding locks to every method.

## 2. Deadlock and liveness

### Q6. What is deadlock? *(Beginner)*

Deadlock occurs when two or more threads wait indefinitely for resources held by each other, so none can make progress. A common case is two threads acquiring the same pair of locks in opposite orders.

Prevent deadlock with consistent lock ordering, reducing nested locks, time-bounded acquisition where appropriate, or redesigning ownership. A timeout can detect a stuck attempt but must be paired with safe rollback and release behavior.

### Q7. What are the four Coffman conditions? *(Advanced)*

The four conditions are mutual exclusion, hold-and-wait, no preemption, and circular wait. Deadlock requires all four. A design can prevent deadlock by ensuring at least one condition cannot hold.

In Java applications, consistent lock ordering commonly breaks circular wait. Avoid memorizing the list without explaining which condition the proposed design removes.

### Q8. What is livelock? *(Advanced)*

Livelock occurs when threads remain active but repeatedly react to each other in a way that prevents useful progress. For example, two workers may repeatedly back off and retry in synchronized patterns without completing work.

Randomized backoff, bounded retries, centralized arbitration, or a simpler state protocol can help. Monitor successful progress, not only whether threads are consuming CPU.

### Q9. What is starvation? *(Advanced)*

Starvation occurs when a thread is repeatedly denied the CPU or a resource it needs while other work continues. It can result from unfair locking, priority interactions, long critical sections, or thread pools saturated by unrelated tasks.

Fair locks may help some lock-acquisition scenarios but can reduce throughput and do not guarantee operating-system scheduling fairness. Diagnose with thread dumps and wait-time metrics.

### Q10. What is thread-pool starvation deadlock? *(Advanced)*

It occurs when all workers in a bounded executor block waiting for tasks that are queued to that same executor. The tasks that would release the waiting workers cannot start because no worker is available.

Avoid blocking nested tasks on the same constrained pool. Use asynchronous composition, separate execution domains, or a design that does not require a worker to wait for another task that needs that worker pool.

## 3. Locking strategies

### Q11. How does lock ordering prevent deadlock? *(Advanced)*

Define a total order over locks and require every thread to acquire multiple locks in that order. This prevents a circular wait cycle among locks governed by the protocol.

Use stable identifiers or explicit lock ranks rather than accidental object ordering. If an operation dynamically selects several locks, sort them according to the shared policy before acquisition.

### Q12. What is optimistic concurrency? *(Advanced)*

Optimistic concurrency allows work to proceed assuming conflicts are uncommon, then checks whether the underlying state changed before committing. Examples include version columns in databases and compare-and-set loops in memory.

On conflict, the operation must retry or report failure safely. The approach works best when conflicts are relatively rare and the operation can be repeated without duplicating side effects.

### Q13. What is a critical section? *(Beginner)*

A critical section is code that accesses shared state and must be coordinated to preserve an invariant. The lock should cover the entire logical operation, not just isolated field accesses.

Keep critical sections short, but do not split them so far that the invariant becomes unprotected. Avoid I/O, callbacks, or unknown external calls while holding a lock whenever possible.

## 4. Diagnosis and incident response

### Q14. How do you identify a deadlock from a thread dump? *(Senior)*

Look for threads blocked on monitors or synchronizers, identify the owner of each lock, and trace whether the owners are waiting for locks held by other threads in a cycle. Tools such as jcmd Thread.print can capture thread dumps.

Take repeated dumps and correlate with application symptoms. A single blocked thread is not necessarily deadlocked; it may be waiting for a slow dependency.

### Q15. What is the difference between BLOCKED and WAITING? *(Intermediate)*

BLOCKED means a thread is waiting to acquire an intrinsic monitor. WAITING means it is waiting indefinitely for another thread or action, such as Object.wait, Thread.join without a timeout, or a synchronizer operation. TIMED_WAITING represents waits with a timeout.

Interpret state together with stack frames and lock ownership. A WAITING thread may be healthy if it is waiting on a queue or latch by design.

### Q16. How do you debug a race that appears only in production? *(Senior)*

Capture thread dumps, logs, metrics, and request/task identifiers while preserving evidence. Identify shared mutable state and write down the invariant and synchronization path. Reproduce with stress tests, controlled scheduling, or concurrency-testing tools rather than relying on timing sleeps.

Reduce the state space through immutability or confinement when possible. Add instrumentation that is low overhead and does not itself alter the timing enough to hide the bug.

### Q17. How should a concurrent cache protect consistency? *(Senior)*

Specify whether stale reads are allowed, how loads are coordinated, whether duplicate loads are acceptable, and how eviction interacts with in-flight requests. ConcurrentHashMap provides thread-safe per-operation behavior but does not automatically implement expiration, size bounds, or a complete cache transaction.

Use a caching library or an explicit per-key loading protocol. Ensure failures and cancellations remove or complete in-flight placeholders so waiters cannot block forever.

### Q18. How do you review a class for thread safety? *(Senior)*

List mutable fields, identify which threads can access them, define invariants across fields, and trace how every access is synchronized or confined. Check safe publication, callback escape, object lifecycle, and whether returned views expose mutable state.

Document the class's concurrency policy: immutable, thread-confined, thread-safe, or externally synchronized. Tests should exercise concurrent operations and failure paths, but code-level reasoning remains essential.

## Official references

1. [Java Language Specification — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
2. [Java SE 21 API — Thread](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html)
3. [Java SE 21 API — ReentrantLock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)
4. [Java SE 21 API — jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)

## Chapter checklist

- [ ] Distinguish data races, race conditions, deadlock, livelock, and starvation.
- [ ] Protect invariants, not just individual statements.
- [ ] Prevent deadlock through ordering and ownership.
- [ ] Diagnose concurrency failures with dumps, metrics, and controlled tests.
- [ ] Define a clear thread-safety contract for every shared component.

## Worked examples: race detection and safe ownership

### Example 1. A read-modify-write operation is not atomic

~~~java
class UnsafeCounter {
    private int count;
    void increment() { count++; }
    int get() { return count; }
}
~~~

Multiple threads can read the same old value and overwrite each other's updates. Use synchronization, an atomic type, or thread confinement depending on whether the count is an independent statistic or part of a larger invariant.

### Example 2. Prefer immutable values across thread boundaries

A fully initialized immutable value can often be shared safely when it is properly published. Immutability prevents later state mutation from racing, but it does not itself repair unsafe publication or make mutable objects referenced by its fields immutable. Design the entire reachable state graph deliberately.

### Example 3. A thread dump is evidence, not the whole diagnosis

Take multiple thread dumps a short interval apart and compare the same stacks, lock ownership, and thread states. Correlate with CPU, queue depth, request latency, and connection-pool metrics. One WAITING thread is often normal; a cycle of monitor ownership or repeated lack of progress is more informative.
