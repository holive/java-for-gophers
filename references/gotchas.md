# Gotchas for Go Developers

Things that will feel wrong, break unexpectedly, or just annoy you.

## 1. NullPointerException (Your New Enemy)

**Go:** Zero values are safe. Empty string is `""`, not nil.

**Java:** References can be null. This compiles but explodes:
```java
String name = null;
name.length();  // NullPointerException at runtime
```

**Survival tactics:**
- Use `Optional<T>` for values that might be absent
- Prefer `record` types (their fields can still be null, but you're thinking about it)
- IntelliJ warns you — pay attention to those warnings
- Java 14+ has helpful NPE messages showing exactly which variable was null

## 2. Exceptions Instead of Error Returns

**Your Go instinct:**
```go
user, err := findUser(id)
if err != nil {
    return nil, err
}
```

**Java wants:**
```java
try {
    User user = findUser(id);
} catch (UserNotFoundException e) {
    // handle it
}
```

**The hard truth:** Fighting this is exhausting. Embrace exceptions for exceptional cases, use `Optional` for expected absence. Your code will look more Java-like, and that's okay.

## 3. Checked vs Unchecked Exceptions

Java has TWO kinds of exceptions:

**Checked** (must handle or declare):
```java
public void readFile() throws IOException {  // Must declare!
    Files.readString(path);
}
```

**Unchecked** (RuntimeException — like Go panics):
```java
throw new IllegalArgumentException("bad input");  // No declaration needed
```

**Rule of thumb:** Create your own exceptions as `extends RuntimeException` to avoid the ceremony.

## 4. Mutability Everywhere

**Go:** Slices and maps are mutable, but structs are often passed by value.

**Java:** Everything is a reference. This modifies the original:
```java
void addItem(List<String> items) {
    items.add("sneaky");  // Mutates caller's list!
}
```

**Survival tactics:**
- Use `List.of()`, `Map.of()` for immutable collections
- Use `record` for immutable data objects
- Be explicit: `new ArrayList<>(original)` to copy

## 5. == vs .equals()

**Go:** `==` works for strings, structs (mostly).

**Java:** `==` compares references, not values!
```java
String a = new String("hello");
String b = new String("hello");
a == b      // false! Different objects
a.equals(b) // true! Same content
```

**Exception:** `==` works for primitives (`int`, `boolean`) and often for String literals (due to interning), but don't rely on it.

## 6. Primitives vs Wrapper Classes

Java has:
- `int` — primitive, can't be null, efficient
- `Integer` — object, can be null, has methods

```java
int count = null;      // Compile error
Integer count = null;  // Valid, but NPE waiting to happen

// Autoboxing magic (and confusion):
Integer a = 100;
Integer b = 100;
a == b  // true (cached small integers)

Integer c = 200;
Integer d = 200;
c == d  // false! (not cached)
```

**Rule:** Use primitives unless you need nullability.

## 7. Package Structure Matters

**Go:** Package name = directory name. Simple.

**Java:** Package must match directory structure exactly:
```
src/main/java/com/company/service/UserService.java
```
The file MUST declare: `package com.company.service;`

Spring Boot scans from the main class's package downward. Put your `@SpringBootApplication` class at the root, or components won't be found.

## 8. Annotations Are Magic (Sometimes Too Magic)

Spring does a lot behind the scenes:

```java
@Autowired  // "Inject a dependency here"
@Transactional  // "Wrap this in a database transaction"
@Async  // "Run this in a separate thread"
```

**The trap:** These work via proxies. Calling an `@Async` method from within the same class bypasses the proxy:

```java
@Service
public class MyService {
    @Async
    public void asyncMethod() { }
    
    public void caller() {
        asyncMethod();  // NOT async! Proxy bypassed!
    }
}
```

## 9. No Multiple Return Values

**Go:**
```go
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("divide by zero")
    }
    return a / b, nil
}
```

**Java options:**
```java
// Option 1: Throw exception
int divide(int a, int b) {
    if (b == 0) throw new ArithmeticException();
    return a / b;
}

// Option 2: Return wrapper
record DivisionResult(int value, String error) {}

// Option 3: Return Optional (for "might not exist")
Optional<Integer> divide(int a, int b) {
    return b == 0 ? Optional.empty() : Optional.of(a / b);
}
```

## 10. Verbose Generics

**Go:**
```go
func Map[T, U any](items []T, f func(T) U) []U
```

**Java:**
```java
public static <T, U> List<U> map(List<T> items, Function<T, U> f)
```

And it gets worse with bounded types. Just accept it.

## 11. Build Tool Complexity

**Go:** `go build`. Done.

**Java:**
- Gradle or Maven (your company uses Gradle — good, it's more Go-like)
- `build.gradle` or `pom.xml` for dependencies
- Spring Initializr to bootstrap projects

**Tip:** Let Spring Initializr (start.spring.io) generate your project skeleton. Don't fight the structure.

## 12. Testing Requires More Setup

**Go:** Table-driven tests, `go test`, done.

**Java:**
- JUnit 5 for test framework
- Mockito for mocks
- `@SpringBootTest` for integration tests (slow!)
- `@WebMvcTest` for controller tests (faster)
- `@DataJpaTest` for repository tests

The test annotations control what Spring loads. Wrong annotation = slow tests or missing beans.

## 13. No defer

**Go:**
```go
f, _ := os.Open("file")
defer f.Close()
```

**Java (try-with-resources):**
```java
try (var f = new FileInputStream("file")) {
    // use f
}  // auto-closed here
```

Only works with `AutoCloseable` types. For other cleanup, you're back to try-finally.

## 14. Time Handling

**Go:** `time.Time` is simple and good.

**Java:** Multiple APIs (the old ones are bad):
- ❌ `java.util.Date` — mutable, broken, avoid
- ❌ `java.sql.Date` — even worse
- ✅ `java.time.Instant` — like Go's `time.Time`
- ✅ `java.time.LocalDateTime` — for no-timezone times
- ✅ `java.time.ZonedDateTime` — with timezone

Always use `java.time.*` classes (Java 8+).
