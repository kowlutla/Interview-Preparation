# Coverage and Technical Review

**Review snapshot:** 2026-10-24  
**Repository branch:** `main`  
**Purpose:** Track measurable coverage separately from technical certification. This file must not be read as a claim that every example has been compiled or every answer independently verified.

## Coverage inventory

| Requirement | Current inventory | Status |
|---|---:|---|
| Numbered Q&A headings | 589 counted across category and tricky-question files | Count target met; deduplication/depth audit remains |
| Coding exercises | 85 numbered exercises | Count target met; runnable implementation depth varies |
| Production scenarios | 75 numbered scenarios | Count target met; review each for investigation evidence and trade-offs |
| Mock interviews | 5 interview files | Count target met; answers are model discussion prompts, not transcripts |
| Topic categories | Parts A–Y represented by category files; exercises/scenarios/mock interviews/tricky questions/references added | Initial breadth present; depth review remains |
| Official documentation | Dedicated references plus per-topic links | Link and version audit remains |
| Code validation | Nine detailed reference-solution files added for REST API, transactions, security, cache/async, Testcontainers/client, configuration/validation, outbox/idempotency, Actuator/metrics, and scheduled work/resilience | Not yet compiled as a single runnable sample application |

## High-priority technical review notes

1. **Version-sensitive claims:** Recheck against the exact target patch. Spring Framework 7 retains a Java 17 baseline and adopts Jakarta EE 11; Boot 4 requires Java 17+ and Framework 7.x. Framework 7 deprecates RestTemplate in favor of RestClient. Spring Cloud compatibility is release-train-specific.
2. **Transaction behavior:** Confirm proxy invocation, propagation, rollback rules, transaction manager selection, and resource boundaries. Do not imply a local transaction automatically spans a broker or remote service.
3. **Security behavior:** Confirm filter-chain matching/order, issuer/signature/claim validation, authority mapping, CSRF assumptions, and tenant isolation. Test both allowed and denied requests.
4. **Reactive behavior:** Do not imply Mono/Flux automatically creates threads or that a reactive return type makes blocking work non-blocking.
5. **Caching and async:** Verify proxy boundaries, transaction ordering, cache provider semantics, executor capacity/rejection, and durability assumptions.
6. **Data access:** Use production-engine integration tests for SQL dialect, locking, isolation, and migration behavior that matters.
7. **Operational guidance:** Distinguish readiness from liveness; use bounded timeouts, retries, pools, and queues; ensure graceful shutdown and recovery semantics are tested.
8. **Code examples:** Worked snippets are illustrative and some refer to domain classes/dependencies defined by the exercise. They are not all a self-contained Maven/Gradle project and have not been compiled in this repository.

## Final review checklist

- [ ] Review 589 questions for near-duplicates and shallow variations; retain the strongest formulation where concepts overlap.
- [ ] Expand answers that need deeper internal mechanics, examples, edge cases, and trade-offs.
- [ ] Verify version-specific statements against versioned official docs and record the baseline.
- [ ] Check every internal Markdown link and official external link.
- [ ] Build runnable sample projects for the most important exercises and execute tests against declared dependency versions.
- [ ] Revisit code snippets for imports, API availability, exception behavior, and configuration completeness.
- [ ] Confirm all 85 exercises have a sufficiently complete reference solution, not only a high-level approach.
- [ ] Confirm all 75 scenarios include symptoms, diagnostic evidence, root cause, solution, trade-offs, prevention, and follow-up questions.
- [ ] Review all five mock interviews for natural progression, follow-up depth, and technical correctness.
- [ ] Update the handbook README only when the relevant audit item is genuinely complete.

## Official sources checked for version-sensitive guidance

- [Spring Framework versions and support](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)
- [Spring Framework 7.0 release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes)
- [Spring Boot 4.0 system requirements](https://docs.spring.io/spring-boot/4.0/system-requirements.html)
- [Spring Boot current system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Cloud compatibility](https://spring.io/projects/spring-cloud)
- [Spring REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
