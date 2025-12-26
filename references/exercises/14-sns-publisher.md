# Exercise 14: SNS Publisher

**Time:** 15 minutes  
**Goal:** Publish events to SNS topics using Spring Cloud AWS

## The Go Version

```go
func (p *Publisher) Publish(ctx context.Context, event TaskEvent) error {
    body, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    _, err = p.sns.Publish(ctx, &sns.PublishInput{
        TopicArn: aws.String(p.topicArn),
        Message:  aws.String(string(body)),
        MessageAttributes: map[string]types.MessageAttributeValue{
            "eventType": {
                DataType:    aws.String("String"),
                StringValue: aws.String(event.Action),
            },
        },
    })
    return err
}
```

## Your Task

1. Add Spring Cloud AWS SNS dependency
2. Create an SNS topic in LocalStack
3. Publish events with `SnsTemplate`
4. Add message attributes for filtering

## Prerequisites

Create SNS topic in LocalStack:

```bash
aws --endpoint-url=http://localhost:4566 sns create-topic --name task-notifications
```

Subscribe SQS queue to topic (for testing):

```bash
aws --endpoint-url=http://localhost:4566 sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:000000000000:task-notifications \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:000000000000:task-events
```

---

## Step by Step

### 1. Add Dependencies (already have from Ex 13)

```groovy
dependencies {
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-sns'
}
```

### 2. Configure SNS Endpoint (2 min)

In `application.yml`:

```yaml
spring:
  cloud:
    aws:
      sns:
        endpoint: http://localhost:4566

app:
  sns:
    task-topic-arn: arn:aws:sns:us-east-1:000000000000:task-notifications
```

### 3. Create Publisher Service (5 min)

Create `TaskEventPublisher.java`:

```java
package com.example.demo;

import io.awspring.cloud.sns.core.SnsTemplate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class TaskEventPublisher {

    private static final Logger log = LoggerFactory.getLogger(TaskEventPublisher.class);
    
    private final SnsTemplate snsTemplate;
    private final String topicArn;

    public TaskEventPublisher(
            SnsTemplate snsTemplate,
            @Value("${app.sns.task-topic-arn}") String topicArn) {
        this.snsTemplate = snsTemplate;
        this.topicArn = topicArn;
    }

    public void publishTaskCreated(String taskId) {
        var event = new TaskEvent(taskId, "created", java.time.Instant.now().toString());
        publish(event);
    }

    public void publishTaskCompleted(String taskId) {
        var event = new TaskEvent(taskId, "completed", java.time.Instant.now().toString());
        publish(event);
    }

    private void publish(TaskEvent event) {
        log.info("Publishing event: {} for task {}", event.action(), event.taskId());
        
        snsTemplate.convertAndSend(topicArn, event);
        
        log.info("Event published successfully");
    }
}
```

### 4. Publish with Headers (5 min)

For message filtering by subscribers:

```java
package com.example.demo;

import io.awspring.cloud.sns.core.SnsTemplate;
import io.awspring.cloud.sns.core.TopicArnResolver;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class TaskEventPublisher {

    private final SnsTemplate snsTemplate;
    private final String topicArn;

    // ... constructor

    public void publishWithAttributes(TaskEvent event) {
        var message = MessageBuilder
            .withPayload(event)
            .setHeader("eventType", event.action())
            .setHeader("priority", "high")
            .build();
        
        snsTemplate.send(topicArn, message);
    }

    // Alternative: Using SnsNotification
    public void publishWithNotification(TaskEvent event) {
        snsTemplate.sendNotification(
            topicArn,
            SnsNotification.builder(event)
                .header("eventType", event.action())
                .subject("Task Event: " + event.action())
                .build()
        );
    }
}
```

### 5. Integrate with Task Service (3 min)

Update `TaskService.java`:

```java
@Service
public class TaskService {

    private final TaskRepository repository;
    private final TaskEventPublisher eventPublisher;

    public TaskService(TaskRepository repository, TaskEventPublisher eventPublisher) {
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }

    public Task create(TaskRequest request) {
        var task = new Task(request.title(), request.description(), "pending");
        Task saved = repository.save(task);
        
        // Publish event after successful save
        eventPublisher.publishTaskCreated(saved.getId());
        
        return saved;
    }

    public void complete(String id) {
        Task task = findById(id);
        task.setStatus("done");
        repository.save(task);
        
        eventPublisher.publishTaskCompleted(id);
    }
}
```

## Test It

```bash
# Create a task (triggers SNS publish)
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Test SNS"}'

# Check SQS for the message (via SNS subscription)
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url http://localhost:4566/000000000000/task-events
```

## Go vs Spring Comparison

| Go | Spring Cloud AWS |
|----|------------------|
| `sns.Publish(ctx, input)` | `snsTemplate.convertAndSend(topic, payload)` |
| Manual `json.Marshal` | Auto-serialization |
| `MessageAttributes` map | Message headers |
| TopicArn as string param | Injected via `@Value` |

## Message Attributes for Filtering

SNS subscriptions can filter by message attributes:

```java
// Publisher adds attribute
snsTemplate.send(topicArn, MessageBuilder
    .withPayload(event)
    .setHeader("eventType", "completed")
    .build());

// SQS subscription filter (set in AWS console or IaC):
// { "eventType": ["completed"] }
// Only receives "completed" events
```

## Async Publishing

```java
@Async
public void publishAsync(TaskEvent event) {
    snsTemplate.convertAndSend(topicArn, event);
}
```

## Error Handling

```java
public void publishWithRetry(TaskEvent event) {
    try {
        snsTemplate.convertAndSend(topicArn, event);
    } catch (SnsException e) {
        log.error("Failed to publish event: {}", event, e);
        // Store in dead letter table, alert, etc.
        throw new EventPublishException("Failed to publish", e);
    }
}
```

## Topic by Name (Auto-Resolve ARN)

```java
// Configure auto-resolution
@Bean
public SnsTemplate snsTemplate(SnsClient snsClient) {
    return new SnsTemplate(snsClient, new TopicArnResolver() {
        @Override
        public String resolveTopicArn(String topicName) {
            // Resolve "task-notifications" to full ARN
            return "arn:aws:sns:us-east-1:000000000000:" + topicName;
        }
    });
}

// Then use topic name
snsTemplate.convertAndSend("task-notifications", event);
```

## SNS + SQS Pattern

Common architecture:

```
Producer -> SNS Topic -> SQS Queue 1 (notifications)
                     -> SQS Queue 2 (analytics)
                     -> Lambda (real-time processing)
```

Each subscriber processes independently, with its own retry/DLQ logic.

## Checkpoint

- [ ] SNS topic exists in LocalStack
- [ ] `SnsTemplate` publishes events
- [ ] SQS subscriber receives messages
- [ ] Message headers work for filtering

**Next:** Exercise 15 - S3 Operations with S3Template
