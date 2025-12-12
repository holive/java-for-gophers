# Exercise 00d: Design Patterns in Spring

**Time:** 15 minutes
**Goal:** Recognize common patterns in Spring codebases
**Mode:** Pattern recognition (low-energy friendly - just reading)

---

## Why Learn Patterns?

You don't need to memorize Gang of Four patterns. But in Java/Spring codebases, certain patterns appear constantly. Recognizing them helps you:

- Understand existing code faster
- Communicate with teammates ("this is a Strategy pattern")
- Know when to apply them yourself

---

## The 6 Patterns You'll See Most

| Pattern | What It Does | Spring Example |
|---------|-------------|----------------|
| Strategy | Swap algorithms at runtime | Multiple `@Service` implementations |
| Factory | Create objects without specifying class | `@Bean` methods, `BeanFactory` |
| Template Method | Define skeleton, subclasses fill in | `JdbcTemplate`, `RestTemplate` |
| Observer | Notify multiple listeners of events | `@EventListener`, `ApplicationEventPublisher` |
| Builder | Construct complex objects step by step | Lombok `@Builder`, fluent APIs |
| Decorator | Add behavior by wrapping | Filters, AOP proxies |

---

## 1. Strategy Pattern

**Intent:** Define a family of algorithms, make them interchangeable.

### In Spring

```java
// the strategy interface
public interface PaymentProcessor {
    void process(Payment payment);
    boolean supports(PaymentMethod method);
}

// concrete strategies
@Service
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public void process(Payment payment) { /* credit card logic */ }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.CREDIT_CARD;
    }
}

@Service
public class PayPalProcessor implements PaymentProcessor {
    @Override
    public void process(Payment payment) { /* paypal logic */ }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.PAYPAL;
    }
}

// context - uses strategies
@Service
public class PaymentService {
    private final List<PaymentProcessor> processors;

    public PaymentService(List<PaymentProcessor> processors) {
        this.processors = processors;  // Spring injects ALL implementations
    }

    public void pay(Payment payment) {
        processors.stream()
            .filter(p -> p.supports(payment.method()))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentMethodException(payment.method()))
            .process(payment);
    }
}
```

### Go Equivalent

```go
type PaymentProcessor interface {
    Process(payment Payment) error
    Supports(method PaymentMethod) bool
}

// same pattern, just no annotations
type PaymentService struct {
    processors []PaymentProcessor
}
```

---

## 2. Factory Pattern

**Intent:** Create objects without exposing creation logic.

### In Spring

```java
@Configuration
public class HttpClientConfig {
    // factory method - creates and configures the object
    @Bean
    public HttpClient httpClient() {
        return HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .followRedirects(HttpClient.Redirect.NORMAL)
            .build();
    }

    @Bean
    @Profile("dev")
    public HttpClient devHttpClient() {
        return HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(30))  // longer for dev
            .build();
    }
}

// usage - just inject, don't know how it's created
@Service
public class ApiClient {
    private final HttpClient httpClient;

    public ApiClient(HttpClient httpClient) {
        this.httpClient = httpClient;
    }
}
```

### Factory Method vs Abstract Factory

- **Factory Method:** One method creates one type (`@Bean` methods)
- **Abstract Factory:** Creates families of related objects (less common in Spring)

---

## 3. Template Method Pattern

**Intent:** Define the skeleton of an algorithm, let subclasses fill in steps.

### In Spring

```java
// Spring's JdbcTemplate is the classic example
// the template handles: connection, exception translation, resource cleanup
// you provide: the query and row mapping

List<User> users = jdbcTemplate.query(
    "SELECT * FROM users WHERE active = ?",
    (rs, rowNum) -> new User(        // you provide this part
        rs.getString("id"),
        rs.getString("email")
    ),
    true
);

// RestTemplate follows same pattern
ResponseEntity<User> response = restTemplate.getForEntity(
    "/users/{id}",
    User.class,
    userId
);
```

### Creating Your Own

```java
public abstract class BaseDataImporter {
    // template method - defines the algorithm
    public final void importData(InputStream source) {
        validate(source);
        List<RawRecord> records = parse(source);
        List<Entity> entities = records.stream()
            .map(this::transform)  // subclass provides
            .toList();
        save(entities);
        notifyCompletion(entities.size());
    }

    // steps subclasses implement
    protected abstract Entity transform(RawRecord record);
    protected abstract void save(List<Entity> entities);

    // steps with default implementation
    protected void validate(InputStream source) {
        Objects.requireNonNull(source);
    }

    protected void notifyCompletion(int count) {
        log.info("imported {} records", count);
    }
}

public class UserImporter extends BaseDataImporter {
    @Override
    protected Entity transform(RawRecord record) {
        return new User(record.get("email"), record.get("name"));
    }

    @Override
    protected void save(List<Entity> entities) {
        userRepository.saveAll(entities);
    }
}
```

---

## 4. Observer Pattern

**Intent:** When one object changes, notify all dependents.

### In Spring

```java
// define an event
public record OrderCreatedEvent(Order order, Instant timestamp) {}

// publish events
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public Order createOrder(CreateOrderRequest request) {
        Order order = // ... create order

        // publish - don't know who listens
        eventPublisher.publishEvent(new OrderCreatedEvent(order, Instant.now()));

        return order;
    }
}

// listen to events - decoupled from publisher
@Component
public class InventoryUpdater {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // update inventory
    }
}

@Component
public class NotificationSender {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // send confirmation email
    }
}

@Component
public class AnalyticsTracker {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // track metrics
    }
}
```

### Go Equivalent

Go doesn't have built-in observer support. You'd typically use channels:

```go
type OrderService struct {
    orderCreated chan Order
}

// listeners read from channel
go func() {
    for order := range orderService.orderCreated {
        // handle
    }
}()
```

---

## 5. Builder Pattern

**Intent:** Construct complex objects step by step.

### In Spring

```java
// Lombok makes this trivial
@Builder
public class NotificationRequest {
    private String recipient;
    private String subject;
    private String body;
    private Priority priority;
    private List<Attachment> attachments;
}

// usage
var request = NotificationRequest.builder()
    .recipient("user@example.com")
    .subject("Hello")
    .body("Welcome!")
    .priority(Priority.HIGH)
    .attachments(List.of())
    .build();

// fluent APIs in Spring itself
MockMvc mockMvc = MockMvcBuilders
    .standaloneSetup(controller)
    .setControllerAdvice(exceptionHandler)
    .addFilters(securityFilter)
    .build();
```

### Without Lombok

```java
public class NotificationRequest {
    private final String recipient;
    private final String subject;
    // ... fields

    private NotificationRequest(Builder builder) {
        this.recipient = builder.recipient;
        this.subject = builder.subject;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String recipient;
        private String subject;

        public Builder recipient(String recipient) {
            this.recipient = recipient;
            return this;
        }

        public Builder subject(String subject) {
            this.subject = subject;
            return this;
        }

        public NotificationRequest build() {
            return new NotificationRequest(this);
        }
    }
}
```

---

## 6. Decorator Pattern

**Intent:** Add behavior by wrapping, not inheriting.

### In Spring

```java
// filters are decorators around the servlet
@Component
public class LoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        log.info("request: {} {}", request.getMethod(), request.getRequestURI());

        filterChain.doFilter(request, response);  // pass to next decorator

        log.info("response: {}", response.getStatus());
    }
}

// AOP is decorator pattern applied automatically
@Aspect
@Component
public class PerformanceAspect {
    @Around("@annotation(Timed)")
    public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        Object result = joinPoint.proceed();  // call the wrapped method

        long duration = System.currentTimeMillis() - start;
        log.info("{} took {}ms", joinPoint.getSignature().getName(), duration);

        return result;
    }
}
```

---

## Pattern Spotting Exercise

Look at this code and identify the patterns:

```java
@Configuration
public class NotificationConfig {
    @Bean
    public NotificationSender emailSender(JavaMailSender mailer) {
        return new EmailNotificationSender(mailer);
    }
}

public interface NotificationSender {
    void send(Notification n);
    boolean supports(NotificationType type);
}

@Service
public class NotificationService {
    private final List<NotificationSender> senders;
    private final ApplicationEventPublisher events;

    public void notify(Notification n) {
        senders.stream()
            .filter(s -> s.supports(n.type()))
            .findFirst()
            .ifPresent(s -> {
                s.send(n);
                events.publishEvent(new NotificationSentEvent(n));
            });
    }
}
```

<details>
<summary>Click to reveal answer</summary>

- **Factory:** `@Bean` method creates `NotificationSender`
- **Strategy:** Multiple `NotificationSender` implementations, selected by `supports()`
- **Observer:** `ApplicationEventPublisher` notifies listeners of `NotificationSentEvent`

</details>

---

## Checkpoint

You've completed this exercise when you can:

- [ ] Spot Strategy pattern (interface + multiple implementations selected at runtime)
- [ ] Spot Factory pattern (`@Bean` methods, `*Factory` classes)
- [ ] Spot Template Method (abstract class with final method calling abstract steps)
- [ ] Spot Observer (`@EventListener`, `ApplicationEventPublisher`)
- [ ] Know when Builder is appropriate (many optional parameters)

**Next:** Exercise 00e - Code Smells & Refactoring
