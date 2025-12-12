# Exercise 00e: Code Smells & Refactoring

**Time:** 15 minutes
**Goal:** Recognize bad OOP and know the fix
**Mode:** Code review exercise

---

## What Are Code Smells?

Code smells are patterns that suggest deeper problems. They're not bugs - the code works - but they make the code harder to understand, modify, and test.

In architecture discussions, you need to articulate WHY code is problematic, not just feel that something is wrong.

---

## The 6 Smells You'll See Most

| Smell | Symptom | Fix |
|-------|---------|-----|
| God Class | Class does too much | Split by responsibility |
| Feature Envy | Method uses another class's data heavily | Move method to that class |
| Primitive Obsession | Using primitives for domain concepts | Create value objects |
| Long Parameter List | Method takes 5+ parameters | Introduce parameter object |
| Inappropriate Intimacy | Classes know too much about each other | Extract interface, reduce coupling |
| Shotgun Surgery | One change requires editing many files | Consolidate related logic |

---

## 1. God Class

**Smell:** A class that does everything.

```java
// bad: OrderService is a god class
@Service
public class OrderService {
    public Order create(CreateOrderRequest req) { /* 50 lines */ }
    public void cancel(String orderId) { /* 30 lines */ }
    public void refund(String orderId) { /* 40 lines */ }
    public List<Order> findByCustomer(String customerId) { /* 20 lines */ }
    public void sendConfirmationEmail(Order order) { /* 25 lines */ }
    public void updateInventory(Order order) { /* 35 lines */ }
    public BigDecimal calculateTax(Order order) { /* 20 lines */ }
    public void notifyWarehouse(Order order) { /* 15 lines */ }
    public Report generateSalesReport(DateRange range) { /* 60 lines */ }
    // ... 20 more methods
}
```

**Fix:** Split by responsibility.

```java
@Service public class OrderService { /* order lifecycle only */ }
@Service public class OrderRepository { /* data access only */ }
@Service public class OrderNotificationService { /* notifications only */ }
@Service public class InventoryService { /* inventory only */ }
@Service public class TaxCalculator { /* tax only */ }
@Service public class SalesReportService { /* reporting only */ }
```

**Go comparison:** Go's package system naturally prevents this. You'd have `order/`, `inventory/`, `notification/` packages.

---

## 2. Feature Envy

**Smell:** A method that uses another class's data more than its own.

```java
// bad: OrderPrinter is envious of Order
public class OrderPrinter {
    public String format(Order order) {
        StringBuilder sb = new StringBuilder();
        sb.append("Order #").append(order.getId()).append("\n");
        sb.append("Customer: ").append(order.getCustomer().getName()).append("\n");
        sb.append("Items:\n");
        for (OrderItem item : order.getItems()) {
            sb.append("  - ").append(item.getProductName())
              .append(" x").append(item.getQuantity())
              .append(" @ $").append(item.getPrice())
              .append(" = $").append(item.getPrice().multiply(
                  BigDecimal.valueOf(item.getQuantity())))
              .append("\n");
        }
        sb.append("Total: $").append(order.getTotal()).append("\n");
        return sb.toString();
    }
}
```

**Fix:** Move the method to the class whose data it uses.

```java
// good: Order knows how to format itself
public record Order(
    String id,
    Customer customer,
    List<OrderItem> items,
    BigDecimal total
) {
    public String toFormattedString() {
        // formatting logic here - uses its own data
    }
}

// or use a dedicated formatter that Order delegates to
public class OrderPrinter {
    public String format(Order order) {
        return order.toFormattedString();
    }
}
```

---

## 3. Primitive Obsession

**Smell:** Using `String`, `int`, etc. for domain concepts.

```java
// bad: primitives everywhere
public class User {
    private String id;           // what format? UUID? sequential?
    private String email;        // should be validated
    private String phoneNumber;  // what format? should be validated
    private int age;             // can be negative?
    private String country;      // free text or ISO code?
}

public void sendSms(String phoneNumber, String message) {
    // is phoneNumber valid? we don't know
}
```

**Fix:** Create value objects.

```java
// good: domain types
public record UserId(UUID value) {
    public UserId {
        Objects.requireNonNull(value);
    }

    public static UserId generate() {
        return new UserId(UUID.randomUUID());
    }
}

public record Email(String value) {
    private static final Pattern PATTERN = Pattern.compile("^[\\w-\\.]+@[\\w-]+\\.[a-z]{2,}$");

    public Email {
        if (!PATTERN.matcher(value).matches()) {
            throw new IllegalArgumentException("invalid email: " + value);
        }
    }
}

public record PhoneNumber(String countryCode, String number) {
    // validation in constructor
}

public record Age(int value) {
    public Age {
        if (value < 0 || value > 150) {
            throw new IllegalArgumentException("invalid age: " + value);
        }
    }
}

// now methods are type-safe
public void sendSms(PhoneNumber phoneNumber, String message) {
    // phoneNumber is guaranteed valid
}
```

**Go comparison:** Go has this too - you'd create `type Email string` with validation methods.

---

## 4. Long Parameter List

**Smell:** Method takes many parameters.

```java
// bad: too many parameters
public Order createOrder(
    String customerId,
    String customerEmail,
    List<OrderItem> items,
    String shippingStreet,
    String shippingCity,
    String shippingState,
    String shippingZip,
    String billingStreet,
    String billingCity,
    String billingState,
    String billingZip,
    String couponCode,
    boolean expedited
) {
    // ...
}
```

**Fix:** Introduce parameter objects.

```java
// good: grouped into meaningful objects
public record CreateOrderRequest(
    CustomerId customerId,
    Email customerEmail,
    List<OrderItem> items,
    Address shippingAddress,
    Address billingAddress,
    Optional<CouponCode> couponCode,
    ShippingSpeed shippingSpeed
) {}

public Order createOrder(CreateOrderRequest request) {
    // cleaner, self-documenting
}
```

---

## 5. Inappropriate Intimacy

**Smell:** Classes know too much about each other's internals.

```java
// bad: PaymentProcessor knows Order's internals
public class PaymentProcessor {
    public void process(Order order) {
        // reaching deep into Order's structure
        Customer customer = order.getCustomer();
        CreditCard card = customer.getPaymentMethods().get(0).getCreditCard();
        String cardNumber = card.getNumber();
        String cvv = card.getCvv();
        int expMonth = card.getExpirationMonth();
        int expYear = card.getExpirationYear();

        // now process with these details
    }
}
```

**Fix:** Tell, don't ask. Let objects do their own work.

```java
// good: Order provides what's needed
public class Order {
    public PaymentDetails getPaymentDetails() {
        return customer.getPrimaryPaymentMethod().toPaymentDetails();
    }
}

public class PaymentProcessor {
    public void process(Order order) {
        PaymentDetails details = order.getPaymentDetails();
        // process with details - don't know how it was constructed
    }
}
```

---

## 6. Shotgun Surgery

**Smell:** One logical change requires editing many files.

```java
// bad: adding a new notification type requires changes in:
// - NotificationType enum
// - NotificationService switch statement
// - NotificationTemplate factory
// - NotificationValidator
// - NotificationRepository
// - 3 test files
```

**Fix:** Consolidate related logic.

```java
// good: Strategy pattern - add new type by adding one class
public interface NotificationHandler {
    NotificationType type();
    void validate(Notification n);
    String renderTemplate(Notification n);
    void send(Notification n);
}

@Service
public class EmailNotificationHandler implements NotificationHandler {
    // all email-related logic in one place
}

// adding SMS? Create SmsNotificationHandler. One file.
```

---

## Your Task: Review This Code

Identify the smells in this service:

```java
@Service
public class UserService {
    private final JdbcTemplate jdbc;
    private final JavaMailSender mailer;

    public String createUser(String firstName, String lastName, String email,
                            String phone, String street, String city,
                            String state, String zip, String country,
                            boolean sendWelcomeEmail, boolean subscribeToNewsletter) {

        // validate email manually
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("invalid email");
        }

        // generate ID
        String id = UUID.randomUUID().toString();

        // save to database
        jdbc.update(
            "INSERT INTO users (id, first_name, last_name, email, phone, " +
            "street, city, state, zip, country, newsletter) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
            id, firstName, lastName, email, phone, street, city, state, zip, country, subscribeToNewsletter
        );

        // send welcome email
        if (sendWelcomeEmail) {
            SimpleMailMessage msg = new SimpleMailMessage();
            msg.setTo(email);
            msg.setSubject("Welcome!");
            msg.setText("Hello " + firstName + ",\n\nWelcome to our platform!");
            mailer.send(msg);
        }

        // log to analytics
        jdbc.update(
            "INSERT INTO analytics_events (event_type, user_id, timestamp) VALUES (?, ?, ?)",
            "USER_CREATED", id, Instant.now()
        );

        return id;
    }
}
```

<details>
<summary>Click to reveal analysis</summary>

**Smells identified:**

1. **Long Parameter List** - 11 parameters. Fix: `CreateUserRequest` record.

2. **Primitive Obsession** - Email, phone, address all as strings. Fix: Value objects.

3. **God Class symptoms** - Service handles user creation, email, AND analytics. Fix: Split into `UserService`, `EmailService`, `AnalyticsService`.

4. **Feature Envy** - Email formatting builds message from user data. Fix: Let `User` or `WelcomeEmail` handle this.

5. **Inappropriate Intimacy** - Direct SQL with all fields exposed. Fix: Use repository abstraction.

**Refactored:**

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final UserNotificationService notifications;
    private final AnalyticsService analytics;

    public UserId createUser(CreateUserRequest request) {
        User user = User.create(request);
        userRepository.save(user);

        if (request.sendWelcomeEmail()) {
            notifications.sendWelcome(user);
        }

        analytics.track(new UserCreatedEvent(user.id()));

        return user.id();
    }
}
```

</details>

---

## Checkpoint

You've completed this exercise when you can:

- [ ] Identify God Class (too many responsibilities)
- [ ] Identify Feature Envy (method uses other class's data)
- [ ] Identify Primitive Obsession (string/int for domain concepts)
- [ ] Identify Long Parameter List (5+ params)
- [ ] Articulate WHY these are problems (testability, maintainability, clarity)

---

## Phase 0 Complete

You now have the vocabulary and mental models to:

- Participate in architecture discussions
- Critique design decisions with specific reasoning
- Understand why Java codebases are structured the way they are

**Ready for Phase 1?** Start with Exercise 01 - Your First Spring Boot Endpoint.
