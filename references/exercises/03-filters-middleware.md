# Exercise 03: Middleware (Filters in Spring)

**Time:** 15 minutes  
**Goal:** Add request logging and timing middleware

## The Go Version

```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        requestID := uuid.New().String()
        
        log.Printf("[%s] Started %s %s", requestID, r.Method, r.URL.Path)
        
        // Add request ID to context for downstream handlers
        ctx := context.WithValue(r.Context(), "requestID", requestID)
        r = r.WithContext(ctx)
        
        next.ServeHTTP(w, r)
        
        log.Printf("[%s] Completed in %v", requestID, time.Since(start))
    })
}

// Usage
r := chi.NewRouter()
r.Use(LoggingMiddleware)
```

## Your Task

Create a filter that:
1. Generates a unique request ID
2. Logs the start of each request
3. Adds the request ID to the response headers
4. Logs completion time after the request finishes

## Step by Step

### 1. Create the Filter (10 min)

Create `RequestLoggingFilter.java`:

```java
package com.example.demo;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
    
    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        
        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString().substring(0, 8);
        
        // Add request ID to response header
        response.setHeader("X-Request-ID", requestId);
        
        // Store in request attributes (like Go's context)
        request.setAttribute("requestId", requestId);
        
        log.info("[{}] Started {} {}", 
            requestId, 
            request.getMethod(), 
            request.getRequestURI());
        
        try {
            // This is the "next.ServeHTTP(w, r)" equivalent
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("[{}] Completed {} {} - {} in {}ms",
                requestId,
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration);
        }
    }
}
```

### 2. Access Request ID in Controller (3 min)

You can access the request ID in any controller:

```java
@GetMapping("/hello")
public HelloResponse hello(HttpServletRequest request) {
    String requestId = (String) request.getAttribute("requestId");
    log.info("[{}] Processing hello request", requestId);
    return new HelloResponse("Hello from Spring Boot");
}
```

### 3. Test It (2 min)

```bash
curl -v http://localhost:8080/hello
```

Check:
- Response includes `X-Request-ID` header
- Console shows log messages with the request ID

## Alternative: HandlerInterceptor

Filters work at the servlet level. For Spring-specific features, use `HandlerInterceptor`:

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        // Before controller
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;  // Continue processing
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler,
                               Exception ex) {
        // After response sent
        long start = (Long) request.getAttribute("startTime");
        log.info("Request completed in {}ms", System.currentTimeMillis() - start);
    }
}

// Register it
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private LoggingInterceptor loggingInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor);
    }
}
```

## Go vs Spring Comparison

| Go | Spring Boot |
|----|-------------|
| Middleware wraps handler | Filter wraps servlet chain |
| `context.WithValue()` | `request.setAttribute()` |
| `next.ServeHTTP(w, r)` | `filterChain.doFilter(req, res)` |
| Manual registration via `r.Use()` | Auto via `@Component` (or explicit order) |

## Controlling Filter Order

Multiple filters execute in order. Control with `@Order`:

```java
@Component
@Order(1)  // Lower = earlier
public class AuthFilter extends OncePerRequestFilter { ... }

@Component  
@Order(2)
public class LoggingFilter extends OncePerRequestFilter { ... }
```

Or register explicitly:

```java
@Configuration
public class FilterConfig {
    
    @Bean
    public FilterRegistrationBean<RequestLoggingFilter> loggingFilter() {
        var registration = new FilterRegistrationBean<>(new RequestLoggingFilter());
        registration.setOrder(1);
        registration.addUrlPatterns("/*");
        return registration;
    }
}
```

## When to Use Each

| Use Case | Filter | Interceptor |
|----------|--------|-------------|
| Logging/timing | ✅ | ✅ |
| Authentication | ✅ | ✅ |
| Request modification | ✅ | ❌ |
| Access to Spring beans | Limited | ✅ |
| Access to handler method info | ❌ | ✅ |

## Checkpoint

✅ Requests show start/completion logs  
✅ Response has X-Request-ID header  
✅ Request ID visible in controller (optional)  

**Next:** Exercise 04 — Service layer and dependency injection
