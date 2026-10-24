# Reactive Programming and WebFlux

## Q&A

### Q1. What are Mono and Flux?

In Project Reactor, `Mono<T>` represents an asynchronous sequence with zero or one item, while `Flux<T>` represents a sequence of zero to many items. They describe a publisher pipeline, not necessarily a separate thread or automatic parallel execution.

### Q2. What is backpressure?

Backpressure is a mechanism for coordinating demand between a subscriber and a publisher so the consumer can signal how many items it is ready to process. It helps manage flow but does not magically bound every external resource or make a blocking call safe.

### Q3. How does WebFlux differ from Spring MVC?

Spring MVC is commonly used with a thread-per-request, blocking programming model. WebFlux is designed for non-blocking reactive processing and can use a small event-loop worker set. WebFlux is most beneficial when the whole I/O path is non-blocking and concurrency demands justify the model's complexity.

### Q4. Why are blocking calls dangerous on event-loop threads?

Blocking work prevents an event-loop worker from servicing other tasks, causing latency and throughput degradation. Use reactive clients and reactive data access where appropriate, or isolate unavoidable blocking work on a suitable bounded scheduler. Moving blocking work to another scheduler still consumes threads and must be capacity-planned.

### Q5. How should errors and retries be handled in a reactive pipeline?

Use Reactor operators to map, recover, or retry at a deliberate boundary. Retry only transient and safe-to-repeat operations, bound attempts, and consider the total deadline. An error recovery operator can change the signal into a fallback value, so avoid accidentally turning genuine failures into apparent success.

### Q6. What is cancellation, and why does it matter?

A subscriber can cancel its subscription when it no longer needs data. Well-designed pipelines propagate cancellation to upstream work where supported, releasing resources and avoiding unnecessary processing. External calls and custom publishers must be examined to ensure cancellation really stops or limits work.

### Q7. How is context propagated in reactive applications?

Reactive execution can hop threads, so ordinary ThreadLocal assumptions do not necessarily hold across the pipeline. Reactor Context is the common mechanism for contextual data, and supported integrations can bridge tracing or security context. Verify context propagation behavior for the exact versions and integrations in use.

## Official references

- [Spring WebFlux reference](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Project Reactor reference](https://projectreactor.io/docs/core/release/reference/)
