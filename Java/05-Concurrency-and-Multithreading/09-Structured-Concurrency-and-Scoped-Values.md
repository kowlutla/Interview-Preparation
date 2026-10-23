# Structured Concurrency and Scoped Values

**Target level:** Advanced → Senior  
**Version status:** Both were preview APIs in Java 21. This chapter describes their Java 21 status and concepts; preview APIs evolved in later releases, so use documentation for the exact runtime you target.

## 1. Structured concurrency

### Q1. What problem does structured concurrency address? *(Advanced)*

Structured concurrency organizes related concurrent tasks into a lexical/dynamic scope with a parent operation responsible for their lifetime. It aims to make task ownership, cancellation, failure propagation, and observability easier to reason about than detached tasks scattered across a service.

The key idea is that child work should have a clear owner and lifecycle. It does not automatically make operations transactional or make remote side effects reversible.

### Q2. What was the status of Structured Concurrency in Java 21? *(Intermediate)*

Structured Concurrency was a preview feature in Java 21 under JEP 453. It was not a permanent Java 21 API and required the appropriate preview configuration.

Preview APIs can change across releases. Pin the JDK version, isolate preview usage, and check the current JEP/API before relying on a specific class or method shape.

### Q3. How does structured concurrency improve error handling? *(Advanced)*

It provides a scope in which child task failures and cancellation can be coordinated by the parent operation. Instead of losing exceptions in detached tasks, the parent can define how sibling tasks are cancelled or how results are combined.

The exact policy depends on the task scope and API version. Define whether one failure cancels siblings, whether partial results are acceptable, and how cleanup handles already-completed remote side effects.

### Q4. When is structured concurrency useful in a backend request? *(Senior)*

A request may need to call several independent services, such as profile and preferences. Structured task ownership can make it easier to cancel outstanding child tasks when the request fails or times out.

Bound downstream concurrency and propagate deadlines. Structured concurrency organizes tasks; it does not create additional service capacity or eliminate idempotency requirements.

### Q5. How is structured concurrency different from CompletableFuture? *(Advanced)*

CompletableFuture provides completion-stage composition and can represent detached asynchronous graphs. Structured concurrency emphasizes lexical/dynamic task lifetime and parent-child relationships. The two approaches have different APIs and operational models and can coexist.

Choose based on framework support, cancellation/error semantics, and task ownership. Avoid forcing a structured scope into a library whose contract is already CompletionStage-based without a clear benefit.

## 2. Scoped values

### Q6. What problem do Scoped Values aim to solve? *(Advanced)*

Scoped Values provide a mechanism for sharing immutable context across a bounded dynamic call scope, such as request metadata, without relying on mutable ThreadLocal state. They can make context ownership and lifetime more explicit.

They do not automatically propagate context through every arbitrary executor or external asynchronous boundary. Understand the API and propagation rules of the exact release.

### Q7. What was the status of Scoped Values in Java 21? *(Intermediate)*

Scoped Values were a preview feature in Java 21 under JEP 446, not a final Java 21 API. The API evolved in later releases.

Do not describe Java 21 preview syntax as a permanent Java SE contract. Isolate preview usage and compile/run with matching release flags.

### Q8. How do Scoped Values compare with ThreadLocal? *(Advanced)*

ThreadLocal associates mutable values with a thread and can be retained by long-lived pooled threads if cleanup is missed. Scoped Values are designed around immutable bindings with bounded dynamic scope, making some context propagation patterns more explicit.

They are not interchangeable for every use case. Verify the release-specific API, lifecycle, and behavior across thread/task boundaries before migration.

### Q9. Can Scoped Values replace every ThreadLocal? *(Advanced)*

No. ThreadLocal supports mutable per-thread state and established APIs; Scoped Values target a different model of scoped immutable bindings. Existing libraries may require ThreadLocal or explicit context propagation.

Evaluate the semantics and lifecycle rather than replacing mechanically. Request context, tracing, security identity, and mutable caches may have different requirements.

## 3. Adoption and operations

### Q10. How should preview concurrency APIs be adopted? *(Senior)*

Record the exact JDK version, preview flags, API dependencies, and rollback strategy. Isolate preview usage behind a small internal boundary and run compatibility tests when upgrading to the next JDK.

Avoid exposing preview types in public library APIs unless consumers explicitly accept the version coupling. Track the JEP history and current release documentation.

### Q11. Do structured concurrency and scoped values eliminate cancellation bugs? *(Senior)*

No. They can improve task ownership and context lifetime, but cancellation remains cooperative and remote side effects may already have occurred. Child tasks can block in operations that do not respond promptly to interruption.

Use deadlines, interruptible APIs, cleanup, idempotency, and reconciliation. Test cancellation at every point where work may be in flight.

### Q12. How do these features relate to virtual threads? *(Advanced)*

Virtual threads make blocking thread-per-task execution more scalable. Structured concurrency organizes related tasks, and scoped values provide scoped context. These concepts can complement each other, but their feature status and exact APIs depend on the JDK release.

Do not assume that a virtual thread automatically provides structured task lifetime or context propagation. Design each concern explicitly.

## Official references

1. [JEP 453 — Structured Concurrency (Preview in Java 21)](https://openjdk.org/jeps/453)
2. [JEP 446 — Scoped Values (Preview in Java 21)](https://openjdk.org/jeps/446)
3. [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
4. [JDK 21 Documentation](https://docs.oracle.com/en/java/javase/21/)

## Chapter checklist

- [ ] Explain task ownership and cancellation goals of structured concurrency.
- [ ] Distinguish preview status from final API guarantees.
- [ ] Compare Scoped Values with ThreadLocal without claiming equivalence.
- [ ] Design deadlines, cancellation, and side-effect recovery explicitly.
