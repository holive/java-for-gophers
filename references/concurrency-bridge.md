# Concurrency Bridge: Goroutines → Java

You love goroutines. Java's concurrency will feel heavier. Here's how to map your mental model.

## The Core Difference

**Go philosophy:** Concurrency is cheap. Spin up goroutines freely.
```go
for _, item := range items {
    go process(item)  // Thousands? No problem.
}
```

**Traditional Java philosophy:** Threads are expensive. Use pools carefully.

**Modern Java (21+):** Virtual threads bring Go-like lightness back.

## Quick Reference

| Go | Java (Traditional) | Java 21+ (Virtual Threads) |
|----|-------------------|---------------------------|
| `go func()` | `executor.submit()` | `Thread.startVirtualThread()` |
| Channels | BlockingQueue | (same, or use structured concurrency) |
| `select` | CompletableFuture.anyOf | (no direct equivalent) |
| `sync.WaitGroup` | CountDownLatch | Structured concurrency |
| `context.Context` | (no equivalent) | Scoped values |
| `sync.Mutex` | synchronized / Lock | (same) |

## Pattern 1: Fire and Forget

**Go:**
```go
go sendEmail(user)
```

**Java (Spring @Async):**
```java
@Service
public class EmailService {
    @Async
    public void sendEmail(User user) {
        // Runs in separate thread
    }
}

// Requires @EnableAsync on a config class
```

**Java 21 (Virtual Threads):**
```java
Thread.startVirtualThread(() -> sendEmail(user));
```

## Pattern 2: Parallel Processing with Results

**Go:**
```go
func processAll(items []Item) []Result {
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

**Java (CompletableFuture):**
```java
public List<Result> processAll(List<Item> items) {
    List<CompletableFuture<Result>> futures = items.stream()
        .map(item -> CompletableFuture.supplyAsync(() -> process(item)))
        .toList();
    
    return futures.stream()
        .map(CompletableFuture::join)
        .toList();
}
```

**Java (Parallel Stream — simpler but less control):**
```java
List<Result> results = items.parallelStream()
    .map(this::process)
    .toList();
```

**Java 21 (Virtual Threads + Structured Concurrency):**
```java
List<Result> processAll(List<Item> items) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        List<Subtask<Result>> tasks = items.stream()
            .map(item -> scope.fork(() -> process(item)))
            .toList();
        
        scope.join().throwIfFailed();
        
        return tasks.stream()
            .map(Subtask::get)
            .toList();
    }
}
```

## Pattern 3: Worker Pool

**Go:**
```go
func workerPool(jobs <-chan Job, results chan<- Result, workers int) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    wg.Wait()
    close(results)
}
```

**Java (ExecutorService):**
```java
ExecutorService executor = Executors.newFixedThreadPool(workers);

List<Future<Result>> futures = jobs.stream()
    .map(job -> executor.submit(() -> process(job)))
    .toList();

List<Result> results = futures.stream()
    .map(f -> {
        try { return f.get(); }
        catch (Exception e) { throw new RuntimeException(e); }
    })
    .toList();

executor.shutdown();
```

**Java 21 (Virtual Thread Pool):**
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Result>> futures = jobs.stream()
        .map(job -> executor.submit(() -> process(job)))
        .toList();
    // ...
}  // auto-shutdown
```

## Pattern 4: Timeout

**Go:**
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

select {
case result := <-doWork(ctx):
    return result, nil
case <-ctx.Done():
    return nil, ctx.Err()
}
```

**Java (CompletableFuture):**
```java
CompletableFuture<Result> future = CompletableFuture.supplyAsync(() -> doWork());

try {
    Result result = future.get(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true);
    throw new RuntimeException("Timeout");
}
```

**Java (with orTimeout):**
```java
Result result = CompletableFuture.supplyAsync(() -> doWork())
    .orTimeout(5, TimeUnit.SECONDS)
    .join();
```

## Pattern 5: Select (Multiple Channels)

**Go:**
```go
select {
case msg := <-channel1:
    handle1(msg)
case msg := <-channel2:
    handle2(msg)
case <-time.After(timeout):
    handleTimeout()
}
```

**Java (CompletableFuture.anyOf):**
```java
CompletableFuture<Object> first = CompletableFuture.anyOf(
    future1,
    future2,
    CompletableFuture.delayedExecutor(5, TimeUnit.SECONDS)
        .execute(() -> null)
);

// But you lose type safety and have to cast...
```

**Honest take:** Java doesn't have a clean equivalent to `select`. You'll often restructure to avoid needing it.

## Pattern 6: Mutex / Synchronized Access

**Go:**
```go
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

**Java (synchronized):**
```java
public class Counter {
    private int count;
    
    public synchronized void increment() {
        count++;
    }
}
```

**Java (explicit Lock):**
```java
public class Counter {
    private final Lock lock = new ReentrantLock();
    private int count;
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

**Java (Atomic):**
```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

## Spring Boot Specifics

### @Async Configuration

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### Virtual Threads in Spring Boot 3.2+

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

That's it. Spring Boot will use virtual threads for request handling. Your goroutine instincts will feel more at home.

## Mental Model Shift

**In Go:** You think about goroutines as the default. Blocking is fine because goroutines are cheap.

**In Java (traditional):** You think about thread pools. Blocking is expensive. You use reactive (WebFlux) or callbacks to avoid blocking.

**In Java 21+:** Virtual threads bring you back to the Go model. Blocking is cheap again. Write synchronous code, get concurrent behavior.

## Recommendation

If you're starting fresh with Java 21+, use virtual threads. They're the closest thing to goroutines and will feel natural.

If stuck on older Java, use `CompletableFuture` for most async work, and Spring's `@Async` when you don't need the result.
