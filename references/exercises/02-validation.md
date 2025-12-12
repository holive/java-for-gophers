# Exercise 02: Request Validation

**Time:** 15 minutes  
**Goal:** Add input validation to a POST endpoint

## The Go Version

```go
type CreateUserRequest struct {
    Email string `json:"email"`
    Name  string `json:"name"`
}

func CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid json", http.StatusBadRequest)
        return
    }
    
    // Manual validation
    if req.Email == "" {
        http.Error(w, "email is required", http.StatusBadRequest)
        return
    }
    if !strings.Contains(req.Email, "@") {
        http.Error(w, "invalid email", http.StatusBadRequest)
        return
    }
    if len(req.Name) < 2 {
        http.Error(w, "name too short", http.StatusBadRequest)
        return
    }
    
    // ... create user
}
```

## Your Task

Create a POST `/users` endpoint that:
1. Accepts JSON body with `email` and `name`
2. Validates email is not blank and is valid format
3. Validates name is at least 2 characters
4. Returns 400 with error details if validation fails
5. Returns 201 with the created user if valid

## Step by Step

### 1. Add Validation Dependency (2 min)

In `build.gradle`:
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

### 2. Create Request DTO (3 min)

Create `CreateUserRequest.java`:

```java
package com.example.demo;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record CreateUserRequest(
    @NotBlank(message = "email is required")
    @Email(message = "invalid email format")
    String email,
    
    @NotBlank(message = "name is required")
    @Size(min = 2, message = "name must be at least 2 characters")
    String name
) {}
```

### 3. Create Response DTO (1 min)

Create `UserResponse.java`:

```java
package com.example.demo;

public record UserResponse(String id, String email, String name) {}
```

### 4. Create Controller (5 min)

Create `UserController.java`:

```java
package com.example.demo;

import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import java.util.UUID;

@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        // In real app, save to database
        String id = UUID.randomUUID().toString();
        return new UserResponse(id, request.email(), request.name());
    }
}
```

### 5. Handle Validation Errors (4 min)

Create `GlobalExceptionHandler.java`:

```java
package com.example.demo;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                e -> e.getField(),
                e -> e.getDefaultMessage(),
                (a, b) -> a  // Keep first if duplicate
            ));
        
        return Map.of(
            "error", "Validation failed",
            "details", errors
        );
    }
}
```

## Test It

**Valid request:**
```bash
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"email": "hugo@example.com", "name": "Hugo"}'
```

Response (201):
```json
{"id": "abc-123", "email": "hugo@example.com", "name": "Hugo"}
```

**Invalid request:**
```bash
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"email": "not-an-email", "name": "H"}'
```

Response (400):
```json
{
  "error": "Validation failed",
  "details": {
    "email": "invalid email format",
    "name": "name must be at least 2 characters"
  }
}
```

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| Manual if/else validation | Annotations on fields |
| Custom error response building | `@RestControllerAdvice` handles globally |
| Check each field explicitly | `@Valid` triggers all validations |

**The trade-off:** More annotation magic, but way less boilerplate for common validations.

## Common Validation Annotations

```java
@NotNull          // != null
@NotBlank         // != null && not empty && not whitespace
@NotEmpty         // != null && not empty (for collections/strings)
@Size(min=, max=) // length/size constraints
@Min(value)       // minimum number value
@Max(value)       // maximum number value
@Email            // valid email format
@Pattern(regexp)  // matches regex
@Past             // date in the past
@Future           // date in the future
```

## Checkpoint

✅ POST /users returns 201 with valid input  
✅ POST /users returns 400 with validation errors  
✅ Error response includes field-specific messages  

**Next:** Exercise 03 — Middleware and filters
