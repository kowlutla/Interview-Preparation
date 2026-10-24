# Reference Solution — Validated Order REST API

This is a compact reference implementation for exercises covering DTOs, constructor injection, validation, HTTP status, and global errors. It targets a Spring Boot 3.x line with Java 17+ and Jakarta Bean Validation; check imports and dependency versions for your selected Boot release.

## Requirements

- Accept `POST /api/orders` with a nonblank product code and positive quantity.
- Return 201 with a response DTO.
- Return a stable validation error for invalid input.
- Do not expose persistence entities.

## Request and response DTOs

Place each public top-level record in its own Java source file (for example, `CreateOrderRequest.java` and `OrderResponse.java`) under the package shown below.

```java
package example.orders.api;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;

public record CreateOrderRequest(
    @NotBlank String productCode,
    @Positive int quantity
) {}

public record OrderResponse(
    String id,
    String productCode,
    int quantity
) {}
```

## Service boundary

```java
package example.orders.application;

import example.orders.api.CreateOrderRequest;
import example.orders.api.OrderResponse;
import org.springframework.stereotype.Service;

import java.util.UUID;

@Service
public class OrderService {
    public OrderResponse create(CreateOrderRequest request) {
        // Replace this in-memory example with a repository-backed transaction.
        return new OrderResponse(
            UUID.randomUUID().toString(),
            request.productCode(),
            request.quantity()
        );
    }
}
```

## Controller

```java
package example.orders.api;

import example.orders.application.OrderService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService service;

    public OrderController(OrderService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(
            @Valid @RequestBody CreateOrderRequest request) {
        OrderResponse created = service.create(request);
        return ResponseEntity.created(
            URI.create("/api/orders/" + created.id())
        ).body(created);
    }
}
```

## Error contract

```java
package example.orders.api;

public record ApiError(String code, String message) {}
```

```java
package example.orders.api;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class ApiExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> invalidRequest(
            MethodArgumentNotValidException exception) {
        return ResponseEntity.badRequest().body(
            new ApiError("INVALID_REQUEST", "Request validation failed")
        );
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> unexpected(Exception exception) {
        // Log the exception with a correlation ID in a real application.
        // Do not return exception.getMessage() or a stack trace to clients.
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ApiError("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}
```

The broad fallback handler is shown to illustrate the boundary, but a production implementation should log the unexpected exception once with safe context and should not hide application-specific exception mappings.

## MockMvc test sketch

```java
@WebMvcTest(OrderController.class)
@Import(ApiExceptionHandler.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;

    @Test
    void rejectsInvalidQuantity() throws Exception {
        mvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"productCode":"SKU-1","quantity":0}
                    """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.code").value("INVALID_REQUEST"));
        verifyNoInteractions(service);
    }
}
```

If the project's selected Spring generation uses a newer test bean replacement annotation, use the API supported by that dependency baseline. Include the web starter, validation starter, and test starter in the build.

## Common incorrect approaches

- Returning a JPA entity directly from the controller.
- Omitting request validation and trusting clients.
- Returning HTTP 200 for failed creation.
- Exposing exception messages or stack traces.
- Mocking the controller rather than testing MVC request handling.

## Validation checklist

- [ ] Valid request returns 201 and a response DTO.
- [ ] Missing/blank product code returns 400.
- [ ] Zero or negative quantity returns 400.
- [ ] Malformed JSON returns the documented client error.
- [ ] Unexpected exceptions return a safe 500 contract.
- [ ] Tests verify the service is not invoked for invalid input.
