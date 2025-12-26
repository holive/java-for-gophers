# Exercise 15: S3 Operations

**Time:** 15 minutes  
**Goal:** Upload and download files using Spring Cloud AWS S3

## The Go Version

```go
func (s *S3Client) Upload(ctx context.Context, bucket, key string, data []byte) error {
    _, err := s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
        Body:   bytes.NewReader(data),
    })
    return err
}

func (s *S3Client) Download(ctx context.Context, bucket, key string) ([]byte, error) {
    output, err := s.client.GetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
    })
    if err != nil {
        return nil, err
    }
    defer output.Body.Close()
    
    return io.ReadAll(output.Body)
}
```

## Your Task

1. Create an S3 bucket in LocalStack
2. Implement upload/download with `S3Template`
3. Create a file storage service
4. Add a controller for file operations

## Prerequisites

Create S3 bucket in LocalStack:

```bash
aws --endpoint-url=http://localhost:4566 s3 mb s3://task-attachments
```

---

## Step by Step

### 1. Add Dependencies (2 min)

```groovy
dependencies {
    implementation 'io.awspring.cloud:spring-cloud-aws-starter-s3'
}
```

### 2. Configure S3 (2 min)

In `application.yml`:

```yaml
spring:
  cloud:
    aws:
      s3:
        endpoint: http://localhost:4566
        path-style-access-enabled: true  # Required for LocalStack

app:
  s3:
    bucket: task-attachments
```

### 3. Create Storage Service (6 min)

Create `FileStorageService.java`:

```java
package com.example.demo;

import io.awspring.cloud.s3.S3Resource;
import io.awspring.cloud.s3.S3Template;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

@Service
public class FileStorageService {

    private static final Logger log = LoggerFactory.getLogger(FileStorageService.class);
    
    private final S3Template s3Template;
    private final String bucket;

    public FileStorageService(
            S3Template s3Template,
            @Value("${app.s3.bucket}") String bucket) {
        this.s3Template = s3Template;
        this.bucket = bucket;
    }

    public String upload(String key, byte[] content, String contentType) {
        log.info("Uploading {} to bucket {}", key, bucket);
        
        s3Template.upload(bucket, key, new ByteArrayInputStream(content));
        
        log.info("Upload complete: {}", key);
        return key;
    }

    public byte[] download(String key) throws IOException {
        log.info("Downloading {} from bucket {}", key, bucket);
        
        S3Resource resource = s3Template.download(bucket, key);
        
        try (InputStream is = resource.getInputStream()) {
            return is.readAllBytes();
        }
    }

    public void delete(String key) {
        log.info("Deleting {} from bucket {}", key, bucket);
        s3Template.deleteObject(bucket, key);
    }

    public boolean exists(String key) {
        return s3Template.objectExists(bucket, key);
    }
}
```

### 4. Create File Controller (5 min)

Create `FileController.java`:

```java
package com.example.demo;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.Map;
import java.util.UUID;

@RestController
@RequestMapping("/files")
public class FileController {

    private final FileStorageService storageService;

    public FileController(FileStorageService storageService) {
        this.storageService = storageService;
    }

    @PostMapping
    public Map<String, String> upload(@RequestParam("file") MultipartFile file) throws IOException {
        String key = UUID.randomUUID() + "-" + file.getOriginalFilename();
        
        storageService.upload(key, file.getBytes(), file.getContentType());
        
        return Map.of(
            "key", key,
            "message", "File uploaded successfully"
        );
    }

    @GetMapping("/{key}")
    public ResponseEntity<byte[]> download(@PathVariable String key) throws IOException {
        if (!storageService.exists(key)) {
            return ResponseEntity.notFound().build();
        }
        
        byte[] content = storageService.download(key);
        
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + key + "\"")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(content);
    }

    @DeleteMapping("/{key}")
    public ResponseEntity<Void> delete(@PathVariable String key) {
        if (!storageService.exists(key)) {
            return ResponseEntity.notFound().build();
        }
        
        storageService.delete(key);
        return ResponseEntity.noContent().build();
    }
}
```

## Test It

```bash
# Upload a file
curl -X POST http://localhost:8080/files \
  -F "file=@/path/to/test.txt"

# Download (use key from upload response)
curl http://localhost:8080/files/{key} -o downloaded.txt

# Delete
curl -X DELETE http://localhost:8080/files/{key}

# Verify in LocalStack
aws --endpoint-url=http://localhost:4566 s3 ls s3://task-attachments/
```

## Go vs Spring Comparison

| Go | Spring Cloud AWS |
|----|------------------|
| `s3.PutObject(ctx, input)` | `s3Template.upload(bucket, key, stream)` |
| `s3.GetObject(ctx, input)` | `s3Template.download(bucket, key)` |
| `io.ReadAll(output.Body)` | `resource.getInputStream()` |
| Manual presigned URL generation | `s3Template.createSignedGetURL()` |

## Advanced Operations

### Upload with Metadata

```java
public void uploadWithMetadata(String key, byte[] content, Map<String, String> metadata) {
    s3Template.upload(bucket, key, new ByteArrayInputStream(content),
        ObjectMetadata.builder()
            .contentType("application/json")
            .metadata(metadata)
            .build());
}
```

### Presigned URLs

For direct client uploads/downloads:

```java
public String getPresignedDownloadUrl(String key, Duration expiry) {
    return s3Template.createSignedGetURL(bucket, key, expiry).toString();
}

public String getPresignedUploadUrl(String key, Duration expiry) {
    return s3Template.createSignedPutURL(bucket, key, expiry).toString();
}
```

### List Objects

```java
public List<String> listFiles(String prefix) {
    return s3Template.listObjects(bucket, prefix).stream()
        .map(S3Resource::getFilename)
        .toList();
}
```

### Copy Objects

```java
public void copyFile(String sourceKey, String destKey) {
    s3Template.copy(bucket, sourceKey, bucket, destKey);
}
```

## Task Attachment Pattern

Link S3 files to tasks:

```java
@Entity
public class Task {
    // ...
    
    @ElementCollection
    private List<String> attachmentKeys = new ArrayList<>();
    
    public void addAttachment(String key) {
        attachmentKeys.add(key);
    }
}

@Service
public class TaskService {
    
    public void addAttachment(String taskId, MultipartFile file) throws IOException {
        Task task = findById(taskId);
        
        String key = "tasks/" + taskId + "/" + file.getOriginalFilename();
        storageService.upload(key, file.getBytes(), file.getContentType());
        
        task.addAttachment(key);
        repository.save(task);
    }
}
```

## Error Handling

```java
public byte[] downloadSafe(String key) {
    try {
        return download(key);
    } catch (S3Exception e) {
        if (e.statusCode() == 404) {
            throw new FileNotFoundException(key);
        }
        throw new StorageException("Failed to download: " + key, e);
    } catch (IOException e) {
        throw new StorageException("IO error downloading: " + key, e);
    }
}
```

## Multipart Upload (Large Files)

For files > 5GB:

```java
// S3Template handles this automatically for large files
// Just use the normal upload method

// Or manually for streaming:
public void uploadLargeFile(String key, InputStream stream, long size) {
    s3Template.upload(bucket, key, stream,
        ObjectMetadata.builder()
            .contentLength(size)
            .build());
}
```

## Checkpoint

- [ ] S3 bucket exists in LocalStack
- [ ] Upload stores file and returns key
- [ ] Download retrieves file content
- [ ] Delete removes file from S3

**Next:** Exercise 16 - DynamoDB with @DynamoDbBean
