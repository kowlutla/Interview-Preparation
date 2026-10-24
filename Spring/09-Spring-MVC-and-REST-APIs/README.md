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

### Q10. What is the difference between a servlet filter and HandlerInterceptor?

A filter participates in the servlet request/response chain and can run before DispatcherServlet. A HandlerInterceptor participates in Spring MVC's handler lifecycle after handler mapping. Use filters for servlet-wide concerns such as request wrapping and security integration, and interceptors for handler-oriented concerns that need MVC metadata.

### Q11. How does Spring select an HTTP message converter?

Spring MVC considers the target method parameter/return type, request Content-Type, response Accept negotiation, and converters registered in the application. Unsupported media types can produce 415 for request content or 406 for an unacceptable response representation. Customize converters only when the API needs behavior beyond standard support.

### Q12. What is the difference between 400 and 422?

Both can represent client input problems depending on the API contract. 400 is broadly used for malformed or invalid requests; 422 can express syntactically valid content whose instructions cannot be processed semantically. Choose a consistent contract rather than assuming the framework will choose your domain's preferred status automatically.

### Q13. How should API sorting parameters be secured?

Map public sort keys to a fixed set of entity properties or query expressions, validate direction, cap page size, and pass the resulting `Sort`/`Pageable` through supported repository APIs. Never concatenate client input into SQL or JPQL identifiers. Add a unique tie-breaker (for example, ID) after the requested sort so equal values do not cause unstable page boundaries.

### Q14. What is cursor pagination's main trade-off?

It can avoid expensive deep offsets and behave better under some concurrent insert patterns, but it complicates arbitrary page jumps and requires stable cursor fields/order. Define whether cursors are opaque, signed, and bound to the original filter/sort criteria.

### Q15. How should request validation differ from domain validation?

Request validation checks shape and basic constraints at the boundary. Domain validation protects invariants regardless of whether the operation came from HTTP, messaging, or a job. Do not rely exclusively on controller annotations for business correctness.

### Q16. How should an API handle unsupported media types and oversized payloads?

Return a clear 415 response when the request media type is unsupported and a documented 413 response when a configured payload limit is exceeded, while recognizing that proxies or servlet containers may reject the request before controller code runs. Set limits at ingress and application layers, keep error bodies stable, and test which layer owns each response. Do not rely on JSON validation alone to constrain payload size.

### Q17. When should an API use streaming responses?

Use streaming for large exports, generated reports, or incremental feeds when avoiding whole-response buffering materially reduces memory or improves time-to-first-byte. Define framing/content type, cancellation, timeout, backpressure, and what happens if an error occurs after headers or partial content have been sent. Clients and intermediaries must support the chosen protocol; for small bounded responses, ordinary serialization is usually simpler.

### Q18. How should large file uploads be streamed and stored efficiently?

Avoid buffering the entire upload in heap memory. Stream to controlled temporary or object storage, enforce byte limits while streaming, apply backpressure/timeouts, and clean up incomplete uploads after cancellation or failure. Promote a file to its final state only after validation succeeds, and make retries or duplicate submissions safe. Keep the security checks from the earlier upload-design question: streaming is a resource strategy, not a substitute for authorization or content validation.
