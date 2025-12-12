# OOP Foundations for Gophers

A quick reference for object-oriented concepts you'll need for Java architecture discussions.

---

## SOLID Principles

The vocabulary of Java design discussions. When someone says "this violates SRP" or "we need to apply DIP here," you need to know what they mean.

| Principle | One-liner | Go Equivalent |
|-----------|-----------|---------------|
| **S**ingle Responsibility | A class should have one reason to change | Small packages, focused functions |
| **O**pen/Closed | Open for extension, closed for modification | Interfaces + new implementations |
| **L**iskov Substitution | Subtypes must be substitutable for base types | Implicit interfaces guarantee this |
| **I**nterface Segregation | Many small interfaces > one fat interface | Go idiom: 1-2 method interfaces |
| **D**ependency Inversion | Depend on abstractions, not concretions | Accept interfaces, return structs |

### S: Single Responsibility

```java
// bad: UserService does authentication AND user CRUD AND email sending
public class UserService {
    public User authenticate(String email, String password) { ... }
    public User create(CreateUserRequest req) { ... }
    public void sendWelcomeEmail(User user) { ... }
}

// good: split by responsibility
public class AuthService { ... }
public class UserService { ... }
public class EmailService { ... }
```

**Go comparison:** Go naturally encourages this through small packages. `auth/`, `user/`, `email/` would be separate packages.

### O: Open/Closed

```java
// bad: modify existing code to add new behavior
public class NotificationService {
    public void send(Notification n) {
        if (n.type().equals("email")) {
            // send email
        } else if (n.type().equals("sms")) {
            // send sms
        }
        // add new type? modify this method
    }
}

// good: extend without modifying
public interface NotificationSender {
    void send(Notification n);
}

public class EmailSender implements NotificationSender { ... }
public class SmsSender implements NotificationSender { ... }
// add new type? add new class, don't touch existing code
```

### L: Liskov Substitution

```java
// bad: Square breaks Rectangle's contract
public class Rectangle {
    protected int width, height;
    public void setWidth(int w) { width = w; }
    public void setHeight(int h) { height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) { width = height = w; }  // breaks LSP!
}

// code expecting Rectangle behavior:
void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50;  // fails for Square!
}
```

**Rule:** If your subclass changes behavior in ways that break expectations, don't use inheritance.

### I: Interface Segregation

```java
// bad: fat interface
public interface DataStore {
    void save(Entity e);
    Entity load(String id);
    void delete(String id);
    List<Entity> query(String sql);
    void beginTransaction();
    void commit();
    void rollback();
    void backup();
    void restore();
}

// good: small, focused interfaces
public interface EntityRepository {
    void save(Entity e);
    Entity load(String id);
    void delete(String id);
}

public interface Queryable {
    List<Entity> query(String sql);
}

public interface Transactional {
    void beginTransaction();
    void commit();
    void rollback();
}
```

**Go idiom:** This is Go's default. `io.Reader` has one method. `io.Writer` has one method. Compose them as needed.

### D: Dependency Inversion

```java
// bad: high-level depends on low-level
public class OrderService {
    private final PostgresRepository repo;  // concrete!

    public OrderService() {
        this.repo = new PostgresRepository();  // tightly coupled
    }
}

// good: both depend on abstraction
public class OrderService {
    private final OrderRepository repo;  // interface!

    public OrderService(OrderRepository repo) {  // injected
        this.repo = repo;
    }
}
```

**Go idiom:** "Accept interfaces, return structs" - same principle.

---

## Composition vs Inheritance

The most common design debate in Java.

### The Rule

> Favor composition over inheritance

This doesn't mean "never use inheritance." It means "when in doubt, use composition."

### When to Use Inheritance

- You have a genuine "is-a" relationship
- The subclass is a true specialization, not just sharing code
- The base class was designed for inheritance

```java
// appropriate inheritance: IOException hierarchy
public class IOException extends Exception { }
public class FileNotFoundException extends IOException { }
// FileNotFoundException IS genuinely an IOException
```

### When to Use Composition

- You want to reuse behavior (not identity)
- The relationship is "has-a" or "uses-a"
- You need flexibility to change implementations

```java
// bad: inheritance for code reuse
public class LoggingService extends FileWriter {
    // LoggingService IS-A FileWriter? no, it USES a FileWriter
}

// good: composition
public class LoggingService {
    private final Writer writer;  // has-a

    public LoggingService(Writer writer) {
        this.writer = writer;
    }
}
```

### Go Comparison

Go has embedding, which looks like inheritance but is composition:

```go
type LoggingService struct {
    writer io.Writer  // composition - this is what Java should do
}

// Go's embedding (still composition, not inheritance)
type EnhancedWriter struct {
    io.Writer  // embedded, but LoggingService doesn't "inherit" from Writer
}
```

### Decision Tree

```
Want to reuse code?
├── Is it an "is-a" relationship?
│   ├── Yes, and base class is designed for it → Inheritance OK
│   └── No, or unsure → Use composition
└── Need to swap implementations?
    └── Yes → Definitely composition
```

---

## Interface vs Abstract Class

| Use Interface When | Use Abstract Class When |
|-------------------|------------------------|
| Defining a contract | Providing partial implementation |
| Multiple implementations expected | Sharing code between related classes |
| Unrelated classes need this capability | You control all subclasses |
| You want to add to existing class hierarchies | You need non-public members |

### Interface (Java 8+)

```java
public interface Notifier {
    void notify(String message);  // abstract

    default void notifyAll(List<String> messages) {  // default implementation
        messages.forEach(this::notify);
    }
}
```

### Abstract Class

```java
public abstract class BaseRepository<T> {
    protected final JdbcTemplate jdbc;  // shared state

    protected BaseRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public final T findById(String id) {  // shared logic
        return jdbc.queryForObject(
            "SELECT * FROM " + tableName() + " WHERE id = ?",
            this::mapRow, id
        );
    }

    protected abstract String tableName();  // subclass provides
    protected abstract T mapRow(ResultSet rs, int rowNum);  // subclass provides
}
```

### Go Comparison

Go doesn't have abstract classes. The pattern is:

```go
// interface for the contract
type Repository interface {
    FindByID(id string) (*Entity, error)
}

// struct with common logic
type baseRepository struct {
    db *sql.DB
}

func (r *baseRepository) query(sql string, args ...any) (*sql.Rows, error) {
    return r.db.Query(sql, args...)
}

// embed base into concrete implementations
type UserRepository struct {
    baseRepository  // embedded
}
```

---

## Pattern Recognition Guide

Patterns you'll see in Spring codebases:

| When You See | It's Probably |
|--------------|---------------|
| Interface + multiple `@Service` implementations | Strategy pattern |
| `@Bean` method returning interface type | Factory pattern |
| `*Template` class (JdbcTemplate, RestTemplate) | Template Method pattern |
| `@EventListener` methods | Observer pattern |
| Fluent API with `.build()` at end | Builder pattern |
| Class wrapping another class, adding behavior | Decorator pattern |

### Strategy in Spring

```java
public interface PaymentProcessor {
    void process(Payment payment);
}

@Service("stripe")
public class StripeProcessor implements PaymentProcessor { ... }

@Service("paypal")
public class PaypalProcessor implements PaymentProcessor { ... }

// usage - inject by name or inject all
@Autowired
private Map<String, PaymentProcessor> processors;
```

### Factory in Spring

```java
@Configuration
public class ClientConfig {
    @Bean
    public HttpClient httpClient() {  // factory method
        return HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();
    }
}
```

### Template Method

```java
// Spring's JdbcTemplate handles connection, exception translation, etc.
// You provide the variable part (the query, the row mapper)
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users WHERE active = ?",
    (rs, rowNum) -> new User(rs.getString("id"), rs.getString("name")),
    true
);
```

---

## Quick Decision Reference

### "Should I create an interface?"

- Will there be multiple implementations? → Yes
- Do I need to mock this in tests? → Probably yes
- Is this a service class? → Yes, Spring works better with interfaces
- Is this a simple data holder? → No, use record

### "Should I use inheritance?"

- Is it truly "is-a"? → Maybe
- Am I just trying to share code? → No, use composition
- Does the parent class have `protected` methods designed for override? → Maybe
- Am I unsure? → No, use composition

### "Is this SOLID?"

- Does this class have one clear purpose? → S
- Can I add features without modifying this code? → O
- Can I swap this subclass for its parent? → L
- Is this interface small and focused? → I
- Does this depend on interfaces, not concrete classes? → D

---

## Recommended Reading

For deeper understanding:

1. **"Effective Java" by Joshua Bloch** - Items 18-23 cover composition vs inheritance
2. **"Head First Design Patterns"** - Visual, practical pattern introduction
3. **"Clean Architecture" by Robert Martin** - SOLID principles in depth
