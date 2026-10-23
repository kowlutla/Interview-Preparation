# Design Patterns and SOLID in Java

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Design patterns are reusable approaches to recurring design problems, not mandatory class diagrams. SOLID principles help reason about responsibilities, extension, substitutability, interfaces, and dependency direction. Apply them when they reduce change cost and improve clarity, not as rules that force abstractions into every small program.

## 1. SOLID principles

### Q1. What is the Single Responsibility Principle? *(Intermediate)*

A class should have one coherent reason to change, often tied to one actor or source of change. A class that validates orders, persists them, sends emails, and renders reports is coupled to concerns that may evolve independently.

SRP does not mean one method per class. Split responsibilities when it improves cohesion and isolates change, but avoid fragmenting a simple workflow into unnecessary types.

### Q2. What is the Open/Closed Principle? *(Intermediate)*

Software entities should be open to extension but closed to repeated modification for every new variation. Interfaces, composition, strategy objects, and well-designed extension points can allow new behavior without editing a central conditional each time.

Do not predict every possible future requirement. Introduce abstractions where change is likely and meaningful; otherwise, a simple implementation can be cheaper and clearer.

### Q3. What is the Liskov Substitution Principle? *(Intermediate)*

A subtype should be usable wherever its supertype is expected without breaking the supertype's behavioral promises. Subtypes should not strengthen preconditions or weaken postconditions in ways that surprise callers.

Inheritance should represent behavioral substitutability, not merely code reuse. If a subtype cannot honor the base contract, use composition or redesign the abstraction.

### Q4. What is the Interface Segregation Principle? *(Intermediate)*

Clients should not be forced to depend on methods they do not use. Large interfaces with unrelated responsibilities create coupling and make implementations provide meaningless methods.

Split interfaces around coherent client needs, but do not create dozens of tiny interfaces without a meaningful boundary. The goal is a stable, focused contract.

### Q5. What is the Dependency Inversion Principle? *(Intermediate)*

High-level policy should depend on abstractions rather than concrete low-level details, and both should depend on stable contracts. Constructor injection and interfaces are common Java techniques.

The abstraction should reflect a meaningful domain or architectural boundary. An interface for every class can add ceremony without improving change isolation.

## 2. Creational patterns

### Q6. What is the Singleton pattern, and what are its risks? *(Intermediate)*

Singleton ensures a single instance within a defined scope, often the class loader or application context. It can be implemented with enum constants or initialization-on-demand holders, but it can also introduce global mutable state, hidden dependencies, and test isolation problems.

A singleton is not automatically one instance across processes or class loaders. Prefer dependency injection when lifecycle and testability matter; use a singleton only when single-instance semantics are genuinely required.

### Q7. What is the Factory Method pattern? *(Intermediate)*

Factory Method delegates object creation to a method or subtype, allowing callers to depend on an abstraction rather than directly constructing a concrete implementation. It is useful when creation policy varies by configuration or environment.

Do not create a factory abstraction for a single trivial constructor unless it adds a real boundary, validation, caching, or selection behavior.

### Q8. What is the Abstract Factory pattern? *(Advanced)*

Abstract Factory creates families of related objects without exposing concrete classes. For example, a UI toolkit may create a matching set of buttons, menus, and dialogs for one platform.

It is useful when related products must remain compatible. It can be overkill when products are independent or only one family is ever needed.

### Q9. What is the Builder pattern? *(Intermediate)*

Builder separates complex object construction from the final representation and can make optional parameters readable. It is useful for objects with many configuration choices or validation across fields.

A builder should validate required fields and establish invariants at build time. Avoid builders for tiny immutable types where a constructor or record is clearer.

## 3. Structural and behavioral patterns

### Q10. What is the Adapter pattern? *(Intermediate)*

Adapter translates one interface into another expected by a client. It is useful for integrating a legacy API or third-party provider behind a stable internal contract.

Keep provider-specific behavior inside the adapter. This prevents external library details from leaking across the application and makes testing or provider replacement easier.

### Q11. What is the Decorator pattern? *(Intermediate)*

Decorator wraps an object implementing the same abstraction and adds behavior while preserving the interface. It can add metrics, logging, caching, or buffering without modifying the original implementation.

Decorator chains can become difficult to debug if too many layers are implicit. Keep ordering and failure behavior clear, especially when wrappers alter timeouts, retries, or resource lifecycle.

### Q12. What is the Proxy pattern? *(Intermediate)*

Proxy controls access to another object and may add lazy loading, remote access, security, caching, or transaction behavior. JDK dynamic proxies support interfaces; class-based proxying requires other mechanisms.

A proxy only intercepts calls that pass through it. Understand self-invocation, method visibility, final methods, and object identity when proxies are used by frameworks.

### Q13. What is the Strategy pattern? *(Intermediate)*

Strategy encapsulates interchangeable algorithms behind a common interface and lets a caller select one. It can replace a large conditional with a set of implementations when variations are independent and likely to grow.

For a small stable conditional, a switch may be simpler. The pattern is valuable when the algorithm varies independently from the caller and can be tested or configured separately.

### Q14. What is the Template Method pattern? *(Intermediate)*

Template Method defines a fixed algorithm structure in a base class and lets subclasses override selected steps. It can be useful for controlled extension frameworks.

It relies on inheritance and can be fragile if subclasses depend on internal sequencing. Prefer composition or functions when the variation points can be passed as collaborators more simply.

### Q15. What is the Observer pattern? *(Intermediate)*

Observer lets subscribers receive notifications when a subject changes. It appears in event listeners, UI systems, and domain event handling.

Define subscription lifecycle, ordering, failure isolation, and thread execution. A forgotten listener can leak memory, and synchronous observers can unexpectedly increase request latency.

### Q16. What is the Chain of Responsibility pattern? *(Intermediate)*

Chain of Responsibility passes a request through a sequence of handlers, each of which can process it or delegate to the next. It is common in filters, middleware, and validation pipelines.

Define whether all handlers run or processing stops after a match. Observability and error handling should make it clear which handler accepted, rejected, or failed the request.

### Q17. What is the Command pattern? *(Intermediate)*

Command encapsulates an operation as an object, allowing it to be queued, logged, retried, scheduled, or undone where the domain supports reversal. A Runnable is a simple command-like abstraction.

For durable distributed commands, persist the command and its idempotency/identity metadata. An in-memory command object is not a durable queue or transaction log.

## 4. Senior-level application

### Q18. How do you avoid overusing design patterns? *(Senior)*

Start with the simplest design that satisfies current requirements. Introduce a pattern when it makes a real variation point, ownership boundary, lifecycle, or testing seam clearer. Avoid patterns that add layers without reducing coupling or complexity.

Review whether the abstraction has multiple meaningful implementations, whether it hides a stable contract, and whether it makes the change you expect easier. Delete abstractions that no longer provide value.

### Q19. How do patterns interact with dependency injection? *(Senior)*

Dependency injection supplies collaborators from outside the class, supporting inversion of control and testability. It can implement or simplify patterns such as Strategy, Factory, Decorator, and Proxy.

Do not confuse a framework annotation with the pattern's underlying design intent. Understand lifecycle, scopes, proxying, and configuration so the dependency graph remains explicit and testable.

### Q20. How would you refactor a large switch on payment provider? *(Senior)*

First identify whether each branch is a stable provider variation with a common behavioral contract. If so, define a provider interface, create one adapter per provider, and select the implementation through explicit configuration or a registry. Keep common orchestration in the service.

Do not blindly convert every switch into a class hierarchy. Validate error mapping, idempotency, retry policy, capability differences, and provider-specific limits before choosing the abstraction.

## Official references

1. [Java SE 21 API documentation](https://docs.oracle.com/en/java/javase/21/docs/api/)
2. [JLS — Classes and Interfaces](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html)
3. [JLS — Interfaces](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html)

## Chapter checklist

- [ ] Explain SOLID as design heuristics with trade-offs.
- [ ] Choose patterns based on actual variation and lifecycle.
- [ ] Keep provider/framework details behind stable boundaries.
- [ ] Avoid global mutable state and over-engineered abstractions.
