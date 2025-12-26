# Exercise 20: Integration Test

**Time:** 15 minutes  
**Goal:** Test the full stack with Testcontainers and LocalStack

## The Go Version

```go
func TestIntegration(t *testing.T) {
    // Start containers
    ctx := context.Background()
    
    localstack, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "localstack/localstack",
            ExposedPorts: []string{"4566/tcp"},
            WaitingFor:   wait.ForLog("Ready."),
        },
        Started: true,
    })
    defer localstack.Terminate(ctx)
    
    postgres, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "postgres:15",
            ExposedPorts: []string{"5432/tcp"},
            Env: map[string]string{
                "POSTGRES_PASSWORD": "test",
            },
        },
        Started: true,
    })
    defer postgres.Terminate(ctx)
    
    // Configure app with container endpoints
    // Run tests...
}
```

## Your Task

1. Add Testcontainers dependencies
2. Create integration test base class
3. Write end-to-end tests with real AWS services
4. Test the full request cycle

## Step by Step

### 1. Add Dependencies (2 min)

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:localstack'
    testImplementation 'org.testcontainers:postgresql'
}
```

### 2. Create Test Configuration (3 min)

Create `src/test/java/com/example/demo/TestcontainersConfig.java`:

```java
package com.example.demo;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.localstack.LocalStackContainer;
import org.testcontainers.utility.DockerImageName;

import static org.testcontainers.containers.localstack.LocalStackContainer.Service.*;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>(DockerImageName.parse("postgres:15"))
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    }

    @Bean
    LocalStackContainer localStackContainer() {
        return new LocalStackContainer(DockerImageName.parse("localstack/localstack:3.0"))
            .withServices(SQS, SNS, S3, DYNAMODB);
    }
}
```

### 3. Create LocalStack Configuration (3 min)

Create `src/test/java/com/example/demo/LocalStackConfig.java`:

```java
package com.example.demo;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.testcontainers.containers.localstack.LocalStackContainer;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sqs.SqsClient;

@TestConfiguration
public class LocalStackConfig {

    @Bean
    @Primary
    public SqsClient sqsClient(LocalStackContainer localStack) {
        return SqsClient.builder()
            .endpointOverride(localStack.getEndpointOverride(LocalStackContainer.Service.SQS))
            .credentialsProvider(StaticCredentialsProvider.create(
                AwsBasicCredentials.create(localStack.getAccessKey(), localStack.getSecretKey())
            ))
            .region(Region.of(localStack.getRegion()))
            .build();
    }
}
```

### 4. Create Integration Test (7 min)

Create `src/test/java/com/example/demo/TaskIntegrationTest.java`:

```java
package com.example.demo;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.localstack.LocalStackContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.CreateQueueRequest;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.testcontainers.containers.localstack.LocalStackContainer.Service.*;

@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
@Import({TestcontainersConfig.class, LocalStackConfig.class})
class TaskIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
        DockerImageName.parse("localstack/localstack:3.0"))
        .withServices(SQS, SNS, S3);

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private TaskRepository taskRepository;

    @Autowired
    private SqsClient sqsClient;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // PostgreSQL
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        // LocalStack
        registry.add("spring.cloud.aws.sqs.endpoint", 
            () -> localStack.getEndpointOverride(SQS).toString());
        registry.add("spring.cloud.aws.sns.endpoint",
            () -> localStack.getEndpointOverride(SNS).toString());
        registry.add("spring.cloud.aws.credentials.access-key", localStack::getAccessKey);
        registry.add("spring.cloud.aws.credentials.secret-key", localStack::getSecretKey);
        registry.add("spring.cloud.aws.region.static", localStack::getRegion);
    }

    @BeforeEach
    void setUp() {
        taskRepository.deleteAll();
        
        // Create SQS queue for tests
        sqsClient.createQueue(CreateQueueRequest.builder()
            .queueName("task-events")
            .build());
    }

    @Test
    void createTask_savesToDatabaseAndPublishesEvent() throws Exception {
        // Create task via API
        String response = mockMvc.perform(post("/tasks")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "title": "Integration Test Task",
                        "description": "Testing the full stack"
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.title").value("Integration Test Task"))
            .andReturn()
            .getResponse()
            .getContentAsString();

        // Verify in database
        var tasks = taskRepository.findAll();
        assertEquals(1, tasks.size());
        assertEquals("Integration Test Task", tasks.get(0).getTitle());

        // Verify SQS message was published
        var messages = sqsClient.receiveMessage(r -> r
            .queueUrl(getQueueUrl("task-events"))
            .maxNumberOfMessages(1)
            .waitTimeSeconds(5)
        ).messages();
        
        assertFalse(messages.isEmpty());
        assertTrue(messages.get(0).body().contains("created"));
    }

    @Test
    void getTask_returnsFromDatabase() throws Exception {
        // Arrange - save directly to DB
        var task = taskRepository.save(new Task("DB Task", "From database", "pending"));

        // Act & Assert
        mockMvc.perform(get("/tasks/" + task.getId()))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.title").value("DB Task"))
            .andExpect(jsonPath("$.description").value("From database"));
    }

    @Test
    void completeWorkflow_createUpdateComplete() throws Exception {
        // 1. Create
        String createResponse = mockMvc.perform(post("/tasks")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\": \"Workflow Test\"}"))
            .andExpect(status().isCreated())
            .andReturn().getResponse().getContentAsString();
        
        String taskId = JsonPath.read(createResponse, "$.id");

        // 2. Update
        mockMvc.perform(put("/tasks/" + taskId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\": \"Updated\", \"status\": \"in_progress\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("in_progress"));

        // 3. Complete
        mockMvc.perform(post("/tasks/" + taskId + "/complete"))
            .andExpect(status().isOk());

        // 4. Verify final state
        mockMvc.perform(get("/tasks/" + taskId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("done"));
    }

    private String getQueueUrl(String queueName) {
        return localStack.getEndpointOverride(SQS) + "/000000000000/" + queueName;
    }
}
```

## Run the Tests

```bash
# Make sure Docker is running
./gradlew test --tests '*IntegrationTest'
```

## Go vs Spring Comparison

| Go (testcontainers-go) | Spring (Testcontainers) |
|------------------------|------------------------|
| `testcontainers.GenericContainer()` | `@Container` annotation |
| Manual port mapping | `@ServiceConnection` auto-config |
| `defer container.Terminate()` | Auto-cleanup after tests |
| Manual env configuration | `@DynamicPropertySource` |
| No framework integration | Deep Spring Boot integration |

## Test Slices vs Full Integration

| Test Type | Annotation | Speed | Scope |
|-----------|------------|-------|-------|
| Unit | None / `@ExtendWith` | Very fast | Single class |
| Controller | `@WebMvcTest` | Fast | HTTP layer |
| Repository | `@DataJpaTest` | Medium | JPA + H2 |
| Full Integration | `@SpringBootTest` | Slow | Everything |

**Rule:** Use the narrowest scope that tests what you need.

## Testcontainers Patterns

### Singleton Container (Faster)

Share containers across tests:

```java
abstract class BaseIntegrationTest {
    
    static final PostgreSQLContainer<?> postgres;
    
    static {
        postgres = new PostgreSQLContainer<>("postgres:15");
        postgres.start();
    }
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }
}

class Test1 extends BaseIntegrationTest { }
class Test2 extends BaseIntegrationTest { }
```

### Wait Strategies

```java
new GenericContainer<>("myimage")
    .waitingFor(Wait.forHttp("/health").forPort(8080))
    .waitingFor(Wait.forLogMessage(".*Ready.*", 1))
    .withStartupTimeout(Duration.ofMinutes(2));
```

## Checkpoint

- [ ] Containers start automatically
- [ ] Tests use real PostgreSQL and LocalStack
- [ ] Database state persists across operations
- [ ] SQS messages are published and readable
- [ ] All tests pass with `./gradlew test`

---

## Congratulations!

You've completed all 20 exercises. You now know how to:

- Build REST APIs with Spring Boot
- Handle validation and errors
- Test controllers and services
- Persist data with JPA and DynamoDB
- Process messages with SQS/SNS
- Store files in S3
- Add observability with health checks and metrics
- Build resilient services with circuit breakers
- Write integration tests with Testcontainers

**What's next?**
- Build the "Notifier" project iterations
- Apply these patterns to your real work
- Revisit exercises to build muscle memory

You're no longer a Go developer who writes Java. You're a developer who knows both.
