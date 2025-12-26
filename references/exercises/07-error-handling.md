# Exercise 07: Error Handling

**Time:** 15 minutes  
**Goal:** Create custom exceptions and centralized error handling

## The Go Version

```go
// Custom error types
type NotFoundError struct {
    Resource string
    ID       string
}

func (e NotFoundError) Error() string {
    return fmt.Sprintf("%s with id %s not found", e.Resource, e.ID)
}

// In handler
func GetTask(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    task, err := taskService.Find(id)
    
    if err != nil {
        var notFound NotFoundError
        if errors.As(err, &notFound) {
            http.Error(w, err.Error(), http.StatusNotFound)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(task)
}
```

## Your Task

1. Create custom exception classes
2. Throw exceptions from the service layer
3. Handle them globally with `@RestControllerAdvice`
4. Return consistent error responses

## Try First (Optional)

Before looking at the solution:
- Create a class that `extends RuntimeException`
- Add `@ResponseStatus` to map it to an HTTP status
- Create a `@RestControllerAdvice` class with `@ExceptionHandler` methods

---

## Step by Step

### 1. Create Custom Exceptions (4 min)

Create `exceptions/TaskNotFoundException.java`:

```java
package com.example.demo.exceptions;

public class TaskNotFoundException extends RuntimeException {
    
    private final String taskId;
    
    public TaskNotFoundException(String taskId) {
        super("Task not found: " + taskId);
        this.taskId = taskId;
    }
    
    public String getTaskId() {
        return taskId;
    }
}
```

Create `exceptions/InvalidTaskStatusException.java`:

```java
package com.example.demo.exceptions;

public class InvalidTaskStatusException extends RuntimeException {
    
    public InvalidTaskStatusException(String status) {
        super("Invalid task status: " + status + ". Valid values: pending, in_progress, done");
    }
}
```

### 2. Create Error Response DTO (2 min)

Create `ErrorResponse.java`:

```java
package com.example.demo;

import java.time.Instant;

public record ErrorResponse(
    String error,
    String message,
    int status,
    Instant timestamp
) {
    public ErrorResponse(String error, String message, int status) {
        this(error, message, status, Instant.now());
    }
}
```

### 3. Create Global Exception Handler (5 min)

Create `GlobalExceptionHandler.java`:

```java
package com.example.demo;

import com.example.demo.exceptions.*;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(TaskNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(TaskNotFoundException ex) {
        return new ErrorResponse(
            "Not Found",
            ex.getMessage(),
            404
        );
    }

    @ExceptionHandler(InvalidTaskStatusException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleInvalidStatus(InvalidTaskStatusException ex) {
        return new ErrorResponse(
            "Bad Request",
            ex.getMessage(),
            400
        );
    }

    // Handle validation errors (from @Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                e -> e.getField(),
                e -> e.getDefaultMessage(),
                (a, b) -> a
            ));
        
        return Map.of(
            "error", "Validation Failed",
            "status", 400,
            "details", errors
        );
    }

    // Catch-all for unexpected errors
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex) {
        // Log the actual error
        ex.printStackTrace();  // Use proper logging in production
        
        return new ErrorResponse(
            "Internal Server Error",
            "An unexpected error occurred",
            500
        );
    }
}
```

### 4. Update Service to Throw Exceptions (4 min)

Update `TaskService.java`:

```java
package com.example.demo;

import com.example.demo.exceptions.*;
import org.springframework.stereotype.Service;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class TaskService {
    
    private static final Set<String> VALID_STATUSES = 
        Set.of("pending", "in_progress", "done");
    
    private final Map<String, TaskResponse> tasks = new ConcurrentHashMap<>();

    public TaskResponse findById(String id) {
        var task = tasks.get(id);
        if (task == null) {
            throw new TaskNotFoundException(id);
        }
        return task;
    }

    public TaskResponse create(TaskRequest request) {
        validateStatus(request.status());
        
        String id = UUID.randomUUID().toString();
        var task = new TaskResponse(
            id,
            request.title(),
            request.description(),
            request.status() != null ? request.status() : "pending"
        );
        tasks.put(id, task);
        return task;
    }

    public TaskResponse update(String id, TaskRequest request) {
        if (!tasks.containsKey(id)) {
            throw new TaskNotFoundException(id);
        }
        validateStatus(request.status());
        
        var updated = new TaskResponse(
            id,
            request.title(),
            request.description(),
            request.status()
        );
        tasks.put(id, updated);
        return updated;
    }

    public void delete(String id) {
        if (tasks.remove(id) == null) {
            throw new TaskNotFoundException(id);
        }
    }
    
    private void validateStatus(String status) {
        if (status != null && !VALID_STATUSES.contains(status)) {
            throw new InvalidTaskStatusException(status);
        }
    }
}
```

### 5. Simplify the Controller

Now the controller can be cleaner - no more `Optional` handling:

```java
@GetMapping("/{id}")
public TaskResponse getTask(@PathVariable String id) {
    return taskService.findById(id);  // Throws if not found
}

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void deleteTask(@PathVariable String id) {
    taskService.delete(id);  // Throws if not found
}
```

## Test It

```bash
# Not found
curl http://localhost:8080/tasks/nonexistent
# {"error":"Not Found","message":"Task not found: nonexistent","status":404,...}

# Invalid status
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Test", "status": "invalid"}'
# {"error":"Bad Request","message":"Invalid task status: invalid...","status":400,...}
```

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| `errors.New()` / custom error type | `extends RuntimeException` |
| `errors.As()` to check type | `@ExceptionHandler(Type.class)` |
| Handle errors at each call site | Handle globally with `@RestControllerAdvice` |
| Return `(result, error)` | Throw exception, catch at boundary |
| `http.Error(w, msg, status)` | `@ResponseStatus` + return body |

## Why Exceptions Feel Wrong (And Why They're Right Here)

**Your Go instinct:** "Exceptions are for exceptional cases! This is just 'not found'."

**The reframe:** In Java/Spring, exceptions are the idiomatic way to signal "this operation cannot complete as requested." The key insight:

1. **Caller doesn't need to check** - No `if err != nil` everywhere
2. **Stack trace preserved** - You know exactly where it originated
3. **Centralized handling** - One place to format all error responses
4. **Clean happy path** - Service and controller code reads linearly

## Exception Hierarchy

```
RuntimeException (unchecked - don't need to declare)
├── TaskNotFoundException
├── InvalidTaskStatusException
└── (your domain exceptions)

Exception (checked - must declare or catch)
├── IOException
└── SQLException
```

**Rule:** Extend `RuntimeException` for your domain exceptions. Checked exceptions add ceremony without benefit in most web apps.

## Alternative: @ResponseStatus on Exception

For simple cases, skip the handler:

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class TaskNotFoundException extends RuntimeException {
    public TaskNotFoundException(String id) {
        super("Task not found: " + id);
    }
}
```

Spring will automatically return 404 when this is thrown. But you lose control over the response body format.

## Checkpoint

- [ ] Custom exception classes exist
- [ ] Service throws exceptions instead of returning Optional
- [ ] `@RestControllerAdvice` catches and formats errors
- [ ] Consistent JSON error responses

**Next:** Exercise 08 - Testing Controllers with MockMvc
