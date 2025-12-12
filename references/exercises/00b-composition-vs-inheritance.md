# Exercise 00b: Composition vs Inheritance

**Time:** 15 minutes
**Goal:** Understand the most common design debate in Java
**Mode:** Refactoring exercise

---

## The Golden Rule

> Favor composition over inheritance

This is probably the most-cited OOP principle. But what does it mean, and when should you break it?

---

## When Inheritance Goes Wrong

Here's a common mistake. A developer notices that `AdminUser` and `RegularUser` share code, so they create a hierarchy:

```java
public class User {
    protected String id;
    protected String email;
    protected String name;

    public User(String id, String email, String name) {
        this.id = id;
        this.email = email;
        this.name = name;
    }

    public void sendNotification(String message) {
        System.out.println("Sending to " + email + ": " + message);
    }

    public boolean canAccessResource(String resourceId) {
        return false;  // default: no access
    }
}

public class RegularUser extends User {
    private List<String> ownedResources;

    public RegularUser(String id, String email, String name, List<String> ownedResources) {
        super(id, email, name);
        this.ownedResources = ownedResources;
    }

    @Override
    public boolean canAccessResource(String resourceId) {
        return ownedResources.contains(resourceId);
    }
}

public class AdminUser extends User {
    private LocalDateTime adminSince;

    public AdminUser(String id, String email, String name, LocalDateTime adminSince) {
        super(id, email, name);
        this.adminSince = adminSince;
    }

    @Override
    public boolean canAccessResource(String resourceId) {
        return true;  // admins can access everything
    }

    public void revokeAccess(User user, String resourceId) {
        // admin-only operation
    }
}
```

### The Problems

1. **Tight coupling:** What if we need a `GuestUser` with no email? Can't extend `User`.

2. **Fragile base class:** If we add a method to `User`, all subclasses are affected.

3. **Inflexible:** What if someone is both an admin AND has special notification preferences? Can't do multiple inheritance.

4. **Testing difficulty:** To test `AdminUser`, you must understand `User` too.

---

## Your Task: Refactor to Composition

Refactor the code above to use composition instead of inheritance.

**Requirements:**
- Users should still have id, email, name
- Access control should be pluggable
- Notification sending should be pluggable
- It should be easy to add new behaviors without modifying existing classes

**Hint:** Think about what behaviors are, not what things are.

---

## Solution

<details>
<summary>Click to reveal the composition-based design</summary>

```java
// core data - just data, no behavior
public record User(String id, String email, String name) {}

// access control is a behavior, not an identity
public interface AccessPolicy {
    boolean canAccess(User user, String resourceId);
}

@Component
public class OwnershipBasedAccess implements AccessPolicy {
    private final ResourceRepository resources;

    public OwnershipBasedAccess(ResourceRepository resources) {
        this.resources = resources;
    }

    @Override
    public boolean canAccess(User user, String resourceId) {
        return resources.isOwnedBy(resourceId, user.id());
    }
}

@Component
public class AdminAccess implements AccessPolicy {
    private final AdminRepository admins;

    public AdminAccess(AdminRepository admins) {
        this.admins = admins;
    }

    @Override
    public boolean canAccess(User user, String resourceId) {
        return admins.isAdmin(user.id());
    }
}

// notification is a behavior, not an identity
public interface NotificationSender {
    void send(User user, String message);
}

@Component
public class EmailNotificationSender implements NotificationSender {
    private final EmailService emailService;

    @Override
    public void send(User user, String message) {
        emailService.send(user.email(), message);
    }
}

// compose behaviors as needed
@Service
public class UserService {
    private final List<AccessPolicy> accessPolicies;
    private final NotificationSender notificationSender;

    public UserService(List<AccessPolicy> accessPolicies, NotificationSender notificationSender) {
        this.accessPolicies = accessPolicies;
        this.notificationSender = notificationSender;
    }

    public boolean canAccess(User user, String resourceId) {
        // any policy granting access is sufficient
        return accessPolicies.stream()
            .anyMatch(policy -> policy.canAccess(user, resourceId));
    }

    public void notify(User user, String message) {
        notificationSender.send(user, message);
    }
}
```

</details>

---

## Why This Is Better

| Inheritance Approach | Composition Approach |
|---------------------|---------------------|
| `AdminUser` IS-A `User` | `User` HAS-A `AccessPolicy` |
| Adding behavior = new subclass | Adding behavior = new component |
| Can't combine behaviors | Compose freely |
| Tight coupling | Loose coupling |
| Hard to test in isolation | Easy to mock policies |

---

## When Inheritance IS Appropriate

Inheritance isn't always wrong. Use it when:

1. **True "is-a" relationship exists**
   ```java
   // FileNotFoundException IS genuinely an IOException
   public class FileNotFoundException extends IOException { }
   ```

2. **The base class was designed for extension**
   ```java
   // Spring's OncePerRequestFilter is designed to be extended
   public class LoggingFilter extends OncePerRequestFilter {
       @Override
       protected void doFilterInternal(...) { }
   }
   ```

3. **You control both classes and their evolution**
   - Internal framework code where you manage all subclasses
   - Test doubles extending real classes

---

## Go Comparison

Go doesn't have inheritance at all. It has embedding:

```go
type User struct {
    ID    string
    Email string
    Name  string
}

type AdminUser struct {
    User        // embedded - looks like inheritance but isn't
    AdminSince  time.Time
}

// but this is still composition - AdminUser HAS-A User
// Go just provides syntactic sugar
```

Go's design forced you to think in composition from day one. Java lets you choose, and the wrong choice is tempting.

---

## Decision Flowchart

```
Do I need to reuse code between classes?
│
├─ Is it a true "is-a" relationship?
│  │
│  ├─ Yes → Is the base class designed for extension?
│  │        │
│  │        ├─ Yes → Inheritance might be OK
│  │        │
│  │        └─ No → Use composition
│  │
│  └─ No → Use composition
│
└─ Am I just avoiding code duplication?
   │
   └─ Yes → Extract shared code to a helper class (composition)
```

---

## Checkpoint

You've completed this exercise when you can:

- [ ] Explain "favor composition over inheritance" to a colleague
- [ ] Identify inheritance used for code sharing (wrong) vs true specialization (OK)
- [ ] Refactor an inheritance hierarchy to composition
- [ ] Know when inheritance IS appropriate (framework extension, true is-a)

**Next:** Exercise 00c - Interface Design
