# Queue, Deque and PriorityQueue

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Queues model work awaiting processing. Different implementations provide different ordering, capacity, blocking, concurrency, and null semantics. Selecting the right queue is particularly important in thread pools, messaging pipelines, schedulers, and producer-consumer systems.

## 1. Queue contracts

### Q1. What is the difference between offer and add? *(Intermediate)*

Both attempt to insert an element. add generally throws an exception when insertion fails; offer returns false when a capacity restriction prevents insertion. For unbounded queues, offer normally succeeds unless another failure occurs.

For bounded queues, offer is useful when the caller needs to handle backpressure without an exception. Choose methods according to the queue's capacity and failure contract.

### Q2. What is the difference between poll and remove? *(Intermediate)*

poll retrieves and removes the head, returning null if the queue is empty. remove retrieves and removes the head but throws NoSuchElementException when empty. Similar pairs exist for element/peek.

Use peek/poll when emptiness is a normal condition. Be aware that queues such as ConcurrentLinkedQueue reject null, which makes null a useful sentinel for “no element” in those APIs.

### Q3. What is a Deque? *(Beginner)*

Deque is a double-ended queue supporting insertion, removal, and inspection at both the front and back. It can implement FIFO queue behavior or LIFO stack behavior. ArrayDeque is a common general-purpose implementation and often preferable to legacy Stack.

ArrayDeque does not permit null elements and is not thread-safe. If multiple threads share a deque, use a concurrent deque or coordinate access.

### Q4. How does PriorityQueue order its elements? *(Intermediate)*

PriorityQueue exposes the least element at its head according to natural ordering or a Comparator. It is heap-based, so insertion and head removal are logarithmic in typical implementations, while peek is constant time.

Its iterator does not guarantee sorted traversal. To obtain all elements in priority order, repeatedly poll from a copy or use another explicit sorting strategy. Do not assume printing a PriorityQueue displays sorted order.

### Q5. What is the difference between PriorityQueue and TreeSet? *(Intermediate)*

PriorityQueue permits duplicate elements and efficiently exposes only the head priority; its iteration order is unspecified. TreeSet maintains unique elements in sorted order and supports sorted traversal and navigation.

Use a priority queue for scheduling the next task. Use a sorted set when uniqueness and ordered iteration/range operations are required.

### Q6. What is a bounded queue, and why is it useful? *(Advanced)*

A bounded queue limits the number of queued elements, making overload visible and preventing unbounded memory growth. When full, the producer must block, reject, drop, or otherwise apply an explicit backpressure policy depending on the chosen API.

An unbounded queue can hide overload until memory or latency becomes unacceptable. Capacity should be chosen from throughput, service time, memory, and latency objectives rather than an arbitrary large number.

## 2. Blocking and concurrent queues

### Q7. What is BlockingQueue? *(Intermediate)*

BlockingQueue supports operations that may wait for the queue to become non-empty or have capacity. It is designed for producer-consumer coordination and provides memory-consistency guarantees for handoff between threads.

Implementations include ArrayBlockingQueue, LinkedBlockingQueue, PriorityBlockingQueue, and SynchronousQueue, each with different capacity and ordering characteristics. Choose based on throughput, memory, and blocking requirements.

### Q8. What is the difference between ArrayBlockingQueue and LinkedBlockingQueue? *(Advanced)*

ArrayBlockingQueue is bounded and array-backed, with a fixed capacity. LinkedBlockingQueue can be configured with a capacity and uses linked nodes; its default constructor has a very large practical bound, so it can behave like an unbounded queue for many applications.

ArrayBlockingQueue has predictable storage and one primary lock strategy; LinkedBlockingQueue may support different producer/consumer concurrency characteristics but allocates nodes. Benchmark with realistic contention and ensure queue capacity reflects overload policy.

### Q9. What is SynchronousQueue? *(Advanced)*

SynchronousQueue has no internal capacity: each insertion must rendezvous with a corresponding removal. It is useful for direct handoff designs, such as certain executor configurations, where tasks should be handed to workers rather than accumulated in a queue.

Because it cannot buffer tasks, producers may block or submission may trigger worker creation/rejection depending on the surrounding executor policy. Understand the full executor configuration, not only the queue class.

### Q10. What is the difference between blocking and non-blocking queue methods? *(Intermediate)*

BlockingQueue put/take wait for capacity or an element; timed offer/poll wait up to a specified duration; immediate offer/poll return without waiting for the condition. Interruption can terminate blocking operations.

Choose according to the system's latency and cancellation policy. Preserve interruption when you cannot propagate InterruptedException, and avoid indefinite blocking without a shutdown or timeout strategy.

### Q11. What is a DelayQueue? *(Advanced)*

DelayQueue holds elements implementing Delayed and makes an element available for removal only when its delay has expired. It is useful for delayed tasks or expiration scheduling, but it does not itself provide a complete scheduler, durable persistence, or cluster coordination.

For distributed delayed work, account for process restarts, duplicate execution, clock behavior, and durable scheduling. An in-memory DelayQueue loses queued work when the process terminates.

## 3. Executor and backpressure scenarios

### Q12. Why does an unbounded executor queue risk hiding overload? *(Senior)*

If producers submit faster than workers can complete tasks, an unbounded queue grows and retains tasks and their captured object graphs. Latency rises, memory pressure increases, and the system may fail long after the original overload began.

Use bounded queues and explicit rejection/backpressure behavior where appropriate. Monitor queue depth, task wait time, completion throughput, and rejection counts; queue length alone is not enough to diagnose capacity.

### Q13. How should a producer-consumer pipeline handle overload? *(Senior)*

Choose a deliberate policy: block producers, reject work, drop low-priority events, batch, persist to durable storage, or shed load upstream. The right choice depends on whether work can be lost, whether callers can retry, and the latency budget.

Bound queues to protect memory and expose saturation through metrics. Ensure rejected tasks have a defined caller response and that retries cannot amplify overload into a feedback loop.

### Q14. What are common PriorityQueue pitfalls? *(Advanced)*

A comparator that violates its contract can break priority ordering. Mutating an element's priority fields after insertion does not automatically reposition it. Equal priorities have no guaranteed stable order unless a tie-breaker is included.

Use immutable queue elements or remove/reinsert after priority changes. Add a sequence number as a tie-breaker when deterministic FIFO ordering among equal priorities is required.

## Official references

1. [Java SE 21 API — Queue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Queue.html)
2. [Java SE 21 API — Deque](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Deque.html)
3. [Java SE 21 API — PriorityQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/PriorityQueue.html)
4. [Java SE 21 API — BlockingQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)

## Chapter checklist

- [ ] Choose queue methods based on empty/full behavior.
- [ ] Distinguish deque, priority queue, and sorted set semantics.
- [ ] Understand blocking, timed, and immediate queue operations.
- [ ] Use bounded queues and explicit overload policy.
- [ ] Include tie-breakers and immutable priority state where needed.
