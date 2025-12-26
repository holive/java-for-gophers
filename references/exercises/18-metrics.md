# Exercise 18: Metrics

**Time:** 15 minutes  
**Goal:** Add custom metrics and expose Prometheus format

## The Go Version

```go
import "github.com/prometheus/client_golang/prometheus"

var (
    tasksCreated = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "tasks_created_total",
        Help: "Total number of tasks created",
    })
    
    taskDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
        Name:    "task_processing_duration_seconds",
        Help:    "Time spent processing tasks",
        Buckets: prometheus.DefBuckets,
    })
)

func init() {
    prometheus.MustRegister(tasksCreated)
    prometheus.MustRegister(taskDuration)
}

func CreateTask() {
    start := time.Now()
    // ... create task
    tasksCreated.Inc()
    taskDuration.Observe(time.Since(start).Seconds())
}
```

## Your Task

1. Add Micrometer Prometheus dependency
2. Create custom counters and gauges
3. Add timing metrics with `@Timed`
4. View metrics in Prometheus format

## Step by Step

### 1. Add Dependencies (2 min)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

### 2. Configure Prometheus Endpoint (2 min)

In `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

### 3. Create Custom Metrics (6 min)

Create `TaskMetrics.java`:

```java
package com.example.demo;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;

@Component
public class TaskMetrics {

    private final Counter tasksCreatedCounter;
    private final Counter tasksCompletedCounter;
    private final Timer taskProcessingTimer;
    private final AtomicInteger activeTasks;

    public TaskMetrics(MeterRegistry registry) {
        // Counter - only goes up
        this.tasksCreatedCounter = Counter.builder("tasks.created")
            .description("Total tasks created")
            .register(registry);
        
        this.tasksCompletedCounter = Counter.builder("tasks.completed")
            .description("Total tasks completed")
            .register(registry);
        
        // Timer - measures duration and count
        this.taskProcessingTimer = Timer.builder("tasks.processing.time")
            .description("Time spent processing tasks")
            .register(registry);
        
        // Gauge - can go up or down
        this.activeTasks = new AtomicInteger(0);
        Gauge.builder("tasks.active", activeTasks, AtomicInteger::get)
            .description("Currently active tasks")
            .register(registry);
    }

    public void recordTaskCreated() {
        tasksCreatedCounter.increment();
        activeTasks.incrementAndGet();
    }

    public void recordTaskCompleted() {
        tasksCompletedCounter.increment();
        activeTasks.decrementAndGet();
    }

    public Timer.Sample startTimer() {
        return Timer.start();
    }

    public void stopTimer(Timer.Sample sample) {
        sample.stop(taskProcessingTimer);
    }
}
```

### 4. Use in Service (3 min)

Update `TaskService.java`:

```java
@Service
public class TaskService {

    private final TaskRepository repository;
    private final TaskMetrics metrics;

    public TaskService(TaskRepository repository, TaskMetrics metrics) {
        this.repository = repository;
        this.metrics = metrics;
    }

    public Task create(TaskRequest request) {
        Timer.Sample timer = metrics.startTimer();
        try {
            var task = new Task(request.title(), request.description(), "pending");
            Task saved = repository.save(task);
            
            metrics.recordTaskCreated();
            return saved;
        } finally {
            metrics.stopTimer(timer);
        }
    }

    public void complete(String id) {
        Task task = findById(id);
        task.setStatus("done");
        repository.save(task);
        
        metrics.recordTaskCompleted();
    }
}
```

### 5. Use @Timed Annotation (2 min)

Simpler approach for timing:

```java
import io.micrometer.core.annotation.Timed;

@Service
public class TaskService {

    @Timed(value = "task.creation", description = "Time to create a task")
    public Task create(TaskRequest request) {
        // ... implementation
    }
}
```

Enable with:

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

## Test It

```bash
# View all metrics
curl http://localhost:8080/actuator/metrics

# View specific metric
curl http://localhost:8080/actuator/metrics/tasks.created

# Prometheus format
curl http://localhost:8080/actuator/prometheus
```

Prometheus output:

```
# HELP tasks_created_total Total tasks created
# TYPE tasks_created_total counter
tasks_created_total{application="demo"} 5.0

# HELP tasks_active Currently active tasks
# TYPE tasks_active gauge
tasks_active{application="demo"} 3.0

# HELP tasks_processing_time_seconds Time spent processing tasks
# TYPE tasks_processing_time_seconds summary
tasks_processing_time_seconds_count{application="demo"} 5.0
tasks_processing_time_seconds_sum{application="demo"} 0.123
```

## Go vs Spring Comparison

| Go (prometheus client) | Spring (Micrometer) |
|------------------------|---------------------|
| `prometheus.NewCounter()` | `Counter.builder().register()` |
| `counter.Inc()` | `counter.increment()` |
| `prometheus.NewGauge()` | `Gauge.builder().register()` |
| `gauge.Set(value)` | Use `AtomicInteger` + supplier |
| `prometheus.NewHistogram()` | `Timer.builder().register()` |
| `histogram.Observe(duration)` | `timer.record(duration)` |

## Metric Types

### Counter (Only Increases)

```java
Counter counter = Counter.builder("events.processed")
    .tag("type", "task")
    .register(registry);

counter.increment();
counter.increment(5.0);
```

### Gauge (Current Value)

```java
AtomicInteger queueSize = new AtomicInteger();

Gauge.builder("queue.size", queueSize, AtomicInteger::get)
    .register(registry);

// Update anytime
queueSize.set(42);
```

### Timer (Duration + Count)

```java
Timer timer = Timer.builder("api.latency")
    .publishPercentiles(0.5, 0.95, 0.99)  // Add percentiles
    .register(registry);

// Option 1: Sample-based
Timer.Sample sample = Timer.start();
// ... do work
sample.stop(timer);

// Option 2: Wrap callable
timer.record(() -> doWork());

// Option 3: Direct duration
timer.record(Duration.ofMillis(123));
```

### Distribution Summary (Non-Time Values)

```java
DistributionSummary summary = DistributionSummary.builder("file.size")
    .baseUnit("bytes")
    .register(registry);

summary.record(fileSize);
```

## Tags (Labels)

```java
Counter.builder("http.requests")
    .tag("method", "GET")
    .tag("status", "200")
    .tag("uri", "/tasks")
    .register(registry)
    .increment();
```

## Built-in Metrics

Spring Boot auto-configures:

- `http.server.requests` - HTTP request metrics
- `jvm.memory.*` - JVM memory
- `jvm.gc.*` - Garbage collection
- `process.cpu.*` - CPU usage
- `jdbc.connections.*` - Database connections
- `spring.data.repository.*` - Repository method timing

## Prometheus + Grafana Setup

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

## Checkpoint

- [ ] `/actuator/prometheus` returns metrics
- [ ] Custom counter increments on task creation
- [ ] Timer records processing duration
- [ ] Gauge shows active task count

**Next:** Exercise 19 - Circuit Breakers with Resilience4j
