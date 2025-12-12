# Java Syntax Refresher

Quick reference for someone who wrote Java years ago and needs a refresh. Modern Java (17+) focus.

## Variables & Types

```java
// Type inference (like Go's :=)
var name = "Hugo";           // Inferred as String
var count = 42;              // Inferred as int
var users = new ArrayList<User>();  // Inferred as ArrayList<User>

// Explicit typing
String name = "Hugo";
int count = 42;
List<User> users = new ArrayList<>();

// Constants
final String API_KEY = "secret";  // Like Go's const (sort of)
```

## Strings

```java
// Concatenation
String full = first + " " + last;

// Formatting (like Go's fmt.Sprintf)
String msg = String.format("Hello %s, you have %d messages", name, count);

// Text blocks (Java 15+) — like Go's backtick strings
String json = """
    {
        "name": "Hugo",
        "active": true
    }
    """;

// Common operations
str.length()                    // len(str) in Go
str.isEmpty()                   // str == "" in Go
str.contains("sub")             // strings.Contains
str.startsWith("pre")           // strings.HasPrefix
str.split(",")                  // strings.Split
str.trim()                      // strings.TrimSpace
str.toLowerCase()
str.toUpperCase()
```

## Collections

```java
// Lists (like Go slices)
List<String> names = new ArrayList<>();
names.add("Hugo");
names.get(0);                   // names[0] in Go
names.size();                   // len(names) in Go
names.isEmpty();
names.contains("Hugo");

// Immutable list (like a frozen slice)
List<String> names = List.of("Alice", "Bob");

// Maps (like Go maps)
Map<String, Integer> ages = new HashMap<>();
ages.put("Hugo", 30);
ages.get("Hugo");               // ages["Hugo"] in Go
ages.getOrDefault("Unknown", 0);
ages.containsKey("Hugo");

// Immutable map
Map<String, Integer> ages = Map.of("Hugo", 30, "Alice", 25);

// Iterating
for (String name : names) { }                    // for _, name := range names
for (var entry : ages.entrySet()) {
    entry.getKey();
    entry.getValue();
}
```

## Control Flow

```java
// If (parentheses required, braces optional for single statement)
if (count > 0) {
    doSomething();
} else if (count == 0) {
    doNothing();
} else {
    handleNegative();
}

// Switch (enhanced, Java 14+)
String result = switch (status) {
    case "active" -> "Running";
    case "paused" -> "On hold";
    default -> "Unknown";
};

// For loops
for (int i = 0; i < 10; i++) { }          // for i := 0; i < 10; i++
for (String item : items) { }              // for _, item := range items
while (condition) { }
```

## Functions/Methods

```java
// Method definition
public String greet(String name) {
    return "Hello " + name;
}

// With multiple params
public void process(String name, int count, boolean active) { }

// Varargs (like Go's ...)
public void log(String... messages) {
    for (String msg : messages) {
        System.out.println(msg);
    }
}
log("one", "two", "three");

// No multiple return values! Use records or throw exceptions instead.
```

## Classes & Records

```java
// Traditional class (verbose)
public class User {
    private String id;
    private String email;
    
    public User(String id, String email) {
        this.id = id;
        this.email = email;
    }
    
    public String getId() { return id; }
    public String getEmail() { return email; }
}

// Record (Java 16+) — like a Go struct, immutable
public record User(String id, String email) {}

// Usage is the same
var user = new User("123", "hugo@example.com");
user.id();      // accessor (note: no "get" prefix for records)
user.email();
```

## Interfaces

```java
// Definition
public interface Notifier {
    void send(String message);
    
    // Default method (has implementation)
    default void sendUrgent(String message) {
        send("URGENT: " + message);
    }
}

// Implementation (explicit, unlike Go)
public class EmailNotifier implements Notifier {
    @Override
    public void send(String message) {
        // send email
    }
}
```

## Null Handling

```java
// Optional (for values that might be absent)
Optional<User> maybeUser = repository.findById(id);

// Check and use
if (maybeUser.isPresent()) {
    User user = maybeUser.get();
}

// Or more elegantly
maybeUser.ifPresent(user -> sendEmail(user));

// With default
User user = maybeUser.orElse(defaultUser);

// Or throw
User user = maybeUser.orElseThrow(() -> new NotFoundException("User not found"));
```

## Lambdas & Streams

```java
// Lambda (like Go's anonymous functions)
Runnable task = () -> System.out.println("Hello");
Function<String, Integer> length = s -> s.length();
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// Method references
list.forEach(System.out::println);  // item -> System.out.println(item)

// Streams (functional pipeline)
List<String> result = users.stream()
    .filter(u -> u.isActive())           // filter
    .map(User::getEmail)                 // transform
    .sorted()                            // sort
    .distinct()                          // remove duplicates
    .limit(10)                           // take first 10
    .collect(Collectors.toList());       // collect to list

// Common terminal operations
.count()
.findFirst()                             // returns Optional
.anyMatch(predicate)
.allMatch(predicate)
.forEach(consumer)
```

## Exception Handling

```java
// Try-catch
try {
    riskyOperation();
} catch (SpecificException e) {
    handleSpecific(e);
} catch (Exception e) {
    handleGeneral(e);
} finally {
    cleanup();  // Always runs
}

// Try-with-resources (auto-close, like Go's defer)
try (var reader = new FileReader("file.txt")) {
    // use reader
}  // auto-closed

// Throwing
throw new IllegalArgumentException("Bad input");
throw new RuntimeException("Something went wrong", cause);
```

## Annotations

```java
@Override           // Method overrides parent
@Deprecated         // Don't use this anymore
@SuppressWarnings   // Suppress compiler warnings
@FunctionalInterface // Interface with single abstract method

// Spring-specific (you'll see these constantly)
@Component          // "This is a Spring-managed bean"
@Service            // Same as @Component, but semantic
@Repository         // Same, for data access
@Controller         // Same, for web controllers
@RestController     // @Controller + @ResponseBody

@Autowired          // "Inject dependency here"
@Value("${prop}")   // "Inject config value here"

@GetMapping("/path")  // HTTP GET handler
@PostMapping          // HTTP POST handler
@PathVariable         // URL path parameter
@RequestParam         // Query parameter
@RequestBody          // JSON body
```

## Modern Java Features

**Records (Java 16+):**
```java
public record Point(int x, int y) {}
```

**Sealed classes (Java 17+):**
```java
public sealed interface Shape permits Circle, Rectangle {}
```

**Pattern matching (Java 21+):**
```java
if (obj instanceof String s) {
    // s is already a String here
}

switch (shape) {
    case Circle c -> handleCircle(c);
    case Rectangle r -> handleRectangle(r);
}
```

**Virtual threads (Java 21+):**
```java
Thread.startVirtualThread(() -> doWork());
```
