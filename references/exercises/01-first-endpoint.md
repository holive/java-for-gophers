# Exercise 01: Your First Spring Boot Endpoint

**Time:** 15 minutes
**Goal:** Get a Spring Boot app running with one endpoint

## Prerequisites (First Time Only)

### Verify Your Environment

Run these commands. All should succeed before continuing.

```bash
java --version    # expect: 17 or 21
```

If Java is missing, see setup options below.

### Project Setup

**Option A: Use the template** (recommended for first time)

```bash
cp -r ~/.claude/skills/java-for-gophers/template/spring-boot-starter ./my-first-spring-app
cd my-first-spring-app
./gradlew bootRun  # should start on port 8080
```

**Option B: Generate from scratch**

Go to [start.spring.io](https://start.spring.io), select Gradle + Java 21 + Spring Web, download, unzip.

### Alternative: Setup via IntelliJ

**IntelliJ Ultimate** (has Spring Initializr built-in):

1. File > New > Project
2. Select "Spring Initializr" (left panel)
3. Settings:
   - Language: Java
   - Type: Gradle - Kotlin or Groovy
   - JDK: 21 (click dropdown > "Download JDK" if not installed)
   - Java: 21
4. Dependencies: Add "Spring Web"
5. Create

**IntelliJ Community** (no Spring Initializr):

1. Generate project at [start.spring.io](https://start.spring.io) (Gradle + Java 21 + Spring Web)
2. Download and unzip
3. File > Open > select the unzipped folder
4. IntelliJ detects Gradle, click "Load Gradle Project"
5. If no JDK: File > Project Structure > SDKs > + > Download JDK > select 21

Both handle Gradle wrapper and project structure. Ultimate is faster, Community works fine.

### Troubleshooting

**"java: command not found"** (terminal)
- macOS: `brew install openjdk@21`
- After install: `export JAVA_HOME=$(/usr/libexec/java_home -v 21)`
- Or: use IntelliJ method above to avoid terminal setup

**"./gradlew: Permission denied"**
- Run: `chmod +x gradlew`

**IntelliJ: No JDK available**
- File > Project Structure > SDKs > + > Download JDK > select 21

**Compile error: "cannot find symbol" or class not found**
- Your file is in the wrong directory
- Spring scans from the @SpringBootApplication package downward
- Files must be in: `src/main/java/com/example/demo/`
- If your main class is in `com.example.demo`, controllers must be in `com.example.demo` or below

---

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

## Why This Structure Matters

Spring Boot has opinions about where code lives:

```
src/main/java/
    com/example/demo/
        DemoApplication.java      <-- @SpringBootApplication lives here
        HelloController.java      <-- must be same package or subpackage
        HelloResponse.java
```

The `@SpringBootApplication` annotation triggers component scanning starting from its package. If you put `HelloController` in a different root package (like `com.other.thing`), Spring won't find it.

This is different from Go where packages are explicit imports. Spring uses classpath scanning - it finds your classes automatically, but only if they're in the right place.

**Common mistake:** Creating a file in the wrong directory causes "cannot find symbol" errors at compile time, or the controller simply won't be registered (no errors, but endpoints don't work).

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
