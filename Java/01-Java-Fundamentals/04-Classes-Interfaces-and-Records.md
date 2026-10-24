# Classes, Interfaces, Nested Types, Records and Sealed Hierarchies

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21, with release status called out for newer language features.

This chapter focuses on type declarations and the boundaries they create: classes, interfaces, nested types, enums, records, sealed classes, and the trade-offs between nominal modeling options. Always distinguish what the language guarantees from style conventions.

## 1. Class and interface design

### Q1. What is the difference between a class and an object? *(Beginner)*

A class is a declaration that defines a reference type, including its members and construction rules. An object is a runtime instance of a class or another concrete reference type. Objects have identity; their state and behavior depend on the class and the instance's fields.

A class does not mean that every object must have mutable fields. Immutable classes, records, and enums can represent values or fixed alternatives. The right model depends on whether the domain concept needs identity, state transitions, or value-based equality.

### Q2. What are access modifiers in Java? *(Intermediate)*

Java has public, protected, package-private (no modifier), and private access. Broadly, public is accessible wherever the declaring type and module are accessible; package-private is accessible within the same package; protected adds subclass access with additional receiver restrictions across packages; private is accessible within the declaring top-level nest under the language's nested-type rules.

Access control is not a complete security boundary for an application. Reflection, modules, runtime configuration, and deployment choices affect what can be accessed. Use the narrowest access that supports the intended API and tests.

### Q3. What is the difference between a top-level class and a nested class? *(Intermediate)*

A top-level class is declared directly in a compilation unit. A nested class is declared inside another class or interface. Nested classes can express a close conceptual relationship and may access private members of the enclosing type according to Java's access rules.

Nested types include static member classes, non-static inner classes, local classes, and anonymous classes. Their scope, lifetime, capture behavior, and generated implementation differ. Choose nesting to express ownership or hide implementation details, not merely to avoid creating a file.

### Q4. What is the difference between a static nested class and an inner class? *(Intermediate)*

A static nested class does not require an enclosing instance and cannot directly access the enclosing object's instance members without a reference. A non-static inner class is associated with an enclosing instance and can access that instance's members.

A static nested class is often preferable when no enclosing instance is semantically required. Inner classes can unintentionally retain a reference to a large outer object, which matters for long-lived callbacks, caches, and asynchronous tasks.

### Q5. What are local and anonymous classes? *(Intermediate)*

A local class is declared within a block, such as a method. An anonymous class declares and instantiates a class expression without naming the class, commonly to provide a one-off implementation of an interface or abstract class.

Local and anonymous classes can capture local variables only when those variables are final or effectively final. Anonymous classes have their own this, referring to the anonymous instance; a lambda expression's this refers to the enclosing context. This distinction can affect callback behavior.

### Q6. What does effectively final mean? *(Intermediate)*

A local variable or parameter is effectively final if it is not reassigned after initialization, even if it is not explicitly declared final. Lambdas and local/anonymous classes can capture such local variables.

The captured value is not a live mutable local-variable cell. Java restricts reassignment because local variables may outlive the method activation through a closure-like object. For mutable shared state, use an object designed for that purpose and apply appropriate synchronization or atomicity; an array or holder is not automatically thread-safe.

### Q7. What is the difference between a constructor and a method? *(Beginner)*

A constructor initializes a new instance and has the class's simple name with no return type. It is invoked as part of object creation and can chain to another constructor or a superclass constructor. A method declares behavior, has a return type (including void), and is invoked on a receiver or class depending on whether it is instance or static.

Constructors are not inherited or overridden. They can be overloaded. If no constructor is declared, Java may provide a default no-argument constructor, subject to the class's inheritance and access context; declaring any constructor means that default constructor is not implicitly supplied.

### Q8. What is a package, and why should package design matter? *(Intermediate)*

A package groups related types under a namespace and defines the boundary for package-private access. Good package design communicates cohesive concepts and limits accidental coupling. Package names also participate in fully qualified type names and interact with modules and class loaders.

Do not treat package-private as a universal module boundary: code in the same package and suitable runtime context can access package-private members. In modular applications, JPMS exports and opens add another layer of access control.

## 2. Interfaces and type contracts

### Q9. Can an interface contain fields, constructors, or instance state? *(Intermediate)*

An interface cannot be instantiated and has no ordinary per-instance fields or constructors. Fields declared in an interface are implicitly public, static, and final. Interfaces can declare abstract methods, default methods, static methods, and private helper methods in modern Java.

Interface constants are often discouraged as a general-purpose configuration mechanism because they expose implementation details to all implementers and callers. Prefer methods for behavior and appropriately scoped constants in a class or dedicated type.

### Q10. How does @FunctionalInterface help preserve an interface's lambda contract? *(Intermediate)*

A functional interface has exactly one abstract method that counts as its functional method, ignoring methods that are public methods of Object and considering inherited declarations under the language rules. It can be implemented by a lambda expression or method reference. The @FunctionalInterface annotation asks the compiler to verify the intended contract.

Default and static methods do not add another abstract method. Functional interfaces are the target types of lambdas; lambdas do not have a standalone nominal class declaration in source.

### Q11. What is the difference between an abstract class and an interface when designing an API? *(Advanced)*

An abstract class can carry instance state, constructors, protected helpers, and a shared implementation with one superclass inheritance slot. An interface represents a contract that many unrelated classes can implement, supports multiple inheritance of type, and can evolve through carefully designed default methods.

Choose based on semantic relationship, state ownership, extension expectations, and compatibility. Do not use an abstract base class only because it feels more object-oriented, and do not create interfaces for every class if no useful abstraction or substitution point exists.

### Q12. How do default methods affect interface evolution? *(Advanced)*

Default methods allow an interface to add behavior without requiring every existing implementer to provide an implementation immediately. This was important for evolving widely implemented APIs such as the collections framework.

Compatibility is not automatic in every case: inherited default conflicts can require an explicit override, and a default implementation can be semantically wrong for a particular implementation. Keep default methods broadly valid for the contract and avoid adding behavior that depends on hidden assumptions about implementers.

### Q13. How are overloaded methods selected when null is passed? *(Advanced)*

The compile-time type of the argument and overload-resolution rules determine the selected method. The null literal can be converted to any reference type, so unrelated overloads can make a call ambiguous.

    static void show(String value) { System.out.println("String"); }
    static void show(Integer value) { System.out.println("Integer"); }
    // show(null); // compile-time ambiguity
    show((String) null); // selects show(String)

Primitive parameters cannot receive null without unboxing, and unboxing a null wrapper throws NullPointerException. Prefer overload sets that remain understandable for null and generic call sites.

## 3. Enums and modeling finite choices

### Q14. What is an enum in Java, and why is it safer than integer constants? *(Intermediate)*

An enum defines a fixed set of named instances with a distinct enum type. It improves readability and type safety compared with unvalidated integer or string constants, and it can include fields, constructors, and methods.

    enum OrderStatus {
        CREATED, PAID, SHIPPED, CANCELLED
    }

Enum values are instances, not integers. Avoid persisting ordinal() as a durable business identifier because reordering or inserting enum constants can change ordinals. Persist a stable explicit code or name according to a deliberate compatibility policy.

### Q15. Can an enum have a constructor and methods? *(Intermediate)*

Yes. An enum can have fields, constructors, and methods; enum constructors are not invoked by application code and enum instances are created by the enum declaration. Enum constructors cannot be public or protected.

Enums are useful for finite strategies when each constant can implement or override behavior. If the set of implementations must be extensible by external modules, an interface or another open abstraction may be more appropriate.

### Q16. What are common pitfalls when using enum ordinal values? *(Advanced)*

Ordinal is the declaration position starting from zero. If an application stores ordinals in a database, event stream, or file, reordering enum constants can reinterpret existing data. Ordinals are useful for limited internal indexing but generally not stable external identifiers.

Use explicit, versioned codes for persistence and wire formats. Also plan how unknown future values are handled by older consumers, especially in distributed systems where producers and consumers may deploy independently.

## 4. Records and data modeling

### Q17. What is a record, and when should you use one? *(Intermediate)*

Records became a permanent feature in Java 16 after preview releases. A record is a concise declaration for a transparent carrier of a fixed set of values. The compiler derives component fields, accessors, a canonical constructor, equals, hashCode, and toString according to record rules.

    public record Point(int x, int y) {}

Records are implicitly final and extend Record. They are suitable for data-transfer values, query results, and domain value objects when their fixed component model fits. They are not simply a shorter syntax for every class, particularly when the type needs mutable identity or inheritance.

### Q18. Are records immutable? *(Advanced)*

Record component fields are final, so a record's component references cannot be reassigned after construction. But a component can refer to a mutable object, so a record is not necessarily deeply immutable.

    public record Request(List<String> tags) {}

If callers can mutate the original list, the record observes those changes unless it defensively copies the list in a compact or canonical constructor. Records provide shallow immutability of component fields, not automatic deep immutability of the entire object graph.

### Q19. What is a compact constructor in a record? *(Intermediate)*

A compact constructor allows validation or normalization of record components without explicitly listing and assigning every parameter. The compiler supplies the component assignments after the compact constructor body completes normally.

    public record Percentage(int value) {
        public Percentage {
            if (value < 0 || value > 100) {
                throw new IllegalArgumentException("value must be 0..100");
            }
        }
    }

Use it to enforce invariants at the boundary. If a component needs transformation, assign the compact-constructor parameter to the normalized value; that value is then used for the generated field assignment.

### Q20. Can a record extend another class or declare additional instance fields? *(Advanced)*

A record cannot extend an arbitrary class because records implicitly extend java.lang.Record. It may implement interfaces. Its instance state is defined by record components; it cannot declare extra non-static instance fields beyond those components.

A record can declare static fields, static methods, instance methods, and permitted constructors. If a type needs hidden mutable instance state unrelated to its components, an ordinary class is likely a better model.

### Q21. How do records compare values, and what should you know about equals/hashCode? *(Advanced)*

The generated equals and hashCode are based on the record components and their defined equality behavior. Two record instances with equal component values are generally equal, even though they are different objects. Component types still matter: arrays, for example, use array reference equality through their ordinary equals implementation rather than deep content equality.

If the intended value semantics differ from the generated component-based semantics, reconsider the type model or explicitly implement the methods where allowed. A record should normally communicate its state transparently through its components.

## 5. Sealed hierarchies and pattern matching

### Q22. What is a sealed class or interface? *(Advanced)*

Sealed classes and interfaces became permanent in Java 17 after preview releases. A sealed type restricts which other types may directly extend or implement it using a permits clause or permitted declarations in the same compilation context where the rules allow inference.

Each permitted direct subtype must declare whether it is final, sealed, or non-sealed. This lets API designers model a controlled set of variants while retaining a nominal type hierarchy. Sealed does not mean immutable, and it does not automatically make every operation exhaustive.

### Q23. What is the difference between final, sealed, and non-sealed? *(Advanced)*

Final prevents further subclassing or implementation of the type. Sealed restricts direct subtypes to an explicitly permitted set. Non-sealed reopens an extension branch beneath a sealed parent, allowing that subtype's descendants to be unrestricted by the parent's permits list.

Choose final for a closed leaf, sealed for a controlled family of variants, and non-sealed when a particular branch should be extensible. The modifier expresses API evolution policy, not merely implementation preference.

### Q24. How do records and sealed interfaces work together? *(Advanced)*

A sealed interface can define a closed family of value variants, and records can implement each variant.

    sealed interface Shape permits Circle, Rectangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double width, double height) implements Shape {}

This models alternatives explicitly and can pair well with pattern matching. Validate domain invariants in each record, such as nonnegative dimensions. A sealed hierarchy only constrains direct subtype declarations; each permitted subtype's final/sealed/non-sealed status controls the next level.

### Q25. What is pattern matching for instanceof? *(Intermediate)*

Pattern matching for instanceof became permanent in Java 16. It combines a type test with a variable binding, avoiding a separate cast when the pattern succeeds.

    if (value instanceof String text) {
        System.out.println(text.length());
    }

The pattern variable's scope depends on control-flow analysis. It is not simply visible everywhere after the if statement. Java 21 also finalized pattern matching for switch, which is a distinct feature with its own rules.

### Q26. What is pattern matching for switch, and what changed by Java 21? *(Advanced)*

Pattern matching for switch became permanent in Java 21 after preview iterations. It allows switch labels to match patterns and can express type-based branching, guarded cases, and null handling under the specified rules. Combined with sealed hierarchies, the compiler can help reason about whether the alternatives are exhaustive.

Version matters: pattern matching for switch was preview in earlier releases, so compiling with an older JDK may require preview flags or may not support the final syntax. State the exact JDK when answering compatibility questions, and consult the release-specific language documentation.

### Q27. What is an exhaustive switch, and why is it useful? *(Advanced)*

An exhaustive switch covers all values permitted by the selector's type under the language's rules, or includes an appropriate default branch where required. For enums and sealed hierarchies, the compiler can sometimes detect coverage of the known alternatives.

Exhaustiveness makes adding a new variant more visible during compilation and can reduce forgotten branches. Runtime evolution, binary compatibility, and default behavior still need deliberate thought when independently deployed components exchange values.

## 6. Applied design and diagnostics

### Q28. When should you choose a record, an ordinary class, an enum, or a sealed hierarchy? *(Senior)*

Use a record for a transparent fixed set of value components with suitable generated value semantics. Use an ordinary class when identity, controlled mutation, hidden state, complex lifecycle, or inheritance behavior is central. Use an enum for a finite set of singleton constants. Use a sealed hierarchy when the domain has a controlled family of variants that should be represented by distinct types.

These options are not mutually exclusive: records can implement sealed interfaces, and ordinary classes can implement interfaces. Choose based on domain semantics, invariants, compatibility, and how callers need to use the type.

### Q29. How can public API design create accidental compatibility problems? *(Senior)*

Changing a public method signature, removing a constructor, changing inheritance, adding a conflicting default method, or changing a serialized representation can break source or binary compatibility. Even adding a method can create overload ambiguity or conflict with existing implementers in some designs.

Before changing a public API, consider source compatibility, binary compatibility, behavioral compatibility, and data/wire compatibility separately. Add compile tests against representative clients and review release notes. A change that compiles inside one repository may still break downstream consumers.

### Q30. How would you model a payment result with success, decline, and technical failure? *(Senior)*

A sealed interface with separate record variants can make the alternatives explicit:

    sealed interface PaymentResult permits Paid, Declined, PaymentFailure {}
    record Paid(String transactionId) implements PaymentResult {}
    record Declined(String reasonCode) implements PaymentResult {}
    record PaymentFailure(String incidentId, boolean retryable) implements PaymentResult {}

This avoids ambiguous combinations such as success=true with a failure reason. Validate required fields and decide which data is safe to expose. Consumers can use pattern matching to handle each variant, while the API owner controls whether external extension is permitted. For Java 21, pattern matching for switch is finalized; older source levels require a different approach.

## Official references

1. [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
2. [JLS Chapter 8 — Classes](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html)
3. [JLS Chapter 9 — Interfaces](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html)
4. [JLS Chapter 14 — Statements, including switch and pattern matching](https://docs.oracle.com/javase/specs/jls/se21/html/jls-14.html)
5. [JEP 395 — Records](https://openjdk.org/jeps/395)
6. [JEP 409 — Sealed Classes](https://openjdk.org/jeps/409)
7. [JEP 394 — Pattern Matching for instanceof](https://openjdk.org/jeps/394)
8. [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
9. [Java SE 21 language changes](https://docs.oracle.com/en/java/javase/21/language/java-language-changes-release.html)

## Chapter checklist

- [ ] Explain access control, nested types, captures, and constructor semantics.
- [ ] Choose classes and interfaces based on contract and state requirements.
- [ ] Model finite alternatives with enums and stable external codes.
- [ ] Explain record shallow immutability and constructor validation.
- [ ] Distinguish final, sealed, and non-sealed types.
- [ ] Identify Java release/status for records, sealed types, and pattern matching.
- [ ] Evaluate source, binary, behavioral, and wire compatibility separately.
