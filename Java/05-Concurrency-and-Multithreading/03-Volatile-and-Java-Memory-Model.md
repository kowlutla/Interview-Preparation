# Volatile and the Java Memory Model

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

The Java Memory Model (JMM) defines when writes by one thread become visible to another and which reorderings are allowed. Correct concurrent programs establish happens-before relationships through synchronization, volatile access, thread start/join, and concurrency utilities rather than relying on timing or hardware intuition.

## 1. Visibility, atomicity and ordering

### Q1. What is the Java Memory Model? *(Intermediate)*

The JMM specifies the legal interactions between threads and memory, including visibility and ordering of reads and writes. It defines happens-before relationships and rules that let compilers and processors optimize execution without violating Java's concurrency semantics.

The JMM is a language-level contract, not a literal diagram of CPU caches. Correctness should be reasoned from happens-before and synchronization rules rather than assumptions about one machine.

### Q2. What is a data race? *(Intermediate)*

A data race occurs when conflicting accesses to the same variable are performed by different threads, at least one is a write, and the accesses are not ordered by happens-before. Data races can permit surprising observations under the JMM.

Avoid races through thread confinement, immutable state, locks, volatile where appropriate, atomics, or concurrent data structures. A program may appear to work on one CPU and still be incorrect.

### Q3. What is happens-before? *(Advanced)*

Happens-before is a partial order that guarantees visibility and ordering between actions. Program order within a thread contributes to the relation, and synchronization actions create cross-thread edges. If action A happens-before action B, B must observe A's effects according to the memory-model rules.

Examples include unlocking a monitor before another thread locks the same monitor, a volatile write before a subsequent read of that variable, and actions before Thread.start happening-before actions in the started thread. Thread.join also provides a completion visibility guarantee.

### Q4. What does volatile guarantee? *(Intermediate)*

A volatile read observes a write to that volatile variable under the JMM's synchronization rules, and volatile accesses establish ordering/visibility relationships. Volatile does not make arbitrary compound operations atomic.

    volatile boolean ready;
    int data;

    // writer: data = 42; ready = true;
    // reader: if (ready) use(data);

If the reader observes ready as true through the volatile read, the prior write to data is visible under the happens-before relationship. The example still requires a clear one-time publication protocol; multiple writers or repeated cycles need additional design.

### Q5. Why is volatile int count++ not atomic? *(Intermediate)*

Increment is a read-modify-write sequence: read count, add one, then write the result. Volatile makes each access visible and ordered, but two threads can both read the same old value and overwrite each other's increment.

Use AtomicInteger.incrementAndGet, a synchronized block, or another suitable atomic/locking design. Volatile is appropriate for simple state publication and flags when the update itself does not require compound atomicity.

### Q6. What is the difference between volatile and synchronized? *(Advanced)*

Volatile provides visibility and ordering for a particular variable without mutual exclusion. Synchronized provides mutual exclusion and happens-before relationships for a monitor-protected region. Synchronized can protect multi-field invariants and compound operations; volatile generally cannot.

Use volatile for simple state flags or publishing a reference when the protocol is valid. Use locks or atomics when multiple operations must act as one logical update.

### Q7. What is safe publication? *(Advanced)*

Safe publication ensures that another thread sees an object's initialized state through a valid synchronization relationship. Examples include publishing through a volatile reference, under a lock, from static initialization, or through properly used concurrent utilities.

Publishing a reference through an unsynchronized mutable field does not generally establish visibility for all preceding writes. Final fields have special guarantees when construction is correct and this does not escape, but that is not a reason to ignore safe publication.

## 2. Reordering and common puzzles

### Q8. Can the compiler reorder statements? *(Advanced)*

Yes, the compiler and runtime may reorder operations as long as single-thread semantics and JMM constraints are preserved. Without a happens-before relationship, another thread may observe effects in an order that source-code intuition does not predict.

Do not reason from source order alone for racy code. Add synchronization that expresses the intended ordering, then reason from the memory model.

### Q9. Why is double-checked locking historically associated with a bug? *(Advanced)*

Without volatile on the shared instance reference, publication can be reordered or observed in a way that exposes a reference before the object is fully initialized under older memory-model assumptions. Correct modern double-checked locking uses a volatile field and synchronized initialization.

    private static volatile Service instance;
    static Service getInstance() {
        Service result = instance;
        if (result == null) {
            synchronized (Service.class) {
                result = instance;
                if (result == null) instance = result = new Service();
            }
        }
        return result;
    }

This is still more complex than a static holder or enum singleton. Prefer simpler initialization-on-demand idioms unless measurement or requirements justify the complexity.

### Q10. What are final-field semantics? *(Advanced)*

The JMM provides special visibility guarantees for properly constructed objects with final fields, assuming the object does not escape during construction. A thread that obtains the reference can receive stronger guarantees about final-field values and certain reachable state as defined by the specification.

These rules do not make the whole object graph immutable or thread-safe, and they do not replace safe publication for every mutable field. Keep constructors simple and avoid leaking this before initialization completes.

### Q11. Why is Thread.sleep not a memory barrier? *(Advanced)*

sleep changes scheduling/timing but does not establish a happens-before relationship for arbitrary shared variables. A reader may still observe stale values if the program has a data race.

Use volatile, locks, atomics, latches, futures, or another documented synchronization mechanism. Timing-based tests are especially unreliable because they may pass on one environment and fail under load.

### Q12. What is instruction reordering in a publication bug? *(Senior)*

A publication bug occurs when a reference becomes visible to another thread without a synchronization relationship that guarantees visibility of all initialization writes. The JIT and hardware can reorder or buffer operations within the allowed memory model, so the consumer may observe inconsistent state.

The fix is not to insert arbitrary delays. Establish a happens-before edge through a valid publication mechanism and ensure the object's construction invariants hold before it is shared.

## 3. Atomics and synchronization selection

### Q13. When should you use AtomicInteger instead of volatile int? *(Intermediate)*

Use AtomicInteger when you need atomic operations such as incrementAndGet, compareAndSet, or getAndAdd. Use volatile int only when individual reads/writes are enough and no compound update must be atomic.

Atomic classes generally provide lock-free operations for common cases, but high contention can still cause retries and performance costs. For complex invariants spanning multiple variables, a lock may be simpler and more correct.

### Q14. What is compare-and-set (CAS)? *(Advanced)*

CAS atomically compares a memory location with an expected value and updates it if the expected value matches, returning whether the update succeeded. It is a building block for atomic counters, state machines, and lock-free algorithms.

CAS loops must handle contention and retry correctly. They can also face the ABA problem, where a value changes from A to B and back to A, making a simple equality check miss the intermediate change.

### Q15. What is the ABA problem? *(Advanced)*

The ABA problem occurs when a CAS operation sees the same value it expected even though the state changed away and then back. The algorithm may incorrectly assume nothing changed.

Solutions include version stamps, sequence numbers, or higher-level synchronization. AtomicStampedReference is one Java utility that pairs a reference with a stamp, though it adds complexity and is not necessary for every CAS use.

## 4. Production diagnosis

### Q16. How do you diagnose a visibility bug? *(Senior)*

Look for unsynchronized shared mutable fields, incorrect publication, and compound operations that are assumed atomic. Review every read/write path and identify the happens-before edge intended to connect them. Use concurrency tests and tools, but do not treat a test that happens to pass as proof.

Simplify the design using confinement, immutability, volatile publication, locks, or concurrent primitives. Memory-model bugs are often easiest to prevent by reducing shared mutable state.

### Q17. How should a shutdown flag be implemented? *(Intermediate)*

A shared shutdown flag may be volatile or accessed under a lock, so worker threads observe changes. If shutdown also requires waking a blocked operation, a flag alone may not be enough; use interruption, queue signaling, or a suitable cancellation mechanism.

Define how quickly shutdown must take effect and how in-flight work is handled. A volatile flag does not forcibly unblock a thread stuck in I/O.

### Q18. How do you choose between volatile, atomics, and locks? *(Senior)*

Use volatile for simple visibility/ordering of a variable when no compound atomic update is required. Use atomics for single-variable atomic transformations and CAS protocols. Use locks for multi-step invariants, multiple related fields, or conditions requiring coordinated waiting.

The simplest correct mechanism is usually best. Consider contention, cancellation, fairness, observability, and maintainability—not only raw throughput.

## Official references

1. [Java Language Specification, Chapter 17 — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
2. [Java SE 21 API — volatile and concurrency package overview](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
3. [Java SE 21 API — AtomicInteger](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html)
4. [Java SE 21 API — AtomicStampedReference](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicStampedReference.html)

## Chapter checklist

- [ ] Explain data races, visibility, atomicity, and happens-before.
- [ ] Choose volatile only when compound atomicity is unnecessary.
- [ ] Explain safe publication, final-field semantics, and reordering.
- [ ] Recognize CAS retries and the ABA problem.
- [ ] Prefer simpler concurrency designs over timing assumptions.

## Worked examples: visibility is not compound atomicity

### Example 1. A volatile flag publishes a stop request

\`\`\`java
final class Worker implements Runnable {
    private volatile boolean running = true;

    void stop() {
        running = false;
    }

    @Override public void run() {
        while (running) {
            doSmallUnitOfWork();
        }
    }

    private void doSmallUnitOfWork() {
        // One bounded unit of work; production code should avoid a busy-spin.
    }
}
\`\`\`

The volatile write of `false` happens-before a subsequent read of that value by another thread. This makes the flag's update visible. It does not guarantee prompt shutdown if the worker is blocked in an uninterruptible operation, and a real worker should generally use blocking coordination or interruption rather than spin continuously.

### Example 2. Volatile does not make increment atomic

\`\`\`java
final class Counter {
    private volatile int count;

    void increment() {
        count++; // read, add, write: not one atomic operation
    }

    int get() {
        return count;
    }
}
\`\`\`

Two threads can read the same old value and overwrite one another's updates. Use `AtomicInteger.incrementAndGet()` for an independent counter, `LongAdder` for high-contention statistics where its semantics fit, or a lock when the update participates in a larger invariant.

### Example 3. Safe publication requires an actual happens-before path

Publishing a fully constructed object through a volatile reference, a synchronized handoff, a properly used concurrent collection, or the specified initialization-safety guarantees of final fields can establish visibility. Merely assigning a reference to an ordinary shared field and hoping another thread observes all constructor writes is not a synchronization strategy.

The Java Memory Model's rules are normative; CPU intuition and tests that happen to pass do not prove correctness. See JLS Chapter 17 and the API contracts of the chosen coordination primitive.

## Worked example: publication, visibility, and ordering

### Example. A volatile reference can publish prior initialization writes

~~~java
final class Config {
    final int limit;
    Config(int limit) { this.limit = limit; }
}

final class ConfigHolder {
    private volatile Config current;

    void publish() {
        current = new Config(100);
    }

    Config read() {
        return current;
    }
}
~~~

The volatile write to current happens-before a subsequent read that observes that write. The construction writes that precede publication are therefore visible to the observing thread under the Java Memory Model. This does not make later mutations to Config safe; the example is deliberately immutable. A plain shared reference without a suitable happens-before path does not provide the same guarantee.

### Checklist for a shared-state claim

When reviewing a concurrency answer, identify: the shared variable, every read/write, the synchronization mechanism, the happens-before edge, whether the operation is compound, and whether object state can mutate after publication. “It usually works” and a passing stress test do not establish correctness.
