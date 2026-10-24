# Spring MVC and REST APIs

## Q&A

### Q1. What role does DispatcherServlet play?

DispatcherServlet is Spring MVC's front controller. It coordinates handler mapping, handler execution, argument resolution, return-value handling, message conversion, and exception resolution. It does not itself contain application business logic; it orchestrates the web request pipeline.

### Q2. What is the difference between @Controller and @RestController?

@Controller marks a Spring MVC controller, where a return value may be interpreted as a view name. @RestController combines controller semantics with response-body handling, so returned objects are generally serialized into the HTTP response. Individual methods can still customize status and headers using ResponseEntity.

### Q3. What is the difference between @PathVariable and @RequestParam?

@PathVariable binds a value from a URI path segment, commonly identifying a resource such as /orders/{id}. @RequestParam binds a query parameter or form parameter, commonly used for filtering, pagination, or optional modifiers. Choose based on resource semantics and the API contract, not personal preference alone.

### Q4. Why use DTOs rather than returning persistence entities directly?

DTOs define the API contract independently of database mapping, reduce accidental exposure of internal fields, and allow validation and serialization shape to evolve deliberately. Returning entities can also trigger lazy-loading queries or recursion through bidirectional relationships. Map at a clear application boundary and avoid leaking persistence details.

### Q5. How does request-body validation work?

Use Jakarta Bean Validation constraints on request DTOs and trigger validation with @Valid or the supported validation mechanism at the controller boundary. Handle validation exceptions consistently and return actionable field errors without echoing sensitive input. Validation is not a replacement for business-rule validation or database constraints.

### Q6. How should pagination be designed?

Accept bounded page/size or cursor parameters, validate limits, and return predictable metadata. Large offset pagination can become expensive for deep pages; cursor/keyset approaches may be preferable for large, frequently changing datasets. Ensure sorting is deterministic, ideally with a unique tie-breaker.

### Q7. What is content negotiation?

Content negotiation selects a representation based on request headers and supported media types. Message converters handle serialization and deserialization for supported types. Explicitly define supported content types and test malformed payloads, unsupported media types, and incompatible Accept headers.

### Q8. What makes an API operation idempotent?

An operation is idempotent if repeating the same request has the same intended effect as performing it once. HTTP method semantics help communicate this, but implementation must honor them. For retryable create operations, an idempotency key and durable deduplication record may be needed to avoid duplicate side effects.

### Q9. How should file uploads be protected?

Set request-size limits at the application and edge layers, validate content and authorization, avoid trusting client-supplied filenames or MIME types, and stream large files when appropriate. Store files outside executable application paths and apply malware scanning or content inspection when required by the threat model.

## Official references

- [Spring MVC reference](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [Validation](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html)
