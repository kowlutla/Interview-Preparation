# Generics and the Java Type System

**Target level:** Intermediate → Senior | **Scope:** Java 8–21

Generics provide compile-time type relationships for classes, interfaces, methods, and constructors. Java implements most generic types using type erasure, which creates important limits around runtime checks, arrays, and overloads.

## 1. Generic fundamentals

### Q1. What problem do generics solve? *(Beginner)*

Generics parameterize a type by another type. List<String> documents that the collection is intended to contain strings, lets the compiler reject incompatible writes, and reduces caller-side casts. Without generics, APIs often accepted Object and type errors appeared later as ClassCastException.

Generics improve reuse and API contracts but do not eliminate all runtime type errors. Raw types, unchecked casts, reflection, and unsafe deserialization can bypass compile-time guarantees.

### Q2. What is a generic class? *(Beginner)*

A generic class declares type parameters used in its fields, methods, or supertypes.

    final class Box<T> {
        private final T value;
        Box(T value) { this.value = value; }
        T value() { return value; }
    }
    Box<String> box = new Box<>("hello");

T is a type variable and Box<String> is a parameterized type. Ordinary Java code cannot ask an object for its actual generic argument at runtime because generic type arguments are mostly erased.

### Q3. What is a generic method? *(Intermediate)*

A generic method declares its own type parameters before its return type.

    static <T> T firstOrNull(List<T> values) {
        return values.isEmpty() ? null : values.get(0);
    }

The compiler can infer T from invocation arguments and the target context. Explicit type witnesses are possible but rarely necessary. A static method can be generic because its type parameter belongs to the method, not to a class instance.

### Q4. What is type inference in generics? *(Intermediate)*

Type inference lets the compiler determine type arguments from invocation arguments, target types, and constraints. The diamond operator allows constructor type arguments to be inferred, as in new ArrayList<>();.

Inference is compile-time behavior. If it fails or selects a broader type than intended, supply an explicit type argument or target type. Do not assume the runtime object remembers the source-level type argument.

### Q5. Why can't we write new T() or new T[10]? *(Intermediate)*

T's concrete type is generally unavailable at runtime after erasure, so the compiler cannot directly instantiate it or create an array with a reified component type. Common alternatives include passing a Supplier<T>, passing a Class<T> token for certain reflective operations, or using List<T> instead of a generic array.

A Class<T> token cannot represent every parameterized type, such as List<String>. Choose a construction strategy that makes the required runtime information explicit.

### Q6. What is a bounded type parameter? *(Intermediate)*

A bound restricts permitted type arguments and exposes operations available on that bound.

    static <T extends Number> double sum(T a, T b) {
        return a.doubleValue() + b.doubleValue();
    }

The keyword extends is used for class and interface bounds. A type parameter can have one class bound and multiple interface bounds; if present, the class bound must come first.

### Q7. What is a wildcard type? *(Intermediate)*

A wildcard uses ? when the exact type argument is unknown or intentionally abstracted. List<?> means a list of some unknown element type; List<? extends Number> accepts a list whose element type is Number or a subtype; List<? super Integer> accepts a list whose element type is Integer or a supertype.

Wildcards express use-site flexibility. They are especially useful for API parameters where exact generic invariance would unnecessarily restrict callers.

### Q8. Why is List<Integer> not a subtype of List<Number>? *(Intermediate)*

Java generic types are invariant. If List<Integer> could be assigned to List<Number>, code could insert a Double into a list intended to contain only Integer values.

    List<Integer> integers = new ArrayList<>();
    // List<Number> numbers = integers; // compile-time error

Use List<? extends Number> when a method needs to read numbers from lists with different concrete element types without writing arbitrary Number values into them.

### Q9. Explain PECS: Producer Extends, Consumer Super. *(Advanced)*

If a structure produces values that a method reads, prefer an extends wildcard. If it consumes values the method writes, prefer a super wildcard.

    static double total(List<? extends Number> values) {
        return values.stream().mapToDouble(Number::doubleValue).sum();
    }
    static void addDefaults(List<? super Integer> output) {
        output.add(0);
        output.add(1);
    }

A List<? extends Number> can be read as Number, but arbitrary Number values cannot safely be added because the actual element type might be Integer. A List<? super Integer> can accept Integer values, but reads are only safely typed as Object. PECS is a guide, not a substitute for understanding the operations.

### Q10. What can you add to a List<?>? *(Intermediate)*

You cannot add an arbitrary non-null value because the hidden element type is unknown. You can read values as Object and can attempt to add null, though a particular collection may reject null.

    List<?> values = new ArrayList<String>();
    Object first = values.isEmpty() ? null : values.get(0);
    // values.add("x"); // compile-time error

If the method must read and write elements of a known type, expose a type parameter or suitable lower-bounded wildcard rather than an unbounded wildcard.

### Q11. What is wildcard capture? *(Advanced)*

Wildcard capture lets the compiler treat an unknown wildcard type as a fresh type variable inside a helper method, preserving a stable relationship between reads and writes.

    static void swap(List<?> list, int i, int j) {
        swapCaptured(list, i, j);
    }
    private static <T> void swapCaptured(List<T> list, int i, int j) {
        T value = list.get(i);
        list.set(i, list.get(j));
        list.set(j, value);
    }

The helper can safely write back values of the captured type without discovering the actual type argument. Capture does not permit unsafe writes.

### Q12. What are intersection bounds? *(Advanced)*

A type variable may have multiple bounds, such as T extends Number & Comparable<T>. This requires T to satisfy both capabilities. The class bound, if any, must appear before interface bounds.

Intersection bounds let an algorithm rely on multiple contracts without casts. Keep bounds as narrow as the algorithm requires; unnecessary constraints make APIs harder to use.

## 2. Erasure and runtime limitations

### Q13. What is type erasure? *(Intermediate)*

Type erasure removes most generic type information from executable bytecode to preserve compatibility with older Java libraries. Type variables are replaced by their erasure, often their leftmost bound, and casts or bridge methods may be inserted.

List<String> and List<Integer> generally share the same runtime class. This is why ordinary instanceof checks cannot test List<String> directly and why untrusted collections need element-level validation.

### Q14. What is a raw type, and why avoid it? *(Intermediate)*

A raw type uses a generic declaration without type arguments, such as List rather than List<String>. Raw types exist for legacy compatibility but bypass generic checks and can introduce heap pollution.

    List raw = new ArrayList<String>();
    raw.add(42); // unchecked warning
    // String s = (String) raw.get(0); // ClassCastException

Use parameterized types in new code. Isolate raw types at legacy boundaries and validate values there.

### Q15. What is heap pollution? *(Advanced)*

Heap pollution occurs when a variable of a parameterized type refers to an object that violates the expected type argument. Raw types, unchecked casts, and unsafe generic varargs are common causes. The failure often appears later when a value is read and cast.

Treat compiler warnings as useful signals. If an unchecked cast is unavoidable, document the invariant that makes it safe and suppress the warning only at the narrowest point.

### Q16. Why can't methods be overloaded only by generic arguments? *(Advanced)*

After erasure, signatures that differ only in type arguments may have the same erased signature and cause a name clash.

    // void process(List<String> values) {}
    // void process(List<Integer> values) {} // erasure clash

Use distinct method names or a different parameter structure. Do not expect erased generic arguments to disambiguate a call at runtime.

### Q17. What are bridge methods? *(Advanced)*

The compiler may generate synthetic bridge methods to preserve polymorphism after type erasure or covariant return changes. A bridge can expose an erased signature and delegate to the source-level implementation.

Bridge methods are bytecode artifacts, not normally methods application code should call directly. They explain why reflection may show synthetic methods and why bytecode or stack traces sometimes include compiler-generated members.

### Q18. Why is instanceof List<String> illegal? *(Intermediate)*

The runtime generally cannot distinguish List<String> from List<Integer> because the type arguments are erased. instanceof List<?> is allowed because it checks only the reifiable runtime type List.

When parsing untrusted data, checking instanceof List<?> is only the first step. Validate each element before treating the list as List<String>. Generic signatures may preserve declaration metadata for reflection, but they do not reify each object's actual type argument.

### Q19. What is a reifiable type? *(Advanced)*

A reifiable type carries enough runtime information for certain checks. Examples include non-generic types, raw types, primitive types, and parameterized types whose arguments are all unbounded wildcards, such as List<?>. List<String> is not reifiable.

Reifiability explains restrictions on generic arrays, instanceof, and varargs. When a feature is prohibited, ask whether its safety would require runtime information that erasure removes.

### Q20. Why are generic arrays prohibited? *(Advanced)*

A new List<String>[10] would need a runtime component type that distinguishes List<String> from List<Integer>, but both erase to the same raw type. The array store-checking model cannot enforce that generic invariant.

Prefer List<List<String>> or another collection abstraction. Low-level libraries sometimes use arrays of erased types with carefully encapsulated unchecked casts, but the invariant must be justified and tested.

### Q21. Why are arrays covariant while generic types are invariant? *(Advanced)*

String[] can be assigned to Object[] because arrays are covariant. The runtime remembers the actual array component type and throws ArrayStoreException if an incompatible value is stored through the broader reference.

Generics are invariant because their type arguments are erased and compile-time checking provides the main safety guarantee. Covariant generic collections would permit unsafe writes without an equivalent runtime component check.

## 3. API design

### Q22. When should you use a type parameter versus a wildcard? *(Advanced)*

Use a named type parameter when multiple positions must share the same unknown type relationship, such as an argument and return value. Use a wildcard when the exact type is irrelevant and variance expresses the operations needed.

An identity method needs <T> T identity(T value). A method that only reads numbers can accept List<? extends Number>. Prefer the simplest signature that communicates the necessary relationship without unnecessarily restricting callers.

### Q23. What is recursive type bounding? *(Advanced)*

A recursive bound constrains T by an interface parameterized with T itself, as in T extends Comparable<T>. It is common in generic algorithms that compare values.

    static <T extends Comparable<? super T>> T max(T a, T b) {
        return a.compareTo(b) >= 0 ? a : b;
    }

The super form supports comparison contracts implemented by a supertype. Recursive bounds are useful but should be introduced only when they encode a real requirement.

### Q24. What are the restrictions on generic exception classes? *(Advanced)*

A generic class cannot directly or indirectly extend Throwable with a type parameter because exception handling relies on runtime exception types while generic arguments are erased. A type variable also cannot be used as a catch type.

Use concrete exception classes with stable runtime meaning and place generic data in fields or associated result types.

### Q25. What is the connection between varargs and heap pollution? *(Advanced)*

Varargs are represented as an array. When the element type is non-reifiable, the compiler may warn that runtime array checks cannot enforce generic type safety. Unsafe code can expose the array through a broader type and store an incompatible value.

@SafeVarargs is an assertion by the programmer that the method or constructor does not perform unsafe operations on the varargs array. It is allowed only on eligible declarations; it is not a runtime check. Do not apply it merely to silence a warning.

### Q26. When is Class<T> useful, and what can it not represent? *(Advanced)*

Class<T> carries a runtime class object for a concrete type and is useful for reflection, factories, and runtime checks. It cannot represent the full parameterization of List<String> because generic arguments are erased.

Some libraries use a super-type-token pattern to capture generic declaration metadata. Such metadata does not automatically validate arbitrary object's contents; external data still requires runtime checks.

### Q27. How should unchecked casts be handled in production? *(Senior)*

First ask whether a better generic signature or typed API eliminates the cast. If not, validate at the trust boundary, document the invariant, suppress warnings narrowly, and test invalid inputs as well as valid ones.

Unchecked casts often mean type information was lost earlier through raw collections or untyped deserialization. Repair the boundary rather than distributing casts throughout business logic.

### Q28. How do generics interact with dependency injection and service registries? *(Senior)*

A registry can use Class<T> as a key for concrete runtime types, but generic service abstractions with parameterized arguments can collide after erasure. Frameworks may capture richer type metadata or require explicit qualifiers and keys.

Do not assume a generic declaration uniquely identifies a runtime bean. Understand how the container resolves types, qualifiers, scopes, and proxies, and use explicit keys where multiple services share the same erased type.

### Q29. How do you diagnose a ClassCastException in generic collection code? *(Senior)*

Find the read that fails, then trace all insertion paths. Look for raw types, unchecked casts, unsafe deserialization, reflection, and APIs returning Object. The failing cast may be far downstream from the insertion that violated the invariant.

Enable compiler warnings, isolate legacy boundaries, and validate untrusted values at entry. Replacing one cast with another rarely fixes the underlying type-safety problem.

### Q30. What generic API design mistakes are common in production? *(Senior)*

Common mistakes include exposing raw collections, using unbounded wildcards where a named type relationship is needed, overconstraining callers with unnecessary bounds, suppressing unchecked warnings globally, and assuming generic arguments are available at runtime.

Design signatures around the operations the method performs. Use wildcards for variance, type parameters for relationships, explicit runtime tokens where necessary, and validation at external boundaries.

## Official references

1. [Java Language Specification, Java SE 21 — Types](https://docs.oracle.com/javase/specs/jls/se21/html/jls-4.html)
2. [JLS Chapter 5 — Conversions and Contexts](https://docs.oracle.com/javase/specs/jls/se21/html/jls-5.html)
3. [JLS Chapter 8 — Classes and generic declarations](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html)
4. [JLS Chapter 9 — Interfaces](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html)
5. [Java SE 21 API — Class](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Class.html)

## Chapter checklist

- [ ] Write generic classes and methods with inferred type arguments.
- [ ] Apply bounds, wildcards, PECS, and wildcard capture.
- [ ] Explain invariance, erasure, raw types, bridge methods, and heap pollution.
- [ ] Recognize restrictions on generic arrays, instanceof, and overload signatures.
- [ ] Design APIs that preserve useful type relationships and diagnose unchecked casts.
