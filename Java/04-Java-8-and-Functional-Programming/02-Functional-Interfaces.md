# Functional Interfaces and Composition

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Functional interfaces are the target types for lambdas and method references. The standard library provides reusable contracts for predicates, transformations, consumers, suppliers, primitive operations, and binary operations. Selecting the right abstraction improves API consistency and makes behavior easier to compose.

## 1. Standard interfaces

### Q1. What is the difference between Predicate, Function, Consumer, and Supplier? *(Beginner)*

Predicate<T> accepts T and returns boolean. Function<T,R> maps T to R. Consumer<T> accepts T and returns no result. Supplier<T> produces T without an input. Their names communicate the direction of data flow.

Prefer standard interfaces for common behavior. Create a custom functional interface when a domain-specific name, checked-exception contract, multiple parameters, or clearer API meaning materially improves the design.

### Q2. How do Predicate.and, or, and negate work? *(Intermediate)*

They compose predicates. and requires both predicates to pass and short-circuits if the first is false; or short-circuits if the first is true; negate reverses the result.

    Predicate<String> nonBlank = s -> s != null && !s.isBlank();
    Predicate<String> shortEnough = s -> s.length() < 100;
    Predicate<String> valid = nonBlank.and(shortEnough);

Evaluation order matters when predicates have side effects or can throw. Keep predicates pure where possible so composition remains predictable.

### Q3. What is Function.andThen versus compose? *(Intermediate)*

andThen applies the current function and then the supplied function. compose applies the supplied function first and passes its result to the current function.

    Function<Integer, Integer> doubleIt = n -> n * 2;
    Function<Integer, Integer> plusOne = n -> n + 1;
    System.out.println(doubleIt.andThen(plusOne).apply(3)); // 7
    System.out.println(doubleIt.compose(plusOne).apply(3)); // 8

Naming and tests should make the intended transformation order clear, especially in pipelines where functions are composed dynamically.

### Q4. What is the difference between Consumer.andThen and composing two consumers manually? *(Intermediate)*

Consumer.andThen returns a consumer that executes the first consumer and then the second. If the first throws, the second is not executed and the exception propagates.

This is sequential composition, not a transaction. If both consumers perform side effects, the first may have completed before the second fails. Use compensation or transactional boundaries when partial side effects are unacceptable.

### Q5. What are primitive-specialized functional interfaces? *(Intermediate)*

Interfaces such as IntPredicate, IntFunction<R>, ToIntFunction<T>, IntConsumer, and IntUnaryOperator avoid boxing for common primitive values. They are useful in numeric processing and hot paths where wrapper allocation or conversion costs matter.

Use them when the primitive data flow is natural, not as a premature optimization. Mixing object and primitive interfaces can introduce conversions, so inspect the whole pipeline.

### Q6. Why are BiFunction and BiConsumer useful? *(Beginner)*

BiFunction<T,U,R> accepts two arguments and returns a result; BiConsumer<T,U> accepts two arguments and returns no result. They model common operations such as map merging, pair processing, and callbacks with two inputs.

For more complex domain operations, a named method or domain type may be more readable than a deeply nested generic functional signature.

### Q7. What is a custom functional interface with checked exceptions? *(Advanced)*

Standard interfaces such as Function do not declare checked exceptions, so code calling an API that throws IOException must handle or translate it inside the lambda. A custom interface can declare the checked exception when it is part of the contract.

    @FunctionalInterface
    interface IOFunction<T, R> {
        R apply(T value) throws IOException;
    }

Custom interfaces should have a clear purpose and avoid proliferating near-duplicates. They can make error handling explicit at the boundary instead of hiding it in unchecked wrappers.

## 2. Method references and inference

### Q8. Why can a method reference be ambiguous? *(Advanced)*

An overloaded method reference may match multiple overloads, and the target functional interface determines which signature is required. If inference cannot choose uniquely, compilation fails.

Use an explicit target type or lambda with parameter types to clarify intent. Method references are shorthand, not a requirement; a small lambda can be easier to understand when overloads are complex.

### Q9. How do generic functional interfaces work with type inference? *(Advanced)*

The compiler infers type arguments from the target context, method parameters, and return types. Generic method references can be especially sensitive to context because the compiler must select both the method and functional-interface type.

If a call becomes ambiguous or infers Object unexpectedly, introduce a typed local variable or explicit type witness. Avoid hiding important API types behind excessively generic helper chains.

### Q10. When should you create a domain-specific functional interface? *(Senior)*

Create one when it gives a meaningful name to a business contract, captures a checked exception, has a useful domain-specific parameter/return shape, or improves documentation. For example, PricePolicy or RetryDecision can communicate more than Function<Request, Boolean>.

Do not create a new interface merely to rename Predicate or Function without adding semantics. Keep public callback contracts stable and document thread-safety, invocation count, ordering, and exception behavior.

## 3. Production considerations

### Q11. Are functional interfaces guaranteed to be stateless? *(Advanced)*

No. A lambda can capture state, and a method reference can call a stateful object. Functional interfaces describe a method shape, not purity, immutability, or thread-safety.

When callbacks may be invoked concurrently or repeatedly, document and enforce their behavior. Stream behavioral parameters have additional non-interference/statelessness expectations; those are not implied by the functional-interface type alone.

### Q12. How do you handle nulls in functional composition? *(Advanced)*

The standard composition methods typically require the supplied function or predicate to be non-null, and the composed operation may still receive a null input depending on its type and body. Decide null semantics at the API boundary.

Do not use Optional or predicates as a way to conceal an unclear null contract. Make whether null is allowed explicit and test it.

### Q13. How do callbacks affect error handling and retries? *(Senior)*

A callback can throw, perform side effects, or be invoked more than once depending on the API. A retrying API may re-execute the callback, so side effects must be idempotent or compensated. An asynchronous API may invoke it on a different thread and propagate failures through a future.

Read the exact callback contract. Document invocation count, thread context, ordering, cancellation, and checked/unchecked failure behavior rather than assuming lambdas execute immediately on the caller's thread.

### Q14. How would you review a functional API for maintainability? *(Senior)*

Check whether the functional-interface type communicates intent, whether checked exceptions are handled transparently, whether callbacks can be invoked multiple times or concurrently, and whether ordering and side effects are specified. Avoid nested composition that obscures the business rule.

Use names for important domain policies, keep lambdas small, and test failure behavior. Functional syntax should reduce incidental complexity rather than make control flow invisible.

## Official references

1. [Java SE 21 API — java.util.function](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/function/package-summary.html)
2. [Java SE 21 API — Predicate](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/function/Predicate.html)
3. [Java SE 21 API — Function](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/function/Function.html)
4. [Java SE 21 API — Consumer](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/function/Consumer.html)

## Chapter checklist

- [ ] Choose the standard functional interface that matches the data flow.
- [ ] Explain composition order and short-circuit behavior.
- [ ] Use primitive-specialized interfaces when appropriate.
- [ ] Make callback exception, concurrency, and invocation contracts explicit.
