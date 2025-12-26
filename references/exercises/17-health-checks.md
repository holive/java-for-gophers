# Exercise 17: Health Checks

**Time:** 15 minutes  
**Goal:** Add health endpoints and custom health indicators

## The Go Version

```go
func HealthHandler(w http.ResponseWriter, r *http.Request) {
    health := struct {
        Status    string            `json:"status"`
        Checks    map[string]string `json:"checks"`
    }{
        Status: "UP",
        Checks: make(map[string]string),
    }
    
    // Check database
    if err := db.Ping(); err != nil {
        health.Status = "DOWN"
        health.Checks["database"] = "DOWN: " + err.Error()
    } else {
        health.Checks["database"] = "UP"
    }
    
    // Check external service
    if resp, err := http.Get(externalURL + "/health"); err != nil || resp.StatusCode != 200 {
        health.Checks["external-api"] = "DOWN"
    } else {
        health.Checks["external-api"] = "UP"
    }
    
    json.NewEncoder(w).Encode(health)
}
```

## Your Task

1. Add Spring Actuator dependency
2. Configure health endpoint exposure
3. Create a custom health indicator
4. Check the health endpoints

## Step by Step

### 1. Add Dependencies (2 min)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

### 2. Configure Actuator (3 min)

In `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always  # Show component details
      show-components: always
  health:
    defaults:
      enabled: true
```

### 3. Test Default Health Endpoint (2 min)

```bash
./gradlew bootRun
curl http://localhost:8080/actuator/health
```

Response:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "H2",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 123456789012
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

### 4. Create Custom Health Indicator (8 min)

Create `ExternalApiHealthIndicator.java`:

```java
package com.example.demo;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;
    private final String externalUrl = "https://api.example.com/health";

    public ExternalApiHealthIndicator() {
        this.restTemplate = new RestTemplate();
    }

    @Override
    public Health health() {
        try {
            var response = restTemplate.getForEntity(externalUrl, String.class);
            
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("url", externalUrl)
                    .withDetail("status", response.getStatusCode().value())
                    .build();
            } else {
                return Health.down()
                    .withDetail("url", externalUrl)
                    .withDetail("status", response.getStatusCode().value())
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("url", externalUrl)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### 5. SQS Queue Health Check

```java
package com.example.demo;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.GetQueueAttributesRequest;

@Component
public class SqsHealthIndicator implements HealthIndicator {

    private final SqsClient sqsClient;
    private final String queueUrl;

    public SqsHealthIndicator(SqsClient sqsClient) {
        this.sqsClient = sqsClient;
        this.queueUrl = "http://localhost:4566/000000000000/task-events";
    }

    @Override
    public Health health() {
        try {
            var response = sqsClient.getQueueAttributes(
                GetQueueAttributesRequest.builder()
                    .queueUrl(queueUrl)
                    .attributeNamesWithStrings("ApproximateNumberOfMessages")
                    .build()
            );
            
            String messageCount = response.attributesAsStrings()
                .get("ApproximateNumberOfMessages");
            
            return Health.up()
                .withDetail("queue", queueUrl)
                .withDetail("approximateMessages", messageCount)
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withDetail("queue", queueUrl)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

## Test It

```bash
curl http://localhost:8080/actuator/health | jq
```

Response:

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "externalApi": { 
      "status": "DOWN",
      "details": {
        "error": "Connection refused"
      }
    },
    "sqs": {
      "status": "UP",
      "details": {
        "queue": "...",
        "approximateMessages": "0"
      }
    }
  }
}
```

## Go vs Spring Comparison

| Go | Spring Boot Actuator |
|----|---------------------|
| Manual `/health` handler | `/actuator/health` auto-configured |
| Manual component checks | `HealthIndicator` interface |
| Build JSON response | `Health.up()` / `Health.down()` builders |
| All-or-nothing status | Component-level status + aggregation |

## Actuator Endpoints Reference

| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | Application health |
| `/actuator/info` | Application info |
| `/actuator/metrics` | All metrics |
| `/actuator/metrics/{name}` | Specific metric |
| `/actuator/env` | Environment properties |
| `/actuator/loggers` | Logger levels (can modify) |
| `/actuator/beans` | All Spring beans |

## Health Status Aggregation

Overall status is the "worst" component status:

- All UP → Overall UP
- Any DOWN → Overall DOWN
- Any UNKNOWN → Overall UNKNOWN (if no DOWN)

## Liveness vs Readiness (Kubernetes)

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

Endpoints:
- `/actuator/health/liveness` - Is the app alive?
- `/actuator/health/readiness` - Can it accept traffic?

```java
// Mark as not ready during startup/shutdown
@Component
public class WarmupHealthIndicator implements HealthIndicator {
    
    private boolean ready = false;
    
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        ready = true;
    }
    
    @Override
    public Health health() {
        return ready ? Health.up().build() : Health.down().build();
    }
}
```

## Custom Health Groups

```yaml
management:
  endpoint:
    health:
      group:
        critical:
          include: db,sqs
        external:
          include: externalApi
```

Check specific group:
```bash
curl http://localhost:8080/actuator/health/critical
```

## Security Considerations

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info  # Only safe endpoints
  endpoint:
    health:
      show-details: when-authorized  # Require auth for details
```

## Checkpoint

- [ ] `/actuator/health` returns status
- [ ] Custom health indicator appears in response
- [ ] `show-details: always` shows component details
- [ ] DOWN status when dependency unavailable

**Next:** Exercise 18 - Metrics with Micrometer
