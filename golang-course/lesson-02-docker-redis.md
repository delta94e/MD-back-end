# Lesson 02 — Docker & Redis Integration

> **Mục tiêu:** Hiểu cách đóng gói app và tích hợp với external services (Redis)
> một cách chuyên nghiệp theo Go-Clean Architecture.

---

## Phần 1: Docker Fundamentals

### 1.1 Container vs VM

```
╔══════════════════════════════════════════════════════════════════════╗
║  VM                              CONTAINER                          ║
╠══════════════════════════════════════════════════════════════════════╣
║  ┌──────────────────┐            ┌──────────────────┐              ║
║  │   App A  │ App B │            │   App A  │ App B │              ║
║  ├──────────┼───────┤            ├──────────┼───────┤              ║
║  │  Guest OS│GuestOS│            │  Libs A  │ Libs B│              ║
║  ├──────────────────┤            ├──────────────────┤              ║
║  │   Hypervisor     │            │   Docker Engine  │              ║
║  ├──────────────────┤            ├──────────────────┤              ║
║  │    Host OS       │            │    Host OS       │              ║
║  └──────────────────┘            └──────────────────┘              ║
║                                                                      ║
║  - Kích thước: GB                - Kích thước: MB                   ║
║  - Khởi động: vài phút           - Khởi động: vài giây             ║
║  - Cô lập: toàn bộ OS            - Cô lập: process-level           ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 1.2 Khái Niệm Cơ Bản

```
Image     = "bản thiết kế" đóng băng (read-only template)
Container = Image đang chạy (running instance)
Dockerfile = Script để build Image
Volume    = Thư mục chia sẻ giữa container và host
Network   = Mạng nối các containers với nhau
Registry  = Kho lưu Image (Docker Hub, ECR, GCR)
```

### 1.3 Docker Commands Thiết Yếu

```bash
# === IMAGE ===
docker build -t myapp:1.0 .           # Build image từ Dockerfile
docker images                          # Liệt kê images
docker pull redis:7-alpine             # Tải image từ registry
docker push myapp:1.0                  # Đẩy lên registry
docker rmi myapp:1.0                   # Xóa image

# === CONTAINER ===
docker run -d \                        # Chạy container (detach)
  --name my-redis \
  -p 6379:6379 \                       # port host:container
  -v redis-data:/data \                # mount volume
  redis:7-alpine

docker ps                              # Liệt kê containers đang chạy
docker ps -a                           # Tất cả containers
docker logs my-redis                   # Xem logs
docker logs -f my-redis                # Follow logs (real-time)
docker exec -it my-redis sh            # Mở shell trong container
docker stop my-redis                   # Dừng container
docker rm my-redis                     # Xóa container
docker stats                           # Monitor resource usage

# === DỌN DẸP ===
docker system prune -f                 # Xóa tất cả unused resources
docker volume prune -f                 # Xóa volumes không dùng
```

---

## Phần 2: Docker Compose — Development Workflow

### 2.1 docker-compose.yml

```yaml
# docker-compose.yml — cho LOCAL DEVELOPMENT
version: "3.9"

services:
  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: app-postgres
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d  # tự động chạy SQL khi khởi tạo
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    container_name: app-redis
    command: >
      redis-server
      --requirepass redispass
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redispass", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # App (chạy khi cần full stack)
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev        # hot-reload với air
    container_name: app-server
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=appdb
      - DB_USER=appuser
      - DB_PASSWORD=apppass
      - REDIS_ADDR=redis:6379
      - REDIS_PASSWORD=redispass
    volumes:
      - .:/app                          # mount code để hot-reload
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  postgres-data:
  redis-data:
```

```bash
# === WORKFLOW HÀNG NGÀY ===
docker compose up -d postgres redis    # Chỉ khởi động infrastructure
docker compose up -d                   # Khởi động tất cả
docker compose logs -f app             # Xem log app
docker compose down                    # Dừng tất cả
docker compose down -v                 # Dừng + xóa volumes
docker compose restart redis           # Restart 1 service

# Chạy app trên host, infrastructure trên Docker (khuyến khích khi dev)
docker compose up -d postgres redis
go run ./cmd/server/main.go            # Dev với hot-reload tự nhiên hơn
```

---

## Phần 3: Redis Deep Dive

### 3.1 Redis Là Gì?

```
Redis = Remote Dictionary Server
- In-memory data structure store
- Single-threaded (1 command tại 1 thời điểm → KHÔNG có race condition)
- Độ trễ dưới millisecond (~0.1ms p99)
- Persistence: RDB snapshot + AOF append-only log
- Use cases: Cache, Session, Rate Limit, Queue, Pub/Sub, Leaderboard
```

### 3.2 Các Data Structures

```bash
# === STRING — cơ bản nhất ===
SET user:123:name "Alice"
GET user:123:name                      # "Alice"
INCR counter                           # atomic increment
SETNX lock:job "worker1"              # Set if Not eXists (distributed lock)
SET session:abc "data" EX 3600        # set với TTL 1 giờ
TTL session:abc                        # xem còn lại bao nhiêu giây
EXPIRE key 300                         # đặt TTL 5 phút
PERSIST key                            # xóa TTL

# === HASH — như struct/map ===
HSET user:123 name "Alice" email "alice@example.com" age 30
HGET user:123 name                     # "Alice"
HGETALL user:123                       # tất cả fields
HMGET user:123 name email              # nhiều fields một lúc
HDEL user:123 age                      # xóa field
HINCRBY user:123 loginCount 1         # increment field số

# === LIST — queue/stack ===
RPUSH queue:email "msg1" "msg2"       # thêm vào cuối (queue pattern)
LPUSH queue:email "urgent"             # thêm vào đầu
LPOP queue:email                       # lấy từ đầu
LLEN queue:email                       # độ dài
BRPOP queue:email 30                   # blocking pop, timeout 30 giây

# === SET — tập hợp không trùng lặp ===
SADD tags:post:1 "go" "redis" "docker"
SISMEMBER tags:post:1 "go"            # 1 (tồn tại)
SMEMBERS tags:post:1                   # tất cả members
SUNION tags:post:1 tags:post:2        # hợp 2 sets
SINTER tags:post:1 tags:post:2        # giao 2 sets

# === SORTED SET — leaderboard ===
ZADD leaderboard 1500 "alice"
ZADD leaderboard 2300 "bob"
ZRANK leaderboard "alice"             # vị trí (0-indexed, tăng dần)
ZREVRANK leaderboard "bob"            # vị trí ngược (giảm dần): 0
ZREVRANGE leaderboard 0 2 WITHSCORES  # top 3
ZINCRBY leaderboard 200 "alice"       # tăng score
```

### 3.3 Key Naming Convention

```bash
# Pattern: {namespace}:{id}:{field}
user:123                               # Hash của user 123
user:123:sessions                      # Set sessions của user 123
session:{token}                        # String session data
rate:limit:{ip}:{endpoint}            # Rate limit counter
lock:{resource}                        # Distributed lock
cache:product:{id}                     # Cached product
```

---

## Phần 4: Tích Hợp Redis Vào Go-Clean Architecture

### 4.1 Cấu Trúc Project Mở Rộng

```
internal/
  domain/
    repository/
      user_repository.go         # interface DB (không đổi)
      user_cache_repository.go   # interface cache (mới)
    usecase/
      user_usecase.go
  repository/
    postgres/
      user_repository.go
    redis/
      user_cache_repository.go   # Redis implementation (mới)
  usecase/
    user_usecase.go

pkg/
  cache/
    redis.go                     # Redis client wrapper
  ratelimit/
    redis_ratelimiter.go

mocks/
  user_repository_mock.go        # auto-generated bởi mockgen
  user_cache_mock.go             # auto-generated bởi mockgen
```

### 4.2 Redis Client Setup

```go
// pkg/cache/redis.go
package cache

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type Config struct {
    Addr     string
    Password string
    DB       int
}

func NewRedisClient(cfg Config) (*redis.Client, error) {
    client := redis.NewClient(&redis.Options{
        Addr:         cfg.Addr,
        Password:     cfg.Password,
        DB:           cfg.DB,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        PoolSize:     10,
        MinIdleConns: 3,
    })

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis ping: %w", err)
    }
    return client, nil
}
```

### 4.3 Domain Interface — Cache Layer

```go
// internal/domain/repository/user_cache_repository.go
package repository

import (
    "context"
    "errors"
    "time"

    "github.com/yourorg/app/internal/domain/entity"
)

// ErrCacheMiss — sentinel error khi không tìm thấy key trong cache
var ErrCacheMiss = errors.New("cache miss")

//go:generate mockgen -source=user_cache_repository.go -destination=../../../mocks/user_cache_mock.go -package=mocks
type UserCacheRepository interface {
    Get(ctx context.Context, id string) (*entity.User, error)
    Set(ctx context.Context, user *entity.User, ttl time.Duration) error
    Delete(ctx context.Context, id string) error
}
```

### 4.4 Redis Implementation

```go
// internal/repository/redis/user_cache_repository.go
package redis

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "time"

    goredis "github.com/redis/go-redis/v9"

    "github.com/yourorg/app/internal/domain/entity"
    "github.com/yourorg/app/internal/domain/repository"
)

type userCacheRepository struct {
    client *goredis.Client
}

func NewUserCacheRepository(client *goredis.Client) repository.UserCacheRepository {
    return &userCacheRepository{client: client}
}

func userKey(id string) string {
    return fmt.Sprintf("user:%s", id)
}

func (r *userCacheRepository) Get(ctx context.Context, id string) (*entity.User, error) {
    data, err := r.client.Get(ctx, userKey(id)).Bytes()
    if err != nil {
        if errors.Is(err, goredis.Nil) {
            return nil, repository.ErrCacheMiss // cache miss — bình thường
        }
        return nil, fmt.Errorf("redis Get %s: %w", id, err)
    }

    var user entity.User
    if err := json.Unmarshal(data, &user); err != nil {
        return nil, fmt.Errorf("redis unmarshal user %s: %w", id, err)
    }
    return &user, nil
}

func (r *userCacheRepository) Set(ctx context.Context, user *entity.User, ttl time.Duration) error {
    data, err := json.Marshal(user)
    if err != nil {
        return fmt.Errorf("redis marshal user: %w", err)
    }
    if err := r.client.Set(ctx, userKey(user.ID), data, ttl).Err(); err != nil {
        return fmt.Errorf("redis Set %s: %w", user.ID, err)
    }
    return nil
}

func (r *userCacheRepository) Delete(ctx context.Context, id string) error {
    if err := r.client.Del(ctx, userKey(id)).Err(); err != nil {
        return fmt.Errorf("redis Del %s: %w", id, err)
    }
    return nil
}

// Compile-time interface check
var _ repository.UserCacheRepository = (*userCacheRepository)(nil)
```

### 4.5 UseCase — Cache-Aside Pattern

```go
// internal/usecase/user_usecase.go
package usecase

import (
    "context"
    "errors"
    "fmt"
    "time"

    "go.uber.org/zap"

    "github.com/yourorg/app/internal/domain/entity"
    "github.com/yourorg/app/internal/domain/repository"
)

const userCacheTTL = 15 * time.Minute

type userUseCase struct {
    userRepo  repository.UserRepository
    userCache repository.UserCacheRepository
    logger    *zap.Logger
}

// GetByID — Cache-Aside pattern
// 1. Check cache → 2. Miss → DB → 3. Populate cache
func (uc *userUseCase) GetByID(ctx context.Context, id string) (*entity.User, error) {
    // Bước 1: Kiểm tra cache trước
    user, err := uc.userCache.Get(ctx, id)
    if err == nil {
        uc.logger.Debug("cache hit", zap.String("user_id", id))
        return user, nil // cache hit — trả về ngay
    }

    if !errors.Is(err, repository.ErrCacheMiss) {
        // Lỗi cache thật sự (network, timeout...) — log warning nhưng VẪN tiếp tục
        // Cache là optional: không để cache down làm sập cả API
        uc.logger.Warn("cache get error", zap.String("user_id", id), zap.Error(err))
    }

    // Bước 2: Đọc từ DB (source of truth)
    uc.logger.Debug("cache miss, querying db", zap.String("user_id", id))
    user, err = uc.userRepo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("GetByID: %w", err)
    }

    // Bước 3: Ghi vào cache (best-effort — không fail request nếu cache lỗi)
    if cacheErr := uc.userCache.Set(ctx, user, userCacheTTL); cacheErr != nil {
        uc.logger.Warn("cache set failed", zap.String("user_id", id), zap.Error(cacheErr))
    }

    return user, nil
}

// Update — invalidate cache sau khi update để tránh stale data
func (uc *userUseCase) Update(ctx context.Context, user *entity.User) (*entity.User, error) {
    updated, err := uc.userRepo.Update(ctx, user)
    if err != nil {
        return nil, fmt.Errorf("Update: %w", err)
    }

    // Xóa cache — lần đọc tiếp theo sẽ load lại từ DB
    if err := uc.userCache.Delete(ctx, user.ID); err != nil {
        uc.logger.Warn("cache delete failed after update",
            zap.String("user_id", user.ID), zap.Error(err))
    }

    return updated, nil
}
```

---

## Phần 5: Rate Limiting Với Redis

```go
// pkg/ratelimit/redis_ratelimiter.go
package ratelimit

import (
    "context"
    "fmt"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/redis/go-redis/v9"
)

type RateLimiter struct {
    client *redis.Client
    limit  int
    window time.Duration
}

func New(client *redis.Client, limit int, window time.Duration) *RateLimiter {
    return &RateLimiter{client: client, limit: limit, window: window}
}

// Allow kiểm tra request có được phép không (Fixed Window Counter)
func (r *RateLimiter) Allow(ctx context.Context, key string) (allowed bool, remaining int, err error) {
    fullKey := fmt.Sprintf("rate:%s", key)

    // Pipeline = 2 commands trong 1 round trip đến Redis
    pipe := r.client.TxPipeline()
    incr := pipe.Incr(ctx, fullKey)
    pipe.Expire(ctx, fullKey, r.window)
    if _, err = pipe.Exec(ctx); err != nil {
        return false, 0, fmt.Errorf("ratelimit pipeline: %w", err)
    }

    count := int(incr.Val())
    remaining = r.limit - count
    if remaining < 0 {
        remaining = 0
    }
    return count <= r.limit, remaining, nil
}

// Gin middleware
func Middleware(rl *RateLimiter) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := fmt.Sprintf("%s:%s", c.ClientIP(), c.FullPath())

        allowed, remaining, err := rl.Allow(c.Request.Context(), key)
        if err != nil {
            // Cache lỗi → fail open (cho qua để không block user)
            c.Next()
            return
        }

        c.Header("X-RateLimit-Remaining", fmt.Sprint(remaining))
        if !allowed {
            c.AbortWithStatusJSON(429, gin.H{
                "error":       "rate limit exceeded",
                "retry_after": rl.window.String(),
            })
            return
        }
        c.Next()
    }
}
```

---

## Phần 6: Testing — Unit Test + Integration Test

### 6.1 Setup mockgen

```bash
# Cài đặt mockgen
go install go.uber.org/mock/mockgen@latest

# Generate mock từ interface (có //go:generate trong file interface)
go generate ./...

# Hoặc chạy trực tiếp:
mockgen \
  -source=internal/domain/repository/user_cache_repository.go \
  -destination=mocks/user_cache_mock.go \
  -package=mocks
```

### 6.2 Unit Test — Dùng Mock

```go
// internal/usecase/user_usecase_test.go
package usecase_test

import (
    "context"
    "errors"
    "testing"
    "time"

    "go.uber.org/mock/gomock"
    "go.uber.org/zap/zaptest"

    "github.com/yourorg/app/internal/domain/entity"
    "github.com/yourorg/app/internal/domain/repository"
    "github.com/yourorg/app/internal/usecase"
    "github.com/yourorg/app/mocks"
)

func TestUserUseCase_GetByID(t *testing.T) {
    alice := &entity.User{ID: "123", Name: "Alice", Email: "alice@example.com"}

    tests := []struct {
        name      string
        setupMock func(repo *mocks.MockUserRepository, cache *mocks.MockUserCacheRepository)
        wantUser  *entity.User
        wantErr   bool
    }{
        {
            name: "cache hit — không gọi DB",
            setupMock: func(repo *mocks.MockUserRepository, cache *mocks.MockUserCacheRepository) {
                cache.EXPECT().Get(gomock.Any(), "123").Return(alice, nil)
                repo.EXPECT().FindByID(gomock.Any(), gomock.Any()).Times(0) // KHÔNG được gọi
            },
            wantUser: alice,
        },
        {
            name: "cache miss — đọc DB rồi populate cache",
            setupMock: func(repo *mocks.MockUserRepository, cache *mocks.MockUserCacheRepository) {
                cache.EXPECT().Get(gomock.Any(), "123").Return(nil, repository.ErrCacheMiss)
                repo.EXPECT().FindByID(gomock.Any(), "123").Return(alice, nil)
                cache.EXPECT().Set(gomock.Any(), alice, 15*time.Minute).Return(nil)
            },
            wantUser: alice,
        },
        {
            name: "cache miss — DB not found",
            setupMock: func(repo *mocks.MockUserRepository, cache *mocks.MockUserCacheRepository) {
                cache.EXPECT().Get(gomock.Any(), "123").Return(nil, repository.ErrCacheMiss)
                repo.EXPECT().FindByID(gomock.Any(), "123").Return(nil, repository.ErrNotFound)
            },
            wantErr: true,
        },
        {
            name: "cache error — vẫn fallback về DB (không fail request)",
            setupMock: func(repo *mocks.MockUserRepository, cache *mocks.MockUserCacheRepository) {
                cache.EXPECT().Get(gomock.Any(), "123").Return(nil, errors.New("redis: connection refused"))
                repo.EXPECT().FindByID(gomock.Any(), "123").Return(alice, nil)
                cache.EXPECT().Set(gomock.Any(), alice, gomock.Any()).Return(nil)
            },
            wantUser: alice, // Vẫn trả về user dù cache lỗi!
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            mockRepo  := mocks.NewMockUserRepository(ctrl)
            mockCache := mocks.NewMockUserCacheRepository(ctrl)
            tt.setupMock(mockRepo, mockCache)

            uc := usecase.NewUserUseCase(mockRepo, mockCache, zaptest.NewLogger(t))

            got, err := uc.GetByID(context.Background(), "123")
            if tt.wantErr {
                if err == nil {
                    t.Error("expected error, got nil")
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if got.ID != tt.wantUser.ID {
                t.Errorf("got user ID %s, want %s", got.ID, tt.wantUser.ID)
            }
        })
    }
}
```

### 6.3 Integration Test — Redis Thật

```go
// internal/repository/redis/user_cache_repository_integration_test.go
//go:build integration

package redis_test

import (
    "context"
    "errors"
    "testing"
    "time"

    goredis "github.com/redis/go-redis/v9"

    "github.com/yourorg/app/internal/domain/entity"
    "github.com/yourorg/app/internal/domain/repository"
    redisrepo "github.com/yourorg/app/internal/repository/redis"
)

func setupTestRedis(t *testing.T) *goredis.Client {
    t.Helper()
    client := goredis.NewClient(&goredis.Options{
        Addr: "localhost:6379", // cần: docker compose up -d redis
    })
    t.Cleanup(func() {
        client.FlushDB(context.Background()) // dọn dẹp sau mỗi test
        client.Close()
    })
    return client
}

func TestUserCacheRepository_Integration(t *testing.T) {
    client := setupTestRedis(t)
    repo   := redisrepo.NewUserCacheRepository(client)
    ctx    := context.Background()

    alice := &entity.User{ID: "alice-123", Name: "Alice", Email: "alice@example.com"}

    t.Run("Set và Get thành công", func(t *testing.T) {
        if err := repo.Set(ctx, alice, time.Minute); err != nil {
            t.Fatalf("Set: %v", err)
        }
        got, err := repo.Get(ctx, alice.ID)
        if err != nil {
            t.Fatalf("Get: %v", err)
        }
        if got.Name != alice.Name {
            t.Errorf("got %s, want %s", got.Name, alice.Name)
        }
    })

    t.Run("Get key không tồn tại → ErrCacheMiss", func(t *testing.T) {
        _, err := repo.Get(ctx, "nonexistent")
        if !errors.Is(err, repository.ErrCacheMiss) {
            t.Errorf("expected ErrCacheMiss, got %v", err)
        }
    })

    t.Run("Delete rồi Get → ErrCacheMiss", func(t *testing.T) {
        _ = repo.Set(ctx, alice, time.Minute)
        if err := repo.Delete(ctx, alice.ID); err != nil {
            t.Fatalf("Delete: %v", err)
        }
        _, err := repo.Get(ctx, alice.ID)
        if !errors.Is(err, repository.ErrCacheMiss) {
            t.Errorf("expected ErrCacheMiss after delete, got %v", err)
        }
    })

    t.Run("TTL hết hạn → ErrCacheMiss", func(t *testing.T) {
        if err := repo.Set(ctx, alice, 100*time.Millisecond); err != nil {
            t.Fatalf("Set: %v", err)
        }
        time.Sleep(200 * time.Millisecond)
        _, err := repo.Get(ctx, alice.ID)
        if !errors.Is(err, repository.ErrCacheMiss) {
            t.Errorf("expected cache to expire, got %v", err)
        }
    })
}
```

```bash
# Chạy unit tests (nhanh, không cần infrastructure)
go test -race ./...

# Chạy integration tests (cần docker compose up -d redis)
go test -v -tags=integration -race ./internal/repository/redis/...

# Chạy tất cả
go test -v -tags=integration -race ./...
```

---

## Phần 7: Đóng Gói App Với Docker

### 7.1 Dockerfile — Multi-Stage Build

```dockerfile
# Dockerfile
# ─── Stage 1: Build binary ──────────────────────────────────────────
FROM golang:1.22-alpine AS builder

RUN apk add --no-cache gcc musl-dev git

WORKDIR /app

# Cache dependencies trước (tận dụng Docker layer cache)
COPY go.mod go.sum ./
RUN go mod download

# Build binary tĩnh (CGO_ENABLED=0 → không phụ thuộc libc)
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s -X main.version=$(git describe --tags --always)" \
    -o /bin/server ./cmd/server/main.go

# ─── Stage 2: Runtime image cực nhỏ ────────────────────────────────
FROM alpine:3.19

# Chứng chỉ SSL và timezone data
RUN apk add --no-cache ca-certificates tzdata && \
    adduser -D -g '' appuser           # tạo non-root user

WORKDIR /app

# Chỉ copy binary từ stage builder (không copy source code!)
COPY --from=builder /bin/server .

# Chạy với non-root user — bảo mật tốt hơn
USER appuser

EXPOSE 8080

# Health check tự động
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q -O- http://localhost:8080/health || exit 1

ENTRYPOINT ["./server"]
```

```bash
# Build và kiểm tra kích thước
docker build -t myapp:latest .
docker images myapp:latest
# → ~15-20 MB thay vì ~1 GB nếu dùng golang image trực tiếp
```

### 7.2 docker-compose.prod.yml

```yaml
# docker-compose.prod.yml
version: "3.9"

services:
  app:
    image: myapp:${APP_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "8080:8080"
    env_file:
      - .env.prod                       # secrets không commit vào git
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      replicas: 2                       # chạy 2 instances song song
      resources:
        limits:
          cpus: "0.5"
          memory: 256M

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
    env_file:
      - .env.prod
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres-data:
  redis-data:
```

### 7.3 Makefile — Tích Hợp Docker

```makefile
APP_NAME  = myapp
VERSION   = $(shell git describe --tags --always --dirty)
REGISTRY  = your-registry.io

.PHONY: docker-build docker-push infra-up infra-down test test-integration

# Khởi động infrastructure (postgres + redis) cho local dev
infra-up:
	docker compose up -d postgres redis

infra-down:
	docker compose down

# Build Docker image
docker-build:
	docker build -t $(APP_NAME):$(VERSION) -t $(APP_NAME):latest .
	@echo "Image size:"
	@docker images $(APP_NAME):latest --format "{{.Size}}"

# Đẩy lên registry
docker-push: docker-build
	docker tag $(APP_NAME):$(VERSION) $(REGISTRY)/$(APP_NAME):$(VERSION)
	docker push $(REGISTRY)/$(APP_NAME):$(VERSION)

# Unit tests
test:
	go test -race -cover ./...

# Integration tests (cần infra đang chạy)
test-integration: infra-up
	sleep 3  # chờ services healthy
	go test -v -tags=integration -race ./...

# Chạy toàn bộ CI pipeline
ci: test docker-build
```

---

## Phần 8: Bài Tập

### Bài 1 — Session Management Với Redis
Implement hệ thống session cho API:
- `POST /auth/login` → tạo token, lưu vào Redis (`session:{token}` với TTL 24h)
- `GET /auth/me` → đọc user từ session token (Authorization header)
- `POST /auth/logout` → xóa session khỏi Redis
- Viết unit test đầy đủ với mock Redis

### Bài 2 — Rate Limit Theo User
Mở rộng Rate Limiter:
- Authenticated users: 100 req/phút
- Anonymous users: 10 req/phút
- Key pattern: `rate:{user_id}` hoặc `rate:{ip}`
- Trả về headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

### Bài 3 — Leaderboard Với Sorted Set
Implement leaderboard API:
- `POST /scores` → cập nhật score (`ZADD`)
- `GET /leaderboard?top=10` → top 10 (`ZREVRANGE WITHSCORES`)
- `GET /leaderboard/rank/{user_id}` → xếp hạng của user (`ZREVRANK`)
- Viết integration test với Redis thật

### Bài 4 — Đóng Gói App Hoàn Chỉnh
- Viết `Dockerfile` multi-stage, kiểm tra image size < 30MB
- Thêm `HEALTHCHECK` endpoint `/health` vào app
- `docker-compose.yml` đầy đủ: postgres + redis + app với healthcheck
- Thêm `make infra-up`, `make docker-build` vào Makefile
- Commit tất cả vào repo

---

## Tổng Kết Lesson 02

```
╔══════════════════════════════════════════════════════════════════════╗
║  LESSON 02 — KEY TAKEAWAYS                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Docker:                                                            ║
║  + Container = nhẹ, nhanh, nhất quán giữa mọi môi trường          ║
║  + Multi-stage build → image nhỏ (~15MB vs ~1GB)                   ║
║  + docker-compose → quản lý nhiều services dễ dàng                 ║
║  + Non-root user + HEALTHCHECK → production-ready                   ║
║                                                                      ║
║  Redis:                                                             ║
║  + In-memory → độ trễ dưới millisecond                             ║
║  + 5 data structures phù hợp với nhiều use case khác nhau          ║
║  + TTL → tự động expiry, không cần cleanup job                     ║
║  + Pipeline → giảm round trips, tăng throughput                    ║
║                                                                      ║
║  Clean Architecture + Redis:                                        ║
║  + Tách interface (domain) khỏi implementation (redis package)     ║
║  + Cache-Aside: check cache → DB → populate cache                  ║
║  + Cache là OPTIONAL: lỗi cache không được fail request            ║
║  + Invalidate cache ngay sau khi data thay đổi                     ║
║                                                                      ║
║  Testing:                                                           ║
║  + mockgen tự động sinh mock từ interface → không viết tay         ║
║  + Unit test: mock tất cả dependencies, test từng trường hợp       ║
║  + Integration test: -tags=integration, dùng Redis thật            ║
║  + -race flag: phát hiện race condition trong concurrency           ║
╚══════════════════════════════════════════════════════════════════════╝
```
