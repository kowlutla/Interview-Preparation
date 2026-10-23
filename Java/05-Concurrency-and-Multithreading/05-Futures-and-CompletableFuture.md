# Future and CompletableFuture

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Asynchronous APIs help compose work without blocking a thread for every dependency. Future provides a handle to completion; CompletableFuture adds completion stages, composition, error recovery, and explicit execution policies. Correct use requires understanding where tasks run, how exceptions propagate, and how cancellation affects the underlying work.

## 1. Future basics

### Q1. What is Future? *(Beginner)*

Future represents the result of an asynchronous computation that may complete later. It provides get, timed get, cancellation, and completion-state inspection.

Future alone does not provide a rich non-blocking composition API. Calling get blocks the caller until completion or timeout, so using it immediately after submission often removes the benefit of asynchronous execution.

### Q2. What is CompletableFuture? *(Intermediate)*

CompletableFuture implements Future and CompletionStage. It can be completed by a task or explicitly, and stages can be chained with methods such as thenApply, thenCompose, thenCombine, exceptionally, and handle.

It supports asynchronous composition, but not every stage is automatically asynchronous. Understand whether a method runs inline on the completing thread or schedules work on an executor.

### Q3. What is the difference between thenApply and thenCompose? *(Intermediate)*

thenApply transforms a value and wraps the result in a new stage. thenCompose is used when the transformation itself returns a CompletionStage and you want to flatten the nested stage.

    CompletableFuture<User> user = loadUser(id);
    CompletableFuture<Order> order =
        user.thenCompose(u -> loadLatestOrder(u.id()));

Using thenApply with a stage-returning function can produce CompletableFuture<CompletableFuture<T>>, which usually is not the intended composition.

### Q4. What is the difference between thenCombine and thenCompose? *(Advanced)*

thenCompose represents a dependent asynchronous step where the next stage depends on the previous result. thenCombine combines results from two independently progressing stages after both complete.

Use thenCombine for parallel independent dependencies such as loading profile and preferences. Use thenCompose for dependent calls where the second request requires the first result.

### Q5. What is the difference between thenApply and thenApplyAsync? *(Advanced)*

thenApply may execute the function in the thread that completes the previous stage or another thread that invokes completion, depending on the completion timing. thenApplyAsync schedules the function asynchronously, using the default asynchronous facility or an explicitly supplied executor.

Do not assume thenApply always runs on a dedicated worker. Use explicit executors when thread affinity, blocking behavior, or resource isolation matters.

### Q6. Which executor does CompletableFuture use by default? *(Advanced)*

Async methods without an explicit executor generally use ForkJoinPool.commonPool(), subject to API rules and implementation behavior. Non-async methods can run in a thread that completes the prior stage.

The common pool is shared and may be unsuitable for long blocking I/O or workloads requiring strict isolation. Supply a dedicated executor for blocking operations and configure its capacity and shutdown policy.

## 2. Errors and completion

### Q7. How do exceptionally and handle differ? *(Intermediate)*

exceptionally provides a recovery value when the stage completes exceptionally. handle receives either the result or the failure and can transform either case into a new result. whenComplete observes completion while generally preserving the prior result/failure unless the observer itself fails.

Choose the method based on whether you are recovering, transforming, or observing. Do not accidentally turn a failure into a successful null result without documenting that behavior.

### Q8. What does exceptionallyCompose do? *(Advanced)*

exceptionallyCompose supports asynchronous recovery by returning another CompletionStage when the prior stage fails. It avoids manually creating nested stages during recovery.

It was added in Java 12. If targeting Java 8, use a compatible composition pattern. Always document which failures are recoverable and avoid retrying permanent errors.

### Q9. How do exceptions propagate through a CompletableFuture chain? *(Advanced)*

A stage that throws or completes exceptionally causes dependent stages to complete exceptionally unless a stage handles or recovers from the failure. get wraps failures in ExecutionException; join throws CompletionException for exceptional completion.

Inspect the underlying cause and preserve context. A failure can be observed at multiple stages, so avoid duplicate logging at every step; log at the boundary that can make a meaningful decision.

### Q10. What is the difference between get and join? *(Intermediate)*

get declares checked InterruptedException and ExecutionException, and it can be timed. join does not declare checked exceptions and wraps exceptional completion in CompletionException. Both can block if the future has not completed.

Prefer composition over either blocking method inside asynchronous pipelines. Blocking on a future from a worker in the same constrained executor can contribute to starvation deadlocks.

### Q11. How do allOf and anyOf work? *(Intermediate)*

allOf returns a future that completes when all supplied futures complete, but it does not directly return a typed list of their results. anyOf completes when one supplied future completes, with an Object result.

For allOf, collect the individual results after completion and preserve type information explicitly. Decide what happens when one of the futures fails or when the first completed result is a failure.

### Q12. Does cancelling a CompletableFuture cancel the underlying operation? *(Advanced)*

Cancellation marks the future as cancelled/exceptionally completed according to its contract, but it does not universally guarantee that the underlying computation or remote operation stops. CompletableFuture is not necessarily tied to an interruptible task in a way that cancellation can terminate.

Propagate cancellation through the application explicitly where possible. For remote requests, use client cancellation, timeouts, and idempotency rules; a cancelled caller does not prove the server did no work.

## 3. Composition patterns

### Q13. How do you run two independent tasks concurrently and combine results? *(Intermediate)*

Create separate stages for independent operations, then combine them using thenCombine or allOf when a larger group is involved. Avoid blocking one task while waiting for the other if composition can express the dependency.

Independent work can improve latency, but it also increases concurrent load on downstream systems. Use bulkheads, timeouts, and concurrency limits.

### Q14. How do you implement a timeout? *(Advanced)*

CompletableFuture.orTimeout completes the future exceptionally if it has not completed before the timeout; completeOnTimeout supplies a fallback value. These methods were added in Java 9. A timeout on the future does not necessarily stop the underlying operation.

Use transport-level deadlines and cancellation where supported. Avoid interpreting timeout as proof that a remote write failed; the outcome may be unknown and require idempotency or reconciliation.

### Q15. How do you retry an asynchronous operation? *(Senior)*

Model retry as a new asynchronous stage after a retryable failure, with bounded attempts, backoff, jitter, deadline, and cancellation handling. Avoid blocking sleeps inside common-pool workers; use a scheduler or resilience library appropriate to the application.

Retry only idempotent operations or use idempotency keys. Ensure that errors such as validation failures are not retried and that retries do not amplify downstream overload.

### Q16. How do you prevent one slow dependency from blocking unrelated work? *(Senior)*

Use explicit executors or bulkheads for different workload classes, set deadlines, and compose results asynchronously. Do not run blocking I/O on a shared executor intended for short CPU-bound continuations.

Measure queue wait and execution time separately. An asynchronous API does not create infinite capacity; every dependency and executor still has resource limits.

## 4. Production scenarios

### Q17. Why can thenApply cause unexpected thread execution? *(Senior)*

Non-async continuation stages can run on the thread that completes the preceding stage, which may be a request thread, network callback thread, or worker thread. If the continuation blocks or performs heavy work, it can delay unrelated completion processing.

Use thenApplyAsync with an explicit executor when isolation is required. Do not add async variants indiscriminately, because each scheduling boundary has overhead and changes ordering/latency.

### Q18. What are common CompletableFuture anti-patterns? *(Senior)*

Common issues include calling get/join too early, blocking the common pool, ignoring exceptional completion, creating unbounded asynchronous work, assuming cancellation stops remote operations, and logging the same failure at every stage.

Draw the dependency graph and specify execution, timeout, cancellation, and failure behavior. Prefer named helper methods for complex chains rather than a single opaque expression.

### Q19. How do you test asynchronous code deterministically? *(Senior)*

Use controllable futures, latches, test executors, and explicit completion rather than Thread.sleep. Test success, failure, timeout, cancellation, and completion ordering. Keep tests bounded so a regression cannot hang the build indefinitely.

Assert observable contracts rather than relying on a specific worker thread name unless thread affinity is part of the requirement.

### Q20. When is synchronous code preferable? *(Senior)*

Synchronous code can be simpler when operations are naturally sequential, latency is low, or concurrency adds little benefit. Async composition is valuable when independent I/O can overlap or a framework is designed around non-blocking completion.

Avoid adopting asynchronous style solely to look scalable. Compare complexity, cancellation, tracing, resource usage, and measured latency under representative load.

## Official references

1. [Java SE 21 API — CompletableFuture](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
2. [Java SE 21 API — CompletionStage](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletionStage.html)
3. [Java SE 21 API — Future](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Future.html)

## Chapter checklist

- [ ] Distinguish thenApply, thenCompose, thenCombine, and async variants.
- [ ] Handle failures and cancellation deliberately.
- [ ] Avoid blocking the common pool or constrained executor.
- [ ] Treat timeout as an unknown remote outcome when appropriate.
- [ ] Test with controlled completion, not sleeps.

## Worked examples: asynchronous composition

### Example 1. Compose dependent stages instead of blocking between them

~~~java
CompletableFuture<User> user = fetchUserAsync(userId);
CompletableFuture<Account> account =
        user.thenCompose(u -> fetchAccountAsync(u.accountId()));
~~~

thenCompose flattens a dependent asynchronous operation. Using thenApply with a function that returns another CompletableFuture instead produces a nested future. Prefer composition to blocking on each stage, especially when the work shares a constrained executor.

### Example 2. Combine independent calls

~~~java
CompletableFuture<Profile> profile = fetchProfileAsync(id);
CompletableFuture<Preferences> preferences = fetchPreferencesAsync(id);

CompletableFuture<View> view = profile.thenCombine(
        preferences, View::new);
~~~

This models independent operations that are both needed for the result. Production code still needs deadlines, error policy, cancellation behavior, and a clear decision about whether partial results are allowed.

### Example 3. Do not treat exceptionally as a universal retry mechanism

exceptionally can translate a failure into a fallback value, but it does not distinguish transient failures from permanent failures automatically. Retry only when the operation is safe to retry, the failure is plausibly transient, the deadline allows it, and backoff/jitter plus an attempt limit are defined.
