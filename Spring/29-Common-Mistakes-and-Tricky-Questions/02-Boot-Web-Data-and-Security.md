# Common Mistakes and Tricky Questions — Boot, Web, Data, Security

## Q&A

### Q1. Does @SpringBootApplication scan the whole classpath?
No. Component scanning starts from the configuration class's package by default, not every package in every dependency.

### Q2. Is every Spring Boot application a web application?
No. Boot supports non-web, servlet, and reactive application types depending on dependencies and configuration.

### Q3. Are starters runtime features?
A starter is primarily a dependency descriptor. The included libraries and Boot auto-configuration provide the actual runtime behavior.

### Q4. Does a starter guarantee a secure configuration?
No. Defaults are not a complete security review. Define authentication, authorization, secrets, endpoint exposure, and operational policies.

### Q5. Is an executable JAR the same as a native image?
No. Executable JAR packaging includes a launcher and dependencies; a native image is a different compiled deployment artifact with distinct constraints.

### Q6. What does ApplicationReadyEvent mean?
It signals a lifecycle point after application startup has completed sufficiently for the ready event. It does not prove every business dependency is healthy unless readiness checks model them.

### Q7. Why should startup runners not be used as durable batch-processing infrastructure?
A runner is a startup hook, not a job repository, checkpointing system, or durable scheduler. If work must resume after a crash, expose progress, or run independently of application readiness, use a controlled job mechanism and persist progress. Keep startup hooks bounded and make repeated invocation safe.

### Q8. Can a runner safely perform an unbounded import?
Usually not. It can block readiness and make deployment unpredictable. Use a controlled job for large or resumable work.

### Q9. What should you inspect when a mapped Spring MVC endpoint unexpectedly returns 404?
Check the effective request path and context path, active profiles and conditional controllers, request mapping conditions (method, path, headers, consumes/produces), servlet registration, and any gateway or filter rewriting. Confirm the handler mapping from logs or diagnostics before changing controller code. A 404 can mean no handler matched, or that the application deliberately hides a resource.

### Q10. How can filter and interceptor ordering cause a request-processing bug?
Filters run in the servlet filter chain before MVC handler execution, while `HandlerInterceptor` callbacks surround handler processing after mapping. A security rule placed at the wrong layer or an interceptor that assumes a handler already exists can produce inconsistent behavior. Trace the actual chain and test both matched and unmatched paths; do not treat MVC interceptors as a replacement for security filters.

### Q11. Does @RestController always return JSON?
It returns response bodies through message converters; the selected representation depends on supported converters and content negotiation, not the annotation alone.

### Q12. Why might an API return an unexpected 401 or 403 despite a valid login?
A valid login does not guarantee the request satisfies the API's authorization rules. Inspect the selected `SecurityFilterChain`, request matcher/order, granted authorities, CSRF requirements, authentication entry point, and access-denied handler. A valid identity with insufficient permission typically gets an access-denied response, but custom handlers and filter failures can change the observable status. Test the exact endpoint and principal rather than diagnosing from the status code alone.

### Q13. Does CORS protect an API from unauthorized non-browser clients?
No. CORS is a browser-enforced cross-origin policy, not authentication or authorization.

### Q14. Should CSRF always be disabled for REST APIs?
No. Evaluate how credentials are sent. Cookie-based browser authentication can remain vulnerable even if endpoints are called REST APIs.

### Q15. Is a decoded JWT trustworthy?
No. Signature and claim validation must succeed against a configured trust model.

### Q16. Does a valid JWT authorize every resource?
No. Authentication is separate from authorization, and tenant/resource access must still be enforced.

### Q17. Is @Transactional a distributed transaction?
No. It typically coordinates resources supported by a selected transaction manager. Separate database and broker writes need an explicit reliability design.

### Q18. Do all exceptions trigger rollback?
No. Default rollback rules commonly differ for unchecked versus checked exceptions; explicit rules and caught exceptions affect outcomes.

### Q19. Does readOnly=true forbid writes at the database?
Not universally. It is metadata/hint whose enforcement depends on the transaction manager, provider, driver, and database.

### Q20. Can a transaction span an @Async task automatically?
Generally not in the ordinary thread-bound transaction model. Establish a transaction within the async operation if needed.

### Q21. Does JpaRepository always include PagingAndSortingRepository in every version?
Do not assume inheritance across versions. Check the Spring Data interface hierarchy for the exact release; interfaces evolved.

### Q22. Does Page<T> always require a count query?
It may require a count query to calculate total pages/elements, though query execution can sometimes optimize or avoid it. Use Slice when total-count metadata is unnecessary.

### Q23. Does eager fetching always fix N+1?
No. It can create large joins and excess data. Use targeted fetch plans or projections and inspect actual SQL.

### Q24. Does an in-memory database prove production SQL correctness?
No. SQL dialect, isolation, locking, collation, constraints, and query plans differ. Use integration tests with the production engine for critical behavior.

### Q25. Is WebClient always faster than RestClient?
No. Performance depends on workload, I/O model, connection management, and usage. Reactive programming has complexity and benefits only when the system uses it appropriately.

### Q26. Does Mono mean work runs on another thread?
No. A reactive publisher does not imply automatic parallelism or a thread switch.

### Q27. Is retry safe for GET and unsafe for POST in every case?
HTTP semantics provide guidance, but actual idempotency depends on implementation and side effects. A nominal GET can be implemented badly; a POST can be safely retried with durable idempotency keys.

### Q28. Does a circuit breaker replace a timeout?
No. Timeouts bound individual waits; a circuit breaker can prevent repeated calls after failures are observed. They solve related but different problems.

### Q29. Does @Cacheable cache method results across all instances?
Only if the configured cache provider is shared. A local cache is typically per process; distributed behavior depends on the cache implementation.

### Q30. Does @CacheEvict run if the transaction rolls back?
Ordering and transaction-aware cache configuration matter. Design invalidation around successful state changes and test rollback semantics for the actual provider.

### Q31. Does @Async make a method durable?
No. Tasks may be lost on process termination unless they are persisted in a durable queue or job system.

### Q32. Does @Scheduled run only once in a cluster?
No. Each application instance can schedule the task unless cluster coordination is implemented.

### Q33. Are Spring application events durable?
No, ordinary application events are in-process events. Use a broker or durable outbox for cross-process reliability.

### Q34. Does a Kafka acknowledgment guarantee an external side effect happened once?
No. Consumer delivery and external system effects need an idempotency/transaction strategy.

### Q35. Does Actuator health guarantee good user experience?
No. Health is only as meaningful as its indicators and probe semantics. Track user-visible latency and errors separately.

### Q36. Should every dependency be included in liveness?
Usually not. A shared dependency outage can cause restart storms if liveness fails for every instance. Use readiness for traffic eligibility and liveness for process progress.

### Q37. Does a metric label accept arbitrary request values?
Technically a library may accept them, but high-cardinality labels can overwhelm monitoring systems. Use bounded categories.

### Q38. Does tracing replace logging?
No. Traces show cross-service paths, logs provide event details, and metrics show aggregate trends.

### Q39. Does Spring Cloud compatibility follow Boot automatically?
No. Use the official release-train compatibility information and matching dependency management.

### Q40. Does an API gateway eliminate service security?
No. It is an edge control point; services still need appropriate resource authorization and trust validation.

### Q41. Does a readiness failure restart the process?
Usually readiness removes an instance from traffic; liveness/startup probe behavior may trigger restart depending on orchestrator configuration.

### Q42. Is a bigger thread pool always better?
No. More threads can increase context switching, contention, and downstream overload. Measure queueing and resource limits.

### Q43. Is a bigger connection pool always better?
No. The database has finite capacity, and excess concurrency can worsen contention and latency.

### Q44. Can heap size be set equal to the container memory limit?
That leaves no room for native memory and other process overhead. Size the heap with measured container headroom.

### Q45. Does increasing cache TTL always improve performance?
It may improve hit ratio but increases staleness risk. Define freshness requirements and invalidation guarantees.

### Q46. Does graceful shutdown guarantee every task completes?
No. Shutdown has deadlines and process failures can be abrupt. Persist or make work safely repeatable.

### Q47. Should a production error response include a stack trace?
No. Keep stack traces in protected diagnostics, not client responses.

### Q48. Is a 500 response always a server bug?
Not necessarily; it indicates an unexpected server-side failure under the API contract, which may arise from configuration or dependency outages. Diagnose the root cause.

### Q49. Is a successful unit test enough for a framework integration?
No. Add context, slice, or integration tests where behavior depends on proxies, filters, serialization, transactions, or external infrastructure.

### Q50. What is the safest response to version-specific uncertainty?
Check the exact versioned official documentation and migration notes, create a minimal reproduction, and state the uncertainty instead of presenting an assumption as fact.
