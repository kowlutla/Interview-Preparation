# Testing and Build Tooling for Java

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21 applications; framework examples use JUnit Jupiter 5 and Mockito where stated

Testing is part of Java engineering, not a final cleanup step. Strong tests verify observable behavior, isolate nondeterminism, model dependency boundaries honestly, and run consistently across developer machines and CI. Build tooling should make the JDK, dependency graph, test lifecycle, and supported runtime matrix explicit. Framework versions also have JDK baselines: Mockito 5 requires Java 11+, while Mockito 4 supports Java 8; select dependencies to match the project's declared runtime baseline.

## 1. Test strategy and JUnit 5

### Q1. What is the difference between a unit test, integration test, and end-to-end test? *(Beginner)*

A unit test checks a small unit of behavior with collaborators controlled or kept in-process. An integration test checks that components work together across a meaningful boundary, such as a repository with a real database. An end-to-end test exercises a larger deployed workflow through public interfaces.

The labels are less important than the test's scope, dependencies, runtime, and failure signal. Keep fast deterministic tests numerous, and reserve slower environment-dependent tests for the boundaries they actually verify.

### Q2. What are the main parts of JUnit 5? *(Intermediate)*

JUnit 5 is an umbrella name for JUnit Platform, JUnit Jupiter, and JUnit Vintage. The Platform launches and integrates test engines; Jupiter provides the modern programming and extension model; Vintage enables execution of older JUnit 3/4 tests through the platform when the relevant engine is present.

Do not assume adding one dependency automatically configures every IDE and build plugin. Align the JUnit engine, test launcher, build plugin, and IDE integration versions.

### Q3. What do common JUnit Jupiter lifecycle annotations do? *(Intermediate)*

@Test marks a test method. @BeforeEach and @AfterEach run around each test method; @BeforeAll and @AfterAll run once per test class lifecycle and are commonly static under the default per-method test instance lifecycle. With @TestInstance(PER_CLASS), non-static class lifecycle methods are permitted.

Use setup for shared fixture construction, not for hiding the behavior under test. Keep each test understandable on its own and clean up external resources even when setup or assertions fail.

### Q4. How should parameterized tests be used? *(Intermediate)*

Parameterized tests run the same behavioral contract over multiple inputs using sources such as @ValueSource, @CsvSource, @MethodSource, or enum sources. They are useful for boundaries, conversion rules, validation, and cases where the same invariant must hold across several values.

Choose cases that explain distinct behavior rather than merely inflating the test count. Include invalid values, boundaries, null policy where relevant, and representative combinations.

### Q5. How do assertThrows and assertAll improve tests? *(Intermediate)*

assertThrows verifies that an action throws an expected exception type and returns the exception for further inspection. assertAll executes a group of assertions and reports multiple failures together rather than stopping at the first failed assertion.

Assert the public contract: exception type, stable error code, or relevant safe message where required. Avoid brittle assertions on full stack traces or implementation-specific exception text.

### Q6. What makes a test deterministic and isolated? *(Advanced)*

A deterministic test has a result that depends on controlled inputs and observable behavior rather than wall-clock timing, thread scheduling luck, shared mutable state, or an external service that changes independently. Isolation means one test's setup or cleanup does not silently affect another test.

Inject clocks, random generators, configuration, and dependency interfaces when those inputs affect behavior. Reset or avoid static mutable state, isolate temporary files and ports, and make cleanup reliable on failure paths.

## 2. Mockito and test doubles

### Q7. What is the difference between a mock, stub, fake, and spy? *(Intermediate)*

A stub supplies predetermined responses; a mock is commonly used to verify interactions; a fake is a working but simplified implementation, such as an in-memory repository; and a spy wraps a real object while allowing selected behavior to be observed or stubbed.

These terms overlap in everyday usage, so explain the role the test double plays. Prefer a fake or real lightweight collaborator when it provides a clearer test than extensive interaction mocking.

### Q8. When should a dependency be mocked? *(Advanced)*

Mock a dependency when its behavior must be controlled, it is expensive or nondeterministic, or the test needs to verify an important interaction contract. Avoid mocking simple value objects, collections, or every internal method of the class under test.

Excessive mocks couple tests to implementation details and make harmless refactoring painful. Keep the system under test real and mock at deliberate boundaries such as an external client, clock, or repository when appropriate.

### Q9. What is the difference between when(...).thenReturn(...) and doReturn(...).when(...) in Mockito? *(Advanced)*

when(...).thenReturn(...) is the usual stubbing form. For a spy, evaluating the method call inside when(...) can invoke the real method during stubbing; doReturn(...).when(spy)... avoids that call and can be useful when the real invocation has side effects or throws.

Prefer not to spy on complex objects as a default strategy. A spy can reveal that the class has too many responsibilities or that a smaller collaborator would make the design easier to test.

### Q10. What do verify and ArgumentCaptor do? *(Intermediate)*

verify checks whether an interaction occurred with specified arguments and invocation counts. ArgumentCaptor captures an argument passed to a mock so the test can inspect its fields or behavior after the call.

Verify interactions when they are part of the contract—for example, a payment command must carry the expected idempotency key. Avoid asserting every internal call order unless order itself matters to correctness.

### Q11. What is strict stubbing, and why does it help? *(Advanced)*

Strict stubbing helps identify unused stubs, argument mismatches, and test setup that does not match actual execution. It reduces false confidence caused by configuring a mock response that the production path never uses.

Do not silence strictness failures indiscriminately. Remove irrelevant setup, correct the expected arguments, or explicitly document the exceptional case where lenient stubbing is justified.

## 3. Testing asynchronous and stateful systems

### Q12. How should asynchronous code be tested without Thread.sleep? *(Senior)*

Use controllable CompletableFuture instances, latches, barriers, test executors, or explicit completion signals so the test controls when work proceeds. Assert observable completion, failure, cancellation, and ordering contracts with bounded timeouts.

Thread.sleep creates timing assumptions and can make tests slow or flaky. A timeout is a safety bound, not a synchronization protocol; the test should coordinate through the same kind of explicit signal the behavior requires.

### Q13. How should concurrent code be tested? *(Senior)*

Test invariants under repeated concurrent operations, coordinate starts and phases with latches or barriers, and bound test duration. Where memory-model or race correctness is critical, use specialized tools such as jcstress in addition to ordinary unit tests; stress tests alone cannot prove correctness.

Keep tests reproducible enough to diagnose failures and avoid assuming one passing run establishes thread safety. Review the synchronization protocol and happens-before relationships separately from empirical test evidence.

### Q14. How can time-dependent code be tested reliably? *(Advanced)*

Inject a java.time.Clock or another time source instead of reading the system clock throughout the business logic. Tests can use a fixed or adjustable clock to verify expiration, scheduling boundaries, and daylight-saving policies without waiting in real time.

Keep wall-clock time separate from monotonic elapsed-time measurement. A fixed clock is useful for business timestamps but is not a substitute for testing timeouts against a monotonic clock where elapsed duration is the requirement.

### Q15. What are property-based tests, and when are they useful? *(Advanced)*

Property-based testing generates many inputs and checks a general invariant instead of relying only on hand-picked examples. It is useful for parsers, normalization, equality contracts, serializers, algorithms, and edge-heavy domain rules.

A property must state what should always hold, and generators should respect or deliberately violate input constraints. Use shrinking or minimized counterexamples when the selected library supports them, and retain important regression examples after discovering a bug.

### Q16. When are Testcontainers or a real database needed? *(Senior)*

Use a real database integration test when correctness depends on SQL dialect, transaction isolation, constraints, locking, query plans, driver mappings, or migrations that an in-memory substitute cannot faithfully model. Testcontainers can provision disposable dependencies for repeatable integration tests where the environment supports containers.

These tests cost more time and infrastructure than unit tests. Keep fast unit coverage, then add targeted integration tests for the actual database and driver behavior the service depends on.

## 4. Build lifecycle and CI

### Q17. What should Maven or Gradle test configuration make explicit? *(Intermediate)*

Make the JDK/toolchain, source release, test framework and engine, test discovery, test task lifecycle, dependency versions, and report locations explicit. Keep unit and integration test tasks separable when they have different runtime costs or environmental requirements.

A green local run is not sufficient if CI selects a different JDK, skips the test engine, excludes a test category, or resolves different dependencies. Verify the build's effective configuration and CI logs.

### Q18. What is the role of Maven Surefire and Failsafe? *(Advanced)*

Maven Surefire is commonly used for unit-test execution during the test phase. Failsafe is designed for integration-test execution across the integration-test and verify phases, allowing post-integration cleanup to occur before the build is finalized.

The exact behavior depends on plugin configuration and version. Ensure CI runs the lifecycle phase that actually executes the intended tests; invoking package or test with the wrong configuration can omit integration coverage.

### Q19. Why use a JDK compatibility matrix in CI? *(Senior)*

A compatibility matrix tests the declared supported Java releases and catches accidental use of newer language syntax, APIs, class-file versions, or runtime behavior. Compile with --release for the intended API/source baseline where applicable, and run tests on the actual target JDKs.

Separate the compilation baseline from the runtime support policy. If the product supports Java 8, compiling only on Java 21 without a Java 8 release constraint can allow newer APIs to slip into the codebase.

### Q20. Does code coverage prove test quality? *(Senior)*

No. Line or branch coverage shows which code was executed, not whether assertions would detect incorrect behavior. A test can execute every line while asserting nothing meaningful, missing boundary conditions, or encoding the same bug as the implementation.

Use coverage to find untested areas, then assess assertions, mutation sensitivity, edge cases, failure behavior, and the risk of the code. Quality comes from meaningful behavioral checks, not from maximizing a percentage alone.

## Official references

1. [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
2. [Mockito Documentation](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
3. [Maven Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/)
4. [Maven Failsafe Plugin](https://maven.apache.org/surefire/maven-failsafe-plugin/)
5. [Gradle Testing Documentation](https://docs.gradle.org/current/userguide/java_testing.html)
6. [Testcontainers for Java](https://java.testcontainers.org/)
7. [OpenJDK jcstress](https://openjdk.org/projects/code-tools/jcstress/)

## Chapter checklist

- [ ] Distinguish unit, integration, and end-to-end tests by scope and dependency boundaries.
- [ ] Use JUnit Jupiter lifecycle, parameterized tests, and assertions deliberately.
- [ ] Mock external boundaries without over-mocking domain behavior.
- [ ] Test asynchronous behavior with explicit coordination rather than sleeps.
- [ ] Make build toolchains and test execution visible in CI.
- [ ] Treat coverage as a diagnostic metric, not proof of correctness.
