# Object-Oriented Programming Principles

**Target level:** Beginner → Senior  
**Scope:** Java 8–21

Object-oriented programming is more than memorizing encapsulation, inheritance, polymorphism, and abstraction. Strong interview answers connect these ideas to Java's type system, dynamic dispatch, object invariants, API design, testability, and change isolation.

## 1. Core concepts

### Q1. What is object-oriented programming, and what problems does it help solve? *(Beginner)*

Object-oriented programming (OOP) organizes software around objects that combine state with behavior. A class defines a type and describes how instances behave; an object is a runtime instance with identity and state where applicable.

OOP can help model domain concepts, protect invariants, isolate implementation details, and provide substitutable implementations behind stable interfaces. It is not automatically superior to procedural or functional styles. The best design depends on the problem, and an object-heavy design can create unnecessary indirection and mutable state.

### Q2. What are the four commonly named pillars of OOP? *(Beginner)*

The common list is encapsulation, abstraction, inheritance, and polymorphism.

- **Encapsulation:** Keep state and the operations that maintain its invariants behind a controlled boundary.
- **Abstraction:** Expose essential behavior while hiding irrelevant implementation details.
- **Inheritance:** Derive a subtype from a supertype, inheriting or specializing behavior where the subtype relationship is valid.
- **Polymorphism:** Use a value through a common type while the actual implementation selected at runtime can vary.

These are teaching categories, not a substitute for design reasoning. A good answer explains why a design benefits from the concept and what trade-offs it introduces.

### Q3. What is encapsulation, and is making fields private enough? *(Intermediate)*

Encapsulation means controlling how an object's state can be observed or changed so that its invariants remain valid. Private fields are a common mechanism, but a class can still be poorly encapsulated if it returns mutable internal objects or offers setters that permit invalid combinations of state.

    public final class BankAccount {
        private long balanceInCents;

        public BankAccount(long openingBalanceInCents) {
            if (openingBalanceInCents < 0) {
                throw new IllegalArgumentException("negative opening balance");
            }
            this.balanceInCents = openingBalanceInCents;
        }

        public synchronized void deposit(long amount) {
            if (amount <= 0) throw new IllegalArgumentException("amount");
            balanceInCents = Math.addExact(balanceInCents, amount);
        }

        public synchronized long balanceInCents() {
            return balanceInCents;
        }
    }

This example shows validation and a simple synchronization policy; a production financial domain would also need transaction, persistence, authorization, and currency rules. The principle is to make invalid states difficult to represent.

### Q4. What is abstraction, and how does it differ from encapsulation? *(Intermediate)*

Abstraction focuses on what a caller needs to know: the capabilities and contract exposed by a component. Encapsulation focuses on protecting the implementation and state behind that contract. They reinforce one another but are not identical.

For example, a payment interface can expose authorize, capture, and refund operations without exposing provider-specific HTTP requests, credentials, retries, or serialization. The implementation is encapsulated; the provider-independent API is an abstraction. A useful abstraction should be based on stable domain concepts rather than simply wrapping every method of an underlying library.

### Q5. What is inheritance in Java? *(Beginner)*

Class inheritance uses extends to derive a subclass from one direct superclass. The subclass inherits accessible members and can add state or behavior, override eligible instance methods, and specialize the inherited contract. Java classes have single class inheritance, although a class can implement multiple interfaces.

Inheritance creates a strong relationship between types and can expose a subclass to assumptions embedded in its superclass. Use it when the subtype genuinely satisfies the supertype's contract, not merely to reuse a few lines of code.

### Q6. What is polymorphism? *(Beginner)*

Polymorphism lets code work with a common supertype while runtime behavior can vary according to the actual object. Overriding instance methods provides runtime dispatch.

    interface Notifier {
        void send(String message);
    }
    final class EmailNotifier implements Notifier {
        public void send(String message) { System.out.println("email: " + message); }
    }
    final class SmsNotifier implements Notifier {
        public void send(String message) { System.out.println("sms: " + message); }
    }

    static void alert(Notifier notifier) {
        notifier.send("service degraded");
    }

The alert method depends on the interface, not on the concrete notifier. New implementations can be introduced without changing the method, provided they honor the contract.

### Q7. What is the difference between compile-time and runtime polymorphism? *(Intermediate)*

Overloading is resolved at compile time using the declared types and applicable overload-resolution rules. Overriding of instance methods uses dynamic dispatch at runtime based on the actual receiver class, subject to access, signature, and other language rules.

    class Parent {
        void show(Object value) { System.out.println("Object"); }
    }
    class Child extends Parent {
        @Override void show(Object value) { System.out.println("Child"); }
        void show(String value) { System.out.println("String"); }
    }

    Parent p = new Child();
    p.show("x"); // Child

The compiler sees Parent's overload set, so it selects show(Object). Runtime dispatch then selects Child's override of that signature. The show(String) overload is not considered merely because the runtime object is Child.

## 2. Inheritance and substitution

### Q8. What is the difference between extends and implements? *(Beginner)*

A class uses extends to inherit from a class, and it uses implements to promise conformance to one or more interfaces. An interface can extend multiple interfaces. A class can extend only one class but can implement multiple interfaces.

The distinction is about the type relationship and contract. An interface is often preferable when the desired relationship is capability or role rather than sharing class state or implementation. Abstract classes can still be useful for shared protected behavior or state when that coupling is intentional.

### Q9. What is method overriding, and what rules must be respected? *(Intermediate)*

Overriding provides a subtype-specific implementation of an inherited instance method with a compatible signature. The overriding method cannot reduce accessibility, and its return type must be compatible (including covariant reference returns). Checked exceptions declared by the overriding method cannot be broader than those allowed by the overridden method. Static methods are hidden rather than overridden.

Use the @Override annotation so the compiler catches signature mistakes. An overload with a different parameter list is a separate method, even if its name looks similar.

### Q10. What is covariant return typing? *(Intermediate)*

An overriding method may return a subtype of the return type declared by the overridden method.

    class Animal {}
    class Dog extends Animal {}
    class Kennel {
        Animal resident() { return new Animal(); }
    }
    class DogKennel extends Kennel {
        @Override Dog resident() { return new Dog(); }
    }

This lets callers using the concrete subtype receive a more specific result while preserving substitutability for callers using the base type. Primitive return types do not support covariance.

### Q11. What is the Liskov Substitution Principle (LSP)? *(Advanced)*

LSP means that instances of a subtype should be usable wherever the supertype is expected without breaking the guarantees clients rely on. In practical terms, a subtype must honor the supertype's behavioral contract, including preconditions, postconditions, invariants, and relevant exceptional behavior.

A classic warning is making a subtype strengthen preconditions or weaken postconditions. If a base type promises that a method accepts any positive amount but a subtype rejects most positive amounts, callers written against the base contract can fail. Inheritance should express a true behavioral subtype, not just a superficial “is-a” label.

### Q12. Why can inheritance violate encapsulation? *(Advanced)*

A subclass may depend on protected methods, overridable methods, initialization order, or internal assumptions of its superclass. A change to the superclass can unexpectedly affect subclasses, even if its public API appears unchanged. Calling overridable methods from constructors is particularly risky because subclass state may not yet be initialized.

Prefer small, stable extension points. Use composition when the goal is reuse without exposing an inheritance contract, and make classes final when subclassing is not intended.

### Q13. What is the difference between composition and inheritance? *(Intermediate)*

Inheritance models a subtype relationship and exposes inherited behavior as part of the subtype's contract. Composition builds a component from collaborators that it owns or delegates to. Composition generally allows implementation choices to change without changing the public type hierarchy.

    final class AuditLogger {
        void record(String event) { /* write audit event */ }
    }
    final class OrderService {
        private final AuditLogger auditLogger;
        OrderService(AuditLogger auditLogger) {
            this.auditLogger = auditLogger;
        }
        void placeOrder() {
            // business logic
            auditLogger.record("order placed");
        }
    }

The dependency can be replaced with a test double or another implementation if modeled behind an interface. Composition is not always better; use inheritance when substitutability is real and the base class offers a well-defined extension contract.

### Q14. What are the risks of calling an overridable method from a constructor? *(Advanced)*

The constructor of a superclass can dispatch to an overridden method in the subclass before the subclass constructor body has run. The override may observe default field values or violate invariants because subclass initialization is incomplete.

    class Base {
        Base() { printState(); }
        void printState() {}
    }
    class Child extends Base {
        private String name = "ready";
        @Override void printState() { System.out.println(name); }
    }

During Base construction, the override can print null because the Child field initializer has not executed yet. Avoid overridable calls from constructors; use private/final helpers or explicit initialization after construction.

### Q15. What is the difference between an abstract class and an interface? *(Intermediate)*

An abstract class can hold instance state, constructors, concrete methods, and abstract methods. An interface defines a type contract and can declare abstract methods, default methods, static methods, and private helper methods in modern Java. Interfaces do not have ordinary per-instance fields; declared fields are implicitly public static final constants.

Use an abstract class when closely related implementations need shared state or protected implementation. Use an interface for a capability or contract that unrelated classes can implement. A type can extend one class and implement many interfaces.

### Q16. Can an interface have default methods, and why were they introduced? *(Intermediate)*

Default methods provide an implementation that implementing classes inherit unless they override it. They allow interfaces to evolve by adding certain behaviors without forcing every existing implementation to add a method immediately.

Default methods do not eliminate all compatibility risks. Conflicts can arise when unrelated interfaces provide the same default signature, and implementers may need to resolve them explicitly. Default methods should express behavior that makes semantic sense for the abstraction, not serve as a dumping ground for unrelated utility methods.

### Q17. How are conflicting interface default methods resolved? *(Advanced)*

If a class inherits competing defaults with the same signature from unrelated interfaces, it must resolve the conflict by overriding the method. A class implementation takes precedence over interface defaults. A more specific subinterface's default can take precedence over a less specific interface's default. Explicitly invoking a particular interface default uses the syntax InterfaceName.super.method() from a permitted context.

Resolve conflicts deliberately and document the semantic choice. A method signature shared by two interfaces does not guarantee that their default implementations have compatible meaning.

### Q18. What are final classes, final methods, and final variables? *(Intermediate)*

A final class cannot be subclassed. A final instance method cannot be overridden. A final variable can be assigned only once according to Java's definite-assignment rules; for a reference variable, final prevents reassignment of the reference, not mutation of the referenced object.

    final List<String> names = new ArrayList<>();
    names.add("Ada"); // allowed
    // names = new ArrayList<>(); // compile-time error

Final is useful for expressing intent and restricting extension, but it does not make a referenced object immutable or automatically thread-safe.

## 3. Object lifecycle and class members

### Q19. What is the difference between static and instance members? *(Intermediate)*

An instance field or method is associated with an object instance. A static field or method belongs to the class-level declaration rather than to one instance. Static methods have no this reference and cannot directly access instance members without an object reference.

Static state can introduce global coupling, initialization-order problems, and concurrency concerns. Use it for behavior or data that genuinely belongs at class scope, not as a default replacement for dependency injection or object state.

### Q20. What is the difference between method overloading and overriding? *(Intermediate)*

Overloading uses the same method name with different parameter lists and is resolved at compile time. Overriding replaces an inherited instance method implementation with a compatible implementation selected by runtime dispatch. Return type alone cannot distinguish overloaded methods.

Static methods are hidden rather than overridden. Private methods are not inherited as overridable methods. Constructors can be overloaded but are not overridden.

### Q21. Can static methods be overridden? *(Intermediate)*

No. Static methods are resolved based on the compile-time type used to name the method and can be hidden by a static method declared in a subclass. They are not dispatched based on the runtime receiver object.

When a static method is called through a reference (which Java permits but discourages stylistically), the declared type determines which method is selected. Prefer calling static methods through the class name to make the behavior obvious.

### Q22. What is constructor chaining? *(Intermediate)*

A constructor can delegate to another constructor in the same class with this(...), or invoke a superclass constructor with super(...). This centralizes initialization and ensures superclass state is initialized before subclass-specific initialization. In Java 8–21 traditional constructor syntax requires explicit constructor invocation to occur first, subject to the language version's rules; Java 22 introduced flexible constructor bodies as a later feature, outside this handbook's Java 8–21 baseline.

Do not duplicate validation across constructors. Delegate toward one constructor that establishes the class invariant. Ensure every constructor path initializes required state before the object escapes.

### Q23. What is the initialization order for static and instance members? *(Advanced)*

At a high level, class initialization runs superclass initialization first, then the class's static field initializers and static initializer blocks in textual order. When an instance is created, memory is default-initialized, superclass construction runs, then the class's instance field initializers and instance initializer blocks execute in textual order before the constructor body.

The full rules include interfaces, constant variables, constructor delegation, and initialization triggers. Use the JLS rather than assuming every class is eagerly initialized at application startup. Circular initialization dependencies can expose default values and cause surprising behavior.

## 4. Equality, identity and object contracts

### Q24. What is the difference between object identity and logical equality? *(Intermediate)*

Identity asks whether two references designate the same object, tested with == for references. Logical equality asks whether objects represent equivalent values, commonly expressed through equals. Two distinct String objects can be logically equal while having different identities.

Design equals around the domain's value semantics. Identity may be important for entities with lifecycle identity, while value objects typically compare their meaningful state. Do not use reference equality for strings or other value-like classes unless identity is specifically intended.

### Q25. What contract should equals obey? *(Advanced)*

The equals contract requires it to be reflexive, symmetric, transitive, consistent while relevant state is unchanged, and false when compared with null. It should also work with hashCode: equal objects must have equal hash codes. The contract is documented by Object.

Inheritance complicates equality because adding state in a subclass can break symmetry or transitivity. For value objects, consider final classes or composition, and implement equality using a deliberate policy rather than comparing arbitrary fields automatically.

### Q26. Why must hashCode be consistent with equals? *(Advanced)*

Hash-based collections use hash codes to locate candidate buckets and then equality to distinguish keys. If two objects are equal but have different hash codes, a HashMap or HashSet may fail to find or deduplicate them correctly.

Equal objects must return equal hash codes; unequal objects are allowed to collide. Hash codes need not remain constant across different process executions, but they should remain stable while fields used by equality are unchanged. Avoid mutable keys whose equality/hash state changes after insertion.

### Q27. Why is using a mutable object as a HashMap key dangerous? *(Advanced)*

If a key's fields used by equals/hashCode change after insertion, the map may search the wrong bucket or fail to locate the entry. The entry may still physically exist in the map but become effectively unreachable through normal lookup.

Prefer immutable key types. If mutation is unavoidable, remove the key before changing equality-relevant state and reinsert it afterward. This is an operational workaround, not as robust as an immutable key design.

### Q28. What is immutability, and how does it help? *(Intermediate)*

An immutable object cannot have its observable state changed after construction. Immutability simplifies reasoning, supports safe sharing, reduces accidental coupling, and makes hash-based keys reliable. A final class with final fields is a common starting point, but it is not sufficient if mutable internals are exposed.

Defensive copies may be required when accepting or returning arrays, collections, dates, or other mutable objects. A read-only wrapper is not necessarily an immutable snapshot if another reference can mutate the backing collection.

### Q29. How would you design an immutable class? *(Advanced)*

A typical approach is to make the class final (or carefully restrict subclassing), keep fields private and final, validate constructor inputs, avoid setters, and defensively copy mutable inputs and outputs. Ensure that any nested objects are themselves immutable or safely encapsulated.

    public final class Team {
        private final List<String> members;
        public Team(List<String> members) {
            this.members = List.copyOf(members);
        }
        public List<String> members() {
            return members;
        }
    }

List.copyOf creates an unmodifiable list and rejects null elements; it does not make the elements deeply immutable. Deep immutability depends on the entire reachable object graph.

## 5. Design principles and senior-level reasoning

### Q30. What is the difference between inheritance for reuse and inheritance for substitutability? *(Senior)*

Inheritance for reuse says “this implementation is convenient to borrow.” Inheritance for substitutability says “this subtype satisfies the behavioral promises of the supertype.” Only the latter establishes a sound type relationship.

Reusing implementation alone can create fragile hierarchies: subclasses become dependent on internal details and can break when the base class changes. Prefer composition for code reuse unless the subtype relationship and extension contract are both valid.

### Q31. What is the Open/Closed Principle? *(Advanced)*

The Open/Closed Principle says software entities should be open to extension but closed to repeated modification for every new variation. In Java, interfaces, composition, strategy objects, and carefully designed extension points can allow new behavior without editing a large conditional in a central method.

It is not a command to predict every future requirement or create an abstraction for every branch. Add extension points where change is likely and meaningful; otherwise, a simple implementation is often cheaper and clearer.

### Q32. What is the Single Responsibility Principle? *(Advanced)*

A class should have one coherent reason to change, often described as responsibility to one actor or source of change. A class that validates orders, persists them, sends emails, and formats reports is coupled to multiple concerns that evolve independently.

The principle is not “one method per class.” Splitting too aggressively can create a maze of tiny types and indirection. Aim for cohesive responsibilities and boundaries that make testing, change, and ownership clearer.

### Q33. How do you apply Dependency Inversion in Java? *(Advanced)*

High-level policy should depend on abstractions rather than directly on low-level implementation details; both can depend on a stable contract. Java interfaces and constructor injection are common ways to make this explicit.

    interface PaymentGateway {
        void charge(long cents);
    }
    final class CheckoutService {
        private final PaymentGateway gateway;
        CheckoutService(PaymentGateway gateway) {
            this.gateway = gateway;
        }
        void checkout(long cents) {
            gateway.charge(cents);
        }
    }

CheckoutService can be tested with a fake gateway and can use different production adapters. An interface is valuable when it represents a meaningful boundary; adding an interface to every class without a change or testing benefit can add noise.

### Q34. When should you prefer composition over inheritance? *(Senior)*

Prefer composition when behavior is a replaceable capability, when the relationship is not a genuine behavioral subtype, when you want to combine multiple collaborators, or when inheritance would expose unstable implementation details. Prefer inheritance when the subtype contract is real and the superclass explicitly supports extension.

Evaluate substitutability, change frequency, testing, lifecycle, state ownership, and the cost of additional indirection. Neither approach is universally correct; the goal is to minimize accidental coupling while keeping the design understandable.

### Q35. How do you avoid creating an anemic domain model or an over-engineered object model? *(Senior)*

An anemic model stores data in objects while all meaningful rules live in unrelated service classes. That can be appropriate for simple data-transfer structures, but it may scatter invariants for a rich domain. At the other extreme, forcing every data structure into a hierarchy creates ceremony without adding useful behavior.

Place behavior with the state and invariants it protects when that improves cohesion. Keep orchestration and infrastructure concerns at suitable boundaries. Use DTOs for transport when appropriate, and domain objects for business rules that benefit from explicit ownership and validation.

### Q36. How would you review a proposed class hierarchy in a code review? *(Senior)*

Ask whether every subtype can honor the base contract, whether callers need the abstraction, whether protected hooks and constructor behavior are safe, and whether subclassing is an intentional supported extension point. Check equality, state mutability, initialization order, and how future changes to the base affect subclasses.

Request examples of valid and invalid subtype behavior. If the hierarchy exists only to share implementation, consider composition. If the hierarchy is justified, document invariants and test contract behavior across implementations.

## Official references

1. [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
2. [JLS Chapter 8 — Classes](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html)
3. [JLS Chapter 9 — Interfaces](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html)
4. [JLS Chapter 15 — Expressions and Method Invocation](https://docs.oracle.com/javase/specs/jls/se21/html/jls-15.html)
5. [Java SE 21 API — Object](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html)
6. [Java SE 21 API — List](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/List.html)

## Chapter checklist

- [ ] Explain encapsulation, abstraction, inheritance, and polymorphism with concrete examples.
- [ ] Distinguish overload resolution, overriding, hiding, and dynamic dispatch.
- [ ] Apply substitutability and choose composition versus inheritance deliberately.
- [ ] Explain initialization hazards, constructor dispatch, and interface default conflicts.
- [ ] Implement equals/hashCode contracts and immutable value objects correctly.
- [ ] Use SOLID principles as design heuristics, not as rigid rules.
