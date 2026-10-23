# Coordination Primitives and Synchronizers

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Use a synchronizer that directly represents the coordination requirement. A lock protects a critical section; a latch waits for events; a semaphore limits concurrent access; a blocking queue transfers work; a barrier or phaser coordinates phases. These abstractions solve different problems and are not interchangeable.

## 1. One-time and reusable coordination

### Q1. What is CountDownLatch? *(Intermediate)*

CountDownLatch lets one or more threads wait until a count reaches zero. Other threads decrement the count using countDown; waiting threads are released when the count reaches zero.

A latch is one-shot and cannot be reset. Use it for startup readiness, waiting for a fixed set of tasks, or tests that need deterministic coordination rather than sleep-based timing.

### Q2. How does CountDownLatch differ from join? *(Intermediate)*

Thread.join waits for a particular thread to terminate. CountDownLatch represents an event count that can be decremented by threads that need not be the waiting thread's direct children.

Use join when thread lifecycle is the thing being awaited. Use a latch when the requirement is “wait until these events have occurred,” independent of a specific thread object.

### Q3. What is CyclicBarrier? *(Advanced)*

CyclicBarrier allows a fixed number of parties to wait for one another at a common barrier point. Once all parties arrive, they are released and the barrier can be reused for another phase.

All parties must participate consistently. If a party fails or is interrupted, the barrier can break and waiting parties must handle that outcome instead of assuming the phase completed normally.

### Q4. How does Phaser differ from CyclicBarrier? *(Advanced)*

Phaser supports multiple phases and dynamic registration/deregistration of parties. It is useful when the set of participants can change between phases or when a more flexible phase protocol is needed.

That flexibility adds lifecycle complexity. Define who registers, arrives, deregisters, and terminates the phaser; otherwise one missing arrival can stall progress.

### Q5. When would you use a semaphore? *(Intermediate)*

Semaphore maintains a number of permits and allows only a bounded number of operations to hold permits concurrently. It can protect scarce resources such as a downstream service's concurrency budget.

A semaphore is not a mutex by default: permits can be greater than one, and the API does not inherently bind a permit to the thread that acquired it. Ensure permits are released exactly once, usually in a finally block, and use timeouts when waiting indefinitely is unacceptable.

### Q6. What is the difference between a semaphore and a lock? *(Advanced)*

A lock generally provides mutual exclusion and has ownership semantics, while a semaphore controls permit counts and may allow several concurrent holders. A semaphore can model a concurrency limit; a lock usually protects shared state invariants.

Do not replace a lock with a semaphore simply because both can block. Choose based on the invariant, ownership, reentrancy, fairness, and timeout requirements.

## 2. Producer-consumer and work distribution

### Q7. What is BlockingQueue? *(Intermediate)*

BlockingQueue supports thread-safe handoff and buffering between producers and consumers. Operations such as put/take can block when the queue is full or empty, depending on the queue's capacity and operation used.

Choose a bounded queue when overload must be controlled. Queue capacity, rejection behavior, shutdown signaling, and consumer failure handling are part of the design—not optional implementation details.

### Q8. How do bounded and unbounded queues affect a thread pool? *(Senior)*

An unbounded queue can accumulate work until memory or latency becomes unacceptable, and it can prevent a ThreadPoolExecutor from growing beyond its core pool size under its normal queuing policy. A bounded queue forces the system to define what happens when demand exceeds capacity.

Use queue-wait metrics, rejection counts, and downstream latency to set limits. A larger queue can hide overload and increase tail latency rather than increase sustainable throughput.

### Q9. What is Exchanger? *(Advanced)*

Exchanger lets two threads meet and exchange objects at a synchronization point. It can be useful for paired producer-consumer designs such as handing off buffers.

It is specialized and can block indefinitely if a partner never arrives unless a timed form is used. For general work distribution, a BlockingQueue is often simpler.

### Q10. What is ForkJoinPool and work stealing? *(Advanced)*

ForkJoinPool is designed for tasks that can be split into smaller subtasks and joined, with workers able to steal work from other workers' queues. This can balance irregular recursive workloads.

It is not a universal replacement for an I/O thread pool. Blocking tasks can reduce parallel progress unless managed appropriately, and the common pool is shared by unrelated code in many default APIs.

## 3. Advanced locks and conditions

### Q11. What is ReentrantReadWriteLock? *(Advanced)*

ReentrantReadWriteLock supports multiple concurrent readers or one writer, subject to its locking rules. It may help when reads are frequent, writes are relatively rare, and the protected work is long enough to justify lock overhead.

It is not automatically faster than synchronized or ReentrantLock. Benchmark the real contention pattern, account for writer/read fairness, and avoid read-to-write upgrade assumptions that can deadlock.

### Q12. What is StampedLock? *(Advanced)*

StampedLock offers write locks, read locks, and optimistic reads validated by a stamp. Optimistic reads can reduce blocking for suitable read-mostly state, but the reader must validate and retry or take a real read lock if validation fails.

StampedLock is not reentrant and stamps must be managed carefully. Do not let an optimistic read use mutable state in a way that can cause unsafe operations before validation.

### Q13. What is a Condition? *(Advanced)*

Condition associates waiting and signaling operations with a Lock, allowing multiple condition queues for one lock. A thread typically waits in a loop that checks the protected predicate, and a state-changing thread signals after updating the predicate while following the lock protocol.

Signal does not transfer the lock directly or guarantee the predicate remains true when a waiter resumes. Always reacquire the lock and recheck the condition.

### Q14. What is a synchronizer's fairness setting? *(Advanced)*

Some synchronizers offer fairness policies that reduce the chance of indefinitely bypassing earlier waiters, often at a throughput cost. Fairness does not guarantee a fixed completion time or solve starvation caused by external resources.

Choose fairness from an explicit latency and throughput requirement. Measure queueing and contention rather than assuming fair locks always produce better user-visible latency.

## 4. Production reasoning

### Q15. How do you choose a coordination primitive? *(Senior)*

Start by stating the invariant or event: mutual exclusion, N events, phase completion, concurrency limit, producer-consumer transfer, or recursive task coordination. Then select the primitive whose API expresses that requirement directly.

Define interruption, timeout, shutdown, and failure behavior. A correct happy path is insufficient if a participant can fail before signaling or releasing its permit.

### Q16. How can a latch cause a production hang? *(Senior)*

A waiting thread can block forever if a code path fails to call countDown, a task never starts, or an exception bypasses the signaling path. Use finally where appropriate, time-bounded waits when suitable, and metrics/logging that identify unfinished participants.

Do not set a timeout and ignore the timeout result. Decide whether to fail the request, cancel tasks, degrade gracefully, or initiate recovery.

### Q17. Why should sleeps not be used as synchronization? *(Intermediate)*

Thread.sleep delays the current thread for at least a requested interval subject to scheduling and timer behavior; it does not establish a happens-before relationship or prove another task has reached a state.

Use latches, barriers, futures, conditions, or other explicit coordination APIs. In tests, use deterministic handshakes and bounded waits rather than relying on timing luck.

### Q18. How do you prevent permit leaks with a semaphore? *(Senior)*

Acquire a permit, execute the protected operation, and release in a finally block so normal and exceptional exits are handled. If acquisition is interruptible or timed, handle failure before entering the protected region without releasing a permit that was never acquired.

Track permits and active operations when diagnosing saturation. A leaked permit can make a healthy downstream dependency appear permanently unavailable.

### Q19. When should a blocking queue be bounded? *(Senior)*

Bound it whenever producers can outpace consumers and the application needs a defined memory and latency budget. Select capacity based on burst tolerance, consumer service time, and end-to-end deadline rather than arbitrary large numbers.

When full, choose explicit backpressure, rejection, shedding, or caller-runs behavior based on the context. Be careful with caller-runs policies when submission occurs on an event loop or request thread.

### Q20. What should be tested for a custom coordination protocol? *(Senior)*

Test successful progress, empty/one-party cases, interruption, timeout, participant failure, shutdown, and repeated phases if the primitive is reusable. Check invariants rather than asserting one exact thread schedule.

A stress test can reveal races but does not prove their absence. Use the primitive's documented contract and keep shared-state ownership clear.
 
## Official references

1. [Java SE 21 API — java.util.concurrent package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
2. [Java SE 21 API — CountDownLatch](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CountDownLatch.html)
3. [Java SE 21 API — Semaphore](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Semaphore.html)
4. [Java SE 21 API — BlockingQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
5. [Java SE 21 API — Phaser](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Phaser.html)
6. [Java SE 21 API — StampedLock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/StampedLock.html)

## Chapter checklist

- [ ] Choose between latches, barriers, phasers, semaphores, and locks.
- [ ] Explain bounded queue backpressure and executor queuing behavior.
- [ ] Recognize work-stealing and blocking-task limitations.
- [ ] Define interruption, timeout, shutdown, and participant-failure behavior.
