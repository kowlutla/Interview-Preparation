# Reference Solution — Testcontainers and HTTP Client

## Testcontainers pattern

The exact annotations and service-connection integration vary by Spring Boot/Testcontainers version. A portable approach is to declare a container and expose its connection details through the test configuration mechanism supported by the project.

```java
@Testcontainers
@DataJpaTest
class ProductRepositoryIntegrationTest {
    @Container
    static final PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired ProductRepository repository;

    @Test
    void savesAndFindsProduct() {
        Product saved = repository.save(new Product("SKU-42", "Keyboard"));
        assertThat(repository.findById(saved.getId()))
            .isPresent()
            .get()
            .extracting(Product::getSku)
            .isEqualTo("SKU-42");
    }
}
```

This assumes a JPA entity/repository matching the test and the PostgreSQL JDBC driver/test dependencies. Pin an image version appropriate to your production database and CI policy; do not use a floating production tag for reproducible tests.

## RestClient pattern (Framework 6.1+)

```java
@Configuration
class InventoryClientConfiguration {
    @Bean
    RestClient inventoryRestClient(RestClient.Builder builder,
                                   InventoryClientProperties properties) {
        return builder
            .baseUrl(properties.baseUrl().toString())
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}
```

This illustrates client construction only. Configure connection/read timeouts through a supported request factory, define authentication, error mapping, and metrics, and use a total deadline compatible with the calling request. RestClient is not available in older Framework baselines; Framework 7 deprecates RestTemplate in favor of RestClient.

## Test strategy

- Use a mock HTTP server for client request shape and failure mapping.
- Test connection refusal and delayed response to verify timeout behavior.
- Use Testcontainers when database dialect, locking, migrations, or SQL behavior matters.
- Keep unit tests independent of Docker; isolate infrastructure integration tests in CI.
- Verify that secrets and complete request/response bodies are not logged.
