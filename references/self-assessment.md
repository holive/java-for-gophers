# Self-Assessment Checklists

Check what you can do without looking it up.

Use this on low-energy days: reviewing what you know is still progress.

---

## Phase 1: Syntax Survival

### I can write...

- [ ] A `@RestController` with `@GetMapping` and `@PostMapping`
- [ ] A `record` for request/response DTOs
- [ ] A filter using `OncePerRequestFilter`
- [ ] A `@Service` class with constructor injection
- [ ] Configuration using `@Value("${property}")`

### I understand...

- [ ] Why Java uses exceptions instead of error returns
- [ ] The difference between `@Component`, `@Service`, `@Repository`, `@Controller`
- [ ] How constructor injection works (no `@Autowired` needed on single constructor)
- [ ] Why `record` is preferred over `class` for DTOs

### I can explain to someone...

- [ ] What Spring Boot's `@SpringBootApplication` actually does
- [ ] Why package structure matters in Spring

---

## Phase 2: Building APIs

### I can write...

- [ ] A complete CRUD controller (GET, POST, PUT, DELETE)
- [ ] Request validation with `@Valid` and Jakarta annotations
- [ ] A `@RestControllerAdvice` for global exception handling
- [ ] A custom exception that maps to a specific HTTP status
- [ ] A controller test with `@WebMvcTest` and `MockMvc`

### I understand...

- [ ] How `@PathVariable` vs `@RequestParam` differ
- [ ] Why `@WebMvcTest` is faster than `@SpringBootTest`
- [ ] How `@MockBean` replaces real beans in tests
- [ ] The purpose of `ResponseEntity<T>`

### I can explain to someone...

- [ ] How Spring's exception handling flows from controller to response
- [ ] Why we mock dependencies instead of using real ones in unit tests

---

## Phase 3: Data & Async

### I can write...

- [ ] A JPA entity with `@Entity`, `@Id`, `@Column`
- [ ] A `JpaRepository` interface with custom query methods
- [ ] An async method using `@Async`
- [ ] An SQS listener with `@SqsListener`
- [ ] S3 operations using `S3Template`

### I understand...

- [ ] How Spring Data generates query implementations from method names
- [ ] Why `@Async` must be on a different class (proxy limitation)
- [ ] How `CompletableFuture` composes async operations
- [ ] The difference between `@Transactional` on read vs write operations

### I can explain to someone...

- [ ] Why Java 21 virtual threads are like goroutines
- [ ] How Spring Cloud AWS simplifies SQS/SNS/S3

---

## Phase 4: Production Patterns

### I can write...

- [ ] A DynamoDB entity with `@DynamoDbBean`
- [ ] A custom health indicator for Actuator
- [ ] A Micrometer counter or timer
- [ ] A circuit breaker with Resilience4j
- [ ] An integration test using `@SpringBootTest` with LocalStack

### I understand...

- [ ] What Spring Actuator exposes and why it matters
- [ ] How circuit breakers prevent cascade failures
- [ ] Why integration tests need containers (LocalStack, Testcontainers)
- [ ] The difference between unit, integration, and end-to-end tests

### I can explain to someone...

- [ ] How to diagnose a production issue using metrics and health checks
- [ ] Why we test with LocalStack instead of real AWS in CI

---

## Final: Gopher No More

### Mindset shifts

- [ ] I reach for exceptions, not error wrappers
- [ ] I trust Spring to wire my dependencies
- [ ] I use annotations without feeling dirty
- [ ] I write interfaces for testability, not just abstraction
- [ ] I let the framework handle validation, not manual if-checks

### Practical proof

- [ ] I can write a REST endpoint without thinking about Go
- [ ] I can debug a Spring Boot stacktrace without panic
- [ ] I know where to put code in Spring's package structure
- [ ] I can read someone else's Spring Boot code and understand it

### Emotional state

- [ ] I feel neutral (not hostile) toward Java
- [ ] I can appreciate what Spring does well
- [ ] I accept that this is a skill worth having
- [ ] I don't mentally translate from Go anymore when writing Java

---

## How to Use This

**On a full-energy day:**
- Work through exercises
- Come back here to check off what you learned

**On a low-energy day:**
- Scan through and check boxes for things you already know
- Notice progress since last time
- Pick ONE unchecked item to review (not learn, just review)

**When frustrated:**
- Count your checkmarks
- Remember: every check is evidence of progress
- Close this file and do something else

---

Checking boxes is progress. You don't have to do exercises to move forward.
