# Java Idioms: When to Forget Go

Go-to-Java translation helps you understand. But good Java code isn't "Go translated to Java" — it's idiomatic Java. This reference covers patterns where you should stop thinking in Go.

## 1. Exceptions Are Not Errors

### Go Instinct
```go
user, err := findUser(id)
if err != nil {
    return nil, fmt.Errorf("finding user: %w", err)
}
```

### Bad Java (Go mentality)
```java
Optional<User> maybeUser = findUser(id);
if (maybeUser.isEmpty()) {
    return Optional.empty();  // Propagating "error" via Optional
}
User user = maybeUser.get();
```

### Good Java
```java
// In service
public User findUser(String id) {
    return repository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}

// Let exceptions bubble up. Handle at boundaries.
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handle(UserNotFoundException e) {
        return new ErrorResponse(e.getMessage());
    }
}
```

**Why:** Java's exception system is designed for this. Fighting it creates awkward code and loses stack traces. Exceptions are not slow (that's a myth for normal usage).

## 2. Trust Dependency Injection

### Go Instinct
```go
// Wire everything explicitly in main()
db := database.Connect()
userRepo := repository.NewUserRepo(db)
userService := service.NewUserService(userRepo)
handler := handler.NewUserHandler(userService)
```

### Bad Java (Go mentality)
```java
// Manual wiring, ignoring Spring
public class Application {
    public static void main(String[] args) {
        var repo = new UserRepository();
        var service = new UserService(repo);
        var controller = new UserController(service);
        // How do I even use this with Spring Boot?
    }
}
```

### Good Java
```java
@Service
public class UserService {
    private final UserRepository repo;
    
    public UserService(UserRepository repo) {  // Spring injects automatically
        this.repo = repo;
    }
}

@RestController
public class UserController {
    private final UserService service;
    
    public UserController(UserService service) {  // Spring injects automatically
        this.service = service;
    }
}
```

**Why:** Spring manages object lifecycles, handles scoping (singleton, request, etc.), and enables powerful features like AOP. Manual wiring gives up all of this.

## 3. Use Framework Features, Don't Rebuild Them

### Go Instinct
```go
// Build your own middleware, validation, config loading...
func ValidateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return errors.New("invalid email")
    }
    return nil
}
```

### Bad Java (Go mentality)
```java
public class Validator {
    public static void validateEmail(String email) {
        if (!email.contains("@")) {
            throw new IllegalArgumentException("invalid email");
        }
    }
}

// Called manually everywhere
public User createUser(CreateUserRequest req) {
    Validator.validateEmail(req.email());
    Validator.validateName(req.name());
    // ...
}
```

### Good Java
```java
public record CreateUserRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 2) String name
) {}

@PostMapping
public User createUser(@Valid @RequestBody CreateUserRequest req) {
    // Validation already happened. Just use the data.
}
```

**Why:** Spring (and Jakarta Bean Validation) has battle-tested implementations. Your hand-rolled version will be less complete and less maintainable.

## 4. Embrace Interfaces for Testability

### Go Instinct
```go
// Interfaces are implicit, define them where you need them
type UserFinder interface {
    FindByID(id string) (*User, error)
}

// Concrete type just implements the methods
type UserRepo struct { db *sql.DB }
func (r *UserRepo) FindByID(id string) (*User, error) { ... }
```

### Bad Java (Go mentality)
```java
// No interfaces, just concrete classes
public class UserRepository {
    public User findById(String id) { ... }
}

public class UserService {
    private final UserRepository repo;  // Concrete type
    
    // Hard to test without a real database
}
```

### Good Java
```java
public interface UserRepository extends JpaRepository<User, String> {
    // Spring Data JPA implements this automatically
}

public class UserService {
    private final UserRepository repo;  // Interface type
    
    // Easy to mock in tests
}

// Test
@MockBean
UserRepository mockRepo;  // Spring injects a mock
```

**Why:** Java's interfaces enable easy mocking, multiple implementations, and Spring's proxy-based features (transactions, caching, etc.).

## 5. Don't Fear Annotations

### Go Instinct
```go
// Explicit is better. I can read main() and see what happens.
```

### Resistance in Java
```java
// "What does @Transactional even do? Where's the code?"
// "This is magic, I can't trust it"
```

### Embrace It
```java
@Transactional  // Wraps method in DB transaction, rolls back on exception
public void transferMoney(Account from, Account to, BigDecimal amount) {
    from.debit(amount);
    to.credit(amount);
    // If either fails, both are rolled back
}
```

**Why:** Annotations are declarative configuration, not magic. Learn what the common ones do:
- `@Transactional` — DB transaction boundary
- `@Async` — Run in separate thread
- `@Cacheable` — Cache return value
- `@Scheduled` — Run periodically
- `@Retryable` — Retry on failure

They're implemented via AOP proxies. Understanding the mechanism removes the "magic" feeling.

## 6. Use Lombok (or Records) — Don't Fight Boilerplate with Willpower

### Go Instinct
```go
type User struct {
    ID    string
    Email string
}
// That's it. No getters, setters, constructors, equals, hashCode...
```

### Bad Java (Stubborn)
```java
// "I'll just write the boilerplate, it's fine"
public class User {
    private String id;
    private String email;
    
    // 50 lines of getters, setters, equals, hashCode, toString...
}
```

### Good Java (Accept help)
```java
// Option 1: Records (for immutable data)
public record User(String id, String email) {}

// Option 2: Lombok (for mutable entities)
@Data
@Builder
public class User {
    private String id;
    private String email;
}
```

**Why:** Fighting Java's verbosity by writing everything manually leads to errors and tedium. Use the tools that exist.

## 7. Collections Are Interfaces

### Go Instinct
```go
users := []User{}  // Slice is a slice
users := make(map[string]User)  // Map is a map
```

### Bad Java (Concrete types)
```java
ArrayList<User> users = new ArrayList<>();
HashMap<String, User> userMap = new HashMap<>();
```

### Good Java (Interface types)
```java
List<User> users = new ArrayList<>();
Map<String, User> userMap = new HashMap<>();
```

**Why:** Programming to interfaces allows changing implementations without changing signatures. `List` could be `ArrayList`, `LinkedList`, or an immutable wrapper.

## 8. Streams for Collections, Not Loops

### Go Instinct
```go
var activeEmails []string
for _, user := range users {
    if user.Active {
        activeEmails = append(activeEmails, user.Email)
    }
}
```

### Bad Java (Go mentality)
```java
List<String> activeEmails = new ArrayList<>();
for (User user : users) {
    if (user.isActive()) {
        activeEmails.add(user.getEmail());
    }
}
```

### Good Java (Streams)
```java
List<String> activeEmails = users.stream()
    .filter(User::isActive)
    .map(User::getEmail)
    .toList();
```

**Why:** Streams are idiomatic Java. They're declarative, composable, and can be parallelized trivially. Go doesn't have this because Go optimizes for different things.

## 9. Configuration Is External

### Go Instinct
```go
// Config struct, loaded from env/file in main()
type Config struct {
    Port    int    `env:"PORT"`
    DBUrl   string `env:"DATABASE_URL"`
}
```

### Bad Java (Go mentality)
```java
public class Config {
    public static final int PORT = Integer.parseInt(System.getenv("PORT"));
    public static final String DB_URL = System.getenv("DATABASE_URL");
}
```

### Good Java (Spring way)
```yaml
# application.yml
server:
  port: ${PORT:8080}

database:
  url: ${DATABASE_URL}
```

```java
@ConfigurationProperties(prefix = "database")
public record DatabaseConfig(String url) {}

@Service
public class MyService {
    private final DatabaseConfig config;  // Injected
}
```

**Why:** Spring's configuration system handles profiles (dev/prod), property sources, validation, and refresh without restart. Rolling your own loses all of this.

## 10. Tests Use Spring's Testing Support

### Go Instinct
```go
func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/users", nil)
    w := httptest.NewRecorder()
    handler(w, req)
    // assert...
}
```

### Bad Java (Go mentality)
```java
@Test
void testController() {
    var controller = new UserController(new UserService(new UserRepository()));
    var result = controller.getUser("123");
    // Manual wiring, no HTTP layer testing
}
```

### Good Java
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService service;
    
    @Test
    void getUser_returnsUser() throws Exception {
        when(service.findById("123")).thenReturn(new User("123", "test@test.com"));
        
        mockMvc.perform(get("/users/123"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("test@test.com"));
    }
}
```

**Why:** Spring's test slices (`@WebMvcTest`, `@DataJpaTest`) start only what you need, handle DI, and provide powerful assertions. Manual wiring in tests defeats the purpose.

---

## The Mindset Shift

**Phase 1 (Understanding):** "How would I do this in Go? What's the Java equivalent?"

**Phase 2 (Fluency):** "What's the idiomatic Java way to solve this?"

**Phase 3 (Mastery):** You stop thinking about Go at all when writing Java.

The goal isn't to like Java more than Go. It's to write good Java when you need to write Java.
