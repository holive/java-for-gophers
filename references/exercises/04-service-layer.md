# Exercise 04: Service Layer

**Time:** 15 minutes  
**Goal:** Extract business logic from controller into a service class

## The Go Version

```go
// In Go, you structure this with explicit dependency passing:
type UserService struct {
    repo *UserRepository
}

func NewUserService(repo *UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) CreateUser(email, name string) (*User, error) {
    user := &User{
        ID:    uuid.New().String(),
        Email: email,
        Name:  name,
    }
    return s.repo.Save(user)
}

// In main.go - you wire everything explicitly
func main() {
    db := database.Connect()
    repo := repository.NewUserRepo(db)
    service := service.NewUserService(repo)
    handler := handler.NewUserHandler(service)
    
    http.Handle("/users", handler)
}
```

## Your Task

Refactor the user creation from Exercise 02:
1. Create a `UserService` class with `@Service`
2. Move user creation logic from controller to service
3. Inject the service into the controller via constructor
4. Keep the controller thin - only HTTP concerns

## Try First (Optional)

Before looking at the solution, try writing:
- A class annotated with `@Service`
- A constructor that takes dependencies (no `@Autowired` needed on single constructor)
- Update the controller to use constructor injection

Skip this if you prefer to learn by following along.

---

## Step by Step

### 1. Create the Service (5 min)

Create `UserService.java`:

```java
package com.example.demo;

import org.springframework.stereotype.Service;
import java.util.UUID;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class UserService {
    
    // In-memory storage for now (we'll add a real database later)
    private final Map<String, UserResponse> users = new ConcurrentHashMap<>();

    public UserResponse createUser(CreateUserRequest request) {
        String id = UUID.randomUUID().toString();
        
        // Business logic lives here
        // In a real app: check for duplicates, hash passwords, etc.
        
        var user = new UserResponse(id, request.email(), request.name());
        users.put(id, user);
        return user;
    }
    
    public UserResponse getUser(String id) {
        return users.get(id);
    }
}
```

### 2. Update the Controller (5 min)

Modify `UserController.java`:

```java
package com.example.demo;

import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    // Constructor injection - Spring automatically provides UserService
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        // Controller is thin - just delegates to service
        return userService.createUser(request);
    }
    
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable String id) {
        return userService.getUser(id);
    }
}
```

### 3. Test It (5 min)

```bash
# Create a user
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"email": "hugo@example.com", "name": "Hugo"}'

# Note the ID in the response, then fetch:
curl http://localhost:8080/users/{id}
```

## What Just Happened?

Spring automatically:
1. Found `UserService` because of `@Service` annotation
2. Created a single instance (singleton by default)
3. Injected it into `UserController`'s constructor
4. Managed the lifecycle for you

You didn't write any wiring code. No `main()` with explicit construction.

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| `NewUserService(repo)` in main | `@Service` + constructor |
| Explicit wiring in main.go | Auto-wiring via component scan |
| You control object lifecycle | Spring manages lifecycle |
| Pass dependencies explicitly | Declare dependencies in constructor |

## Why This Feels Wrong (And Why It's OK)

**Your Go instinct:** "Where's the wiring? How do I know what gets injected?"

**The answer:** Spring scans for `@Component`, `@Service`, `@Repository`, `@Controller` classes and builds a dependency graph. When it creates `UserController`, it sees the constructor needs `UserService`, finds the `@Service` class, and injects it.

**To see what Spring wired:**
```java
// Add to your main application class
@Bean
CommandLineRunner printBeans(ApplicationContext ctx) {
    return args -> {
        Arrays.stream(ctx.getBeanDefinitionNames())
            .filter(name -> name.contains("user") || name.contains("User"))
            .forEach(System.out::println);
    };
}
```

## The Stereotype Annotations

```java
@Component   // Generic Spring-managed bean
@Service     // Business logic (same as @Component, semantic meaning)
@Repository  // Data access (same as @Component + exception translation)
@Controller  // Web controller (same as @Component + request mapping)
```

They're all `@Component` under the hood. The different names are for clarity and some extra features (like `@Repository`'s exception translation).

## Common Mistake: Manual Instantiation

```java
// WRONG - bypasses Spring, no injection
@RestController
public class UserController {
    private final UserService userService = new UserService();
}

// RIGHT - let Spring inject
@RestController
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

## Checkpoint

- [ ] `UserService` exists with `@Service` annotation
- [ ] Controller uses constructor injection (no `new UserService()`)
- [ ] POST and GET endpoints still work

**Next:** Exercise 05 - Configuration with @Value and profiles
