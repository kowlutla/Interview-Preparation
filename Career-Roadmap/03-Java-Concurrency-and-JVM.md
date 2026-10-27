# Java Concurrency and JVM

> Numbered learning checklist. Use the existing handbook in this repository for detailed explanations and interview Q&A.

### 8. Java Concurrency and Multithreading
- [ ] Thread Lifecycle and Thread API
- [ ] Runnable, Callable, Future, and FutureTask
- [ ] Thread Safety and Shared Mutable State
- [ ] Race Conditions and Atomicity
- [ ] Java Memory Model
  - [ ] Visibility and Ordering
  - [ ] Happens-Before Relationships
  - [ ] volatile Semantics
  - [ ] Safe Publication
  - [ ] Final-Field Semantics
- [ ] synchronized and Intrinsic Locks
- [ ] ReentrantLock, ReadWriteLock, StampedLock
- [ ] wait(), notify(), and notifyAll()
- [ ] Thread Interruption and Cancellation
- [ ] Atomic Variables and CAS
- [ ] LongAdder and Contention
- [ ] Concurrent Collections
- [ ] Executor Framework
  - [ ] ThreadPoolExecutor
  - [ ] Queue Selection and Rejection Policies
  - [ ] Core/Max Pool Sizing
  - [ ] Workload Isolation and Bulkheads
  - [ ] ScheduledExecutorService
- [ ] ForkJoinPool and Work Stealing
- [ ] CompletableFuture
  - [ ] Composition and Exception Handling
  - [ ] Async Execution and Executor Selection
  - [ ] Timeouts and Cancellation
- [ ] Virtual Threads
  - [ ] Blocking I/O Workloads
  - [ ] Thread-Per-Request Model
  - [ ] Pinning and Native Calls
  - [ ] Thread-Local Considerations
  - [ ] Resource Limits and Backpressure
- [ ] Locks, Conditions, Semaphores, and Latches
- [ ] CyclicBarrier, Phaser, and Exchanger
- [ ] Deadlock Detection and Avoidance
- [ ] Lock-Free and Wait-Free Concepts
- [ ] Producer–Consumer and Bounded Queues
- [ ] Thread Confinement and Immutable Objects
- [ ] Structured Concurrency and Scoped Values
- [ ] Concurrency Testing and Stress Testing
- [ ] Thread Dumps and Deadlock Diagnostics

### 9. JVM, Garbage Collection, and Performance
- [ ] JVM Architecture
- [ ] Class Loaders and Class Loader Leaks
- [ ] Runtime Data Areas
  - [ ] Heap and Generational Layout
  - [ ] Java Stacks and Native Method Stacks
  - [ ] Metaspace
  - [ ] Program Counter and Code Cache
- [ ] Bytecode and JIT Compilation
- [ ] Interpreted vs Compiled Execution
- [ ] Escape Analysis and Scalar Replacement
- [ ] Deoptimization and Inlining
- [ ] Garbage Collection Fundamentals
  - [ ] Reachability and Object Lifecycles
  - [ ] Young and Old Generation Concepts
  - [ ] Stop-the-World Pauses
  - [ ] G1 GC
  - [ ] ZGC and Other Collector Overview
  - [ ] GC Logs and Tuning
- [ ] Memory Leaks and Retained Objects
- [ ] OutOfMemoryError Types
- [ ] StackOverflowError
- [ ] Native Memory and Direct Buffers
- [ ] JVM Flags and Container Awareness
- [ ] JFR, JMC, jcmd, jstack, jmap, and jstat
- [ ] Heap Dumps and Thread Dumps
- [ ] CPU Profiling and Allocation Profiling
- [ ] JMH and Benchmark Design
- [ ] Latency, Throughput, Warm-Up, and Contention
- [ ] Production JVM Troubleshooting

## Phase 3 — Build, Testing, Code Quality, and Development Workflow
