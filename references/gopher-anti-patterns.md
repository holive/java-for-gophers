# Gopher Anti-Patterns

Code a Go developer would write in Java — and the idiomatic fix.

Each pattern shows: Go code -> Bad Java -> Why it's wrong -> Idiomatic Java

---

## 1. Error Tuple Simulation

**Your Go instinct:**
```go
func FindUser(id string) (*User, error) {
    user, err := db.Get(id)
    if err != nil {
        return nil, fmt.Errorf("user not found: %w", err)
    }
    return user, nil
}

// caller
user, err := FindUser(id)
if err != nil {
    return nil, err
}
```

**Bad Java (fighting the language):**
```java
// creating Result<T, E> wrapper to simulate Go
record Result<T>(T value, String error) {
    boolean isError() { return error != null; }
}

public Result<User> findUser(String id) {
    try {
        return new Result<>(db.get(id), null);
    } catch (Exception e) {
        return new Result<>(null, "user not found: " + e.getMessage());
    }
}

// caller - feels like Go but is awful Java
var result = findUser(id);
if (result.isError()) {
    return new Result<>(null, result.error());
}
```

**Why it's wrong:** You're writing 3x the code to avoid a language feature. Every caller must check `.isError()`. You lose stack traces. It doesn't compose.

**Idiomatic Java:**
```java
public User findUser(String id) {
    return db.get(id)  // throws if not found
        .orElseThrow(() -> new UserNotFoundException(id));
}

// caller - no error handling here, it bubbles up
User user = findUser(id);

// handle at the boundary (controller advice)
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorResponse(e.getMessage()));
    }
}
```

---

## 2. Manual Wiring

**Your Go instinct:**
```go
func main() {
    db := postgres.New(cfg.DatabaseURL)
    cache := redis.New(cfg.RedisURL)
    userRepo := repository.NewUserRepo(db)
    userService := service.NewUserService(userRepo, cache)
    handler := handler.NewUserHandler(userService)

    http.Handle("/users", handler)
}
```

**Bad Java (explicit construction):**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // "I don't trust Spring to wire things"
        var db = new PostgresConnection(System.getenv("DB_URL"));
        var cache = new RedisClient(System.getenv("REDIS_URL"));
        var userRepo = new UserRepository(db);
        var userService = new UserService(userRepo, cache);
        var userController = new UserController(userService);

        // now what? manually register routes?
    }
}
```

**Why it's wrong:** You're fighting the framework. No lifecycle management. No proxies (so @Transactional won't work). No testing support.

**Idiomatic Java:**
```java
@Repository
public class UserRepository {
    // Spring injects the JPA EntityManager automatically
}

@Service
public class UserService {
    private final UserRepository userRepo;
    private final RedisTemplate<String, User> cache;

    // constructor injection - Spring wires this
    public UserService(UserRepository userRepo, RedisTemplate<String, User> cache) {
        this.userRepo = userRepo;
        this.cache = cache;
    }
}

@RestController
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
// just define the beans. Spring wires the graph.
```

---

## 3. Avoiding Interfaces

**Your Go instinct:**
```go
// implicit interface - any struct with this method satisfies it
type UserStore interface {
    GetUser(id string) (*User, error)
}

// no "implements" declaration needed
type PostgresStore struct { db *sql.DB }
func (s *PostgresStore) GetUser(id string) (*User, error) { ... }
```

**Bad Java (concrete types everywhere):**
```java
@Service
public class UserService {
    // directly depends on concrete implementation
    private final PostgresUserRepository repo;

    public UserService(PostgresUserRepository repo) {
        this.repo = repo;
    }
}

// testing is painful - can't easily mock PostgresUserRepository
```

**Why it's wrong:** Tight coupling. Testing requires real database or complex reflection mocking. Can't swap implementations.

**Idiomatic Java:**
```java
// explicit interface
public interface UserRepository {
    Optional<User> findById(String id);
    User save(User user);
}

@Repository
public class PostgresUserRepository implements UserRepository {
    @Override
    public Optional<User> findById(String id) { ... }

    @Override
    public User save(User user) { ... }
}

@Service
public class UserService {
    private final UserRepository repo;  // depends on interface

    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}

// testing is trivial
@Test
void testUserService() {
    UserRepository mockRepo = mock(UserRepository.class);
    when(mockRepo.findById("123")).thenReturn(Optional.of(testUser));

    var service = new UserService(mockRepo);
    // test without database
}
```

---

## 4. Defensive Nil Checks

**Your Go instinct:**
```go
func ProcessUser(user *User) error {
    if user == nil {
        return errors.New("user is nil")
    }
    if user.Email == "" {
        return errors.New("email is empty")
    }
    // finally do the work
}
```

**Bad Java (null paranoia):**
```java
public void processUser(User user) {
    if (user == null) {
        throw new IllegalArgumentException("user is null");
    }
    if (user.getEmail() == null) {
        throw new IllegalArgumentException("email is null");
    }
    if (user.getEmail().isEmpty()) {
        throw new IllegalArgumentException("email is empty");
    }
    if (user.getName() == null) {
        // ... 50 more null checks
    }
}
```

**Why it's wrong:** Defensive programming everywhere is exhausting. The real question is: why is null reaching this code at all?

**Idiomatic Java:**
```java
// option 1: validate at the boundary, trust internally
public record CreateUserRequest(
    @NotNull @Email String email,
    @NotNull @Size(min = 1) String name
) {}

@PostMapping("/users")
public User createUser(@Valid @RequestBody CreateUserRequest request) {
    // if we get here, request is already validated
    return userService.create(request.email(), request.name());
}

// option 2: use Optional for "might not exist"
public Optional<User> findUser(String id) {
    return repository.findById(id);  // Optional, not null
}

// option 3: fail fast with Objects.requireNonNull for internal APIs
public void processUser(User user) {
    Objects.requireNonNull(user, "user must not be null");
    // one line, clear intent
}
```

---

## 5. Fighting Exceptions

**Your Go instinct:**
```go
func DoWork() error {
    if err := step1(); err != nil {
        return err  // explicit propagation
    }
    if err := step2(); err != nil {
        return err
    }
    return nil
}
```

**Bad Java (catching everything):**
```java
public void doWork() {
    try {
        step1();
    } catch (Exception e) {
        // swallow it? log it? return null?
        logger.error("step1 failed", e);
        // continue anyway???
    }
    try {
        step2();
    } catch (Exception e) {
        logger.error("step2 failed", e);
    }
}
```

**Why it's wrong:** You've lost the error. Execution continues in unknown state. No stack trace reaches the caller.

**Idiomatic Java:**
```java
// just let it throw
public void doWork() {
    step1();  // throws? bubbles up
    step2();  // throws? bubbles up
}

// if you need cleanup, use try-with-resources
public void doWorkWithCleanup() {
    try (var resource = acquireResource()) {
        step1();
        step2();
    }  // resource auto-closed even if exception thrown
}

// handle at service boundary, not at every call site
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception e) {
        logger.error("request failed", e);
        return ResponseEntity.status(500).body(new ErrorResponse("Internal error"));
    }
}
```

---

## 6. Over-Minimizing

**Your Go instinct:**
```go
type User struct {
    ID    string
    Name  string
    Email string
}
// that's it. clean. minimal.
```

**Bad Java (avoiding "magic"):**
```java
public class User {
    private String id;
    private String name;
    private String email;

    public User() {}

    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    @Override
    public boolean equals(Object o) { /* 10 lines */ }

    @Override
    public int hashCode() { /* 5 lines */ }

    @Override
    public String toString() { /* 5 lines */ }
}
// 50 lines for a data holder
```

**Why it's wrong:** Boilerplate obscures intent. Easy to forget equals/hashCode. Mutable by default (setters everywhere).

**Idiomatic Java:**
```java
// Java 17+ record - immutable, with equals/hashCode/toString
public record User(String id, String name, String email) {}
// done. 1 line.

// or with Lombok if you need mutability
@Data
@Builder
public class User {
    private String id;
    private String name;
    private String email;
}
// generates getters, setters, equals, hashCode, toString, builder
```

---

## 7. Explicit Threading

**Your Go instinct:**
```go
go func() {
    sendEmail(user)
}()
// fire and forget, beautiful
```

**Bad Java (raw threads):**
```java
new Thread(() -> {
    sendEmail(user);
}).start();
// no error handling, thread management, or lifecycle control
```

**Why it's wrong:** Threads are expensive. No backpressure. Exceptions vanish. No integration with Spring's context.

**Idiomatic Java:**
```java
// option 1: @Async (Spring-managed)
@Service
public class EmailService {
    @Async
    public void sendEmail(User user) {
        // runs in Spring's thread pool
        // exceptions logged automatically
    }
}

// option 2: CompletableFuture for composition
CompletableFuture.runAsync(() -> sendEmail(user))
    .exceptionally(e -> {
        logger.error("email failed", e);
        return null;
    });

// option 3: virtual threads (Java 21+) - closest to goroutines
Thread.startVirtualThread(() -> sendEmail(user));
// lightweight, like goroutines, but still prefer @Async in Spring
```

---

## 8. Reinventing Framework Features

**Your Go instinct:**
```go
func CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid json", 400)
        return
    }
    if req.Email == "" {
        http.Error(w, "email required", 400)
        return
    }
    if !isValidEmail(req.Email) {
        http.Error(w, "invalid email format", 400)
        return
    }
    // ... more validation
}
```

**Bad Java (manual everything):**
```java
@PostMapping("/users")
public ResponseEntity<?> createUser(@RequestBody String body) {
    CreateUserRequest req;
    try {
        req = objectMapper.readValue(body, CreateUserRequest.class);
    } catch (JsonProcessingException e) {
        return ResponseEntity.badRequest().body("invalid json");
    }

    if (req.email() == null || req.email().isEmpty()) {
        return ResponseEntity.badRequest().body("email required");
    }
    if (!req.email().matches("^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$")) {
        return ResponseEntity.badRequest().body("invalid email");
    }
    // ... 50 more lines of validation
}
```

**Why it's wrong:** You're rebuilding what Spring already does better. No consistent error format. Validation logic scattered.

**Idiomatic Java:**
```java
public record CreateUserRequest(
    @NotNull(message = "email is required")
    @Email(message = "invalid email format")
    String email,

    @NotBlank(message = "name is required")
    @Size(min = 2, max = 100, message = "name must be 2-100 chars")
    String name
) {}

@PostMapping("/users")
public User createUser(@Valid @RequestBody CreateUserRequest request) {
    // if we reach here, request is valid
    // Spring auto-returns 400 with validation errors if invalid
    return userService.create(request);
}

// customize error format once, globally
@RestControllerAdvice
public class ValidationHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException e) {
        var errors = e.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
        return ResponseEntity.badRequest().body(errors);
    }
}
```

---

## The Pattern

Notice the common thread:

| Go Strength | Java Equivalent |
|-------------|-----------------|
| Explicit control | Declarative annotations |
| Minimal abstraction | Framework conventions |
| DIY | Let the framework do it |

The instinct to control everything made you a good Go developer. In Java, that instinct produces worse code. Trust the framework.
