# spring boot starter template

minimal spring boot 3.4 project for java-for-gophers exercises.

## quick start

```bash
./gradlew bootRun
```

then visit http://localhost:8080 (will show whitelabel error page - that's expected, no endpoints yet).

## adding your first endpoint

create `src/main/java/com/example/demo/HelloController.java` - see exercise 01.

## project structure

```
src/main/java/com/example/demo/
    DemoApplication.java      <-- entry point, @SpringBootApplication
    (your controllers go here)

src/main/resources/
    application.properties    <-- configuration
```

spring scans for components starting from the package where @SpringBootApplication lives.
your code must be in `com.example.demo` or a subpackage (like `com.example.demo.controllers`).