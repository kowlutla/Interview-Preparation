# Production Scenarios — Resilience, Security, and Operations

## Scenario 11 — Async task fails silently

**Symptoms:** Notifications are missing although the HTTP request succeeded.

**Investigation:** Check executor rejection, exception handling for void-returning async methods, future completion, queue depth, shutdown logs, and downstream failures.

**Resolution:** Surface failures through an async exception handler or observed future, instrument rejections and outcomes, and use durable messaging if delivery must survive process failure.

**Trade-offs/prevention:** @Async is not a durable queue. Test executor saturation and process termination.

**Follow-up:** What is lost if the process exits with tasks only in memory?

## Scenario 12 — Scheduled job executes on every replica

**Symptoms:** Duplicate cleanup, billing, or batch work occurs after scaling.

**Investigation:** Confirm how many instances register the schedule, whether execution is idempotent, and whether a distributed lock or clustered scheduler exists.

**Resolution:** Use shared coordination or a scheduler with cluster semantics, plus durable idempotency.

**Trade-offs/prevention:** Distributed locks need lease expiry and crash recovery. A lock alone does not make side effects exactly once.

**Follow-up:** Why is synchronized insufficient across pods?

## Scenario 13 — Cache returns stale data

**Symptoms:** Read endpoints show old values after a successful update.

**Investigation:** Compare cache keys, eviction path, proxy interception, TTL, concurrent updates, multiple cache layers, and source-of-truth writes from other services.

**Resolution:** Correct invalidation/update semantics and define a freshness contract. For shared state, evaluate a distributed cache or event-driven invalidation.

**Trade-offs/prevention:** Strong consistency can cost latency and throughput. Test concurrent reads/writes and provider outage behavior.

**Follow-up:** What is a cache stampede?

## Scenario 14 — Reactive pipeline blocks

**Symptoms:** WebFlux latency rises sharply during a slow blocking database or SDK call.

**Investigation:** Inspect thread dumps for event-loop workers blocked in synchronous APIs and review the full I/O stack.

**Resolution:** Replace blocking calls with reactive clients where possible; isolate unavoidable blocking work on a bounded scheduler with explicit capacity.

**Trade-offs/prevention:** A scheduler switch does not make a blocking dependency non-blocking. Load-test concurrency and cancellation.

**Follow-up:** Why does returning Mono not guarantee non-blocking execution?

## Scenario 15 — Health check is green during dependency outage

**Symptoms:** Orchestrator routes traffic to an instance that cannot serve core requests.

**Investigation:** Determine whether the dependency is included in readiness, whether check timeouts are bounded, and whether health caching or endpoint exposure changes the result.

**Resolution:** Reflect critical dependency status in readiness where appropriate; keep liveness focused on process progress to avoid restart loops.

**Trade-offs/prevention:** Marking every dependency as readiness-critical can remove all capacity during a shared outage. Define criticality and degraded modes explicitly.

**Follow-up:** Why should liveness and readiness be separate?

## Scenario 16 — Boot upgrade fails at startup

**Symptoms:** Missing class/method, namespace mismatch, or changed auto-configuration behavior.

**Investigation:** Read the migration guide, inspect dependency convergence, Java baseline, removed APIs, Jakarta imports, condition report, and startup cause chain.

**Resolution:** Align the dependency BOM, update incompatible libraries, migrate APIs, and run production-like integration tests.

**Trade-offs/prevention:** Randomly pinning transitive versions may create binary incompatibility. Use a staged rollout and rollback plan.

**Follow-up:** Why should Spring Cloud compatibility be checked separately?

## Scenario 17 — Kafka event processed twice

**Symptoms:** Duplicate notification, payment, or record update.

**Investigation:** Review delivery semantics, acknowledgment timing, retries, consumer restart, rebalance, and whether the external side effect is idempotent.

**Resolution:** Use a durable event ID/deduplication record or idempotent operation; coordinate state and deduplication in one transaction where possible.

**Trade-offs/prevention:** Exactly-once broker processing does not automatically make an external API side effect exactly once.

**Follow-up:** How does the outbox pattern address a database-plus-publish dual write?

## Scenario 18 — Downstream timeout cascades

**Symptoms:** Request threads and connection pools fill while one dependency slows.

**Investigation:** Compare caller deadline, client timeout, retries, circuit-breaker state, thread/connection pools, and fan-out concurrency.

**Resolution:** Set coherent end-to-end budgets, bound retries with jitter, apply circuit breaking or bulkheads where justified, and provide safe fallbacks for noncritical functions.

**Trade-offs/prevention:** Aggressive retries can amplify overload. Load-test failure scenarios, not only healthy paths.

**Follow-up:** How can retries at multiple layers multiply load?

## Scenario 19 — Memory grows continuously

**Symptoms:** Heap after GC trends upward or container memory approaches its limit.

**Investigation:** Compare post-GC heap, allocation rate, heap-dump dominators, cache/queue sizes, thread counts, and native memory. Check retained references, ThreadLocals, listeners, and unbounded collections.

**Resolution:** Fix the retention source and add bounded capacity. Adjust heap/container sizing only after understanding native overhead and allocation behavior.

**Trade-offs/prevention:** A larger heap may delay but not fix a leak. Reproduce with a representative load test.

**Follow-up:** Why can container OOM occur when Java heap is below its configured maximum?

## Scenario 20 — Database update and event publish diverge

**Symptoms:** Database state commits but consumers never see the event, or an event is published for a rolled-back change.

**Investigation:** Map the database commit and broker publish sequence, failure windows, retry behavior, and message durability.

**Resolution:** Use an outbox written in the same local transaction and a retrying publisher, with idempotent consumers.

**Trade-offs/prevention:** Outbox introduces relay lag, cleanup, and duplicate handling. Monitor pending age and publication failures.

**Follow-up:** Why doesn't a plain @Transactional annotation guarantee atomicity across a database and Kafka?
