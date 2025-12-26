# Java for Gophers - Curriculum

A practical guide to the course structure and content.

## Overview

- **Target audience:** Experienced Go developers learning Java/Spring Boot
- **Exercise format:** 15-minute focused exercises
- **Total exercises:** 26 (5 optional OOP foundations + 21 core)
- **Estimated time:** ~6.5 hours for all exercises

## Prerequisites

- Go proficiency (you write production Go)
- Basic HTTP/REST concepts
- Docker installed (for LocalStack exercises)
- JDK 21+ and your preferred IDE

## Curriculum

### Phase 0: OOP Foundations (Optional)

*Skip if comfortable with OOP. Return later if architecture discussions feel foreign.*

| # | Exercise | Time | Focus |
|---|----------|------|-------|
| 00a | SOLID Principles | 15 min | Architecture vocabulary |
| 00b | Composition vs Inheritance | 15 min | Design tradeoffs |
| 00c | Interface Design | 15 min | Contracts, abstract classes |
| 00d | Design Patterns | 15 min | Strategy, Factory, Observer |
| 00e | Code Smells | 15 min | Recognizing bad OOP |

### Phase 1: Syntax Survival

*Get running. Build muscle memory for basic Spring patterns.*

| # | Exercise | Time | Focus |
|---|----------|------|-------|
| 01 | First Endpoint | 15 min | @RestController, records |
| 02 | Validation | 15 min | @Valid, jakarta.validation |
| 03 | Filters/Middleware | 15 min | OncePerRequestFilter |
| 04 | Service Layer | 15 min | @Service, DI |
| 05 | Configuration | 15 min | @Value, profiles |

### Phase 2: Building APIs

*Complete REST APIs with proper error handling and tests.*

| # | Exercise | Time | Focus |
|---|----------|------|-------|
| 06 | CRUD Controller | 15 min | All HTTP methods |
| 07 | Error Handling | 15 min | @RestControllerAdvice |
| 08 | Testing Controllers | 15 min | @WebMvcTest, MockMvc |
| 09 | Testing Services | 15 min | Mockito |
| 10 | Database Setup | 15 min | JPA, H2, entities |

### Phase 3: Data & Async

*Persistence patterns and async processing with AWS.*

| # | Exercise | Time | Focus |
|---|----------|------|-------|
| 11 | Repository Layer | 15 min | JpaRepository, pagination |
| 12 | Async Processing | 15 min | @Async, CompletableFuture |
| 13 | SQS Consumer | 15 min | @SqsListener |
| 14 | SNS Publisher | 15 min | SnsTemplate |
| 15 | S3 Operations | 15 min | S3Template |

### Phase 4: Production Patterns

*Observability, resilience, and integration testing.*

| # | Exercise | Time | Focus |
|---|----------|------|-------|
| 16 | DynamoDB | 15 min | DynamoDbEnhancedClient |
| 17 | Health Checks | 15 min | Actuator, custom indicators |
| 18 | Metrics | 15 min | Micrometer, Prometheus |
| 19 | Circuit Breakers | 15 min | Resilience4j |
| 20 | Integration Test | 15 min | Testcontainers, LocalStack |

## Capstone Project

After exercises, build the "Notifier" service through iterations:

| Iteration | Milestone |
|-----------|-----------|
| 1 | Minimal - one endpoint, hardcoded response |
| 2 | Validation, DTOs, error handling |
| 3 | Service layer, dependency injection |
| 4 | SQS listener, SNS publisher |
| 5 | DynamoDB persistence |
| 6 | Unit + integration tests |
| 7 | Actuator, metrics |

**Core path:** Complete iterations 1-3.
**Extended path:** Continue through iteration 7.

## Reference Materials

| Document | Use When |
|----------|----------|
| syntax-refresher.md | Forgot Java syntax basics |
| go-to-spring-mapping.md | Translating Go patterns |
| java-idioms.md | Writing idiomatic Java |
| gopher-anti-patterns.md | Code review, fixing Go-isms |
| gotchas.md | Something breaks unexpectedly |
| concurrency-bridge.md | Working with threads/async |
| aws-spring-cloud.md | AWS integration questions |
| oop-foundations.md | Architecture discussions |
| learning-techniques.md | Understanding the methodology |
| self-assessment.md | Checking your progress |
| resources.md | Low-energy days, extra reading |
