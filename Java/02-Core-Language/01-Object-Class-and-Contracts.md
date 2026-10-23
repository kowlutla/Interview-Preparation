# Object Class and Object Contracts

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Every Java class ultimately participates in the Object contract unless it is a primitive type. Methods such as equals, hashCode, toString, getClass, clone, wait, notify, and notifyAll have different contracts and design implications. This chapter concentrates on correct overrides, identity, diagnostics, and synchronization-related methods.

## 1. Object's core methods

### Q1. What methods are declared by java.lang.Object? *(Beginner)*

Important methods include equals(Object), hashCode(), toString(), getClass(), clone(), finalize() in older APIs, and wait/notify/notifyAll for monitor coordination. Object also declares protected clone and finalize methods, with finalize deprecated for removal in modern Java.

Most application classes should deliberately implement equals/hashCode and often toString. They should not casually override getClass or monitor methods, and new code should not depend on finalization for resource management.

### Q2. What is the contract of equals? *(Intermediate)*

equals must be reflexive, symmetric, transitive, consistent while relevant information is unchanged, and false when compared with null. It should compare the logical value defined by the class rather than incidental implementation details.

A broken equals method can make collections behave unpredictably. When implementing it, consider inheritance, mutability, null handling, and whether identity or value equality is appropriate. Records provide generated component-based equality, but component types still define their own equality behavior.

### Q3. What is the contract of hashCode? *(Intermediate)*

If two objects are equal according to equals, they must have the same hash code. Unequal objects may share a hash code; collisions are allowed. Repeated calls should be consistent while information used by equals remains unchanged.

HashMap and HashSet use hash codes to find candidate buckets and equality to distinguish entries. A bad hashCode implementation can cause poor performance or violate lookup behavior when it disagrees with equals. It is not required to be unique or stable across process runs.

### Q4. Why must equals and hashCode be overridden together? *(Intermediate)*

If equals says two objects are equal but hashCode returns different values, a hash-based collection may place them in different buckets and fail to find an equal key. Implementing both from the same equality-relevant state preserves the required contract.

Use Objects.equals and Objects.hash where appropriate, while remembering that convenience helpers may allocate and that arrays require deliberate content semantics. Test equal-but-distinct instances, unequal values, null comparisons, and collection lookup.

### Q5. Why is mutable state dangerous in equals and hashCode? *(Advanced)*

If fields used by equality change after an object is inserted into a HashMap or HashSet, its hash bucket may no longer correspond to its current hash code. The entry can become difficult to find or remove using normal operations.

Prefer immutable keys. If mutable keys are unavoidable, do not mutate equality-relevant state while the key is stored in a hash-based collection; remove and reinsert around the change. That discipline is fragile, so an immutable key is usually the better design.

### Q6. What should toString provide? *(Beginner)*

toString should provide a useful textual representation for debugging and diagnostics. It is not automatically a stable serialization format, a public API contract, or a safe place to expose every field.

Avoid including passwords, access tokens, personal data, payment information, or huge object graphs. A useful diagnostic representation identifies the object and relevant non-sensitive state. Records generate a component-oriented toString, so sensitive record components need special consideration.

### Q7. When should getClass be used instead of instanceof? *(Advanced)*

getClass() == Other.class checks exact runtime class identity. instanceof checks whether an object is an instance of a type or its subtype. Exact-class equality can be useful in an equality strategy that deliberately rejects subclasses, while instanceof can support subtype-aware comparisons only if the equality contract remains symmetric and transitive.

There is no universal choice for equals. Inheritance makes value equality difficult; final classes or records often simplify the contract. Choose a policy that all participating subclasses can honor.

## 2. Cloning and object creation

### Q8. How does Object.clone work? *(Advanced)*

Object.clone performs a field-by-field copy of the object when cloning is permitted, which is shallow: referenced objects are not recursively copied. The method is protected and Cloneable is a marker interface indicating that cloning is allowed by the convention used by Object.clone.

Cloneable does not declare a clone method, and implementing it does not automatically create a public clone API. Many modern designs prefer copy constructors, static factories, or explicit copy methods because they make copy depth and invariants clearer.

### Q9. What is the difference between shallow and deep copying? *(Intermediate)*

A shallow copy duplicates the top-level object's fields but preserves references to the same nested objects. A deep copy also duplicates selected reachable mutable objects so the copy has independent state, according to a defined policy.

Deep copy is not simply “copy everything recursively”: cycles, shared references, resource handles, identity, and immutable objects complicate the operation. Define what independence means for the domain and test it. Serialization-based copying can be expensive and unsafe if the serialization format is not controlled.

### Q10. Why is Cloneable often discouraged in new designs? *(Advanced)*

The Cloneable contract is unusual: it is a marker interface, while the cloning operation is inherited from Object and is protected. The default clone is shallow, constructors are not invoked in the usual way, and subclasses must coordinate correctly to maintain invariants.

Copy constructors or named factories communicate intent and allow validation. If a class supports copying, specify whether it is shallow or deep, how mutable collaborators are treated, and whether subclass state is preserved.

### Q11. What replaced finalization for resource management? *(Advanced)*

Use try-with-resources and AutoCloseable for deterministic cleanup. Cleaner can provide a fallback cleanup mechanism for certain native-resource designs, but it is not a deterministic replacement for close. Finalization is deprecated for removal and should not be used in new code.

Resource cleanup should be explicit and testable. Relying on garbage collection or finalizer timing can retain resources too long and make application behavior unpredictable.

## 3. Identity and class metadata

### Q12. What does getClass return? *(Intermediate)*

getClass returns the runtime Class object representing the actual object's class. It is final, so subclasses cannot override it. The result can be used for reflection, exact-type checks, and metadata access.

Class identity includes the defining class loader. Two classes with the same fully qualified name loaded by different class loaders are distinct runtime types. This matters in application servers, plugin systems, hot deployment, and class-cast or linkage failures.

### Q13. Does Object.hashCode return a memory address? *(Intermediate)*

No such guarantee exists. Object.hashCode provides a hash value consistent with the API contract, but its relationship to object identity or memory layout is implementation-specific. It can be computed, stored, or generated by JVM-specific mechanisms.

Never use identity hash codes as unique IDs or memory addresses. Collisions are possible. Use a proper unique identifier or identity-aware data structure for the actual requirement.

### Q14. What are wait, notify, and notifyAll? *(Advanced)*

These methods coordinate threads using an object's intrinsic monitor. The calling thread must own the monitor when invoking wait, notify, or notifyAll, otherwise IllegalMonitorStateException is thrown. wait releases that monitor while waiting and reacquires it before returning.

These primitives are low-level and easy to misuse. Conditions should be checked in a loop because wakeups can be spurious and another thread may change the condition before the waiter reacquires the lock. Modern code often uses higher-level concurrency utilities such as BlockingQueue, Condition, CountDownLatch, or Semaphore.

### Q15. Why should wait always be used in a loop? *(Advanced)*

A waiting thread can wake spuriously, be notified when another waiter consumes the condition first, or observe that the condition has changed before it reacquires the monitor. Therefore the condition, not the fact of wakeup, determines whether the thread may proceed.

    synchronized (lock) {
        while (!ready) {
            lock.wait();
        }
        consume();
    }

The lock must protect both the condition check and the state transition. In production code, a higher-level concurrency primitive often expresses the protocol more safely.

## 4. Equality patterns and common traps

### Q16. Why can instanceof-based equals violate symmetry? *(Advanced)*

Suppose a base class considers only its own fields and accepts subclasses with instanceof. A subclass may add a field and include it in its own equality. The base object can then consider the subclass equal while the subclass considers the base object unequal, violating symmetry.

Possible strategies include exact-class equality, final value classes, carefully designed canEqual patterns, or composition. There is no universal pattern for every inheritance model; avoid mixing incompatible equality policies in a hierarchy.

### Q17. How should arrays be handled in equals and hashCode? *(Intermediate)*

Arrays inherit Object.equals and hashCode, so ordinary array equality is identity-based rather than element-based. Use Arrays.equals and Arrays.hashCode for one-dimensional arrays, and Arrays.deepEquals/deepHashCode for nested object arrays when deep semantics are intended.

Be careful with primitive arrays versus object arrays and with nested mutable elements. The equality contract should reflect the value semantics of the containing class, not merely whichever helper is easiest to call.

### Q18. How do records affect equals, hashCode, and toString? *(Intermediate)*

Records generate equals, hashCode, and toString based on their components unless explicitly overridden where permitted. This is convenient for transparent value carriers, but generated behavior delegates to each component's own equality and string representation.

A record with an array component will not automatically compare array contents deeply. A record containing a secret may also reveal it through generated toString. Choose components and overrides with the domain contract and information sensitivity in mind.

### Q19. How should equals handle null and type checks? *(Intermediate)*

equals must return false when passed null. A common implementation first checks reference identity, then checks null or exact type, then compares fields. Modern Java can use pattern matching for instanceof when the target source level supports it.

    @Override public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof Key key)) return false;
        return id == key.id;
    }

This form is appropriate when subtype-aware equality is intended and the class hierarchy preserves the contract. For final value classes, exact class checks can be simpler.

### Q20. What is the difference between identity equality and value equality in domain modeling? *(Senior)*

Value objects are usually equal when their meaningful attributes are equal, such as Money(amount, currency). Entities may retain identity even when mutable attributes change, such as an order with a stable order ID and changing status.

Choose equality based on domain semantics and persistence lifecycle. If a generated database ID is null before persistence and assigned later, using it naively in hashCode can change the object's hash after insertion into a set. Design equality around a stable identity policy.

## 5. Senior-level production reasoning

### Q21. Why can toString create a security or performance problem? *(Senior)*

toString may expose secrets or personal data, trigger lazy-loading behavior, traverse huge graphs, recurse through cyclic relationships, or allocate a large amount of text. Logging an entity can therefore cause privacy leaks, database queries, or latency spikes.

Prefer explicit structured logging of safe identifiers and selected fields. Test diagnostic output for sensitive values and avoid relying on generated string representations for large domain objects.

### Q22. How do you diagnose a HashMap lookup that unexpectedly returns null? *(Senior)*

Check whether the key was mutated after insertion, whether equals and hashCode are consistent, whether the lookup key is logically equal to the inserted key, and whether the same class was loaded by an unexpected class loader. Also verify null-value semantics: a map returning null does not distinguish absence from a key mapped to null without containsKey.

Use a minimal reproduction and inspect key state at insertion and lookup. Fix the equality invariant instead of replacing the map arbitrarily.

### Q23. How do class loaders affect Class identity and casts? *(Senior)*

Runtime type identity depends on both the binary name and the defining class loader. Two classes named com.example.Plugin loaded by different class loaders are different types, so a cast can fail even though the names look identical.

This commonly appears in plugin systems, application servers, test isolation, and duplicated dependency deployments. Diagnose the defining loader of each Class object and the dependency graph; comparing only class names is insufficient.

### Q24. Should finalize be used to release sockets, files, or database connections? *(Senior)*

No. Finalization timing is nondeterministic, finalizers can delay reclamation, and finalization has been deprecated for removal. Use explicit close operations and try-with-resources. For complex native-resource wrappers, Cleaner may be a fallback safety net while explicit close remains the primary path.

Test resource release on success, exception, and cancellation paths. Do not assume the garbage collector will promptly close a file descriptor or return a database connection to a pool.

### Q25. What is a reliable checklist for implementing a value object's contract? *(Senior)*

Define which fields determine logical equality; decide how inheritance is handled; keep equality-relevant state stable; implement equals and hashCode consistently; make toString useful but safe; and test behavior in HashMap and HashSet. Consider arrays, floating-point values, nulls, proxies, and lifecycle transitions where relevant.

For records, verify that generated component semantics match the domain. For entities, distinguish persistent identity from mutable state. A correct contract is a domain decision backed by Java's documented rules, not simply a boilerplate generator output.

## Official references

1. [Java SE 21 API — Object](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html)
2. [Java SE 21 API — Cloneable](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Cloneable.html)
3. [Java SE 21 API — Cleaner](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ref/Cleaner.html)
4. [JEP 421 — Deprecate Finalization for Removal](https://openjdk.org/jeps/421)
5. [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)

## Chapter checklist

- [ ] Implement equals/hashCode contracts consistently.
- [ ] Explain shallow/deep copy and the limitations of Cloneable.
- [ ] Keep toString useful and free of sensitive data.
- [ ] Understand monitor methods and prefer higher-level concurrency primitives.
- [ ] Diagnose class-loader identity and mutable-key failures.
- [ ] Replace finalization with deterministic resource management.
