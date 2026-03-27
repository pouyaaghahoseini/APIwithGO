# Unit 10: Deployment & Production

**Duration**: 90-120 minutes  
**Prerequisites**: Units 1-9 (Complete API development and testing knowledge)  
**Goal**: Deploy your API to production with Docker, Kubernetes, monitoring, and logging

---

## 10.1 Why Production Readiness Matters

### The Problem: Development vs Production

```
Development: Works on my machine! ✅
Production: 
  - Server crashes at 2 AM 💥
  - No logs to debug 🤷
  - Can't handle traffic spike 📈
  - Database connection pool exhausted 💀
  - Security vulnerabilities exposed 🔓
```

**Consequences**:
- 💸 Revenue loss during downtime
- 😡 Angry users
- 🚨 PagerDuty alerts at 3 AM
- 📉 Damaged reputation
- 🔥 Team burnout

### The Solution: Production-Ready Deployment

With proper deployment:
```
✅ Containerized for consistency
✅ Orchestrated for scaling
✅ Monitored for observability
✅ Logged for debugging
✅ Secured with best practices
✅ Automated deployments
✅ Health checks and recovery
```

---

## 10.2 The 12-Factor App Principles

### Key Principles for Cloud-Native Apps

1. **Codebase**: One codebase tracked in Git
2. **Dependencies**: Explicitly declare dependencies (go.mod)
3. **Config**: Store config in environment variables
4. **Backing Services**: Treat as attached resources
5. **Build, Release, Run**: Strict separation
6. **Processes**: Execute as stateless processes
7. **Port Binding**: Export services via port binding
8. **Concurrency**: Scale out via the process model
9. **Disposability**: Fast startup and graceful shutdown
10. **Dev/Prod Parity**: Keep dev and prod similar
11. **Logs**: Treat logs as event streams
12. **Admin Processes**: Run admin tasks as one-off processes

---

## 10.3 Environment Configuration

### Using Environment Variables

```go
package config

import (
    "fmt"
    "os"
    "strconv"
)

type Config struct {
    // Server
    Port         string
    Environment  string
    
    // Database
    DBHost       string
    DBPort       string
    DBUser       string
    DBPassword   string
    DBName       string
    DBSSLMode    string
    
    // Redis
    RedisURL     string
    
    // Security
    JWTSecret    string
    
    // External Services
    APIKey       string
}

func LoadConfig() (*Config, error) {
    cfg := &Config{
        Port:        getEnv("PORT", "8080"),
        Environment: getEnv("ENVIRONMENT", "development"),
        
        DBHost:      getEnv("DB_HOST", "localhost"),
        DBPort:      getEnv("DB_PORT", "5432"),
        DBUser:      getEnv("DB_USER", "postgres"),
        DBPassword:  getEnv("DB_PASSWORD", ""),
        DBName:      getEnv("DB_NAME", "myapp"),
        DBSSLMode:   getEnv("DB_SSLMODE", "disable"),
        
        RedisURL:    getEnv("REDIS_URL", "redis://localhost:6379"),
        
        JWTSecret:   getEnv("JWT_SECRET", ""),
    }
    
    // Validate required fields
    if cfg.Environment == "production" {
        if cfg.JWTSecret == "" {
            return nil, fmt.Errorf("JWT_SECRET is required in production")
        }
        if cfg.DBPassword == "" {
            return nil, fmt.Errorf("DB_PASSWORD is required in production")
        }
    }
    
    return cfg, nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intVal, err := strconv.Atoi(value); err == nil {
            return intVal
        }
    }
    return defaultValue
}

func (c *Config) DatabaseURL() string {
    return fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s?sslmode=%s",
        c.DBUser,
        c.DBPassword,
        c.DBHost,
        c.DBPort,
        c.DBName,
        c.DBSSLMode,
    )
}
```

### .env File for Development

```bash
# .env
PORT=8080
ENVIRONMENT=development

DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=myapp_dev
DB_SSLMODE=disable

REDIS_URL=redis://localhost:6379

JWT_SECRET=dev-secret-change-in-production
```

---

## 10.4 Containerization with Docker

### Dockerfile

```dockerfile
# Multi-stage build for smaller image

# Stage 1: Build
FROM golang:1.21-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git gcc musl-dev

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Runtime
FROM alpine:latest

# Install ca-certificates for HTTPS
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy binary from builder
COPY --from=builder /app/main .

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1

# Run
CMD ["./main"]
```

### .dockerignore

```
# .dockerignore
.git
.env
*.md
Dockerfile
docker-compose.yml
coverage.out
*.test
tmp/
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USER=postgres
      - DB_PASSWORD=password
      - DB_NAME=myapp
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
  
  postgres:
    image: postgres:14-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

---

## 10.5 Graceful Shutdown

### Handling Termination Signals

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Create server
    srv := &http.Server{
        Addr:    ":8080",
        Handler: createRouter(),
    }
    
    // Start server in goroutine
    go func() {
        fmt.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            fmt.Printf("Server error: %v\n", err)
            os.Exit(1)
        }
    }()
    
    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    fmt.Println("Server shutting down...")
    
    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        fmt.Printf("Server forced to shutdown: %v\n", err)
    }
    
    fmt.Println("Server exited")
}
```

---

## 10.6 Health Checks

### Health and Readiness Endpoints

```go
type HealthChecker struct {
    db    *sql.DB
    redis *redis.Client
}

func (h *HealthChecker) HealthHandler(w http.ResponseWriter, r *http.Request) {
    // Simple health check - is server running?
    respondJSON(w, http.StatusOK, map[string]string{
        "status": "ok",
    })
}

func (h *HealthChecker) ReadinessHandler(w http.ResponseWriter, r *http.Request) {
    // Readiness check - can server handle requests?
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()
    
    status := map[string]interface{}{
        "status": "ready",
        "checks": map[string]string{},
    }
    
    // Check database
    if err := h.db.PingContext(ctx); err != nil {
        status["status"] = "not ready"
        status["checks"].(map[string]string)["database"] = "unhealthy"
        respondJSON(w, http.StatusServiceUnavailable, status)
        return
    }
    status["checks"].(map[string]string)["database"] = "healthy"
    
    // Check Redis
    if _, err := h.redis.Ping(ctx).Result(); err != nil {
        status["status"] = "not ready"
        status["checks"].(map[string]string)["redis"] = "unhealthy"
        respondJSON(w, http.StatusServiceUnavailable, status)
        return
    }
    status["checks"].(map[string]string)["redis"] = "healthy"
    
    respondJSON(w, http.StatusOK, status)
}

func (h *HealthChecker) LivenessHandler(w http.ResponseWriter, r *http.Request) {
    // Liveness check - should container be restarted?
    // Check for deadlocks, goroutine leaks, etc.
    
    respondJSON(w, http.StatusOK, map[string]string{
        "status": "alive",
    })
}
```

---

## 10.7 Structured Logging

### Using Zerolog

```go
package main

import (
    "os"
    "time"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func setupLogger(environment string) {
    // Configure based on environment
    if environment == "production" {
        // JSON logging for production
        zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
    } else {
        // Pretty logging for development
        log.Logger = log.Output(zerolog.ConsoleWriter{
            Out:        os.Stdout,
            TimeFormat: time.RFC3339,
        })
    }
    
    // Set global log level
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
    if environment == "development" {
        zerolog.SetGlobalLevel(zerolog.DebugLevel)
    }
}

// HTTP request logging middleware
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Create response writer wrapper to capture status
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        
        // Process request
        next.ServeHTTP(wrapped, r)
        
        // Log request
        duration := time.Since(start)
        
        log.Info().
            Str("method", r.Method).
            Str("path", r.URL.Path).
            Str("remote_addr", r.RemoteAddr).
            Int("status", wrapped.statusCode).
            Dur("duration_ms", duration).
            Msg("HTTP request")
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

---

## 10.8 Monitoring with Prometheus

### Metrics Collection

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    HttpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    HttpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
    
    DatabaseConnectionsActive = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "database_connections_active",
            Help: "Number of active database connections",
        },
    )
    
    CacheHitsTotal = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "cache_hits_total",
            Help: "Total number of cache hits",
        },
    )
    
    CacheMissesTotal = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "cache_misses_total",
            Help: "Total number of cache misses",
        },
    )
)

// Middleware to record metrics
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        wrapped := &metricsResponseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start).Seconds()
        
        // Record metrics
        HttpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            strconv.Itoa(wrapped.statusCode),
        ).Inc()
        
        HttpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}
```

### Exposing Metrics Endpoint

```go
import (
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    r := mux.NewRouter()
    
    // Metrics endpoint
    r.Handle("/metrics", promhttp.Handler())
    
    // Your API routes
    r.HandleFunc("/api/users", getUsers)
    
    http.ListenAndServe(":8080", r)
}
```

---

## 10.9 Kubernetes Deployment

### Kubernetes Manifests

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  labels:
    app: myapp-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
    spec:
      containers:
      - name: api
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: ENVIRONMENT
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**service.yaml**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
spec:
  selector:
    app: myapp-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

**secrets.yaml** (base64 encoded):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
data:
  db-host: <base64-encoded>
  db-password: <base64-encoded>
  jwt-secret: <base64-encoded>
```

---

## 10.10 Database Migrations

### Using golang-migrate

```go
package database

import (
    "database/sql"
    "fmt"

    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(db *sql.DB, migrationsPath string) error {
    driver, err := postgres.WithInstance(db, &postgres.Config{})
    if err != nil {
        return fmt.Errorf("could not create migrate driver: %w", err)
    }
    
    m, err := migrate.NewWithDatabaseInstance(
        migrationsPath,
        "postgres",
        driver,
    )
    if err != nil {
        return fmt.Errorf("could not create migrate instance: %w", err)
    }
    
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("migration failed: %w", err)
    }
    
    return nil
}
```

**migrations/000001_create_users_table.up.sql**:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

**migrations/000001_create_users_table.down.sql**:
```sql
DROP TABLE IF EXISTS users;
```

---

## 10.11 CI/CD Pipeline

### Complete GitHub Actions Workflow

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v3
      with:
        go-version: '1.21'
    
    - name: Run tests
      run: |
        go test -v -cover ./...
        go test -v -tags=integration ./...
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/testdb?sslmode=disable
  
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
  
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v4
      with:
        manifests: |
          k8s/deployment.yaml
          k8s/service.yaml
        images: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

---

## 10.12 Production Best Practices

### ✅ Security Checklist

- [ ] Use HTTPS everywhere
- [ ] Secure cookies (HttpOnly, Secure, SameSite)
- [ ] Environment variables for secrets
- [ ] Rate limiting enabled
- [ ] Input validation
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS protection
- [ ] CORS configured properly
- [ ] Helmet/security headers
- [ ] Regular dependency updates

### ✅ Performance Checklist

- [ ] Database connection pooling
- [ ] Redis caching implemented
- [ ] Gzip compression enabled
- [ ] Static asset CDN
- [ ] Database indexes optimized
- [ ] Query optimization
- [ ] Pagination for large datasets
- [ ] Background jobs for long tasks

### ✅ Reliability Checklist

- [ ] Health checks configured
- [ ] Graceful shutdown
- [ ] Circuit breakers for external services
- [ ] Retry logic with exponential backoff
- [ ] Database migrations automated
- [ ] Horizontal scaling configured
- [ ] Load balancer set up
- [ ] Backup strategy

### ✅ Observability Checklist

- [ ] Structured logging
- [ ] Prometheus metrics
- [ ] Distributed tracing (Jaeger)
- [ ] Error tracking (Sentry)
- [ ] Dashboards (Grafana)
- [ ] Alerting rules
- [ ] Log aggregation (ELK/Loki)

---

## Key Takeaways

✅ **12-Factor App** principles for cloud-native  
✅ **Environment variables** for configuration  
✅ **Docker** for containerization  
✅ **Graceful shutdown** for zero-downtime  
✅ **Health checks** for orchestration  
✅ **Structured logging** for debugging  
✅ **Prometheus metrics** for monitoring  
✅ **Kubernetes** for orchestration  
✅ **Database migrations** automated  
✅ **CI/CD** for automated deployment  

---

## Congratulations! 🎉

You've completed the **Complete Go API Development Course**!

You now know how to:
- Build production-grade APIs in Go
- Implement authentication and authorization
- Version and document APIs
- Optimize with caching and pagination
- Protect with rate limiting
- Test comprehensively
- **Deploy to production** ← You are here

**You're ready to build and deploy world-class APIs!** 🚀
