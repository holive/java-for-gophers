# Exercise 00a: SOLID Principles

**Time:** 15 minutes
**Goal:** Learn the vocabulary of Java architecture discussions
**Mode:** Code review (reading, not writing)

---

## Why This Matters

In architecture discussions, you'll hear things like:

- "This violates SRP"
- "We should apply DIP here"
- "That breaks LSP"

If you don't know what these mean, you can't participate meaningfully. SOLID isn't just theory - it's the shared vocabulary of Java developers.

---

## The Principles (Quick Reference)

| Letter | Principle | One-liner |
|--------|-----------|-----------|
| S | Single Responsibility | One class, one reason to change |
| O | Open/Closed | Extend behavior without modifying code |
| L | Liskov Substitution | Subtypes must honor parent's contract |
| I | Interface Segregation | Small, focused interfaces |
| D | Dependency Inversion | Depend on abstractions |

---

## Your Task: Find the Violations

Review this Spring service class. Identify which SOLID principles it violates.

```java
@Service
public class OrderService {
    private final JdbcTemplate jdbc;
    private final JavaMailSender mailSender;

    public OrderService(JdbcTemplate jdbc, JavaMailSender mailSender) {
        this.jdbc = jdbc;
        this.mailSender = mailSender;
    }

    public Order createOrder(CreateOrderRequest request) {
        // validate
        if (request.items().isEmpty()) {
            throw new IllegalArgumentException("order must have items");
        }

        // calculate total
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItem item : request.items()) {
            BigDecimal itemTotal = item.price().multiply(BigDecimal.valueOf(item.quantity()));
            total = total.add(itemTotal);
        }

        // apply discount
        if (request.couponCode() != null) {
            if (request.couponCode().equals("SAVE10")) {
                total = total.multiply(BigDecimal.valueOf(0.9));
            } else if (request.couponCode().equals("SAVE20")) {
                total = total.multiply(BigDecimal.valueOf(0.8));
            }
            // add new coupon? modify this method
        }

        // save to database
        String orderId = UUID.randomUUID().toString();
        jdbc.update(
            "INSERT INTO orders (id, customer_id, total, status) VALUES (?, ?, ?, ?)",
            orderId, request.customerId(), total, "PENDING"
        );

        for (OrderItem item : request.items()) {
            jdbc.update(
                "INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (?, ?, ?, ?)",
                orderId, item.productId(), item.quantity(), item.price()
            );
        }

        // send confirmation email
        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(request.customerEmail());
        message.setSubject("Order Confirmation #" + orderId);
        message.setText("Thank you for your order! Total: $" + total);
        mailSender.send(message);

        return new Order(orderId, request.customerId(), total, "PENDING");
    }

    public List<Order> getOrdersByCustomer(String customerId) {
        return jdbc.query(
            "SELECT * FROM orders WHERE customer_id = ?",
            (rs, rowNum) -> new Order(
                rs.getString("id"),
                rs.getString("customer_id"),
                rs.getBigDecimal("total"),
                rs.getString("status")
            ),
            customerId
        );
    }

    public void cancelOrder(String orderId) {
        jdbc.update("UPDATE orders SET status = 'CANCELLED' WHERE id = ?", orderId);

        // refund logic would go here...
        // notification logic would go here...
    }
}
```

---

## Analysis (Try First, Then Read)

<details>
<summary>Click to reveal violations</summary>

### S - Single Responsibility: VIOLATED

This class has multiple reasons to change:
- Order business logic changes
- Database schema changes
- Email template changes
- Discount/coupon logic changes

**Fix:** Split into `OrderService`, `OrderRepository`, `EmailService`, `DiscountService`

### O - Open/Closed: VIOLATED

The coupon code logic requires modifying the method to add new coupons:

```java
if (request.couponCode().equals("SAVE10")) { ... }
else if (request.couponCode().equals("SAVE20")) { ... }
// new coupon = modify this code
```

**Fix:** Create a `DiscountStrategy` interface:

```java
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal total, String couponCode);
}
```

### L - Liskov Substitution: NOT APPLICABLE

No inheritance in this code. LSP typically applies when using class hierarchies.

### I - Interface Segregation: NOT DIRECTLY VIOLATED

But if this were an interface, it would be too fat (order creation, retrieval, cancellation, emailing all in one).

### D - Dependency Inversion: PARTIALLY VIOLATED

- Good: Uses `JdbcTemplate` (abstraction) not a concrete DB connection
- Bad: Uses `JavaMailSender` directly. If we want to swap to SMS or Slack notifications, we'd need to modify this class.

**Fix:** Depend on a `NotificationSender` interface.

</details>

---

## The Refactored Version

<details>
<summary>Click to see SOLID-compliant code</summary>

```java
// single responsibility: just orchestrates order creation
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final DiscountService discountService;
    private final NotificationService notificationService;

    public OrderService(
            OrderRepository orderRepository,
            DiscountService discountService,
            NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.discountService = discountService;
        this.notificationService = notificationService;
    }

    public Order createOrder(CreateOrderRequest request) {
        validateOrder(request);

        BigDecimal total = calculateTotal(request.items());
        total = discountService.applyDiscount(total, request.couponCode());

        Order order = orderRepository.save(new Order(
            UUID.randomUUID().toString(),
            request.customerId(),
            total,
            OrderStatus.PENDING
        ));

        notificationService.sendOrderConfirmation(order, request.customerEmail());

        return order;
    }

    private void validateOrder(CreateOrderRequest request) {
        if (request.items().isEmpty()) {
            throw new EmptyOrderException();
        }
    }

    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.price().multiply(BigDecimal.valueOf(item.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// open/closed: add new discounts without modifying existing code
public interface DiscountStrategy {
    boolean supports(String couponCode);
    BigDecimal apply(BigDecimal total);
}

@Service
public class DiscountService {
    private final List<DiscountStrategy> strategies;

    public DiscountService(List<DiscountStrategy> strategies) {
        this.strategies = strategies;  // Spring injects all implementations
    }

    public BigDecimal applyDiscount(BigDecimal total, String couponCode) {
        if (couponCode == null) return total;

        return strategies.stream()
            .filter(s -> s.supports(couponCode))
            .findFirst()
            .map(s -> s.apply(total))
            .orElse(total);
    }
}

// dependency inversion: depend on abstraction
public interface NotificationService {
    void sendOrderConfirmation(Order order, String recipient);
}

@Service
public class EmailNotificationService implements NotificationService {
    // implementation
}
```

</details>

---

## Go Comparison

Go encourages SOLID naturally:

| SOLID | How Go helps |
|-------|-------------|
| SRP | Small packages with focused purpose |
| OCP | Interfaces + new implementations |
| LSP | Implicit interfaces can't break contracts |
| ISP | Go idiom: 1-2 method interfaces |
| DIP | "Accept interfaces, return structs" |

The discipline required in Java is often implicit in Go's design. But Go developers can still write SOLID-violating code - it's just culturally discouraged.

---

## Checkpoint

You've completed this exercise when you can:

- [ ] Name all 5 SOLID principles from memory
- [ ] Spot SRP violations (class doing too many things)
- [ ] Spot OCP violations (if/else chains for extensibility)
- [ ] Explain why we depend on interfaces, not concrete classes

**Next:** Exercise 00b - Composition vs Inheritance
