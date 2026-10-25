# Java Interview Handbook — Complete Topic-Wise Revision Guide (Java 8–25)

A focused, version-aware handbook for Java backend engineers preparing for experienced and senior roles. Each question has one canonical topic home, and the contents follow a learning path from fundamentals through core APIs and runtime behavior to Java-version evolution and production-level reasoning.

**Scope:** Java 8 fundamentals through Java 25, with particular emphasis on LTS releases Java 8, 11, 17, 21, and 25.  
**Audience:** Developers preparing for experienced and senior Java/backend roles.  
**How to use this guide:** Read in order. Questions retain stable Q numbers for linkability but are grouped by subject rather than numeric order. Complete core topics before moving to Java release evolution, code-reading practice, and production scenarios.

> Interview-question frequency varies by company, interviewer, and role. This selection is curated from recurring themes and cross-checked against official Java documentation; it is not a guarantee that any particular company will ask every question.

## Contents

1. [Java foundations and execution](#1-java-foundations-and-execution)
2. [Object-oriented programming, contracts, and design](#2-object-oriented-programming-contracts-and-design)
3. [Strings, wrappers, numeric behavior, and equality](#3-strings-wrappers-numeric-behavior-and-equality)
4. [Exception handling and resource safety](#4-exception-handling-and-resource-safety)
5. [Generics, type system, annotations, and reflection](#5-generics-type-system-annotations-and-reflection)
6. [Collections framework and data structures](#6-collections-framework-and-data-structures)
7. [Functional programming, Optional, date/time, and everyday APIs](#7-functional-programming-optional-datetime-and-everyday-apis)
8. [Stream API fundamentals and coding questions](#8-stream-api-fundamentals-and-coding-questions)
9. [Concurrency, Java Memory Model, executors, and asynchronous programming](#9-concurrency-java-memory-model-executors-and-asynchronous-programming)
10. [JVM internals, garbage collection, and diagnostics](#10-jvm-internals-garbage-collection-and-diagnostics)
11. [I/O, networking, JDBC, security, testing, and performance](#11-io-networking-jdbc-security-testing-and-performance)
12. [Java version evolution and migration: Java 8–25](#12-java-version-evolution-and-migration-java-825)
13. [Code-reading and output questions](#13-code-reading-and-output-questions)
14. [Production, senior-level scenarios, and engineering judgement](#14-production-senior-level-scenarios-and-engineering-judgement)
15. [Last-minute revision](#15-last-minute-revision)
16. [Java Version Features (Java 8–25)](#16-java-version-features-java-825)
17. [Official references](#17-official-references)
18. [Editorial and validation notes](#18-editorial-and-validation-notes)

**Reading order:** Java execution and language foundations → OOP and contracts → strings/exceptions/generics → Collections → functional APIs and Streams → concurrency → JVM → I/O, networking and engineering practices → Java 8–25 evolution → code reading → production/senior scenarios.

## 1. Java foundations and execution

### Q1. What are the JDK, JVM, and JRE, and how does a Java program run?

The **JDK** is the development kit: it includes tools such as `javac`, `java`, `jar`, `javadoc`, and diagnostic utilities, together with the runtime components. The **JVM** is the abstract machine that loads and executes class files and defines runtime behavior. The **JRE** traditionally refers to the JVM plus the libraries and supporting files needed to run applications; since Java 9's modular runtime-image model, a separate JRE download is not universally provided.

In the common workflow, `javac` compiles source code into class files containing bytecode and metadata. The class loader locates definitions, the JVM links them (including verification and preparation), initializes classes when required, and executes bytecode using interpretation and/or just-in-time compilation. The exact execution strategy is an implementation detail.

“Platform independent” means compatible bytecode can run on a compatible JVM across platforms. It does not remove dependencies on native libraries, OS behavior, permissions, locale, time zones, or environment configuration.

### Q2. Is Java pass-by-value or pass-by-reference?

Java is **always pass-by-value**. For a primitive, the method receives a copy of the primitive value. For an object argument, the method receives a copy of the reference value, so both references can point to the same object.

That allows a method to mutate the referenced object's state, but reassigning its parameter does not change the caller's variable. This distinction explains why a method can modify a passed-in list but cannot make the caller's list variable refer to a newly created list simply by assigning to its parameter.

### Equality and string contracts

### Q3. What is the difference between a primitive and a reference type?

Primitive variables hold values from primitive types such as int, boolean, and double. Reference variables hold references to objects or null. Primitives do not have object identity or instance methods, while reference values participate in object-oriented behavior and can be null. Wrapper types bridge primitive values and APIs that require objects, such as generic collections.

### Q4. What is numeric promotion in Java?

In arithmetic expressions, smaller integral types are commonly promoted to int, and mixed numeric operands are promoted according to Java's binary numeric promotion rules. This is why adding two byte values produces an int expression and assigning it directly to a byte requires a cast. Be careful with overflow: integer overflow does not automatically throw an exception.

### Q5. What is the difference between widening and narrowing conversions?

Widening conversions generally move to a type with a broader range or precision, such as int to long, and are often implicit. Narrowing conversions can lose information, such as long to int or double to int, and usually require an explicit cast. An explicit cast does not guarantee the value fits; validate ranges when data loss is unacceptable.

### Q6. What is the difference between final, finally, and finalize?

final restricts reassignment, overriding, or inheritance depending on where it is applied. finally is a control-flow block commonly used for cleanup. finalize was an object cleanup mechanism that is deprecated and should not be relied on; it is unpredictable and unsuitable for resource management. Use try-with-resources for closeable resources.

### Q7. What does static mean for fields, methods, and nested classes?

A static field belongs to the class rather than each instance, and a static method is invoked with class-level context rather than an implicit this reference. A static nested class does not require an enclosing instance. Static members can be useful for constants and stateless utilities, but mutable global static state complicates testing, concurrency, and lifecycle management.

### Q8. Can a static method be overridden?

No. Static methods are hidden rather than overridden. The method selected depends on the compile-time type used to refer to the class, whereas overridden instance methods use dynamic dispatch based on the runtime object. This distinction matters in output questions and is a reason not to model polymorphic behavior with static methods.

### Q9. What is the difference between this and super?

this refers to the current instance and can invoke another constructor in the same class. super accesses superclass members or invokes a superclass constructor. A constructor delegation call using this(...) or super(...) has placement rules; modern Java has relaxed some constructor-body restrictions, but code must still follow the target release's rules.

### Q10. Why are constructors not inherited or overridden?

A constructor initializes an instance of the class in which it is declared; it is not an ordinary inherited method. A subclass constructor must arrange for superclass initialization, directly or through another constructor. Constructors can be overloaded, but not overridden, because constructor selection occurs as part of object creation rather than virtual method dispatch.

### Q11. What is covariant return typing?

An overriding method may return a subtype of the return type declared by the overridden method. This allows an override to return a more specific result without forcing callers to cast. Parameter types do not work the same way: changing parameter types creates an overload rather than a valid override.

### Q12. Why are enums often better than string constants?

Enums provide a closed set of typed values, can hold methods and fields, and work naturally with switch statements and collections. They reduce invalid values caused by typos and make the domain explicit. Do not persist enum ordinal values as a durable external contract because declaration order can change; use stable names or explicit codes when needed.

---

---

## 2. Object-oriented programming, contracts, and design

### Q13. What is the difference between an interface and an abstract class?

An interface defines a contract that unrelated classes can implement, and a class can implement multiple interfaces. Interfaces may include abstract methods, default methods, static methods, and private helper methods; they do not provide per-instance state in the way ordinary classes do.

An abstract class can hold instance state, constructors, and shared protected or public implementation, but a class can extend only one class. Prefer interfaces for capabilities and replaceable contracts. Use an abstract base class when closely related types genuinely share state or implementation that belongs in a common inheritance hierarchy. Neither is automatically better; avoid inheritance solely to reuse a few methods.

---

### Q14. Explain encapsulation, abstraction, inheritance, and polymorphism with practical meaning.

**Encapsulation** protects an object's invariants by controlling access to state and behavior. Private fields alone are not enough if methods expose mutable internals or allow invalid state transitions. **Abstraction** exposes a useful contract while hiding details callers do not need. **Inheritance** models a valid subtype relationship and reuses or specializes behavior. **Polymorphism** lets callers use a common type while the runtime dispatches an overridden instance method to the implementation appropriate for the actual object.

In backend code, an interface such as `PaymentProvider` can hide whether the implementation calls one external processor or another. That is useful abstraction and polymorphism. The design is only sound if implementations honor the same behavioral contract; a subtype that violates caller expectations is not a good substitute merely because it compiles.

### Q15. What is the difference between overloading and overriding?

**Overloading** means declaring methods with the same name but different parameter lists. The compiler selects the overload using the compile-time types and overload-resolution rules. Return type alone cannot distinguish overloads.

**Overriding** means a subtype provides an implementation of an inherited instance method with a compatible signature and return type. For ordinary virtual instance methods, the implementation is selected at runtime based on the actual object's class. Static methods are hidden rather than overridden, and fields are hidden rather than dynamically dispatched. This is why a reference's declared type can determine which field or static method is accessed while the object's class determines the overridden instance method.

### Q16. What are SOLID principles, and how should they influence design?

- **Single Responsibility:** A module should have one coherent reason to change.
- **Open/Closed:** Extend behavior without repeatedly modifying stable, well-tested core logic where practical.
- **Liskov Substitution:** A subtype must honor the behavioral expectations of its base type.
- **Interface Segregation:** Clients should not depend on methods they do not need.
- **Dependency Inversion:** High-level policy should depend on abstractions rather than directly on low-level implementation details.

SOLID is a set of design heuristics, not a requirement to create an interface for every class. Applying it well reduces change coupling and improves testability. Applying it mechanically can produce excessive indirection, fragmented interfaces, and abstractions that have only one trivial implementation. Use the principles to explain concrete design decisions and trade-offs.

### Q17. Why are immutability and defensive copying useful?

An immutable object cannot be changed after construction, so its invariants are easier to reason about and its state can be shared without coordinating later mutations. Immutability is especially useful for value objects, configuration, map keys, and data crossing concurrency boundaries.

A class must protect its internal state, not merely declare fields `final`. A final reference can still point to a mutable list. Copy mutable inputs on the way in and avoid returning mutable internal objects directly; expose immutable views or copies where appropriate. Deep immutability may require copying nested mutable values too, so understand ownership rather than relying on a shallow copy.

---

### Q18. Which design patterns are especially useful in Java backend development?

Patterns are reusable ways to structure recurring design problems, not a checklist to apply everywhere. **Strategy** makes an algorithm replaceable behind a common contract; **Factory** centralizes object creation when construction choices vary; **Decorator** adds behavior around an object while preserving its interface; **Builder** simplifies construction when an object has many optional parameters; and **Adapter** translates one interface into another.

Spring itself uses familiar patterns such as proxies, factories, templates, and dependency injection. The interview answer should connect a pattern to the problem it solves and its trade-off. For example, Strategy can remove a large conditional by selecting an implementation, but it can also create unnecessary classes if the variation is trivial and unlikely to grow.

### Q19. How should you reason about Java API design for nullability?

Decide whether absence is valid for each parameter, field, and return value, then document and enforce that contract. Use validation at boundaries and return meaningful errors instead of allowing a null to travel through multiple layers until an unrelated dereference fails. Optional can communicate an optional return value, but it is not automatically the right type for every field or parameter.

Avoid mixing conventions without explanation. In large codebases, annotations and static analysis can make nullability contracts more visible, but they do not replace clear API design and runtime validation where untrusted input is involved.

### Q20. How do you make a Java object immutable in practice?

Make the class's state private, initialize it completely during construction, avoid setters, and prevent mutable internal objects from escaping. Use defensive copies for mutable input and output collections. Final fields help express initialization guarantees but do not make a referenced list or date-like mutable object immutable by themselves.

### Q21. What is the difference between an abstract class and an interface in modern Java?

An abstract class can hold instance state, constructors, and a mix of abstract and concrete methods. An interface defines a type contract and can include abstract, default, static, and private methods, but it does not provide ordinary per-instance fields. Use an interface for capabilities and interchangeable implementations; use an abstract class when shared state or protected implementation is genuinely part of the design.

### Q22. What are default-method conflicts in interfaces?

If unrelated interfaces provide conflicting default implementations, a class may need to override the method and resolve the conflict explicitly. A class implementation takes precedence over an interface default, and a more specific interface can take precedence over a less specific one. Default methods support interface evolution but should not be used to hide ambiguous domain design.

### Q23. What is the difference between shallow copy and deep copy?

A shallow copy duplicates the outer object but may retain references to the same nested mutable objects. A deep copy duplicates the relevant nested state as well. Deep copying can be expensive and is domain-dependent; defensive copying of exposed mutable state is often a better targeted solution than indiscriminately copying an entire object graph.

### Q24. What is defensive copying, and where is it needed?

Defensive copying prevents callers from mutating internal state through references they supplied or received. Copy mutable constructor arguments before storing them and return a copy or unmodifiable representation when exposing mutable internal collections. An unmodifiable wrapper alone is not a snapshot if another reference can still mutate the backing collection.

### Q25. What is the Factory pattern and when is it useful?

A factory encapsulates object creation when the caller should not know the concrete implementation or construction rules. It is useful when selection depends on configuration, input type, or a family of implementations. A factory adds indirection, so it is not necessary for every simple constructor call.

### Q26. What is the Strategy pattern?

Strategy represents interchangeable behavior behind a common contract and selects an implementation at runtime. It can replace a large conditional when behaviors vary independently, such as choosing a pricing or retry policy. It is less useful when there is only one trivial behavior and no realistic variation.

### Q27. What is the Builder pattern and when should it be used?

Builder separates complex object construction from the final representation, especially when many optional parameters or validation rules exist. It can make call sites readable and reduce telescoping constructors. For a small value object with two required fields, a builder may add unnecessary ceremony.

### Q28. What is the Decorator pattern?

Decorator wraps an object with the same interface and adds behavior without modifying the original implementation. It is useful for composing concerns such as metrics, caching, compression, or additional validation. Many layers of decorators can complicate debugging, so make composition visible and test the order of behavior.

### Q29. What is the Adapter pattern?

Adapter translates one interface into another so components with incompatible contracts can work together. It is useful when integrating a legacy library or external provider without spreading its model across the application. Keep the adapter at the boundary and translate errors and data into the application's own contract.

### Q30. What is Dependency Injection, and how is it related to design patterns?

Dependency injection supplies an object with its collaborators rather than having it construct them internally. It improves testability and separates object creation from business behavior. DI is a technique, not a guarantee of good architecture; excessive interfaces or a service locator can still create hidden dependencies.

### Q31. What is the difference between composition and inheritance?

Inheritance models an is-a relationship and couples a subclass to superclass behavior and lifecycle. Composition builds behavior by holding collaborators and delegating to them, often making change and testing easier. Prefer composition when behavior should vary independently or the relationship is not a true subtype.

### Q32. What is the Liskov Substitution Principle in practical terms?

A subtype should be usable wherever its base type is expected without violating the base contract. A subtype that strengthens preconditions, weakens postconditions, or changes promised invariants can break callers even if it compiles. Design contracts before inheritance and use composition when behavior does not fit the parent abstraction.

### Q33. How should a Java API handle null arguments?

Define nullability explicitly, validate at public boundaries, and fail early with a meaningful exception when null is invalid. If absence is part of the normal result, represent it consistently, for example with Optional return values where appropriate. Do not let a null propagate through several layers and fail at an unrelated dereference.

### Q34. How do you review a Java pull request for correctness?

Review behavior and invariants, error handling, concurrency, nullability, resource lifecycle, security boundaries, observability, tests, and compatibility with the target Java release. Look for duplicated logic and unnecessary abstraction, but prioritize correctness and maintainability over stylistic preference. Ask what happens under empty input, duplicates, timeout, partial failure, and concurrent requests.

### Q35. How do you decide whether to introduce an interface?

Introduce an interface when it represents a meaningful contract, enables multiple implementations, separates a boundary, or improves testing without obscuring the design. Avoid creating one interface per class mechanically when no substitution or boundary exists. The right abstraction should clarify dependencies rather than add layers for their own sake.

---

---

## 3. Strings, wrappers, numeric behavior, and equality

### Q36. What is the difference between `==` and `equals()`?

For primitives, `==` compares values. For reference types, `==` checks whether both references identify the same object. `equals()` expresses logical equality according to the class's contract; the default implementation inherited from `Object` behaves like identity equality unless overridden.

For example, two distinct `String` objects can be equal according to `equals()` while `==` is false. Use logical equality for values and domain objects when their class defines it. Be careful with arrays: their inherited `equals()` does not compare contents; use `Arrays.equals` or `Arrays.deepEquals` as appropriate.

### Q37. Why is String immutable, and when should StringBuilder or StringBuffer be used?

A `String` cannot have its character content changed after construction. Immutability makes strings safe to share, supports string-pool reuse, simplifies their use as map keys, and reduces synchronization needs. Operations such as concatenation create a resulting string rather than modifying the original.

Use `StringBuilder` for repeated modifications within one thread, such as constructing output in a loop. `StringBuffer` synchronizes its individual methods, but that does not automatically make a multi-step workflow atomic. For simple concatenation expressions, the compiler may optimize construction; choose based on the actual operation rather than assuming every `+` is expensive.

### Q38. What is the contract between equals() and hashCode()?

If two objects are equal according to `equals()`, they must return the same `hashCode()`. Unequal objects may share a hash code, although a good implementation aims to distribute values well. The hash code must remain consistent while the information used by equality remains unchanged.

Hash-based collections use the hash to find a candidate bucket and equality to identify the key. If equality is overridden without a compatible `hashCode()`, lookups can fail even for logically equal keys. Mutable fields used in equality or hashing are particularly dangerous: changing them after insertion can make an entry difficult to find or remove.

### Exception handling and API contracts

### Q39. What is the difference between String, StringBuilder, and StringBuffer?

`String` is immutable: every apparent modification produces a new string value. That makes strings safe to share and useful as map keys, but repeated concatenation inside a loop can create many temporary objects. `StringBuilder` is mutable and usually the right choice for building text in one thread. `StringBuffer` offers synchronized methods for legacy thread-safe mutation, but synchronization does not automatically make a multi-step sequence of operations atomic.

Use `+` for simple readable concatenation; the compiler can optimize many such expressions. Prefer a builder when accumulating text repeatedly in a loop or when construction logic is complex. Do not choose `StringBuffer` by default merely because a service is multithreaded: a builder should generally be local to the operation rather than shared between threads.

### Q40. How does the String pool work, and what does intern() do?

The runtime maintains a pool of canonical strings. String literals and compile-time constant string expressions are interned, so equal literals can refer to the same pooled object. Calling `intern()` returns the canonical pooled representation for that string value, either an existing one or a pooled entry.

This is different from value equality: use `equals()` to compare string contents. Do not rely on reference identity as an application-level string comparison strategy. Interning arbitrary, high-cardinality input can also add complexity and memory pressure without a useful benefit; use it only when the workload and lifecycle make canonicalization worthwhile.

### Q41. Why is String a good key for HashMap, and what happens if a key is mutable?

A hash-based map expects a key's equality and hash code to remain stable while it is stored. String is immutable, so its content cannot change after insertion and invalidate the bucket chosen by the map. A mutable key can be inserted under one hash code and later searched for under another, making the entry effectively unreachable through normal lookup even though it remains in the map.

If a custom key represents several fields, base equality and hashing on the same logical identity fields and make those fields immutable where practical. If mutability is unavoidable, remove the key before changing its equality-relevant state and reinsert it afterward.

### Q42. Explain autoboxing, unboxing, and a common Integer comparison trap.

Autoboxing converts a primitive to its wrapper type, such as `int` to `Integer`; unboxing extracts the primitive value. Java guarantees caching for certain small integral wrapper values, but code must not depend on two boxed references being identical for arbitrary values. Therefore, `a == b` can compare references when both operands are wrappers, while `a.equals(b)` compares wrapper values.

Unboxing a null wrapper throws `NullPointerException`. Be explicit about nullability at API boundaries, and use primitive types when null has no meaning. For value comparisons, unbox safely or use the wrapper's value equality; don't use reference identity.

### Q43. What should you know about floating-point equality and BigDecimal?

Binary floating-point types such as `float` and `double` cannot represent many decimal fractions exactly. This can produce results such as a sum that is close to, but not exactly, the expected decimal value. For approximate scientific calculations, use a documented tolerance appropriate to the scale and domain rather than direct equality.

For decimal quantities requiring predictable decimal arithmetic, such as many financial calculations, `BigDecimal` is often appropriate. Constructing it from a decimal string avoids importing the binary approximation of a double. Also note that `BigDecimal.equals()` considers both value and scale, whereas `compareTo()` compares numerical value; this distinction matters for equality and sorted collections.

### Q44. What is the difference between equals(), hashCode(), and compareTo() contracts?

`equals()` defines logical equality, and equal objects must have equal hash codes. `compareTo()` defines ordering and returns zero when values are considered equal under that ordering. The `Comparable` contract recommends consistency with `equals()`, though it does not strictly require it in all cases.

Hash-based collections use equality and hashing; sorted collections use comparison. Inconsistent contracts can cause a value to appear present in one collection but behave differently in another. For domain objects, choose stable identity fields and test the contracts together.

### Q45. Why can String.length() differ from the number of visible characters?

Java String stores UTF-16 code units, so `length()` reports the number of UTF-16 code units, not necessarily Unicode code points or user-perceived characters. A character outside the Basic Multilingual Plane uses a surrogate pair and therefore occupies two code units. Some visible symbols are also made from multiple code points, such as a base character plus a combining mark or a joined emoji sequence.

For simple ASCII-oriented identifiers, `length()` often matches expectations. For Unicode-aware processing, use code-point APIs such as `codePointCount` and be clear whether the requirement concerns code units, code points, or grapheme clusters.

### Q46. What are common pitfalls with String.split(), trim(), strip(), and isBlank()?

`split()` accepts a regular expression, not a literal delimiter, and its default behavior discards trailing empty fields. Use a quoted delimiter or an appropriate regex when the delimiter has special meaning, and use a negative limit when trailing empty fields must be preserved. `trim()` removes a narrower range of leading and trailing characters than Unicode-aware `strip()`; `isEmpty()` checks whether length is zero, while `isBlank()` also recognizes strings containing only whitespace.

These methods have different semantics, so choose based on the input format and Java version. In data processing, losing trailing empty columns can silently corrupt records; write a test for empty input, repeated delimiters, trailing delimiters, and non-ASCII whitespace.

### Q47. What are the rules for equals() symmetry and transitivity?

Equality must be reflexive, symmetric, transitive, consistent, and false for null. A common mistake is implementing equality in a superclass and then extending it with extra equality fields in a subclass, which can violate symmetry. Prefer well-defined value types and avoid inheritance structures that make equality semantics ambiguous.

### Q48. Why should hashCode use the same logical fields as equals?

If two objects are equal according to equals, they must return the same hash code. Hash collisions are allowed, so unequal objects can share a hash code. When equality-relevant fields change while an object is a map key, lookup can fail. Base both methods on stable identity fields and test them together.

### Q49. What is the difference between == and equals for arrays?

Array equality with == compares whether both references point to the same array object. Arrays.equals compares elements for one-dimensional arrays, and Arrays.deepEquals supports nested arrays. Arrays also inherit Object's equals implementation, so two arrays with identical contents are not equal by default under ordinary equals calls.

### Q50. What are common pitfalls with BigDecimal?

Constructing BigDecimal from a double captures the double's binary approximation; use a string or an appropriate exact constructor when decimal intent matters. equals considers both numerical value and scale, while compareTo compares numerical magnitude. Choose a consistent representation and rounding policy, particularly for monetary calculations.

### Q51. Why can == behave unexpectedly with boxed numbers?

When both operands are wrapper references, == compares object identity. Some wrapper values are cached, so certain comparisons may appear true while other values are false. Use value equality or unbox safely, and guard against null before unboxing to avoid NullPointerException.

---

---

## 4. Exception handling and resource safety

### Q52. What is the difference between checked and unchecked exceptions?

Checked exceptions must be caught or declared by the method signature, except for the special treatment of unchecked exceptions. They often represent recoverable conditions that callers are expected to consider, such as certain I/O failures. Unchecked exceptions are subclasses of `RuntimeException` or `Error`; runtime exceptions often indicate invalid arguments, illegal state, or programming errors, while `Error` generally signals serious conditions an application should not casually try to recover from.

The choice is an API-design decision, not simply “checked is good, unchecked is bad.” Use meaningful exception types, preserve the cause when wrapping, and avoid catching an exception unless the code can recover, translate it usefully, add context, or handle it at an appropriate boundary. Do not swallow failures or use exceptions as ordinary control flow.

### Q53. When should you create a custom exception?

Create a custom exception when it communicates a meaningful domain or boundary failure that callers can handle differently, such as an invalid business operation or a missing domain resource. Include useful context, preserve the original cause when wrapping, and choose checked versus unchecked based on the contract rather than habit.

Avoid creating a new exception class for every low-level failure if callers cannot act on the distinction. Do not expose secrets, raw credentials, or sensitive payloads in exception messages. At service boundaries, translate implementation-specific exceptions into stable domain or API errors while retaining enough internal diagnostic context.

### Q54. What is exception chaining, and why should the cause be preserved?

Exception chaining wraps a lower-level exception in a higher-level exception while retaining the original cause. For example, a repository layer might translate a driver exception into a persistence-specific exception. The higher layer can then reason about the abstraction it owns, while logs and diagnostics retain the original failure and stack trace.

When wrapping, pass the original exception as the cause. Avoid logging the same exception at every layer and then rethrowing it, because that creates noisy duplicate logs. Log where the failure can be meaningfully handled or where ownership of the error is clear.

### Q55. How do try-with-resources and suppressed exceptions work?

Try-with-resources automatically closes resources that implement `AutoCloseable`, including when the body throws. Resources are closed in reverse order of initialization. If the body throws and closing also throws, the body exception remains primary and the close failure is recorded as a suppressed exception.

This is usually safer than manual `finally` cleanup, where a close exception can accidentally replace the original failure. Use it for files, streams, JDBC resources, and other closeable objects. When diagnosing failures, inspect suppressed exceptions too; the primary stack trace may not tell the whole story.

### Q56. Can a finally block change a method's return value or hide an exception?

Yes, but it should not. A `return` or a thrown exception from `finally` can override a return or exception from the `try` or `catch` block, which makes control flow surprising and can hide the original failure. The safe rule is to use `finally` for cleanup only and avoid returning or throwing from it unless there is a very specific, justified reason.

Try-with-resources is generally preferable for closeable resources because it preserves the primary exception and records close failures as suppressed exceptions.

### Q57. What is the difference between checked exceptions and unchecked exceptions in API design?

Checked exceptions require callers to catch or declare them, making recoverable failure possibilities visible in a method signature. Unchecked exceptions are often used for programming errors or failures that callers cannot reasonably recover from locally. Choose based on the abstraction's contract, preserve causes when wrapping, and avoid catching exceptions only to log and rethrow unchanged.

### Q58. Can a catch block catch an exception thrown by another catch block?

No. A catch block handles exceptions from the associated try block; an exception thrown while executing a catch block is not handled by a sibling catch clause. It propagates outward unless another enclosing handler catches it. This is useful to remember when writing exception output questions and nested error-handling logic.

### Q59. What is multi-catch, and what restrictions apply?

Multi-catch allows one catch block to handle several unrelated exception types using a pipe-separated declaration. The alternatives cannot be related by subclassing, because one would be redundant. The multi-catch parameter is effectively final, which prevents reassignment and keeps the declared union type safe.

### Q60. What is the difference between throw and throws?

throw actually raises an exception object at a point in execution. throws declares exception types that a method may propagate as part of its signature. A declaration does not itself throw anything; it describes the method contract and affects compile-time checking for checked exceptions.

### Q61. What is the difference between a checked exception and an Error?

Exceptions generally represent conditions application code may handle or propagate. Error represents serious problems associated with the runtime or environment, such as linkage failures or resource exhaustion, and application code usually should not attempt to recover from arbitrary Errors. Cleanup in finally or try-with-resources still matters during abrupt termination, but do not swallow fatal conditions indiscriminately.

### Q62. What is try-with-resources and why is it preferred?

Try-with-resources automatically closes AutoCloseable resources when the scope exits, including when an exception occurs. Resources close in reverse declaration order. If the body throws and closing also throws, the body exception remains primary and close failures are recorded as suppressed exceptions.

### Q63. What happens when a finally block returns a value?

A return or throw from finally can override a return or exception from the try or catch block, hiding the original outcome. Avoid returning or throwing from finally. Use finally for cleanup, or try-with-resources for AutoCloseable resources so the primary exception is preserved and close failures are suppressed.

---

---

## 5. Generics, type system, annotations, and reflection

### Q64. What are generics, and what is type erasure?

Generics let APIs express relationships between types at compile time, reducing casts and making invalid combinations harder to represent. For example, `List<String>` communicates that the list is intended to contain strings.

Java implements most generic type information through **type erasure**: type parameters are erased to their bounds or `Object` in the runtime representation, with compiler-generated casts and bridge methods where needed. This is why `new T()` is not generally allowed, `List<String>` and `List<Integer>` cannot be distinguished using `instanceof` at runtime in the ordinary way, and generic types are invariant. Wildcards such as `? extends T` and `? super T` express safe producer/consumer relationships.

### Q65. What is the difference between List<? extends T> and List<? super T>?

`? extends T` means the list's element type is some unknown subtype of T. It is useful when reading T values from a producer, but you generally cannot add a non-null T because the actual list could be a list of a narrower subtype.

`? super T` means the element type is T or one of its supertypes. It is useful when writing T values to a consumer, though values read back are only safely known as `Object`. This is the basis of PECS: **Producer Extends, Consumer Super**. It is a useful guideline rather than a substitute for understanding the exact type relationship.

### Q66. What are reflection, annotations, and the Java module system used for?

Annotations attach metadata to program elements; reflection can inspect classes, methods, fields, and annotations at runtime. Frameworks use these mechanisms for dependency injection, mapping, serialization, and test discovery. Annotation processing can also inspect source-level metadata at compile time and generate code before runtime.

The Java Platform Module System (JPMS), introduced in Java 9, adds explicit module dependencies and package exports/opens. Strong encapsulation means reflective access to non-open packages can be restricted, even when code previously relied on deep reflection. Use reflection deliberately, account for module boundaries, and prefer public APIs or compile-time processing when that is simpler and safer.

### Q67. Why can't you create arrays of parameterized types, and what is reifiable?

Java arrays are covariant and enforce their component type at runtime, while generic type parameters are mostly erased at runtime. Creating a `new List<String>[10]` would imply runtime checks that the JVM cannot reliably perform for the erased element type, so generic array creation is prohibited in the usual form.

A reifiable type is one whose runtime type information is fully available, such as a non-generic class or an array of reifiable component type. Generic code sometimes needs a collection instead of an array or a carefully encapsulated varargs operation with appropriate safety guarantees. Do not suppress unchecked warnings without understanding the invariant that makes the operation safe.

### Q68. What are bounded type parameters and generic methods useful for?

A bounded type parameter restricts the types that can be supplied and makes a useful contract available inside the implementation, such as requiring a type to implement `Comparable`. A generic method declares its own type parameter, allowing the same method to work safely across multiple related types without relying on raw types or casts.

Prefer a type signature that expresses the actual relationship between inputs and outputs. Avoid raw collections: they bypass compile-time checks and can cause failures far from the point where an unsafe value entered the system.

### Q69. What is reflection, and what are its trade-offs?

Reflection lets code inspect classes, methods, fields, constructors, and annotations at runtime, and in some cases invoke or access them dynamically. Frameworks use it for dependency injection, serialization, mapping, and test tooling. The trade-offs include weaker compile-time guarantees, more complex error handling, possible performance costs, and access restrictions under the module system.

Use reflection when runtime discovery is a real requirement, not as a default replacement for ordinary interfaces and polymorphism. Cache reflective metadata when justified by profiling, handle missing members deliberately, and test behavior against the target JDK and module configuration.

### Q70. What are annotations and annotation processing used for?

Annotations attach metadata to declarations and can be consumed by the compiler, build tools, runtime reflection, or frameworks. Retention policy determines whether an annotation is available only in source, in class files, or at runtime. Annotation processors can inspect source-level elements and generate additional source or resources during compilation.

Annotations do not automatically cause behavior; some tool or runtime component must interpret them. When debugging framework behavior, check the annotation's retention, target, processor configuration, and whether the relevant framework actually scans that class or package.

### Q71. What is type erasure, and what limitations does it create?

Java generics are largely implemented by erasing type parameters during compilation and inserting casts or bridge methods where required. This supports compatibility with older bytecode, but means some runtime operations cannot distinguish parameterized types such as List<String> and List<Integer>. It also explains restrictions such as generic array creation and many parameterized instanceof checks.

### Q72. What are bridge methods?

The compiler can generate synthetic bridge methods to preserve polymorphism after type erasure or covariant return changes. They help ensure that overriding behavior remains correct when generic source-level signatures erase to compatible JVM-level signatures. They are generally invisible in normal source code but may appear in reflection or bytecode inspection.

### Q73. What is the difference between extends and super wildcards?

Use ? extends T when a generic structure produces values of type T or a subtype and you primarily need to read them. Use ? super T when it consumes values of type T and you need to write them. The PECS mnemonic—Producer Extends, Consumer Super—is a useful starting point, but the actual API contract determines the correct type.

### Q74. Why are generics invariant when arrays are covariant?

A List<Integer> is not a subtype of List<Number>, because otherwise code could insert a Double into a list intended to contain Integer values. Arrays allow covariance but enforce component types at runtime, which can produce ArrayStoreException. Generic invariance catches unsafe assignments at compile time; wildcards express safe producer or consumer relationships.

### Q75. What are raw types, and why should they be avoided?

A raw type uses a generic class without its type argument, such as List rather than List<String>. It bypasses generic type checking and can introduce values that fail later with ClassCastException. Use parameterized types throughout modern code and isolate any unavoidable raw-type interaction behind a narrow, documented boundary.

### Q76. What is the difference between a generic method and a generic class?

A generic class declares type parameters that apply to its instance or class-level API. A generic method declares its own type parameters for one method invocation, even if its enclosing class is not generic. Use the narrowest type parameter scope that expresses the relationship between the method's inputs and outputs.

### Q77. What is an annotation's retention policy?

SOURCE retention means the annotation is discarded by the compiler, CLASS retention keeps it in the class file but does not require runtime reflection visibility, and RUNTIME retention makes it available to runtime reflection. Frameworks that inspect annotations at runtime need appropriate retention. Annotation metadata alone does not trigger behavior; a compiler, processor, or runtime component must interpret it.

### Q78. What is the difference between reflection and annotation processing?

Reflection inspects runtime classes and members while the program is running. Annotation processing operates during compilation and can inspect source-level declarations and generate source or resources. Compile-time processing can catch or generate things earlier, while reflection supports runtime discovery but adds runtime complexity and access constraints.

### Q79. What is a dynamic proxy, and where is it used?

A dynamic proxy can create an object that implements specified interfaces and forwards method calls to an invocation handler. Frameworks use proxies for cross-cutting behavior such as transactions, security, and interception. Proxy-based mechanisms have limitations around concrete classes, final methods, self-invocation, and method visibility depending on the proxy technology.

---

---

## 6. Collections framework and data structures

### Q80. How does HashMap work internally?

A `HashMap` uses a key's hash information to locate a bucket and then uses equality to find the matching key among candidates. Multiple keys can map to the same bucket, so a hash collision is not the same as key equality. The implementation resizes the table as it grows, redistributing entries to maintain useful lookup performance.

Average lookup and insertion are commonly described as O(1) under reasonable hash distribution; this is not an unconditional worst-case guarantee. Modern OpenJDK implementations can use tree bins for heavily populated buckets under certain capacity and threshold conditions. Treat those internal thresholds as implementation details rather than portable API guarantees. Pre-sizing may reduce resizing when the expected size is known, but excessive capacity wastes memory.

### Q81. Why must HashMap keys have stable equals() and hashCode() values?

A map uses the key's hash to find where its mapping belongs. If fields used by `hashCode()` or `equals()` change after insertion, a later lookup can search a different bucket or fail to match the stored key. The mapping may still physically exist but become practically inaccessible through the mutated key.

Prefer immutable keys such as strings, enums, records whose components are themselves suitably immutable, or dedicated value objects. If mutable objects must be used, ensure the fields participating in equality and hashing do not change while the object is a key.

### Q82. Compare ArrayList and LinkedList.

`ArrayList` stores elements in a resizable array. Indexed reads are O(1), and appending is amortized O(1); inserting or removing near the beginning or middle usually shifts elements and costs O(n). It has good memory locality and is generally the default choice for list workloads.

`LinkedList` stores nodes linked together. Inserting or removing a node is O(1) once the position/node is known, but finding an index or position is O(n), and each node adds memory and pointer-chasing overhead. As a result, LinkedList is not automatically faster for frequent insertions. Choose from the actual access pattern; for most ordinary list use, ArrayList is the more practical default.

### Q83. Compare HashSet, LinkedHashSet, and TreeSet.

`HashSet` provides uniqueness according to `equals()` and `hashCode()`, with no iteration-order guarantee. `LinkedHashSet` preserves a defined encounter order, typically insertion order, at additional memory cost. `TreeSet` keeps elements sorted using natural ordering or a comparator, typically with O(log n) basic operations.

For a sorted set, ordering must be consistent with the intended notion of uniqueness. If a comparator returns zero for two values, TreeSet treats them as the same set element even if their `equals()` methods say otherwise. Pick the implementation based on uniqueness, ordering, and performance needs.

### Q84. What is the difference between HashMap and ConcurrentHashMap?

`HashMap` is not thread-safe for concurrent structural modification. Concurrent access that includes writes requires external coordination or a concurrent collection. `ConcurrentHashMap` is designed for concurrent access and provides thread-safe operations with implementation techniques that allow more concurrency than locking the entire map for every operation.

Use atomic methods such as `computeIfAbsent`, `compute`, `merge`, or `putIfAbsent` when the operation must combine lookup and update safely. A sequence like `if (!map.containsKey(k)) map.put(k, v)` is not atomic just because the map itself is concurrent. Also, thread safety of the map does not make mutable objects stored as values thread-safe.

### Q85. What are fail-fast iterators, and can they guarantee thread safety?

Some collection iterators detect structural modification outside the iterator's supported mutation methods and may throw `ConcurrentModificationException`. This is a best-effort bug-detection mechanism, not a synchronization guarantee. Code must not depend on the exception always occurring.

For single-threaded removal during iteration, use the iterator's `remove()` when supported or use appropriate collection methods. For concurrent workloads, choose a concurrent collection or coordinate access explicitly. For example, `CopyOnWriteArrayList` provides snapshot-style iteration that is useful for read-heavy, write-light workloads but can be expensive when writes are frequent.

### Q86. When should a Queue, Deque, PriorityQueue, or BlockingQueue be used?

A `Queue` represents elements waiting to be processed; a `Deque` supports operations at both ends and can act as a queue or stack. A `PriorityQueue` retrieves elements according to priority/order rather than insertion order, so it is useful for scheduling or selecting the next minimum/maximum. A `BlockingQueue` supports producer-consumer coordination by waiting when appropriate conditions are not met.

Choose based on the ordering and coordination contract. A priority queue is not a sorted list for iteration, and a thread-safe queue alone does not bound the total work accepted by a service. In production, capacity, rejection/backpressure, and shutdown behavior matter as much as the data structure.

---

### Q87. How do HashMap collisions work, and why does capacity matter?

A hash map uses a key's hash to select a bucket and then uses equality to distinguish keys that land in the same bucket. Collisions are expected; a correct map must still distinguish unequal keys with the same hash. In modern OpenJDK implementations, heavily populated buckets can be treeified under certain capacity and collision conditions, but the exact implementation details are not all part of the general Map contract.

Resizing occurs as the map grows beyond its threshold, redistributing entries across a larger table. If an approximate entry count is known, sizing the map sensibly can reduce resize work, but excessive preallocation wastes memory. Correct `equals()` and `hashCode()` implementations matter more than trying to avoid every collision.

### Q88. How do HashMap, LinkedHashMap, and TreeMap differ?

`HashMap` provides hash-based lookup and does not promise iteration order. `LinkedHashMap` maintains a predictable encounter order, usually insertion order, and can also maintain access order for patterns such as a simple LRU cache. `TreeMap` keeps keys ordered according to their natural ordering or a supplied comparator, with logarithmic-time basic operations.

Choose based on the contract you need: no order, predictable encounter order, or sorted keys. A comparator used by a sorted map should be consistent with equality when practical; otherwise the map's notion of duplicate keys can differ from `equals()`, which can surprise callers.

### Q89. What is the difference between Comparable and Comparator?

`Comparable` defines a type's natural ordering through `compareTo()`. `Comparator` defines an external ordering and allows multiple orderings for the same type, such as sorting employees by name, salary, or start date. Comparators can be composed using methods such as `comparing()` and `thenComparing()`.

Ordering must be transitive and consistent. In sorted sets and maps, comparison returning zero means the elements are treated as the same ordering key, even if their `equals()` methods say they differ. Document intentional differences and test boundary values, null handling, and tie-breakers.

### Q90. What are the trade-offs of CopyOnWriteArrayList and BlockingQueue?

`CopyOnWriteArrayList` copies its backing array on mutation, so reads and iteration are simple and iterators observe a snapshot. It works well for small collections with many reads and rare writes, but frequent updates can be expensive in allocation and copying. It is not a general replacement for `ArrayList`.

A `BlockingQueue` supports coordination between producers and consumers, including blocking when a bounded queue is full or empty. Bounded queues can provide backpressure and prevent unbounded buffering, but they require an explicit policy for saturation: block, reject, drop, or apply another domain-specific response.

### Q91. What is the difference between an unmodifiable collection and an immutable collection?

An unmodifiable view prevents mutation through that view, but the underlying collection may still be changed elsewhere. A truly immutable collection cannot have its contents changed after creation, although objects stored inside it may themselves be mutable. An unmodifiable wrapper around a mutable list is therefore not automatically a snapshot.

When exposing collection state from an object, consider defensive copying or immutable collection factories so callers cannot mutate internal state through an alias. Also consider whether element mutability can violate the class's invariants.

### Q92. What does computeIfAbsent() solve, and what should you avoid inside its mapping function?

`computeIfAbsent(key, mappingFunction)` atomically associates a computed value when the key is absent under the map's contract, which is useful for patterns such as building a list per key or caching a result. It is safer than a separate contains/get followed by put when the map is concurrent and the operation needs atomic coordination.

Keep the mapping function short and avoid recursively modifying the same map or performing slow, blocking work inside it. Concurrent map implementations can impose specific restrictions on the mapping function; consult the relevant API contract rather than assuming all Map implementations behave identically.

### Q93. What is IdentityHashMap, and when would you use it?

`IdentityHashMap` compares keys using reference identity (`==`) rather than logical equality (`equals()`). This makes it useful for specialized tasks such as tracking object identity during graph traversal, serialization-like processing, or cycle detection where two equal-but-distinct objects must remain separate.

It is not a general replacement for `HashMap`. Most application maps should use logical equality, and callers should be told explicitly if identity semantics are required because the behavior can be surprising for strings, records, and value objects.

### Q94. What is the difference between Collection and Collections?

Collection is the root interface for many collection types, such as List, Set, and Queue. Collections is a utility class containing static methods for operations such as sorting, searching, synchronization wrappers, and unmodifiable views. Map belongs to the collections framework but does not extend Collection because it represents key-value associations.

### Q95. What is the difference between List, Set, and Queue?

List is ordered and allows duplicates; Set models uniqueness; Queue models elements waiting for processing and defines ordering/removal behavior according to its implementation. Choose based on the required contract, not just familiarity. A Deque can support both queue and stack operations.

### Q96. Why is ArrayList usually preferred over LinkedList?

ArrayList provides constant-time indexed access and good cache locality, while insertion or deletion in the middle may require shifting elements. LinkedList has costly indexed access and allocates a node per element; insertion is only cheap when the position is already known through an iterator or node reference. In most ordinary application workloads, ArrayList is the simpler default.

### Q97. How does ArrayList grow, and why should capacity be considered?

ArrayList maintains a backing array and grows it when capacity is insufficient, copying existing references to a larger array. Growth is amortized efficient, but an individual expansion can be costly. If the approximate size is known, pre-sizing can reduce repeated growth; do not allocate huge capacity without evidence because it consumes memory.

### Q98. What is the difference between HashSet and TreeSet?

HashSet uses hashing and does not promise sorted iteration. TreeSet maintains sorted order according to natural ordering or a comparator, generally with logarithmic basic operations. TreeSet treats elements that compare as zero as duplicates, so inconsistent ordering can produce behavior that differs from equals-based collections.

### Q99. What is LinkedHashMap useful for?

LinkedHashMap maintains a predictable iteration order, commonly insertion order, and can also use access order. Access-order mode is useful as a building block for a simple LRU-style cache, but production caches still need concurrency policy, expiration, capacity, and memory behavior to be designed deliberately.

### Q100. What is the difference between Iterator and ListIterator?

Iterator traverses forward and can remove elements where supported. ListIterator is specific to lists and supports bidirectional traversal, index information, and element replacement or insertion where supported. Structural modifications made outside the iterator can invalidate fail-fast iterators; those checks are best-effort bug detection, not synchronization.

### Q101. What is the difference between fail-fast and weakly consistent iteration?

Fail-fast iterators may throw ConcurrentModificationException when they detect structural modification outside the supported iterator operations, but this is not guaranteed thread safety. Weakly consistent iterators from some concurrent collections can tolerate concurrent updates and reflect some state without throwing that exception. Understand the concrete collection's contract rather than assuming all iterators behave alike.

### Q102. Why is ConcurrentHashMap safer than HashMap for concurrent access?

ConcurrentHashMap supports concurrent retrieval and updates with documented thread-safety properties and atomic methods such as computeIfAbsent. HashMap does not support unsynchronized concurrent structural modification. ConcurrentHashMap disallows null keys and values, which avoids ambiguity in concurrent lookup results and should be considered in API design.

### Q103. How do you make a check-then-act operation atomic?

Prefer one atomic collection method, such as putIfAbsent or computeIfAbsent, when it expresses the intended state transition. A separate containsKey call followed by put can race because another thread may change the map between operations. If the invariant spans several objects or operations, a broader lock or transaction may be necessary.

---

### Q104. What is the difference between an unmodifiable view and a snapshot?

An unmodifiable view prevents changes through that reference but can reflect changes made through another reference to the backing collection. A snapshot is a separate copy of the state at a point in time. Use a defensive copy when the API promises stable state rather than merely preventing mutation through one handle.

---

## 7. Functional programming, Optional, date/time, and everyday APIs

### Q105. What are Optional's intended uses and limitations?

`Optional<T>` models a value that may be present or absent and can make return-value absence explicit. Methods such as `map`, `flatMap`, `orElse`, `orElseGet`, and `orElseThrow` support composition without repeated null checks.

A common trap is that `orElse(expression)` evaluates its argument eagerly, while `orElseGet(supplier)` evaluates the supplier only when the Optional is empty. Optional is generally most useful as a return type; using it indiscriminately for fields, method parameters, or collections of optional values can complicate APIs and serialization. It does not replace validation or exception handling for operational failures.

### Q106. What is the difference between a Predicate, Function, Consumer, and Supplier?

A `Predicate<T>` accepts a value and returns a boolean, commonly for filtering. A `Function<T,R>` transforms a value into another value. A `Consumer<T>` accepts a value and returns no result, often for an action or side effect. A `Supplier<T>` produces a value without taking an input.

These interfaces make behavior composable and are used by streams, collection APIs, and asynchronous APIs. Keep functions passed to stream pipelines focused and avoid hidden side effects, because lazy evaluation and possible parallel execution can make side-effect timing and ordering harder to reason about.

### Q107. Why are side effects in stream pipelines risky?

Streams describe a computation over a source; intermediate operations are lazy, and a terminal operation triggers evaluation. Side effects inside intermediate operations can be skipped when they do not affect the result, run in an unexpected order, or race when a pipeline is parallel. For example, mutating a shared list inside a parallel `forEach` is not automatically safe.

Prefer transformations and reductions that express the result directly. If the operation is inherently side-effecting, make the boundary explicit and use a suitable concurrency or transaction strategy rather than assuming the stream framework will serialize it.

### Q108. When should Optional be returned, and when should it not be used?

`Optional<T>` is useful as a return type when absence is a normal, expected result and callers should handle it explicitly. It is not a universal replacement for null: using it for every field, method parameter, or collection element can add unnecessary complexity and may not match framework serialization or persistence conventions.

Avoid calling `get()` without checking presence. Methods such as `map`, `flatMap`, `orElseGet`, and `orElseThrow` express intent clearly. Remember that `orElse` evaluates its argument eagerly, while `orElseGet` evaluates the supplier only when needed.

### Q109. What is the difference between orElse() and orElseGet()?

`orElse(value)` evaluates the argument before the call, even when the Optional contains a value. `orElseGet(supplier)` invokes the supplier only when the Optional is empty. If the fallback is cheap and already available, `orElse` is fine; if it performs expensive work or has side effects, `orElseGet` avoids unnecessary evaluation.

The key interview point is evaluation behavior, not merely syntax. A fallback that performs a database call should not be placed in `orElse` if it must only run when the value is absent.

### Q110. Why is the Java Date and Time API preferred over Date and Calendar?

The `java.time` API introduced in Java 8 provides clearer types for different concepts: `LocalDate` for a date without a time zone, `Instant` for a point on the UTC timeline, `LocalDateTime` for a local wall-clock date and time, and `ZonedDateTime` for a date-time associated with a time zone. Most of these types are immutable and thread-safe.

Choose the type based on the domain. A birthday is usually a date, an audit event is usually an instant, and a scheduled meeting often needs a local time plus a zone. A `LocalDateTime` alone does not identify a unique instant because it has no time-zone offset or daylight-saving rules.

### Q111. What is the difference between Instant, LocalDate, LocalDateTime, and ZonedDateTime?

Instant represents a point on the UTC timeline. LocalDate represents a calendar date without a time or zone. LocalDateTime represents local wall-clock date and time without identifying a unique instant. ZonedDateTime combines date-time information with a time zone's rules. Select the type that matches the domain instead of converting everything to LocalDateTime.

### Q112. Why is LocalDateTime not enough to represent an event timestamp?

LocalDateTime contains no zone or offset, so it may correspond to different instants in different zones and can be ambiguous or invalid around daylight-saving transitions. For an audit event or globally ordered timestamp, use Instant; for a user appointment, preserve the intended zone where it matters.

### Q113. Why should you avoid hard-coding time zones and offsets?

Time-zone rules change due to government decisions and daylight-saving policies. A fixed offset does not capture the historical and future rules of a region. Use ZoneId for region-based rules and test around transitions; use UTC/Instant for storing globally comparable event times where appropriate.

### Q114. What is the difference between Period and Duration?

Period represents date-based amounts such as years, months, and days. Duration represents time-based amounts in seconds and nanoseconds. A month does not have a fixed number of seconds, so choose the abstraction that matches the business meaning instead of treating every temporal difference as a fixed duration.

### Q115. How should date/time parsing and formatting be handled?

Use DateTimeFormatter with an explicit format when the external contract defines one, and choose a locale and zone intentionally. Prefer immutable java.time types for modern code. Validate malformed external input at the boundary and avoid silently interpreting a local time in the server's default zone.

---

---

## 8. Stream API fundamentals and coding questions

### Q116. What is the difference between map() and flatMap() in streams?

`map()` transforms each input into one output, while `flatMap()` transforms each input into a stream-like result and flattens the nested results into one stream. For example, mapping a list of orders to each order's line items naturally creates nested collections; flatMap turns them into one stream of line items.

Use `map` for one-to-one transformations and `flatMap` for one-to-many or nested-result transformations. If a simple loop is clearer or avoids awkward state, it is perfectly reasonable to use a loop instead of forcing a stream pipeline.

### Q117. How do intermediate and terminal Stream operations differ, and why does laziness matter?

Intermediate operations such as `filter`, `map`, and `sorted` build a pipeline and are generally lazy. Terminal operations such as `collect`, `reduce`, `count`, and `findFirst` trigger evaluation. Short-circuiting terminals can stop once the result is known, so not every element necessarily passes through every stage.

Do not rely on side effects inside intermediate operations for correctness. The Stream contract allows implementations to elide behavioral-parameter execution when it cannot affect the terminal result. Prefer transformations that are stateless and side-effect-free, and use explicit terminal operations or loops for necessary effects.

### Q118. When should parallel streams be used, and what can go wrong?

Parallel streams split work across tasks, often using the common ForkJoinPool. They can help when the dataset is large, the work per element is substantial, operations are independent, and the source splits efficiently. They can hurt when tasks are small, the source is difficult to split, operations block on I/O, shared mutable state creates contention, or preserving encounter order is expensive.

Parallelism is not a free speedup. Measure with representative data and realistic contention. Avoid parallel streams for blocking network/database calls without careful resource and executor design; a shared common pool can interfere with unrelated application work.

### Q119. What makes a Stream reduce operation correct?

A reduction combines elements using an identity and an accumulator, and parallel reduction may partition the input and combine partial results. For correctness, the reduction operation must obey the required associativity/identity rules; otherwise sequential and parallel execution can produce different or invalid results.

Summing numbers is a natural reduction. Building a mutable collection with an incorrect `reduce` is not; use `collect` and a proper collector for mutable reduction. When results differ under parallel execution, inspect side effects, non-associative operations, encounter-order assumptions, and shared mutable state.

---

### Q120. How do you filter even numbers and square them?

    List<Integer> squares = numbers.stream()
        .filter(n -> n % 2 == 0)
        .map(n -> n * n)
        .toList();

Filter selects values and map transforms them. Stream.toList() is available from Java 16 and returns an unmodifiable list; for Java 8–15 use Collectors.toList(), whose mutability is not guaranteed by the general contract either.

### Q121. How do you count the frequency of each word?

    Map<String, Long> counts = words.stream()
        .collect(Collectors.groupingBy(
            Function.identity(), Collectors.counting()));

groupingBy creates a group for each distinct word and counting reduces each group to a count. Normalize case and define punctuation handling before counting if the input is natural language.

### Q122. How do you group employees by department?

    Map<String, List<Employee>> byDepartment = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));

The classifier determines each map key. If a department can be null, decide how to represent missing values explicitly because the default groupingBy collector does not accept a null classifier result.

### Q123. How do you count employees in each department?

    Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment, Collectors.counting()));

This is a downstream collector. It retains counts rather than lists of employees, which is more efficient when the grouped objects are not needed later.

### Q124. How do you find the second-highest distinct salary?

    Optional<Integer> second = employees.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();

The result is optional because there may be fewer than two distinct salaries. Sorting costs O(n log n); for a very large input where only two values are needed, a one-pass approach can be more efficient.

### Q125. How do you find duplicate values in a list?

    Map<String, Long> counts = values.stream()
        .collect(Collectors.groupingBy(
            Function.identity(), Collectors.counting()));

    Set<String> duplicates = counts.entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toSet());

Counting first avoids shared mutable state inside the stream. A side-effect-based solution using a shared HashSet is not safe for parallel execution.

### Q126. How do you remove duplicates while preserving encounter order?

    List<String> unique = values.stream()
        .distinct()
        .toList();

For an ordered stream, distinct preserves the first encountered occurrence. Equality relies on equals, so custom objects need correct equality semantics.

### Q127. How do you partition numbers into even and odd groups?

    Map<Boolean, List<Integer>> result = numbers.stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));

partitioningBy is appropriate when there are exactly two logical groups. groupingBy is better when the number of categories is variable.

### Q128. How do you flatten a list of lists?

    List<String> flattened = nested.stream()
        .flatMap(List::stream)
        .toList();

map would produce a stream of lists; flatMap combines the inner streams into one stream. Decide how null inner lists should be handled rather than allowing accidental failures.

### Q129. How do you split sentences into distinct words?

    Set<String> words = sentences.stream()
        .flatMap(s -> Arrays.stream(s.split("\\s+")))
        .map(s -> s.toLowerCase(Locale.ROOT))
        .collect(Collectors.toSet());

This is a simple whitespace tokenizer, not a full natural-language tokenizer. Punctuation, Unicode rules, and locale-specific normalization may require additional handling.

### Q130. How do you convert a list into a map, and what if keys repeat?

    Map<Long, Employee> byId = employees.stream()
        .collect(Collectors.toMap(Employee::getId, Function.identity()));

toMap throws IllegalStateException when duplicate keys occur. If duplicates are valid, provide a merge function and document whether the first value, last value, or a combined value should win.

### Q131. How do you sum values when multiple objects share the same key?

    Map<String, Integer> totals = items.stream()
        .collect(Collectors.toMap(
            Item::getType, Item::getAmount, Integer::sum));

The merge function combines values with the same key. Consider overflow and make sure summing matches the business rule.

### Q132. How do you sort strings by length and then alphabetically?

    List<String> sorted = words.stream()
        .sorted(Comparator.comparingInt(String::length)
            .thenComparing(Comparator.naturalOrder()))
        .toList();

Comparator composition expresses the primary and tie-breaker order. A deterministic tie-breaker makes results and tests more predictable.

### Q133. How do you calculate sum, average, minimum, and maximum in one pass?

    IntSummaryStatistics stats = numbers.stream()
        .mapToInt(Integer::intValue)
        .summaryStatistics();

Primitive streams can reduce boxing. Be mindful of numeric overflow and precision; floating-point statistics are not a substitute for decimal arithmetic in financial domains.

### Q134. How do you join strings with a delimiter?

    String joined = names.stream()
        .filter(Objects::nonNull)
        .collect(Collectors.joining(", "));

joining supports delimiters, prefixes, and suffixes. Decide explicitly whether null values should be rejected, filtered, or represented.

### Q135. How do you test whether any, all, or none of the elements match?

    boolean anyHigh = salaries.stream().anyMatch(s -> s > limit);
    boolean allValid = salaries.stream().allMatch(s -> s >= 0);
    boolean noneNegative = salaries.stream().noneMatch(s -> s < 0);

These are short-circuiting terminal operations. On an empty stream, anyMatch is false while allMatch and noneMatch are true, which follows their logical definitions.

### Q136. What is the difference between findFirst() and findAny()?

findFirst returns the first element according to encounter order when the stream has one. findAny permits any matching element and can offer more flexibility in parallel pipelines. Choose findFirst when order is part of the requirement; do not use it as a substitute for sorting to find a minimum or maximum.

### Q137. What is the difference between reduce() and collect()?

reduce combines elements into a result using an operation such as sum or maximum. collect supports mutable reduction using a supplier, accumulator, and combiner, such as building a list or map. Avoid using reduce with a shared mutable list; use a collector designed for the reduction.

### Q138. Why must a parallel reduce operation be associative?

Parallel execution can split input into chunks and combine partial results in a different grouping. If the operation is not associative, changing the grouping can change the result. The identity must also be a valid identity for the operation. Subtraction is a classic example of an operation that is not associative.

### Q139. How do you get a set of employee names per department?

    Map<String, Set<String>> namesByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.mapping(Employee::getName, Collectors.toSet())));

The downstream mapping collector transforms each employee to a name before collecting. This avoids collecting full employee lists and then traversing them again.

### Q140. How do you calculate average salary per department?

    Map<String, Double> averageByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.averagingDouble(Employee::getSalary)));

This produces one average per group. For financial values, choose a decimal representation and explicit rounding policy when binary floating-point is not appropriate.

### Q141. How do you collect entries into a sorted map?

    Map<String, Long> counts = words.stream()
        .collect(Collectors.groupingBy(
            Function.identity(), TreeMap::new, Collectors.counting()));

The map factory selects TreeMap, which sorts keys. Use it when sorted iteration is a requirement, not simply because sorted output looks convenient.

### Q142. Why should peek() not be used for required business side effects?

peek is primarily an inspection/debugging operation. Stream optimizations may skip processing that does not affect the terminal result, so required side effects inside peek are not reliable. Use an explicit terminal operation or place the side effect at a clear boundary.

### Q143. Why can a Stream only be consumed once?

A Stream is a pipeline, not a reusable collection. After a terminal operation, it is consumed and a second terminal operation normally fails. Create a new stream from a reusable source when multiple traversals are needed.

### Q144. When is parallelStream() a poor choice?

It can be slower for small inputs, cheap operations, blocking I/O, pipelines with shared mutable state, or strict ordering requirements. Parallel streams may also contend for shared fork/join infrastructure. Benchmark representative workloads instead of assuming parallel is faster.

### Q145. How do you find the top three most frequent words?

    List<Map.Entry<String, Long>> top = counts.entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed()
            .thenComparing(Map.Entry.comparingByKey()))
        .limit(3)
        .toList();

Sorting all k distinct entries costs O(k log k). For huge k and very small top-N, a bounded heap may be more efficient; this solution is straightforward and readable.

### Q146. How do you find common elements between two lists efficiently?

    Set<String> lookup = new HashSet<>(second);
    List<String> common = first.stream()
        .filter(lookup::contains)
        .distinct()
        .toList();

The set gives efficient average membership checks and the output preserves the first list's encounter order. Clarify whether duplicates should be preserved, because set intersection and multiset intersection are different requirements.

### Q147. How do you safely handle an empty numeric stream?

    OptionalInt maximum = numbers.stream()
        .mapToInt(Integer::intValue)
        .max();

The optional result represents the absence of a maximum. Avoid retrieving a value without checking presence or defining a fallback.

### Q148. How do you compose methods that return Optional values?

    Optional<Country> country = person
        .flatMap(Person::getAddress)
        .flatMap(Address::getCountry);

flatMap avoids nested Optional values when each function already returns Optional. The example assumes the methods use Optional consistently; normalize nullable boundaries deliberately.

### Q149. When is a loop better than a stream?

Loops can be clearer for early exits, complex state changes, coordinated mutation, or detailed error handling. Streams are often concise for filter-map-reduce and collector-based transformations. Explain readability and complexity rather than claiming one style is always faster.

### Q150. How do you implement a collector for a custom aggregation?

A Collector defines how to create an accumulation container, add one element, combine partial containers, and optionally finish the result. Its combiner and characteristics must satisfy the collector contract, particularly if parallel execution is allowed. Prefer standard collectors unless the custom aggregation is genuinely clearer and tested in both sequential and parallel use.

### Q151. What should you check when a stream pipeline gives an unexpected result?

Check the source ordering, intermediate operation order, terminal operation, equals/hashCode behavior used by distinct or map keys, null handling, and whether side effects were incorrectly placed in intermediate operations. For parallel streams, also check ordering requirements and whether the collector supports parallel reduction. Reduce the problem to a small input and test edge cases such as empty input, duplicates, nulls, and ties.

### Q152. How do you find the longest string in a list with streams?

    Optional<String> longest = words.stream()
        .max(Comparator.comparingInt(String::length));

The result is optional because the source may be empty. If several strings have the same maximum length, the contract may not express which one should win in every stream configuration; define a tie-breaker if deterministic selection matters.

### Q153. How do you safely handle null values before sorting?

    List<String> sorted = values.stream()
        .filter(Objects::nonNull)
        .sorted()
        .toList();

Filtering nulls is one possible policy, but it is not always the correct business behavior. Another option is a null-aware comparator such as Comparator.nullsLast. State the policy explicitly rather than letting a NullPointerException decide it.

### Q154. How do you use mapMulti, and when might it be useful?

    Stream<String> tokens = lines.stream()
        .<String>mapMulti((line, out) -> {
            for (String token : line.split("\\s+")) {
                out.accept(token);
            }
        });

mapMulti can emit zero or more results for each input element and can be useful for simple one-to-many transformations without creating an inner stream for every element. It became a standard Stream API method in Java 16. Prefer flatMap when it communicates the transformation more clearly.

### Q155. How do you use takeWhile and dropWhile?

    List<Integer> prefix = values.stream()
        .takeWhile(n -> n < 10)
        .toList();

    List<Integer> remainder = values.stream()
        .dropWhile(n -> n < 10)
        .toList();

For ordered streams, takeWhile keeps the longest matching prefix and dropWhile skips that prefix. They are not equivalent to filter: filter can retain later matching values after a non-matching value appears.

### Q156. What does Collectors.teeing do?

    Summary summary = values.stream().collect(
        Collectors.teeing(
            Collectors.minBy(Integer::compareTo),
            Collectors.maxBy(Integer::compareTo),
            Summary::new));

teeing sends each input to two downstream collectors and combines their results with a merger. It can calculate related summaries in one traversal, but the result type and empty-input behavior still need a clear design.

### Q157. What are the common pitfalls of Collectors.toMap?

Duplicate keys cause an exception unless a merge function is supplied. A null key or value may also be rejected depending on the collector and map implementation. Decide whether duplicates are invalid, should overwrite, or should be aggregated; do not add a merge function just to silence an exception without defining the rule.

### Q158. How do you use a method reference, and when is a lambda clearer?

    names.stream().map(String::trim).toList();

A method reference is concise when an existing method exactly matches the functional interface's shape. A lambda can be clearer when it performs additional logic, handles edge cases, or makes the transformation more explicit. Choose readability rather than using method references everywhere.

### Q159. What is the effectively final rule for captured local variables?

A lambda can capture a local variable only when it is final or effectively final, meaning its value is assigned once and not reassigned. This rule avoids the appearance that a lambda is sharing a mutable local stack variable across execution contexts. It does not make the referenced object immutable: a captured reference can still point to a mutable object.

### Q160. How should you use Comparator.comparing with nullable fields?

    Comparator<Employee> byName = Comparator.comparing(
        Employee::getName,
        Comparator.nullsLast(Comparator.naturalOrder()));

The comparator defines how null names are ordered. Make null handling explicit in the comparator or validate the data before sorting. Comparator composition should also include tie-breakers when deterministic ordering is important.

### Q161. How do you avoid accidental integer overflow in stream sums?

    long total = values.stream()
        .mapToLong(Integer::longValue)
        .sum();

Summing into a wider primitive reduces the risk of 32-bit overflow, but long can overflow too. For exact large integer totals, use BigInteger; for decimal financial values, choose BigDecimal with an explicit rounding policy.

### Q162. What is the difference between mapToInt and map?

map transforms each input into another reference value and produces a Stream of objects. mapToInt produces an IntStream, which provides primitive-specialized operations such as sum and summaryStatistics and avoids boxing each value. Use primitive streams when the result is numeric and their operations make the code clearer.

---

---

## 9. Concurrency, Java Memory Model, executors, and asynchronous programming

### Q163. What is the difference between a process and a thread?

A process is an operating-system execution environment with its own address space and resources. Threads within a process share heap memory and many process resources but have their own execution state and stacks. Sharing memory makes communication efficient, but also creates the possibility of data races and visibility bugs.

Java's `Thread` API abstracts thread behavior, while the runtime maps platform threads to operating-system threads and schedules virtual threads using runtime-managed mechanisms. Avoid assuming every Java thread corresponds one-to-one with an OS thread.

### Q164. What is the difference between synchronized and volatile?

`synchronized` provides mutual exclusion for a monitor and establishes visibility/order guarantees when a monitor is unlocked and later locked. It can protect compound invariants by ensuring only one thread executes a critical section at a time.

`volatile` provides visibility and ordering for reads/writes to that variable but does not make compound read-modify-write operations atomic. A volatile flag can publish state safely under a well-defined protocol; a shared counter increment usually needs an atomic class or a lock. Choose based on whether the requirement is visibility alone or an indivisible multi-step state transition.

### Q165. What does happens-before mean?

Happens-before is a memory-model relationship that guarantees visibility and ordering between actions. Program order within one thread contributes to it; synchronization constructs create cross-thread relationships. For example, actions before a thread starts happen-before actions in that thread, and a thread's actions happen-before another thread successfully returns from joining it.

A volatile write happens-before a subsequent read of that same variable under the specified rules, and unlocking a monitor happens-before a later lock of that monitor. These are contractual guarantees, not guesses about CPU caches. Reason about concurrency by identifying the synchronizing edge that makes one thread's writes visible to another.

### Q166. Why is count++ not atomic, even if count is volatile?

Increment is a read-modify-write sequence: read the current value, calculate the next value, then write it. Two threads can read the same value and both write the same incremented result, losing one update. Volatile guarantees visibility and ordering of the individual accesses, not atomicity of the whole sequence.

Use `AtomicInteger.incrementAndGet()` for a simple atomic counter, `LongAdder` for high-contention statistical counters where sum consistency requirements allow it, or a lock when the update belongs to a larger invariant. The choice depends on semantics, not just speed.

### Q167. What is the difference between Runnable, Callable, Future, and CompletableFuture?

`Runnable` represents work that does not return a result through its method signature; `Callable<V>` can return a value and throw checked exceptions. `Future<V>` represents a result that may become available later and supports waiting and cancellation, but its basic API can be awkward for composing multiple asynchronous stages.

`CompletableFuture` supports composition, combining results, and explicit completion. Methods ending in `Async` may use a default or supplied executor, depending on the overload. Always handle failures and understand which executor runs each stage; otherwise, blocking work can occupy the common pool or exceptions can remain hidden until a result is observed.

### Q168. Why use ExecutorService instead of creating threads directly?

An `ExecutorService` separates task submission from execution policy and provides lifecycle management, result tracking, and controlled concurrency. A bounded platform-thread pool can cap active work and queue tasks, but its queue and rejection policy must be chosen deliberately. An unbounded queue can turn overload into growing latency and memory use.

Use a suitable executor for the workload, propagate cancellation and interruption correctly, and shut it down gracefully. A pool is not just a performance setting: it is part of the service's capacity and backpressure design. Avoid creating a new platform thread for every task under high load without understanding the cost and concurrency limits.

### Q169. What causes deadlocks, and how can they be prevented or diagnosed?

A deadlock occurs when threads wait indefinitely for resources held by one another. A common pattern is inconsistent lock ordering: one thread acquires A then waits for B, while another acquires B then waits for A. Deadlocks can also involve locks combined with callbacks, futures, database locks, or external resources.

Prevent them by enforcing a global lock order, keeping critical sections small, avoiding blocking calls while holding locks, and using timeouts where the semantics permit. Diagnose using thread dumps and identify the wait-for cycle rather than blindly restarting or changing pool sizes. A timeout may reduce indefinite waiting but does not automatically make a partially completed operation safe to retry.

### Q170. What are race conditions, and how do concurrent collections help?

A race condition occurs when a result depends on the timing or interleaving of operations. A data race is a more specific Java Memory Model concept involving conflicting unsynchronized accesses. Not every race is a data race, and both can lead to incorrect application behavior.

Concurrent collections provide thread-safe operations and specified visibility guarantees for their own data structures. They do not automatically make multi-operation workflows atomic or protect mutable objects stored inside them. Use atomic compound methods such as `compute` where applicable, external locks for larger invariants, or immutable values and clear ownership boundaries.

### Q171. What are virtual threads, and when should they be used?

Virtual threads became a final feature in Java 21. They are lightweight Java threads scheduled by the runtime over a smaller set of carrier platform threads. They make thread-per-task code practical for applications with many concurrent tasks that spend substantial time waiting on blocking I/O.

They improve scalability/throughput for suitable workloads, not the speed of CPU-bound computation and not the capacity of downstream services. Use a virtual thread per task rather than pooling virtual threads as if they were scarce OS threads, but still limit access to scarce resources such as database connections, API quotas, and memory. Check the target JDK's guidance for pinning and observability details.

### Q172. How should interruption and cancellation be handled?

Interruption is a cooperative cancellation signal, not a mechanism that forcibly kills arbitrary code. Blocking methods may throw `InterruptedException`, typically clearing the interrupt status when they do. Code that cannot propagate the exception should usually restore the status with `Thread.currentThread().interrupt()` after performing any necessary cleanup.

Cancellation must be designed through the call chain: tasks need to notice the signal, release resources, and avoid committing partial results incorrectly. Cancelling a Future does not guarantee that underlying I/O or a remote operation has stopped. Treat timeout, cancellation, and unknown remote outcome as distinct states when deciding whether retry is safe.

### Q173. What are CountDownLatch, Semaphore, and BlockingQueue used for?

A `CountDownLatch` lets one or more threads wait until a count reaches zero; it is one-shot. A `Semaphore` controls access to a limited number of permits and is useful for bounding concurrent use of a resource. A `BlockingQueue` coordinates producers and consumers and can apply backpressure when bounded.

Choose the primitive that expresses the invariant. A semaphore does not itself manage a resource lifecycle, and a queue does not guarantee that downstream processing succeeds. Always consider interruption, shutdown, queue capacity, permit release in `finally`, and failure handling.

---

### Q174. What is the difference between synchronized, Lock, and atomic classes?

`synchronized` provides mutual exclusion and memory-visibility guarantees tied to monitor acquisition and release. Explicit `Lock` implementations provide capabilities such as interruptible acquisition, timed attempts, and multiple condition variables, but require disciplined unlocking, normally in a `finally` block. Atomic classes support specific thread-safe updates such as compare-and-set without requiring a monitor for that operation.

Choose the simplest mechanism that matches the invariant. Atomics are ideal for independent counters or state transitions, but they do not automatically make a multi-field invariant atomic. A lock or synchronized critical section may be clearer when several values must change together.

### Q175. What are atomicity, visibility, and ordering in concurrent code?

Atomicity means an operation appears indivisible; visibility means one thread's writes become observable to another; ordering concerns which operations may be observed before others. These are related but distinct. A volatile write/read pair establishes visibility and ordering guarantees, but does not turn a compound read-modify-write operation into an atomic one.

Use synchronization, volatile, atomics, or higher-level concurrency utilities according to the actual guarantee required. Avoid reasoning from how code appears to execute on one machine: the Java Memory Model defines which behaviors are permitted across threads.

### Q176. What is safe publication, and why does it matter?

Safe publication means making an object visible to other threads in a way that ensures they see its correctly initialized state. This can be achieved through mechanisms such as static initialization, a volatile reference, synchronization, or thread-safe concurrent utilities. Publishing an object before construction is complete can allow another thread to observe inconsistent state.

Immutability simplifies safe sharing, especially when fields are final and the object does not leak its reference during construction. But final fields do not excuse leaking `this` from a constructor or using unsafe publication for mutable surrounding state.

### Q177. How do wait(), notify(), and notifyAll() work, and why should wait be in a loop?

These methods coordinate threads using an object's monitor. A thread must own that monitor before calling `wait()`, `notify()`, or `notifyAll()`. Calling `wait()` releases the monitor while waiting and reacquires it before returning. A thread should check its condition in a loop because wakeups can be spurious and another thread may consume or change the condition before the awakened thread reacquires the lock.

For new code, higher-level utilities such as blocking queues, latches, semaphores, or condition variables are often easier to reason about. Avoid using notification as a substitute for defining a shared state predicate and its synchronization discipline.

### Q178. What is thread starvation, and how is it different from deadlock and livelock?

Deadlock means threads are stuck waiting on resources held by one another. Starvation means a thread repeatedly fails to get the CPU, lock, or resource it needs because others keep winning. Livelock means threads remain active but repeatedly react to one another without making useful progress.

The remedies differ: deadlock prevention often focuses on lock ordering and reducing nested locks; starvation may require fairer scheduling or bounded resource access; livelock may require backoff or a change in coordination strategy. Thread dumps and runtime metrics help distinguish symptoms rather than treating every hang as a deadlock.

### Q179. How should CompletableFuture exception handling and composition work?

Use `thenApply` to transform a completed result, `thenCompose` to chain an asynchronous operation that itself returns a future, and `thenCombine` when combining independent futures. Exception paths can be handled with methods such as `exceptionally`, `handle`, or `whenComplete`, but these have different semantics: some recover with a value, some transform the outcome, and some observe completion.

Do not block carelessly with `get()` or `join()` inside constrained executor threads; this can cause thread starvation or deadlock-like behavior. Consider the executor, timeout policy, cancellation behavior, and whether failures are propagated or converted into fallback results.

### Q180. How do you choose an ExecutorService and prevent thread-pool exhaustion?

Choose pool sizing and queue behavior based on the workload, resource limits, and latency target. CPU-bound tasks often benefit from a pool near the number of available processors; blocking I/O workloads need a different approach because workers spend time waiting. These are starting points, not universal formulas—measure the actual workload.

Unbounded queues can hide overload while latency and memory grow. Unbounded thread creation can exhaust memory or operating-system resources. Define rejection or backpressure behavior, track active threads and queue depth, bound task duration where possible, and shut down executors deliberately. For blocking task-per-request workloads on Java 21+, virtual threads may simplify concurrency, but they do not remove database connection limits or downstream capacity constraints.

### Q181. What are structured concurrency and scoped values, and what should you know about their status?

Structured concurrency aims to make related concurrent tasks share a clear lifetime and failure boundary, so child tasks are not left running unintentionally after the parent operation ends. Scoped values provide a controlled way to share context across call chains without passing it through every method parameter or relying on mutable thread-local state.

Their availability and final status have changed across JDK releases, and some versions expose these capabilities as preview APIs. In an interview, explain the design goal and verify the target JDK's status before claiming a particular API is final or production-ready. Do not treat preview features as stable language contracts.

### Q182. What is the ABA problem in compare-and-set algorithms?

With compare-and-set (CAS), a thread checks that a value still equals an expected value before replacing it. The ABA problem occurs when another thread changes the value from A to B and then back to A; a CAS operation sees A again and cannot tell that an intermediate change occurred. Whether this matters depends on the state invariant and what the value represents.

Where change history matters, use a versioned or stamped value, such as an appropriate stamped atomic reference, or redesign the state transition. Do not assume that a successful CAS proves nothing changed in between.

### Q183. Why is double-checked locking unsafe without volatile?

Double-checked locking tries to avoid acquiring a lock on every access by checking a shared reference before and after entering a synchronized block. Without the correct memory semantics, another thread could observe a published reference before construction is safely visible, because object initialization and reference publication are not automatically one indivisible action.

In modern Java, a correctly implemented pattern uses a volatile reference and synchronized initialization, but the initialization-on-demand holder idiom or an enum singleton is often simpler. Explain both the visibility guarantee and why avoiding unnecessary custom singleton machinery is usually preferable.

### Q184. Why is Thread.sleep() not a synchronization mechanism?

`Thread.sleep()` pauses the current thread for a period, subject to scheduling, but it does not establish a happens-before relationship and does not guarantee that another thread's writes become visible. A test or program that “works” after sleeping may still contain a data race and can fail under different timing or hardware.

Coordinate threads through synchronized state, volatile/atomic variables, latches, queues, futures, or other documented synchronization mechanisms. Sleeps can be useful for pacing or simulation, but not for proving that another thread has completed an operation.

### Q185. How do you start a virtual thread?

    Thread.startVirtualThread(() -> processRequest());

Or create a named thread:

    Thread worker = Thread.ofVirtual()
        .name("request-worker")
        .start(this::processRequest);

Virtual threads make blocking task-per-thread code more scalable for I/O-bound work. They do not increase database capacity, remove downstream bottlenecks, or make CPU-heavy tasks infinitely faster.

### Q186. How do you run tasks with a virtual-thread-per-task executor?

    try (ExecutorService executor =
             Executors.newVirtualThreadPerTaskExecutor()) {
        Future<Result> result = executor.submit(this::loadResult);
        Result value = result.get();
    }

Use deadlines, cancellation, and explicit error handling in production. A virtual thread per task is not the same as an unbounded amount of downstream capacity; use bounded resource pools and backpressure where needed.

### Q187. How do you combine two independent asynchronous tasks with CompletableFuture?

    CompletableFuture<Result> a = loadA();
    CompletableFuture<Result> b = loadB();

    CompletableFuture<Combined> combined =
        a.thenCombine(b, Combined::new);

thenCombine waits for both stages and combines their results. The tasks are independent only if their creation starts the work independently. Consider failure propagation, deadlines, executor choice, and whether one failed result should cancel or merely affect the combined outcome.

### Q188. What is the difference between thenApply and thenCompose?

thenApply transforms a result synchronously into another value. thenCompose is used when the transformation itself returns a CompletableFuture, flattening the nested asynchronous result. Use thenCompose to chain dependent asynchronous operations without producing CompletableFuture<CompletableFuture<T>>.

### Q189. How do you handle an exceptional CompletableFuture result?

    future.exceptionally(error -> fallbackValue);

exceptionally can recover from a failure by producing a fallback result. handle can inspect either success or failure and transform both; whenComplete is intended for observation while preserving the original outcome. Do not silently convert every error into a default value if doing so hides a business failure.

### Q190. What are the lifecycle states of a Java thread?

The Thread.State enum includes NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, and TERMINATED. RUNNABLE includes threads that may be executing or ready to run; it does not mean the thread is continuously using a CPU. Thread states are diagnostic snapshots, so interpret them with stack traces and repeated observations.

### Q191. What is the difference between sleep(), wait(), and join()?

sleep pauses the current thread for a duration and does not release a monitor it owns. wait releases the monitor of the object on which it is called and waits for notification or another permitted wakeup condition. join waits for another thread to terminate. Use each only when its synchronization semantics match the requirement; sleep is not a substitute for coordination.

### Q192. What is thread interruption?

Interruption is a cooperative cancellation signal. Blocking methods may throw InterruptedException, which clears the interrupted status when thrown; code that cannot propagate the exception should generally restore the status with Thread.currentThread().interrupt() after handling or deciding to stop. Do not swallow interruption and continue indefinitely if the caller expects cancellation.

### Q193. What is the difference between shutdown() and shutdownNow()?

shutdown stops accepting new tasks and allows submitted tasks to finish. shutdownNow attempts to stop active tasks, typically by interrupting worker threads, and returns tasks that never commenced. Neither guarantees that a task will stop immediately; task code must cooperate with interruption and resource operations may have their own cancellation behavior.

### Q194. Why can an unbounded executor queue be dangerous?

An unbounded queue can allow requests to accumulate faster than workers process them. The service may appear healthy initially while queueing delay and retained memory grow, eventually causing timeouts or memory pressure. Use a bounded queue or an appropriate executor strategy, define rejection behavior, and expose queue depth and task wait time as metrics.

### Q195. What is backpressure in a Java service?

Backpressure prevents producers from overwhelming a slower consumer or dependency. It can be implemented through bounded queues, demand-aware reactive streams, concurrency limits, admission control, or rejection with a clear response. Adding more worker threads without controlling downstream capacity often moves the queue to another layer rather than solving overload.

### Q196. What is the difference between Future and CompletableFuture?

Future represents an asynchronous result and supports waiting, cancellation, and status checks. CompletableFuture also supports composition, transformation, combination, and explicit completion stages. CompletableFuture does not guarantee that every continuation runs on a new thread; non-async stages may execute on the thread completing the prior stage, while async methods use an executor according to their overload and configuration.

### Q197. What is the common pool, and why can it matter?

Many asynchronous operations use ForkJoinPool.commonPool by default. Blocking tasks or long-running work can interfere with unrelated operations that use the same pool. Select executors deliberately for blocking work, keep task behavior predictable, and inspect pool saturation when asynchronous pipelines become slow.

### Q198. What are CountDownLatch, CyclicBarrier, and Semaphore used for?

CountDownLatch lets one or more threads wait until a count reaches zero and cannot be reset. CyclicBarrier allows a fixed set of threads to meet at a reusable barrier. Semaphore limits concurrent access through permits. Choose the primitive that matches the coordination model and ensure failure paths do not leave other threads waiting forever.

### Q199. What is the difference between ReentrantLock and synchronized?

Both can provide mutual exclusion and visibility guarantees. ReentrantLock supports timed and interruptible lock acquisition, optional fairness, and multiple Condition objects, but requires explicit unlock, usually in finally. synchronized is simpler and releases the monitor automatically when the block exits. Prefer the simpler construct unless an explicit Lock feature is needed.

### Q200. What is a race condition, and how do you prevent it?

A race condition occurs when correctness depends on the timing or interleaving of operations that are not properly coordinated. Prevent it by protecting shared invariants with synchronization, atomic operations, thread-safe data structures, immutability, or confinement of state to one thread. Making one field volatile does not make a multi-step business operation atomic.

### Q201. Why are virtual threads not a replacement for every executor?

Virtual threads are particularly useful for high-concurrency workloads that spend time waiting, especially blocking I/O. They do not accelerate CPU-bound calculations and do not remove limits on database connections, remote-service quotas, memory, or file descriptors. Use semaphores or bounded resource pools when the downstream resource needs a concurrency limit.

### Q202. What is thread confinement?

Thread confinement means mutable state is accessed only by one thread, avoiding synchronization for that state. Local variables and objects that never escape the creating thread are common examples. Thread confinement can simplify correctness, but sharing the reference through callbacks, static fields, or asynchronous handoffs can break the assumption.

### Q203. What is the difference between ThreadLocal and ScopedValue?

ThreadLocal associates a value with a thread and can be useful for thread-specific state, but pooled-thread reuse requires careful cleanup and propagation across asynchronous boundaries can be confusing. ScopedValue models context bound to a bounded dynamic scope and became final in Java 25. Choose based on lifetime and propagation requirements; neither should be used to hide broad mutable global state.

---

---

## 10. JVM internals, garbage collection, and diagnostics

### Q204. What are heap and stack memory, and what causes StackOverflowError versus OutOfMemoryError?

The heap holds objects and arrays managed by the JVM, while each thread has execution-stack frames containing local variables, operand-stack state, and call information. The JVM specification defines abstract runtime areas; exact object layout and memory allocation strategies are implementation-dependent, and a JIT may optimize allocations when observable behavior is preserved.

`StackOverflowError` commonly results from excessive call depth, such as unbounded recursion. `OutOfMemoryError` indicates that the JVM could not satisfy an allocation or another memory-related requirement; causes include heap exhaustion, native-memory pressure, too many threads, or limits imposed by the environment. Diagnose the specific error and its evidence instead of assuming every memory incident is a heap leak.

### Q205. How does garbage collection determine which objects can be reclaimed?

Tracing garbage collectors begin from GC roots—runtime references such as those associated with live threads, stack frames, static references, and JVM internals—and follow reachable references. Objects not reachable from the roots are eligible for reclamation, subject to reference processing and collector behavior.

A Java memory leak usually means an object remains reachable through an unintended reference, not that code forgot to call free. Common causes include unbounded caches, static collections, listener registrations that are never removed, ThreadLocal values, and retaining request data. A heap dump and dominator/retained-size analysis can help locate the first unwanted retaining path.

### Q206. What is the difference between throughput and low-latency garbage collectors?

Collectors trade off application throughput, pause behavior, CPU use, memory overhead, and operational complexity. Some prioritize overall throughput; others are designed to reduce pause times and perform more work concurrently. Collector names and behavior evolve across JDK versions, and terms such as “minor” and “major” collection do not mean the same thing for every collector.

Choose based on measured workload and service-level objectives. Compare pause percentiles, allocation rate, live-set size, CPU, memory footprint, and tail latency under representative load. A low average pause can hide a rare long pause, while a low-pause collector can still reduce throughput or consume additional resources.

### Q207. What is the difference between memory leak and high memory usage?

High memory usage is an observation, not a diagnosis. It may reflect a legitimate live set, allocation bursts, cache warming, heap sizing, native memory, thread stacks, direct buffers, or a leak. A leak is suggested when retained memory grows over time beyond expected workload behavior and objects remain reachable without a valid business reason.

Start with trends and evidence: heap usage after collection, allocation rate, GC logs, heap dumps, class histograms, native-memory tracking where enabled, and thread counts. Identify whether memory is retained in the Java heap or outside it before changing heap settings. Increasing the heap may delay symptoms without fixing an unbounded retention path.

### Q208. How do you investigate a slow Java service?

First establish the symptom: increased latency, reduced throughput, CPU saturation, blocked requests, allocation pressure, GC pauses, lock contention, or downstream delay. Compare the incident against a known baseline and inspect metrics and traces to narrow the scope before changing runtime flags.

Use thread dumps for blocked or stuck threads, Java Flight Recorder (JFR) for runtime events and allocation/CPU evidence, GC logs for collection behavior, and a profiler for hotspots. Check database and remote-call latency too; a Java stack may be waiting on a downstream dependency rather than consuming CPU. Form a hypothesis, gather evidence, apply a reversible mitigation, and verify the effect.

### Q209. What is JIT compilation, and why can a Java application behave differently after warm-up?

The JVM may initially interpret bytecode and later compile frequently executed code into native machine code. The JIT uses runtime information to optimize hot paths, inline calls, eliminate redundant work, and apply other transformations while preserving Java's specified behavior. It can also deoptimize code when assumptions stop holding.

As a result, startup performance can differ from steady-state performance. Microbenchmarks must account for warm-up, dead-code elimination, constant folding, allocation, and measurement noise. Use JMH for microbenchmarks rather than ad hoc loops, and validate performance at application level because a faster micro-operation may not improve end-to-end latency.

### Q210. What is the difference between an OutOfMemoryError and a memory leak in production diagnosis?

An `OutOfMemoryError` is a failure symptom; a leak is one possible cause. The error can occur because of a legitimately oversized live set, a constrained container, native-memory exhaustion, direct-buffer usage, thread creation, or inability to allocate a particular region. Different error messages and runtime evidence point to different areas.

Capture the relevant heap dump or diagnostic evidence when safe and configured, inspect GC logs and container limits, and distinguish Java heap from native memory. Do not automatically increase `-Xmx`: in a container, the total budget also includes metaspace, code cache, direct buffers, thread stacks, native libraries, and other process memory.

---

### Q211. How does class-loader delegation work, and why can class-loading problems occur?

Class loaders locate and define classes at runtime. The traditional parent-delegation model asks a parent loader to load a class before the child defines it, helping avoid duplicate definitions of core classes and supporting a consistent runtime type system. Application servers, plugin systems, module layers, and custom class loaders can use more complex arrangements.

Two classes with the same fully qualified name are not necessarily the same runtime type if they were defined by different class loaders. This can cause confusing `ClassCastException`, `NoClassDefFoundError`, or linkage errors in application servers and plugin architectures. Diagnose the actual class loader, classpath/module path, dependency conflicts, and loading origin rather than assuming the source code type name uniquely identifies the runtime class.

---

### Q212. What happens during class loading, linking, and initialization?

At a high level, a class is loaded, linked, and initialized. Linking includes verification, preparation, and resolution; initialization executes class initializers and static field initializers according to Java's initialization rules. These stages are related but not identical, and some resolution can be lazy.

Initialization order can explain surprising static initialization bugs and class initialization failures. A failed class initializer can cause an initialization error, and later uses may report that the class could not be initialized. Keep static initialization simple and avoid complex dependency work in static blocks.

### Q213. What is the difference between NoClassDefFoundError, ClassNotFoundException, and LinkageError?

`ClassNotFoundException` is a checked exception commonly encountered when code explicitly requests class loading by name and the loader cannot find the class. `NoClassDefFoundError` is a linkage error that can occur when the runtime cannot find a class definition needed by already-compiled code, including certain failed initialization scenarios. `LinkageError` is a broader family covering incompatibilities encountered when linking classes, including some version or duplicate-definition problems.

Diagnose the underlying classpath, module path, class loader, dependency versions, and original startup logs. The visible exception is often a symptom; the first failure in the logs may be more informative than the final error reported by the framework.

### Q214. What are strong, soft, weak, and phantom references?

A strong reference keeps an object reachable and therefore normally prevents garbage collection. Soft, weak, and phantom references provide weaker reachability semantics for specialized use cases, with different collection and notification behavior. Weak references are used in designs such as certain caches or canonicalization maps, but they should not be treated as a general-purpose cache policy.

Reference queues can help a program learn when referenced objects become eligible for cleanup or associated bookkeeping. Do not depend on a specific garbage-collection time, and do not use finalization for resource management. Prefer explicit lifecycle management and try-with-resources for external resources.

### Q215. What is a class-loader memory leak?

A class loader can become collectible only when it and the classes it defined are no longer reachable through relevant runtime references. In application servers or plugin systems, static registries, thread locals, long-lived threads, callbacks, or cached objects can retain classes from a redeployed application and therefore retain its class loader.

Symptoms can include growing metaspace usage across redeployments and increasing numbers of loaded classes. Investigate class-loader statistics and heap dumps, then identify the retaining references. Closing resources and clearing registries or thread-local values at the correct lifecycle boundary can prevent the leak.

### Q216. What are heap dumps, thread dumps, and Java Flight Recorder used for?

A heap dump captures object and reference information at a point in time and helps investigate retained memory, object populations, and possible leaks. A thread dump captures thread states and stack traces, useful for deadlocks, blocked threads, and executor problems. Java Flight Recorder (JFR) records runtime events such as allocations, garbage collection, locks, and thread activity with configurable overhead.

Use the diagnostic artifact that matches the symptom and correlate it with metrics and timestamps. A heap dump alone may show what retains memory but not why the workload allocated it; a thread dump alone may not explain a latency spike that has already passed. Collect data safely in production because dumps can be large and may contain sensitive values.

### Q217. What is the difference between metaspace, heap, and direct memory?

The heap stores Java objects managed by the garbage collector. Metaspace stores class metadata and is managed separately from the Java object heap. Direct buffers can use memory outside the ordinary heap, which can reduce copying for some I/O paths but still consumes finite native memory.

A process can run out of native memory even when heap usage appears acceptable. When diagnosing memory pressure, inspect heap occupancy, metaspace, direct-buffer usage, thread stacks, native allocations, and the container or operating-system limit rather than looking only at `-Xmx`.

### Q218. How do you investigate high CPU usage in a Java process?

First establish whether the process is truly CPU-saturated and whether the usage is in application threads, GC, JIT compilation, or native code. Correlate operating-system process and thread metrics with JVM telemetry, then capture thread dumps or a Java Flight Recorder recording while the issue is active. Repeated hot stack traces can help identify loops, excessive serialization, expensive regular expressions, contention-related spinning, or unexpectedly high request volume.

Do not tune the garbage collector or rewrite code before identifying the dominant CPU consumer. Compare the evidence with request rate and deployment changes, then validate a targeted fix under representative load.

### Q219. What tools help diagnose a running JVM, and what is each useful for?

Common tools include `jcmd` for diagnostic commands and recordings, `jstack` or equivalent thread-dump mechanisms for thread states, `jmap` or supported heap-dump commands for heap investigation, `jstat` for selected JVM statistics, and Java Flight Recorder with Java Mission Control for runtime-event analysis. Exact availability and command options depend on the JDK distribution, version, permissions, and environment.

Use the least disruptive tool that answers the question, and check the tool's documentation for the target runtime. Heap dumps and recordings can be large and may contain sensitive data, so plan collection, storage, access control, and cleanup.

### Q220. What is the difference between heap and stack memory?

The heap holds objects and arrays managed by the garbage collector. Each thread has its own stack of frames containing method-call state, local variables, and intermediate execution information. These are conceptual runtime areas; JIT optimizations can eliminate allocations or move values in ways that do not correspond directly to a simple source-level picture.

### Q221. What is the difference between StackOverflowError and OutOfMemoryError?

StackOverflowError often occurs when a thread's stack is exhausted, commonly due to unbounded recursion or deep call chains. OutOfMemoryError indicates that the JVM could not satisfy an allocation or another memory-related need; causes can include heap, metaspace, direct memory, or native resource exhaustion. Diagnose the exact message and runtime metrics rather than treating every memory error as a heap leak.

### Q222. How does garbage collection determine whether an object is reachable?

The garbage collector traces reachability from GC roots such as active thread stacks, static references, and other runtime roots. Objects no longer reachable from these roots can be reclaimed, subject to the collector's behavior. Garbage collection does not automatically close files, sockets, or database connections; those resources require explicit lifecycle management.

### Q223. What is the difference between a memory leak and high allocation rate?

A memory leak occurs when objects remain reachable longer than intended, causing retained memory to grow. High allocation rate means the application creates many objects, which may still be reclaimed promptly but can increase GC work. Heap dumps help investigate retention; allocation profiling and JFR help identify where objects are created.

### Q224. What is a stop-the-world pause?

A stop-the-world pause is a period when application threads are paused for a JVM operation, often related to garbage collection or runtime coordination. The impact depends on pause duration, frequency, workload, and latency requirements. Modern collectors aim to reduce pause impact through different trade-offs, but no collector makes all runtime coordination disappear.

### Q225. How do G1, ZGC, and Shenandoah differ at a high level?

G1 is a general-purpose regional collector designed to balance throughput and pause-time goals. ZGC targets very low pause times and supports large heaps; Shenandoah also emphasizes low pause times through concurrent work. Exact capabilities and defaults vary by JDK release and distribution. Select based on measured latency, throughput, heap size, and operational constraints rather than a universal “best” collector.

### Q226. What is metaspace and how can it be exhausted?

Metaspace stores class metadata outside the ordinary Java heap. It can grow when many classes or class loaders are retained, including in application servers or plugin environments. Investigate class counts, class-loader retention, redeployment patterns, and the exact OutOfMemoryError message; increasing the limit alone may only delay a leak.

### Q227. What is escape analysis?

Escape analysis lets the JIT determine whether an object is confined to a limited scope or escapes to other code or threads. This can enable optimizations such as scalar replacement or lock elimination in suitable cases. It is an optimization, not a language guarantee; do not assume every source-level object necessarily becomes a heap allocation or that a particular optimization always occurs.

### Q228. What is JIT compilation and why does warm-up matter?

The JVM can compile frequently executed bytecode into optimized machine code based on runtime observations. Early execution may have different performance from warmed-up execution as compilation and profiling progress. Microbenchmarks must account for warm-up, dead-code elimination, constant folding, and measurement noise; use JMH rather than timing a loop once.

### Q229. How do you diagnose a suspected Java deadlock?

Capture a thread dump while the application is stuck and inspect monitor ownership and waiting relationships. Look for cycles in which each thread waits for a resource held by another. Then map the stacks to application code, review lock acquisition order, and reduce nested locking or external calls while holding locks. Do not infer deadlock from high thread count alone.

### Q230. What is the difference between a heap dump and a thread dump?

A heap dump captures objects and references for retained-memory analysis. A thread dump captures thread states and stack traces for blocked threads, deadlocks, and executor problems. Choose the artifact that matches the symptom; collect them with care because dumps may be large and contain sensitive data.

### Q231. Why can ClassNotFoundException differ from NoClassDefFoundError?

ClassNotFoundException commonly occurs when code explicitly asks a loader to find a class and it cannot. NoClassDefFoundError is a linkage error raised when the runtime cannot find a class definition needed by already-compiled code, and can also appear after initialization failures. Check the earliest failure, classpath/module path, and class loader identity.

### Q232. Why can two classes with the same fully qualified name be incompatible?

Runtime type identity includes the defining class loader. Two classes with the same name loaded by different class loaders are distinct runtime types, which can cause confusing ClassCastException or linkage failures. This appears in plugin systems, application servers, and dependency isolation; inspect class loader origins rather than only class names.

---

---

## 11. I/O, networking, JDBC, security, testing, and performance

### Q233. How should Java I/O, NIO, and resource management be handled?

Classic `java.io` APIs provide stream- and reader/writer-oriented access, while `java.nio` adds buffers, channels, paths, and facilities for more flexible file and network I/O. NIO does not mean every operation is non-blocking: channels and APIs have different blocking and asynchronous contracts, so the chosen abstraction must match the workload.

Use try-with-resources for resources implementing `AutoCloseable`, such as streams, readers, channels, and JDBC objects. This ensures cleanup on normal and exceptional paths and preserves suppressed exceptions when close itself fails. Choose explicit charsets such as UTF-8 instead of relying on the platform default, and distinguish bytes, UTF-16 code units, Unicode code points, and user-perceived characters when processing text.

### Q234. What are the risks of Java native serialization?

Java's built-in object serialization can reconstruct object graphs from a byte stream, but deserializing untrusted input is dangerous because the stream can cause classes to be instantiated and methods such as deserialization hooks to execute in surprising ways. It also couples the data format to class structure and versioning details.

Avoid native Java deserialization for untrusted data where possible. Prefer an explicit, schema-oriented format such as JSON, Protocol Buffers, or another format appropriate to the system, and apply authentication, size limits, validation, and versioning. If native serialization must be used, apply strict deserialization filters and treat the input as a security boundary. Serialization is not encryption and does not establish trust.

### Q235. What should a Java backend engineer understand about JDBC transactions and connection pools?

JDBC provides lower-level database access through connections, prepared statements, and result sets. A connection's transaction behavior depends on autocommit and explicit commit/rollback; resource cleanup is essential, and prepared statements should be used for parameter binding rather than concatenating untrusted values into SQL.

In a pooled application, the pool manages physical connections while application code borrows and returns them. Do not close a shared pool itself from request code, hold connections while waiting on remote services, or assume a timed-out client request means the database statement stopped. Align connection, query, and transaction timeouts, and ensure rollback/cleanup occurs when failures happen.

### Q236. How should Java code be benchmarked and performance-tuned?

Start by defining the observed problem and a measurable target, then profile a representative workload to find the dominant cost. Use JMH for JVM microbenchmarks because warm-up, JIT compilation, dead-code elimination, constant folding, allocation, and measurement noise can invalidate naive timing loops. For application-level performance, use production-like load tests and observability rather than extrapolating from one microbenchmark.

Change one variable at a time and compare results against a baseline. Do not tune GC flags, object allocation, synchronization, or data structures based on intuition alone. Include latency percentiles, throughput, CPU, allocation rate, memory, and contention in the evaluation; an optimization that improves average throughput may still worsen tail latency or resource use.

### Q237. What should Java testing and build-tool compatibility cover?

Tests should cover behavior and contracts at appropriate levels: unit tests for isolated logic, integration tests for interactions, and concurrency tests designed around synchronization rather than arbitrary sleeps. JUnit and Mockito are common tools, but mocks should not replace tests of important framework wiring, database constraints, or real serialization behavior. Use deterministic coordination primitives and bounded timeouts when testing concurrent code.

Build configuration is also part of compatibility. Understand Maven/Gradle toolchains, the compiler's `--release` option, test plugins, dependency versions, and the difference between the JDK used to run the build and the bytecode/API level targeted by compilation. A project that compiles on a developer's newest JDK may still fail on the production runtime if its source/target/API compatibility is not configured and tested correctly.

---

### Q238. What security mistakes should Java backend developers avoid?

Security depends on the application's trust boundaries, not just on using a particular Java API. Validate and constrain untrusted input at the appropriate boundary, use parameterized SQL rather than string concatenation, avoid unsafe deserialization of untrusted data, prevent path traversal by resolving and validating paths against an intended root, and do not disable TLS certificate/hostname verification to “fix” a connection problem.

Use established cryptographic APIs and libraries instead of inventing algorithms, store secrets outside source control, apply least privilege, and avoid logging credentials, tokens, or sensitive payloads. For outbound requests, consider SSRF risks including redirects and DNS resolution behavior. A strong answer explains the threat model and layered controls; no single normalization or validation step guarantees safety in every environment.

### Q239. What should be configured when making HTTP calls from Java?

A production HTTP client needs explicit connection and request/response timeouts, sensible connection reuse/pooling where supported, TLS verification, redirect policy, resource cleanup, and observability. The standard `java.net.http.HttpClient` became a standard API in Java 11; older code may use other client libraries with their own configuration and lifecycle rules.

Timeouts should align with the overall request deadline, and retries should be bounded and restricted to operations that are safe to repeat. A timeout only means the caller did not receive a response in time; the remote service may still have completed the operation. Record useful metrics and correlation context without leaking secrets, and avoid creating a new client for every request when the client is intended to be reused.

### Q240. Why is the platform default charset a production risk?

The default charset can differ between environments, so code that relies on it may read or write text differently across machines. Specify a charset such as UTF-8 when encoding or decoding text unless the protocol explicitly requires something else. Also distinguish bytes from characters: network protocols and file formats define byte encodings, not an abstract Java string.

Use APIs that accept an explicit charset and test non-ASCII content. This avoids data corruption that may appear only when deployment locale, container image, or operating-system defaults change.

### Q241. What is the difference between byte streams and character streams?

Byte streams such as `InputStream` and `OutputStream` handle raw bytes and are appropriate for binary content. Character streams such as `Reader` and `Writer` handle text through character encoding and decoding. A text file is still stored as bytes, so the encoding must be known at the boundary.

Do not use a character reader for arbitrary binary content or convert bytes to strings without knowing the encoding. For large files, stream the content and manage buffers rather than loading everything into memory unnecessarily.

### Q242. What are Path and Files in NIO.2 useful for?

The `Path` API represents a filesystem path, and `Files` provides operations for reading, writing, copying, moving, and inspecting files. These APIs are generally more expressive than older `File` methods and support attributes, directory traversal, and explicit options. For large files, prefer streaming or buffered operations over convenience methods that read the entire file into memory.

When paths come from users, normalize and resolve them against an intended root and verify that the resolved target remains within that root. Symbolic links and filesystem races can complicate security, so path string checks alone are not a complete defense for sensitive file operations.

### Q243. How should JDBC resources and transactions be managed?

Use try-with-resources for `Connection`, `PreparedStatement`, and `ResultSet` where the surrounding transaction framework does not own their lifecycle. Use prepared statements for parameter values rather than concatenating user input into SQL. A transaction should group changes that must succeed or fail together, and failures should trigger the appropriate rollback behavior.

In frameworks such as Spring, understand whether the transaction is managed declaratively or manually and avoid closing a connection that the framework owns. Keep transactions reasonably short, define isolation requirements, and be careful about remote calls inside a database transaction because they extend lock and connection occupancy.

### Q244. What are transaction isolation levels, and why can anomalies occur?

Isolation defines which concurrent transaction effects are visible to a transaction. Common anomalies include dirty reads, non-repeatable reads, and phantom reads; the exact behavior and implementation details depend on the database. Higher isolation can reduce some anomalies but may increase contention, retries, or locking costs.

Choose isolation based on correctness requirements, not simply by selecting the strongest level everywhere. Understand the database's actual semantics and transaction mechanism, and test concurrency-sensitive workflows. Application-level checks can still race unless they are protected by suitable constraints, locking, or transactional design.

### Q245. Why are prepared statements safer than constructing SQL strings?

Prepared statements separate SQL structure from parameter values, allowing the database driver to treat supplied values as data rather than executable SQL syntax. This is a key defense against SQL injection for values. Dynamic identifiers such as table names or sort directions usually cannot be handled as ordinary bind parameters, so they should be selected from an allowlist of known identifiers.

Validation remains useful for business rules, but escaping or manual string replacement is not a reliable substitute for parameterized queries. Use the same principle for other interpreters: keep untrusted data separate from executable commands whenever the API permits it.

### Q246. What is deserialization risk in Java, and how can it be reduced?

Deserializing untrusted native Java object streams can trigger complex object construction and gadget behavior before application code has validated the result. The safest approach is generally to avoid native serialization for untrusted input and prefer a data format with an explicit schema and constrained types. If native serialization must be used, apply strict filtering and allowlisting appropriate to the application and JDK.

Do not assume that a signature alone makes every deserialization path safe, and do not deserialize arbitrary classes from external clients. Review dependency chains and keep serialization boundaries narrow.

### Q247. How do you test concurrent code without flaky sleeps?

Test synchronization conditions, not elapsed time guesses. Use latches, barriers, bounded waits, controlled executors, and deterministic setup to coordinate the interleavings relevant to the behavior. A timeout can prevent a hung test from running forever, but a fixed sleep does not prove that a particular thread reached a state.

For complex concurrency algorithms, consider stress testing and specialized tools in addition to unit tests. Ensure tests fail with useful diagnostics and clean up executors so a failed test does not leave background threads running.

### Q248. What is the difference between unit, integration, and contract testing in a Java service?

Unit tests isolate a small unit of behavior and run quickly. Integration tests verify that components work together, such as persistence mappings, serialization, configuration, and framework wiring. Contract tests verify agreed behavior at a boundary between independently developed components, such as a REST API or message schema.

Use the level that gives confidence in the relevant risk. Mocking every dependency can make tests fast but may miss configuration, SQL, transaction, and serialization problems. Conversely, using a full integration environment for every simple branch can make feedback slow and debugging difficult.

### Q249. What does good logging look like in a Java backend service?

Logs should explain significant events and failures with useful context such as correlation identifiers, operation names, durations, and safe identifiers. Use appropriate severity levels and avoid logging secrets, access tokens, personal data, or large unredacted payloads. Include the exception cause when it helps diagnosis, but avoid logging and rethrowing the same failure repeatedly at every layer.

Logging is not a replacement for metrics or traces. Metrics reveal rates and trends; traces connect work across service boundaries; logs provide event-level detail. Together they help narrow down production failures without relying on one noisy log line.

### Q250. How should a Java service handle configuration and secrets?

Separate environment-specific configuration from application code, validate required configuration at startup, and keep secrets in an appropriate secret-management system rather than source control or plain logs. Avoid silent fallbacks for critical settings because they can make a service start with unsafe or incorrect behavior.

Configuration should have clear ownership, safe defaults where possible, and documented units and timeouts. Treat configuration changes as operational changes: test them, audit sensitive changes, and expose enough sanitized effective configuration to diagnose behavior.

### Q251. How do TCP, message framing, and blocking I/O affect Java network code?

TCP provides a reliable byte stream, not message boundaries. A single read may return part of a message, one message, or multiple messages, so the application protocol must define framing—for example, a length prefix, delimiter with escaping, or fixed-size records. Assuming one read equals one request is a common networking bug.

Blocking I/O is straightforward but ties up a thread while waiting. Non-blocking I/O and selectors can multiplex many channels but require more complex state management. Virtual threads can make blocking code easier to scale for many I/O-bound tasks, but connection limits, downstream capacity, timeouts, and backpressure still need explicit handling.

### Q252. When should Java code use byte-oriented versus character-oriented I/O?

InputStream and OutputStream operate on bytes and suit binary data. Reader and Writer operate on characters through an encoding. Text is stored or transmitted as bytes, so the encoding must be explicit at the boundary. Do not decode arbitrary binary data as text or rely on a machine's default charset for a file format.

### Q253. Why should text encoding be specified explicitly?

The default charset can vary across environments and has changed in modern Java releases. Specify UTF-8 or the protocol's required charset when reading and writing text. Otherwise, characters may be corrupted only after deployment to a different container, host, or locale.

### Q254. What is the difference between Path and File?

Path, part of NIO.2, represents a path and works with the Files API for richer filesystem operations. File is an older abstraction with a smaller and less expressive API. Prefer Path and Files for new code, specify encodings for text, and stream large files rather than loading them entirely into memory.

### Q255. How do you protect against path traversal?

Resolve untrusted relative paths against an intended base directory, normalize them, and verify that the resulting path remains within the allowed root. Consider symbolic links and filesystem races for sensitive operations; checking whether a string contains ".." is not sufficient. Apply authorization and least privilege in addition to path validation.

### Q256. What is the difference between serialization and JSON mapping?

Java native serialization encodes object graphs using Java-specific mechanisms and can invoke complex object construction behavior during deserialization. JSON mapping usually converts between explicit data representations and Java types through a library. For untrusted input, avoid native deserialization where possible and use constrained data formats with validation.

### Q257. Why is deserializing untrusted Java objects dangerous?

Object deserialization can instantiate classes and invoke hooks before the application has validated the resulting object. Certain dependency chains can expose gadget behavior. Prefer explicit schemas and data transfer formats; if native serialization is unavoidable, apply strict filters and allowlists and keep the boundary narrow.

### Q258. What is a socket, and what does TCP guarantee?

A socket is an endpoint for network communication. TCP provides a reliable, ordered byte stream, but it does not preserve application message boundaries. The application protocol must define framing, such as a length prefix or delimiter. One read call is not guaranteed to return one complete message.

### Q259. What is the difference between connect timeout and request timeout?

A connect timeout limits how long establishing a connection may take. A request or response timeout limits some part of the operation after connection setup, depending on the client API. Production clients should have explicit timeouts aligned with an overall deadline; one timeout setting rarely covers every stage of a distributed request.

### Q260. Why is retrying a timed-out write dangerous?

A timeout means the caller did not receive a response in time, not that the remote operation definitely failed. The server may have committed the write and lost the response. Use idempotency keys, unique constraints, or durable operation identifiers for operations that must not be duplicated; retry only under a defined policy.

### Q261. What is JDBC, and what are Connection, PreparedStatement, and ResultSet?

JDBC is Java's standard API for database access. Connection represents a database session and transaction context, PreparedStatement represents a parameterized SQL statement, and ResultSet exposes query results. Use parameterized values, manage resources with try-with-resources when appropriate, and understand whether the framework or application owns the connection lifecycle.

### Q262. Why are PreparedStatement objects preferred over concatenated SQL?

Prepared statements separate SQL structure from parameter values and help prevent SQL injection for those values. Dynamic identifiers such as table names or sort directions generally cannot be bound as values, so choose them from a strict allowlist. Parameterization is not a substitute for business validation or authorization.

### Q263. What is transaction isolation?

Isolation controls how concurrent transactions observe each other's changes. Common anomalies include dirty reads, non-repeatable reads, and phantom reads, although exact behavior depends on the database and its implementation. Choose isolation based on correctness requirements and test concurrent workflows rather than assuming the strongest level is always best.

### Q264. What is the difference between commit and rollback?

Commit makes the transaction's changes durable according to the database's transaction contract. Rollback abandons uncommitted changes. In application code, ensure exceptions trigger the correct transaction outcome and do not catch an exception and accidentally commit partial work. Framework-managed transactions should be used consistently rather than mixing manual and declarative lifecycle control.

### Q265. What is connection pooling, and what can go wrong?

A connection pool reuses physical database connections to avoid repeatedly paying connection-establishment cost. Problems include pool exhaustion, leaks, too-large pool sizes that overload the database, and connections held during slow remote calls. Monitor active, idle, pending, and timeout metrics; size the pool with the database's capacity and workload in mind.

### Q266. Why should database transactions usually be short?

Long transactions hold connections and may retain locks or snapshots, increasing contention and reducing pool availability. Avoid performing slow remote HTTP calls while holding a database transaction open unless the consistency requirement truly demands it. Design boundaries so the critical database work is as short as correctness allows.

### Q267. What is SQL injection and how does Java code prevent it?

SQL injection occurs when untrusted input is interpreted as SQL syntax. Use PreparedStatement parameters for values, allowlist dynamic identifiers, apply least-privilege database permissions, and avoid leaking query details in public error responses. Manual escaping is fragile and should not replace parameterization.

### Q268. How should secrets be handled in a Java service?

Keep credentials out of source control, logs, and exception messages. Use an approved secret-management mechanism, apply least privilege, rotate secrets where supported, and avoid exposing them through diagnostics or configuration endpoints. Validate required configuration at startup and fail clearly rather than silently using an unsafe default.

### Q269. What is SSRF and why can it affect Java HTTP clients?

Server-side request forgery occurs when an attacker influences a server to make requests to unintended destinations. Validate destination hosts and schemes, account for redirects and DNS resolution, block access to sensitive internal address ranges where appropriate, and apply egress controls. A simple URL string check alone may not cover all routing behaviors.

### Q270. What is the difference between authentication and authorization?

Authentication establishes who or what is making a request; authorization determines which actions that identity is allowed to perform. A valid token or authenticated session does not imply access to every resource. Enforce authorization at the correct boundary and test both allowed and denied cases.

### Q271. How do you avoid leaking sensitive data through logs?

Log enough context to diagnose the event without including passwords, access tokens, private keys, or unnecessary personal data. Redact sensitive fields, constrain payload logging, and use correlation identifiers to connect events. Logs should complement metrics and traces rather than becoming a copy of all request and response bodies.

### Q272. What is the difference between unit and integration testing?

Unit tests verify a small piece of behavior in isolation and should be fast and focused. Integration tests verify interactions among components such as the database, serialization, configuration, and framework wiring. Use mocks where they clarify a unit contract, but do not mock away every integration boundary that could fail in production.

### Q273. What is contract testing?

Contract testing verifies that independently developed components honor an agreed interface, such as an HTTP API or message schema. It can catch incompatible request, response, or event changes before deployment. Contract tests complement, rather than replace, unit and integration tests.

### Q274. How do you test asynchronous code without arbitrary sleeps?

Use latches, futures, barriers, controlled executors, and bounded waits to coordinate the behavior being tested. A fixed sleep only guesses that a thread has reached a state and can make tests flaky. Ensure test failures include diagnostics and that executors and resources are cleaned up.

### Q275. What is the difference between a mock and a stub?

A stub provides predetermined responses; a mock also allows interaction verification in many test frameworks. Both can isolate a unit, but excessive mocking can couple tests to implementation details and hide integration defects. Prefer testing observable behavior and use real lightweight collaborators where practical.

### Q276. Why should performance tests use representative workloads?

Performance depends on input size, data distribution, concurrency, cache state, allocation, and downstream dependencies. A microbenchmark can isolate a small operation but may not predict end-to-end service behavior. Use JMH for JVM microbenchmarks and load tests with production-like traffic, data, and resource constraints for service-level questions.

### Q277. What should be measured before and after a performance optimization?

Measure the relevant outcome—such as throughput, p95/p99 latency, CPU, allocation rate, GC pauses, lock contention, and downstream saturation—against a baseline. Change one variable at a time where feasible and repeat tests to account for noise. An optimization that improves average throughput can still worsen tail latency or memory use.

### Q278. What is JMH and why is a naive benchmark misleading?

JMH is a harness designed for JVM microbenchmarking. Naive timing loops can be distorted by JIT warm-up, dead-code elimination, constant folding, allocation behavior, and measurement noise. Use proper warm-up, forks, and benchmark structure, and avoid extrapolating microbenchmark results directly to a whole service.

---

---

## 12. Java version evolution and migration: Java 8–25

This section focuses on interview questions about compatibility, upgrade decisions, and safe migration—not a second feature-by-feature release history. For the detailed Java 8–25 feature timeline, examples, and release status, see [Section 16: Java Version Features (Java 8–25)](#16-java-version-features-java-8-25).

### Q279. What is the difference between source, binary, and behavioral compatibility?

Source compatibility means source code continues to compile; binary compatibility means already-compiled code can continue to link; behavioral compatibility means the program continues to behave as expected. A change can preserve one form of compatibility while breaking another.

When upgrading a JDK or library, check API signatures, runtime behavior, dependency support, and integration tests. Compilation alone does not prove that a deployment is compatible.

### Q280. What should you consider when upgrading a Java service to a newer JDK?

Check supported JDK versions for frameworks and dependencies, build-tool and plugin compatibility, removed or encapsulated JDK internals, TLS and security defaults, charset and locale assumptions, JVM flags, container images, agents, and native libraries. Compile and test against the intended release, then run integration and performance tests in a production-like environment.

Compare startup time, memory, garbage collection, latency, and error rates against a baseline. Roll out gradually where possible and keep a rollback plan.

### Q281. What does the compiler option --release do?

The `--release` option selects the language rules, class-file target, and documented Java platform API surface for a specified release. It is generally safer than setting only `-source` and `-target` when building with a newer JDK, because it also restricts the platform APIs available to the compilation.

Use the release target that matches your compatibility commitment, and test on the actual runtime used in deployment. Third-party dependencies must also support that target.

### Q282. What is the difference between the classpath and module path?

The classpath uses the traditional class-loading model, where classes generally belong to the unnamed module. The module path supports named modules with explicit dependencies and package exports, enabling stronger encapsulation and more predictable dependency declarations.

When migrating to modules or diagnosing access errors, inspect the actual launch configuration and dependency graph. Do not treat broad reflective access to internal JDK packages as a permanent fix when a supported public API is available.

### Q283. What is the difference between an LTS release and a feature release?

An LTS release is one for which a vendor offers a longer support commitment; the duration and terms depend on the vendor and distribution. Feature releases deliver platform changes on a shorter cadence.

Choose a production baseline based on security support, framework and library compatibility, operational readiness, and upgrade cost—not the LTS label alone. Confirm the support policy for the exact JDK distribution you deploy.

### Q284. How should you evaluate a preview or incubating feature before adopting it?

Verify the feature's status in the exact target release, required compiler and runtime flags, compatibility expectations, and whether dependencies and production tooling support it. Preview features can change or be removed before finalization; incubating APIs likewise are not a stable standard API contract.

Adopt one only when its benefits justify the upgrade and maintenance cost. Document the dependency, test the deployment path, and have a plan to revise or remove it if the API changes. Use Section 16 and the official release documentation to check feature status.

### Q285. How would you plan a migration from Java 8 to a current LTS release?

Treat it as a compatibility and operational project rather than just a source-code update. Inventory the JDK, framework, libraries, build plugins, runtime flags, agents, and native dependencies. Check removed JDK modules and unsupported internal API use, then upgrade dependencies along supported paths.

Compile with an explicit release target, run unit and integration tests on the destination JDK, and test startup, security, performance, and production-like traffic. Where the distance between versions is large, use staged upgrades or intermediate baselines if that makes incompatibilities easier to isolate. Canary the rollout, compare metrics, and preserve a rollback path.

### Q286. What are the main reasons to upgrade from Java 17 or 21 to a newer LTS release?

Potential reasons include newer language and library capabilities, runtime and garbage-collection improvements, security updates, and continued framework or vendor support. The value depends on the workload, libraries, deployment environment, and support policy.

Build a small evidence-based case: identify the features or operational improvements that matter, verify dependencies, benchmark representative traffic, and quantify migration risk. Do not assume every application becomes faster simply by changing the JDK.

### Q287. How should a team introduce a new Java language or runtime feature safely?

Start with a real need and confirm the minimum JDK version and feature status. Check code style, framework support, build and IDE support, operational diagnostics, and whether the feature affects public APIs or deployment requirements. Keep adoption incremental and make sure the team can maintain the code.

For runtime or performance features, define baseline metrics and test under representative load. For preview features, explicitly document the non-final status and the cost of revisiting the implementation when upgrading.

### Q288. How do you distinguish a Java feature's introduction release from its final release?

A feature may first appear as preview or incubating, evolve over multiple releases, and only later become final—or be withdrawn. A reliable version reference should distinguish the initial release, preview/incubator iterations, finalization release, and any later status changes.

When answering an interview question, state the status in the specific release being discussed rather than saying a feature is simply “available since” the first release where it appeared. See Section 16 for the release-by-release reference and verify details against the official JDK documentation.

## 13. Code-reading and output questions

These questions are best practiced by predicting the result before reading the explanation. For each snippet, first decide whether it compiles, then work out the output or exception. Snippets are independent; assume each is placed in a suitable class and that required imports are present.

### Q289. What is printed when a method is overloaded with primitive, boxed, and varargs parameters?

```java
static void pick(long value)    { System.out.println("long"); }
static void pick(Integer value) { System.out.println("Integer"); }
static void pick(int... values) { System.out.println("varargs"); }

pick(5);
pick(Integer.valueOf(5));
pick();
```

Output:

```text
long
Integer
varargs
```

Overload resolution is performed at compile time. For `pick(5)`, primitive widening to `long` is considered before boxing to `Integer`; for an `Integer` argument, the `Integer` overload is an exact match. The varargs overload is used when no arguments are supplied. Resolve overloads from compile-time argument types and applicable conversion rules, not from a guess about runtime values.

### Q290. Why can string concatenation and numeric promotion produce surprising output?

```java
System.out.println(1 + 2 + "3");
System.out.println("1" + 2 + 3);
System.out.println('A' + 1);
```

Output:

```text
33
123
66
```

The first expression performs numeric addition before concatenating with the string. The second begins with a string, so the remaining operands are concatenated. A `char` participates in numeric arithmetic using its numeric value, so `'A' + 1` evaluates to the integer `66`.

### Q291. What is the difference between compile-time errors and runtime exceptions in code-output questions?

This code does not compile:

```java
int count = "5";
```

This code compiles, but throws a `NullPointerException` when executed:

```java
Integer count = null;
System.out.println(count + 1);
```

The first example violates Java's type rules. In the second, unboxing the null `Integer` to an `int` triggers the exception. Always determine whether the code compiles before tracing runtime behavior.

### Q292. What is printed when an overloaded method accepts int, long, and Integer?

```java
static void show(int value)     { System.out.println("int"); }
static void show(long value)    { System.out.println("long"); }
static void show(Integer value) { System.out.println("Integer"); }

show(7);
show(7L);
show(Integer.valueOf(7));
```

Output:

```text
int
long
Integer
```

Each argument has a different best match: an `int` literal selects `int`, a `long` literal selects `long`, and an `Integer` reference selects `Integer`. Overload resolution uses compile-time types and conversion rules; it does not inspect a wrapper's runtime value to choose a method.

### Q293. How can compile-time constants affect switch labels and output questions?

```java
static final int LIMIT = 10;

switch (value) {
    case LIMIT -> System.out.println("limit");
    default -> System.out.println("other");
}
```

A constant variable such as `LIMIT` can be used as a case constant. Replacing it with an ordinary variable assigned at runtime would make the label invalid. This example uses arrow-style switch labels, available from Java 14; account for the target release when analyzing syntax.

### Q294. Why can switch expressions be safer than traditional switch statements?

```java
enum Status { NEW, DONE }

Status status = Status.DONE;
String label = switch (status) {
    case NEW -> "new";
    case DONE -> "done";
};
System.out.println(label);
```

Output:

```text
done
```

A switch expression produces a value, and arrow cases do not fall through into the next case. For an enum, the compiler can check that all enum constants are covered. Switch expressions became final in Java 14; pattern matching for switch became final in Java 21, so do not confuse the release histories.

### Q295. How do you approach an unfamiliar Java output question?

Use a repeatable sequence: first check compilation; then identify compile-time types and overload resolution; trace initialization and operator precedence; distinguish overloading from runtime overriding; and finally follow control flow and exceptions.

For example, overloading is selected using compile-time types, while an overridden instance method is dispatched according to the runtime class:

```java
class Parent {
    void show() { System.out.println("Parent"); }
}
class Child extends Parent {
    @Override void show() { System.out.println("Child"); }
}

Parent value = new Child();
value.show();
```

Output:

```text
Child
```

The declared type `Parent` controls which members are available at compile time, but the overridden implementation is selected at runtime.

---

## 14. Production, senior-level scenarios, and engineering judgement

### Q296. A HashMap lookup unexpectedly returns null even though the key was inserted. What would you investigate?

First confirm that the same map instance is being queried and that the insertion succeeded. Then inspect the key's `equals()` and `hashCode()` implementations, whether fields used by them changed after insertion, whether a different but logically equivalent key is being used, and whether null is being confused with an absent mapping. `containsKey` can distinguish a present mapping to null from no mapping, though the larger issue may be key design.

Also check concurrency: unsynchronized writes to a shared HashMap are not safe. Reproduce with a minimal example and inspect the key's hash and equality behavior before replacing the data structure. The long-term fix is usually stable key semantics and correct synchronization, not retries.

### Q297. An API becomes slow under load, but CPU usage is low. How would you investigate?

Low CPU with high latency often suggests waiting rather than computation: thread pool exhaustion, blocked I/O, database connection-pool saturation, slow downstream services, lock contention, or requests queued behind a constrained resource. Compare request traces, active/queued threads, pool metrics, downstream latency, timeout counts, and thread dumps.

Check whether the service accepts work faster than it can complete it. Increasing the thread pool can worsen the problem if all tasks wait for the same limited database connections. Identify the bottleneck, apply backpressure or capacity limits, and tune the constrained resource based on measurements.

### Q298. A service experiences repeated garbage-collection pressure. What steps would you take?

Determine whether the issue is high allocation rate, a growing retained live set, an undersized heap, or pauses caused by the selected collector and workload. Review GC logs, allocation profiling, post-collection occupancy, request throughput, object lifetime, and container memory limits. JFR and heap dumps can provide complementary evidence.

Look for avoidable temporary allocations, unbounded caches, large buffers, and retained request data. Change one variable at a time and compare the same workload before and after. Collector or heap tuning can help, but should not conceal a leak or a capacity problem.

### Q299. How do you choose between synchronization, atomics, and concurrent collections?

Start with the invariant being protected. Use `synchronized` or a lock when multiple fields or operations must change as one consistent unit. Use atomic variables for small state transitions such as counters or compare-and-set updates. Use concurrent collections when the shared data structure's concurrency contract matches the workload.

Avoid choosing solely by benchmark claims. Atomics can suffer contention and are not a replacement for multi-field invariants; concurrent maps do not make their values thread-safe; locks can be simpler and more correct for compound updates. Make the correctness requirement explicit, then measure if performance is a concern.

### Q300. A task submitted to an executor never appears to finish. What would you check?

Inspect thread dumps and executor metrics to determine whether the task is running, blocked, queued, deadlocked, or waiting on a future that cannot complete. Check pool size, queue growth, rejection policy, nested task submission, and whether all pool threads are waiting for work submitted to the same saturated pool—a form of thread-starvation deadlock.

Observe task exceptions and completion paths; a task may have failed while the caller waits elsewhere. Set realistic timeouts, avoid blocking a pool's workers on tasks that require the same exhausted pool, and ensure cancellation and shutdown are handled. Increasing pool size without identifying the wait cycle can hide the symptom or make resource pressure worse.

### Q301. How should retries be designed when a remote call times out?

A timeout means the caller did not receive a result within the allowed time; it does not prove the remote operation failed. The remote side may have completed the operation while the response was delayed or lost. Blind retries can duplicate payments, orders, or other side effects.

Use bounded retries with backoff and jitter only when the operation is safe to retry or protected by an idempotency mechanism. Set an overall deadline, distinguish transient from permanent failures, and consider circuit breaking and load shedding. For writes, define how the client can query or deduplicate the outcome instead of assuming timeout equals failure.

---

### Q302. A production service has rising latency but normal CPU utilization. What do you check first?

Look for waiting rather than computation: thread-pool queue depth, blocked or parked threads, connection-pool saturation, slow database queries, downstream latency, lock contention, rate limits, and timeout/retry amplification. Compare request latency percentiles with dependency timings and resource metrics. A thread dump taken during the incident can reveal where request threads are waiting.

Avoid increasing thread counts blindly. If the database pool is exhausted, adding application threads can increase queueing and contention without increasing throughput. Find the constrained resource, bound waiting, and align deadlines across service and dependency calls.

### Q303. A service has high allocation rates but no obvious memory leak. How do you investigate?

High allocation and a memory leak are different problems. Frequent short-lived allocation can increase GC work even if objects are eventually reclaimed; a leak involves objects remaining reachable longer than intended. Use allocation profiling or JFR to identify allocation sites, correlate allocation rate with request volume and latency, and inspect GC behavior.

Optimize only after identifying the dominant allocation source. Reuse can help in selected hot paths, but unnecessary object pools often add complexity and contention. First consider algorithmic improvements, avoid repeated parsing or copying, and verify any change under a representative workload.

### Q304. A request times out, but the client retries and duplicate work occurs. How do you prevent it?

A timeout does not prove that the original operation failed; the server may have completed it while the response was lost. For operations that must not be duplicated, design idempotency into the boundary—for example, use an idempotency key stored with the operation result, unique constraints, or a durable state transition that makes repeated requests safe.

Retries should be bounded, use backoff and jitter, respect an overall deadline, and be applied only when repeating the operation is safe. Exactly-once effects across distributed systems are not achieved merely by retrying or using a message broker; they require careful coordination of durable state and side effects.

### Q305. A CompletableFuture chain hangs under load. What would you investigate?

Inspect which executor each stage uses, whether blocking calls occupy a limited pool, whether a stage waits on work scheduled to the same saturated executor, and whether a future is never completed on an error path. Look at thread dumps, executor metrics, queue depth, timeout behavior, and the chain's exception-handling stages.

Avoid calling blocking joins inside a constrained worker pool when dependent work needs that same pool to make progress. Make timeout and cancellation behavior explicit, and ensure exceptional completion cannot silently leave the request waiting forever.

### Q306. How would you choose between a concurrent collection and external synchronization?

Start with the shared invariant. If threads only need independent thread-safe map or queue operations, a concurrent collection may be sufficient. If correctness depends on a sequence of operations being atomic—for example, check-then-act across multiple map calls—use an atomic collection method, a suitable lock, or redesign the state transition so it is indivisible.

Thread-safe individual methods do not automatically make a multi-step workflow thread-safe. Document the invariant, choose the smallest synchronization boundary that protects it, and test concurrent interleavings that could violate it.

---

### Q307. What are p95 and p99 latency, and why are averages insufficient?

p95 latency is the value at or below which 95% of observed requests complete; p99 similarly describes the 99th percentile. Averages can hide a small fraction of extremely slow requests, even though those tail requests often dominate user experience and timeouts in distributed call chains.

Track percentiles alongside throughput, error rates, saturation, and dependency timings. Compare them over a meaningful window and sufficient request volume, and avoid treating one percentile as the whole story. Tail latency often points to queueing, contention, retries, garbage collection, or a slow dependency.

### Q308. A service starts slowly after a JDK upgrade. How do you investigate?

Compare cold-start timing and logs against a baseline, then inspect class loading, JIT warm-up, dependency initialization, configuration, network calls, and runtime flags. Use JFR or startup diagnostics where available. Separate application initialization from JDK/runtime changes and validate the cause before adding AOT or caching options.

### Q309. A service has many blocked threads. What do you investigate?

Capture thread dumps during the incident and group stacks by state and blocking location. Check connection pools, synchronized monitors, downstream waits, queue saturation, and lock contention. Correlate with request latency and resource metrics; increasing the thread count may worsen the issue if the true bottleneck is a database or remote dependency.

### Q310. How do you diagnose connection-pool exhaustion?

Inspect active, idle, pending, and timed-out connection metrics, then look for long-running queries, unclosed resources, long transactions, and slow remote work inside transaction boundaries. Confirm whether the pool size matches database capacity. Raising the pool size without understanding the database limit can make contention worse.

### Q311. How should retries be designed for transient failures?

Use bounded retries with backoff and jitter, honor a request deadline, and retry only failures that are plausibly transient. Writes need idempotency or deduplication so a timeout does not create duplicate effects. Avoid retry storms and align retry budgets across service layers so every layer does not independently multiply attempts.

### Q312. What is bulkheading in a Java backend?

Bulkheading isolates resource pools or concurrency limits so failure or saturation in one dependency does not consume all capacity for unrelated work. Separate executors, semaphores, connection pools, or bounded queues can provide isolation. Too many isolated pools increase operational complexity, so boundaries should follow real failure domains.

### Q313. What is graceful shutdown for a Java service?

Graceful shutdown stops accepting new work, allows in-flight work to complete within a deadline, closes executors and clients, and releases resources. It should also define what happens to queued tasks and interrupted work. Kubernetes termination grace periods and application shutdown hooks need to be configured consistently.

### Q314. How do you prevent duplicate processing of a Kafka or queue message in Java?

Design the handler to be idempotent where possible, use stable message or business identifiers, and coordinate durable state changes with acknowledgment or offset commits. A crash can occur between processing and acknowledgment, so delivery semantics alone do not guarantee exactly-once external side effects. Use transactions, deduplication records, or an outbox/inbox pattern where the domain requires it.

### Q315. What is the transactional outbox pattern, and why might a Java service use it?

The service writes its business change and an outgoing event record in the same database transaction. A separate publisher delivers the event and marks it processed, allowing retries after failures. This avoids the dual-write problem of committing a database change and publishing a message as unrelated operations, although duplicates still need handling.

### Q316. How do you make a scheduled Java job safe to retry?

Make the operation idempotent, persist progress or a stable execution identifier where necessary, define what happens after partial failure, and use bounded retry/backoff. In distributed deployments, ensure only the intended number of instances run the job or make concurrent runs safe. Log and measure each execution without exposing sensitive payloads.

### Q317. What is the difference between throughput and latency?

Throughput measures how much work completes per unit of time; latency measures how long an individual operation takes. Improving throughput can increase queueing and worsen tail latency if the system is saturated. Evaluate both alongside error rate and resource saturation rather than optimizing a single metric.

### Q318. Why are p95 and p99 latency important in distributed systems?

Percentiles reveal the tail of the latency distribution, which averages can hide. Slow requests can consume connection pools, trigger retries, and amplify delays across a call chain. Track percentiles with sufficient volume and correlate them with dependency timings, queueing, GC, and resource saturation.

### Q319. How do you choose between caching and recomputing a result?

Consider computation cost, access frequency, freshness requirements, memory use, invalidation complexity, and consistency. A cache can reduce latency but introduces stale-data and stampede risks. Start with measurement, define TTL or invalidation rules, and make cache misses and backend overload safe.

### Q320. What is a cache stampede and how can it be reduced?

A cache stampede occurs when many requests miss the same key and simultaneously recompute or fetch the value. Use single-flight coordination, bounded refresh concurrency, jittered expiration, stale-while-revalidate where appropriate, or prewarming. Ensure the coordination itself does not become a bottleneck or deadlock.

### Q321. How should a Java service manage configuration across environments?

Separate configuration from code, validate required settings at startup, keep secrets in a secret manager, and document units and timeout semantics. Avoid silent fallback values for critical security or reliability settings. Configuration changes should be observable, tested, and rolled out with the same care as code changes.

### Q322. How should you handle a third-party library upgrade?

Read release notes and compatibility guidance, review transitive dependency changes, compile and run unit/integration/contract tests, and test security-sensitive paths. Run representative performance tests if the library is on a hot path. Upgrade in a controlled change with a rollback option rather than bundling unrelated refactors.
---

## 15. Last-minute revision

Use these links to revisit a single canonical question instead of repeating its content.

1. Java execution, pass-by-value, and conversions — [Section 1](#1-java-foundations-and-execution), Q1–Q2 and Q3–Q11.
2. Object-oriented design and SOLID — [Section 2](#2-object-oriented-programming-contracts-and-design), Q13–Q17.
3. Strings, wrappers, equality, and numeric behavior — [Section 3](#3-strings-wrappers-numeric-behavior-and-equality), Q36–Q38 and Q39–Q43.
4. Exceptions and resource cleanup — [Section 4](#4-exception-handling-and-resource-safety), Q52 and Q53–Q56.
5. Generics and type-system rules — [Section 5](#5-generics-type-system-annotations-and-reflection), Q64–Q65 and Q67–Q68.
6. Collections and HashMap — [Section 6](#6-collections-framework-and-data-structures), Q80–Q86, Q87–Q91, and Q94–Q103.
7. Stream fundamentals and coding exercises — [Section 8](#8-stream-api-fundamentals-and-coding-questions), Q120–Q151.
8. Concurrency, the Java Memory Model, executors, and virtual threads — [Section 9](#9-concurrency-java-memory-model-executors-and-asynchronous-programming).
9. JVM, garbage collection, and diagnostics — [Section 10](#10-jvm-internals-garbage-collection-and-diagnostics).
10. I/O, JDBC, security, tests, and performance — [Section 11](#11-io-networking-jdbc-security-testing-and-performance).
11. Java-version milestones and migration — [Section 12](#12-java-version-evolution-and-migration-java-8-25).
12. Finish with [code-reading questions](#13-code-reading-and-output-questions) and [production/senior scenarios](#14-production-senior-level-scenarios-and-engineering-judgement).

### Q323. What should a final Java interview revision checklist include?

Revise language contracts, OOP and equality, strings and wrappers, exceptions, collections, generics, streams and coding exercises, date/time, concurrency and the Java Memory Model, executors and virtual threads, JVM and GC diagnostics, I/O/JDBC/security, testing, performance, and version changes. For each topic, practise one concise explanation, one edge case, one code example, and one production trade-off.

**Revision method:** Start with a concise explanation, then add the underlying mechanism, one edge case, and one practical trade-off.

---

## 16. Java Version Features (Java 8–25)

This section is a practical companion to the interview Q&A: it explains the Java features that are most useful to recognize in code, discuss in interviews, and adopt in backend services. It is organized by release rather than as a matrix.

**How to read the status labels**
- **Final:** available as a standard feature in that release; no preview flag is needed.
- **Preview:** available for evaluation in that release, but not yet a permanent language/API feature. Preview features can change or be withdrawn and normally require preview options at compile and run time.
- **Incubator:** an experimental API module that may change before becoming standard.

The examples are deliberately small snippets, not complete programs. Some snippets use APIs from a later release than the language syntax around them; the stated release identifies when the feature became available. Prioritize Java 8, 11, 17, 21, and 25 when preparing for LTS-oriented backend interviews.

### Java 8 — Functional programming and modern core APIs

Java 8 is a major milestone because it introduced a functional style to the language and standard library. Its features remain central to everyday Java backend development.

#### Lambda expressions and functional interfaces

A lambda provides an implementation of a functional interface: an interface with one abstract method. Lambdas reduce boilerplate for behavior passed to methods.

```java
Runnable task = () -> System.out.println("Running");
Comparator<String> byLength =
        (left, right) -> Integer.compare(left.length(), right.length());
```

Use lambdas with APIs such as Streams, executors, and collection callbacks. A lambda does not automatically make code faster; clarity and the behavior of the surrounding API still matter.

#### Method references

Method references are a compact form for lambdas that simply call an existing method or constructor.

```java
names.forEach(System.out::println);
List<String> normalized = names.stream()
        .map(String::toLowerCase)
        .collect(Collectors.toList());
```

For Java 8, the compatible terminal line is `collect(Collectors.toList())`. Method references can target static methods, instance methods, or constructors.

#### Stream API

Streams express data transformations as a pipeline of intermediate operations and a terminal operation. Intermediate operations are generally lazy.

```java
List<String> result = names.stream()
        .filter(name -> name != null && !name.trim().isEmpty())
        .map(String::trim)
        .map(String::toUpperCase)
        .collect(Collectors.toList());
```

This pipeline is compatible with Java 8. Avoid side effects in stream lambdas, and do not assume parallel streams are automatically faster.

#### Optional

`Optional<T>` represents a value that may be absent and encourages explicit handling of that absence.

```java
String displayName = findUser(id)
        .map(User::getName)
        .filter(name -> !name.isEmpty())
        .orElse("Unknown");
```

It is most useful as a return type when absence is an expected outcome. It is usually not a replacement for every nullable field, method parameter, or collection.

#### Default and static interface methods

Interfaces can provide default behavior and static utility methods, helping evolve APIs without forcing every existing implementation to add a method.

```java
interface Identified {
    String id();

    default boolean hasId() {
        return id() != null && !id().isEmpty();
    }

    static boolean isValid(String id) {
        return id != null && !id.isEmpty();
    }
}
```

A class can inherit defaults from interfaces, but conflicts between unrelated defaults must be resolved explicitly.

#### `java.time` date and time API

The modern date/time API provides immutable types with clearer semantics than the legacy `Date` and `Calendar` APIs.

```java
LocalDate today = LocalDate.now();
LocalDate dueDate = today.plusDays(30);
Instant eventTime = Instant.now();
ZonedDateTime localMeeting =
        eventTime.atZone(ZoneId.of("Asia/Kolkata"));
```

Use `Instant` for a point on the timeline, `LocalDate` for a calendar date without a time zone, and `ZonedDateTime` when regional time-zone rules matter.

#### CompletableFuture

`CompletableFuture` supports asynchronous composition without manually nesting callbacks.

```java
CompletableFuture<User> user = loadUserAsync(id);
CompletableFuture<String> name =
        user.thenApply(User::getName);
```

Know the difference between transformation (`thenApply`), asynchronous transformation (`thenApplyAsync`), and composition (`thenCompose`). The default async executor and blocking calls inside a chain deserve attention in production.

#### Other Java 8 additions worth knowing

- **Streams collectors:** `groupingBy`, `partitioningBy`, `mapping`, `joining`, and `toMap` are common interview and production tools.
- **Base64 API:** `Base64.getEncoder().encodeToString(bytes)` provides standard Base64 encoding.
- **Repeating annotations and type annotations:** improve annotation expressiveness for frameworks and tooling.
- **Nashorn JavaScript engine:** introduced in Java 8 but later deprecated and removed; do not assume it exists on modern JDKs.

### Java 9 — Modules and collection/API improvements

#### Java Platform Module System (JPMS)

The module system lets an application declare dependencies and control which packages it exposes. A module descriptor is named `module-info.java`.

```java
module com.example.orders {
    requires java.sql;
    exports com.example.orders.api;
}
```

`requires` declares a module dependency; `exports` makes a package accessible to other modules. Modules add stronger boundaries but do not mean every application must be converted to a modular deployment immediately.

#### Collection factory methods

`List.of`, `Set.of`, and `Map.of` create compact unmodifiable collections.

```java
List<String> roles = List.of("READER", "EDITOR");
Map<String, Integer> limits = Map.of("basic", 10, "pro", 100);
```

These collections reject null elements/keys/values and are unmodifiable. They are not the same as a mutable `ArrayList` or `HashMap`.

#### Private interface methods

Interfaces can use private helper methods to share implementation among default or static methods without exposing the helper as part of the public API.

#### Stream API additions

- `takeWhile` and `dropWhile` process an ordered stream up to or after the first element that fails a predicate.
- `ofNullable` creates an empty stream for a null value or a single-element stream otherwise.
- `iterate(seed, predicate, next)` supports a bounded iteration form.

#### Other Java 9 changes

- The `ProcessHandle` API provides information about and control over operating-system processes.
- The `Flow` API defines interfaces for reactive-stream-style publish/subscribe interactions.
- The platform's runtime image and internal APIs changed substantially; avoid depending on unsupported JDK internals.

### Java 10 — Local-variable type inference

#### `var` for local variables

Java 10 introduced local-variable type inference. The compiler infers the static type from the initializer; Java remains statically typed.

```java
var names = new ArrayList<String>();
var retries = 3;
var customer = customerRepository.findById(id);
```

Use `var` when the initializer makes the type clear. It cannot be used for fields, method parameters, or return types, and a declaration needs an initializer. Avoid it when it hides an important type or makes code harder to read.

### Java 11 — Standard HTTP client and everyday API improvements

Java 11 is an LTS release and a common migration target from Java 8.

#### Standard `HttpClient`

The standardized client supports synchronous and asynchronous HTTP requests and common HTTP features.

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder(URI.create("https://example.com"))
        .GET()
        .build();

HttpResponse<String> response =
        client.send(request, HttpResponse.BodyHandlers.ofString());
```

The snippet is synchronous. For service code, consider timeouts, status codes, connection behavior, and whether asynchronous execution is appropriate.

#### String API additions

Java 11 added methods such as `isBlank()`, `strip()`, `stripLeading()`, `stripTrailing()`, `lines()`, and `repeat()`.

```java
boolean emptyOrWhitespace = input.isBlank();
String normalized = input.strip();
String banner = "-".repeat(20);
```

`strip()` uses Unicode-aware whitespace handling, while `trim()` follows older, narrower behavior.

#### Files API additions

`Files.readString(path)` and `Files.writeString(path, content)` simplify common text-file operations. Choose an explicit charset when the file format requires one, and avoid reading very large files wholly into memory.

#### Other Java 11 changes

- Local-variable syntax (`var`) is permitted for lambda parameters, useful when annotations are needed.
- The Java EE and CORBA modules were removed from the JDK distribution; applications relying on them may need external dependencies.
- The single-file source launch mode makes small scripts easier to run, but it does not replace a build system for a service.

### Java 12 — Switch-expression preview and collector additions

#### Switch expressions (preview in Java 12)

The initial switch-expression preview allowed a `switch` to produce a value. The syntax and details evolved in later previews before becoming final in Java 14. Do not treat the Java 12 preview as the final specification.

#### `Collectors.teeing`

The collector combines the results of two downstream collectors into one result, useful when two aggregates should be calculated in a single collection pass.

```java
var summary = values.stream().collect(
        Collectors.teeing(
                Collectors.minBy(Integer::compareTo),
                Collectors.maxBy(Integer::compareTo),
                (min, max) -> List.of(min, max)));
```

This example uses `var` and `List.of`, so it assumes Java 10 and Java 9 APIs respectively. In real code, handle the empty-stream case explicitly because min/max may be absent.

### Java 13 — Text blocks and switch-expression preview iterations

#### Text blocks (preview)

Text blocks introduced a more readable way to represent multiline strings. They were previewed in Java 13 and became final in Java 15.

```java
String json = """
        {
          "status": "ok",
          "count": 3
        }
        """;
```

At the preview stage, syntax and escape behavior were not yet final. For modern code, follow the finalized Java 15 rules and pay attention to incidental indentation and trailing newlines.

#### Switch expressions (second preview)

The switch-expression preview was revised, including the use of `yield` to return a value from a multi-statement case block. See the finalized Java 14 form below.

### Java 14 — Switch expressions finalized

#### Switch expressions (final)

Switch expressions let `switch` return a value. Arrow labels do not fall through, and a multi-statement branch uses `yield` to provide its result.

```java
String category = switch (statusCode) {
    case 200, 201 -> "Success";
    case 400, 404 -> "Client error";
    case 500, 503 -> "Server error";
    default -> {
        logUnknownStatus(statusCode);
        yield "Other";
    }
};
```

This is useful for concise, exhaustive mappings. Unlike a traditional colon-style switch statement, arrow cases do not accidentally continue into the next case.

#### Helpful NullPointerExceptions

The JVM can provide more informative messages about which expression was null in certain null dereferences. This improves diagnosis but does not replace null validation or clear error handling.

#### Records and pattern matching for `instanceof` (preview)

Both features were previewed in Java 14 and finalized in later releases. For production code, use their finalized forms as described under Java 16.

### Java 15 — Text blocks finalized

#### Text blocks (final)

Text blocks became a permanent language feature in Java 15, making multiline SQL, JSON, HTML, and test fixtures easier to read.

```java
String query = """
        SELECT id, name
        FROM customer
        WHERE status = 'ACTIVE'
        ORDER BY name
        """;
```

Text blocks do not parameterize SQL. Continue to use prepared statements for user-supplied values; a readable multiline query is not inherently safe from injection.

#### Hidden classes

Hidden classes are intended mainly for frameworks and runtime-generated classes that are not meant to be discovered by ordinary application code. Most business applications do not need to define hidden classes directly.

### Java 16 — Records and pattern matching for instanceof finalized

#### Records

Records provide a compact declaration for transparent data carriers. The compiler supplies component fields, accessors, a canonical constructor, `equals`, `hashCode`, and `toString`.

```java
public record UserSummary(long id, String name) {}
```

A record is only **shallowly immutable**: a component can refer to a mutable object. Records are a good fit for DTOs and value-like data, but they are not a drop-in replacement for every entity or class.

#### Pattern matching for `instanceof`

Pattern matching combines a type check and cast, reducing boilerplate.

```java
if (value instanceof String text && !text.isBlank()) {
    System.out.println(text.strip());
}
```

The pattern variable is in scope where the match is known to be true. This example uses Java 11 String methods as well.

#### `Stream.toList()` and `mapMulti`

- `Stream.toList()` returns an unmodifiable list of the stream's elements.
- `mapMulti` can emit zero or more output elements per input element, sometimes avoiding the overhead of creating a nested stream for each item.

Do not assume `Stream.toList()` returns a mutable list; use `Collectors.toCollection(ArrayList::new)` when you specifically need a mutable result.

### Java 17 — Sealed classes and a modern LTS baseline

#### Sealed classes and interfaces (final)

Sealed types restrict which classes or interfaces may extend or implement them. Permitted subtypes must declare an appropriate relationship such as `final`, `sealed`, or `non-sealed`.

```java
public sealed interface PaymentResult
        permits Approved, Declined {}

record Approved(String transactionId) implements PaymentResult {}
record Declined(String reason) implements PaymentResult {}
```

Sealed hierarchies model a controlled set of alternatives. Combined with records and pattern matching, they can make domain modeling and exhaustive handling clearer.

#### Strong encapsulation of JDK internals

Java 17 strongly encapsulates many internal JDK APIs. Applications that relied on unsupported internals may fail or emit warnings after an upgrade. Prefer supported public APIs and investigate reflective-access errors rather than relying on broad JVM escape flags as a permanent fix.

#### Pattern matching for switch (preview)

Java 17 introduced the first preview of pattern matching for `switch`. The feature evolved across releases and became final in Java 21. Use the finalized syntax in modern examples.

### Java 18 — UTF-8 by default

#### Default charset

Java 18 standardized UTF-8 as the default charset in most standard Java APIs, reducing platform-dependent behavior. Explicitly specify a charset at file, network, and integration boundaries when the external format defines one; explicit intent remains valuable.

```java
String text = Files.readString(path, StandardCharsets.UTF_8);
```

#### Simple web server and finalization deprecation

Java 18 included a minimal command-line web server useful for local testing, not as a production server. Object finalization was deprecated for removal; use explicit resource management, try-with-resources, or appropriate cleanup mechanisms instead.

### Java 19 — Virtual threads and record-pattern previews

#### Virtual threads (preview)

Virtual threads were introduced as a preview in Java 19 and became final in Java 21. They are lightweight threads intended to support large numbers of mostly-blocking tasks in a thread-per-request style.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> result = executor.submit(() -> callRemoteService());
    System.out.println(result.get());
}
```

This executor API requires Java 21 for the finalized virtual-thread support shown here; Java 19's preview required preview enablement. Virtual threads improve scalability for blocking workloads, not CPU-bound throughput by themselves. Avoid pooling virtual threads as though they were scarce platform threads.

#### Record patterns (preview)

Record patterns began as a preview in Java 19, enabling components of records to be matched and unpacked declaratively. The syntax evolved before becoming final in Java 21.

#### Structured concurrency (incubator)

Structured concurrency explored treating related concurrent tasks as one unit of work, with coordinated cancellation and failure handling. It was an incubating API at this stage and continued to evolve; do not confuse it with the finalized virtual-thread feature.

### Java 20 — Continued concurrency and pattern previews

Java 20 was primarily an iteration release for several preview and incubator features rather than a major new LTS baseline.

- **Virtual threads:** continued preview refinements.
- **Record patterns:** preview refinements and better type inference.
- **Pattern matching for switch:** another preview iteration.
- **Structured concurrency and scoped values:** continued incubator/preview evolution.

For interviews, understand the problems these features target, but clearly distinguish the Java 20 preview/incubator status from the finalized Java 21 versions where applicable.

### Java 21 — Virtual threads, pattern matching, and sequenced collections

Java 21 is an important LTS release for backend developers.

#### Virtual threads (final)

Virtual threads make thread-per-task designs more scalable when tasks spend much of their time waiting on I/O. They are scheduled by the runtime over a smaller set of platform threads.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> handleRequest(request));
}
```

The try-with-resources block waits for submitted tasks to complete when the executor closes. In a real service, manage task lifetimes, cancellation, timeouts, downstream connection pools, and error propagation deliberately. Virtual threads do not remove downstream capacity limits.

#### Pattern matching for switch (final)

A switch can select behavior by runtime type and use guards with `when`.

```java
static String describe(Object value) {
    return switch (value) {
        case null -> "null";
        case String text when text.isBlank() -> "blank text";
        case String text -> "text: " + text;
        case Integer number -> "integer: " + number;
        default -> "other";
    };
}
```

The compiler checks coverage in many switch expressions. Order patterns carefully because a broader pattern can dominate a narrower one.

#### Record patterns (final)

Record patterns let code match a record and bind its components in one step.

```java
if (value instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);
}
```

Nested record patterns can unpack nested data structures. Use them when the structure is known and the resulting code is clearer than repeated accessor calls.

#### Sequenced collections

Sequenced collection interfaces provide a consistent way to access elements in encounter order, including first/last elements and reversed views.

```java
SequencedCollection<String> names = new ArrayList<>(List.of("Ada", "Linus", "Grace"));
String first = names.getFirst();
String last = names.getLast();
SequencedCollection<String> reversedView = names.reversed();
```

The reversed result is a reverse-ordered view, not necessarily a separately copied collection. Mutability and view behavior depend on the underlying collection.

#### Other Java 21 features

- **Generational ZGC:** improves garbage-collection efficiency by separating young and old object generations.
- **String templates (preview):** introduced as a preview but later withdrawn; do not treat them as a permanent Java feature.
- **Unnamed patterns and variables (preview):** underscore placeholders can mark values that are intentionally unused; this feature became final in Java 22.
- **Scoped values (preview):** explored immutable data sharing across a bounded call context and became final in Java 25.

### Java 22 — Foreign Function & Memory API and unnamed variables

#### Foreign Function & Memory (FFM) API (final)

The FFM API provides a supported way to access native memory and call native functions without relying on JNI for many use cases.

It matters for libraries that integrate with native code or need controlled access to off-heap memory. It is lower-level than ordinary business application code; memory lifetime and safety remain important.

#### Unnamed variables and patterns (final)

An underscore can mark a variable or pattern that is intentionally unused.

```java
try {
    process();
} catch (IOException _) {
    logFailure();
}
```

This avoids misleading placeholder names when the value is deliberately ignored. Use it only when the value truly is not needed; otherwise, handle or log the meaningful information.

#### Other Java 22 changes

- The Stream Gatherers API was introduced as a preview for custom intermediate stream operations and became final in Java 24.
- The Class-File API was introduced as a preview and became final in Java 24.
- Several foreign-function and concurrency improvements continued to evolve.

### Java 23 — Continued language and API previews

Java 23 continued work on newer language features and tooling APIs.

- **Module import declarations (preview):** proposed a concise way to import exported packages from a module; finalized in Java 25.
- **Primitive types in patterns, instanceof, and switch (preview):** explored matching primitive values using pattern constructs; still preview in Java 25.
- **Markdown documentation comments (final):** Java 23 finalized Markdown-style `///` documentation comments for Javadoc.
- **Stream Gatherers (preview iteration):** custom intermediate stream transformations continued to evolve before Java 24 finalization.

For interview answers, explain the goal and status accurately rather than presenting a preview API as a stable production contract.

### Java 24 — Stream Gatherers and the Class-File API finalized

#### Stream Gatherers (final)

Gatherers allow developers to define custom intermediate operations for stream pipelines, complementing built-in operations such as `map`, `filter`, and `flatMap`.

They are useful for transformations that are awkward to express with the built-in operations, such as custom windowing or stateful grouping. Prefer built-in operations when they already express the intent clearly.

#### Class-File API (final)

The Class-File API provides a supported API for parsing, generating, and transforming Java class files. It is relevant to bytecode tools, agents, frameworks, and language tooling—not something most application developers need in routine business logic.

#### Ahead-of-time class loading and linking

Java 24 introduced ahead-of-time class loading and linking capabilities intended to reduce startup work in suitable applications. The benefit depends on application characteristics and deployment workflow; measure startup and runtime behavior rather than assuming every service will improve.

#### Other Java 24 changes

- More improvements to garbage collection and runtime performance.
- Continued preview iterations for flexible constructor bodies, module imports, and primitive patterns.
- The Vector API remained an incubating API; incubating status means it is not a stable standard API contract.

### Java 25 — New LTS release and finalized language improvements

Java 25 is the next LTS release after Java 21 in this guide's scope. It finalizes several features that had been previewed in earlier releases.

#### Module import declarations (final)

A module import declaration can import the packages exported by a module, reducing repetitive imports for certain applications.

```java
import module java.base;
```

This is a language feature with specific rules; it does not mean all packages from every dependency become accessible without module or class-path configuration. Use it when it improves clarity, and keep ordinary explicit imports where they better communicate dependencies.

#### Compact source files and instance main methods (final)

Java 25 finalizes a simplified source-file form intended to make small programs easier to write. It reduces ceremony for introductory tools and scripts; traditional classes and explicit `main` methods remain fully valid and are still common in production applications.

#### Flexible constructor bodies (final)

This feature permits certain statements before an explicit superclass constructor invocation, while preserving restrictions that protect object initialization. It helps validate or prepare constructor arguments before calling `super(...)`, but it does not permit arbitrary access to an incompletely initialized instance.

#### Scoped Values (final)

Scoped values provide a way to share immutable data with callees for a bounded dynamic scope. They can be useful for context such as request metadata and work naturally with structured concurrency.

```java
private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

ScopedValue.where(REQUEST_ID, requestId)
        .run(() -> processRequest());
```

Scoped values are not a universal replacement for every `ThreadLocal`. They are designed for scoped, typically immutable context propagation and should be evaluated against the framework's request-context mechanisms.

#### Key Derivation Function API (preview)

Java 25 introduced the KDF API as a preview API for deriving keys from input keying material and optional additional data. It requires preview-feature enablement and may change in a later release. Use established cryptographic guidance and approved algorithms rather than inventing a custom key derivation scheme.

#### Features that remain preview or incubating in Java 25

- **Primitive types in patterns, `instanceof`, and `switch`:** still a preview feature in Java 25; do not present it as finalized.
- **Structured concurrency:** preview API, still evolving.
- **Vector API:** incubating API, still evolving.
- **Key Derivation Function API:** preview in Java 25; it is not a finalized Java SE API in that release.
- **Stable Values:** preview in Java 25.
- **PEM Encodings of Cryptographic Objects:** preview in Java 25.
- **Other preview features:** check the exact JDK release documentation and build/runtime flags before experimenting.

### LTS migration checklist: Java 8 → 11 → 17 → 21 → 25

When discussing a Java upgrade, focus on more than syntax:

1. **Build and runtime compatibility:** verify the compiler release, target bytecode, JDK used by CI, container base image, and production runtime.
2. **Dependencies and frameworks:** check supported JDK ranges, removed Java EE/CORBA modules, reflective access, bytecode tooling, agents, and native libraries.
3. **Behavior changes:** review charset assumptions, TLS/security defaults, garbage collector defaults, time-zone data, and locale-sensitive behavior.
4. **Concurrency:** assess executor sizing and blocking behavior before adopting virtual threads; test downstream pools and cancellation.
5. **Observability and performance:** establish baseline startup, throughput, latency percentiles, allocation rate, and GC behavior, then compare under representative load.
6. **Rollout strategy:** run tests on the target JDK, canary the deployment, monitor error and latency changes, and keep a rollback path.

### Quick revision: features interviewers commonly compare

| Comparison | Core distinction |
|---|---|
| Lambda vs method reference | A method reference is a concise form when the lambda merely calls an existing method or constructor. |
| Stream vs collection | A collection stores data; a stream describes a computation over a source. |
| `var` vs dynamic typing | `var` infers a static compile-time type; Java is not dynamically typed. |
| `List.of()` vs `Arrays.asList()` | `List.of()` is unmodifiable and rejects nulls; `Arrays.asList()` is fixed-size and backed by the array. |
| Switch statement vs switch expression | An expression produces a value; arrow cases avoid fall-through. |
| Record vs ordinary class | Records are concise, shallowly immutable data carriers with generated value-oriented methods. |
| Sealed vs final class | A final class allows no subclasses; a sealed type allows a declared, controlled set. |
| Platform vs virtual thread | Platform threads map closely to OS threads; virtual threads are lightweight runtime-managed threads. |
| Preview vs final feature | Preview features require explicit opt-in and may change; final features are standard for that release. |
| ScopedValue vs ThreadLocal | Scoped values model bounded context sharing; ThreadLocal associates mutable state with a thread. |

### Official version references

Use the official release documentation to verify details, especially when a feature changed from preview to final or was withdrawn.

- [Java language changes by release — Java 25](https://docs.oracle.com/en/java/javase/25/language/java-language-changes-release.html)
- [Java language changes summary — Java 25](https://docs.oracle.com/en/java/javase/25/language/java-language-changes-summary.html)
- [Java SE 25 API documentation](https://docs.oracle.com/en/java/javase/25/docs/api/)
- [Java SE specifications index](https://docs.oracle.com/javase/specs/)
- [JEP 361 — Switch Expressions](https://openjdk.org/jeps/361)
- [JEP 395 — Records](https://openjdk.org/jeps/395)
- [JEP 409 — Sealed Classes](https://openjdk.org/jeps/409)
- [JEP 431 — Sequenced Collections](https://openjdk.org/jeps/431)
- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
- [JEP 485 — Stream Gatherers](https://openjdk.org/jeps/485)
- [JEP 506 — Scoped Values](https://openjdk.org/jeps/506)

---

## 17. Official references

- [Java 25 language changes by release](https://docs.oracle.com/en/java/javase/25/language/java-language-changes-release.html)
- [Java 25 language feature status summary](https://docs.oracle.com/en/java/javase/25/language/java-language-changes-summary.html)
- [JDK 25 release notes](https://www.oracle.com/java/technologies/javase/25all-relnotes.html)


Use official specifications and API documentation as the final authority for language and runtime contracts. Secondary interview guides are useful for discovering recurring themes, not for proving technical behavior or guaranteeing company-specific frequency.

- [Java SE 25 specifications](https://docs.oracle.com/en/java/javase/25/docs/specs/)
- [Java SE 25 API documentation](https://docs.oracle.com/en/java/javase/25/docs/api/)
- [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
- [Java Virtual Machine Specification, Java SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)
- [Java SE 21 API documentation](https://docs.oracle.com/en/java/javase/21/docs/api/)
- [Virtual Threads adoption guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [Java Microbenchmark Harness (JMH)](https://openjdk.org/projects/code-tools/jmh/)
- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 440 — Record Patterns](https://openjdk.org/jeps/440)
- [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
- [JEP 431 — Sequenced Collections](https://openjdk.org/jeps/431)
- [Java SE specifications index](https://docs.oracle.com/javase/specs/)
- [Java interview question themes (secondary source)](https://www.crackr.dev/interview-guides/java-interview-questions)
- [Java interview question set (secondary source)](https://coddy.tech/interview-questions/java)

---

---

## 18. Editorial and validation notes

- The handbook contains **355 numbered interview questions**. Stable Q IDs are retained for linkability, but questions are grouped by topic and therefore are not shown in numeric order.
- Every question has one primary topic location; the last-minute revision section links back rather than duplicating question content.
- The section order follows one learning path: language/OOP, strings and exceptions, generics, Collections, functional APIs and Streams, concurrency, JVM, engineering APIs and practices, Java-version evolution, code reading, then production/senior scenarios.
- The Java release matrix highlights LTS milestones and preview/incubator caveats. It is a revision map, not a replacement for release notes.
- Structural checks cover question inventory, topic assignment, section organization, TOC anchors, and Markdown fences. They do not establish that every code snippet was compiled or every technical claim was independently audited line by line.
- The Java directory was consolidated to this single handbook; the former chapter-wise Markdown files and folders were removed as part of the requested cleanup. Other technology guides outside the Java directory were left untouched.
