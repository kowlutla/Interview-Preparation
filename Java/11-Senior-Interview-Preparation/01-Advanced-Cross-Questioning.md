# Advanced Java Cross-Questioning Guide

Interviewers often test whether an answer survives a changed assumption. Use these prompts to practice explaining the mechanism, edge cases, trade-offs, and production implications rather than memorizing one response.

## Object model and language

### Q1. You said a record is immutable. What if it contains a mutable list?

A record's component fields are final, but the referenced list may still mutate. Defensively copy mutable inputs and decide whether elements themselves require deep immutability.

### Q2. You said equals compares values. What if a subclass adds a field?

An equals implementation using instanceof can violate symmetry when the subclass includes additional state. Choose a coherent hierarchy policy—often final value types or composition—and test the contract.

### Q3. You said String is pooled. Can you use == for string equality?

No. Pooling is not a general content-equality guarantee. Use equals, and use intern only when canonicalization is intentional and measured.

### Q4. You said generics prevent type errors. Can raw types bypass that?

Yes. Raw types and unchecked casts can cause heap pollution, with ClassCastException later at a read. Keep legacy boundaries narrow and fix type safety at the source.

### Q5. You said an interface is an abstraction. Should every class have an interface?

No. Add an interface when it creates a meaningful substitution point, testing seam, or architectural boundary. An interface without a useful contract can add indirection without value.

## Collections

### Q6. You said HashMap is O(1). Is that guaranteed?

Expected constant-time behavior depends on hash distribution and implementation. Collisions, expensive equals/hashCode, resizing, and memory locality affect performance.

### Q7. You said TreeSet is sorted. Can it drop values that are not equals?

Yes. Comparator equivalence (compare returns zero) governs sorted-set uniqueness. Ideally the comparator is consistent with equals, or the semantics must be documented.

### Q8. You said ConcurrentHashMap is thread-safe. Is check-then-put atomic?

No. Individual methods do not make a multi-call sequence atomic. Use putIfAbsent, computeIfAbsent, or a lock around the complete invariant.

### Q9. You said an unmodifiable list is immutable. What if the backing list changes?

An unmodifiable view can reflect changes to its backing collection. Use an unmodifiable snapshot when stable membership is required, and remember that copying is shallow.

### Q10. You said streams are lazy. Will every peek run?

Not necessarily. Short-circuiting and optimizations can skip work. Never place required business side effects in peek or other intermediate behavioral parameters.

## Concurrency

### Q11. You said volatile gives visibility. Why is volatile count++ unsafe?

Increment is a compound read-modify-write sequence. Use an atomic operation or a lock.

### Q12. You said synchronized is thread-safe. What if a method calls external code while holding the lock?

External code can block, call back, or acquire another lock, causing contention or deadlock. Keep critical sections small and avoid unknown calls while holding locks.

### Q13. You said cancellation interrupts a task. Does that undo a remote write?

No. Cancellation is cooperative, and the remote operation may already have committed. Use idempotency, deadlines, and reconciliation.

### Q14. You said virtual threads are cheap. Can you create unlimited database requests?

No. The database, connection pool, network, and memory remain bounded. Use concurrency limits and backpressure around scarce resources.

### Q15. You said CompletableFuture is asynchronous. Does thenApply always use a worker thread?

No. Non-async continuations can run in the completing thread. Use explicit async stages and executors when isolation is required.

## JVM and operations

### Q16. You said objects are on the heap. Can the JIT eliminate an allocation?

Yes, escape analysis and scalar replacement can remove some physical allocations when semantics permit. It is an optimization, not a language guarantee.

### Q17. You said memory is high. Is that necessarily a heap leak?

No. Native memory, direct buffers, thread stacks, Metaspace, code cache, and container limits can cause process OOM while heap remains healthy.

### Q18. You said GC caused the latency spike. What evidence proves it?

Correlate pause timestamps and request traces, but account for queueing, CPU, and downstream waits. Correlation alone does not prove causation.

## API and security

### Q19. You said retrying a timeout is safe. What if the server committed but the response was lost?

The outcome is unknown. Retry only with idempotency/deduplication or a reconciliation protocol.

### Q20. You said input is validated. Is that enough to prevent SQL injection?

Not if values are concatenated into SQL syntax. Use parameterized statements; validate dynamic identifiers through an allowlist.

### Q21. You said TLS is enabled. What if hostname verification is disabled?

The client may accept an attacker-controlled endpoint despite encryption. Keep certificate and hostname verification enabled and fix trust configuration.

### Q22. You said a preview feature exists in Java 21. Is it final?

Not necessarily. Virtual threads, records patterns, and pattern matching for switch are final in Java 21; structured concurrency and scoped values were preview features in Java 21. State feature status precisely.

## Answer-quality checklist

For every response, answer: **what it is → why it works → what it guarantees → what it does not guarantee → trade-offs → failure mode → how to verify in production**.
