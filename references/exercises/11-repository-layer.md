# Exercise 11: Repository Layer

**Time:** 15 minutes  
**Goal:** Add custom queries and pagination to your repository

## The Go Version

```go
func (r *TaskRepo) FindByStatus(ctx context.Context, status string) ([]Task, error) {
    rows, err := r.db.QueryContext(ctx, 
        "SELECT * FROM tasks WHERE status = $1 ORDER BY created_at DESC", 
        status)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var tasks []Task
    for rows.Next() {
        var t Task
        rows.Scan(&t.ID, &t.Title, ...)
        tasks = append(tasks, t)
    }
    return tasks, nil
}

func (r *TaskRepo) FindPaginated(ctx context.Context, page, size int) ([]Task, error) {
    offset := page * size
    rows, err := r.db.QueryContext(ctx,
        "SELECT * FROM tasks ORDER BY created_at DESC LIMIT $1 OFFSET $2",
        size, offset)
    // ...
}
```

## Your Task

1. Add query methods using Spring Data naming conventions
2. Add a custom `@Query` for complex queries
3. Implement pagination with `Pageable`

## Try First (Optional)

Before looking at the solution:
- Add `List<Task> findByStatus(String status)` to repository
- Add `Page<Task> findAll(Pageable pageable)` to repository
- Try `@Query("SELECT t FROM Task t WHERE ...")` for custom JPQL

---

## Step by Step

### 1. Add Query Methods (5 min)

Update `TaskRepository.java`:

```java
package com.example.demo;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.Instant;
import java.util.List;

public interface TaskRepository extends JpaRepository<Task, String> {

    // Spring Data generates: SELECT * FROM tasks WHERE status = ?
    List<Task> findByStatus(String status);

    // SELECT * FROM tasks WHERE title LIKE '%keyword%'
    List<Task> findByTitleContaining(String keyword);

    // SELECT * FROM tasks WHERE status = ? ORDER BY created_at DESC
    List<Task> findByStatusOrderByCreatedAtDesc(String status);

    // Multiple conditions: SELECT * FROM tasks WHERE status = ? AND title LIKE ?
    List<Task> findByStatusAndTitleContaining(String status, String keyword);

    // Custom JPQL query
    @Query("SELECT t FROM Task t WHERE t.createdAt > :since")
    List<Task> findRecentTasks(@Param("since") Instant since);

    // Native SQL (when JPQL isn't enough)
    @Query(value = "SELECT * FROM tasks WHERE status = :status LIMIT :limit", 
           nativeQuery = true)
    List<Task> findTopByStatus(@Param("status") String status, @Param("limit") int limit);

    // Count queries
    long countByStatus(String status);

    // Exists queries
    boolean existsByTitle(String title);

    // Delete queries
    void deleteByStatus(String status);
}
```

### 2. Add Pagination Support (5 min)

Update `TaskController.java`:

```java
package com.example.demo;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/tasks")
public class TaskController {

    private final TaskRepository repository;

    public TaskController(TaskRepository repository) {
        this.repository = repository;
    }

    // GET /tasks?page=0&size=10&sort=createdAt,desc
    @GetMapping
    public Page<Task> listTasks(Pageable pageable) {
        return repository.findAll(pageable);
    }

    // GET /tasks/status/pending?page=0&size=10
    @GetMapping("/status/{status}")
    public Page<Task> listByStatus(
            @PathVariable String status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return repository.findByStatus(status, pageable);
    }
}
```

Add paginated method to repository:

```java
Page<Task> findByStatus(String status, Pageable pageable);
```

### 3. Test It (5 min)

```bash
# Create some tasks first
curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" \
  -d '{"title": "Task 1", "status": "pending"}'
curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" \
  -d '{"title": "Task 2", "status": "done"}'
curl -X POST http://localhost:8080/tasks -H "Content-Type: application/json" \
  -d '{"title": "Task 3", "status": "pending"}'

# Paginated list
curl "http://localhost:8080/tasks?page=0&size=2"

# By status
curl "http://localhost:8080/tasks/status/pending"

# Check SQL in logs (show-sql: true)
```

## Go vs Spring Data Comparison

| Go | Spring Data JPA |
|----|-----------------|
| Write SQL manually | Method name generates SQL |
| `WHERE status = $1` | `findByStatus(status)` |
| `WHERE x = ? AND y = ?` | `findByXAndY(x, y)` |
| `LIKE '%keyword%'` | `findByTitleContaining(keyword)` |
| `ORDER BY created_at DESC` | `findByStatusOrderByCreatedAtDesc` |
| Manual LIMIT/OFFSET | `Pageable` parameter |
| Return `[]Task` | Return `Page<Task>` with metadata |

## Query Method Keywords

```java
// Comparison
findByAgeGreaterThan(int age)          // age > ?
findByAgeLessThan(int age)             // age < ?
findByAgeBetween(int min, int max)     // age BETWEEN ? AND ?

// String matching
findByTitleContaining(String s)        // LIKE '%s%'
findByTitleStartingWith(String s)      // LIKE 's%'
findByTitleEndingWith(String s)        // LIKE '%s'
findByTitleIgnoreCase(String s)        // LOWER(title) = LOWER(?)

// Null checks
findByDescriptionIsNull()              // description IS NULL
findByDescriptionIsNotNull()           // description IS NOT NULL

// Boolean
findByActiveTrue()                     // active = true
findByActiveFalse()                    // active = false

// Collections
findByStatusIn(List<String> statuses)  // status IN (?, ?, ?)
findByStatusNotIn(List<String> s)      // status NOT IN (...)

// Ordering
findByStatusOrderByCreatedAtDesc()     // ORDER BY created_at DESC
findAllByOrderByTitleAsc()             // ORDER BY title ASC

// Limiting
findTop5ByStatus(String status)        // LIMIT 5
findFirst10ByOrderByCreatedAtDesc()    // LIMIT 10
```

## Page Response Structure

```json
{
  "content": [
    {"id": "1", "title": "Task 1", ...},
    {"id": "2", "title": "Task 2", ...}
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 10,
    "sort": {"sorted": true, "direction": "DESC"}
  },
  "totalElements": 25,
  "totalPages": 3,
  "last": false,
  "first": true,
  "numberOfElements": 10
}
```

## Custom Queries

### JPQL (Java Persistence Query Language)

```java
// Uses entity/field names (not table/column names)
@Query("SELECT t FROM Task t WHERE t.status = :status AND t.createdAt > :since")
List<Task> findRecentByStatus(@Param("status") String status, @Param("since") Instant since);

// With JOIN
@Query("SELECT t FROM Task t JOIN t.assignee a WHERE a.email = :email")
List<Task> findByAssigneeEmail(@Param("email") String email);
```

### Native SQL

```java
// When JPQL isn't enough (database-specific features)
@Query(value = "SELECT * FROM tasks WHERE status = ? LIMIT 10", nativeQuery = true)
List<Task> findTop10ByStatusNative(String status);

// With named parameters
@Query(value = "SELECT * FROM tasks WHERE EXTRACT(YEAR FROM created_at) = :year", 
       nativeQuery = true)
List<Task> findByYear(@Param("year") int year);
```

## Testing Repositories

```java
@DataJpaTest  // Loads only JPA components with H2
class TaskRepositoryTest {

    @Autowired
    private TaskRepository repository;

    @Test
    void findByStatus_returnsMatchingTasks() {
        // Arrange
        repository.save(new Task("Task 1", "Desc", "pending"));
        repository.save(new Task("Task 2", "Desc", "done"));
        repository.save(new Task("Task 3", "Desc", "pending"));

        // Act
        List<Task> pending = repository.findByStatus("pending");

        // Assert
        assertEquals(2, pending.size());
    }
}
```

## Checkpoint

- [ ] Custom query methods work (`findByStatus`, etc.)
- [ ] Pagination returns `Page<Task>` with metadata
- [ ] `@Query` with JPQL works for complex queries
- [ ] SQL is visible in logs

**Next:** Exercise 12 - Async Processing with @Async
