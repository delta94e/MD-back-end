# Lesson 01 — Xây Dựng & Cấu Trúc một API Application Chuyên Nghiệp

> **Mục tiêu bài học:** Nắm rõ cách xây dựng và cấu trúc project của mình một cách chuyên nghiệp, chỉnh chu theo tiêu chuẩn của các công ty production.
>
> **Framework minh họa:** Go-Gin | **Kiến trúc:** Go-Clean (Hexagonal) | **Đối tượng:** Backend Engineer hướng Senior

---

## Bản Đồ Tổng Quan — Mental Map

```
╔══════════════════════════════════════════════════════════════════════╗
║            LESSON 01 — PRODUCTION-GRADE GO API                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. Khởi tạo API Application (Go-Gin)                               ║
║     └─► Go Modules → Framework → Project scaffold                   ║
║                                                                      ║
║  2. Go-Clean (Hexagonal) Architecture                               ║
║     └─► Domain → Use Case → Infrastructure → Adapter               ║
║                                                                      ║
║  3. Viết API đầu tiên theo chuẩn Clean Arch                        ║
║     └─► Handler → UseCase → Repository → DB                        ║
║                                                                      ║
║  4. Unit Test theo chuẩn Clean Architecture                         ║
║     └─► Mock → Table-driven → Coverage                              ║
║                                                                      ║
║  5. App Config với Environment Variables                             ║
║     └─► 12-Factor App → viper / envconfig                          ║
║                                                                      ║
║  6. API Documentation (OpenAPI / Swagger)                           ║
║     └─► swaggo/swag → generated docs                               ║
║                                                                      ║
║  7. Makefile — Tự động hóa mọi thứ                                 ║
║     └─► build, test, lint, migrate, docs                            ║
║                                                                      ║
║  8. Go Test Commands & Code Coverage                                 ║
║     └─► go test flags → coverage reports → CI gate                 ║
║                                                                      ║
║  9. Agile trong Khóa Học                                            ║
║     └─► Sprint → Story → Task → Review                              ║
║                                                                      ║
║  10. Bài Tập — Implement tính năng mới                              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## §1. Khởi Tạo API Application

### Tại sao Go-Gin? Và tại sao KHÔNG bị phụ thuộc vào nó?

```
╔══════════════════════════════════════════════════════════════════════╗
║  "Framework là công CỤ, không phải kiến TRÚC."                      ║
║                                                                      ║
║  Mục tiêu khóa học: bạn có thể swap Gin sang Fiber, Echo, hay      ║
║  thậm chí net/http thuần túy mà KHÔNG cần rewrite business logic.  ║
║  Đây là dấu hiệu của một clean architecture THỰC SỰ.               ║
╚══════════════════════════════════════════════════════════════════════╝
```

**So sánh các framework phổ biến:**

| Framework | Stars | Đặc điểm | Khi nào dùng |
|-----------|-------|-----------|--------------|
| `net/http` | stdlib | Không dependency, low-level | Study, micro-services nhỏ |
| **Gin** | ~80k | Nhanh, middleware phong phú, cộng đồng lớn | Production API (khuyến nghị) |
| Fiber | ~35k | Express.js-like, rất nhanh (fasthttp) | High-throughput, HTTP/1.1 |
| Echo | ~30k | Clean API, middleware tốt | Thay thế Gin |
| Chi | ~18k | Stdlib-compatible, lightweight | Ưa thích stdlib |

> **Triết lý khóa học:** Chúng ta dùng Gin để minh họa, nhưng mọi concept (clean arch, testing, config, docs) đều **framework-agnostic**. Bạn học được tư duy, không học thuộc framework.

### Khởi Tạo Project — Từng Bước Chi Tiết

```bash
# Bước 1: Tạo thư mục project
mkdir go-api-boilerplate && cd go-api-boilerplate

# Bước 2: Khởi tạo Go Module
# Format: github.com/<org>/<repo> — phải match với VCS path bạn sẽ dùng
# Module name này được dùng trong toàn bộ import paths của project
go mod init github.com/your-org/go-api-boilerplate

# Sau lệnh này, go.mod được tạo:
# module github.com/your-org/go-api-boilerplate
# go 1.22

# Bước 3: Cài đặt dependencies
go get github.com/gin-gonic/gin@v1.10.0     # HTTP framework
go get github.com/joho/godotenv@v1.5.1      # Load .env file
go get go.uber.org/zap@v1.27.0             # Structured logging (production-grade)
go get github.com/google/uuid@v1.6.0        # UUID generation
go get golang.org/x/crypto@latest           # bcrypt password hashing

# Bước 4: Tạo cấu trúc thư mục
mkdir -p cmd/api
mkdir -p internal/{domain/{entity,repository,usecase},usecase/user,repository/postgres,delivery/http/{handler,middleware}}
mkdir -p pkg/{config,logger,response,apperrors}
mkdir -p migrations docs

# Verify go.sum được tạo (lock file — LUÔN commit lên git)
cat go.sum | head -5
```

### Go Modules — Hiểu Sâu Hơn

```
╔══════════════════════════════════════════════════════════════════════╗
║  go.mod vs go.sum — Tại Sao Cần Cả Hai?                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  go.mod = "Shopping List" (commit vào git)                         ║
║  → Khai báo module name, Go version                                 ║
║  → List tất cả direct + indirect dependencies                       ║
║  → Ví dụ: require github.com/gin-gonic/gin v1.10.0                ║
║                                                                      ║
║  go.sum = "Receipt" (commit vào git — KHÔNG xóa)                  ║
║  → Cryptographic hash của từng dependency version                   ║
║  → Đảm bảo reproducible builds: cùng go.sum → cùng binary         ║
║  → Nếu ai đó tamper dependency → go.sum sẽ FAIL                   ║
║                                                                      ║
║  Quy tắc:                                                           ║
║  ✅ LUÔN commit go.mod và go.sum                                    ║
║  ✅ Chạy `go mod tidy` sau khi thêm/xóa dependencies               ║
║  ✅ Review go.mod changes trong PR như review code                  ║
║  ❌ KHÔNG xóa go.sum thủ công                                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

```
# Ví dụ go.mod hoàn chỉnh:
module github.com/your-org/go-api-boilerplate

go 1.22

require (
    github.com/gin-gonic/gin v1.10.0
    github.com/google/uuid v1.6.0
    github.com/joho/godotenv v1.5.1
    go.uber.org/zap v1.27.0
    golang.org/x/crypto v0.22.0
)

require (
    // indirect = không dùng trực tiếp, nhưng dependency của dependency
    github.com/bytedance/sonic v1.11.6 // indirect
    github.com/gin-contrib/sse v0.1.0 // indirect
    // ... (auto-managed bởi go mod tidy)
)
```

### Cấu Trúc Thư Mục — Giải Thích Chi Tiết Từng Folder

```
go-api-boilerplate/
│
├── cmd/                         # ★ Entry points của application
│   └── api/                     # Mỗi binary = 1 subfolder trong cmd/
│       └── main.go              # CHỈ làm 3 việc: load config, wire deps, start server
│                                # KHÔNG có business logic nào ở đây
│
├── internal/                    # ★ Private code — Go compiler ENFORCE không import từ ngoài
│   │
│   ├── domain/                  # Tầng 1: DOMAIN — Trái tim, không phụ thuộc gì
│   │   ├── entity/              # Business objects (User, Order, Product...)
│   │   │   ├── user.go          # Struct + business methods + business rules
│   │   │   └── errors.go        # Domain-specific errors (ErrUserNotFound...)
│   │   ├── repository/          # Interfaces (PORTS) cho data access
│   │   │   └── user_repository.go
│   │   └── usecase/             # Interfaces (PORTS) cho business logic
│   │       └── user_usecase.go  # DTO (Request/Response types) cũng nằm ở đây
│   │
│   ├── usecase/                 # Tầng 2: USE CASE — Business Logic Implementation
│   │   └── user/
│   │       ├── user_usecase.go      # Implement domain/usecase.UserUseCase interface
│   │       ├── user_usecase_test.go # Unit tests — không cần DB!
│   │       └── helpers.go           # Private helpers (hashPassword, etc.)
│   │
│   ├── repository/              # Tầng 3: INFRASTRUCTURE — Data persistence
│   │   ├── postgres/
│   │   │   ├── user_repository.go       # Implement domain/repository.UserRepository
│   │   │   └── user_repository_test.go  # Integration tests (testcontainers)
│   │   └── redis/                        # Tương tự cho Redis cache
│   │       └── cache_repository.go
│   │
│   └── delivery/                # Tầng 4: ADAPTER — Kết nối thế giới ngoài
│       └── http/
│           ├── handler/
│           │   ├── user_handler.go       # HTTP handler, chỉ biết usecase interface
│           │   └── user_handler_test.go  # Test với httptest (không cần server thật)
│           ├── middleware/
│           │   ├── auth.go               # JWT validation
│           │   ├── logger.go             # Request logging
│           │   ├── cors.go               # CORS headers
│           │   └── recovery.go           # Panic recovery → 500
│           └── router.go                 # Route định nghĩa và middleware chain
│
├── pkg/                         # PUBLIC shared packages — có thể dùng ở project khác
│   ├── config/                  # App configuration loader
│   ├── logger/                  # Structured logger wrapper (zap)
│   ├── response/                # Chuẩn hóa HTTP response format
│   └── apperrors/               # Application-wide error types và codes
│
├── migrations/                  # Database migrations (golang-migrate format)
│   ├── 000001_create_users.up.sql
│   └── 000001_create_users.down.sql
│
├── docs/                        # Auto-generated bởi `swag init`
│   ├── docs.go                  # Go code — import để register swagger
│   ├── swagger.json
│   └── swagger.yaml
│
├── .air.toml                    # Hot-reload config (air tool)
├── .env.example                 # COMMIT — template với fake values
├── .env                         # KHÔNG COMMIT — actual secrets
├── .gitignore
├── .golangci.yml                # Linter configuration
├── docker-compose.yml           # Local dev: postgres, redis
├── Dockerfile                   # Production container
├── Makefile
└── README.md
```

> **Tại sao CMD, INTERNAL, PKG?**
>
> | Folder | Visibility | Mục đích |
> |--------|-----------|----------|
> | `cmd/` | Public | Binary entry points — mỗi `main.go` = 1 executable |
> | `internal/` | **Private** (Go enforced) | Core business code, không expose API |
> | `pkg/` | Public | Reusable utilities — safe to import từ project khác |
>
> Rule of thumb: **khi nghi ngờ → đặt vào `internal/`**. Promote ra `pkg/` chỉ khi thực sự cần share.

### main.go — Viết Đúng Chuẩn (Wiring Layer)

```go
// cmd/api/main.go
// main.go làm đúng 3 việc: Load Config → Wire Dependencies → Start Server
// Đây là nơi DUY NHẤT biết về concrete implementations
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    _ "github.com/joho/godotenv/autoload" // auto-load .env file khi init
    _ "github.com/lib/pq"                  // postgres driver (side-effect import)

    "github.com/your-org/go-api-boilerplate/internal/delivery/http/handler"
    "github.com/your-org/go-api-boilerplate/internal/delivery/http/router"
    postgresRepo "github.com/your-org/go-api-boilerplate/internal/repository/postgres"
    userUC "github.com/your-org/go-api-boilerplate/internal/usecase/user"
    "github.com/your-org/go-api-boilerplate/pkg/config"
    "github.com/your-org/go-api-boilerplate/pkg/logger"
)

func main() {
    // 1. LOAD CONFIG — từ environment variables (và .env file)
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("failed to load config: %v", err)
    }

    // 2. SETUP LOGGER
    log := logger.New(cfg.App.Env)
    defer log.Sync() // flush buffered log entries khi shutdown

    // 3. CONNECT INFRASTRUCTURE (DB, Cache...)
    db, err := sql.Open("postgres", cfg.Database.DSN())
    if err != nil {
        log.Fatalf("failed to connect to database: %v", err)
    }
    defer db.Close()

    // Connection pool settings
    db.SetMaxOpenConns(cfg.Database.MaxOpenConns)
    db.SetMaxIdleConns(cfg.Database.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.Database.ConnMaxLifetime)

    // Verify connection
    if err := db.Ping(); err != nil {
        log.Fatalf("failed to ping database: %v", err)
    }
    log.Info("database connected successfully")

    // 4. WIRE DEPENDENCIES (Manual DI — không cần framework)
    //    Thứ tự: Infrastructure → UseCase → Handler
    //    Repository (concrete) → implements domain interface
    userRepo := postgresRepo.NewUserRepository(db)

    //    UseCase nhận domain interface (không biết postgres)
    userUseCase := userUC.New(userRepo)

    //    Handler nhận usecase interface (không biết business logic)
    userHandler := handler.NewUserHandler(userUseCase)

    // 5. SETUP HTTP SERVER
    r := router.Setup(cfg, userHandler)

    srv := &http.Server{
        Addr:         fmt.Sprintf(":%d", cfg.App.Port),
        Handler:      r,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // 6. GRACEFUL SHUTDOWN — best practice cho production
    // Chạy server trong goroutine để không block signal handling
    go func() {
        log.Infof("server starting on port %d (env: %s)", cfg.App.Port, cfg.App.Env)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Block cho đến khi nhận SIGINT hoặc SIGTERM
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Info("shutting down server...")

    // Cho 30 giây để finish in-flight requests
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Errorf("server forced to shutdown: %v", err)
    }
    log.Info("server exited gracefully")
}
```

> **Graceful Shutdown là gì?** Khi Kubernetes rolling-update hay user Ctrl+C, app nhận SIGTERM.
> Thay vì kill ngay (mất requests đang xử lý), ta chờ tối đa 30s để tất cả requests hiện tại hoàn thành rồi mới tắt.
> Đây là **production requirement** — không làm được điều này = app không production-ready.

---

## §2. Go-Clean (Hexagonal) Architecture

### Contextual History — Tại Sao Cần Clean Architecture?

**2003 — "Payroll System" của Uncle Bob (Robert C. Martin):** Sau vài năm phát triển, hầu hết hệ thống đều trở thành "Big Ball of Mud" — database logic lẫn với business logic, lẫn với HTTP handler. Kết quả: test khó, không thay đổi được, không bảo trì được.

**Vấn đề cụ thể mà Clean Architecture giải quyết:**

```
╔══════════════════════════════════════════════════════════════════════╗
║  ANTI-PATTERN: "Spaghetti Architecture" (phổ biến ở junior code)   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  func CreateUser(c *gin.Context) {    // Handler làm TẤT CẢ        ║
║      var body CreateUserBody                                         ║
║      c.ShouldBindJSON(&body)                                         ║
║                                                                      ║
║      // Business logic TRONG handler — SAI!                         ║
║      if body.Password == body.ConfirmPassword { ... }               ║
║                                                                      ║
║      // Database call TRỰC TIẾP từ handler — SAI!                  ║
║      db.Exec("INSERT INTO users ...", body.Name, body.Email)        ║
║                                                                      ║
║      // Gửi email TRONG handler — SAI!                              ║
║      sendWelcomeEmail(body.Email)                                    ║
║  }                                                                   ║
║                                                                      ║
║  Hậu quả:                                                           ║
║  ❌ Không test được (cần Gin context, DB thật, email server)       ║
║  ❌ Không reuse được logic (CLI dùng cùng logic? Impossible)       ║
║  ❌ Thay DB? Phải sửa handler                                       ║
║  ❌ Thay framework? Phải rewrite toàn bộ                           ║
╚══════════════════════════════════════════════════════════════════════╝
```

**2005 — Alistair Cockburn** đề xuất "Hexagonal Architecture" (còn gọi là Ports & Adapters):

```
╔══════════════════════════════════════════════════════════════════════╗
║  "Application should have NO knowledge of the outside world."       ║
║                                                                      ║
║  Hexagonal Architecture — Nhìn Từ Trên Xuống:                      ║
║                                                                      ║
║         [HTTP Client]   [CLI]   [Test]   [gRPC Client]              ║
║              ↓            ↓        ↓           ↓                    ║
║         ┌─────────── PRIMARY ADAPTERS ────────────┐                 ║
║         │        (Delivery / Handlers)             │                 ║
║         └────────────────┬───────────────────────┘                  ║
║                          │ calls via PORT (interface)               ║
║         ┌────────────────▼───────────────────────┐                  ║
║         │           APPLICATION CORE              │                 ║
║         │     (Domain + Use Cases)                │                 ║
║         │    ← Không biết gì về outside →        │                 ║
║         └────────────────┬───────────────────────┘                  ║
║                          │ calls via PORT (interface)               ║
║         ┌────────────────▼───────────────────────┐                  ║
║         │       SECONDARY ADAPTERS               │                 ║
║         │  (Repository / Infrastructure)          │                 ║
║         └─────────────────────────────────────────┘                 ║
║              ↓            ↓        ↓           ↓                    ║
║         [Postgres]   [Redis]   [Email]   [S3 Storage]               ║
║                                                                      ║
║  PORT = Interface trong Go (UserRepository, UserUseCase)           ║
║  ADAPTER = Concrete implementation (postgresRepo, ginHandler)       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Dependency Rule — Quy Tắc Vàng

```
╔══════════════════════════════════════════════════════════════════════╗
║           DEPENDENCY DIRECTION (mũi tên = "import")                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Outer layers → Inner layers (LUÔN CÓ)                              ║
║  Inner layers ⟶ Outer layers (KHÔNG BAO GIỜ!)                      ║
║                                                                      ║
║  [cmd/main.go]                                                       ║
║       │ import                                                        ║
║       ▼                                                               ║
║  [delivery/handler]  →  [domain/usecase interface]                  ║
║  [repository/postgres] → [domain/repository interface]              ║
║       │                        │                                     ║
║       │ implement              │ implement                           ║
║       ▼                        ▼                                     ║
║  [usecase/user] → [domain/usecase interface]                        ║
║                 → [domain/repository interface] (inject)            ║
║                        │                                             ║
║                        ▼                                             ║
║                 [domain/entity]  ← CỐT LÕI, không import gì cả    ║
║                                                                      ║
║  Kiểm tra nhanh: nếu thay Postgres bằng MySQL,                      ║
║  bạn chỉ phải sửa: internal/repository/postgres/ ← CHỈ ĐÂY THÔI  ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4 Tầng Trong Thực Tế — Code Chi Tiết

#### Tầng 1: Domain (Entity + Errors + Interfaces)

```go
// internal/domain/entity/user.go
// ★ RULE: File này KHÔNG được import bất kỳ package nào của project
package entity

import (
    "regexp"
    "strings"
    "time"
    "unicode/utf8"
)

// User — Business Entity
// ★ KHÔNG có json tags vì entity không biết về HTTP
// ★ KHÔNG có db tags vì entity không biết về database
// ★ KHÔNG có validation tags của gin/validator vì entity độc lập với framework
type User struct {
    ID        string
    Name      string
    Email     string
    Password  string // always hashed, never plaintext
    IsActive  bool
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt *time.Time // nil = not deleted (soft delete)
}

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

// Validate — BUSINESS RULE, thuộc về domain
// Không bao giờ để validation này ở handler (HTTP concern)
func (u *User) Validate() error {
    name := strings.TrimSpace(u.Name)
    if name == "" {
        return ErrInvalidName
    }
    if utf8.RuneCountInString(name) < 2 {
        return ErrNameTooShort
    }
    if !emailRegex.MatchString(u.Email) {
        return ErrInvalidEmail
    }
    return nil
}

// IsDeleted — business method, logic thuộc về entity
func (u *User) IsDeleted() bool {
    return u.DeletedAt != nil
}

// SoftDelete — business action
func (u *User) SoftDelete() {
    now := time.Now()
    u.DeletedAt = &now
    u.IsActive = false
    u.UpdatedAt = now
}
```

```go
// internal/domain/entity/errors.go
// ★ Domain errors — KHÔNG phụ thuộc vào HTTP status codes hay DB error codes
// HTTP mapping sẽ được làm ở tầng delivery
package entity

import "errors"

var (
    // User errors
    ErrUserNotFound    = errors.New("user not found")
    ErrUserInactive    = errors.New("user account is inactive")
    ErrEmailDuplicate  = errors.New("email address already registered")

    // Validation errors
    ErrInvalidName  = errors.New("name is required")
    ErrNameTooShort = errors.New("name must be at least 2 characters")
    ErrInvalidEmail = errors.New("invalid email format")
)

// IsDomainError kiểm tra xem err có phải domain error không
// Dùng ở handler để map sang HTTP status code
func IsDomainError(err error) bool {
    domainErrors := []error{
        ErrUserNotFound, ErrUserInactive, ErrEmailDuplicate,
        ErrInvalidName, ErrNameTooShort, ErrInvalidEmail,
    }
    for _, e := range domainErrors {
        if errors.Is(err, e) {
            return true
        }
    }
    return false
}
```

```go
// internal/domain/repository/user_repository.go
// ★ PORT — Interface định nghĩa CONTRACT dữ liệu
// UseCase chỉ biết interface này, không biết Postgres hay bất cứ DB nào
package repository

import (
    "context"
    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
)

type ListUsersFilter struct {
    Name   string
    Email  string
    Limit  int
    Offset int
}

type ListUsersResult struct {
    Users []*entity.User
    Total int64
}

type UserRepository interface {
    FindByID(ctx context.Context, id string) (*entity.User, error)
    FindByEmail(ctx context.Context, email string) (*entity.User, error)
    List(ctx context.Context, filter ListUsersFilter) (*ListUsersResult, error)
    Save(ctx context.Context, user *entity.User) error
    Update(ctx context.Context, user *entity.User) error
    Delete(ctx context.Context, id string) error // soft delete
}
```

```go
// internal/domain/usecase/user_usecase.go
// ★ PORT — Interface cho business operations
// Handler chỉ biết interface này, không biết implementation cụ thể
package usecase

import (
    "context"
    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
    "github.com/your-org/go-api-boilerplate/internal/domain/repository"
)

// DTOs (Data Transfer Objects) — "hợp đồng" giữa handler và usecase
// Tách biệt với entity để tự do thay đổi API contract độc lập với domain
type CreateUserRequest struct {
    Name     string
    Email    string
    Password string // plaintext — usecase sẽ hash
}

type UpdateUserRequest struct {
    Name  *string // con trỏ: nil = không update, non-nil = update
    Email *string
}

type ListUsersRequest struct {
    Name  string
    Page  int
    Limit int
}

type ListUsersResponse struct {
    Users []*entity.User
    Total int64
    Page  int
    Limit int
}

type UserUseCase interface {
    GetUser(ctx context.Context, id string) (*entity.User, error)
    ListUsers(ctx context.Context, req ListUsersRequest) (*ListUsersResponse, error)
    CreateUser(ctx context.Context, req CreateUserRequest) (*entity.User, error)
    UpdateUser(ctx context.Context, id string, req UpdateUserRequest) (*entity.User, error)
    DeleteUser(ctx context.Context, id string) error
}

// Compile-time check — đảm bảo mọi implementation phải đầy đủ
// Đặt ở đây hoặc trong test file của usecase implementation
var _ repository.UserRepository = (*interface{})(nil) // placeholder
```

#### Tầng 2: Use Case (Business Logic Implementation)

```go
// internal/usecase/user/user_usecase.go
package user

import (
    "context"
    "fmt"
    "time"

    "github.com/google/uuid"
    "golang.org/x/crypto/bcrypt"
    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
    domainRepo "github.com/your-org/go-api-boilerplate/internal/domain/repository"
    domainUC "github.com/your-org/go-api-boilerplate/internal/domain/usecase"
)

// userUseCase implements domain/usecase.UserUseCase
// CHỈ biết về domain interfaces, KHÔNG biết Postgres hay HTTP
type userUseCase struct {
    userRepo domainRepo.UserRepository
}

// New — Constructor, trả về interface (không expose struct)
func New(userRepo domainRepo.UserRepository) domainUC.UserUseCase {
    return &userUseCase{userRepo: userRepo}
}

// Compile-time check: nếu userUseCase thiếu bất kỳ method nào, lỗi ngay tại build
var _ domainUC.UserUseCase = (*userUseCase)(nil)

func (uc *userUseCase) GetUser(ctx context.Context, id string) (*entity.User, error) {
    user, err := uc.userRepo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("GetUser: %w", err)
    }
    // Map "not found" sang domain error — không làm ở handler!
    if user == nil {
        return nil, entity.ErrUserNotFound
    }
    if user.IsDeleted() {
        return nil, entity.ErrUserNotFound // ẩn soft-deleted user
    }
    return user, nil
}

func (uc *userUseCase) CreateUser(ctx context.Context, req domainUC.CreateUserRequest) (*entity.User, error) {
    // Business rule 1: kiểm tra email đã tồn tại chưa
    existing, err := uc.userRepo.FindByEmail(ctx, req.Email)
    if err != nil {
        return nil, fmt.Errorf("CreateUser: check email: %w", err)
    }
    if existing != nil && !existing.IsDeleted() {
        return nil, entity.ErrEmailDuplicate
    }

    // Business rule 2: hash password trước khi lưu
    // bcrypt cost 12 = production recommended (balance security vs performance)
    hashedPw, err := bcrypt.GenerateFromPassword([]byte(req.Password), 12)
    if err != nil {
        return nil, fmt.Errorf("CreateUser: hash password: %w", err)
    }

    user := &entity.User{
        ID:        uuid.New().String(),
        Name:      req.Name,
        Email:     req.Email,
        Password:  string(hashedPw),
        IsActive:  true,
        CreatedAt: time.Now().UTC(),
        UpdatedAt: time.Now().UTC(),
    }

    // Business rule 3: validate theo domain rules
    if err := user.Validate(); err != nil {
        return nil, err // domain errors (ErrInvalidName, ErrInvalidEmail...)
    }

    if err := uc.userRepo.Save(ctx, user); err != nil {
        return nil, fmt.Errorf("CreateUser: save: %w", err)
    }

    // Quan trọng: ko trả về password hash cho caller
    user.Password = ""
    return user, nil
}

func (uc *userUseCase) UpdateUser(ctx context.Context, id string, req domainUC.UpdateUserRequest) (*entity.User, error) {
    // Phải lấy user hiện tại trước
    user, err := uc.GetUser(ctx, id)
    if err != nil {
        return nil, err // ErrUserNotFound propagates up
    }

    // Partial update — chỉ update field được gửi lên (non-nil pointer)
    if req.Name != nil {
        user.Name = *req.Name
    }
    if req.Email != nil {
        // Business rule: nếu email thay đổi, kiểm tra trùng lặp
        if *req.Email != user.Email {
            existing, _ := uc.userRepo.FindByEmail(ctx, *req.Email)
            if existing != nil && existing.ID != user.ID {
                return nil, entity.ErrEmailDuplicate
            }
        }
        user.Email = *req.Email
    }

    user.UpdatedAt = time.Now().UTC()

    if err := user.Validate(); err != nil {
        return nil, err
    }

    if err := uc.userRepo.Update(ctx, user); err != nil {
        return nil, fmt.Errorf("UpdateUser: %w", err)
    }

    user.Password = ""
    return user, nil
}

func (uc *userUseCase) DeleteUser(ctx context.Context, id string) error {
    user, err := uc.GetUser(ctx, id)
    if err != nil {
        return err
    }
    user.SoftDelete() // gọi domain method, không xóa thật
    return uc.userRepo.Update(ctx, user)
}
```

#### Tầng 3: Repository (Postgres Implementation)

```go
// internal/repository/postgres/user_repository.go
package postgres

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
    domainRepo "github.com/your-org/go-api-boilerplate/internal/domain/repository"
)

type userRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) domainRepo.UserRepository {
    return &userRepository{db: db}
}

// Compile-time check
var _ domainRepo.UserRepository = (*userRepository)(nil)

// userModel là DB representation — TÁCH BIỆT với domain entity
// Lý do: DB schema và domain entity có thể diverge theo thời gian
type userModel struct {
    ID        string         `db:"id"`
    Name      string         `db:"name"`
    Email     string         `db:"email"`
    Password  string         `db:"password"`
    IsActive  bool           `db:"is_active"`
    CreatedAt time.Time      `db:"created_at"`
    UpdatedAt time.Time      `db:"updated_at"`
    DeletedAt sql.NullTime   `db:"deleted_at"`
}

func (r *userRepository) FindByID(ctx context.Context, id string) (*entity.User, error) {
    var m userModel
    query := `
        SELECT id, name, email, password, is_active, created_at, updated_at, deleted_at
        FROM users
        WHERE id = $1
    `
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &m.ID, &m.Name, &m.Email, &m.Password,
        &m.IsActive, &m.CreatedAt, &m.UpdatedAt, &m.DeletedAt,
    )
    if err == sql.ErrNoRows {
        return nil, nil // chưa tìm thấy — không phải lỗi, trả nil
    }
    if err != nil {
        return nil, fmt.Errorf("FindByID: %w", err)
    }
    return m.toEntity(), nil
}

func (r *userRepository) Save(ctx context.Context, user *entity.User) error {
    query := `
        INSERT INTO users (id, name, email, password, is_active, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
    `
    _, err := r.db.ExecContext(ctx, query,
        user.ID, user.Name, user.Email, user.Password,
        user.IsActive, user.CreatedAt, user.UpdatedAt,
    )
    if err != nil {
        return fmt.Errorf("Save: %w", err)
    }
    return nil
}

// toEntity chuyển từ DB model sang domain entity
// Một số fields có thể transform khác nhau (e.g. NullTime → *time.Time)
func (m *userModel) toEntity() *entity.User {
    u := &entity.User{
        ID:        m.ID,
        Name:      m.Name,
        Email:     m.Email,
        IsActive:  m.IsActive,
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }
    if m.DeletedAt.Valid {
        t := m.DeletedAt.Time
        u.DeletedAt = &t
    }
    // Password KHÔNG được trả về ở đây (repository concern)
    // UseCase sẽ quyết định khi nào cần password
    return u
}
```

#### Tầng 4: HTTP Handler với Error Mapping

```go
// internal/delivery/http/handler/user_handler.go
package handler

import (
    "errors"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
    domainUC "github.com/your-org/go-api-boilerplate/internal/domain/usecase"
    "github.com/your-org/go-api-boilerplate/pkg/response"
)

type UserHandler struct {
    userUC domainUC.UserUseCase
}

func NewUserHandler(userUC domainUC.UserUseCase) *UserHandler {
    return &UserHandler{userUC: userUC}
}

// GetUser godoc
// @Summary      Get user by ID
// @Tags         users
// @Produce      json
// @Param        id   path      string  true   "User UUID"
// @Success      200  {object}  response.Success{data=UserResponse}
// @Failure      404  {object}  response.Error
// @Failure      500  {object}  response.Error
// @Security     BearerAuth
// @Router       /users/{id} [get]
func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")

    user, err := h.userUC.GetUser(c.Request.Context(), id)
    if err != nil {
        // ★ Error Mapping: domain error → HTTP status code
        // Đây là trách nhiệm của Adapter layer, không phải UseCase
        h.handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, response.Ok(toUserResponse(user)))
}

// Create godoc
// @Summary      Create new user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        request  body      CreateUserBody  true  "User data"
// @Success      201      {object}  response.Success{data=UserResponse}
// @Failure      400      {object}  response.Error
// @Failure      409      {object}  response.Error
// @Failure      500      {object}  response.Error
// @Router       /users [post]
func (h *UserHandler) Create(c *gin.Context) {
    var body CreateUserBody
    if err := c.ShouldBindJSON(&body); err != nil {
        // ShouldBindJSON lỗi = validation fail (HTTP 400)
        c.JSON(http.StatusBadRequest, response.Fail("validation failed", err))
        return
    }

    user, err := h.userUC.CreateUser(c.Request.Context(), domainUC.CreateUserRequest{
        Name:     body.Name,
        Email:    body.Email,
        Password: body.Password,
    })
    if err != nil {
        h.handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, response.Ok(toUserResponse(user)))
}

// handleError — centralized error → HTTP status mapping
// ★ Đây là nơi DUY NHẤT quyết định HTTP status code cho từng loại error
func (h *UserHandler) handleError(c *gin.Context, err error) {
    switch {
    case errors.Is(err, entity.ErrUserNotFound):
        c.JSON(http.StatusNotFound, response.Fail("user not found", nil))
    case errors.Is(err, entity.ErrEmailDuplicate):
        c.JSON(http.StatusConflict, response.Fail("email already registered", nil))
    case errors.Is(err, entity.ErrInvalidName),
        errors.Is(err, entity.ErrNameTooShort),
        errors.Is(err, entity.ErrInvalidEmail):
        c.JSON(http.StatusBadRequest, response.Fail(err.Error(), nil))
    default:
        // Unknown error → 500 (không leak internal error ra ngoài)
        c.JSON(http.StatusInternalServerError, response.Fail("internal server error", nil))
    }
}

// Request/Response types — HTTP layer DTOs (khác với domain DTOs!)
type CreateUserBody struct {
    Name     string `json:"name"     binding:"required,min=2"`
    Email    string `json:"email"    binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}

// UserResponse loại bỏ password và thêm json tags
type UserResponse struct {
    ID        string     `json:"id"`
    Name      string     `json:"name"`
    Email     string     `json:"email"`
    IsActive  bool       `json:"is_active"`
    CreatedAt string     `json:"created_at"`
}

func toUserResponse(u *entity.User) UserResponse {
    return UserResponse{
        ID:       u.ID,
        Name:     u.Name,
        Email:    u.Email,
        IsActive: u.IsActive,
        CreatedAt: u.CreatedAt.Format("2006-01-02T15:04:05Z07:00"),
    }
}
```

> **Key insight:** `handleError()` được centralize trong handler — không phải mỗi endpoint tự quyết định HTTP status. Khi domain error thay đổi, chỉ cần sửa 1 nơi.

---

## §3. Chuẩn Hóa API Response — Envelope Pattern

```go
// pkg/response/response.go
// Envelope pattern: mọi response đều có cấu trúc nhất quán
// Frontend/Mobile có thể predictably parse bất kỳ response nào
package response

type Success struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data"`
}

type PaginatedSuccess struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data"`
    Meta    Pagination  `json:"meta"`
}

type Pagination struct {
    Total  int64 `json:"total"`
    Page   int   `json:"page"`
    Limit  int   `json:"limit"`
    Pages  int64 `json:"pages"`
}

type Error struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
    // Detail chỉ expose ở non-production (tránh leak internal)
    Detail  string `json:"detail,omitempty"`
}

func Ok(data interface{}) Success {
    return Success{Success: true, Data: data}
}

func OkPaginated(data interface{}, total int64, page, limit int) PaginatedSuccess {
    pages := total / int64(limit)
    if total%int64(limit) > 0 {
        pages++
    }
    return PaginatedSuccess{
        Success: true,
        Data:    data,
        Meta:    Pagination{Total: total, Page: page, Limit: limit, Pages: pages},
    }
}

func Fail(message string, err error) Error {
    detail := ""
    if err != nil {
        detail = err.Error()
    }
    return Error{Success: false, Message: message, Detail: detail}
}
```

**Response format mẫu:**

```json
// GET /api/v1/users?page=1&limit=2  → 200 OK
{
  "success": true,
  "data": [
    {"id": "abc-123", "name": "John", "email": "john@example.com", "is_active": true},
    {"id": "def-456", "name": "Jane", "email": "jane@example.com", "is_active": true}
  ],
  "meta": {"total": 100, "page": 1, "limit": 2, "pages": 50}
}

// POST /api/v1/users với email đã tồn tại  → 409 Conflict
{
  "success": false,
  "message": "email already registered"
}
```

---

## §4. Unit Test Theo Chuẩn Clean Architecture

### Tại Sao Clean Architecture Dễ Test Hơn?

```
╔══════════════════════════════════════════════════════════════════════╗
║  "Code khó test = code có design xấu."                              ║
║                       — Michael Feathers, Working Effectively With  ║
║                         Legacy Code                                  ║
║                                                                      ║
║  Clean Architecture → Interfaces → Dễ Mock → Test nhanh            ║
║  Spaghetti Code     → Hard dependencies → Khó mock → Chạy DB thật ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Mock Repository — Trái Tim Của Unit Test

```go
// internal/usecase/user/user_usecase_test.go
package user_test

import (
    "context"
    "errors"
    "strings"
    "testing"

    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
    domainRepo "github.com/your-org/go-api-boilerplate/internal/domain/repository"
    domainUC "github.com/your-org/go-api-boilerplate/internal/domain/usecase"
    userUC "github.com/your-org/go-api-boilerplate/internal/usecase/user"
)

// mockUserRepo: chúng ta viết tay, KHÔNG dùng code-gen mock (hiểu nguyên lý trước)
// Trong production có thể dùng: github.com/golang/mock hoặc github.com/vektra/mockery
type mockUserRepo struct {
    findByIDFn    func(ctx context.Context, id string) (*entity.User, error)
    findByEmailFn func(ctx context.Context, email string) (*entity.User, error)
    saveFn        func(ctx context.Context, user *entity.User) error
    updateFn      func(ctx context.Context, user *entity.User) error
    deleteFn      func(ctx context.Context, id string) error
    listFn        func(ctx context.Context, f domainRepo.ListUsersFilter) (*domainRepo.ListUsersResult, error)
}

func (m *mockUserRepo) FindByID(ctx context.Context, id string) (*entity.User, error) {
    if m.findByIDFn != nil {
        return m.findByIDFn(ctx, id)
    }
    return nil, nil
}
func (m *mockUserRepo) FindByEmail(ctx context.Context, email string) (*entity.User, error) {
    if m.findByEmailFn != nil {
        return m.findByEmailFn(ctx, email)
    }
    return nil, nil
}
func (m *mockUserRepo) Save(ctx context.Context, user *entity.User) error {
    if m.saveFn != nil {
        return m.saveFn(ctx, user)
    }
    return nil
}
func (m *mockUserRepo) Update(ctx context.Context, user *entity.User) error {
    if m.updateFn != nil {
        return m.updateFn(ctx, user)
    }
    return nil
}
func (m *mockUserRepo) Delete(ctx context.Context, id string) error {
    if m.deleteFn != nil {
        return m.deleteFn(ctx, id)
    }
    return nil
}
func (m *mockUserRepo) List(ctx context.Context, f domainRepo.ListUsersFilter) (*domainRepo.ListUsersResult, error) {
    if m.listFn != nil {
        return m.listFn(ctx, f)
    }
    return &domainRepo.ListUsersResult{}, nil
}

// Compile-time interface compliance check — THÓI QUEN CỦA SENIOR ENGINEER
var _ domainRepo.UserRepository = (*mockUserRepo)(nil)
```

### Table-Driven Tests — Chuẩn Go

```go
func TestCreateUser(t *testing.T) {
    tests := []struct {
        name        string
        req         domainUC.CreateUserRequest
        setupMock   func() *mockUserRepo
        expectErr   bool
        errIs       error  // kiểm tra exact error type với errors.Is
        checkResult func(t *testing.T, u *entity.User) // assertions phức tạp
    }{
        {
            name: "success/create new user",
            req:  domainUC.CreateUserRequest{Name: "John Doe", Email: "john@example.com", Password: "SecurePass123"},
            setupMock: func() *mockUserRepo {
                return &mockUserRepo{} // defaults: FindByEmail trả nil, Save trả nil
            },
            expectErr: false,
            checkResult: func(t *testing.T, u *entity.User) {
                if u.ID == "" {
                    t.Error("expected non-empty ID")
                }
                if u.Password != "" {
                    t.Error("password should be cleared before returning")
                }
                if u.Name != "John Doe" {
                    t.Errorf("expected name %q, got %q", "John Doe", u.Name)
                }
            },
        },
        {
            name: "fail/email already registered",
            req:  domainUC.CreateUserRequest{Name: "John", Email: "existing@example.com", Password: "SecurePass123"},
            setupMock: func() *mockUserRepo {
                return &mockUserRepo{
                    findByEmailFn: func(_ context.Context, _ string) (*entity.User, error) {
                        return &entity.User{Email: "existing@example.com", IsActive: true}, nil
                    },
                }
            },
            expectErr: true,
            errIs:     entity.ErrEmailDuplicate,
        },
        {
            name: "fail/invalid email format",
            req:  domainUC.CreateUserRequest{Name: "John", Email: "not-an-email", Password: "SecurePass123"},
            setupMock: func() *mockUserRepo {
                return &mockUserRepo{}
            },
            expectErr: true,
            errIs:     entity.ErrInvalidEmail,
        },
        {
            name: "fail/name too short",
            req:  domainUC.CreateUserRequest{Name: "J", Email: "john@example.com", Password: "SecurePass123"},
            setupMock: func() *mockUserRepo {
                return &mockUserRepo{}
            },
            expectErr: true,
            errIs:     entity.ErrNameTooShort,
        },
        {
            name: "fail/database error on save",
            req:  domainUC.CreateUserRequest{Name: "John Doe", Email: "john@example.com", Password: "SecurePass123"},
            setupMock: func() *mockUserRepo {
                return &mockUserRepo{
                    saveFn: func(_ context.Context, _ *entity.User) error {
                        return errors.New("pq: connection refused")
                    },
                }
            },
            expectErr: true,
            errIs:     nil, // lỗi này wrapped, không kiểm tra exact type
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := tt.setupMock()
            uc := userUC.New(repo)

            result, err := uc.CreateUser(context.Background(), tt.req)

            if tt.expectErr {
                if err == nil {
                    t.Fatal("expected error but got nil")
                }
                // errors.Is kiểm tra cả wrapped errors (dùng %w khi wrap)
                if tt.errIs != nil && !errors.Is(err, tt.errIs) {
                    t.Errorf("expected error %v, got %v", tt.errIs, err)
                }
                return
            }

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if result == nil {
                t.Fatal("expected non-nil result")
            }
            if tt.checkResult != nil {
                tt.checkResult(t, result)
            }
        })
    }
}
```

### Test Helper Functions — Giữ Test Sạch

```go
// internal/usecase/user/user_usecase_test.go (tiếp)

// requireNoError thay thế mẫu boúlet lặp đi lặp lại
func requireNoError(t *testing.T, err error, msg string) {
    t.Helper() // đánh dấu được gọi từ helper → stack trace chỉ đâu test fail
    if err != nil {
        t.Fatalf("%s: unexpected error: %v", msg, err)
    }
}

func requireError(t *testing.T, err error, msg string) {
    t.Helper()
    if err == nil {
        t.Fatalf("%s: expected error but got nil", msg)
    }
}

// newTestUser tạo entity.User cần thiết cho tests
func newTestUser(opts ...func(*entity.User)) *entity.User {
    u := &entity.User{
        ID:       "test-user-id-123",
        Name:     "Test User",
        Email:    "test@example.com",
        IsActive: true,
    }
    for _, opt := range opts {
        opt(u)
    }
    return u
}

// Ví dụ sử dụng:
func TestGetUser_Deleted(t *testing.T) {
    repo := &mockUserRepo{
        findByIDFn: func(_ context.Context, _ string) (*entity.User, error) {
            u := newTestUser(func(u *entity.User) {
                u.SoftDelete() // simulate deleted user
            })
            return u, nil
        },
    }
    uc := userUC.New(repo)
    _, err := uc.GetUser(context.Background(), "test-user-id-123")

    requireError(t, err, "GetUser with deleted user")
    if !errors.Is(err, entity.ErrUserNotFound) {
        t.Errorf("expected ErrUserNotFound, got %v", err)
    }
}

// containsString — tiện ích cho string assertions
func containsString(s, substr string) bool {
    return strings.Contains(s, substr)
}
```

> **errors.Is vs errors.As:**
>
> | | Mục đích | Ví dụ |
> |--|--|--|
> | `errors.Is(err, target)` | Kiểm tra err == target (kể cả wrapped) | `errors.Is(err, entity.ErrNotFound)` |
> | `errors.As(err, &target)` | Unwrap sang type cụ thể | `errors.As(err, &validationErr)` |
>
> Dùng `fmt.Errorf("context: %w", err)` khi wrap để `errors.Is/As` vẫn hoạt động đúng quê.

---

## §5. App Config Với Environment Variables

### 12-Factor App — Nguyên Tắc Config

```
╔══════════════════════════════════════════════════════════════════════╗
║  12-Factor App (Heroku, 2012) — Factor III: Config                  ║
║                                                                      ║
║  "Store config in the environment"                                   ║
║                                                                      ║
║  Config = bất cứ thứ gì THAY ĐỔI giữa các môi trường:             ║
║  → Database credentials                                              ║
║  → API keys (secrets)                                                ║
║  → Port numbers                                                      ║
║  → Feature flags                                                     ║
║                                                                      ║
║  ❌ KHÔNG hardcode config trong code                                 ║
║  ❌ KHÔNG commit .env lên git (commit .env.example thôi)            ║
║  ✅ Đọc từ environment variables                                     ║
║  ✅ Có default values hợp lý cho local dev                          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Cấu Trúc Config — Đầy Đủ Production

```go
// pkg/config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

type Config struct {
    App      AppConfig
    Database DatabaseConfig
    JWT      JWTConfig
    Redis    RedisConfig
}

type AppConfig struct {
    Name        string
    Env         string // "development", "staging", "production"
    Port        int
    Debug       bool
    Version     string
    AllowOrigins []string // CORS
}

// IsProduction giúp các module khác kiểm tra env mà không biết string "production"
func (a AppConfig) IsProduction() bool { return a.Env == "production" }
func (a AppConfig) IsDevelopment() bool { return a.Env == "development" }

type DatabaseConfig struct {
    Host            string
    Port            int
    Name            string
    User            string
    Password        string
    SSLMode         string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

// DSN trả về Postgres connection string — được dùng trong main.go
func (d DatabaseConfig) DSN() string {
    return fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        d.Host, d.Port, d.User, d.Password, d.Name, d.SSLMode,
    )
}

type JWTConfig struct {
    Secret      string
    ExpireHours int
}

type RedisConfig struct {
    Addr     string
    Password string
    DB       int
}

func Load() (*Config, error) {
    cfg := &Config{
        App: AppConfig{
            Name:         getEnv("APP_NAME", "go-api-boilerplate"),
            Env:          getEnv("APP_ENV", "development"),
            Port:         getEnvInt("APP_PORT", 8080),
            Debug:        getEnvBool("APP_DEBUG", true),
            Version:      getEnv("APP_VERSION", "v0.0.1"),
            AllowOrigins: []string{getEnv("CORS_ALLOW_ORIGIN", "http://localhost:3000")},
        },
        Database: DatabaseConfig{
            Host:            getEnv("DB_HOST", "localhost"),
            Port:            getEnvInt("DB_PORT", 5432),
            Name:            getEnv("DB_NAME", "go_api_db"),
            User:            getEnv("DB_USER", "postgres"),
            Password:        getEnv("DB_PASSWORD", ""),
            SSLMode:         getEnv("DB_SSLMODE", "disable"),
            MaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
            MaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 5),
            ConnMaxLifetime: time.Duration(getEnvInt("DB_CONN_MAX_LIFETIME_MIN", 5)) * time.Minute,
        },
        JWT: JWTConfig{
            Secret:      getEnv("JWT_SECRET", ""),
            ExpireHours: getEnvInt("JWT_EXPIRE_HOURS", 24),
        },
        Redis: RedisConfig{
            Addr:     getEnv("REDIS_ADDR", "localhost:6379"),
            Password: getEnv("REDIS_PASSWORD", ""),
            DB:       getEnvInt("REDIS_DB", 0),
        },
    }

    return cfg, cfg.validate()
}

// validate tách riêng để dễ test
func (c *Config) validate() error {
    if c.App.IsProduction() {
        if c.JWT.Secret == "" {
            return fmt.Errorf("JWT_SECRET is required in production")
        }
        if len(c.JWT.Secret) < 32 {
            return fmt.Errorf("JWT_SECRET must be at least 32 characters")
        }
        if c.Database.Password == "" {
            return fmt.Errorf("DB_PASSWORD is required in production")
        }
    }
    return nil
}

func getEnv(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}

func getEnvInt(key string, defaultVal int) int {
    if val := os.Getenv(key); val != "" {
        if i, err := strconv.Atoi(val); err == nil {
            return i
        }
    }
    return defaultVal
}

func getEnvBool(key string, defaultVal bool) bool {
    if val := os.Getenv(key); val != "" {
        if b, err := strconv.ParseBool(val); err == nil {
            return b
        }
    }
    return defaultVal
}
```

### Logger Setup — Structured Logging với Zap

```go
// pkg/logger/logger.go
// Zap = production-grade logger của Uber
// Vs fmt.Println: Zap structured (JSON output), zero-allocation, levels
package logger

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func New(env string) *zap.SugaredLogger {
    var cfg zap.Config

    if env == "production" {
        // Production: JSON format, không có caller/stacktrace cho WARN
        cfg = zap.NewProductionConfig()
        cfg.EncoderConfig.TimeKey = "timestamp"
        cfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
    } else {
        // Development: human-readable, màu sắc, stacktrace
        cfg = zap.NewDevelopmentConfig()
        cfg.EncoderConfig.EncodeLevel = zapcore.CapitalColorLevelEncoder
    }

    logger, _ := cfg.Build()
    return logger.Sugar() // SugaredLogger: log.Infof("hello %s", name)
}

// Output so sánh:
// Development: 2024-01-15T10:30:00+07:00  INFO  server starting  {"port": 8080}
// Production:  {"level":"info","timestamp":"2024-01-15T03:30:00Z","msg":"server starting","port":8080}
```

```bash
# .env.example — commit file này lên git
APP_NAME=go-api-boilerplate
APP_ENV=development
APP_PORT=8080
APP_DEBUG=true
APP_VERSION=v1.0.0
CORS_ALLOW_ORIGIN=http://localhost:3000

DB_HOST=localhost
DB_PORT=5432
DB_NAME=go_api_db
DB_USER=postgres
DB_PASSWORD=your_password_here
DB_SSLMODE=disable
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME_MIN=5

JWT_SECRET=your-very-long-secret-key-minimum-32-chars
JWT_EXPIRE_HOURS=24

REDIS_ADDR=localhost:6379
REDIS_PASSWORD=
REDIS_DB=0
```

> **Anti-pattern thường gặp:** Đừng bao giờ log secret vào stdout/stderr dù là debug mode. Dùng `log.Infof("config loaded for env: %s", cfg.App.Env)` — KHÔNG `log.Infof("%+v", cfg)` (sẽ in password ra).

---

## §6. API Documentation — OpenAPI với Swaggo

### Tại Sao Documentation Quan Trọng?

```
╔══════════════════════════════════════════════════════════════════════╗
║  "Code nói cho máy tính biết phải làm gì.                          ║
║   Documentation nói cho con người biết tại sao."                   ║
║                                                                      ║
║  OpenAPI Spec (formerly Swagger) = CONTRACT giữa:                  ║
║  → Backend team và Frontend/Mobile team                             ║
║  → Bạn hiện tại và bạn 6 tháng sau                                 ║
║  → API owner và API consumer                                         ║
║                                                                      ║
║  Production benefit:                                                 ║
║  → Auto-generate client SDKs (TypeScript, Python, Java...)          ║
║  → Interactive docs (Swagger UI) = manual testing không cần Postman║
║  → Contract testing (Pact)                                           ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Cài Đặt và Sử Dụng Swaggo

```bash
# Cài swag CLI (chỉ cần 1 lần)
go install github.com/swaggo/swag/cmd/swag@latest

# Cài dependencies
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/swag
go get github.com/swaggo/files

# Generate docs (chạy mỗi khi thay đổi annotations)
swag init -g cmd/api/main.go -o docs/
```

### Annotations Trong Code

```go
// cmd/api/main.go
// @title           Go API Boilerplate
// @version         1.0
// @description     Production-grade Go API with Clean Architecture
// @termsOfService  http://swagger.io/terms/

// @contact.name   API Support
// @contact.email  support@yourorg.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8080
// @BasePath  /api/v1

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.
func main() {
    // ...
}
```

```go
// internal/delivery/http/handler/user_handler.go

// GetUser godoc
// @Summary      Get user by ID
// @Description  Retrieve a single user's information by their UUID
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      string  true  "User ID (UUID format)"
// @Success      200  {object}  response.Success{data=UserResponse}
// @Failure      404  {object}  response.Error  "User not found"
// @Failure      500  {object}  response.Error  "Internal server error"
// @Security     BearerAuth
// @Router       /users/{id} [get]
func (h *UserHandler) GetUser(c *gin.Context) {
    // implementation...
}
```

```go
// internal/delivery/http/router.go — Full router với middleware chain
package http

import (
    "net/http"

    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"

    _ "github.com/your-org/go-api-boilerplate/docs" // side-effect: register swagger
    "github.com/your-org/go-api-boilerplate/internal/delivery/http/handler"
    "github.com/your-org/go-api-boilerplate/internal/delivery/http/middleware"
    "github.com/your-org/go-api-boilerplate/pkg/config"
)

func Setup(cfg *config.Config, userHandler *handler.UserHandler) *gin.Engine {
    // gin.New() thay vì gin.Default() để tự kiểm soát middleware
    r := gin.New()

    // ── Global Middleware ─────────────────────────────────────────────
    r.Use(middleware.Recovery())             // Panic → 500, không crash server
    r.Use(middleware.RequestLogger())        // Log request/response time
    r.Use(middleware.CORS(cfg.App.AllowOrigins)) // CORS headers

    // ── Health Check (không cần auth) ─────────────────────────────────
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "status":  "ok",
            "version": cfg.App.Version,
            "env":     cfg.App.Env,
        })
    })

    // ── Swagger UI (chỉ bật ngoài production) ─────────────────────────
    if !cfg.App.IsProduction() {
        r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    }

    // ── API v1 Routes ──────────────────────────────────────────────────
    v1 := r.Group("/api/v1")
    {
        // Public routes — không cần authentication
        v1.POST("/users", userHandler.Create)

        // Protected routes — cần JWT token
        protected := v1.Group("")
        protected.Use(middleware.Auth(cfg.JWT.Secret))
        {
            protected.GET("/users", userHandler.List)
            protected.GET("/users/:id", userHandler.GetUser)
            protected.PATCH("/users/:id", userHandler.Update)
            protected.DELETE("/users/:id", userHandler.Delete)
        }
    }

    return r
}
```

### .golangci.yml — Linter Configuration

```yaml
# .golangci.yml — cấu hình golangci-lint
# Commit file này — mọi developer dùng cùng 1 bộ rules
run:
  timeout: 5m
  modules-download-mode: readonly

linters:
  enable:
    - gofmt          # Format check
    - goimports      # Import ordering
    - govet          # Official go vet
    - errcheck       # Không bỏ qua errors
    - staticcheck    # Nhiều kiểm tra tĩnh
    - gosimple       # Simplify code gợi ý
    - ineffassign    # Phát hiện gán giá trị không dùng
    - unused         # Phát hiện code không dùng
    - misspell       # Lỗi chính tả trong comments
    - godot          # Comments phải kết thúc bằng dấu chấm
    - noctx          # HTTP request phải có context
    - contextcheck   # Context phải được propagate đúng

linters-settings:
  errcheck:
    check-type-assertions: true  # Kiểm tra type assertion không có ok
  govet:
    enable-all: true

issues:
  exclude-rules:
    - path: _test\.go              # Nới lỏng rules cho test files
      linters:
        - errcheck
        - godot
```

---

## §7. Makefile — Tự Động Hóa Mọi Thứ

### Tại Sao Cần Makefile?

```
╔══════════════════════════════════════════════════════════════════════╗
║  "Nếu bạn phải chạy > 1 command để làm một việc,                  ║
║   hãy đưa nó vào Makefile."                                         ║
║                                                                      ║
║  Makefile = RUNBOOK của developer:                                   ║
║  → make dev    : Chạy app local                                      ║
║  → make test   : Chạy tất cả tests                                  ║
║  → make lint   : Check code style                                    ║
║  → make docs   : Generate Swagger docs                               ║
║  → make migrate: Chạy database migrations                            ║
╚══════════════════════════════════════════════════════════════════════╝
```

```makefile
# Makefile
.PHONY: help dev build test test-coverage lint docs migrate-up migrate-down docker-up

# Default target
help: ## Show this help message
	@echo "Available commands:"
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ─── Development ─────────────────────────────────────────────────────
dev: ## Run development server with hot-reload (requires air)
	air -c .air.toml

run: ## Run without hot-reload
	go run ./cmd/api/main.go

# ─── Build ───────────────────────────────────────────────────────────
build: ## Build production binary
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
		go build -ldflags="-w -s -X main.Version=$(VERSION)" \
		-o bin/api ./cmd/api/main.go

# ─── Testing ─────────────────────────────────────────────────────────
test: ## Run all tests
	go test ./... -v -race

test-unit: ## Run unit tests only (fast, no integration)
	go test ./internal/usecase/... -v -race

test-coverage: ## Run tests with coverage report
	go test ./... -coverprofile=coverage.out -covermode=atomic
	go tool cover -html=coverage.out -o coverage.html
	@echo "Coverage report: coverage.html"

test-coverage-check: ## Fail if coverage < 80%
	go test ./... -coverprofile=coverage.out -covermode=atomic
	go tool cover -func=coverage.out | grep total | \
		awk '{print $$3}' | tr -d '%' | \
		awk '{if ($$1 < 80) {print "Coverage " $$1 "% is below 80%"; exit 1} \
		else {print "Coverage " $$1 "% OK"}}'

# ─── Code Quality ────────────────────────────────────────────────────
lint: ## Run linter (requires golangci-lint)
	golangci-lint run ./...

fmt: ## Format all Go files
	gofmt -w .
	goimports -w .

vet: ## Run go vet
	go vet ./...

check: fmt vet lint ## Run all code quality checks

# ─── Documentation ───────────────────────────────────────────────────
docs: ## Generate Swagger documentation
	swag init -g cmd/api/main.go -o docs/ --parseDependency
	@echo "Docs generated: http://localhost:8080/swagger/index.html"

# ─── Database ────────────────────────────────────────────────────────
migrate-up: ## Run all pending migrations
	migrate -path migrations/ -database "$(DATABASE_URL)" up

migrate-down: ## Rollback last migration
	migrate -path migrations/ -database "$(DATABASE_URL)" down 1

migrate-create: ## Create new migration (usage: make migrate-create NAME=add_users)
	migrate create -ext sql -dir migrations/ -seq $(NAME)

# ─── Docker ──────────────────────────────────────────────────────────
docker-up: ## Start all services (DB, Redis) via Docker Compose
	docker compose up -d postgres redis

docker-down: ## Stop all Docker services
	docker compose down

docker-build: ## Build Docker image
	docker build -t go-api-boilerplate:latest .

# ─── Utilities ───────────────────────────────────────────────────────
tidy: ## Tidy go.mod and go.sum
	go mod tidy

install-tools: ## Install all required dev tools
	go install github.com/swaggo/swag/cmd/swag@latest
	go install github.com/air-verse/air@latest
	curl -sSfL https://raw.githubusercontent.com/golangci-lint/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin
```

---

## §8. Go Test Commands & Code Coverage

### Hiểu Sâu `go test` Flags

```bash
# ─── Cơ bản ──────────────────────────────────────────────────────────

# Chạy tất cả tests trong project
go test ./...

# Chạy với output chi tiết (pass/fail từng test)
go test ./... -v

# Chạy test trong 1 package cụ thể
go test ./internal/usecase/user/...

# Chạy 1 test function cụ thể (regex)
go test ./... -run TestCreateUser

# Chạy tests match pattern
go test ./... -run "TestCreateUser/success"

# ─── Performance & Race Conditions ───────────────────────────────────

# Phát hiện race conditions (LUÔN dùng trong CI)
go test ./... -race

# Timeout cho toàn bộ test suite
go test ./... -timeout 60s

# Bỏ cache (force re-run)
go test ./... -count=1

# ─── Coverage ────────────────────────────────────────────────────────

# Generate coverage profile
go test ./... -coverprofile=coverage.out -covermode=atomic

# Xem coverage tổng hợp per file
go tool cover -func=coverage.out

# Xem coverage dạng HTML (visual, highlight từng line)
go tool cover -html=coverage.out -o coverage.html

# Xem coverage % ngay trong terminal
go test ./... -cover

# ─── Benchmarks ──────────────────────────────────────────────────────

# Chạy benchmarks (không chạy cùng go test ./... mặc định)
go test ./... -bench=. -benchmem

# Benchmark với số lần lặp cụ thể
go test ./... -bench=BenchmarkCreateUser -benchtime=5s
```

### Coverage Theo Chuẩn Production

```
╔══════════════════════════════════════════════════════════════════════╗
║  TARGET COVERAGE BY LAYER:                                          ║
║                                                                      ║
║  Domain (Entity, Business Rules)   → 100%  (zero dependency)       ║
║  Use Case (Business Logic)         → 90%+  (có thể mock hoàn toàn) ║
║  Repository (Data Access)          → 70%+  (integration test tách) ║
║  Handler (HTTP Adapter)            → 80%+  (httptest package)      ║
║                                                                      ║
║  Overall project target: ≥ 80%                                      ║
║                                                                      ║
║  ⚠️ Coverage 100% không có nghĩa không có bug.                      ║
║     Coverage là SAFETY NET, không phải mục tiêu cuối cùng.         ║
╚══════════════════════════════════════════════════════════════════════╝
```

```go
// internal/delivery/http/handler/user_handler_test.go — FULL EXAMPLE
package handler_test

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/your-org/go-api-boilerplate/internal/domain/entity"
    domainUC "github.com/your-org/go-api-boilerplate/internal/domain/usecase"
    "github.com/your-org/go-api-boilerplate/internal/delivery/http/handler"
)

// mockUserUseCase — fake UseCase cho handler tests
type mockUserUseCase struct {
    getUserFn    func(ctx context.Context, id string) (*entity.User, error)
    createUserFn func(ctx context.Context, req domainUC.CreateUserRequest) (*entity.User, error)
}

func (m *mockUserUseCase) GetUser(ctx context.Context, id string) (*entity.User, error) {
    if m.getUserFn != nil {
        return m.getUserFn(ctx, id)
    }
    return nil, nil
}
func (m *mockUserUseCase) CreateUser(ctx context.Context, req domainUC.CreateUserRequest) (*entity.User, error) {
    if m.createUserFn != nil {
        return m.createUserFn(ctx, req)
    }
    return nil, nil
}
// ... implement remaining methods

var _ domainUC.UserUseCase = (*mockUserUseCase)(nil)

// setupRouter tạo Gin router với handler — reuse trong nhiều tests
func setupRouter(h *handler.UserHandler) *gin.Engine {
    gin.SetMode(gin.TestMode) // tắt verbose logs khi test
    r := gin.New()
    r.POST("/users", h.Create)
    r.GET("/users/:id", h.GetUser)
    return r
}

func TestUserHandler_Create(t *testing.T) {
    tests := []struct {
        name           string
        body           string
        setupMock      func() *mockUserUseCase
        wantStatusCode int
        wantSuccess    bool
        checkBody      func(t *testing.T, body []byte)
    }{
        {
            name: "success/201 created",
            body: `{"name":"John Doe","email":"john@example.com","password":"SecurePass123"}`,
            setupMock: func() *mockUserUseCase {
                return &mockUserUseCase{
                    createUserFn: func(_ context.Context, req domainUC.CreateUserRequest) (*entity.User, error) {
                        return &entity.User{ID: "uuid-123", Name: req.Name, Email: req.Email, IsActive: true}, nil
                    },
                }
            },
            wantStatusCode: http.StatusCreated,
            wantSuccess:    true,
            checkBody: func(t *testing.T, body []byte) {
                var resp struct {
                    Success bool `json:"success"`
                    Data    struct {
                        ID    string `json:"id"`
                        Name  string `json:"name"`
                        Email string `json:"email"`
                    } `json:"data"`
                }
                if err := json.Unmarshal(body, &resp); err != nil {
                    t.Fatalf("invalid JSON response: %v", err)
                }
                if resp.Data.ID != "uuid-123" {
                    t.Errorf("expected id uuid-123, got %s", resp.Data.ID)
                }
                if resp.Data.Name != "John Doe" {
                    t.Errorf("expected name John Doe, got %s", resp.Data.Name)
                }
            },
        },
        {
            name:           "fail/400 missing name",
            body:           `{"email":"john@example.com","password":"SecurePass123"}`,
            setupMock:      func() *mockUserUseCase { return &mockUserUseCase{} },
            wantStatusCode: http.StatusBadRequest,
            wantSuccess:    false,
        },
        {
            name:           "fail/400 invalid email",
            body:           `{"name":"John","email":"not-email","password":"SecurePass123"}`,
            setupMock:      func() *mockUserUseCase { return &mockUserUseCase{} },
            wantStatusCode: http.StatusBadRequest,
            wantSuccess:    false,
        },
        {
            name: "fail/409 email duplicate",
            body: `{"name":"John","email":"existing@example.com","password":"SecurePass123"}`,
            setupMock: func() *mockUserUseCase {
                return &mockUserUseCase{
                    createUserFn: func(_ context.Context, _ domainUC.CreateUserRequest) (*entity.User, error) {
                        return nil, entity.ErrEmailDuplicate
                    },
                }
            },
            wantStatusCode: http.StatusConflict,
            wantSuccess:    false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            h := handler.NewUserHandler(tt.setupMock())
            router := setupRouter(h)

            req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(tt.body))
            req.Header.Set("Content-Type", "application/json")
            w := httptest.NewRecorder()

            router.ServeHTTP(w, req)

            // Assert status code
            if w.Code != tt.wantStatusCode {
                t.Errorf("expected status %d, got %d. body: %s",
                    tt.wantStatusCode, w.Code, w.Body.String())
            }

            // Assert success field
            var baseResp struct{ Success bool `json:"success"` }
            if err := json.Unmarshal(w.Body.Bytes(), &baseResp); err == nil {
                if baseResp.Success != tt.wantSuccess {
                    t.Errorf("expected success=%v, got %v", tt.wantSuccess, baseResp.Success)
                }
            }

            // Custom body assertions
            if tt.checkBody != nil {
                tt.checkBody(t, w.Body.Bytes())
            }
        })
    }
}
```

> **`httptest` package:** Không cần chạy server thật. `httptest.NewRequest` + `httptest.NewRecorder` + `router.ServeHTTP` = full HTTP request/response cycle trong memory. Tests chạy trong micro-seconds.

---

## §9. Agile Trong Khóa Học

### Cơ Bản Về Agile — Tại Sao Cần?

```
╔══════════════════════════════════════════════════════════════════════╗
║  AGILE MANIFESTO (2001) — 4 GIÁ TRỊ CỐT LÕI & 12 NGUYÊN TẮc           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Individuals & Interactions  >  Processes & Tools                   ║
║  Working Software            >  Comprehensive Documentation         ║
║  Customer Collaboration      >  Contract Negotiation                ║
║  Responding to Change        >  Following a Plan                    ║
║                                                                      ║
║  Vì sao cần Agile trong Software?                                    ║
║  → Requirements thay đổi liên tục (business changes)               ║
║  → Không ai biết chính xác cần build gì sau 6 tháng              ║
║  → Feedback loop ngắn = đi đúng hướng sớm, sửa lỗi sới                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Scrum vs Kanban — Chọn Cái Nào?

| | **Scrum** | **Kanban** |
|--|--|--|
| **Timeboxing** | Sprints cố định (1-4 tuần) | Không có timeboxing |
| **Roles** | PO, Scrum Master, Dev Team | Không quy định |
| **Planning** | Sprint Planning bắt buộc | Tùy chọn |
| **WIP Limit** | Giới hạn qua Sprint capacity | Giới hạn chính thức (WIP limit per column) |
| **Phù hợp** | Feature development, tính năng mới | Bug fixes, support, ops |
| **Trong khóa học** | ✅ Áp dụng cho features | |

### Scrum Trong Khóa Học — Thực Tế

```
╔══════════════════════════════════════════════════════════════════════╗
║  SPRINT STRUCTURE (1-2 tuần / sprint trong khóa học)               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Ngày 1 — Sprint Planning (30 phút):                               ║
║  → Chọn User Stories từ Product Backlog                            ║
║  → Break xuống Tasks (≤ 4 giờ mỗi task — nhỏ để dễ estimate)     ║
║  → Estimate bằng Story Points (Fibonacci: 1,2,3,5,8)               ║
║  → Tạo branch cho Sprint: sprint/1-user-management                 ║
║                                                                      ║
║  Hàng ngày — Daily Standup (15 phút MAX):                          ║
║  → Hôm qua: commit gì? PR nào đã merge?                           ║
║  → Hôm nay: sẽ làm task nào? estimate được đến đâu?             ║
║  → Blocker: có vấn đề gì? cần gì?                                 ║
║                                                                      ║
║  Ngày cuối — Sprint Review (demo):                                  ║
║  → Demo working API (Swagger UI hoặc Postman)                       ║
║  → Chạy `make test` trước mặt người xem                            ║
║  → Show coverage report                                              ║
║                                                                      ║
║  Ngày cuối — Retrospective:                                         ║
║  → Went well: test viết chuẩn hơn, code sạch hơn                   ║
║  → Improve: estimate sai bao nhiêu? Tại sao?                        ║
║  → Action: thêm integration test trước sprint sau                   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Git Workflow Gắn Với Agile

```bash
# Đây là cách team thực tế làm việc với git + agile

# 1. Nhận User Story → tạo branch
git checkout -b feature/US-001-user-registration

# 2. Commit thường xuyên, mỗi commit = 1 logic unit
# Conventional Commits format: <type>(<scope>): <description>
git commit -m "feat(user): add CreateUser use case with validation"
git commit -m "feat(user): add POST /users endpoint with swagger docs"
git commit -m "test(user): add table-driven tests for CreateUser"
git commit -m "docs(user): update README with new endpoint"

# 3. Push và tạo PR (Pull Request)
git push origin feature/US-001-user-registration
# PR description: link User Story, Acceptance Criteria, test coverage %

# 4. CI pipeline tự động chạy:
# make lint && make test && make test-coverage-check
# Fail = không được merge

# 5. Code review + merge vào main/develop
```

**Conventional Commits — Angular Convention:**

| Prefix | Ý nghĩa | Ví dụ |
|--------|---------|--------|
| `feat` | Tính năng mới | `feat(auth): add JWT login` |
| `fix` | Sửa bug | `fix(user): handle nil email in validation` |
| `test` | Thêm/sửa test | `test(user): add handler tests` |
| `refactor` | Refactor (không change behavior) | `refactor(repo): extract query builder` |
| `docs` | Tài liệu | `docs(api): update swagger annotations` |
| `chore` | Config, CI | `chore: update golangci-lint config` |

```
╔══════════════════════════════════════════════════════════════════════╗
║  SPRINT STRUCTURE (2 tuần / sprint)                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Sprint Planning (đầu sprint):                                      ║
║  → Chọn User Stories từ Product Backlog                             ║
║  → Break xuống Tasks (≤ 8 giờ mỗi task)                            ║
║  → Estimate bằng Story Points (Fibonacci: 1,2,3,5,8,13)            ║
║                                                                      ║
║  Daily Standup (15 phút):                                           ║
║  → Hôm qua làm gì?                                                  ║
║  → Hôm nay sẽ làm gì?                                               ║
║  → Có blocker nào không?                                             ║
║                                                                      ║
║  Sprint Review (cuối sprint):                                       ║
║  → Demo working software (không phải slides)                        ║
║  → Stakeholder feedback                                              ║
║                                                                      ║
║  Sprint Retrospective:                                               ║
║  → What went well?                                                   ║
║  → What could be improved?                                          ║
║  → Action items cho sprint sau                                      ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### User Story Format — Chuẩn Agile

```
As a [type of user],
I want to [perform some task],
So that [achieve some goal].

Acceptance Criteria:
- GIVEN [context]
  WHEN [action]
  THEN [expected outcome]

─────────────────────────────────────────────────────────────────────
Ví dụ thực tế trong khóa học:

Story: "User Registration"
As an anonymous visitor,
I want to register an account with email and password,
So that I can access protected features of the application.

Acceptance Criteria:
- GIVEN a valid email and password (min 8 chars)
  WHEN I POST /api/v1/auth/register
  THEN I receive 201 Created with user data (without password)

- GIVEN an already-registered email
  WHEN I POST /api/v1/auth/register
  THEN I receive 409 Conflict with clear error message

- GIVEN invalid email format
  WHEN I POST /api/v1/auth/register
  THEN I receive 400 Bad Request with validation details
```

---

## §10. Bài Tập — Sprint 1

### Product Backlog (Sprint 1)

**Starter:** Project đã được setup với Clean Architecture boilerplate (xem GitHub repo kèm theo).

**Nhiệm vụ của bạn:** Implement các User Stories sau theo đúng chuẩn Clean Architecture:

---

**Story 1 — User Listing** ⭐ (3 Story Points)
```
As an admin,
I want to retrieve a paginated list of users,
So that I can manage user accounts.

Acceptance Criteria:
- GET /api/v1/users?page=1&limit=10
- Response bao gồm: data (array), total, page, limit
- Support filter by name (GET /users?name=john)
- Unit tests cho UseCase (mock repository)
- Handler test với httptest
```

**Story 2 — User Update** ⭐ (2 Story Points)
```
As an authenticated user,
I want to update my profile (name, email),
So that my information stays current.

Acceptance Criteria:
- PATCH /api/v1/users/:id
- Chỉ update fields được gửi lên (partial update)
- Validate email format nếu email được gửi
- Return 404 nếu user không tồn tại
- Unit + handler tests
```

**Story 3 — Auth Middleware** ⭐⭐ (5 Story Points)
```
As a system,
I want to protect sensitive endpoints with JWT authentication,
So that only authenticated users can access them.

Acceptance Criteria:
- Middleware đọc Bearer token từ Authorization header
- Validate JWT signature và expiry
- Inject user_id vào request context
- Return 401 cho token invalid/expired
- Return 403 nếu user không có quyền
- Unit test cho middleware logic
```

### Definition of Done (DoD)

```
☐ Code implement đúng Clean Architecture (không vi phạm Dependency Rule)
☐ Unit test coverage ≥ 80% cho Use Case layer
☐ Handler tests viết bằng httptest
☐ Swagger annotations đầy đủ
☐ Không có lint errors (make lint pass)
☐ go vet pass
☐ go test -race pass
☐ README cập nhật với endpoint mới
```

---

## Tổng Kết — Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║  SAU BÀI HỌC NÀY, BẠN NẮM ĐƯỢC:                                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ✅ Project structure chuẩn production (cmd/, internal/, pkg/)      ║
║                                                                      ║
║  ✅ Clean Architecture: Domain → UseCase → Repository → Delivery    ║
║     → Dependency Rule: phụ thuộc hướng vào trong                   ║
║     → Interfaces làm cầu nối giữa các tầng                         ║
║                                                                      ║
║  ✅ Unit Testing không cần DB:                                      ║
║     → Mock interface implementation                                  ║
║     → Table-driven tests                                             ║
║     → Compile-time interface check                                   ║
║                                                                      ║
║  ✅ Config với env vars (12-Factor App)                             ║
║     → Không hardcode, không commit secret                           ║
║                                                                      ║
║  ✅ OpenAPI docs với swaggo (annotation-driven)                     ║
║                                                                      ║
║  ✅ Makefile automation (dev, test, lint, docs, migrate)            ║
║                                                                      ║
║  ✅ go test -race, -cover, table-driven, httptest                   ║
║                                                                      ║
║  ✅ Agile: User Story, Sprint, Acceptance Criteria, DoD             ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

> **Câu hỏi để tự kiểm tra:**
>
> 1. Nếu bạn muốn đổi từ Gin sang Fiber, bạn phải sửa code ở tầng nào?
> 2. Nếu bạn muốn đổi từ PostgreSQL sang MongoDB, bạn phải sửa code ở tầng nào?
> 3. Tại sao `internal/domain/` không được import bất kỳ package nào của bạn?
> 4. `var _ UserRepository = (*mockUserRepo)(nil)` có tác dụng gì?
> 5. Sự khác biệt giữa `-covermode=set` và `-covermode=atomic` là gì?

---

*📖 Next: Lesson 02 — Database Integration, Migrations & Repository Pattern Deep Dive*
