# Exercise 01: Your First Spring Boot Endpoint

**Time:** 15 minutes  
**Goal:** Get a Spring Boot app running with one endpoint

## The Go Version (What You'd Write)

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Response struct {
    Message string `json:"message"`
}

func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(Response{Message: "Hello from Go"})
    })
    http.ListenAndServe(":8080", nil)
}
```

## Your Task

Create a Spring Boot equivalent that:
1. Runs on port 8080
2. Has a GET endpoint at `/hello`
3. Returns JSON: `{"message": "Hello from Spring Boot"}`

## Try First (Optional)

Before looking at the solution, try writing from memory:
- A class with `@RestController`
- A method with `@GetMapping("/hello")`
- A record for the response

Skip this if you prefer to learn by following along - no judgment.

---

## Step by Step

### 1. Generate Project (2 min)

Go to [start.spring.io](https://start.spring.io) or use CLI:
- Project: Gradle - Kotlin DSL (or Groovy)
- Language: Java
- Spring Boot: 3.2+ (latest stable)
- Dependencies: Spring Web

Download and unzip.

### 2. Create the Response Record (3 min)

Create `src/main/java/com/example/demo/HelloResponse.java`:

```java
package com.example.demo;

public record HelloResponse(String message) {}
```

That's it. Records auto-generate constructor, getters, equals, hashCode, toString.

### 3. Create the Controller (5 min)

Create `src/main/java/com/example/demo/HelloController.java`:

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public HelloResponse hello() {
        return new HelloResponse("Hello from Spring Boot");
    }
}
```

### 4. Run It (5 min)

```bash
./gradlew bootRun
```

Test:
```bash
curl http://localhost:8080/hello
```

Expected output:
```json
{"message":"Hello from Spring Boot"}
```

## What Just Happened?

| Go | Spring Boot |
|----|-------------|
| `http.HandleFunc("/hello", ...)` | `@GetMapping("/hello")` |
| `json.NewEncoder(w).Encode(...)` | Return object → auto-serialized |
| `http.ListenAndServe(":8080", nil)` | Auto-configured, port in `application.properties` |

Spring Boot did a lot behind the scenes:
- Started embedded Tomcat server
- Scanned for `@RestController` classes
- Registered routes from `@GetMapping` annotations
- Configured JSON serialization (Jackson)

## Stretch Goal (if time)

Add a second endpoint `/hello/{name}` that returns `"Hello, {name}"`:

```java
@GetMapping("/hello/{name}")
public HelloResponse helloName(@PathVariable String name) {
    return new HelloResponse("Hello, " + name);
}
```

## Checkpoint

✅ App runs without errors  
✅ `/hello` returns JSON  
✅ You didn't need to write serialization code  

**Next:** Exercise 02 — Adding request validation
