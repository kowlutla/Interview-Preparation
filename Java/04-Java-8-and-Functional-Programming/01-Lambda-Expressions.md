# Lambda Expressions

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Lambda expressions made behavior a first-class argument in Java, especially for collections, asynchronous APIs, and event-driven systems. Understanding target typing, functional interfaces, capture rules, and side effects is more important than memorizing arrow syntax.

## 1. Lambda fundamentals

### Q1. What is a lambda expression? *(Beginner)*

A lambda expression is a concise way to provide an implementation of a functional interface's abstract method. It can represent behavior and pass it to another method, such as a comparator, event handler, or stream operation.

    Runnable task = () -> System.out.println("run");
    new Thread(task).start();

A lambda has no standalone declared class name in source. Its type is inferred from the target functional-interface type, so a lambda generally requires a target context.

### Q2. What is a functional interface? *(Beginner)*

A functional interface has one abstract method that defines its functional contract, accounting for inherited declarations and public Object methods under the language rules. It may still declare default, static, and private methods.

The @FunctionalInterface annotation makes the intended contract compiler-checked. Common examples include Predicate<T>, Function<T,R>, Consumer<T>, Supplier<T>, and Runnable.

### Q3. What is target typing? *(Intermediate)*

A lambda's type is inferred from the context in which it appears. For example, the same lambda body may be compatible with different functional interfaces if the target parameter and return types fit.

    Predicate<String> nonEmpty = s -> !s.isEmpty();
    Function<String, Integer> length = s -> s.length();

Without a suitable target type, a lambda may not be a standalone expression that the compiler can type. Overloaded methods accepting different functional interfaces can therefore become ambiguous.

### Q4. What does effectively final mean for lambda capture? *(Intermediate)*

A lambda can capture local variables and parameters that are final or effectively final, meaning they are not reassigned after initialization. The captured local value is not a mutable stack slot that the lambda can change later.

    int threshold = 10;
    Predicate<Integer> large = value -> value > threshold;

If threshold were reassigned, the capture would fail to compile. For shared mutable state, use an explicitly designed holder or concurrency primitive and protect it appropriately; capture restrictions are not a thread-safety mechanism.

### Q5. How does this differ between a lambda and an anonymous class? *(Advanced)*

In a lambda, this refers to the enclosing instance; a lambda does not introduce a new this scope. In an anonymous class, this refers to the anonymous class instance. Anonymous classes also declare their own fields and methods, whereas a lambda is primarily a functional-interface implementation.

Lambdas can make callbacks more concise, but anonymous classes remain useful when a named local implementation body or a distinct this context is needed. Choose based on semantics and readability.

### Q6. Are lambdas always faster than anonymous classes? *(Advanced)*

No universal performance guarantee applies. The JVM can implement lambda objects using invokedynamic and runtime linkage strategies, and optimization depends on usage, JDK, escape behavior, and workload. Anonymous classes may also be optimized.

Do not choose syntax based on folklore. Prefer the form that communicates intent, then benchmark representative hot paths if performance matters.

## 2. Functional interfaces and method references

### Q7. What is Predicate<T>? *(Beginner)*

Predicate<T> represents a function that accepts T and returns boolean. It is commonly used for filtering and validation.

    Predicate<String> hasText = s -> s != null && !s.isBlank();

Predicate supports composition such as and, or, and negate. Be careful about null handling: a predicate can throw if its body dereferences a null value unless the contract excludes null.

### Q8. What are Function, Consumer, Supplier, and UnaryOperator? *(Intermediate)*

Function<T,R> transforms T into R. Consumer<T> accepts a value and returns no result. Supplier<T> produces a value without an input. UnaryOperator<T> is a specialization of Function<T,T>.

Use the standard interfaces when their semantics fit rather than creating a new interface for every callback. If the behavior needs checked exceptions, multiple arguments, or a meaningful domain name, a custom functional interface may make the contract clearer.

### Q9. What is a method reference? *(Intermediate)*

A method reference is shorthand for a lambda that delegates to an existing method or constructor. Forms include a static method reference, a bound instance method reference, an unbound instance method reference, and a constructor reference.

    Function<String, Integer> parse = Integer::parseInt;
    Supplier<List<String>> create = ArrayList::new;
    Consumer<String> print = System.out::println;

The target functional interface determines how the reference is adapted. A method reference is not necessarily clearer than a short lambda, especially when overload resolution is complex.

### Q10. What is the difference between bound and unbound instance method references? *(Advanced)*

A bound reference such as object::method captures a particular receiver. An unbound reference such as String::length uses the first functional argument as the receiver when the method is invoked.

The receiver expression of a bound method reference is evaluated when the reference is created. If the receiver is null, creating the reference can throw NullPointerException immediately, unlike a lambda whose body dereferences the value later.

### Q11. How do default methods in functional interfaces support composition? *(Intermediate)*

Functional interfaces often provide default methods that combine behavior while keeping one abstract method. Predicate.and, Predicate.or, and Function.andThen are examples.

Composition order matters. Function.andThen applies the current function first and then the supplied function, while compose applies the supplied function first. Validate null arguments where the API requires it and consider exceptions/side effects during composition.

## 3. Correctness and production usage

### Q12. Why should lambdas passed to streams avoid side effects? *(Advanced)*

Stream operations may be lazy, optimized, or executed in parallel depending on the pipeline. Side effects can make results dependent on execution order, introduce races, and prevent safe parallelization.

Prefer transformations and collectors that express the result declaratively. When a side effect is genuinely required, make ordering, synchronization, failure behavior, and exactly-once expectations explicit.

### Q13. Can a lambda throw a checked exception? *(Advanced)*

A lambda can throw a checked exception only when the target functional method's throws clause permits it. Standard interfaces such as Function generally do not declare checked exceptions, so calling a checked-throwing API inside one requires handling or translating the exception.

Avoid generic “sneaky throw” helpers unless there is a well-justified library-level reason. Prefer an explicit adapter or custom functional interface whose contract declares the checked exception.

### Q14. Why can overloaded methods make a lambda ambiguous? *(Advanced)*

If two overloads accept different functional-interface types and the lambda can fit both, the compiler may not be able to select one. The ambiguity is resolved at compile time, not based on runtime behavior.

Use a typed local variable, an explicit cast, or a clearer overload design. Avoid APIs with unrelated functional-interface overloads that make common lambda calls difficult to read.

### Q15. How do you avoid capturing a large object accidentally? *(Senior)*

A lambda or inner class can retain captured references for as long as the callback remains reachable. A long-lived scheduler, listener, cache, or asynchronous task can therefore keep a service or request graph alive unexpectedly.

Capture only what is needed, unregister listeners, cancel tasks, and inspect heap retention when investigating leaks. A lambda's concise syntax does not remove ordinary object-lifetime consequences.

### Q16. How should you test lambda-based behavior? *(Senior)*

Test the functional contract with representative inputs, boundaries, exceptions, and null behavior where allowed. For composed predicates/functions, verify composition order and short-circuit behavior. For asynchronous callbacks, test scheduling and cancellation at the relevant abstraction boundary.

Avoid tests that assert implementation details of the generated lambda class. Test observable behavior and contracts rather than synthetic class names or bytecode shape.

## Official references

1. [Java SE 21 API — java.util.function](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/function/package-summary.html)
2. [JLS Chapter 15 — Lambda Expressions and Method References](https://docs.oracle.com/javase/specs/jls/se21/html/jls-15.html)
3. [JEP 126 — Lambda Expressions](https://openjdk.org/jeps/126)
4. [Java SE 21 API — Runnable](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Runnable.html)

## Chapter checklist

- [ ] Explain target typing, capture, and the lambda/anonymous-class this difference.
- [ ] Choose standard functional interfaces appropriately.
- [ ] Understand method-reference forms and overload ambiguity.
- [ ] Avoid side effects and accidental object retention.
- [ ] Test observable functional behavior, not generated implementation details.
