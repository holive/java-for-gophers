# Exercise 05: Configuration

**Time:** 15 minutes  
**Goal:** Externalize configuration using Spring's property system

## The Go Version

```go
type Config struct {
    Port        int    `env:"PORT" envDefault:"8080"`
    DatabaseURL string `env:"DATABASE_URL,required"`
    FeatureFlag bool   `env:"ENABLE_NEW_FEATURE" envDefault:"false"`
}

func LoadConfig() (*Config, error) {
    var cfg Config
    if err := env.Parse(&cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}

// Usage in main.go
func main() {
    cfg, err := LoadConfig()
    if err != nil {
        log.Fatal(err)
    }
    
    server := NewServer(cfg)
    server.Run()
}
```

## Your Task

Configure your application with:
1. A welcome message that can be changed via configuration
2. A feature flag to enable/disable a greeting suffix
3. Different values for "local" vs "prod" profiles

## Try First (Optional)

Before looking at the solution, try:
- Adding a property to `application.yml`
- Injecting it with `@Value("${property.name}")`
- Creating `application-local.yml` with different values

---

## Step by Step

### 1. Add Configuration Properties (3 min)

Create/edit `src/main/resources/application.yml`:

```yaml
app:
  welcome-message: "Hello from Spring Boot"
  greeting-suffix-enabled: false
  
server:
  port: 8080
```

### 2. Use @Value for Simple Properties (4 min)

Update `HelloController.java`:

```java
package com.example.demo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @Value("${app.welcome-message}")
    private String welcomeMessage;
    
    @Value("${app.greeting-suffix-enabled}")
    private boolean suffixEnabled;

    @GetMapping("/hello")
    public HelloResponse hello() {
        String message = suffixEnabled 
            ? welcomeMessage + " - Have a great day!"
            : welcomeMessage;
        return new HelloResponse(message);
    }
}
```

### 3. Create a Profile-Specific Config (3 min)

Create `src/main/resources/application-local.yml`:

```yaml
app:
  welcome-message: "Hello from LOCAL environment"
  greeting-suffix-enabled: true
  
server:
  port: 8081
```

Run with the local profile:

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
```

### 4. Use @ConfigurationProperties for Type Safety (5 min)

For more complex configuration, use a dedicated class.

Create `AppConfig.java`:

```java
package com.example.demo;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app")
public record AppConfig(
    String welcomeMessage,
    boolean greetingSuffixEnabled
) {}
```

Enable configuration properties in your main class:

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

@SpringBootApplication
@EnableConfigurationProperties(AppConfig.class)
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

Now inject the config object:

```java
@RestController
public class HelloController {

    private final AppConfig config;

    public HelloController(AppConfig config) {
        this.config = config;
    }

    @GetMapping("/hello")
    public HelloResponse hello() {
        String message = config.greetingSuffixEnabled()
            ? config.welcomeMessage() + " - Have a great day!"
            : config.welcomeMessage();
        return new HelloResponse(message);
    }
}
```

## Test It

```bash
# Default profile
./gradlew bootRun
curl http://localhost:8080/hello
# {"message":"Hello from Spring Boot"}

# Local profile
./gradlew bootRun --args='--spring.profiles.active=local'
curl http://localhost:8081/hello
# {"message":"Hello from LOCAL environment - Have a great day!"}
```

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| `env:"VAR_NAME"` struct tags | `@Value("${var.name}")` |
| `envDefault:"value"` | Default in YAML or `@Value("${var:default}")` |
| Load in main, pass around | Inject anywhere with `@Value` or config class |
| Single config struct | `@ConfigurationProperties` for groups |
| Environment-specific via `.env` files | Profiles: `application-{profile}.yml` |

## Property Sources (Priority Order)

Spring loads configuration from multiple sources. Higher priority wins:

1. Command line arguments (`--server.port=9000`)
2. Environment variables (`SERVER_PORT=9000`)
3. `application-{profile}.yml`
4. `application.yml`
5. Default values in code

**Environment variable naming:** `app.welcome-message` becomes `APP_WELCOME_MESSAGE`

## Common Patterns

### Required Property (Fail Fast)

```java
@Value("${database.url}")  // Fails on startup if missing
private String databaseUrl;
```

### Optional with Default

```java
@Value("${feature.enabled:false}")
private boolean featureEnabled;
```

### Injecting Lists

```yaml
app:
  allowed-origins:
    - http://localhost:3000
    - https://myapp.com
```

```java
@Value("${app.allowed-origins}")
private List<String> allowedOrigins;
```

### Secret Management

For secrets, use environment variables or a vault:

```yaml
database:
  password: ${DB_PASSWORD}  # From environment
```

## When to Use What

| Approach | Use When |
|----------|----------|
| `@Value` | Simple, single property injection |
| `@ConfigurationProperties` | Related properties that belong together |
| Environment variables | Secrets, container/cloud deployments |
| Profiles | Environment-specific config (local, dev, prod) |

## Checkpoint

- [ ] Property loads from `application.yml`
- [ ] Different value loads when using `-local` profile
- [ ] `@ConfigurationProperties` class works with record

**Next:** Exercise 06 - Complete CRUD Controller
