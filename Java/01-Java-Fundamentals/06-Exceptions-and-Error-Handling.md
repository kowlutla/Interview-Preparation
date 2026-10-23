# Exceptions and Error Handling

**Target level:** Beginner → Senior | **Scope:** Java 8–21

Exception handling is a control-flow mechanism and an API design decision. Production code distinguishes recoverable failures from defects, preserves diagnostic context, avoids hiding root causes, and gives callers enough information to choose a response.

## 1. Hierarchy and exception types

### Q1. What is the Java exception hierarchy? *(Beginner)*

Throwable is the root of throwable objects. Its major subclasses are Error and Exception; RuntimeException is a subclass of Exception. Checked exceptions are exception types other than RuntimeException and its subclasses and Error and its subclasses. Errors and runtime exceptions are unchecked.

Errors generally indicate serious JVM or environment problems that applications should not attempt to recover from broadly. Exceptions may represent conditions an application can handle. The correct policy depends on the specific failure, not only its class name.

### Q2. What is the difference between checked and unchecked exceptions? *(Beginner)*

The compiler requires checked exceptions to be caught or declared in a throws clause. RuntimeException subclasses and Error subclasses do not have that compile-time obligation.

Checked exceptions can make recoverable conditions explicit, but overuse may force implementation details through many layers. Unchecked exceptions simplify signatures but still need documented handling. Neither category automatically tells you whether a particular failure is recoverable.

### Q3. What is the difference between Exception and Error? *(Intermediate)*

Exception represents conditions applications may want to catch or propagate. Error is intended for serious problems that ordinary applications generally should not handle, such as certain linkage or virtual-machine failures. Both extend Throwable.

Avoid catch(Throwable) as a routine recovery strategy: it catches both exceptions and errors, including conditions the process may not be able to recover from. Broad catching may be justified at a process boundary for logging or cleanup, but preserve fatal behavior and interruption semantics.

### Q4. When should you create a custom exception? *(Intermediate)*

Create one when it communicates a meaningful domain or API failure callers need to distinguish, such as an invalid order or a payment decline. Include useful context but avoid secrets or internal details that should not leave the service.

Do not create a new exception class for every minor branch. Prefer a stable hierarchy aligned with caller decisions and preserve the original cause when translating lower-level failures.

### Q5. How do you choose between checked and unchecked custom exceptions? *(Advanced)*

Ask whether callers can reasonably recover and whether the API should force them to acknowledge the condition. Checked exceptions may suit expected alternatives that callers can handle locally. Unchecked exceptions often suit violated preconditions, programming defects, or failures that cannot be usefully handled at every intermediate layer.

Consistency within an API matters. A checked exception does not guarantee recovery, and an unchecked exception does not mean it can be ignored. Document failure semantics.

## 2. try, catch, finally and propagation

### Q6. How does try-catch-finally work? *(Beginner)*

The try block may complete normally or abruptly. A matching catch handles a thrown exception. A finally block normally executes as control leaves the try/catch construct, including on return or an exception.

If finally itself completes abruptly, it can replace the earlier reason for completion. This is why returning or throwing a new exception from finally is dangerous. Use finally for cleanup and try-with-resources for AutoCloseable resources.

### Q7. What happens if a catch block throws another exception? *(Intermediate)*

The new exception propagates outward unless caught by an enclosing handler. The original exception is not automatically retained if the new exception is created independently.

    try {
        repository.load(id);
    } catch (SQLException e) {
        throw new DataAccessException("Could not load record " + id, e);
    }

Wrapping preserves the cause while adding domain context. Avoid leaking credentials, sensitive query values, or personal data in messages.

### Q8. What happens when try and finally both return? *(Advanced)*

A return in finally overrides a pending return from try or catch. It can also suppress an exception that was about to propagate. This behavior is defined by Java but makes code hard to reason about.

    static int example() {
        try {
            return 1;
        } finally {
            // return 2; would make this return 2
        }
    }

Do not return from finally. Let cleanup happen without changing the original result or exception.

### Q9. What is try-with-resources? *(Intermediate)*

Try-with-resources automatically closes resources implementing AutoCloseable when the block exits, including when an exception occurs. Resources close in reverse order of initialization.

    try (var input = Files.newBufferedReader(path)) {
        return input.readLine();
    }

It reduces boilerplate and handles close failures consistently. Java 9 also permits an effectively final variable declared earlier to be used as a resource, when the selected source level supports it.

### Q10. What are suppressed exceptions? *(Advanced)*

If the try-with-resources body throws and closing a resource also throws, the body exception remains primary and the close failure is attached as a suppressed exception. Retrieve these with getSuppressed().

This preserves the primary failure instead of letting cleanup hide it. Inspect both causes and suppressed exceptions when diagnosing resource problems; logging only the message can discard essential context.

### Q11. How are multiple catch blocks matched? *(Intermediate)*

Catch clauses are tested in source order; the first compatible catch handles the exception. A specific type must appear before a broader supertype, or the later catch may be unreachable and rejected by the compiler.

Multi-catch uses a vertical bar to handle alternatives in one block, subject to compatibility rules. It cannot combine types where one alternative is a subtype of another.

### Q12. What is multi-catch? *(Intermediate)*

Multi-catch allows unrelated exception types to share a handler.

    try {
        performOperation();
    } catch (IOException | SQLException e) {
        throw new OperationException("Operation failed", e);
    }

Use it when the recovery or translation policy is genuinely identical. Separate handlers are clearer when each failure needs different remediation.

### Q13. What is exception propagation? *(Beginner)*

If a method does not handle a thrown exception, it propagates up the call stack until a compatible handler is found or the thread's uncaught-exception handling is reached. Checked exceptions must be caught or declared; unchecked exceptions can propagate without appearing in the method signature.

Translate exceptions only when crossing a meaningful abstraction boundary or adding useful context. Repeatedly catching and rethrowing without value adds noise and can obscure the stack trace.

## 3. Reliable error handling

### Q14. Why preserve the original cause? *(Intermediate)*

The cause contains the lower-level failure and stack trace. Wrapping it preserves both domain-level meaning and diagnostic detail. Use constructors that accept a cause rather than replacing it with only a message.

If the original cause could leak sensitive data through a public API, sanitize the external presentation while retaining appropriately controlled internal diagnostics.

### Q15. Why can catching Exception broadly be a bad idea? *(Advanced)*

A broad catch can combine failures requiring different responses, hide programming bugs, swallow interruption, or continue with corrupted state. It makes the recovery contract unclear.

Broad catches can be appropriate at process, request, job, or thread boundaries. At those boundaries, define what happens next, preserve observability, and do not report success after failure. Catch narrowly inside business logic.

### Q16. Why should exceptions not be swallowed? *(Intermediate)*

Ignoring a failure without informing the caller or observability systems can leave the application with incomplete data while reporting success. If a failure is intentionally ignorable, document why and consider a metric or appropriately leveled log.

Avoid logging the same exception at every layer. A single well-chosen boundary often gives clearer diagnostics without flooding logs.

### Q17. What is the difference between rethrowing and wrapping? *(Intermediate)*

Rethrowing the same exception preserves its type and causal identity. Wrapping creates a new exception that can add context or translate implementation-specific failures into a stable abstraction. Java also supports precise rethrow analysis for suitable code.

Choose based on what callers should know. Wrapping every exception indiscriminately creates excessive layers and makes diagnostics harder to read.

### Q18. How should InterruptedException be handled? *(Advanced)*

InterruptedException signals that a blocking operation was interrupted. If a method cannot propagate it, it commonly restores the interrupt flag before returning or translating the failure.

    try {
        queue.take();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return;
    }

Do not silently swallow interruption in executor, request, or shutdown paths. If the method declares InterruptedException, usually propagate it rather than converting it to an unrelated unchecked exception.

### Q19. When is an exception better than an ordinary result? *(Advanced)*

Exceptions are suitable when an operation cannot proceed or a failure is exceptional relative to the API contract. Ordinary branching or a result type is often better for frequent expected alternatives, such as a search that may find no result.

Validate preconditions near the boundary and avoid using exceptions as routine control flow in hot paths. The best representation depends on frequency, caller decisions, and the API's domain meaning.

### Q20. How should exception messages be written? *(Senior)*

A useful message identifies the failed operation and safe contextual identifiers while avoiding passwords, tokens, payment details, and uncontrolled user input. Preserve the cause and use structured logging fields where possible.

Messages should help operators answer what failed and where. Avoid vague text and avoid serializing entire request objects that may contain personal or confidential data.

## 4. Resources, transactions and services

### Q21. What happens if close throws? *(Advanced)*

In try-with-resources, if the body completes normally and close throws, the close exception can become primary. If the body already threw, close failures are suppressed on the primary exception. Multiple resources close in reverse declaration order.

Design resource dependencies so that close ordering is correct. Inspect suppressed exceptions when troubleshooting shutdown or I/O failures.

### Q22. Should you catch exceptions inside a database transaction? *(Senior)*

Catch only when you can make a correct decision. Swallowing a persistence exception and continuing can leave a transaction marked rollback-only or permit partial work where atomicity is not guaranteed. A translated exception should still cause rollback when the operation failed.

In Spring, default rollback behavior generally differs for unchecked and checked exceptions, and can be configured. Verify the framework's actual rules and explicitly define rollback behavior for critical business operations.

### Q23. How should exceptions cross REST API boundaries? *(Senior)*

Translate internal failures into stable HTTP responses at a centralized boundary, such as Spring's controller advice mechanism. Return a safe error code, client-facing message, and correlation identifier where appropriate. Do not expose stack traces, SQL details, internal hostnames, or secrets.

Map validation errors, not-found conditions, conflicts, authentication/authorization failures, and unexpected server errors deliberately. Keep public error schemas stable while preserving internal diagnostics.

### Q24. What is an exception taxonomy in microservices? *(Senior)*

An exception taxonomy groups failures by operational meaning: validation, not found, conflict, dependency unavailable, timeout, rate limit, and unexpected defect. The taxonomy should guide retry, alerting, HTTP mapping, and caller behavior.

Do not retry every exception. Validation errors are not transient, while a timeout may or may not be safe to retry depending on idempotency and whether the remote operation committed. Define retryability explicitly rather than inferring it from messages.

### Q25. How do retries interact with exceptions? *(Senior)*

Retry only failures that may succeed later and only when the operation is safe to repeat. Use bounded attempts, exponential backoff, jitter, deadlines, and a retry budget. Avoid retry layers multiplying attempts unexpectedly.

A timeout does not prove a remote side effect did not occur. For non-idempotent operations, use idempotency keys, deduplication, or reconciliation. Exceptions should carry enough structured information for policy to distinguish transient from permanent failures.

## 5. Traps and diagnostics

### Q26. What happens if finally throws after try throws? *(Advanced)*

    try {
        throw new IllegalStateException("primary");
    } finally {
        throw new IllegalArgumentException("cleanup");
    }

The IllegalArgumentException from finally becomes the exception that propagates; the original exception is hidden rather than automatically suppressed. This is one reason not to throw from finally. Try-with-resources has specific suppressed-exception behavior for cleanup.

### Q27. Can an overriding method declare a broader checked exception? *(Intermediate)*

No. An overriding method cannot broaden the checked exceptions allowed by the overridden method. It can declare fewer checked exceptions or narrower compatible ones and may throw unchecked exceptions.

This preserves substitutability: callers using the base type should not need to handle a new checked exception merely because the runtime object is a subtype.

### Q28. What happens when class initialization fails? *(Advanced)*

If class initialization completes abruptly, the first active use may receive ExceptionInInitializerError when the original throwable is not already an Error. The class is marked erroneous for that class loader, and later active uses can fail with NoClassDefFoundError.

Keep static initialization simple. Avoid network calls, complex configuration, and fragile external dependencies in static initializers; explicit startup validation provides clearer failure reporting.

### Q29. How do you debug a production exception with a long cause chain? *(Senior)*

Identify the exception type, the first relevant application frame, and the deepest meaningful cause. Inspect suppressed exceptions and structured context such as request IDs, dependency name, timeout, and retry attempt. Correlate logs and traces without exposing sensitive payloads.

Find the first boundary where the failure was created or translated. Determine whether it is permanent, transient, a timeout with unknown outcome, or a defect. Fix the violated invariant or dependency behavior rather than only changing the final catch block.

### Q30. What is a robust exception-handling strategy for a backend service? *(Senior)*

Validate at boundaries, use domain errors where they clarify caller decisions, preserve causes, and translate failures at stable API or messaging boundaries. Retry only safe transient operations, restore interrupt status, and never report success after a failed operation.

Use structured logs, metrics, traces, correlation identifiers, and bounded error responses. Use try-with-resources, make transaction rollback behavior explicit, and test timeout, cancellation, duplicate delivery, resource-close failure, and partial remote success.

## Official references

1. [Java Language Specification, Java SE 21 — Exceptions](https://docs.oracle.com/javase/specs/jls/se21/html/jls-11.html)
2. [JLS Chapter 14 — Statements and try](https://docs.oracle.com/javase/specs/jls/se21/html/jls-14.html)
3. [Java SE 21 API — Throwable](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Throwable.html)
4. [Java SE 21 API — AutoCloseable](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/AutoCloseable.html)
5. [Java SE 21 API — InterruptedException](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/InterruptedException.html)

## Chapter checklist

- [ ] Distinguish checked exceptions, RuntimeException, and Error.
- [ ] Preserve causes and suppressed exceptions.
- [ ] Use try-with-resources and understand reverse close order.
- [ ] Avoid swallowing interruption or returning from finally.
- [ ] Design retries and transaction policies around idempotency and failure semantics.
- [ ] Translate internal exceptions into safe, stable API responses.

## Worked examples: exception contracts

### Example 1. Try-with-resources preserves the primary exception

~~~java
class Resource implements AutoCloseable {
    @Override public void close() throws Exception {
        throw new Exception("close failed");
    }
}

static void demonstrateSuppression() throws Exception {
    try (Resource resource = new Resource()) {
        throw new Exception("work failed");
    }
}
~~~

When both the body and close() fail, the exception from the body remains primary and the close failure is recorded as a suppressed exception. Inspect getSuppressed() during diagnosis. This behavior is safer than hand-written cleanup that accidentally replaces the original failure.

### Example 2. Preserve interruption when you cannot propagate it

~~~java
try {
    Thread.sleep(100);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return; // or stop/cancel the current operation
}
~~~

Many blocking APIs clear the interrupted status when they throw InterruptedException. If the method cannot propagate the exception, restore the status and stop or transition to a defined cancellation path. Swallowing interruption can break shutdown, cancellation, and task ownership.

### Example 3. Do not turn every failure into an empty result

If a repository returns Optional.empty() for “record not found,” a database timeout should not also become Optional.empty(). Absence is a domain result; a timeout is an operational failure that may require retry, an error response, or a circuit-breaker policy.

Keep error mapping at clear boundaries. Preserve causes for diagnostics, avoid leaking sensitive internals to clients, and do not retry a non-idempotent operation without a deduplication or reconciliation strategy.
