# Exercise Index

Exercises organized by topic. Each takes ~15 minutes.

## Phase 0: OOP Foundations (Optional)

*Recommended for architectural confidence. Skip if you want to start coding immediately - you can return later.*

| # | Exercise | You'll Learn |
|---|----------|--------------|
| 00a | [SOLID Principles](00a-solid.md) | The vocabulary of architecture discussions |
| 00b | [Composition vs Inheritance](00b-composition-vs-inheritance.md) | The most common design debate |
| 00c | [Interface Design](00c-interface-design.md) | Contracts, default methods, abstract classes |
| 00d | [Design Patterns](00d-design-patterns.md) | Strategy, Factory, Observer in Spring context |
| 00e | [Code Smells](00e-code-smells.md) | Recognizing bad OOP and the fix |

**Reference:** [OOP Foundations Quick Reference](../oop-foundations.md)

---

## Phase 1: Syntax Survival

| # | Exercise | You'll Learn |
|---|----------|--------------|
| 01 | [First Endpoint](01-first-endpoint.md) | @RestController, @GetMapping, records |
| 02 | [Validation](02-validation.md) | @Valid, jakarta.validation, error handling |
| 03 | [Filters/Middleware](03-filters-middleware.md) | OncePerRequestFilter, request attributes |
| 04 | Service Layer | @Service, constructor injection |
| 05 | Configuration | @Value, @ConfigurationProperties, profiles |

## Phase 2: Building APIs

| # | Exercise | You'll Learn |
|---|----------|--------------|
| 06 | CRUD Controller | All HTTP methods, path variables, query params |
| 07 | Error Handling | @RestControllerAdvice, custom exceptions |
| 08 | Testing Controllers | @WebMvcTest, MockMvc, @MockBean |
| 09 | Testing Services | @SpringBootTest, Mockito |
| 10 | Database Setup | Spring Data JPA, H2, entities |

## Phase 3: Data & Async

| # | Exercise | You'll Learn |
|---|----------|--------------|
| 11 | Repository Layer | JpaRepository, queries, pagination |
| 12 | Async Processing | @Async, CompletableFuture |
| 13 | SQS Consumer | @SqsListener, message handling |
| 14 | SNS Publisher | SnsTemplate, event publishing |
| 15 | S3 Operations | S3Template, upload/download |

## Phase 4: Production Patterns

| # | Exercise | You'll Learn |
|---|----------|--------------|
| 16 | DynamoDB | @DynamoDbBean, DynamoDbEnhancedClient |
| 17 | Health Checks | Spring Actuator, custom health indicators |
| 18 | Metrics | Micrometer, Prometheus export |
| 19 | Circuit Breakers | Resilience4j, @CircuitBreaker |
| 20 | Integration Test | @SpringBootTest with LocalStack |

## Project Iterations

After exercises, build the "Notifier" service. Iterations are milestones, not requirements.

### Core Path (minimum viable)
Complete these 3 to consider yourself successful:

| Iteration | Focus |
|-----------|-------|
| 1 | Minimal — one endpoint, hardcoded response |
| 2 | Add validation, proper DTOs, error handling |
| 3 | Add service layer, dependency injection |

### Extended Path (optional)
Continue if you want deeper mastery:

| Iteration | Focus |
|-----------|-------|
| 4 | Add SQS listener and SNS publisher |
| 5 | Add DynamoDB persistence |
| 6 | Add tests (unit + integration) |
| 7 | Add observability (actuator, metrics) |

### Alternative: Switch Projects
After iteration 3, you can build a different service instead of continuing Notifier:
- A file processor (S3 trigger -> transform -> store)
- A webhook receiver (validate -> enqueue -> acknowledge)

Variety prevents tedium. The goal is pattern internalization, not Notifier completion.

Each iteration can be started fresh to build muscle memory.
