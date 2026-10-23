# Java 8 Features

**Target level:** Intermediate → Senior  
**Release:** Java 8 (2014)

Java 8 changed everyday Java programming through lambdas, functional interfaces, streams, Optional, the java.time API, default methods, and major library updates. The key is knowing both syntax and behavioral contracts, including laziness, ordering, mutation, and compatibility.

## 1. Language and library features

### Q1. What were the most important Java 8 features? *(Beginner)*

Major features include lambda expressions, method references, functional interfaces, the Stream API, Optional, default and static interface methods, the java.time API, CompletableFuture, Base64 utilities, and enhancements to collections and Map.

Do not reduce Java 8 to streams alone. These features work together to make behavior composable while retaining Java's static type system and compatibility goals.

### Q2. What problem did default methods solve? *(Intermediate)*

Default methods let interfaces add certain behaviors without forcing every existing implementation to add a method immediately. They helped evolve APIs such as collections while maintaining source compatibility for many implementers.

Conflicts between unrelated defaults can require an explicit override. A default method must make semantic sense for the abstraction; it should not be used as a place to put arbitrary implementation code.

### Q3. What is the difference between a lambda and an anonymous class in Java 8? *(Intermediate)*

A lambda targets a functional interface and uses the enclosing this context. An anonymous class declares a new class body and has its own this. Local variables captured by either mechanism must be final or effectively final.

Lambdas are concise for behavior parameters, while anonymous classes remain useful for stateful local implementations or a distinct this context.

### Q4. What did the Stream API add? *(Beginner)*

Streams support declarative pipelines over data sources using operations such as filter, map, sorted, reduce, and collect. Intermediate operations are generally lazy; terminal operations trigger evaluation.

Streams are not collections and do not automatically make operations parallel or faster. Correctness depends on stateless behavioral parameters, source non-interference, and clear ordering semantics.

### Q5. Why was Optional introduced? *(Intermediate)*

Optional represents a present non-null value or absence, especially for method return contracts. It can make absence explicit and reduce accidental null dereferences.

Optional is not a universal replacement for exceptions or nullable fields. Use it when absence is a legitimate result and keep operational failures distinct from missing data.

### Q6. What did java.time improve over Date and Calendar? *(Intermediate)*

java.time introduced immutable, clearer types such as LocalDate, Instant, ZonedDateTime, Duration, Period, and DateTimeFormatter. These types distinguish calendar values from global timeline points and make time-zone rules more explicit.

For globally meaningful events, store an Instant or another type with clear offset semantics. For future local schedules, retain the relevant ZoneId and define daylight-saving gap/overlap behavior.

### Q7. What is CompletableFuture in Java 8? *(Intermediate)*

CompletableFuture implements Future and CompletionStage and supports asynchronous composition, completion, and exception handling. It enables pipelines without manually coordinating every result through blocking get calls.

Not every continuation is asynchronous, and default async operations generally use the common pool. Use explicit executors for blocking tasks and define timeout/cancellation semantics.

### Q8. What are the Java 8 improvements to Map? *(Intermediate)*

Map gained methods such as forEach, getOrDefault, putIfAbsent, remove(key,value), replace, computeIfAbsent, computeIfPresent, compute, and merge. These express common map updates more directly.

For concurrent maps, read the concrete implementation's atomicity guarantees. Do not assume a default interface method turns a compound sequence into a transaction for every Map implementation.

### Q9. What is effectively final, and why does it matter in Java 8? *(Intermediate)*

A local variable is effectively final if it is not reassigned after initialization. Lambdas and local/anonymous classes can capture such variables.

The captured local is not a mutable shared cell. For shared state, use a suitable object or concurrency primitive and explicitly handle visibility and atomicity.

### Q10. What are method references? *(Beginner)*

Method references provide shorthand for lambdas that call existing methods or constructors, such as Integer::parseInt, System.out::println, and ArrayList::new. Their target functional-interface type determines the adaptation.

They are useful when they improve readability, but a short lambda can be clearer when overload resolution or argument transformation is nontrivial.

## 2. Compatibility and traps

### Q11. Can Java 8 source use var, records, or text blocks? *(Intermediate)*

No. Local-variable type inference with var became permanent in Java 10, records in Java 16, and text blocks in Java 15. Compiling for Java 8 compatibility rejects these newer language features.

Use the build's --release 8 setting when the artifact must target Java 8 language/API compatibility. A newer compiler alone does not make a newer language feature available to a Java 8 runtime.

### Q12. How do you make a Java 8 stream pipeline safe? *(Advanced)*

Avoid modifying the source while it is being traversed, keep intermediate behavioral parameters stateless, and use collectors rather than shared mutable accumulation. Parallel execution can reorder operations and introduce concurrency.

A stream pipeline should express the computation, not hide required side effects in peek or mutable captured variables. Use a loop when complex control flow or checked exceptions are clearer.

### Q13. What are common Java 8 migration risks? *(Senior)*

Risks include changed overload resolution in source code, default-method conflicts, new API dependencies that raise the runtime baseline, different date/time semantics, and accidental parallel-stream use. Library behavior and performance should be tested on the target runtime.

Compile against the intended release, run integration tests, and verify deployment JDKs. Avoid assuming code that compiles on a modern JDK will run on Java 8.

### Q14. What is the difference between source compatibility and runtime compatibility? *(Advanced)*

Source compatibility concerns whether source code compiles under a language/API level. Runtime compatibility concerns whether compiled class files and referenced APIs can run on the target JVM. Binary compatibility concerns linkage between compiled artifacts and runtime dependencies.

Use javac --release where supported to constrain language features, bytecode target, and documented platform APIs together. Test against the actual runtime baseline.

### Q15. How should you answer “What changed in Java 8?” in an interview? *(Senior)*

Organize the answer into language features, library APIs, concurrency, date/time, and compatibility. Explain one feature's motivation and a practical trade-off—for example, streams are lazy and composable but side effects and parallel execution require care.

Avoid listing every feature without depth. Connect the release to real code you have written and be precise about later features that are not available in Java 8.

## Official references

1. [Java SE 8 API documentation](https://docs.oracle.com/javase/8/docs/api/)
2. [JEP 126 — Lambda Expressions](https://openjdk.org/jeps/126)
3. [Java SE 8 Stream package](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)
4. [Java SE 8 Date-Time package](https://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html)

## Chapter checklist

- [ ] Explain lambdas, functional interfaces, streams, and Optional.
- [ ] Explain default methods and interface evolution.
- [ ] Use java.time types according to temporal semantics.
- [ ] Distinguish Java 8 APIs from later language features.
- [ ] Verify source/API/runtime compatibility explicitly.
