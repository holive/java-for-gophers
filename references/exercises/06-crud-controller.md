# Exercise 06: CRUD Controller

**Time:** 15 minutes  
**Goal:** Build a complete REST API with all HTTP methods

## The Go Version

```go
func main() {
    r := chi.NewRouter()
    
    r.Route("/tasks", func(r chi.Router) {
        r.Get("/", listTasks)           // GET /tasks
        r.Post("/", createTask)         // POST /tasks
        r.Get("/{id}", getTask)         // GET /tasks/123
        r.Put("/{id}", updateTask)      // PUT /tasks/123
        r.Delete("/{id}", deleteTask)   // DELETE /tasks/123
    })
    
    // Query params
    r.Get("/tasks/search", func(w http.ResponseWriter, r *http.Request) {
        status := r.URL.Query().Get("status")  // ?status=pending
        // ...
    })
}

func getTask(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    // ...
}
```

## Your Task

Build a Task API with:
1. `GET /tasks` - List all tasks (with optional `?status=` filter)
2. `POST /tasks` - Create a task
3. `GET /tasks/{id}` - Get one task
4. `PUT /tasks/{id}` - Update a task
5. `DELETE /tasks/{id}` - Delete a task

## Try First (Optional)

Before looking at the solution, try:
- `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
- `@PathVariable` for URL parameters
- `@RequestParam` for query parameters

---

## Step by Step

### 1. Create the DTOs (3 min)

```java
package com.example.demo;

// Request for creating/updating
public record TaskRequest(
    String title,
    String description,
    String status  // "pending", "in_progress", "done"
) {}

// Response
public record TaskResponse(
    String id,
    String title,
    String description,
    String status
) {}
```

### 2. Create the Service (4 min)

```java
package com.example.demo;

import org.springframework.stereotype.Service;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class TaskService {
    
    private final Map<String, TaskResponse> tasks = new ConcurrentHashMap<>();

    public List<TaskResponse> findAll(String status) {
        return tasks.values().stream()
            .filter(t -> status == null || t.status().equals(status))
            .toList();
    }

    public Optional<TaskResponse> findById(String id) {
        return Optional.ofNullable(tasks.get(id));
    }

    public TaskResponse create(TaskRequest request) {
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

    public Optional<TaskResponse> update(String id, TaskRequest request) {
        if (!tasks.containsKey(id)) {
            return Optional.empty();
        }
        var updated = new TaskResponse(
            id,
            request.title(),
            request.description(),
            request.status()
        );
        tasks.put(id, updated);
        return Optional.of(updated);
    }

    public boolean delete(String id) {
        return tasks.remove(id) != null;
    }
}
```

### 3. Create the Controller (8 min)

```java
package com.example.demo;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/tasks")
public class TaskController {

    private final TaskService taskService;

    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }

    // GET /tasks or GET /tasks?status=pending
    @GetMapping
    public List<TaskResponse> listTasks(
            @RequestParam(required = false) String status) {
        return taskService.findAll(status);
    }

    // GET /tasks/{id}
    @GetMapping("/{id}")
    public ResponseEntity<TaskResponse> getTask(@PathVariable String id) {
        return taskService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST /tasks
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public TaskResponse createTask(@RequestBody TaskRequest request) {
        return taskService.create(request);
    }

    // PUT /tasks/{id}
    @PutMapping("/{id}")
    public ResponseEntity<TaskResponse> updateTask(
            @PathVariable String id,
            @RequestBody TaskRequest request) {
        return taskService.update(id, request)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // DELETE /tasks/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable String id) {
        if (taskService.delete(id)) {
            return ResponseEntity.noContent().build();  // 204
        }
        return ResponseEntity.notFound().build();  // 404
    }
}
```

## Test It

```bash
# Create
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Spring", "description": "Complete exercises"}'

# List all
curl http://localhost:8080/tasks

# List filtered
curl "http://localhost:8080/tasks?status=pending"

# Get one (use ID from create response)
curl http://localhost:8080/tasks/{id}

# Update
curl -X PUT http://localhost:8080/tasks/{id} \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Spring", "description": "Done!", "status": "done"}'

# Delete
curl -X DELETE http://localhost:8080/tasks/{id}
```

## Go vs Spring Comparison

| Go (chi) | Spring Boot |
|----------|-------------|
| `r.Get("/", handler)` | `@GetMapping` |
| `r.Post("/", handler)` | `@PostMapping` |
| `r.Put("/{id}", handler)` | `@PutMapping("/{id}")` |
| `r.Delete("/{id}", handler)` | `@DeleteMapping("/{id}")` |
| `chi.URLParam(r, "id")` | `@PathVariable String id` |
| `r.URL.Query().Get("status")` | `@RequestParam String status` |
| `http.Error(w, msg, 404)` | `ResponseEntity.notFound()` |

## Annotations Reference

### Path Variables

```java
// Single
@GetMapping("/{id}")
public Task getTask(@PathVariable String id) { }

// Multiple
@GetMapping("/{userId}/tasks/{taskId}")
public Task getTask(
    @PathVariable String userId,
    @PathVariable String taskId) { }

// Renamed
@GetMapping("/{task_id}")
public Task getTask(@PathVariable("task_id") String taskId) { }
```

### Query Parameters

```java
// Optional
@GetMapping
public List<Task> list(@RequestParam(required = false) String status) { }

// Required (default)
@GetMapping
public List<Task> list(@RequestParam String status) { }

// With default value
@GetMapping
public List<Task> list(@RequestParam(defaultValue = "pending") String status) { }

// Multiple
@GetMapping
public List<Task> list(
    @RequestParam(required = false) String status,
    @RequestParam(defaultValue = "10") int limit) { }
```

### Response Status Codes

```java
// Fixed status
@PostMapping
@ResponseStatus(HttpStatus.CREATED)  // Always 201
public Task create(...) { }

// Dynamic status
@GetMapping("/{id}")
public ResponseEntity<Task> get(@PathVariable String id) {
    return service.find(id)
        .map(task -> ResponseEntity.ok(task))           // 200
        .orElse(ResponseEntity.notFound().build());     // 404
}

// Common ResponseEntity patterns
ResponseEntity.ok(body)                    // 200 with body
ResponseEntity.created(uri).body(body)     // 201 with Location header
ResponseEntity.noContent().build()         // 204 no body
ResponseEntity.notFound().build()          // 404
ResponseEntity.badRequest().body(error)    // 400
```

## Common Patterns

### Combining Validation

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public TaskResponse createTask(@Valid @RequestBody TaskRequest request) {
    return taskService.create(request);
}
```

### Pagination (Preview)

```java
@GetMapping
public Page<Task> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {
    return taskService.findAll(PageRequest.of(page, size));
}
```

## Checkpoint

- [ ] All 5 endpoints work (list, get, create, update, delete)
- [ ] Query param filtering works (`?status=pending`)
- [ ] Correct status codes (201 for create, 204 for delete, 404 for not found)

**Next:** Exercise 07 - Error Handling with custom exceptions
