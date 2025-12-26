# Exercise 19: Circuit Breakers

**Time:** 15 minutes  
**Goal:** Add resilience patterns with Resilience4j

## The Go Version

```go
// Manual circuit breaker implementation
type CircuitBreaker struct {
    failures    int
    threshold   int
    state       string  // "closed", "open", "half-open"
    lastFailure time.Time
    timeout     time.Duration
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if cb.state == "open" {
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = "half-open"
        } else {
            return errors.New("circuit breaker open")
        }
    }
    
    err := fn()
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.threshold {
            cb.state = "open"
        }
        return err
    }
    
    cb.failures = 0
    cb.state = "closed"
    return nil
}
```

## Your Task

1. Add Resilience4j dependencies
2. Configure circuit breaker properties
3. Apply to external service calls
4. Add fallback methods

## Step by Step

### 1. Add Dependencies (2 min)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-aop'
    implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.1.0'
}
```

### 2. Configure Circuit Breaker (3 min)

In `application.yml`:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      externalApi:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
  
  retry:
    instances:
      externalApi:
        maxAttempts: 3
        waitDuration: 1s
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
  
  timelimiter:
    instances:
      externalApi:
        timeoutDuration: 5s
        cancelRunningFuture: true
```

### 3. Create External Service Client (5 min)

Create `ExternalApiClient.java`:

```java
package com.example.demo;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.concurrent.CompletableFuture;

@Service
public class ExternalApiClient {

    private static final Logger log = LoggerFactory.getLogger(ExternalApiClient.class);
    
    private final RestTemplate restTemplate;
    private final String baseUrl = "https://api.example.com";

    public ExternalApiClient() {
        this.restTemplate = new RestTemplate();
    }

    @CircuitBreaker(name = "externalApi", fallbackMethod = "getUserFallback")
    @Retry(name = "externalApi")
    public UserData getUser(String userId) {
        log.info("Calling external API for user: {}", userId);
        
        return restTemplate.getForObject(
            baseUrl + "/users/" + userId,
            UserData.class
        );
    }

    // Fallback method - same signature + Exception parameter
    private UserData getUserFallback(String userId, Exception e) {
        log.warn("Circuit breaker fallback for user {}: {}", userId, e.getMessage());
        
        return new UserData(
            userId,
            "Unknown User",
            "unavailable@example.com"
        );
    }

    // Async version with TimeLimiter
    @CircuitBreaker(name = "externalApi", fallbackMethod = "getOrdersFallback")
    @TimeLimiter(name = "externalApi")
    @Retry(name = "externalApi")
    public CompletableFuture<OrderList> getOrdersAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("Fetching orders for user: {}", userId);
            return restTemplate.getForObject(
                baseUrl + "/users/" + userId + "/orders",
                OrderList.class
            );
        });
    }

    private CompletableFuture<OrderList> getOrdersFallback(String userId, Exception e) {
        log.warn("Orders fallback for {}: {}", userId, e.getMessage());
        return CompletableFuture.completedFuture(new OrderList(List.of()));
    }
}

record UserData(String id, String name, String email) {}
record OrderList(List<Order> orders) {}
record Order(String id, String status) {}
```

### 4. Manual Circuit Breaker Usage (5 min)

For programmatic control:

```java
package com.example.demo;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.stereotype.Service;

import java.util.function.Supplier;

@Service
public class ManualCircuitBreakerService {

    private final CircuitBreaker circuitBreaker;
    private final RestTemplate restTemplate;

    public ManualCircuitBreakerService(CircuitBreakerRegistry registry) {
        this.circuitBreaker = registry.circuitBreaker("externalApi");
        this.restTemplate = new RestTemplate();
    }

    public String fetchData() {
        Supplier<String> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                return restTemplate.getForObject(
                    "https://api.example.com/data",
                    String.class
                );
            });

        return io.vavr.control.Try.ofSupplier(decoratedSupplier)
            .recover(throwable -> "fallback response")
            .get();
    }

    // Check circuit breaker state
    public String getCircuitBreakerStatus() {
        return circuitBreaker.getState().name();
    }
}
```

## Test It

```bash
# Make requests - some will fail, triggering circuit breaker
for i in {1..20}; do
  curl http://localhost:8080/external/user/123
  sleep 0.5
done

# Check circuit breaker state via actuator
curl http://localhost:8080/actuator/health | jq '.components.circuitBreakers'
```

## Go vs Spring Comparison

| Go | Spring (Resilience4j) |
|----|----------------------|
| Manual state tracking | `@CircuitBreaker` annotation |
| Manual failure counting | `slidingWindowSize`, `failureRateThreshold` |
| Manual timeout handling | `waitDurationInOpenState` |
| Return error when open | Fallback method called |
| No standard library | Full-featured library |

## Circuit Breaker States

```
CLOSED  →  (failures exceed threshold)  →  OPEN
   ↑                                          ↓
   ← (success in half-open) ← HALF_OPEN ← (wait duration expires)
```

- **CLOSED**: Normal operation, requests pass through
- **OPEN**: Requests fail immediately, call fallback
- **HALF_OPEN**: Limited requests to test if service recovered

## Configuration Options

```yaml
resilience4j:
  circuitbreaker:
    instances:
      myService:
        # Window configuration
        slidingWindowType: COUNT_BASED  # or TIME_BASED
        slidingWindowSize: 10           # 10 calls or 10 seconds
        minimumNumberOfCalls: 5         # Min calls before evaluation
        
        # Thresholds
        failureRateThreshold: 50        # % failures to trip
        slowCallRateThreshold: 80       # % slow calls to trip
        slowCallDurationThreshold: 2s   # What counts as slow
        
        # Recovery
        waitDurationInOpenState: 30s    # Time before half-open
        permittedNumberOfCallsInHalfOpenState: 3
        
        # What counts as failure
        recordExceptions:
          - java.io.IOException
        ignoreExceptions:
          - com.example.BusinessException
```

## Combining Patterns

```java
@CircuitBreaker(name = "api", fallbackMethod = "fallback")
@Retry(name = "api")
@RateLimiter(name = "api")
@Bulkhead(name = "api")
public Response callExternalApi() {
    // Order of execution (outside to inside):
    // Bulkhead → RateLimiter → Retry → CircuitBreaker → actual call
}
```

## Rate Limiter

```yaml
resilience4j:
  ratelimiter:
    instances:
      api:
        limitRefreshPeriod: 1s
        limitForPeriod: 10        # 10 requests per second
        timeoutDuration: 0s       # Don't wait, fail immediately
```

## Bulkhead (Concurrency Limiter)

```yaml
resilience4j:
  bulkhead:
    instances:
      api:
        maxConcurrentCalls: 25
        maxWaitDuration: 0s
```

## Monitoring

Circuit breaker metrics are auto-registered with Micrometer:

```bash
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.calls
```

## Checkpoint

- [ ] Circuit breaker trips after failures
- [ ] Fallback method returns default response
- [ ] Circuit opens and closes based on config
- [ ] Actuator shows circuit breaker state

**Next:** Exercise 20 - Integration Test with LocalStack
