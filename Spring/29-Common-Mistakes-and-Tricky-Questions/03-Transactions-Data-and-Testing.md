# Advanced Questions — Transactions, Data, and Testing

## Q&A

### Q1. Why does transaction propagation matter?
It defines whether a method joins, creates, or avoids an existing transaction. The choice affects atomicity, locks, rollback, and connection use.

### Q2. What is the risk of REQUIRES_NEW?
It suspends an outer transaction and starts another where supported. The inner transaction may commit even if the outer transaction later rolls back, and it may require additional connection capacity.

### Q3. What does NESTED propagation mean?
Where supported, it can use a savepoint within an existing transaction. Support depends on the transaction manager/resource; do not assume it behaves identically across databases.

### Q4. What is transaction synchronization?
It lets participating code react to transaction lifecycle phases. It is useful for coordination but is not a substitute for durable messaging.

### Q5. What is a rollback-only transaction?
A transaction can be marked so commit ultimately fails or rolls back, even if an inner exception was caught. Diagnose where rollback-only was set.

### Q6. Why is a transaction manager important?
It coordinates transactions for a resource model. Choosing the wrong manager can mean the expected resource is not covered by the annotated boundary.

### Q7. Can two databases participate in one transaction?
It depends on the transaction infrastructure and coordination model. Two ordinary local transaction managers do not become atomic simply because one method has @Transactional.

### Q8. Why keep transactions short?
Long transactions hold connections and locks, increase contention, and make failure recovery harder. Avoid remote calls and lengthy computation inside database transaction boundaries.

### Q9. What is transaction isolation's practical limitation?
Database implementations differ, and isolation labels do not fully predict every engine's behavior. Test concurrency assumptions against the production database.

### Q10. What is optimistic locking useful for?
It detects conflicting updates without holding long-lived pessimistic locks throughout application processing. Handle conflicts as a domain decision rather than blindly retrying.

### Q11. When is pessimistic locking appropriate?
When contention and correctness requirements justify database locks. Keep transactions short and understand lock ordering, timeout, and deadlock behavior.

### Q12. Why should business validation and database constraints coexist?
Application validation improves feedback, but concurrent operations can race. The database constraint is the final integrity boundary.

### Q13. What does JdbcTemplate simplify?
It manages repetitive JDBC resource handling and translates many JDBC exceptions into Spring's data-access exception hierarchy. It does not optimize SQL automatically.

### Q14. Why use named SQL parameters?
They improve readability and reduce positional binding mistakes, especially when queries have many parameters. They do not make string-concatenated SQL safe.

### Q15. How should SQL injection be prevented?
Use parameter binding for values, whitelist identifiers that cannot be bound, and avoid concatenating untrusted input into SQL fragments.

### Q16. What is the difference between Page and Slice?
Page commonly exposes total-count metadata; Slice indicates whether more results exist without necessarily computing the total. Use the least expensive contract that meets product needs.

### Q17. Why might a count query be slow?
Complex joins and predicates may make total counting expensive. Inspect generated SQL and consider Slice or a customized count query when total totals are not essential.

### Q18. What is a repository fragment?
A custom implementation component composed with a Spring Data repository to handle behavior that derived methods or @Query do not express cleanly.

### Q19. What is auditing in Spring Data?
It records metadata such as creator/modifier and timestamps through supported auditing infrastructure. Configure auditor identity and time semantics deliberately.

### Q20. What causes LazyInitializationException?
A lazy association is accessed after its persistence context/session is no longer available. Fix the data-fetch boundary or map required data within the intended transaction; do not blindly enable open-session-in-view to mask design issues.

### Q21. What is Open EntityManager in View's trade-off?
It can allow lazy loading during response rendering, but may trigger hidden database queries after the service transaction and obscure query costs. Decide explicitly whether it fits the architecture.

### Q22. Why should entities usually not be API DTOs?
Persistence mappings and API contracts evolve for different reasons. Direct entity serialization can expose internal fields, cause recursion, and trigger lazy loads.

### Q23. How do you test transaction rollback?
Invoke the Spring-managed service boundary, force a failure after a write, and inspect database state after the transaction outcome. Ensure the test's own transaction does not hide the result.

### Q24. Why can test-level @Transactional confuse rollback tests?
The test framework may wrap the test in a transaction that rolls back at the end, masking the service's own transaction behavior. Explicitly control commit/rollback boundaries in the test.

### Q25. What is the value of Testcontainers?
It lets tests use real infrastructure in disposable containers, increasing fidelity for database dialect, broker, and integration behavior. It costs startup time and CI resources.

### Q26. Why is H2 not a complete substitute for PostgreSQL or Oracle?
Dialect, locking, collation, types, query planning, and constraints differ. Use the production engine for behavior where those differences matter.

### Q27. What does context caching optimize?
Spring's test framework can reuse compatible application contexts across tests, reducing startup cost. Different configurations or explicit dirtiness can force new contexts.

### Q28. What can make a test context dirty?
Tests that mutate singleton state or change context internals may require context eviction. Prefer isolated test design over frequent cache eviction.

### Q29. Why are sleep-based async tests fragile?
They assume scheduling and execution timing, which varies with machine load. Await a future, use a latch, or control the executor with a bounded timeout.

### Q30. What should a meaningful unit test assert?
Observable behavior and important invariants, not every internal call. Verify interactions when they are part of the contract or necessary to prevent side-effect regressions.

### Q31. What is contract testing useful for?
It verifies that a service's API/message contract remains compatible with consumers. It complements unit and integration tests rather than replacing them.

### Q32. Why test negative authorization paths?
A success test proves access is possible; it does not prove forbidden principals are rejected. Security tests should cover anonymous, authenticated-denied, and authorized cases.

### Q33. How do you avoid flaky integration tests?
Control external dependencies, data setup, time, asynchronous completion, and shared state. Use bounded waits and useful diagnostics.

### Q34. What is a test slice's limitation?
It loads selected infrastructure and may omit configuration that exists in the full application. It is fast and focused but cannot prove full-context startup.

### Q35. What is the role of @SpringBootTest(webEnvironment=RANDOM_PORT)?
It starts a real embedded server on a random port for tests that need actual HTTP transport behavior, at a higher cost than MockMvc.

### Q36. When is WebTestClient useful?
It can test reactive applications and can also be used in supported testing setups for HTTP interactions. Verify setup requirements for the target application type and version.

### Q37. What is a Mockito spy risk in Spring tests?
A spy wraps real behavior and can accidentally invoke methods or side effects when stubbing/verification is careless. Prefer explicit mocks or real collaborators where appropriate.

### Q38. Why might @MockitoBean not compile in an older project?
`@MockitoBean` is available from Spring Framework 6.2, so projects on earlier Framework/Boot baselines may need the older Boot `@MockBean` API. Spring Boot 3.4 deprecated `@MockBean`/`@SpyBean` for removal, and Boot 4 removed that support in favor of `@MockitoBean`/`@MockitoSpyBean`. Use the annotation supported by the project's managed dependency line and follow the migration guide when upgrading.

### Q39. What is the danger of mocking repositories in every test?
It can hide query, mapping, transaction, and database behavior. Keep unit tests, but include targeted repository integration tests.

### Q40. How should test data be managed?
Use deterministic builders/fixtures, isolate records per test, and avoid relying on execution order or shared mutable fixtures.

### Q41. What should a test do when a dependency is unavailable?
Verify the designed timeout/fallback/error contract using a controlled failure, rather than making the suite depend on a real outage.

### Q42. Why is code coverage not sufficient?
Coverage shows executed code, not whether assertions detect incorrect behavior. Mutation resistance, edge cases, and meaningful invariants matter.

### Q43. What is a useful test pyramid?
Many fast unit tests, focused slice tests, and a smaller set of realistic integration/end-to-end tests. The exact mix follows architecture and risk.

### Q44. How should migration tests be handled?
Run migrations from a known baseline against the production database engine and representative schema/data sizes. Test rollback or forward recovery strategy explicitly.

### Q45. How do you choose between a mock and a fake?
A mock verifies interactions; a fake provides a lightweight working implementation. Choose the simplest test double that proves the behavior without hiding relevant integration semantics.
