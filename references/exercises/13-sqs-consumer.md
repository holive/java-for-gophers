# Exercise 13: SQS Consumer

**Time:** 15 minutes  
**Goal:** Consume messages from SQS using Spring Cloud AWS

## The Go Version

```go
func (c *Consumer) Listen(ctx context.Context) {
    for {
        output, err := c.sqs.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
            QueueUrl:            aws.String(c.queueURL),
            MaxNumberOfMessages: 10,
            WaitTimeSeconds:     20,
        })
        if err != nil {
            log.Printf("receive error: %v", err)
            continue
        }
        
        for _, msg := range output.Messages {
            var event TaskEvent
            json.Unmarshal([]byte(*msg.Body), &event)
            
            if err := c.process(event); err != nil {
                log.Printf("process error: %v", err)
                continue
            }
            
            // Delete on success
            c.sqs.DeleteMessage(ctx, &sqs.DeleteMessageInput{
                QueueUrl:      aws.String(c.queueURL),
                ReceiptHandle: msg.ReceiptHandle,
            })
        }
    }
}
```

## Your Task

1. Set up LocalStack for local SQS
2. Add Spring Cloud AWS SQS dependency
3. Create a message listener with `@SqsListener`
4. Handle message acknowledgment

## Prerequisites

Start LocalStack (for local AWS):

```bash
docker run -d --name localstack \
  -p 4566:4566 \
  -e SERVICES=sqs,sns,s3,dynamodb \
  localstack/localstack
```

Create a queue:

```bash
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name task-events
```

---

## Step by Step

### 1. Add Dependencies (2 min)

In `build.gradle`:

```groovy
dependencyManagement {
    imports {
        mavenBom "io.awspring.cloud:spring-cloud-aws-dependencies:3.1.0"
    }
}

dependencies {
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-sqs'
}
```

### 2. Configure for LocalStack (2 min)

In `application.yml`:

```yaml
spring:
  cloud:
    aws:
      credentials:
        access-key: test
        secret-key: test
      region:
        static: us-east-1
      sqs:
        endpoint: http://localhost:4566
```

### 3. Create Event DTO (2 min)

Create `TaskEvent.java`:

```java
package com.example.demo;

public record TaskEvent(
    String taskId,
    String action,     // "created", "updated", "completed"
    String timestamp
) {}
```

### 4. Create the Listener (5 min)

Create `TaskEventListener.java`:

```java
package com.example.demo;

import io.awspring.cloud.sqs.annotation.SqsListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class TaskEventListener {

    private static final Logger log = LoggerFactory.getLogger(TaskEventListener.class);

    // Simple listener - auto-acknowledges on success
    @SqsListener("task-events")
    public void handleTaskEvent(TaskEvent event) {
        log.info("Received event: {} for task {}", event.action(), event.taskId());
        
        // Process the event
        switch (event.action()) {
            case "created" -> handleCreated(event);
            case "completed" -> handleCompleted(event);
            default -> log.warn("Unknown action: {}", event.action());
        }
        
        // If method completes normally, message is auto-deleted
        // If exception thrown, message returns to queue
    }

    private void handleCreated(TaskEvent event) {
        log.info("Processing task creation: {}", event.taskId());
        // Send welcome notification, update analytics, etc.
    }

    private void handleCompleted(TaskEvent event) {
        log.info("Processing task completion: {}", event.taskId());
        // Send completion notification, update metrics, etc.
    }
}
```

### 5. Manual Acknowledgment (Optional)

For more control:

```java
import io.awspring.cloud.sqs.annotation.SqsListener;
import io.awspring.cloud.sqs.listener.acknowledgement.Acknowledgement;

@SqsListener("task-events")
public void handleWithManualAck(TaskEvent event, Acknowledgement ack) {
    log.info("Received: {}", event);
    
    try {
        processEvent(event);
        ack.acknowledge();  // Explicitly delete from queue
    } catch (Exception e) {
        log.error("Failed to process, will retry", e);
        // Don't acknowledge - message will return to queue
    }
}
```

### 6. Test It (4 min)

Send a test message:

```bash
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://localhost:4566/000000000000/task-events \
  --message-body '{"taskId":"123","action":"created","timestamp":"2024-01-15T10:00:00Z"}'
```

Watch the logs - you should see the message processed.

## Go vs Spring Comparison

| Go | Spring Cloud AWS |
|----|------------------|
| Manual polling loop | `@SqsListener` annotation |
| `ReceiveMessage` + loop | Spring handles polling |
| Manual `DeleteMessage` | Auto-delete on success |
| Manual JSON unmarshal | Auto-deserialization |
| Manual error handling | Exception = no-ack = retry |

## Listener Configuration

### Multiple Messages

```java
@SqsListener(value = "task-events", maxConcurrentMessages = "10", maxMessagesPerPoll = "5")
public void handleBatch(TaskEvent event) {
    // Called for each message, up to 10 concurrent
}
```

### With Headers/Attributes

```java
@SqsListener("task-events")
public void handleWithHeaders(
        TaskEvent event,
        @Header("MessageId") String messageId,
        @Header("ApproximateReceiveCount") String receiveCount) {
    
    log.info("Message {} received {} times", messageId, receiveCount);
}
```

### Dead Letter Queue Handling

Messages that fail repeatedly go to DLQ. Handle them separately:

```java
@SqsListener("task-events-dlq")
public void handleDeadLetter(TaskEvent event) {
    log.error("DLQ message: {}", event);
    // Alert, store for investigation, etc.
}
```

## Error Handling Patterns

### Retry with Backoff

```java
@SqsListener("task-events")
public void handleWithRetry(TaskEvent event) {
    try {
        process(event);
    } catch (TransientException e) {
        // Throw to retry
        throw e;
    } catch (PermanentException e) {
        // Log and acknowledge to prevent infinite retry
        log.error("Permanent failure, dropping message: {}", event, e);
    }
}
```

### Visibility Timeout Extension

For long-running processing:

```java
@SqsListener(value = "task-events", 
             messageVisibilitySeconds = "300")  // 5 minutes
public void handleLongRunning(TaskEvent event) {
    // Long processing...
}
```

## Configuration Reference

```yaml
spring:
  cloud:
    aws:
      sqs:
        listener:
          max-concurrent-messages: 10
          max-messages-per-poll: 10
          poll-timeout: 20s
```

## Publishing Messages (Preview)

To send messages (covered in detail in Exercise 14):

```java
@Service
public class TaskEventPublisher {
    
    private final SqsTemplate sqsTemplate;
    
    public void publishEvent(TaskEvent event) {
        sqsTemplate.send("task-events", event);
    }
}
```

## Checkpoint

- [ ] LocalStack running with SQS
- [ ] `@SqsListener` receives messages
- [ ] Messages auto-delete on successful processing
- [ ] Failed messages return to queue

**Next:** Exercise 14 - SNS Publisher with SnsTemplate
