# Reference Solution — Validated Configuration and Custom Validation

## Type-safe configuration

For a Boot application with configuration-properties scanning enabled:

```java
@ConfigurationProperties(prefix = "inventory")
@Validated
public record InventoryClientProperties(
    @NotNull URI baseUrl,
    @NotNull Duration timeout,
    @Min(1) @Max(100) int maxItems
) {}
```

Example configuration:

```yaml
inventory:
  base-url: https://inventory.internal
  timeout: 800ms
  max-items: 25
```

Ensure the properties type is registered with configuration-properties scanning or explicit enablement, and include the validation implementation required by the selected Boot line. Validate URLs, durations, and limits at startup. Avoid logging the entire properties object if it can contain credentials.

## Custom constraint

A custom constraint consists of an annotation, a validator, and tests. Keep the validator deterministic and avoid expensive network calls.

```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ProductCodeValidator.class)
public @interface ValidProductCode {
    String message() default "Invalid product code";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
public class ProductCodeValidator
        implements ConstraintValidator<ValidProductCode, String> {
    private static final Pattern FORMAT = Pattern.compile("[A-Z0-9-]{3,32}");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && FORMAT.matcher(value).matches();
    }
}
```

The exact constraint target list should match the supported Jakarta Bean Validation version and intended use. Decide separately whether null is valid; here it is invalid, while @NotNull can express nullability as a separate composable rule.

## Validation plan

- [ ] Valid canonical value passes.
- [ ] Lowercase, spaces, and unsupported punctuation fail if excluded by the contract.
- [ ] Null behavior is intentional and tested.
- [ ] Validation runs at the HTTP boundary and non-HTTP entry points enforce domain invariants separately.
- [ ] Error responses do not echo sensitive values.
