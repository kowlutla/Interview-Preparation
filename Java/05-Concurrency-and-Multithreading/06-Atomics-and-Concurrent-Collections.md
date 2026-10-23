# Atomics and Concurrent Collections

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Atomic classes and concurrent collections provide building blocks for shared state without manually synchronizing every operation. They do not automatically make multi-step business invariants atomic. Choose the smallest primitive that correctly expresses the update, and understand contention, visibility, and compound-operation semantics.

## 1. Atomic variables

### Q1. What is AtomicInteger? *(Intermediate)*

AtomicInteger supports atomic operations on an int, including get, set, incrementAndGet, getAndAdd, and compareAndSet. These operations allow multiple threads to update one variable without a separate synchronized block for each operation.

Atomicity is limited to the provided operation. If an invariant spans several variables, use a lock or a higher-level algorithm rather than assuming multiple atomic calls form one transaction.

### Q2. What is compareAndSet? *(Advanced)*

compareAndSet(expected, update) atomically updates the value only if it currently equals expected and returns whether the update occurred. It supports retry loops and lock-free state transitions.

A CAS loop must recompute based on the latest observed value after failure. Do not put irreversible side effects inside a retry loop because the body can execute multiple times.

### Q3. What is LongAdder, and when is it better than AtomicLong? *(Advanced)*

LongAdder spreads updates across multiple internal cells under contention and combines them when sum is requested. It can provide higher throughput for highly contended statistical counters, while AtomicLong provides a single atomic value with stronger suitability for exact read-modify-write coordination.

LongAdder.sum is not an atomic snapshot with respect to concurrent updates. It is excellent for metrics where approximate concurrent observation is acceptable, but not for sequence IDs or correctness-critical balance values.

### Q4. What is the ABA problem? *(Advanced)*

A CAS operation can observe value A, another thread can change A to B and back to A, and the first thread's CAS may succeed despite the intermediate state change. This can break algorithms that assume the value never changed.

Use version stamps or a design that makes state transitions observable. AtomicStampedReference pairs a reference with a stamp, but added complexity should be justified by the actual algorithm.

### Q5. What is false sharing? *(Advanced)*

False sharing occurs when independent variables used by different threads occupy the same cache line, causing cache-coherence traffic even though the threads do not logically share the variable. It can degrade throughput in highly contended low-level code.

Diagnose with profiling and benchmarks before padding fields or changing data layout. The JVM and hardware determine physical placement, so source-level adjacency does not guarantee a particular cache layout.

## 2. Concurrent collections

### Q6. How does ConcurrentHashMap differ from HashMap? *(Intermediate)*

ConcurrentHashMap supports concurrent retrieval and updates with documented thread-safety and atomic operations. It rejects null keys and values, which avoids ambiguity in concurrent lookup. HashMap is not safe for unsynchronized concurrent mutation.

ConcurrentHashMap does not make arbitrary compound workflows atomic. Use methods such as computeIfAbsent, merge, or explicit coordination for per-key transitions, and do not use size() as a correctness gate under concurrent updates.

### Q7. What is a weakly consistent iterator? *(Advanced)*

An iterator over many concurrent collections tolerates concurrent modification and may reflect some updates without throwing ConcurrentModificationException. It is not necessarily a snapshot and does not guarantee that every concurrent change is observed.

This can be appropriate for monitoring and concurrent traversal, but not for algorithms that require a stable point-in-time view. Use a snapshot or coordination protocol when exact consistency is required.

### Q8. When is CopyOnWriteArrayList useful? *(Advanced)*

CopyOnWriteArrayList copies its backing array on mutation, while reads and iterators operate on stable snapshots. It is useful for read-heavy, write-rare collections such as listener registries.

Each write can allocate and copy an array, making it unsuitable for frequent mutation or very large lists. Snapshot iterators do not see later writes and do not support iterator mutation.

### Q9. What is BlockingQueue useful for? *(Intermediate)*

BlockingQueue coordinates producers and consumers with operations that wait for an element or capacity. It provides a useful handoff and memory-consistency contract and is commonly used for work queues.

Queue capacity and full-queue behavior are architectural decisions. Unbounded queues can retain unlimited pending work; bounded queues require a rejection, blocking, or backpressure policy.

### Q10. What is ConcurrentSkipListMap? *(Advanced)*

ConcurrentSkipListMap is a concurrent navigable map with sorted keys and expected logarithmic operations. It supports ordered traversal and range/navigation methods under concurrent updates.

It can be useful when both concurrency and sorted navigation are required. It has different performance and consistency characteristics from ConcurrentHashMap, so choose based on the required operations rather than using one concurrent map universally.

## 3. Correctness patterns

### Q11. Why is check-then-act unsafe with concurrent collections? *(Advanced)*

A sequence such as if (!map.containsKey(key)) map.put(key, value) has a race between the check and the update. Another thread can change the map after the check, so the sequence is not atomic.

Use putIfAbsent or computeIfAbsent when their semantics fit. If the invariant spans multiple keys or external state, a per-key atomic method may not be sufficient; use a lock or transaction at the correct boundary.

### Q12. Why is ConcurrentHashMap.computeIfAbsent mapping function constrained? *(Advanced)*

The mapping function participates in an atomic per-key update protocol and must not recursively update the same map in prohibited ways. Long-running or blocking mapping functions can delay other operations that contend for related internal state.

Keep the function short and avoid side effects that cannot safely be repeated or abandoned. For expensive remote loading, consider a carefully designed future-valued cache or caching library with explicit load coordination.

### Q13. What is the difference between AtomicReference and volatile reference? *(Advanced)*

A volatile reference provides visibility and ordering for reference reads/writes, but does not make compound compare-and-update atomic. AtomicReference adds operations such as compareAndSet and updateAndGet.

Use volatile when publishing a replacement reference is sufficient. Use AtomicReference when the update must conditionally compare and replace. A lock may be clearer for multi-field invariants.

### Q14. How should a concurrent counter be designed? *(Senior)*

Choose based on semantics. AtomicLong is suitable for an exact atomic counter or sequence generator under its contract; LongAdder is often better for high-contention metrics where a non-atomic sum snapshot is acceptable. A database sequence or distributed coordination mechanism may be required for global uniqueness across processes.

Define overflow behavior, reset semantics, and whether the value must be linearizable. Do not mistake a process-local atomic counter for a distributed ID generator.

### Q15. How do you test concurrent data structures? *(Senior)*

Test the documented contract under contention, including lost updates, duplicate insertion, visibility, cancellation, and iterator behavior. Use stress tests and concurrency-testing tools where possible; ordinary unit tests may not expose rare interleavings.

Avoid asserting one specific interleaving unless synchronization explicitly enforces it. Include timeouts and diagnostics so a deadlocked test fails usefully instead of hanging indefinitely.

## Official references

1. [Java SE 21 API — Atomic package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)
2. [Java SE 21 API — ConcurrentHashMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
3. [Java SE 21 API — LongAdder](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html)
4. [Java SE 21 API — CopyOnWriteArrayList](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)
5. [Java SE 21 API — ConcurrentSkipListMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentSkipListMap.html)

## Chapter checklist

- [ ] Choose atomic variables based on the required consistency semantics.
- [ ] Explain CAS, ABA, and contention.
- [ ] Distinguish ConcurrentHashMap and concurrent collection iterator semantics.
- [ ] Use atomic compound methods instead of check-then-act sequences.
- [ ] Test correctness under contention and distinguish local from distributed guarantees.

## Worked examples: atomic updates and concurrent maps

### Example 1. Use compare-and-set for a small state transition

~~~java
AtomicInteger state = new AtomicInteger(0);
boolean changed = state.compareAndSet(0, 1);
System.out.println(changed); // true if state was 0 at the instant of the CAS
~~~

CAS atomically changes the value only if it equals the expected value. In retry loops, the update function may run more than once, so it must not perform non-idempotent external side effects. CAS also does not automatically protect a multi-variable invariant.

### Example 2. Use LongAdder for contended statistics, not every counter

~~~java
LongAdder requests = new LongAdder();
requests.increment();
long observed = requests.sum();
~~~

LongAdder can improve throughput under high contention by spreading updates across cells, but sum() is not an atomic snapshot with respect to concurrent updates. It is well suited to statistics; it is not a replacement for a precise sequence number or a counter that must participate in a strict invariant.

### Example 3. Concurrent map does not make contained objects thread-safe

~~~java
ConcurrentHashMap<String, List<String>> groups = new ConcurrentHashMap<>();
groups.computeIfAbsent("a", key -> new ArrayList<>()).add("value");
~~~

The map's own operations are concurrent, but the ArrayList stored as a value remains mutable and unsynchronized. If multiple threads mutate the same value, protect that value, use an appropriate concurrent value type, or redesign the update as an atomic map-level operation.

## Additional review: common concurrency API selection

| Requirement | Common choice | Important limitation |
|---|---|---|
| Exact atomic integer update | AtomicInteger | Does not protect a multi-variable invariant by itself |
| High-contention statistics | LongAdder | sum() is not a linearizable snapshot of all concurrent updates |
| Concurrent key/value operations | ConcurrentHashMap | Mutable values need their own thread-safety design |
| Wait for N one-time events | CountDownLatch | Cannot be reset for another round |
| Reusable phase coordination | CyclicBarrier or Phaser | Requires careful lifecycle and failure handling |
| Limit concurrent access to a resource | Semaphore | Permit acquisition/release must be balanced |
| Producer/consumer handoff with backpressure | BlockingQueue | Queue capacity and shutdown policy must be explicit |
| Read-mostly access with versioned optimistic reads | StampedLock | Optimistic reads must validate; lock is not reentrant |
| Task dependency and completion composition | CompletableFuture | Executor choice, deadlines, and cancellation still matter |

Choose by invariant and lifecycle, not by popularity. A concurrent class does not automatically make a larger workflow atomic, bounded, or failure-safe.
