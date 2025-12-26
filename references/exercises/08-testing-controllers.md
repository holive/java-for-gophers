# Exercise 08: Testing Controllers

**Time:** 15 minutes  
**Goal:** Write controller tests with MockMvc and mocked dependencies

## The Go Version

```go
func TestGetTask(t *testing.T) {
    // Create mock service
    mockService := &MockTaskService{
        tasks: map[string]*Task{
            "123": {ID: "123", Title: "Test Task"},
        },
    }
    
    // Create handler with mock
    handler := NewTaskHandler(mockService)
    
    // Create test request
    req := httptest.NewRequest("GET", "/tasks/123", nil)
    w := httptest.NewRecorder()
    
    // Execute
    handler.ServeHTTP(w, req)
    
    // Assert
    if w.Code != http.StatusOK {
        t.Errorf("expected 200, got %d", w.Code)
    }
    
    var task Task
    json.Unmarshal(w.Body.Bytes(), &task)
    if task.Title != "Test Task" {
        t.Errorf("expected 'Test Task', got %s", task.Title)
    }
}
```

## Your Task

Write tests for the TaskController:
1. Test successful GET returns 200 and task data
2. Test GET for missing task returns 404
3. Test POST creates and returns 201
4. Use `@MockBean` to mock the service layer

## Try First (Optional)

Before looking at the solution:
- Annotate test class with `@WebMvcTest(TaskController.class)`
- Inject `MockMvc` with `@Autowired`
- Mock the service with `@MockBean`

---

## Step by Step

### 1. Add Test Dependencies (already included with Spring Boot)

In `build.gradle`, these come with `spring-boot-starter-test`:
```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
// Includes: JUnit 5, Mockito, MockMvc, AssertJ, Hamcrest
```

### 2. Create the Test Class (10 min)

Create `src/test/java/com/example/demo/TaskControllerTest.java`:

```java
package com.example.demo;

import com.example.demo.exceptions.TaskNotFoundException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.List;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(TaskController.class)
class TaskControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private TaskService taskService;

    @Test
    void getTask_returnsTask_whenExists() throws Exception {
        // Arrange
        var task = new TaskResponse("123", "Test Task", "Description", "pending");
        when(taskService.findById("123")).thenReturn(task);

        // Act & Assert
        mockMvc.perform(get("/tasks/123"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("123"))
            .andExpect(jsonPath("$.title").value("Test Task"))
            .andExpect(jsonPath("$.status").value("pending"));
    }

    @Test
    void getTask_returns404_whenNotFound() throws Exception {
        // Arrange
        when(taskService.findById("999"))
            .thenThrow(new TaskNotFoundException("999"));

        // Act & Assert
        mockMvc.perform(get("/tasks/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("Not Found"));
    }

    @Test
    void listTasks_returnsList() throws Exception {
        // Arrange
        var tasks = List.of(
            new TaskResponse("1", "Task 1", "Desc", "pending"),
            new TaskResponse("2", "Task 2", "Desc", "done")
        );
        when(taskService.findAll(null)).thenReturn(tasks);

        // Act & Assert
        mockMvc.perform(get("/tasks"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.length()").value(2))
            .andExpect(jsonPath("$[0].title").value("Task 1"))
            .andExpect(jsonPath("$[1].title").value("Task 2"));
    }

    @Test
    void createTask_returns201_withCreatedTask() throws Exception {
        // Arrange
        var created = new TaskResponse("new-id", "New Task", "Description", "pending");
        when(taskService.create(any(TaskRequest.class))).thenReturn(created);

        // Act & Assert
        mockMvc.perform(post("/tasks")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "title": "New Task",
                        "description": "Description"
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("new-id"))
            .andExpect(jsonPath("$.title").value("New Task"));
    }

    @Test
    void deleteTask_returns204_whenDeleted() throws Exception {
        // Arrange - service.delete() returns void, no setup needed

        // Act & Assert
        mockMvc.perform(delete("/tasks/123"))
            .andExpect(status().isNoContent());
    }
}
```

### 3. Run the Tests (5 min)

```bash
./gradlew test --tests TaskControllerTest
```

Or in IntelliJ: Right-click the test class -> Run

## What's Happening Here?

### @WebMvcTest

- Loads **only** the web layer (controllers, filters, exception handlers)
- Does **not** load services, repositories, or full Spring context
- Fast because minimal context

### @MockBean

- Creates a Mockito mock of `TaskService`
- Replaces the real bean in the Spring context
- You control its behavior with `when(...).thenReturn(...)`

### MockMvc

- Simulates HTTP requests without starting a real server
- No network, no ports - just in-memory servlet simulation
- Returns `ResultActions` for fluent assertions

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| `httptest.NewRequest()` | `MockMvc.perform(get(...))` |
| `httptest.NewRecorder()` | MockMvc captures response internally |
| Manual mock structs | `@MockBean` with Mockito |
| `if w.Code != 200` | `.andExpect(status().isOk())` |
| `json.Unmarshal()` then assert | `.andExpect(jsonPath("$.field").value(...))` |

## Common MockMvc Patterns

### Request Building

```java
// GET with query params
mockMvc.perform(get("/tasks")
    .param("status", "pending")
    .param("limit", "10"))

// POST with JSON body
mockMvc.perform(post("/tasks")
    .contentType(MediaType.APPLICATION_JSON)
    .content("{\"title\": \"Test\"}"))

// With headers
mockMvc.perform(get("/tasks")
    .header("Authorization", "Bearer token123"))
```

### Response Assertions

```java
// Status codes
.andExpect(status().isOk())           // 200
.andExpect(status().isCreated())      // 201
.andExpect(status().isNoContent())    // 204
.andExpect(status().isNotFound())     // 404
.andExpect(status().isBadRequest())   // 400

// JSON assertions
.andExpect(jsonPath("$.id").value("123"))
.andExpect(jsonPath("$.items.length()").value(5))
.andExpect(jsonPath("$.items[0].name").value("First"))
.andExpect(jsonPath("$.nested.field").exists())
.andExpect(jsonPath("$.optional").doesNotExist())

// Headers
.andExpect(header().string("X-Custom", "value"))

// Content type
.andExpect(content().contentType(MediaType.APPLICATION_JSON))
```

### Mockito Patterns

```java
// Return value
when(service.find("id")).thenReturn(task);

// Throw exception
when(service.find("bad")).thenThrow(new NotFoundException());

// Any argument
when(service.create(any(TaskRequest.class))).thenReturn(task);

// Capture argument
ArgumentCaptor<TaskRequest> captor = ArgumentCaptor.forClass(TaskRequest.class);
verify(service).create(captor.capture());
assertEquals("Title", captor.getValue().title());

// Verify called
verify(service).delete("123");
verify(service, never()).delete("456");
verify(service, times(2)).findAll(any());
```

## Test Slice Annotations

| Annotation | What It Loads | Use For |
|------------|---------------|---------|
| `@WebMvcTest` | Controllers, filters, advice | Controller tests |
| `@DataJpaTest` | JPA, repositories, H2 | Repository tests |
| `@SpringBootTest` | Everything | Integration tests |

**Rule:** Use the most specific slice. `@WebMvcTest` is faster than `@SpringBootTest`.

## Checkpoint

- [ ] Test class has `@WebMvcTest(TaskController.class)`
- [ ] Service is mocked with `@MockBean`
- [ ] Tests for success and error cases pass
- [ ] No real service or database is used

**Next:** Exercise 09 - Testing Services with Mockito
