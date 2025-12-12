# AWS with Spring Cloud AWS 3.x

Your company uses Spring Cloud AWS 3.4.0 with AWS SDK v2. This reference covers the patterns you'll use most.

## Setup (build.gradle)

```groovy
dependencyManagement {
    imports {
        mavenBom "io.awspring.cloud:spring-cloud-aws-dependencies:3.4.0"
        mavenBom "software.amazon.awssdk:bom:2.31.68"
    }
}

dependencies {
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-s3'
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-sqs'
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-sns'
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-dynamodb'
}
```

## S3

### Go Pattern:
```go
func (s *S3Client) Upload(ctx context.Context, bucket, key string, data []byte) error {
    _, err := s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
        Body:   bytes.NewReader(data),
    })
    return err
}
```

### Spring Cloud AWS Pattern:
```java
@Service
public class StorageService {
    private final S3Template s3Template;
    
    public StorageService(S3Template s3Template) {
        this.s3Template = s3Template;
    }
    
    public void upload(String bucket, String key, byte[] data) {
        s3Template.upload(bucket, key, new ByteArrayInputStream(data));
    }
    
    public byte[] download(String bucket, String key) {
        S3Resource resource = s3Template.download(bucket, key);
        return resource.getContentAsByteArray();
    }
    
    public void delete(String bucket, String key) {
        s3Template.deleteObject(bucket, key);
    }
}
```

**Configuration:**
```yaml
spring:
  cloud:
    aws:
      s3:
        region: us-east-1
      credentials:
        access-key: ${AWS_ACCESS_KEY_ID}
        secret-key: ${AWS_SECRET_ACCESS_KEY}
```

## SQS

### Go Pattern:
```go
// Send
func (q *Queue) Send(ctx context.Context, msg Message) error {
    body, _ := json.Marshal(msg)
    _, err := q.client.SendMessage(ctx, &sqs.SendMessageInput{
        QueueUrl:    aws.String(q.url),
        MessageBody: aws.String(string(body)),
    })
    return err
}

// Receive (polling loop)
func (q *Queue) Listen(ctx context.Context, handler func(Message)) {
    for {
        out, _ := q.client.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{...})
        for _, msg := range out.Messages {
            var m Message
            json.Unmarshal([]byte(*msg.Body), &m)
            handler(m)
            q.client.DeleteMessage(ctx, &sqs.DeleteMessageInput{...})
        }
    }
}
```

### Spring Cloud AWS Pattern:

**Sending:**
```java
@Service
public class NotificationService {
    private final SqsTemplate sqsTemplate;
    
    public NotificationService(SqsTemplate sqsTemplate) {
        this.sqsTemplate = sqsTemplate;
    }
    
    public void sendNotification(NotificationEvent event) {
        sqsTemplate.send("notification-queue", event);
    }
    
    // With options
    public void sendWithDelay(NotificationEvent event) {
        sqsTemplate.send(to -> to
            .queue("notification-queue")
            .payload(event)
            .delaySeconds(30)
        );
    }
}
```

**Receiving (annotation-driven — no polling loop!):**
```java
@Component
public class NotificationListener {
    
    @SqsListener("notification-queue")
    public void handleNotification(NotificationEvent event) {
        // Process the event
        // Auto-acknowledged on success, auto-retried on exception
    }
    
    // With acknowledgment control
    @SqsListener("important-queue")
    public void handleImportant(NotificationEvent event, Acknowledgement ack) {
        try {
            process(event);
            ack.acknowledge();  // Manual ack
        } catch (Exception e) {
            // Don't ack — message returns to queue
        }
    }
}
```

**Configuration:**
```yaml
spring:
  cloud:
    aws:
      sqs:
        region: us-east-1
        listener:
          max-concurrent-messages: 10
          max-messages-per-poll: 10
```

## SNS

### Go Pattern:
```go
func (n *Notifier) Publish(ctx context.Context, topic, message string) error {
    _, err := n.client.Publish(ctx, &sns.PublishInput{
        TopicArn: aws.String(topic),
        Message:  aws.String(message),
    })
    return err
}
```

### Spring Cloud AWS Pattern:
```java
@Service
public class EventPublisher {
    private final SnsTemplate snsTemplate;
    
    public EventPublisher(SnsTemplate snsTemplate) {
        this.snsTemplate = snsTemplate;
    }
    
    public void publishEvent(String topicArn, OrderEvent event) {
        snsTemplate.sendNotification(topicArn, event, "Order Event");
    }
    
    // With message attributes
    public void publishWithAttributes(String topicArn, OrderEvent event) {
        snsTemplate.send(topicArn, SnsNotification.<OrderEvent>builder()
            .payload(event)
            .header("event-type", "ORDER_CREATED")
            .build());
    }
}
```

**Receiving SNS via SQS subscription:**
```java
@SqsListener("order-events-queue")
public void handleOrderEvent(
    @Payload OrderEvent event,
    @Header("event-type") String eventType
) {
    // SNS message attributes available as headers
}
```

## DynamoDB

### Go Pattern:
```go
type User struct {
    PK        string `dynamodbav:"pk"`
    SK        string `dynamodbav:"sk"`
    Email     string `dynamodbav:"email"`
    CreatedAt int64  `dynamodbav:"created_at"`
}

func (r *UserRepo) Save(ctx context.Context, user User) error {
    av, _ := attributevalue.MarshalMap(user)
    _, err := r.client.PutItem(ctx, &dynamodb.PutItemInput{
        TableName: aws.String(r.table),
        Item:      av,
    })
    return err
}
```

### Spring Cloud AWS Pattern:

**Entity:**
```java
@DynamoDbBean
public class User {
    private String pk;
    private String sk;
    private String email;
    private Instant createdAt;
    
    @DynamoDbPartitionKey
    public String getPk() { return pk; }
    
    @DynamoDbSortKey
    public String getSk() { return sk; }
    
    // Other getters/setters...
}
```

**Repository:**
```java
@Repository
public class UserRepository {
    private final DynamoDbEnhancedClient dynamoDbEnhancedClient;
    private final DynamoDbTable<User> userTable;
    
    public UserRepository(DynamoDbEnhancedClient client) {
        this.dynamoDbEnhancedClient = client;
        this.userTable = client.table("users", TableSchema.fromBean(User.class));
    }
    
    public void save(User user) {
        userTable.putItem(user);
    }
    
    public Optional<User> findByKey(String pk, String sk) {
        Key key = Key.builder().partitionValue(pk).sortValue(sk).build();
        return Optional.ofNullable(userTable.getItem(key));
    }
    
    public List<User> findByPartitionKey(String pk) {
        QueryConditional query = QueryConditional.keyEqualTo(
            Key.builder().partitionValue(pk).build()
        );
        return userTable.query(query).items().stream().toList();
    }
}
```

**Configuration:**
```yaml
spring:
  cloud:
    aws:
      dynamodb:
        region: us-east-1
```

## Key Differences Summary

| Aspect | Go | Spring Cloud AWS |
|--------|-----|------------------|
| SQS Listener | Manual polling loop | `@SqsListener` annotation |
| Message ack | Explicit delete call | Auto on success, manual with `Acknowledgement` |
| Serialization | `json.Marshal/Unmarshal` | Automatic Jackson |
| DynamoDB mapping | Struct tags | Annotations + `@DynamoDbBean` |
| Config | Explicit client creation | Auto-configured from `application.yml` |

## Local Development

Use LocalStack for local AWS:

```yaml
# application-local.yml
spring:
  cloud:
    aws:
      endpoint: http://localhost:4566
      region:
        static: us-east-1
      credentials:
        access-key: test
        secret-key: test
```
