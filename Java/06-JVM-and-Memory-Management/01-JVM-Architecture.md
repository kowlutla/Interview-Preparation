# JVM Architecture

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

The Java Virtual Machine executes class files and provides runtime services such as class loading, memory management, bytecode interpretation/JIT compilation, thread coordination, and diagnostics. The JVM specification defines the abstract machine; HotSpot is one implementation with specific runtime structures and optimizations.

## 1. JVM components

### Q1. What is the JVM? *(Beginner)*

The JVM is the abstract machine that loads and executes Java class files and defines runtime behavior for types, memory areas, instructions, and threads. Implementations may interpret bytecode, compile it to native machine code, and optimize execution dynamically.

Java's platform portability comes from the class-file and JVM contracts, not from every runtime having identical performance or internal architecture. Native libraries, OS behavior, architecture, and JDK versions still matter.

### Q2. What are the major JVM runtime areas? *(Intermediate)*

The specification describes the heap, method area, runtime constant pools, Java stacks, PC registers, and native method stacks. The heap is shared among JVM threads; each Java thread has its own stack and PC register. Implementations may map these abstract areas to different physical structures.

Do not assume the specification's method area maps exactly to one HotSpot memory region. Metaspace is a HotSpot implementation detail for class metadata, not a universal JVM-spec term.

### Q3. What is the difference between heap and stack? *(Beginner)*

The heap is the runtime area from which objects and arrays are allocated and is shared among threads. A Java stack contains frames for method invocations, including local variables and partial results, and is associated with one thread.

This conceptual distinction is useful, but physical storage can be optimized by the JIT. Escape analysis and scalar replacement may eliminate some allocations, and values can live in registers. Diagnose memory issues with runtime evidence rather than assuming source variables map one-to-one to memory locations.

### Q4. What is a stack frame? *(Intermediate)*

A frame is created for a method invocation and holds local variables, an operand stack, and bookkeeping needed for dynamic linking, return, and exception handling according to the JVM model. Frames are pushed and popped as methods are invoked and return.

Deep recursion or large stack use can lead to StackOverflowError. Thread stack size is implementation/configuration dependent; increasing it may postpone overflow but does not fix unbounded recursion.

### Q5. What is the program counter (PC) register? *(Advanced)*

Each JVM thread has a PC register that tracks the current JVM instruction for the thread's execution, subject to the specification's rules. It supports resumption and execution state. Native method execution has special treatment in the abstract model.

The PC register is a conceptual runtime area, not a public application API. It is mainly useful for understanding JVM architecture and debugging metadata.

### Q6. What is the method area? *(Advanced)*

The method area is a JVM-specification runtime area for per-class structures such as runtime constant pools, field and method data, and code for methods/constructors. Its physical implementation is not mandated.

In HotSpot, class metadata is commonly allocated in native-memory Metaspace, while other related data may live elsewhere. Class unloading and class-loader reachability can affect metadata retention.

## 2. Execution engine

### Q7. What is the difference between interpreter and JIT compiler? *(Intermediate)*

An interpreter executes bytecode instructions without compiling an entire hot method to native code first. A JIT compiler compiles frequently executed code to native machine instructions and can optimize based on runtime profiling.

HotSpot commonly uses tiered compilation, combining interpreter execution and multiple compilation tiers. Startup performance and steady-state throughput can differ; a benchmark must account for warmup.

### Q8. What is tiered compilation? *(Advanced)*

Tiered compilation uses different execution and compilation levels, gathering profiling information before applying more expensive optimizations to hot code. It balances startup and profiling cost against peak performance.

Compilation thresholds and exact tiers are implementation and configuration details. Use JFR, compiler logs, and JVM diagnostics when investigating a specific runtime rather than assuming one universal threshold.

### Q9. What is deoptimization? *(Advanced)*

Deoptimization lets the JVM abandon optimized machine code when assumptions used by the optimization no longer hold, returning execution to a less optimized representation while preserving program semantics. Optimizations may rely on profiling assumptions such as observed types or call targets.

Deoptimization is a normal runtime mechanism, not necessarily a bug. Excessive deoptimization or unstable profiles can affect performance, so inspect compiler diagnostics and workload behavior before tuning.

### Q10. What is escape analysis? *(Advanced)*

Escape analysis determines whether an object reference can escape a method or thread context. The JIT may use this information to eliminate allocations through scalar replacement or to optimize synchronization for non-escaping objects.

It is an optimization, not a language guarantee. Do not write code that depends on allocation elimination for correctness, and measure allocation behavior with the target JDK.

### Q11. What is intrinsic substitution? *(Advanced)*

A JVM may recognize certain library methods and replace them with specialized machine instructions or optimized compiler sequences. These intrinsics can make standard APIs faster while preserving their semantics.

Intrinsics are implementation details. Use documented APIs and benchmark rather than assuming a particular method always maps to one hardware instruction.

## 3. Class loading and linking

### Q12. What are loading, linking, and initialization? *(Intermediate)*

Loading locates a binary representation and creates the runtime Class object. Linking includes verification, preparation, and resolution. Initialization executes class initialization code such as static field initializers and static blocks.

These phases are related but distinct and may occur lazily. Errors can arise at different phases, producing different diagnostic symptoms.

### Q13. What is the parent-delegation model? *(Advanced)*

The conventional class-loader delegation model asks a parent loader to load a class before the child tries to define it. This avoids duplicate definitions of core classes and supports shared platform types. Custom class loaders may use different delegation policies for legitimate isolation or plugin use cases.

Class identity includes the defining loader. Two same-named classes from different defining loaders are different types, which can cause ClassCastException or linkage problems.

### Q14. What is the runtime constant pool? *(Advanced)*

Each class or interface has a runtime constant pool derived from its class-file constant_pool table. It contains symbolic references and constants used by the runtime, including references that can be resolved to runtime entities.

Resolution can occur at different times according to JVM rules and implementation strategy. A linkage error may therefore appear on first use rather than during initial class loading.

## 4. Diagnostics and performance

### Q15. What is the difference between Java heap and native memory? *(Intermediate)*

Heap memory is managed for Java objects and arrays. Native memory includes memory used outside the Java heap, such as class metadata, thread stacks, direct buffers, JIT code cache, and native libraries. A process can run out of native memory even when heap usage appears healthy.

Monitor process RSS, heap, Metaspace, direct buffers, thread count, and code cache. Heap dashboards alone cannot explain every container OOM or native allocation failure.

### Q16. What is a safepoint? *(Advanced)*

A safepoint is a JVM coordination point where threads reach states suitable for certain global runtime operations, such as some garbage collection or deoptimization work. The exact implementation mechanics vary by JVM.

Long time-to-safepoint or long pauses can affect latency. Use supported diagnostics and JFR to determine whether the problem is application work, runtime coordination, or a specific JVM operation.

### Q17. What is the code cache? *(Advanced)*

The code cache stores generated native code for compiled methods and related runtime stubs in HotSpot. If it fills, compilation behavior and performance can be affected.

Inspect compiler/code-cache logs and runtime metrics before changing its size. Code cache is separate from Java heap, so heap free space does not prove that the code cache has capacity.

### Q18. How do you approach a JVM performance investigation? *(Senior)*

Start with a measurable symptom and baseline: throughput, latency percentiles, CPU, allocation rate, GC pauses, thread states, queue depth, and dependency latency. Use JFR, GC logs, thread dumps, heap histograms, and compiler diagnostics as appropriate.

Change one variable at a time and compare under a reproducible workload. Avoid blind flags copied from unrelated incidents; tuning should follow evidence about the actual bottleneck.

## Official references

1. [Java Virtual Machine Specification, Java SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)
2. [JVMS Chapter 2 — The Structure of the JVM](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html)
3. [JVMS Chapter 5 — Loading, Linking, and Initializing](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)
4. [JDK 21 Flight Recorder documentation](https://docs.oracle.com/en/java/javase/21/jfapi/index.html)
5. [Java SE 21 tools reference](https://docs.oracle.com/en/java/javase/21/docs/specs/man/index.html)

## Chapter checklist

- [ ] Distinguish JVM-spec concepts from HotSpot implementation details.
- [ ] Explain runtime areas, frames, class loading, and execution tiers.
- [ ] Understand JIT, deoptimization, escape analysis, and code cache.
- [ ] Diagnose heap and native-memory issues with evidence.
