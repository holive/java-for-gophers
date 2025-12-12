# Exercise 00c: Interface Design

**Time:** 15 minutes
**Goal:** Master Java interfaces and know when to use abstract classes
**Mode:** Design exercise

---

## Java Interfaces vs Go Interfaces

The biggest mental shift:

| Go | Java |
|----|----|
| Implicit: any type with matching methods satisfies the interface | Explicit: must declare `implements` |
| Define interfaces where you USE them | Define interfaces where you PROVIDE them |
| Typically 1-2 methods | Can be larger (but shouldn't be) |

```go
// go: interface defined by consumer
type UserFinder interface {
    FindByID(id string) (*User, error)
}

// any struct with this method satisfies it - no declaration needed
type PostgresRepo struct { }
func (r *PostgresRepo) FindByID(id string) (*User, error) { ... }
```

```java
// java: interface defined by provider, explicitly implemented
public interface UserRepository {
    Optional<User> findById(String id);
}

public class PostgresUserRepository implements UserRepository {
    @Override
    public Optional<User> findById(String id) { ... }
}
```

---

## Modern Java Interfaces (Java 8+)

Java interfaces aren't just method signatures anymore:

```java
public interface NotificationSender {
    // abstract method - implementors must provide
    void send(String recipient, String message);

    // default method - implementors inherit (can override)
    default void sendBatch(List<String> recipients, String message) {
        recipients.forEach(r -> send(r, message));
    }

    // static method - utility, called on interface itself
    static NotificationSender noOp() {
        return (recipient, message) -> { };  // does nothing
    }
}
```

### Default Methods: When to Use

- Adding methods to existing interfaces without breaking implementations
- Providing sensible defaults that most implementations share
- NOT for adding complex logic (that belongs in an abstract class)

---

## Your Task: Design a Notification System

Design an interface hierarchy for a notification system that supports:

1. Different channels (email, SMS, Slack, push notification)
2. Different message types (plain text, HTML, templated)
3. Batching (send to multiple recipients)
4. Retry logic (some channels may need it)

**Constraints:**
- Follow Interface Segregation (small, focused interfaces)
- Make it easy to add new channels without modifying existing code
- Make it testable

---

## Solution

<details>
<summary>Click to reveal a well-designed interface</summary>

```java
// core interface - minimal
public interface NotificationSender {
    void send(Notification notification);
}

// the notification itself (data)
public record Notification(
    String recipient,
    String subject,
    String body,
    NotificationType type
) {}

public enum NotificationType {
    PLAIN_TEXT, HTML, TEMPLATE
}

// optional capability: batching
public interface BatchCapable {
    default void sendBatch(List<Notification> notifications) {
        notifications.forEach(this::send);
    }

    void send(Notification notification);
}

// optional capability: retry
public interface Retryable {
    int maxRetries();
    Duration retryDelay();
}

// implementations compose capabilities
@Service
public class EmailSender implements NotificationSender, BatchCapable, Retryable {
    private final JavaMailSender mailSender;

    @Override
    public void send(Notification notification) {
        // send via email
    }

    @Override
    public void sendBatch(List<Notification> notifications) {
        // optimized batch send
    }

    @Override
    public int maxRetries() { return 3; }

    @Override
    public Duration retryDelay() { return Duration.ofSeconds(5); }
}

@Service
public class SlackSender implements NotificationSender {
    // slack doesn't need batch or retry - just implements core interface

    @Override
    public void send(Notification notification) {
        // send via slack webhook
    }
}

// orchestrator checks capabilities
@Service
public class NotificationService {
    private final Map<String, NotificationSender> senders;

    public void send(String channel, Notification notification) {
        NotificationSender sender = senders.get(channel);

        if (sender instanceof Retryable retryable) {
            sendWithRetry(sender, notification, retryable);
        } else {
            sender.send(notification);
        }
    }

    private void sendWithRetry(NotificationSender sender, Notification n, Retryable config) {
        // retry logic using config.maxRetries() and config.retryDelay()
    }
}
```

</details>

---

## Interface vs Abstract Class

| Use Interface | Use Abstract Class |
|---------------|-------------------|
| Contract definition | Partial implementation |
| Multiple inheritance of type | Shared state or complex logic |
| Unrelated classes need capability | Related classes in hierarchy |
| Spring dependency injection | Template method pattern |

### When to Use Abstract Class

```java
// template method pattern - abstract class is appropriate
public abstract class BaseNotificationSender {
    // shared logic
    protected final AuditLogger auditLogger;

    protected BaseNotificationSender(AuditLogger auditLogger) {
        this.auditLogger = auditLogger;
    }

    // template method
    public final void send(Notification notification) {
        validate(notification);
        doSend(notification);
        auditLogger.log("Sent notification to " + notification.recipient());
    }

    protected void validate(Notification notification) {
        Objects.requireNonNull(notification.recipient());
        Objects.requireNonNull(notification.body());
    }

    // subclasses implement this
    protected abstract void doSend(Notification notification);
}

public class EmailSender extends BaseNotificationSender {
    @Override
    protected void doSend(Notification notification) {
        // email-specific sending
    }
}
```

---

## Functional Interfaces

Interfaces with exactly one abstract method can be lambdas:

```java
@FunctionalInterface
public interface NotificationSender {
    void send(Notification notification);
}

// can be used as lambda
NotificationSender consoleSender = n -> System.out.println(n.body());

// or method reference
NotificationSender sender = this::sendViaEmail;
```

Spring uses this everywhere:
- `Function<T, R>` - transform
- `Consumer<T>` - accept and process
- `Supplier<T>` - provide
- `Predicate<T>` - test

---

## Go Comparison

Go's approach is simpler but less flexible:

```go
// go: can't have default implementations
type NotificationSender interface {
    Send(n Notification) error
}

// composition instead of default methods
type BatchSender struct {
    sender NotificationSender
}

func (b *BatchSender) SendBatch(notifications []Notification) error {
    for _, n := range notifications {
        if err := b.sender.Send(n); err != nil {
            return err
        }
    }
    return nil
}
```

Go forces you to use composition where Java lets you use default methods. Both work; Java's approach is more convenient, Go's is more explicit.

---

## Design Guidelines

### Keep Interfaces Small

```java
// bad: fat interface
public interface UserService {
    User create(CreateUserRequest req);
    User update(UpdateUserRequest req);
    void delete(String id);
    User findById(String id);
    List<User> findAll();
    List<User> findByEmail(String email);
    void sendWelcomeEmail(User user);
    void resetPassword(String email);
    boolean validateCredentials(String email, String password);
}

// good: segregated interfaces
public interface UserRepository {
    User save(User user);
    Optional<User> findById(String id);
    List<User> findAll();
}

public interface UserAuthenticator {
    boolean validateCredentials(String email, String password);
    void resetPassword(String email);
}

public interface UserNotifier {
    void sendWelcomeEmail(User user);
}
```

### Name Interfaces by Capability

```java
// describes what it does, not what it is
public interface Sendable { }
public interface Retryable { }
public interface Cacheable { }

// or by role
public interface UserRepository { }
public interface NotificationSender { }
```

---

## Checkpoint

You've completed this exercise when you can:

- [ ] Design a small, focused interface for a given requirement
- [ ] Explain when to use interface vs abstract class
- [ ] Use default methods appropriately
- [ ] Recognize when an interface is too fat (violates ISP)

**Next:** Exercise 00d - Design Patterns in Spring
