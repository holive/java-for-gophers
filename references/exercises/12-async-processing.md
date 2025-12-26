# Exercise 12: Async Processing

**Time:** 15 minutes  
**Goal:** Run background tasks with @Async and CompletableFuture

## The Go Version

```go
// Fire and forget
go sendNotification(user)

// With result
func processAsync(items []Item) []Result {
    results := make(chan Result, len(items))
    
    for _, item := range items {
        go func(i Item) {
            results <- process(i)
        }(item)
    }
    
    var output []Result
    for range items {
        output = append(output, <-results)
    }
    return output
}
```

## Your Task

1. Enable async processing with `@EnableAsync`
2. Create fire-and-forget async methods
3. Use `CompletableFuture` for async with results
4. Compose multiple async operations

## Try First (Optional)

Before looking at the solution:
- Add `@EnableAsync` to a configuration class
- Create a method with `@Async` annotation
- Return `CompletableFuture<T>` for async with results

---

## Step by Step

### 1. Enable Async Support (2 min)

Create `AsyncConfig.java`:

```java
package com.example.demo;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

@Configuration
@EnableAsync
public class AsyncConfig {
    // Default executor is fine for learning
    // Production would customize thread pool
}
```

### 2. Create Async Service (5 min)

Create `NotificationService.java`:

```java
package com.example.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

    // Fire and forget - returns void
    @Async
    public void sendEmailAsync(String to, String subject) {
        log.info("Starting email send to {} on thread {}", to, Thread.currentThread().getName());
        
        // Simulate slow operation
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        log.info("Email sent to {}", to);
    }

    // Async with result
    @Async
    public CompletableFuture<String> sendEmailWithConfirmation(String to, String subject) {
        log.info("Starting email to {} on thread {}", to, Thread.currentThread().getName());
        
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        String confirmationId = "EMAIL-" + System.currentTimeMillis();
        log.info("Email sent, confirmation: {}", confirmationId);
        
        return CompletableFuture.completedFuture(confirmationId);
    }

    // Async computation
    @Async
    public CompletableFuture<Integer> computeExpensiveValue(int input) {
        log.info("Computing for {} on thread {}", input, Thread.currentThread().getName());
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        return CompletableFuture.completedFuture(input * input);
    }
}
```

### 3. Use Async Methods (5 min)

Create `TaskCompletionService.java`:

```java
package com.example.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;

@Service
public class TaskCompletionService {

    private static final Logger log = LoggerFactory.getLogger(TaskCompletionService.class);
    
    private final NotificationService notificationService;

    public TaskCompletionService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void completeTask(Task task) {
        log.info("Task completed: {}", task.getTitle());
        
        // Fire and forget - returns immediately
        notificationService.sendEmailAsync("user@example.com", "Task completed: " + task.getTitle());
        
        log.info("Returned immediately, email sending in background");
    }

    public String completeTaskWithConfirmation(Task task) {
        log.info("Task completed: {}", task.getTitle());
        
        // Wait for result
        CompletableFuture<String> future = notificationService
            .sendEmailWithConfirmation("user@example.com", "Task completed");
        
        // Block and get result (in real app, might return the Future instead)
        return future.join();
    }

    public List<Integer> parallelCompute(List<Integer> inputs) {
        log.info("Starting parallel computation for {} items", inputs.size());
        
        // Launch all computations in parallel
        List<CompletableFuture<Integer>> futures = inputs.stream()
            .map(notificationService::computeExpensiveValue)
            .toList();
        
        // Wait for all to complete
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        // Collect results
        return futures.stream()
            .map(CompletableFuture::join)
            .toList();
    }
}
```

### 4. Add Controller Endpoint (3 min)

```java
@PostMapping("/{id}/complete")
public Map<String, Object> completeTask(@PathVariable String id) {
    Task task = taskService.findById(id);
    taskCompletionService.completeTask(task);
    
    return Map.of(
        "message", "Task marked complete, notification sending in background",
        "taskId", id
    );
}
```

## Test It

```bash
# Create and complete a task
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Test async"}'

# Complete it (returns immediately, email sends in background)
curl -X POST http://localhost:8080/tasks/{id}/complete

# Watch logs - you'll see different thread names
```

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| `go func()` | `@Async` method |
| `chan Result` | `CompletableFuture<Result>` |
| `<-channel` | `future.join()` or `future.get()` |
| `sync.WaitGroup` | `CompletableFuture.allOf()` |
| Manual goroutine | Spring manages thread pool |

## CompletableFuture Patterns

### Chaining Operations

```java
CompletableFuture<String> result = service.fetchUser(id)
    .thenApply(user -> user.getEmail())           // Transform result
    .thenCompose(email -> service.sendEmail(email)) // Chain async call
    .exceptionally(ex -> "fallback@example.com");   // Handle errors
```

### Combining Results

```java
CompletableFuture<User> userFuture = service.fetchUser(id);
CompletableFuture<List<Order>> ordersFuture = service.fetchOrders(id);

// Wait for both
CompletableFuture<UserProfile> profile = userFuture
    .thenCombine(ordersFuture, (user, orders) -> new UserProfile(user, orders));
```

### First to Complete

```java
CompletableFuture<String> fast = service.fetchFromCache(id);
CompletableFuture<String> slow = service.fetchFromDb(id);

// Use whichever completes first
CompletableFuture<String> result = CompletableFuture.anyOf(fast, slow)
    .thenApply(obj -> (String) obj);
```

### Timeout

```java
CompletableFuture<String> result = service.slowOperation()
    .orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> "timeout fallback");
```

## Critical: The Proxy Trap

```java
@Service
public class MyService {
    
    @Async
    public void asyncMethod() {
        // This runs async when called from OUTSIDE
    }
    
    public void caller() {
        asyncMethod();  // NOT ASYNC! Bypasses proxy!
    }
}
```

**Why?** `@Async` works via Spring proxies. Internal calls bypass the proxy.

**Fix:** Call async methods from a different bean, or inject self:

```java
@Service
public class MyService {
    
    @Autowired
    private MyService self;  // Inject proxy
    
    public void caller() {
        self.asyncMethod();  // Now it's async
    }
}
```

## Custom Thread Pool (Production)

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

Use specific executor:

```java
@Async("taskExecutor")
public void asyncMethod() { }
```

## Virtual Threads (Java 21+)

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Now all async operations use lightweight virtual threads - closest to goroutines!

## Checkpoint

- [ ] `@EnableAsync` is configured
- [ ] Fire-and-forget method returns immediately
- [ ] `CompletableFuture` returns async results
- [ ] Logs show different thread names for async work

**Next:** Exercise 13 - SQS Consumer with @SqsListener
