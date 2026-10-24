# Java 17 Features

**Target level:** Advanced → Senior  
**Release:** Java 17 LTS  
**Feature status:** Records and sealed classes are final in Java 17; pattern matching for switch was not final in Java 17.

Java 17 is an important LTS release and a common enterprise baseline. Strong answers distinguish permanent features from preview features and describe the migration implications of stronger encapsulation and runtime changes.

## 1. Language features

### Q1. What are the key Java 17 features to know? *(Intermediate)*

Important topics include records (final in Java 16), sealed classes (final in Java 17), pattern matching for instanceof (final in Java 16), switch pattern matching as a preview feature in Java 17, strong encapsulation of JDK internals, and the new LTS release baseline.

Do not describe every feature discussed in Java 17 as a permanent Java 17 feature. Feature status must be tied to the release in which it was previewed or finalized.

### Q2. What are sealed classes in Java 17? *(Intermediate)*

Sealed classes and interfaces restrict which types may directly extend or implement them. A permitted subtype must declare whether it is final, sealed, or non-sealed, making the extension policy explicit.

They are useful for modeling controlled variants and can support exhaustiveness reasoning. Sealed does not imply immutability or automatically make every switch exhaustive.

### Q3. What should Java 17 developers know about pattern matching for instanceof? *(Intermediate)*

Pattern matching for instanceof is already a permanent feature in Java 17 because it was finalized in Java 16. It can be used without preview flags on a Java 17 toolchain.

    if (value instanceof String text) {
        System.out.println(text.length());
    }

When explaining version history, distinguish this finalized feature from pattern matching for switch, which was still a preview feature in Java 17.

The pattern variable's scope follows flow analysis. This feature is final in Java 17, unlike switch pattern matching in that release.

### Q4. What is the status of pattern matching for switch in Java 17? *(Advanced)*

Pattern matching for switch was a preview feature in Java 17, not a permanent language feature. Using it required enabling preview features with the appropriate compiler and runtime flags for that release.

It became permanent in Java 21 after further preview iterations. For production code, state the exact source release and feature status rather than treating preview syntax as universally supported.

### Q5. What are records and are they a Java 17 feature? *(Intermediate)*

Records became permanent in Java 16, so they are available as final language functionality in Java 17. They provide concise declarations for transparent carriers of fixed components and generate accessors, canonical construction, equals, hashCode, and toString according to record rules.

Records are shallowly immutable, not automatically deeply immutable. Mutable component objects can still change, and generated toString can expose component values.

### Q6. What does strong encapsulation of JDK internals mean? *(Advanced)*

JEP 403 strongly encapsulated most internal JDK elements, reducing reliance on unsupported internals. Code that used reflection or compiler flags to access private JDK implementation packages may fail or emit warnings.

Replace internal APIs with supported public APIs where possible. Diagnose module-access failures and avoid treating --add-opens across many packages as a permanent migration strategy.

## 2. Runtime and migration

### Q7. Why is Java 17 an important LTS baseline? *(Intermediate)*

Java 17 is a long-term-support release used by many organizations as a stable production baseline. It includes multiple years of language, runtime, security, and library evolution since Java 8.

LTS does not mean every vendor provides the same support dates or update policy. Verify the JDK distribution, support contract, patch cadence, and deployment constraints.

### Q8. What can break when migrating Java 8 applications to Java 17? *(Senior)*

Potential issues include removed or encapsulated internal APIs, reflection access failures, dependency incompatibilities, TLS/security defaults, garbage collector/runtime behavior changes, and reliance on undocumented implementation details. Build tools and frameworks may also need updates.

Run compatibility analysis, compile and test with the target JDK, inspect illegal reflective access, and exercise production-like integration paths. Do not assume that a successful compilation proves runtime compatibility.

### Q9. How do you compile and run a Java 17 preview feature deliberately? *(Intermediate)*

For Java 17 preview language features, compile with the Java 17 release target and preview enabled (for example, `javac --release 17 --enable-preview ...`) and run with `java --enable-preview ...` on a compatible Java 17 runtime. Preview syntax is tied to the specific feature release; a later JDK does not automatically accept Java 17 preview syntax unchanged.

Do not enable preview globally without a deliberate decision. Keep preview use isolated, document the exact toolchain and flags, and migrate or remove the feature before relying on a permanent API contract.

### Q10. How should teams evaluate Java 17 upgrades? *(Senior)*

Test functional compatibility, dependencies, startup, memory, GC, throughput, tail latency, container behavior, TLS/security, and operational tooling. Use representative load tests and compare against a baseline rather than assuming newer means faster.

Document the exact vendor/build and JVM options. Roll out gradually with observability and rollback criteria.

## Official references

1. [JDK 17 Documentation](https://docs.oracle.com/en/java/javase/17/)
2. [JEP 409 — Sealed Classes](https://openjdk.org/jeps/409)
3. [JEP 394 — Pattern Matching for instanceof](https://openjdk.org/jeps/394)
4. [JEP 403 — Strongly Encapsulate JDK Internals](https://openjdk.org/jeps/403)
5. [JEP 406 — Pattern Matching for switch (Preview in Java 17)](https://openjdk.org/jeps/406)

## Chapter checklist

- [ ] Distinguish final Java 17 features from preview features.
- [ ] Explain sealed classes and pattern matching for instanceof.
- [ ] Understand Java 8-to-17 compatibility risks.
- [ ] Evaluate migrations with functional and performance evidence.
