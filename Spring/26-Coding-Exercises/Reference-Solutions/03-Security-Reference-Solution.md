# Reference Solution — JWT-Protected Resource Server

This is a configuration pattern, not a complete identity-provider deployment. Use the Spring Security OAuth 2.0 Resource Server support and configure the issuer/JWK URL for the trusted identity provider. Exact DSL options should be checked against the project's Spring Security version.

## Security configuration

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfiguration {
    @Bean
    SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.DELETE, "/api/orders/**")
                    .hasAuthority("SCOPE_orders.write")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

Configure the trusted issuer using the supported Boot properties, for example the issuer URI setting for the resource server. Do not accept a token merely because it can be Base64-decoded. The configured decoder validates the token signature and standard claims according to the configured trust model.

## Service-level authorization

```java
@Service
public class OrderAdministrationService {
    @PreAuthorize("hasAuthority('SCOPE_orders.write')")
    public void cancelOrder(String orderId) {
        // Load and validate ownership/state before changing the order.
    }
}
```

Method security must be enabled, the service must be Spring-managed, and the call must pass through the applicable proxy. Authorization should include resource and tenant checks when a scope alone is insufficient.

## Required tests

- Anonymous request to a protected endpoint is rejected.
- Authenticated principal without the required authority is denied.
- Principal with the required authority can reach the operation.
- Expired, malformed, wrong-issuer, and wrong-audience tokens are rejected according to configuration.
- CORS is tested separately and is not treated as authorization.
- CSRF configuration is justified by the credential transport model; do not disable it solely because the API uses REST.

## Common incorrect approaches

- Extending removed legacy configuration classes in modern Spring Security.
- Parsing JWT claims manually without signature and claim validation.
- Assuming authentication implies authorization.
- Trusting an arbitrary tenant header without binding it to authenticated identity.
- Returning the same status for every failure without documenting the API contract.
