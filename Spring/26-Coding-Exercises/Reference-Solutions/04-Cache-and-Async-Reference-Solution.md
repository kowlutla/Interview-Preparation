# Reference Solution — Cache Invalidation and Bounded Async Work

## Cache example

```java
@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable(cacheNames = "products", key = "#productId")
    public ProductDto find(String productId) {
        return repository.findById(productId)
            .map(ProductDto::from)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    @CacheEvict(cacheNames = "products", key = "#productId")
    @Transactional
    public void update(String productId, UpdateProductRequest request) {
        Product product = repository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        product.update(request.name(), request.price());
    }
}
```

The example illustrates a key and invalidation path. Verify cache/transaction ordering and provider behavior for the selected configuration; for strong correctness, consider transaction-aware cache behavior or evict after successful commit. If updates can happen through other services, local method annotations alone cannot invalidate every cache.

## Bounded executor configuration

```java
@Configuration
@EnableAsync
public class AsyncConfiguration {
    @Bean(name = "notificationExecutor")
    ThreadPoolTaskExecutor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("notification-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(20);
        return executor;
    }
}
```

Use `@Async("notificationExecutor")` on a Spring-managed method called through its proxy. Define rejection and failure handling explicitly for the workload. A bounded in-memory executor is not durable delivery; use a broker or persistent job mechanism if work must survive process termination.

## Tests and validation

- Verify repeated reads hit the cache with the configured provider.
- Update data and confirm the next read sees fresh data after successful commit.
- Test rollback behavior and cache invalidation timing.
- Saturate the executor and observe queue/rejection metrics.
- Verify async exceptions are observed by the caller or an appropriate exception handler.
- Test shutdown with queued tasks and confirm the documented policy.
