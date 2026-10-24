# Data Types, Conversions, Operators and Control Flow

**Target level:** Beginner → Senior  
**Scope:** Java 8–21. Examples use Java language rules unless a version is explicitly named.

This chapter explains the mechanics behind Java expressions: primitive and reference values, initialization, numeric promotion, conversions, overflow, evaluation order, branching, loops, and abrupt completion. For output questions, reason from the specification rather than intuition.

## 1. Types and values

### Q1. What are Java's eight primitive types? *(Beginner)*

Java has boolean, byte, short, int, long, char, float, and double. The integer types are signed 8-, 16-, 32-, and 64-bit types except char, which is an unsigned 16-bit UTF-16 code unit. Float and double use IEEE 754 binary floating-point formats. Boolean represents true or false; Java does not specify it as a numeric width.

A char is not necessarily a complete Unicode character: supplementary code points use a surrogate pair. For Unicode-aware text processing, consider code-point APIs rather than assuming one char equals one user-perceived character.

### Q2. What is the difference between a primitive variable and a reference variable? *(Beginner)*

A primitive variable holds a primitive value. A reference variable holds null or a reference to an object or array. Java does not expose pointer arithmetic. Copying a primitive copies its value; copying a reference copies the reference value, so two variables can designate the same mutable object.

    int a = 10;
    int b = a;
    b++;
    System.out.println(a + ", " + b); // 10, 11

    StringBuilder first = new StringBuilder("hi");
    StringBuilder second = first;
    second.append("!");
    System.out.println(first); // hi!

Java is always pass-by-value, including when the value passed is a reference.

### Q3. Are objects always stored on the heap and local variables always stored on the stack? *(Intermediate)*

That is a useful mental model, not a complete physical-layout guarantee. The JVM specification defines abstract runtime areas but does not mandate every implementation's physical layout. HotSpot may optimize allocations away through escape analysis and scalar replacement; values can also live in registers.

For correctness, reason about language semantics and identity. For performance, identify the JVM build and measure representative workloads instead of assuming a particular memory layout.

### Q4. What are the default values of fields, array elements, and local variables? *(Beginner)*

Fields and array components are default-initialized: numeric primitives to zero, boolean to false, char to the null character, and references to null. Local variables do not receive a usable default value under Java's definite-assignment rules; the compiler rejects reads before assignment.

    class Defaults {
        int count;
        boolean enabled;
        String name;
        void run() {
            // int local;
            // System.out.println(local); // compile-time error
            System.out.println(count + " " + enabled + " " + name);
        }
    }

The fields print as 0 false null. Default initialization of storage is different from definite assignment of a local variable.

### Q5. What is the difference between declaration, initialization, and assignment? *(Intermediate)*

A declaration introduces a variable and its type. Initialization supplies its initial value. Assignment stores a new value into a variable that has already been declared.

    int count; // declaration; not yet definitely assigned
    count = 1; // first assignment
    count = count + 1;

For fields, storage first receives default values, then explicit initializers execute according to class/instance initialization rules. Instance initializers and field initializers run in textual order as part of construction, with superclass construction and constructor delegation affecting the full sequence.

### Q6. What is the difference between null and a default value? *(Intermediate)*

Null means that a reference variable does not designate an object. It is the default value for reference fields and array components, but cannot be assigned to a primitive variable. Dereferencing null normally throws NullPointerException.

Autoboxing can trigger unboxing and throw too:

    Integer value = null;
    // int n = value; // NullPointerException during unboxing

Validate nullability at boundaries where a value is required. Wrapper types are not interchangeable with primitives: they can be null and have object identity/equality considerations.

## 2. Literals, conversions and numeric promotion

### Q7. Why is byte b = 127 valid but byte b = 128 invalid? *(Intermediate)*

The literal 127 is an int constant expression whose value fits in byte, so the assignment conversion permits it. The value 128 is outside byte's range, so the second assignment is a compile-time error. The same exception does not generally allow an arbitrary int variable to be assigned to byte without a cast.

### Q8. Why does byte + byte produce an int? *(Intermediate)*

Binary numeric promotion promotes byte, short, and char operands to int for most arithmetic operations.

    byte a = 10;
    byte b = 20;
    // byte c = a + b; // compile-time error: expression is int
    byte c = (byte) (a + b);
    int total = a + b;

The cast can truncate if the result is outside byte's range. Use int for ordinary integer arithmetic unless narrower storage is specifically useful.

### Q9. What are widening and narrowing primitive conversions? *(Intermediate)*

Widening moves to a type with a broader representable domain, such as int to long or float to double. Widening does not always preserve precision: a large long cannot necessarily be represented exactly as float.

Narrowing moves to a type with a smaller range or precision and can discard information. Integral narrowing can discard high-order bits; floating-point narrowing can round or overflow to infinity. Validate values when loss is unacceptable.

### Q10. What happens when int or long arithmetic overflows? *(Intermediate)*

Java integer arithmetic does not throw an exception on overflow. The result retains the low-order bits and is interpreted as a signed two's-complement value.

    System.out.println(Integer.MAX_VALUE + 1); // -2147483648

Overflow can corrupt counters, sizes, monetary calculations, and bounds checks. Use Math.addExact, Math.multiplyExact, or Math.toIntExact when overflow should be detected; use BigInteger when arbitrary precision is needed.

### Q11. Why can 0.1 + 0.2 == 0.3 be false? *(Intermediate)*

Most decimal fractions do not have exact finite binary floating-point representations. Each literal is rounded to a nearby representable value, and arithmetic rounds again, so the computed sum need not equal the stored representation of 0.3.

For approximate calculations, choose a tolerance appropriate to the scale and error model. For exact decimal quantities such as money, use BigDecimal with deliberate scale and rounding rules, typically constructing from strings rather than from double values.

### Q12. Why should long literals usually use an uppercase L suffix? *(Beginner)*

An unsuffixed decimal integer literal is normally int when representable. Use the L suffix for long values that exceed the int range. Uppercase L is easier to distinguish from the digit 1.

    long population = 8_000_000_000L;
    int hex = 0xFF;
    int binary = 0b1010;

Underscores can improve readability but must be placed in valid positions between digits. Literal type affects overload resolution and arithmetic before assignment.

### Q13. How does char interact with arithmetic? *(Intermediate)*

Char is an unsigned 16-bit UTF-16 code unit. Arithmetic commonly promotes it to int, so 'A' + 1 evaluates numerically to 66, not to a char automatically.

    char letter = 'A';
    System.out.println(letter + 1); // 66
    System.out.println((char) (letter + 1)); // B

A char can contain a surrogate half, not a complete Unicode code point. Use String.codePoints() for code-point-aware processing.

### Q14. What is numeric promotion in a mixed-type expression? *(Advanced)*

For most binary numeric operations, if either operand is double, the operation is double; otherwise float wins, then long, otherwise operands are promoted to int. Shift operators have separate rules, and unary operators can also affect operand types.

    int x = 5;
    long y = 2;
    long result = x / y; // value 2; Java 8-compatible declaration

Both operands are integral, so division truncates toward zero before any later conversion. A cast after division cannot recover a fraction already discarded.

### Q15. What is the difference between i / 2 and i / 2.0? *(Intermediate)*

If i is an integer, i / 2 uses integral division and truncates toward zero. i / 2.0 uses floating-point division.

    System.out.println(5 / 2);   // 2
    System.out.println(5 / 2.0); // 2.5
    System.out.println(-5 / 2);  // -2

Truncation toward zero differs from floor for negative values: floor(-2.5) is -3. This matters in indexing, pagination, and distributing quantities.

### Q16. What does == mean for primitives and references? *(Intermediate)*

For primitives, == compares values, applying numeric promotion where required. For references, it checks whether both operands refer to the same object or are both null; it does not generally compare object contents.

    String a = new String("java");
    String b = new String("java");
    System.out.println(a == b);      // false
    System.out.println(a.equals(b)); // true

Use equals for value equality when the class defines the intended contract. Arrays require Arrays.equals or Arrays.deepEquals for content comparison.

## 3. Operators and evaluation order

### Q17. What is the difference between && and & for boolean operands? *(Intermediate)*

&& is conditional AND: when the left operand is false, the right operand is not evaluated. & with boolean operands evaluates both operands. With integral operands, & performs bitwise AND.

    int count = 0;
    boolean result = false && (++count > 0);
    System.out.println(count); // 0

Short-circuiting is useful for guards such as checking a reference before invoking a method on it. Do not depend on side effects in a right-hand expression unless the evaluation rule is intentional.

### Q18. What is the difference between || and |? *(Intermediate)*

|| is conditional OR and skips the right operand when the left side is true. | evaluates both boolean operands; with integral operands it performs bitwise OR. Use the non-short-circuit form only when evaluating both sides is intended.

### Q19. Does Java guarantee left-to-right operand evaluation? *(Advanced)*

Java specifies left-to-right evaluation of operands and arguments in the relevant expression contexts. Precedence and associativity determine grouping; they are not substitutes for evaluation order.

    static int f(String label, int value) {
        System.out.println(label);
        return value;
    }
    int sum = f("left", 1) + f("right", 2);
    // prints left, then right

Avoid expressions with many side effects. Separate statements are easier to review and debug.

### Q20. What is the difference between precedence and associativity? *(Intermediate)*

Precedence determines grouping between different operators; associativity determines grouping among operators at the same precedence. For example, a + b * c groups as a + (b * c), while 10 - 3 - 2 groups as (10 - 3) - 2 and produces 5. Parentheses in complex expressions communicate intent even when the precedence is known.

### Q21. What does x += y do differently from x = x + y? *(Advanced)*

Compound assignment evaluates the left-hand expression only once and includes an implicit conversion to the type of the left-hand variable.

    byte x = 1;
    x += 1; // valid
    // x = x + 1; // compile-time error without a cast

The single-evaluation rule matters when the left side includes array access or another expression with side effects. Compound assignment can still overflow or narrow; it is not checked arithmetic.

### Q22. What is the difference between pre-increment and post-increment? *(Intermediate)*

Both increment a variable by one. ++x increments first and yields the new value; x++ yields the old value and then increments.

    int x = 3;
    int a = x++; // a = 3, x = 4
    int b = ++x; // x = 5, b = 5

The difference is invisible when used as a standalone statement. Avoid combining multiple mutations in one expression.

### Q23. Why can the conditional operator's type be surprising? *(Advanced)*

The expression condition ? expr1 : expr2 selects one branch at runtime, but its type is determined at compile time from both alternatives and Java's typing rules. Numeric promotion, boxing, unboxing, and reference typing can affect the result.

    var value = true ? 1 : 2L; // long

Do not infer the type solely from the branch that happens to execute.

### Q24. How do Java shift operators work? *(Advanced)*

<< shifts left, >> shifts right with sign extension, and >>> shifts right with zero fill. For int, only the low five bits of the shift distance are used; for long, only the low six bits are used. Byte and short operands are promoted to int before shifting.

    System.out.println(1 << 32); // 1: distance is masked to zero
    System.out.println(-8 >> 1); // -4
    System.out.println(-8 >>> 1); // 2147483644

Bit shifts do not provide overflow checking. Use explicit types and parentheses in bit-manipulation code.

## 4. Branching and loops

### Q25. Which selector types are supported by traditional switch? *(Intermediate)*

Traditional switch supports the specified integral selector types (byte, short, char, int and their wrapper types subject to unboxing), enums, and String. Later releases expanded switch with switch expressions and pattern matching. Feature availability and status must be tied to the exact source release; do not assume a modern switch feature is valid under Java 8 source compatibility.

### Q26. What is fall-through in a traditional switch? *(Intermediate)*

In colon-form cases, execution can continue into later case bodies unless control exits, for example through break, return, or throw. Grouped labels can intentionally share a body.

    switch (day) {
        case 1:
        case 2:
            System.out.println("group");
            break;
        default:
            System.out.println("other");
    }

Arrow-form switch rules avoid accidental fall-through and are often clearer when the target Java version supports them.

### Q27. What is the difference between break, continue, and return? *(Intermediate)*

Break exits the nearest enclosing loop or switch; a labeled break can exit a named enclosing statement. Continue skips the rest of the current loop iteration and proceeds to the next iteration; a labeled continue targets a named enclosing loop. Return exits the current method, subject to finally execution. Exceptions cause abrupt completion unless handled.

### Q28. How does the enhanced for loop work? *(Intermediate)*

Enhanced for iterates over arrays or Iterable values. For an Iterable, the compiler uses iterator operations conceptually; for arrays, it traverses array elements. The loop variable receives each element value.

    int[] values = {1, 2, 3};
    for (int value : values) {
        value *= 2; // does not modify the array element
    }

Reassigning a primitive loop variable does not update the array. For mutable object elements, mutating the referenced object can affect that object, but reassigning the loop variable does not replace the collection's element.

### Q29. What is the scope of a variable declared in a for initializer? *(Intermediate)*

A variable declared in the initializer of a basic for statement is scoped to that loop statement, including its condition, update, and body. It is unavailable after the loop.

    for (int i = 0; i < 3; i++) {
        System.out.println(i);
    }
    // i is out of scope here

Scope and definite assignment are compile-time concepts. Pattern variables have additional flow-scoping rules in newer language versions.

### Q30. What happens when continue is used in a for loop? *(Advanced)*

In a basic for loop, continue transfers control to the update expression and then rechecks the condition. In while and do-while loops, control proceeds to the condition check. This difference can create an infinite loop if continue skips the statement that advances the loop variable.

    for (int i = 0; i < 5; i++) {
        if (i == 2) continue;
        System.out.print(i);
    }
    // 0134

### Q31. What is the difference between while and do-while? *(Beginner)*

While checks its condition before executing the body, so the body may run zero times. Do-while checks after the body, so the body runs at least once. A do-while statement requires a semicolon after its condition. Choose the construct that directly expresses whether a first execution is required.

### Q32. What is definite assignment? *(Advanced)*

Definite assignment is compile-time analysis that ensures a local variable is assigned on every possible control-flow path before it is read.

    int result;
    if (args.length > 0) {
        result = 1;
    } else {
        result = 2;
    }
    System.out.println(result);

If a path can complete normally without assigning result, the compiler rejects the read. This analysis does not prove that the assigned value is logically correct.

### Q33. How do finally, return, and exceptions interact? *(Advanced)*

A finally block normally executes when control leaves try/catch, including through return or an exception. If finally itself completes abruptly—for example by throwing or returning—it can replace the earlier completion reason and hide an exception or override a return value.

Avoid return or throwing a new exception from finally unless overriding earlier control flow is deliberate. Prefer try-with-resources for resource cleanup; it preserves close failures as suppressed exceptions when another exception is already in flight.

## 5. Output prediction and production pitfalls

### Q34. What is the output, and why does compound assignment compile? *(Advanced)*

    byte b = 127;
    b++;
    b += 1;
    System.out.println(b);

Output: -127. After b++, the value wraps from 127 to -128. Then b += 1 adds one and narrows back to byte, producing -127. This demonstrates integer overflow and implicit narrowing in increment/compound assignment.

### Q35. What is the output of this short-circuit example? *(Intermediate)*

    int x = 0;
    boolean a = false && (++x > 0);
    boolean b = true || (++x > 0);
    System.out.println(x + " " + a + " " + b);

Output: 0 false true. Neither right-hand operand executes: false && ... is already false, and true || ... is already true.

### Q36. What is the output of integer division followed by a cast? *(Intermediate)*

    System.out.println((double) (5 / 2));
    System.out.println((double) 5 / 2);

Output is 2.0 followed by 2.5. In the first expression, integer division occurs before the cast. In the second, casting one operand causes floating-point division. Cast placement changes the operation's type.

### Q37. Why does if (x = 5) fail to compile? *(Intermediate)*

Assignment is an expression, but x = 5 has type int, not boolean. An if condition must be boolean, so the compiler rejects it. Java does not treat integers as truthy or falsy values.

### Q38. Why is NaN == NaN false? *(Advanced)*

Under Java floating-point comparison rules, NaN is unequal to every value, including itself. Use Double.isNaN or Float.isNaN to test for NaN. Ordering comparisons involving NaN are also false, so validate numeric inputs before relying on ordinary comparisons.

### Q39. Why does Math.abs(Integer.MIN_VALUE) remain negative? *(Advanced)*

Integer.MIN_VALUE is -2,147,483,648, while Integer.MAX_VALUE is 2,147,483,647. The positive magnitude of the minimum value cannot fit in int, so Math.abs(Integer.MIN_VALUE) returns the same negative value. Widen before taking the absolute value or use a range-checking strategy when a nonnegative result is required.

### Q40. How should you solve a tricky Java output question? *(Senior)*

1. Determine the source level and whether the code compiles.
2. Record each variable's declared type and current value.
3. Apply precedence, associativity, and evaluation-order rules.
4. Track promotions, casts, boxing/unboxing, and overflow at the exact operation.
5. Mark operands skipped by short-circuit evaluation.
6. Track loop updates and abrupt completion, including finally.
7. Derive output only after the state trace is complete.

In production code reviews, simplify expressions with side effects rather than preserving clever code. Readability reduces the chance that later maintenance invalidates an implicit assumption.

## Official references

1. [Java Language Specification, Java SE 21 — Types, Values, and Variables](https://docs.oracle.com/javase/specs/jls/se21/html/jls-4.html)
2. [JLS Chapter 5 — Conversions and Contexts](https://docs.oracle.com/javase/specs/jls/se21/html/jls-5.html)
3. [JLS Chapter 14 — Blocks and Statements](https://docs.oracle.com/javase/specs/jls/se21/html/jls-14.html)
4. [JLS Chapter 15 — Expressions](https://docs.oracle.com/javase/specs/jls/se21/html/jls-15.html)
5. [JLS Chapter 16 — Definite Assignment](https://docs.oracle.com/javase/specs/jls/se21/html/jls-16.html)
6. [JLS Chapter 10 — Arrays](https://docs.oracle.com/javase/specs/jls/se21/html/jls-10.html)
7. [Java SE 21 API — Math](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Math.html)
8. [Java SE 21 API — BigDecimal](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigDecimal.html)

## Chapter checklist

- [ ] Distinguish primitives, references, null, default initialization, and definite assignment.
- [ ] Predict numeric promotion, narrowing, overflow, floating-point comparisons, and compound assignment.
- [ ] Explain short-circuit evaluation, precedence, associativity, and evaluation order.
- [ ] Reason about switch, loop control, scope, finally, and abrupt completion.
- [ ] Solve output questions by tracking types and state step by step.

## Worked examples: type rules and control flow

### Example 1. Integer arithmetic happens before a widening assignment

~~~java
long product = 50_000 * 50_000;
System.out.println(product); // -1794967296
long correct = 50_000L * 50_000;
System.out.println(correct); // 2500000000
~~~

In the first expression, both operands are int, so multiplication is performed as int and overflows before assignment to long. Widening the already-overflowed result cannot recover the lost value. Make at least one operand long before multiplication when the intended result requires long range.

### Example 2. Compound assignment includes an implicit conversion

~~~java
byte value = 10;
value += 1;       // legal: compound assignment includes a cast
// value = value + 1; // compile-time error without an explicit cast
System.out.println(value); // 11
~~~

The expression value + 1 is evaluated as int due to binary numeric promotion. Compound assignment converts the result back to the left-hand variable's type. Legality does not imply safety from information loss or overflow.

### Example 3. Short-circuit operators control right-operand evaluation

~~~java
int calls = 0;
boolean first = false && (++calls > 0);
boolean second = true || (++calls > 0);
System.out.println(calls); // 0
~~~

For &&, the right operand is evaluated only when the left operand is true. For ||, it is evaluated only when the left operand is false. The non-short-circuit boolean operators & and | evaluate both operands. Avoid hiding required business side effects inside boolean expressions.

### Example 4. Floating-point equality is not decimal arithmetic

~~~java
System.out.println(0.1 + 0.2 == 0.3); // false
~~~

Binary floating-point represents many decimal fractions approximately. For monetary quantities, use a deliberately designed decimal representation such as BigDecimal constructed from decimal strings, with explicit scale and rounding policy. For numerical algorithms, choose a tolerance based on the error model rather than applying an arbitrary epsilon everywhere.

### Example 5. Avoid replacing a pending outcome in finally

A finally block normally runs during abrupt completion of try/catch flow, subject to process termination and other exceptional circumstances. Returning or throwing from finally can suppress a pending exception or replace a return value. Use try-with-resources for resource cleanup and keep finally blocks from changing control flow unnecessarily.
