# Synchronization and Locks

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Synchronization protects invariants across concurrent operations. Correct locking requires identifying shared mutable state, choosing a consistent lock, minimizing critical sections, and understanding both mutual exclusion and memory visibility. Locks should protect a defined invariant, not be added randomly around individual lines.

## 1. Intrinsic monitors

### Q1. What does synchronized guarantee? *(Intermediate)*

A synchronized instance method acquires the receiver's intrinsic monitor; a synchronized static method acquires the Class object's monitor. A synchronized block acquires the monitor of its specified object. Only one thread at a time can own a given monitor.

Exiting a synchronized region happens-before a subsequent successful acquisition of the same monitor. This provides visibility as well as mutual exclusion for correctly coordinated accesses.

### Q2. What is the difference between synchronized instance and static methods? *(Intermediate)*

An instance synchronized method locks this. A static synchronized method locks the Class object associated with the declaring class. They therefore use different monitors and do not automatically exclude each other.

If static and instance methods mutate the same shared state, the different locks may fail to protect the common invariant. Choose a lock based on the state being protected, not merely on the method modifier.

### Q3. What is a synchronized block, and why use it? *(Intermediate)*

A synchronized block lets code choose a specific lock and limit the critical section to the operations that require protection.

    synchronized (lock) {
        // read and update the invariant together
    }

Keep the lock object private and stable. Locking on public objects, strings, or class literals can allow unrelated code to contend for the same monitor and cause unexpected deadlocks.

### Q4. Is synchronized reentrant? *(Intermediate)*

Yes. A thread that already owns an intrinsic monitor can acquire the same monitor again, and the monitor tracks the reentrant hold count. This permits synchronized methods to call other synchronized methods on the same lock without self-deadlocking.

Reentrancy does not make arbitrary nested locking safe. Two threads acquiring multiple locks in inconsistent orders can still deadlock.

### Q5. What is the difference between mutual exclusion and visibility? *(Advanced)*

Mutual exclusion prevents concurrent execution of protected critical sections for the same lock. Visibility ensures writes performed by one thread become observable to another under a defined memory-ordering relationship. Synchronization often provides both.

A variable can be visible but still participate in a non-atomic compound operation. Conversely, a lock around only one side of an access may fail to establish the intended coordination. Protect the whole invariant consistently.

## 2. Explicit locks

### Q6. What is ReentrantLock? *(Advanced)*

ReentrantLock provides explicit locking with capabilities beyond intrinsic monitors, including timed and interruptible lock acquisition, optional fairness, and multiple Condition objects. It must be released in a finally block after successful acquisition.

    lock.lock();
    try {
        updateState();
    } finally {
        lock.unlock();
    }

If a thread acquires the lock and fails to release it, other threads can block indefinitely. Keep critical sections short and avoid calling unknown external code while holding a lock.

### Q7. What is the difference between lock() and tryLock()? *(Advanced)*

lock waits until the lock is acquired. tryLock attempts to acquire immediately and reports success or failure; timed variants can wait for a bounded duration, and interruptible methods allow interruption while waiting.

Use tryLock only when the failure path is meaningful. A failed attempt should not lead to unsafe access to protected state. Timeouts can help avoid indefinite waits but require a defined fallback or retry policy.

### Q8. What is a fair lock? *(Advanced)*

A fair ReentrantLock favors the longest-waiting thread under its documented acquisition policy, reducing some starvation scenarios. Fairness can reduce throughput and does not guarantee operating-system scheduling fairness.

Choose fairness only when the latency/ordering requirement justifies its cost. Most systems should begin with the simpler default and measure starvation or tail-latency evidence before tuning.

### Q9. What is Condition? *(Advanced)*

Condition provides wait/notify-style coordination associated with an explicit Lock and allows multiple condition queues for one lock. await releases the lock while waiting and reacquires it before returning; signal and signalAll wake waiting threads according to the condition contract.

Always await in a loop that checks the protected condition. A signal is not proof that the desired state is true, and interruption/timeout paths must preserve invariants.

### Q10. What is ReadWriteLock, and when is it useful? *(Advanced)*

ReadWriteLock separates read and write locking so multiple readers can proceed concurrently while writes require exclusive access. ReentrantReadWriteLock is a common implementation.

It helps only when reads are sufficiently frequent/long and contention is meaningful. Lock overhead, write starvation, and cache effects can make it slower than a simple synchronized block for small critical sections.

## 3. Lock design and deadlocks

### Q11. What is lock ordering? *(Advanced)*

Lock ordering imposes a global order in which multiple locks must be acquired. If every code path follows the same order, circular wait can be prevented for those locks.

For example, assign stable ranks to account locks and always acquire the lower-ranked lock first. Avoid relying on incidental object identity or call order; encapsulate the protocol and test concurrent transfers.

### Q12. What are the four Coffman conditions for deadlock? *(Advanced)*

Deadlock requires mutual exclusion, hold-and-wait, no preemption, and circular wait. Breaking at least one condition can prevent deadlock in a given design. Lock ordering is a common way to prevent circular wait.

Timeouts may detect a stuck acquisition but do not automatically make the algorithm correct. A timed-out operation must release held locks and restore or abandon partial work safely.

### Q13. Why should external calls be avoided inside critical sections? *(Senior)*

External calls can block, perform I/O, acquire unknown locks, invoke callbacks, or re-enter the same service. Holding a lock across them increases contention and can create lock-order cycles involving code outside your control.

Capture the needed state under the lock, release it, perform the external operation, then reacquire and validate state if necessary. This pattern may require a state machine or optimistic validation to preserve correctness.

### Q14. Can synchronized prevent all race conditions? *(Advanced)*

Only if all accesses participating in the invariant follow a compatible synchronization protocol. A field read outside the lock or a compound operation split across separate synchronized blocks can still race.

Identify the invariant and enumerate every read/write path, including callbacks and background tasks. Use encapsulation to keep protected state and its locking policy together.

## 4. Production diagnosis

### Q15. How do you diagnose deadlock? *(Senior)*

Capture a thread dump and inspect monitor ownership and waiting relationships. JVM thread dumps often identify Java-level deadlocks automatically, but lock contention without a cycle can also cause severe latency.

Use repeated dumps, lock/blocked-time metrics, and tracing to identify the path. Fix inconsistent lock ordering or long critical sections rather than simply restarting the service and losing the evidence.

### Q16. What is lock contention, and how can it affect throughput? *(Senior)*

Lock contention occurs when multiple threads compete for the same lock, causing waiting and reducing parallel progress. A heavily contended lock can serialize a workload even when many CPU cores are available.

Measure blocked time and throughput, then consider reducing critical-section duration, partitioning state, using immutable snapshots, or choosing a concurrent data structure. Fine-grained locking adds complexity and should be justified by evidence.

### Q17. What is lock striping? *(Advanced)*

Lock striping divides a larger shared state into multiple locks, allowing unrelated partitions to proceed concurrently. It reduces contention when operations can be mapped safely to independent stripes.

Operations that touch multiple stripes need a consistent acquisition order. More locks increase complexity and memory overhead, and hash collisions can still cause unrelated operations to contend on the same stripe.

### Q18. How should you choose between synchronized and ReentrantLock? *(Senior)*

Use synchronized for straightforward lexical critical sections because lock release is automatic and the code is concise. Choose ReentrantLock when timed/interruptible acquisition, multiple conditions, fairness, or other explicit lock operations are required.

Do not choose explicit locks just because they seem more advanced. Whichever mechanism is used, define the protected invariant, minimize the critical section, and test contention and failure paths.

## Official references

1. [Java Language Specification — Synchronization](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
2. [Java SE 21 API — ReentrantLock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)
3. [Java SE 21 API — Lock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/Lock.html)
4. [Java SE 21 API — Condition](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/Condition.html)

## Chapter checklist

- [ ] Explain monitor ownership, reentrancy, mutual exclusion, and visibility.
- [ ] Use explicit locks with reliable unlock paths.
- [ ] Prevent deadlock through consistent lock ordering.
- [ ] Keep external calls outside critical sections where possible.
- [ ] Diagnose contention using dumps and measurements.

## Worked examples: locks and condition invariants

### Example 1. Protect a compound invariant with one lock

~~~java
final class SafeCounter {
    private int count;

    synchronized void increment() {
        count++;
    }

    synchronized int get() {
        return count;
    }
}
~~~

The same monitor protects both the update and the read. Synchronizing only the writer or only the reader would not establish the intended mutual-exclusion protocol for all accesses. Keep the protected invariant and the lock boundary explicit.

### Example 2. Wait in a loop that checks the condition

~~~java
synchronized (lock) {
    while (!conditionIsTrue()) {
        lock.wait();
    }
    consumeConditionProtectedState();
}
~~~

A thread can wake without the application condition being true, and another thread may consume the condition before this thread reacquires the monitor. The loop rechecks the invariant after every wake-up. The condition and state must be guarded by the same coordination protocol.

### Example 3. Avoid lock-order deadlocks

If transfer code must lock two accounts, impose one stable ordering rule and acquire both locks in that order. Without a global ordering, thread A can hold lock X while waiting for Y, while thread B holds Y while waiting for X. Thread dumps can expose the cycle; prevention is generally safer than attempting to recover automatically.
