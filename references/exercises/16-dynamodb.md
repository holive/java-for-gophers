# Exercise 16: DynamoDB

**Time:** 15 minutes  
**Goal:** Store and retrieve data using DynamoDB with Spring Cloud AWS

## The Go Version

```go
type User struct {
    PK        string `dynamodbav:"pk"`
    SK        string `dynamodbav:"sk"`
    Email     string `dynamodbav:"email"`
    Name      string `dynamodbav:"name"`
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

func (r *UserRepo) FindByPK(ctx context.Context, pk string) ([]User, error) {
    output, err := r.client.Query(ctx, &dynamodb.QueryInput{
        TableName:              aws.String(r.table),
        KeyConditionExpression: aws.String("pk = :pk"),
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":pk": &types.AttributeValueMemberS{Value: pk},
        },
    })
    // ... unmarshal results
}
```

## Your Task

1. Create a DynamoDB table in LocalStack
2. Define an entity with `@DynamoDbBean`
3. Create a repository using `DynamoDbEnhancedClient`
4. Implement basic CRUD operations

## Prerequisites

Create DynamoDB table in LocalStack:

```bash
aws --endpoint-url=http://localhost:4566 dynamodb create-table \
  --table-name tasks \
  --attribute-definitions \
      AttributeName=pk,AttributeType=S \
      AttributeName=sk,AttributeType=S \
  --key-schema \
      AttributeName=pk,KeyType=HASH \
      AttributeName=sk,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

---

## Step by Step

### 1. Add Dependencies (2 min)

```groovy
dependencies {
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-dynamodb'
}
```

### 2. Configure DynamoDB (2 min)

In `application.yml`:

```yaml
spring:
  cloud:
    aws:
      dynamodb:
        endpoint: http://localhost:4566
```

### 3. Create DynamoDB Entity (5 min)

Create `TaskItem.java`:

```java
package com.example.demo;

import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.*;
import java.time.Instant;

@DynamoDbBean
public class TaskItem {

    private String pk;          // Partition key: "TASK"
    private String sk;          // Sort key: task ID
    private String title;
    private String description;
    private String status;
    private Instant createdAt;

    // Default constructor required
    public TaskItem() {}

    public TaskItem(String id, String title, String description, String status) {
        this.pk = "TASK";
        this.sk = id;
        this.title = title;
        this.description = description;
        this.status = status;
        this.createdAt = Instant.now();
    }

    @DynamoDbPartitionKey
    public String getPk() { return pk; }
    public void setPk(String pk) { this.pk = pk; }

    @DynamoDbSortKey
    public String getSk() { return sk; }
    public void setSk(String sk) { this.sk = sk; }

    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }

    // Convenience method
    public String getId() { return sk; }
}
```

### 4. Create Repository (6 min)

Create `DynamoTaskRepository.java`:

```java
package com.example.demo;

import org.springframework.stereotype.Repository;
import software.amazon.awssdk.enhanced.dynamodb.*;
import software.amazon.awssdk.enhanced.dynamodb.model.*;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Repository
public class DynamoTaskRepository {

    private final DynamoDbTable<TaskItem> table;

    public DynamoTaskRepository(DynamoDbEnhancedClient enhancedClient) {
        this.table = enhancedClient.table("tasks", TableSchema.fromBean(TaskItem.class));
    }

    public TaskItem save(TaskItem item) {
        if (item.getSk() == null) {
            item.setSk(UUID.randomUUID().toString());
            item.setPk("TASK");
        }
        table.putItem(item);
        return item;
    }

    public Optional<TaskItem> findById(String id) {
        Key key = Key.builder()
            .partitionValue("TASK")
            .sortValue(id)
            .build();
        return Optional.ofNullable(table.getItem(key));
    }

    public List<TaskItem> findAll() {
        QueryConditional query = QueryConditional.keyEqualTo(
            Key.builder().partitionValue("TASK").build()
        );
        
        return table.query(query).items().stream().toList();
    }

    public List<TaskItem> findByStatus(String status) {
        // Scan with filter (not efficient for large tables)
        return table.scan(ScanEnhancedRequest.builder()
                .filterExpression(Expression.builder()
                    .expression("#status = :status")
                    .putExpressionName("#status", "status")
                    .putExpressionValue(":status", AttributeValue.builder().s(status).build())
                    .build())
                .build())
            .items().stream().toList();
    }

    public void delete(String id) {
        Key key = Key.builder()
            .partitionValue("TASK")
            .sortValue(id)
            .build();
        table.deleteItem(key);
    }
}
```

## Test It

```bash
# Create a task
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "DynamoDB task", "description": "Test"}'

# List tasks
curl http://localhost:8080/tasks

# Verify in LocalStack
aws --endpoint-url=http://localhost:4566 dynamodb scan --table-name tasks
```

## Go vs Spring Comparison

| Go | Spring Cloud AWS |
|----|------------------|
| `dynamodbav:"pk"` struct tags | `@DynamoDbPartitionKey` |
| `attributevalue.MarshalMap()` | Automatic via `TableSchema.fromBean()` |
| `client.PutItem(ctx, input)` | `table.putItem(item)` |
| `client.Query(ctx, input)` | `table.query(QueryConditional)` |
| Manual expression building | `Expression.builder()` |

## DynamoDB Annotations

```java
@DynamoDbBean                    // Marks class as DynamoDB entity
@DynamoDbPartitionKey            // Partition key (required)
@DynamoDbSortKey                 // Sort key (optional)
@DynamoDbAttribute("col_name")   // Custom attribute name
@DynamoDbIgnore                  // Don't persist this field
@DynamoDbSecondaryPartitionKey   // GSI partition key
@DynamoDbSecondarySortKey        // GSI/LSI sort key
```

## Key Design Patterns

### Single Table Design

```java
// All entities in one table with different pk/sk patterns
@DynamoDbBean
public class TaskItem {
    // pk = "TASK", sk = taskId
}

@DynamoDbBean  
public class UserItem {
    // pk = "USER", sk = userId
}

@DynamoDbBean
public class CommentItem {
    // pk = "TASK#taskId", sk = "COMMENT#commentId"
}
```

### Query by Partition Key

```java
public List<TaskItem> findByUser(String userId) {
    QueryConditional query = QueryConditional.keyEqualTo(
        Key.builder().partitionValue("USER#" + userId).build()
    );
    return table.query(query).items().stream().toList();
}
```

### Query with Sort Key Condition

```java
public List<TaskItem> findTasksAfter(String userId, Instant after) {
    QueryConditional query = QueryConditional.sortGreaterThan(
        Key.builder()
            .partitionValue("USER#" + userId)
            .sortValue(after.toString())
            .build()
    );
    return table.query(query).items().stream().toList();
}
```

## Global Secondary Index (GSI)

```java
@DynamoDbBean
public class TaskItem {
    // ...
    
    @DynamoDbSecondaryPartitionKey(indexNames = "status-index")
    public String getStatus() { return status; }
}

// Query GSI
public List<TaskItem> findByStatusViaGsi(String status) {
    DynamoDbIndex<TaskItem> index = table.index("status-index");
    
    QueryConditional query = QueryConditional.keyEqualTo(
        Key.builder().partitionValue(status).build()
    );
    
    return index.query(query).stream()
        .flatMap(page -> page.items().stream())
        .toList();
}
```

## Conditional Writes

```java
public void updateIfExists(TaskItem item) {
    table.putItem(PutItemEnhancedRequest.builder(TaskItem.class)
        .item(item)
        .conditionExpression(Expression.builder()
            .expression("attribute_exists(pk)")
            .build())
        .build());
}
```

## Checkpoint

- [ ] DynamoDB table exists in LocalStack
- [ ] Entity uses `@DynamoDbBean` and key annotations
- [ ] Save and query operations work
- [ ] Items visible in `dynamodb scan`

**Next:** Exercise 17 - Health Checks with Spring Actuator
