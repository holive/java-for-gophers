# Exercise 10: Database Setup

**Time:** 15 minutes  
**Goal:** Add JPA persistence with H2 for development

## The Go Version

```go
type Task struct {
    ID          string    `db:"id"`
    Title       string    `db:"title"`
    Description string    `db:"description"`
    Status      string    `db:"status"`
    CreatedAt   time.Time `db:"created_at"`
}

func (r *TaskRepo) Save(ctx context.Context, task *Task) error {
    _, err := r.db.ExecContext(ctx, `
        INSERT INTO tasks (id, title, description, status, created_at)
        VALUES ($1, $2, $3, $4, $5)
    `, task.ID, task.Title, task.Description, task.Status, task.CreatedAt)
    return err
}
```

## Your Task

1. Add JPA and H2 dependencies
2. Create a Task entity with proper annotations
3. Create a JPA repository interface
4. Update the service to use JPA

## Try First (Optional)

Before looking at the solution:
- Add `spring-boot-starter-data-jpa` and `h2` dependencies
- Create a class with `@Entity` and `@Id` annotations
- Create an interface extending `JpaRepository<Task, String>`

---

## Step by Step

### 1. Add Dependencies (2 min)

In `build.gradle`:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'
}
```

### 2. Configure H2 (2 min)

In `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  
  jpa:
    hibernate:
      ddl-auto: create-drop  # Auto-create tables
    show-sql: true           # Log SQL (helpful for learning)
  
  h2:
    console:
      enabled: true          # Access at /h2-console
```

### 3. Create the Entity (5 min)

Create `Task.java` (entity, not record - JPA needs mutable objects):

```java
package com.example.demo;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "tasks")
public class Task {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String title;

    @Column(length = 1000)
    private String description;

    @Column(nullable = false)
    private String status;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    // JPA requires no-arg constructor
    protected Task() {}

    public Task(String title, String description, String status) {
        this.title = title;
        this.description = description;
        this.status = status;
        this.createdAt = Instant.now();
    }

    // Getters
    public String getId() { return id; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public String getStatus() { return status; }
    public Instant getCreatedAt() { return createdAt; }

    // Setters for mutable fields
    public void setTitle(String title) { this.title = title; }
    public void setDescription(String description) { this.description = description; }
    public void setStatus(String status) { this.status = status; }
}
```

### 4. Create the Repository (2 min)

Create `TaskRepository.java`:

```java
package com.example.demo;

import org.springframework.data.jpa.repository.JpaRepository;

public interface TaskRepository extends JpaRepository<Task, String> {
    // That's it! JpaRepository provides:
    // - save(entity)
    // - findById(id)
    // - findAll()
    // - deleteById(id)
    // - existsById(id)
    // - count()
}
```

### 5. Update the Service (4 min)

```java
package com.example.demo;

import com.example.demo.exceptions.TaskNotFoundException;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class TaskService {

    private final TaskRepository repository;

    public TaskService(TaskRepository repository) {
        this.repository = repository;
    }

    public Task create(TaskRequest request) {
        var task = new Task(
            request.title(),
            request.description(),
            request.status() != null ? request.status() : "pending"
        );
        return repository.save(task);
    }

    public Task findById(String id) {
        return repository.findById(id)
            .orElseThrow(() -> new TaskNotFoundException(id));
    }

    public List<Task> findAll() {
        return repository.findAll();
    }

    public Task update(String id, TaskRequest request) {
        Task task = findById(id);
        task.setTitle(request.title());
        task.setDescription(request.description());
        task.setStatus(request.status());
        return repository.save(task);
    }

    public void delete(String id) {
        if (!repository.existsById(id)) {
            throw new TaskNotFoundException(id);
        }
        repository.deleteById(id);
    }
}
```

## Test It

```bash
./gradlew bootRun

# Create a task
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn JPA", "description": "Persist data"}'

# Check the H2 console
# Go to: http://localhost:8080/h2-console
# JDBC URL: jdbc:h2:mem:testdb
# Username: sa
# Password: (empty)
```

## Go vs Spring Comparison

| Go | Spring Data JPA |
|----|-----------------|
| `db:"column_name"` struct tags | `@Column(name = "column_name")` |
| Manual SQL queries | Generated from interface methods |
| `db.ExecContext(ctx, sql, ...)` | `repository.save(entity)` |
| `db.QueryContext(ctx, sql, ...)` | `repository.findById(id)` |
| Handle `sql.ErrNoRows` | Returns `Optional<T>` |
| Manual transaction management | `@Transactional` annotation |

## Entity Annotations Reference

```java
@Entity                          // Marks class as JPA entity
@Table(name = "tasks")           // Custom table name

@Id                              // Primary key
@GeneratedValue(strategy = ...)  // Auto-generate ID
  - GenerationType.UUID          // Generate UUID string
  - GenerationType.IDENTITY      // Database auto-increment
  - GenerationType.SEQUENCE      // Database sequence

@Column(
    name = "col_name",           // Custom column name
    nullable = false,            // NOT NULL constraint
    unique = true,               // UNIQUE constraint
    length = 255,                // VARCHAR length
    updatable = false            // Can't update after insert
)

@Temporal(TemporalType.TIMESTAMP)  // For java.util.Date
@CreatedDate                       // Auto-set on create (needs @EnableJpaAuditing)
@LastModifiedDate                  // Auto-set on update
```

## Why Entity, Not Record?

JPA entities need:
- No-arg constructor (records can't have one)
- Mutable setters (records are immutable)
- Proxy creation for lazy loading

**Pattern:** Use `record` for DTOs, `class` for entities.

```
Request DTO (record) -> Service -> Entity (class) -> Repository
                                         |
Response DTO (record) <-- mapper/convert-+
```

## Common Gotchas

### 1. No-Arg Constructor

```java
// JPA needs this, but make it protected to prevent misuse
protected Task() {}
```

### 2. Equals/HashCode

For entities, base on ID only:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Task task)) return false;
    return id != null && id.equals(task.id);
}

@Override
public int hashCode() {
    return getClass().hashCode();
}
```

### 3. ToString Avoid Lazy Collections

```java
@Override
public String toString() {
    return "Task{id=" + id + ", title=" + title + "}";
    // Don't include lazy-loaded relations
}
```

## Checkpoint

- [ ] H2 console accessible at `/h2-console`
- [ ] Tasks persist across requests (within same run)
- [ ] `show-sql: true` shows SQL in console
- [ ] Repository interface has no implementation code

**Next:** Exercise 11 - Repository Layer with custom queries
