# Exercise 09: Testing Services

**Time:** 15 minutes  
**Goal:** Write unit tests for service classes using Mockito

## The Go Version

```go
func TestTaskService_Create(t *testing.T) {
    // Create mock repository
    mockRepo := &MockTaskRepo{}
    mockRepo.On("Save", mock.Anything).Return(&Task{ID: "123"}, nil)
    
    // Create service with mock
    service := NewTaskService(mockRepo)
    
    // Execute
    task, err := service.Create("Title", "Description")
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "123", task.ID)
    mockRepo.AssertCalled(t, "Save", mock.Anything)
}
```

## Your Task

Write unit tests for TaskService:
1. Test create returns a new task with generated ID
2. Test findById throws when task doesn't exist
3. Test update modifies existing task
4. Mock the repository dependency

## Try First (Optional)

Before looking at the solution:
- Use `@ExtendWith(MockitoExtension.class)`
- Create mocks with `@Mock`
- Inject mocks with `@InjectMocks`
- Setup behavior with `when(...).thenReturn(...)`

---

## Step by Step

### 1. Refactor Service for Testability (3 min)

First, let's add a repository interface so we can mock it:

```java
package com.example.demo;

import java.util.Optional;
import java.util.List;

public interface TaskRepository {
    TaskResponse save(TaskResponse task);
    Optional<TaskResponse> findById(String id);
    List<TaskResponse> findAll();
    void deleteById(String id);
    boolean existsById(String id);
}
```

In-memory implementation (for now):

```java
package com.example.demo;

import org.springframework.stereotype.Repository;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Repository
public class InMemoryTaskRepository implements TaskRepository {
    
    private final Map<String, TaskResponse> tasks = new ConcurrentHashMap<>();

    @Override
    public TaskResponse save(TaskResponse task) {
        tasks.put(task.id(), task);
        return task;
    }

    @Override
    public Optional<TaskResponse> findById(String id) {
        return Optional.ofNullable(tasks.get(id));
    }

    @Override
    public List<TaskResponse> findAll() {
        return new ArrayList<>(tasks.values());
    }

    @Override
    public void deleteById(String id) {
        tasks.remove(id);
    }

    @Override
    public boolean existsById(String id) {
        return tasks.containsKey(id);
    }
}
```

Update service to use repository:

```java
@Service
public class TaskService {
    
    private final TaskRepository repository;
    
    public TaskService(TaskRepository repository) {
        this.repository = repository;
    }

    public TaskResponse create(TaskRequest request) {
        String id = UUID.randomUUID().toString();
        var task = new TaskResponse(
            id,
            request.title(),
            request.description(),
            request.status() != null ? request.status() : "pending"
        );
        return repository.save(task);
    }

    public TaskResponse findById(String id) {
        return repository.findById(id)
            .orElseThrow(() -> new TaskNotFoundException(id));
    }
    
    // ... other methods
}
```

### 2. Create the Service Test (12 min)

Create `src/test/java/com/example/demo/TaskServiceTest.java`:

```java
package com.example.demo;

import com.example.demo.exceptions.TaskNotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class TaskServiceTest {

    @Mock
    private TaskRepository repository;

    @InjectMocks
    private TaskService taskService;

    @Test
    void create_generatesIdAndSaves() {
        // Arrange
        var request = new TaskRequest("Test Title", "Description", null);
        when(repository.save(any(TaskResponse.class)))
            .thenAnswer(invocation -> invocation.getArgument(0));

        // Act
        TaskResponse result = taskService.create(request);

        // Assert
        assertNotNull(result.id());
        assertEquals("Test Title", result.title());
        assertEquals("pending", result.status());  // Default status
        
        verify(repository).save(any(TaskResponse.class));
    }

    @Test
    void create_usesProvidedStatus() {
        // Arrange
        var request = new TaskRequest("Title", "Desc", "in_progress");
        when(repository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // Act
        TaskResponse result = taskService.create(request);

        // Assert
        assertEquals("in_progress", result.status());
    }

    @Test
    void findById_returnsTask_whenExists() {
        // Arrange
        var task = new TaskResponse("123", "Title", "Desc", "pending");
        when(repository.findById("123")).thenReturn(Optional.of(task));

        // Act
        TaskResponse result = taskService.findById("123");

        // Assert
        assertEquals("123", result.id());
        assertEquals("Title", result.title());
    }

    @Test
    void findById_throwsException_whenNotExists() {
        // Arrange
        when(repository.findById("999")).thenReturn(Optional.empty());

        // Act & Assert
        TaskNotFoundException exception = assertThrows(
            TaskNotFoundException.class,
            () -> taskService.findById("999")
        );
        
        assertEquals("999", exception.getTaskId());
    }

    @Test
    void create_capturesCorrectData() {
        // Arrange
        var request = new TaskRequest("Captured Title", "Captured Desc", "done");
        when(repository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // Act
        taskService.create(request);

        // Assert - capture the argument
        ArgumentCaptor<TaskResponse> captor = ArgumentCaptor.forClass(TaskResponse.class);
        verify(repository).save(captor.capture());
        
        TaskResponse captured = captor.getValue();
        assertEquals("Captured Title", captured.title());
        assertEquals("Captured Desc", captured.description());
        assertEquals("done", captured.status());
    }
}
```

### 3. Run the Tests

```bash
./gradlew test --tests TaskServiceTest
```

## What's Happening Here?

### @ExtendWith(MockitoExtension.class)

- Initializes Mockito mocks before each test
- No Spring context - pure unit test, very fast

### @Mock

- Creates a mock implementation of the interface
- All methods return default values (null, 0, empty) until you define behavior

### @InjectMocks

- Creates the real service class
- Injects all `@Mock` fields into the constructor

### No Spring Context

Unlike `@WebMvcTest`, this test doesn't start Spring at all. It's a pure unit test:
- Faster execution
- Tests only the service logic
- Dependencies are fully controlled mocks

## Go vs Spring Comparison

| Go (testify/mock) | Java (Mockito) |
|-------------------|----------------|
| `mockRepo.On("Save", ...)` | `when(repo.save(...))` |
| `.Return(value, nil)` | `.thenReturn(value)` |
| `mock.Anything` | `any()` or `any(Type.class)` |
| `mockRepo.AssertCalled(t, ...)` | `verify(repo).method(...)` |
| Manual struct mock | `@Mock` annotation |
| Pass mock to constructor | `@InjectMocks` handles it |

## Mockito Patterns Reference

### Stubbing Returns

```java
// Return value
when(repo.findById("123")).thenReturn(Optional.of(task));

// Return different values on subsequent calls
when(repo.findById("123"))
    .thenReturn(Optional.empty())
    .thenReturn(Optional.of(task));

// Throw exception
when(repo.findById("bad")).thenThrow(new RuntimeException("DB error"));

// Dynamic answer
when(repo.save(any())).thenAnswer(inv -> {
    TaskResponse arg = inv.getArgument(0);
    return new TaskResponse("generated-id", arg.title(), arg.description(), arg.status());
});
```

### Argument Matchers

```java
when(repo.findById(any())).thenReturn(...);
when(repo.findById(anyString())).thenReturn(...);
when(repo.findById(eq("specific"))).thenReturn(...);
when(repo.findById(argThat(id -> id.startsWith("test-")))).thenReturn(...);
```

### Verification

```java
// Was called
verify(repo).save(any());

// Was called with specific arg
verify(repo).findById("123");

// Never called
verify(repo, never()).deleteById(any());

// Called N times
verify(repo, times(2)).findAll();

// Called at least/most
verify(repo, atLeast(1)).save(any());
verify(repo, atMost(3)).findById(any());
```

### Argument Captor

```java
ArgumentCaptor<TaskResponse> captor = ArgumentCaptor.forClass(TaskResponse.class);
verify(repo).save(captor.capture());

TaskResponse captured = captor.getValue();
assertEquals("expected", captured.title());

// For multiple captures
List<TaskResponse> allCaptured = captor.getAllValues();
```

## When to Use What

| Test Type | Annotation | Use For |
|-----------|------------|---------|
| Pure unit test | `@ExtendWith(MockitoExtension.class)` | Service logic with mocked deps |
| Controller test | `@WebMvcTest` | HTTP layer with mocked services |
| Repository test | `@DataJpaTest` | JPA with real H2 database |
| Full integration | `@SpringBootTest` | Everything together |

## Checkpoint

- [ ] Tests use `@ExtendWith(MockitoExtension.class)` (no Spring)
- [ ] Repository is mocked with `@Mock`
- [ ] Service is created with `@InjectMocks`
- [ ] Tests cover success and error paths

**Next:** Exercise 10 - Database Setup with Spring Data JPA
