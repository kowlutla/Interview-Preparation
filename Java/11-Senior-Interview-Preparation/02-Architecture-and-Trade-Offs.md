# Java Architecture and Trade-Offs

This guide connects core Java decisions to backend service design. The aim is not to force one “best” technology, but to make the constraints and failure behavior explicit.

## 1. State and ownership

### Q1. Immutable objects versus synchronized mutable objects

Immutable objects reduce coordination and simplify sharing but may require copies for updates. Synchronized mutable objects can be efficient for controlled state transitions but require a clear locking policy and can contend.

Choose based on mutation frequency, object size, ownership, and correctness requirements. Prefer immutability for values and explicit synchronization for truly shared mutable invariants.

### Q2. In-memory map versus database index

An in-memory map offers low-latency local access but consumes process memory, is generally local to one instance, and may lose state on restart. A database index provides durable/shared lookup but adds network and storage latency.

Choose based on durability, scale, consistency, and operational ownership. If a map is used as a cache, define invalidation, bounds, and stale-read semantics.

### Q3. ArrayList versus LinkedList

ArrayList offers fast indexed access, locality, and amortized append. LinkedList has cheap local node insertion/removal but expensive index traversal and higher object overhead.

Prefer ArrayList for most list workloads; choose LinkedList only when its deque/list operation pattern justifies the costs. Benchmark realistic access patterns.

### Q4. HashMap versus TreeMap

HashMap suits exact key lookup with expected constant-time operations; TreeMap provides sorted order, range views, and navigation at logarithmic cost.

Use TreeMap when ordering is a requirement, not merely because sorted output seems convenient. For one final sort, sorting a list may be cheaper.

## 2. Concurrency and execution

### Q5. Platform threads versus virtual threads

Platform threads map more directly to OS threads and remain appropriate for many CPU-bound tasks and existing thread-pool designs. Virtual threads are designed for high-concurrency blocking tasks and became final in Java 21.

Neither choice removes downstream limits. Bound connections, remote calls, and queued work, then measure throughput and tail latency.

### Q6. Synchronous code versus CompletableFuture

Synchronous code is often easier to trace and debug. CompletableFuture can overlap independent operations and compose completion without blocking at every stage, but adds scheduling, cancellation, and error-propagation complexity.

Use async composition when the workload benefits and the surrounding framework supports it. Do not call get/join immediately and then claim the design is non-blocking.

### Q7. Locking versus atomic variables

Atomics are suitable for single-variable operations such as counters and CAS state transitions. Locks are usually clearer for multi-field invariants or condition-based coordination.

Do not implement a multi-step business transaction as several independent atomic calls unless the algorithm proves it safe. Choose the simplest correct mechanism.

### Q8. Bounded versus unbounded queues

Unbounded queues smooth temporary bursts but can hide sustained overload and retain unbounded tasks. Bounded queues expose saturation and force an explicit block/reject/drop/persist policy.

Choose capacity from latency and memory budgets. Monitor queue wait time and rejection, not just queue size.

## 3. API and compatibility

### Q9. Record versus class

A record fits a transparent fixed component model and generated value semantics. An ordinary class fits hidden state, mutable lifecycle, controlled identity, and arbitrary inheritance/implementation needs.

Use a record when its semantics match the domain, not simply because it reduces boilerplate.

### Q10. Checked versus unchecked exceptions

Checked exceptions make some failure conditions explicit in signatures but can leak implementation details through layers. Unchecked exceptions simplify signatures but require clear documentation and boundary handling.

Choose based on caller recovery responsibility and consistency across the API. Preserve causes and distinguish absence from operational failure.

### Q11. Reflection versus compile-time generation

Reflection supports dynamic discovery but can add runtime cost, access constraints, and refactoring fragility. Compile-time annotation processing can validate and generate code earlier but adds build complexity.

Choose based on dynamic requirements, startup budget, modular access, and maintenance. Keep reflective code isolated and test discovery behavior.

### Q12. Native serialization versus schema-based formats

Native Java serialization preserves object graph semantics but has compatibility and security concerns, especially for untrusted data. Schema-based formats make wire contracts more explicit and often support language interoperability.

For new service protocols, prefer a deliberate versioned format. Never deserialize untrusted native Java data without strict filtering and a clear threat model.

## 4. Reliability and operations

### Q13. Retry versus fail fast

Retry can recover transient failures but can amplify overload and duplicate side effects. Fail-fast behavior protects resources and returns a clear error but may expose transient failures to callers.

Use bounded retries, exponential backoff, jitter, deadlines, idempotency, and retry budgets. Classify permanent failures separately.

### Q14. Cache versus source of truth

A cache can reduce latency and load but introduces staleness, invalidation, eviction, and memory constraints. The source of truth remains authoritative.

Define TTL, consistency, invalidation, and fallback behavior. A cache failure should not silently corrupt business state.

### Q15. Logging versus metrics versus tracing

Logs capture discrete events and context; metrics quantify aggregate behavior; traces show a request's path across components. They complement one another.

Use structured logs, low-cardinality metrics, and trace context. Avoid high-cardinality metric labels and sensitive payload logging.

### Q16. GC tuning versus allocation reduction

GC tuning changes runtime trade-offs; allocation reduction addresses the source of pressure. Neither is automatically the right fix.

Profile allocation, live set, pauses, and CPU. Fix unbounded retention and algorithmic waste before tuning flags without evidence.

## Decision framework

For any design choice, write down: semantics, workload, capacity limits, failure modes, consistency needs, version baseline, observability, and rollback plan. The right answer is the one that meets the requirements with the least accidental complexity.
