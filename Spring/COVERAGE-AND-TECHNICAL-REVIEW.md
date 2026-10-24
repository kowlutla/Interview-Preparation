# Coverage and Technical Review

**Review snapshot:** 2026-10-24  
**Repository branch:** `main`  
**Purpose:** Track measurable coverage separately from technical certification. This file must not be read as a claim that every example has been compiled or every answer independently verified.

## Coverage inventory

| Requirement | Current inventory | Status |
|---|---:|---|
| Numbered Q&A headings | 610 counted across 25 core chapters (270), advanced ecosystem (20), and tricky-question files (320) | Count target and first-pass title/depth screens met; exhaustive claim-by-claim technical certification is not asserted |
| Coding exercises | 85 numbered exercises | Count target met; runnable implementation depth varies |
| Production scenarios | 75 numbered scenarios | Count target met; review each for investigation evidence and trade-offs |
| Mock interviews | 5 interview files | Count target met; answers are model discussion prompts, not transcripts |
| Topic categories | Parts A–Y plus a dedicated advanced ecosystem chapter; exercises/scenarios/mock interviews/tricky questions/references added | Initial breadth present; depth review remains |
| Official documentation | Dedicated references plus per-topic links | Sampled core official URLs were opened successfully on 2026-10-24; a repository-wide external-URL audit remains incomplete |
| Code validation | Eleven detailed reference-solution files added, including core/operations and testing/cloud patterns, alongside REST API, transactions, security, cache/async, Testcontainers/client, configuration/validation, outbox/idempotency, Actuator/metrics, and scheduled work/resilience | Not yet compiled as a single runnable sample application |

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

- [x] Run normalized-title duplicate screens in two batches (core chapters 1–15; core chapters 16–25 plus the tricky-question collection): no exact repeated titles were found within either batch. A global cross-batch title comparison and full answer-level semantic deduplication were not completed; overlap still requires judgment.
- [x] Run an answer-depth screen across the 270 core and 20 advanced ecosystem answers: each has a substantive explanatory answer of at least 25 words. The 320 tricky questions intentionally use concise rapid-fire answers; selected shallow explanations were expanded. This screen measures structure/depth signals, not independent proof of every technical claim.
- [x] Recheck the dated compatibility snapshot against the current official Framework, Boot, and Spring Cloud pages; correct the Boot 4.1.2 requirements paragraph and standardize Q&A heading levels in the advanced ecosystem chapter.
- [x] Check all internal Markdown targets across the 68 Spring Markdown files and the root/Spring index links; no missing local targets were found. Scan local fragment links; none were present. Targeted official external-link checks were completed for the core reference index, but this was not a crawler of every external URL in every chapter.
- [x] Manually review the nine detailed reference-solution files for obvious API-contract, exception-handling, security, and example-consistency issues. Compilation/runtime execution remain intentionally out of scope; no example is claimed as compiled or tested.
- [x] Confirm all 85 exercises have a reference approach, common-mistake note, and validation points; detailed runnable implementation depth varies and only nine patterns have dedicated solution files.
- [x] Review all 75 incident and architecture/design scenarios against their relevant diagnostic/design structure, trade-offs, prevention/recovery guidance, and follow-up prompts.
- [x] Confirm all five mock interview tracks exist and follow increasing scope/difficulty; they are authored practice prompts, not transcripts of real interviews.
- [x] Update handbook indexes and review status to reflect completed structural/first-pass audits without claiming exhaustive technical certification.

## Structural checks completed on 2026-10-24

- Checked the Q&A heading sequence in all 25 core chapters, the advanced ecosystem chapter, and all seven tricky-question files. Numbering is continuous within each file, with no gaps or duplicate question numbers found.
- Checked exercise numbering across all four exercise files: 1–85 is continuous with no gaps or overlaps.
- Checked scenario numbering across all seven scenario files: 1–75 is continuous with no gaps or overlaps.
- Reviewed the incident-scenario structure across scenarios 1–20 and 31–75; standardized the Investigation/Resolution/Prevention labels in scenarios 56–75 and added explicit prevention guidance to scenarios 31, 39, 42, and 44. Scenarios 21–30 are architecture/design prompts and are assessed against requirements, reasoning, trade-offs, validation, and follow-up instead of incident-only labels.
- Checked relative/local Markdown links across all 68 Markdown files under `Spring/` against the repository tree. No missing local file targets were found. This check does not validate fragment anchors.
- Opened the main official reference links for Framework, Boot, Batch, Modulith, GraphQL, Session, Integration, Authorization Server, Cloud Contract, Cloud compatibility, Boot system requirements/migration, Framework release list/7.1 prerelease, and REST clients. The sampled official pages loaded; a full availability check of every external URL is still pending.
- Checked for the specific formatting defect where a question is repeated verbatim as the first non-empty line immediately below its heading. No instances were found in those 33 Q&A files.
- These are structural checks only. They do not prove semantic uniqueness, answer correctness, link validity, or runnable-code correctness; those remain separate review tasks.


## Official-reference URL spot-check — 2026-10-24

Opened the following official documentation targets from the core chapter indexes during this pass; each resolved to a relevant documentation page at the time checked:

- Framework overview, container basics, dependency injection, annotation configuration, bean scopes, bean lifecycle/nature, AOP overview and proxying, MVC overview, Bean Validation, exception handling/REST error responses, transaction management, JDBC, testing, REST clients, WebFlux, caching, and task scheduling.
- Boot reference, auto-configuration, externalized configuration, profiles, SpringApplication startup, Maven executable packaging, testing, and Actuator metrics.
- Spring Security servlet architecture and JWT resource-server guidance; Spring Data JPA reference and query methods; Spring for Apache Kafka; and the Spring Cloud project compatibility matrix.
- Corrected the Java Flight Recorder link in the performance chapter to Oracle's JDK 21 `jfr` command documentation after the previous `jfapi/flight-recorder.html` target could not be confirmed.

This confirms only the sampled targets above. It is **not** a full crawler or exhaustive reachability/content audit of every external URL in all 68 Markdown files. URLs that could not be checked should be marked unconfirmed rather than automatically called broken. The official pages may show newer documentation versions than a project's chosen dependency baseline, so links should be paired with the version-specific docs for upgrade-sensitive claims.

## Focused answer-depth improvements completed on 2026-10-24

Expanded selected explanations covering auto-configuration diagnostics, external configuration, startup readiness, API sorting, streaming responses, database-specific tests, password hashing, and reactive concurrency. This is a targeted sample, not a full depth audit of all 610 Q&As. The tricky-question collection intentionally uses short rapid-fire prompts, and its index directs readers to explain assumptions and consult the core chapters.

## Focused manual code review completed on 2026-10-24

Reviewed all nine detailed reference-solution files for obvious API-contract and example-consistency issues without compiling or executing code. Corrections made:

- Separated the REST request and response records into explicitly separate source files.
- Added a malformed-JSON handler so invalid request bodies do not fall through to the generic 500 handler.
- Clarified that JWT issuer configuration alone does not guarantee audience validation; an explicit audience validator is needed when the API requires one.
- Changed the metrics example to derive its outcome tag from the operation result instead of accepting a potentially misleading caller-supplied value.

This was a focused manual review, not a guarantee that every snippet in the handbook is error-free. Compilation and runtime testing remain intentionally out of scope for this review.

## Recent audit corrections

- The original inventory was 589 (269 core-category Q&As plus 320 tricky questions). Added 20 advanced ecosystem Q&As and one migration question about preview-versus-stable releases. The current breakdown is 270 core-category Q&As, 20 advanced ecosystem Q&As, and 320 tricky questions: 610 total.
- During the title-level semantic review, refined overlapping REST chapter questions (content negotiation versus media-type/payload failures; safe upload design versus large-upload streaming), sharpened the Jakarta namespace migration question, and replaced repeated tricky-question definitions for HTTP 401/403 and reactive backpressure with diagnostic/advanced variants; replaced repeated startup-runner, DispatcherServlet, filter/interceptor, Page/Slice, condition-report, relaxed-binding, composed-annotation, context-hierarchy, outbox, and consumer-idempotency prompts with troubleshooting, failure-window, or design questions. Some concept overlap between core chapters and the deliberately tricky-question collection is intentional reinforcement, not automatically a duplicate.
- Added 20 advanced ecosystem Q&As covering Spring Batch, Modulith, GraphQL, WebSocket, AOT/native images, virtual threads, OAuth2 roles, Spring Session, Integration, Contract, and retry design. Current inventory: 610 numbered Q&A headings.
- Removed a duplicated status sentence from the main README.
- Refreshed the compatibility snapshot against official documentation checked on 2026-10-24; the latest stable listings are Framework 7.0.10 and Boot 4.1.2, with Framework 7.1.0-RC1 and Boot 4.2.0-RC1 as prereleases. This remains a dated snapshot and must be rechecked before a real upgrade.

## Official sources checked for version-sensitive guidance

- [Spring Framework versions and support](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)
- [Spring Framework 7.0 release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes)
- [Spring Boot 4.0 system requirements](https://docs.spring.io/spring-boot/4.0/system-requirements.html)
- [Spring Boot current system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Cloud compatibility](https://spring.io/projects/spring-cloud)
- [Spring REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)

## Continued audit update — 2026-10-24

### Version snapshot and transaction semantics

- Rechecked the compatibility page and migration Q&A against official release listings on 2026-10-24: Framework 7.0.10 is stable, Framework 7.1.0-RC1 is a prerelease, and Boot 4.1.2 is stable and aligned with Framework 7.0.10 or above. Generic system-requirements pages may lag patch-level releases, so verify the exact Boot BOM and resolved Framework version. These are dated observations, not permanent support guarantees.
- Reframed the repeated REQUIRED-versus-REQUIRES_NEW question into the more operationally useful connection-pool starvation risk from nested independent transactions. The propagation basics remain in the earlier question.
- Checked all seven tricky-question files for exact repeated question titles; none were found within that collection. This does not establish semantic uniqueness across the whole handbook, and deliberate reinforcement of core topics still needs judgment-based review.

### Repository-wide local Markdown target check

Checked relative Markdown links across all 68 Markdown files under Spring/, plus the repository root README and the Spring main README, against the current GitHub tree. No missing local Markdown targets were found in this pass. Counts from the index checks were 2/2 root README links and 52/52 Spring README links. This check validates local targets only; it does not verify external URL reachability, anchor correctness on every target page, or technical correctness of linked documentation.

### Further version and exercise review — 2026-10-24

- Clarified that Boot 4 removes the older Boot `@MockBean`/`@SpyBean` support in favor of Spring Framework `@MockitoBean`/`@MockitoSpyBean`; the 3.4 deprecation and older-baseline caveat are now explicit in the Testing Q&A and tricky-question collection. Source: the official [Spring Boot 4.0 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide).
- Labeled the `RestClient` timeout exercise as requiring Framework 6.1+ (normally Boot 3.2+) and directed older Boot 3.x projects to a compatible client API.
- Split the REST reference solution's request and response record declarations into separate filename-labeled source examples so a reader does not paste two public top-level records into one Java file.
- Updated the compatibility snapshot to Spring Cloud 2025.1.4, released October 23, 2026, based on Boot 4.0.9 and also compatible with Boot 4.1.2. Source: the official [Spring Cloud 2025.1.4 announcement](https://spring.io/blog/2026/10/23/spring-cloud-2025-1-4-has-been-released/).

These are manual documentation/API-version corrections only. No compilation or runtime validation was performed.

### Exercise navigation and scenario review — 2026-10-24

- Added two detailed reference-solution files for 25 exercises that previously had only short reference approaches. The handbook now has 11 detailed reference-solution files and 83 exercise-to-solution links, plus three links to core chapters. The added files cover pagination, conditional beans, profiles, lifecycle cleanup, reactive composition, startup diagnostics, Problem Details, multi-data-source boundaries, rate limiting, uploads, startup runners, query-count tests, auto-configuration tests, reactive context, memory investigations, upgrade smoke suites, JSON serialization, context caching, startup conditions, pool saturation, gateway routes, load balancing, and BOM upgrades. This improves navigation but does not claim that every exercise now has a fully runnable implementation.
- Read through the complete 1–75 production-scenario set. Each scenario or design prompt has a diagnostic/reasoning path, a resolution or design direction, trade-offs/prevention or failure modes, and a follow-up question. No new factual correction was confirmed in this pass. The scenarios are still narrative guidance, not substitutes for environment-specific runbooks or executable incident drills.
- Refined the Spring Boot testing guidance to state that `@MockBean`/`@SpyBean` were deprecated for removal in Boot 3.4 and removed in Boot 4, with `@MockitoBean`/`@MockitoSpyBean` as the modern path. The exact annotation API remains tied to the selected Framework/Boot line.
- The local Markdown target audit remains clean after the added exercise links; external URL reachability and anchor correctness are still a separate audit item.

### Transaction and Boot 4 compatibility edge cases — 2026-10-24

- Added the Spring Framework 6.2+ global rollback-policy nuance: traditional defaults roll back on unchecked exceptions and Errors, while transaction management can opt into `ALL_EXCEPTIONS`; explicit rules and caught exceptions still matter. This is reflected in the core transaction Q&A and tricky-question collection.
- Added Boot 4 modular-starter/test-module and test-annotation migration guidance to the compatibility page. The reference guide should remain version-specific: Boot 4 removed `@MockBean`/`@SpyBean` support, and tests may need explicit MVC/client test auto-configuration rather than relying on older `@SpringBootTest` behavior.
- Rechecked the updated exercise files after adding navigation links to detailed solutions and related chapters: 62 local Markdown links were found in the four exercise files, and every target resolves. This includes 58 detailed-solution cross-links, three chapter links, and one pre-existing local link.

### Exercise-format consistency pass — 2026-10-24

Reviewed the 85 exercise entries structurally. All 85 now include a reference approach/solution section, a common-mistake note, and validation points. Added 54 missing common-mistake notes across exercises 26–85 and aligned the exercise index wording with the agreed review scope (manual technical/API-version review; no compilation or execution). The four exercise files now have complete structural coverage for those fields. This does not yet mean every reference approach is a complete implementation: depth and solution sufficiency remain an explicit review item, and the nine detailed solution documents cover selected patterns rather than all 85 exercises.

### Official-reference URL audit — targeted pass — 2026-10-24

Opened the core links in the official references index. Found and corrected two stale/misleading destinations: the Boot API Javadoc link now uses the current `/spring-boot/api/java/` path, and the Spring Cloud reference now points to the current Spring Cloud Release documentation instead of the legacy `current` URL that still advertised release train 2022.0.5 / Boot 3.0.13. The Reactor reference now points directly to its current guide page rather than a redirecting index URL. Most remaining core links opened successfully in this targeted pass; the Oracle JFR guide URL could not be fetched by the web tool, so its reachability remains unconfirmed rather than classified as broken. This is not yet a crawl of every external URL in every chapter, and versioned docs should still be selected for the exact project baseline.

The stale Spring Cloud legacy-reference URL was also removed from the Spring main README and the Spring Cloud chapter. The README audit status now distinguishes the completed local-target check from the still-pending fragment-anchor and broader external-URL review.

### Local fragment audit — 2026-10-24

Scanned all 68 Spring Markdown files for links containing fragments. No local Markdown fragment links were found, so there are no local heading anchors to resolve in this corpus. The only `#...` fragments found in the indexes were two external `#support` links to the Spring Framework and Spring Boot project pages. The main README keeps the external-anchor and remaining external-URL checks open rather than claiming a complete crawl.

### Q&A numbering order recheck — 2026-10-24

Rechecked the question-number sequence in all 25 core chapter files and all seven tricky-question files. Each uses continuous ascending numbering with no gaps or out-of-order headings. The migration chapter had an out-of-order Q13 inserted before Q9–Q12; the questions have now been reordered to Q1–Q13 before the official-reference section. The advanced ecosystem chapter uses a different heading level (`## Qn`) but its sequence is Q1–Q20. The total remains 610 Q&A headings (270 core + 20 ecosystem + 320 tricky).

### Prerelease API nuance — test bean overrides — 2026-10-24

The official Framework 7.1 prerelease notes add constructor-parameter support for `@MockitoBean`/`@MockitoSpyBean`. The testing chapter and tricky-question answer now distinguish that prerelease capability from stable Framework 7.0 usage, avoiding accidental advice to use a newer API form on a stable baseline.

### Chapter section-order audit — 2026-10-24

A follow-up structural scan found several Q&A batches had been appended after the Official references section even though their question numbering was continuous. Reordered the affected blocks so all Q&A appears before references in these chapters: Spring Boot Fundamentals, Spring MVC/REST, Data Access/Transactions, Spring Data JPA, Spring Security, Testing, Reactive/WebFlux, Actuator/Observability, Caching, Spring Cloud, and Version Evolution/Migration. The section-order scan found no additional affected core chapter or the advanced ecosystem chapter after these corrections. This preserves all questions and their numbering; it is a layout/navigation correction, not a content-count increase.

### Patch-level requirements evidence correction — 2026-10-24

The generic Boot system-requirements page currently renders Boot 4.1.0 (Java 17 minimum, Java 26 maximum, Framework 7.0.8+) even though the release snapshot lists Boot 4.1.2 and Framework 7.0.10 as latest stable patches. The version-compatibility guide now separates those facts and no longer claims the generic page itself verifies 4.1.2's exact maximum JDK. The exact versioned 4.1.2 requirements page could not be independently confirmed through this review, so the guide explicitly directs readers to verify the target BOM/requirements before setting build/runtime bounds.


### Final editorial and consistency pass — 2026-10-24

- Ran normalized-title duplicate screens in two batches (core chapters 1–15; core chapters 16–25 plus the tricky-question collection); no exact repeated titles were found within either batch. A global cross-batch title comparison and full answer-level semantic deduplication were not completed. Cross-topic reinforcement remains intentional where it serves learning.
- Screened the 270 core and 20 advanced ecosystem answers for explanatory depth; all passed a 25-word minimum in the answer body. A follow-up screen after expanding 29 concise core answers found that all 290 core and advanced ecosystem answers now pass a 35-word minimum. The 320 tricky questions intentionally remain concise rapid-fire checks. Word-count screens are depth signals, not proof of correctness.
- Standardized the advanced ecosystem chapter's 20 question headings to the same `### Qn.` level used elsewhere.
- Rechecked current official docs on 2026-10-24: Framework 7.0.10 and Boot 4.1.2 are listed as stable; Boot 4.1.2's current system-requirements page states Java 17 minimum, Java 27 maximum, Framework 7.0.10+, Maven 3.6.3+, Gradle 8.14+/9.x, and GraalVM 25+ for native images. Corrected the compatibility guide to avoid conflating those values with older Boot 4.1.0 page content.
- Reopened the core official references index: Framework, Boot, Security, Data JPA, Kafka, Batch, Modulith, GraphQL, Session, Integration, Authorization Server, Cloud Contract, Jakarta Validation, Reactor, Testcontainers, and Java SE pages loaded. Micrometer's old docs root redirects to the current Micrometer site. The web checker could not fetch the Spring Cloud release-train reference URL directly, so the authoritative Spring Cloud project compatibility table remains the fallback. This remains a targeted official-reference check, not an all-URLs crawl of every chapter.
- No code compilation, runtime execution, or test execution was performed, per the review constraint. Do not interpret completion of the editorial/structural pass as certification that all examples are runnable or that every claim is defect-free.


### Additional answer-depth pass — 2026-10-24

- Expanded 29 concise core answers across configuration/profiles, Boot startup and shutdown, transactions, JPA auditing/native SQL/performance diagnosis, security/tenant isolation/session handling, testing, REST client pooling/credential propagation, reactive error recovery, Actuator endpoint security/health/SLO alerts, caching metrics, scheduled time zones, distributed tracing/timeouts/resilience, and deprecated API migration.
- Re-screened the 290 core and advanced ecosystem answers (270 + 20) after these edits: every answer body contains at least 35 words. The 320 tricky questions remain intentionally concise; their brevity is part of the rapid-fire format and should not be confused with the depth standard for core chapters.
- Reconfirmed exact title uniqueness within two review batches: core chapters 1–15 (179 Q&As) and core chapters 16–25 plus the advanced ecosystem chapter (111 Q&As). A global cross-batch comparison and full semantic answer-by-answer deduplication are still not claimed; concept reinforcement across chapters needs judgment.
- No compilation, runtime execution, or test execution was performed.


### Cross-topic overlap review — 2026-10-24

- Compared question headings from core chapters 1–10, 11–15, and 16–25/advanced ecosystem against the 320 rapid-fire questions using token-overlap screening and manual review of the strongest candidates. This screening is a triage aid, not a semantic proof: generic words such as “difference,” “useful,” and “role” produce false positives, while differently phrased duplicates may not share many words.
- Refined overlapping tricky questions into diagnostic or scenario-oriented prompts, including JDK proxy injection failures, distinguishing component scanning from auto-configuration conditions, transaction propagation/rollback boundaries, REQUIRES_NEW commit behavior, savepoint support for NESTED, optimistic/pessimistic locking outcomes, test-slice blind spots, Testcontainers' engine-specific value, WebClient pool exhaustion, trace bottleneck diagnosis, and Boot/Cloud upgrade failures.
- Rechecked the seven tricky-question files after edits: all 320 questions remain numbered continuously and no exact repeated titles exist within the tricky-question collection.
- Intentional reinforcement remains useful for interview practice, but a global human semantic review of every answer pair is still not certified complete. Do not interpret a title similarity scan as exhaustive deduplication.
- Reviewed all five mock-interview files for progression and follow-up prompts. They progress from fundamentals through mid-level implementation, senior troubleshooting, microservices, and product-company scenarios; no compilation or execution was involved.


### Expanded exercise solutions — 2026-10-24

- Added 10-Core-and-Operations-Patterns.md and 11-Testing-and-Cloud-Patterns.md with detailed implementation patterns for 25 exercises that previously relied on concise reference approaches. The examples call out assumptions, version-dependent APIs, negative/failure-path tests, and operational trade-offs.
- Added links from those exercises to the new solution files. The exercise bank now contains 11 detailed solution files and 83 exercise-to-solution links. Two exercises retain chapter-level references rather than a dedicated detailed solution because their existing approach is already small and directly scoped; review the exact index count before describing coverage.
- These solutions remain illustrative and were not compiled or executed. Several code fragments rely on domain classes and dependencies defined by their exercises; they are not a single ready-to-run application.
