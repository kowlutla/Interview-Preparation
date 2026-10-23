# Executors and Thread Pools

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Executors separate task submission from execution policy. Production thread-pool design requires an explicit queue capacity, worker limit, rejection strategy, shutdown lifecycle, task error handling, and metrics. A thread pool is a resource-control mechanism, not merely a convenience wrapper around Thread.

## 1. Executor fundamentals

### Q1. What is Executor? *(Beginner)*

Executor abstracts task execution through execute(Runnable), separating the caller from the mechanism that runs the task. Implementations may run the task immediately, on another thread, or through a scheduling policy.

Do not assume execute always means asynchronous execution. The contract is intentionally minimal; use ExecutorService or a more specific interface when lifecycle, results, or scheduling are needed.

### Q2. What does ExecutorService add? *(Intermediate)*

ExecutorService adds task submission methods that return Future results, bulk invocation, and lifecycle operations such as shutdown, shutdownNow, and awaitTermination. It manages task execution but does not automatically make tasks safe or bound all resources.

Always define shutdown behavior and observe task failures. A submitted task that fails may store its exception in a Future rather than propagating it to the submitting thread.

### Q3. What is the difference between execute and submit? *(Intermediate)*

execute accepts a Runnable and returns no result. submit accepts Runnable or Callable and returns a Future; task exceptions are captured and can be observed through Future.get.

A common operational bug is using submit and ignoring the returned Future, thereby hiding task failures. Use a result-handling policy that logs, reports, retries safely, or propagates failures as appropriate.

### Q4. What is a thread pool? *(Beginner)*

A thread pool reuses worker threads to execute tasks, reducing repeated thread creation and centralizing concurrency limits. Pool sizing and queue behavior determine how much work can run and how much can wait.

A pool with too few workers can underutilize resources; too many can cause context switching and resource pressure. An unbounded queue can hide overload by accumulating tasks.

### Q5. What are the main ThreadPoolExecutor parameters? *(Advanced)*

Core pool size, maximum pool size, keep-alive time, work queue, thread factory, and rejection handler determine execution behavior. The order in which tasks are queued and workers created matters: after core workers, tasks are generally queued before the pool grows toward maximum, depending on queue capacity and state.

Understand the combination rather than tuning one number in isolation. A large maximum pool does not help if an unbounded queue accepts all tasks first.

### Q6. Why can Executors.newFixedThreadPool still be dangerous? *(Advanced)*

The standard factory creates a fixed number of worker threads backed by an unbounded shared queue. If producers outpace consumers, queued tasks can accumulate without a fixed memory bound.

For production services, consider a ThreadPoolExecutor with a bounded queue and explicit rejection/backpressure policy. Monitor queue depth and task wait time, and define what callers should do when the system is saturated.

### Q7. What is a rejection policy? *(Intermediate)*

When a ThreadPoolExecutor cannot accept a task because of capacity or shutdown, its RejectedExecutionHandler determines the outcome. Built-in policies include aborting with an exception, running the task in the caller, discarding a task, or discarding the oldest queued task.

Choose based on correctness. Silent discard is unsafe for required business work; CallerRunsPolicy can provide backpressure but may block request threads and change latency behavior.

## 2. Lifecycle and futures

### Q8. What is the difference between shutdown and shutdownNow? *(Intermediate)*

shutdown stops accepting new tasks but allows previously submitted tasks to complete. shutdownNow attempts to interrupt active tasks and returns tasks that never commenced execution; it does not guarantee that running tasks stop.

Tasks must respond cooperatively to interruption. A robust shutdown sequence typically calls shutdown, waits for a bounded interval, escalates to shutdownNow, and waits again while preserving interruption status.

### Q9. What is Future? *(Intermediate)*

Future represents a result or completion state that may be available later. get waits for completion and returns the result or throws an exception; cancel requests cancellation; isDone and isCancelled inspect state.

Calling get too early can block the caller. Avoid waiting on futures in a way that exhausts the same executor's workers, which can cause thread-starvation deadlocks.

### Q10. What is the difference between Future.get and isDone? *(Intermediate)*

isDone checks whether the computation has completed, failed, or been cancelled; it does not return the result. get returns the result after waiting if necessary and reports failure through ExecutionException or cancellation through CancellationException.

Polling isDone in a tight loop wastes CPU. Use blocking completion, timed waits, callbacks, or a higher-level asynchronous composition API.

### Q11. What does Future.cancel(true) mean? *(Advanced)*

cancel(true) requests cancellation and may interrupt the executing thread if the task is running. It cannot guarantee that code stops: tasks may ignore interruption or be inside operations that do not respond promptly.

Treat cancellation as cooperative. Ensure partial work is handled correctly, and remember that cancelling a future does not necessarily undo a remote side effect that already committed.

### Q12. How do you observe exceptions from submitted tasks? *(Advanced)*

For submit, call get on the returned Future or use an explicit completion/error-reporting mechanism. ExecutionException wraps the task's exception as its cause. For execute, uncaught task exceptions may reach the worker's uncaught-exception path depending on executor behavior.

Do not discard Futures for important work. Instrument task failures and define whether each task is retryable, reportable, or fatal to the enclosing workflow.

## 3. Pool sizing and overload

### Q13. How do you size a thread pool? *(Senior)*

Start from task type, CPU count, blocking ratio, service time, throughput target, downstream limits, and latency objective. CPU-bound work often benefits from a pool near available processor count; blocking workloads may need more concurrency, but only if the downstream systems can handle it.

Use measurements and load tests. Oversized pools can amplify database connection contention, memory use, and context switching. A pool should reflect the constrained resource, not merely the number of cores.

### Q14. Why should thread pools have bounded queues? *(Senior)*

Bounded queues make overload explicit and limit the number of pending tasks and their retained object graphs. They support backpressure or rejection instead of allowing latency and memory use to grow without bound.

Queue capacity should align with acceptable wait time and memory budget. Monitor queue depth, enqueue-to-start delay, rejection count, active workers, and task duration.

### Q15. What is CallerRunsPolicy and when can it be harmful? *(Advanced)*

CallerRunsPolicy executes the rejected task on the submitting thread unless the executor is shut down under its defined behavior. This can slow producers and provide backpressure.

It can also unexpectedly run expensive work on an HTTP request thread, event loop, or other latency-sensitive thread. Use it only when caller-thread execution is safe and included in the system's latency and thread-affinity model.

### Q16. What is thread-starvation deadlock? *(Advanced)*

Thread-starvation deadlock can occur when all workers in a bounded executor block waiting for tasks that are queued to the same executor. The needed tasks cannot start because no worker is free.

Avoid blocking nested submissions to the same limited pool. Use asynchronous composition, separate executors for distinct blocking domains, or redesign dependencies so workers do not wait for work only they can execute.

## 4. Production scenarios

### Q17. How should a service shut down its executor? *(Senior)*

Stop accepting new work, initiate shutdown, wait for a bounded grace period, request interruption if necessary, and wait again for termination. Preserve the interrupt flag if the shutdown thread itself is interrupted.

Define what happens to queued and in-flight work. For durable business jobs, a process-local executor is not a substitute for a persistent queue or recoverable job system.

### Q18. Why can a task queue be full even when CPU usage is low? *(Senior)*

Tasks may be blocked on I/O, waiting for database connections, sleeping on locks, or stalled on a downstream dependency. Low CPU does not imply available task-processing capacity.

Inspect thread dumps, task wait time, connection pools, downstream latency, and blocked states. Increase thread count only after confirming that the constrained resource can support the additional concurrency.

### Q19. How should you handle long-running tasks? *(Senior)*

Separate long-running or blocking tasks from latency-sensitive short tasks when they have different resource profiles. Use bounded queues, explicit cancellation, timeouts, and observability. Avoid letting long tasks occupy every worker needed for short requests.

If tasks must survive process restarts, use durable scheduling or messaging rather than relying on in-memory executor state.

### Q20. What metrics should be collected for a thread pool? *(Senior)*

Track active worker count, pool size, queue size/capacity, completed tasks, rejected tasks, task execution duration, queue wait duration, failures, and shutdown state. Correlate with CPU, memory, downstream latency, and request-level SLOs.

Metrics should reveal saturation before the service fails. Queue depth alone is insufficient because the same depth can mean very different things at different arrival and completion rates.

## Official references

1. [Java SE 21 API — ExecutorService](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ExecutorService.html)
2. [Java SE 21 API — ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
3. [Java SE 21 API — Future](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Future.html)
4. [Java SE 21 API — RejectedExecutionHandler](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/RejectedExecutionHandler.html)

## Chapter checklist

- [ ] Distinguish execute, submit, Future, and task failure handling.
- [ ] Design bounded queues and rejection policies intentionally.
- [ ] Size pools from workload and downstream constraints.
- [ ] Implement graceful shutdown and cooperative cancellation.
- [ ] Monitor saturation, wait time, rejection, and task failures.
