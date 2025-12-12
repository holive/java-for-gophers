# Go to Spring Boot Concept Mapping

Quick reference for translating Go patterns to Spring Boot equivalents.

## HTTP Handling

### Basic Endpoint

**Go:**
```go
func GetUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")  // or mux.Vars(r)["id"]
    user := userService.FindByID(id)
    json.NewEncoder(w).Encode(user)
}

// Registration
r.Get("/users/{id}", GetUser)
```

**Spring Boot:**
```java
@RestController
@RequestMapping("/users")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable String id) {
        return userService.findById(id);
    }
}
```

Key differences:
- Annotations replace explicit routing
- Return value automatically serialized to JSON
- No manual response writing

### Request/Response Bodies

**Go:**
```go
func CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    // ...
}
```

**Spring Boot:**
```java
@PostMapping
public User createUser(@RequestBody @Valid CreateUserRequest req) {
    return userService.create(req);
}
```

Key differences:
- `@RequestBody` handles deserialization
- `@Valid` triggers validation automatically
- Exceptions handle errors (not explicit returns)

## Middleware → Filters/Interceptors

**Go middleware:**
```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("Request: %s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}
```

**Spring Boot Filter:**
```java
@Component
public class LoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) {
        log.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        chain.doFilter(request, response);
    }
}
```

Alternative: `HandlerInterceptor` for more Spring-integrated approach.

## Dependency Injection

**Go (manual):**
```go
func main() {
    db := database.Connect()
    userRepo := repository.NewUserRepo(db)
    userService := service.NewUserService(userRepo)
    handler := handler.NewUserHandler(userService)
}
```

**Spring Boot (automatic):**
```java
@Service
public class UserService {
    private final UserRepository userRepo;
    
    public UserService(UserRepository userRepo) {  // Auto-injected
        this.userRepo = userRepo;
    }
}
```

Spring scans `@Component`, `@Service`, `@Repository`, `@Controller` and wires dependencies automatically.

## Error Handling

**Go:**
```go
user, err := userService.FindByID(id)
if err != nil {
    if errors.Is(err, ErrNotFound) {
        http.Error(w, "not found", http.StatusNotFound)
        return
    }
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
}
```

**Spring Boot:**
```java
// In service
public User findById(String id) {
    return repo.findById(id)
        .orElseThrow(() -> new NotFoundException("User not found"));
}

// Global handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(NotFoundException ex) {
        return new ErrorResponse(ex.getMessage());
    }
}
```

This will feel wrong. Embrace it — exceptions are idiomatic Java.

## Concurrency

**Go goroutines:**
```go
go func() {
    sendEmail(user)
}()
```

**Spring Boot @Async:**
```java
@Async
public CompletableFuture<Void> sendEmail(User user) {
    // runs in separate thread
    return CompletableFuture.completedFuture(null);
}
```

Requires `@EnableAsync` on config class.

**Java 21 Virtual Threads** (closest to goroutines):
```java
Thread.startVirtualThread(() -> sendEmail(user));

// Or with executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> sendEmail(user));
}
```

## Configuration

**Go:**
```go
type Config struct {
    Port     int    `env:"PORT" envDefault:"8080"`
    DBUrl    string `env:"DATABASE_URL,required"`
}

// Load with envconfig or viper
```

**Spring Boot:**
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
```

## Structs → Records/Classes

**Go struct:**
```go
type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}
```

**Java record (immutable, like Go):**
```java
public record User(
    String id,
    String email,
    Instant createdAt
) {}
```

**Java class (mutable, with boilerplate):**
```java
public class User {
    private String id;
    private String email;
    private Instant createdAt;
    
    // getters, setters, constructor...
    // Or use Lombok: @Data, @Builder
}
```

Prefer records for DTOs. Use classes for entities.

## Testing

**Go:**
```go
func TestGetUser(t *testing.T) {
    req := httptest.NewRequest("GET", "/users/123", nil)
    w := httptest.NewRecorder()
    
    handler.GetUser(w, req)
    
    assert.Equal(t, 200, w.Code)
}
```

**Spring Boot:**
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    MockMvc mockMvc;
    
    @MockBean
    UserService userService;
    
    @Test
    void getUser_returnsUser() throws Exception {
        when(userService.findById("123")).thenReturn(new User("123", "test@example.com"));
        
        mockMvc.perform(get("/users/123"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
}
```

More setup, but more built-in assertions.
