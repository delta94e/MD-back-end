# Lesson 05 — SQL Database, GORM, Auth & Testing

> **Mục tiêu:** Hiểu cách tích hợp SQL database vào app Go-Clean,
> implement Authentication/Authorization với JWT, và viết test chuyên nghiệp.

---

## Phần 1: GORM + PostgreSQL Setup

### 1.1 GORM Là Gì?

```
GORM = Go Object Relational Mapper
→ Thư viện giúp tương tác với SQL database thông qua Go structs
→ Tự động tạo SQL từ code, xử lý connection pool, migrations

So sánh các approach:
┌──────────────────────┬───────────────────────────────────────────┐
│  database/sql        │ Low-level, viết SQL tay, nhiều boilerplate│
│  sqlx                │ Mỏng hơn gorm, vẫn viết SQL, ít magic     │
│  GORM                │ High-level, auto SQL, migrations, hooks    │
│  sqlc                │ Generate code từ SQL file (type-safe)      │
└──────────────────────┴───────────────────────────────────────────┘
Khóa học dùng GORM vì phổ biến, đủ tính năng cho production
```

### 1.2 Cài Đặt Dependencies

```bash
go get gorm.io/gorm
go get gorm.io/driver/postgres

# Bcrypt cho hash password
go get golang.org/x/crypto/bcrypt

# JWT
go get github.com/golang-jwt/jwt/v5

# Validator
go get github.com/go-playground/validator/v10

# Migration
go get github.com/golang-migrate/migrate/v4
go get github.com/golang-migrate/migrate/v4/database/postgres
go get github.com/golang-migrate/migrate/v4/source/file
```

### 1.3 Kết Nối Database

```go
// pkg/database/postgres.go
package database

import (
    "fmt"
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type Config struct {
    Host     string
    Port     int
    Name     string
    User     string
    Password string
    SSLMode  string
}

func NewPostgresDB(cfg Config, isProd bool) (*gorm.DB, error) {
    dsn := fmt.Sprintf(
        "host=%s port=%d dbname=%s user=%s password=%s sslmode=%s TimeZone=UTC",
        cfg.Host, cfg.Port, cfg.Name, cfg.User, cfg.Password, cfg.SSLMode,
    )

    logLevel := logger.Info
    if isProd {
        logLevel = logger.Warn  // production: chỉ log slow queries
    }

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger:                                   logger.Default.LogMode(logLevel),
        PrepareStmt:                              true,   // cache prepared statements
        DisableForeignKeyConstraintWhenMigrating: false,
    })
    if err != nil {
        return nil, fmt.Errorf("gorm open: %w", err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, fmt.Errorf("get sql.DB: %w", err)
    }

    // Connection pool settings
    sqlDB.SetMaxOpenConns(25)
    sqlDB.SetMaxIdleConns(5)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)
    sqlDB.SetConnMaxIdleTime(1 * time.Minute)

    return db, nil
}
```

---

## Phần 2: Domain Entities & Migrations

### 2.1 User Entity

```go
// internal/domain/entity/user.go
package entity

import (
    "time"
)

type User struct {
    ID          string    `json:"id"           gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    Name        string    `json:"name"         gorm:"not null"`
    Email       string    `json:"email"        gorm:"uniqueIndex;not null"`
    Password    string    `json:"-"            gorm:"not null"`         // json:"-" = không trả về trong JSON
    Role        UserRole  `json:"role"         gorm:"type:varchar(20);default:'user'"`
    IsActive    bool      `json:"is_active"    gorm:"default:true"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

type UserRole string

const (
    RoleUser  UserRole = "user"
    RoleAdmin UserRole = "admin"
)

func (User) TableName() string {
    return "users"
}
```

### 2.2 Database Migrations

```bash
# Tạo migration files
migrate create -ext sql -dir migrations -seq create_users_table
# → migrations/000001_create_users_table.up.sql
# → migrations/000001_create_users_table.down.sql
```

```sql
-- migrations/000001_create_users_table.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";  -- cho gen_random_uuid()

CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    role        VARCHAR(20)  NOT NULL DEFAULT 'user',
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

```sql
-- migrations/000001_create_users_table.down.sql
DROP TABLE IF EXISTS users;
```

```go
// pkg/database/migrate.go
package database

import (
    "fmt"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(dsn string) error {
    m, err := migrate.New("file://migrations", dsn)
    if err != nil {
        return fmt.Errorf("migrate new: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("migrate up: %w", err)
    }
    return nil
}
```

---

## Phần 3: Repository Với GORM

### 3.1 Interface (Domain Layer)

```go
// internal/domain/repository/user_repository.go
package repository

import (
    "context"

    "github.com/yourorg/app/internal/domain/entity"
)

//go:generate mockgen -source=user_repository.go -destination=../../../mocks/user_repository_mock.go -package=mocks
type UserRepository interface {
    Create(ctx context.Context, user *entity.User) error
    FindByID(ctx context.Context, id string) (*entity.User, error)
    FindByEmail(ctx context.Context, email string) (*entity.User, error)
    Update(ctx context.Context, user *entity.User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, offset, limit int) ([]*entity.User, int64, error)
}
```

### 3.2 GORM Implementation

```go
// internal/repository/postgres/user_repository.go
package postgres

import (
    "context"
    "errors"
    "fmt"

    "gorm.io/gorm"

    "github.com/yourorg/app/internal/domain/entity"
    "github.com/yourorg/app/internal/domain/repository"
)

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) repository.UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *entity.User) error {
    result := r.db.WithContext(ctx).Create(user)
    if result.Error != nil {
        return mapGORMError(result.Error, "Create")
    }
    return nil
}

func (r *userRepository) FindByID(ctx context.Context, id string) (*entity.User, error) {
    var user entity.User
    result := r.db.WithContext(ctx).First(&user, "id = ?", id)
    if result.Error != nil {
        return nil, mapGORMError(result.Error, "FindByID")
    }
    return &user, nil
}

func (r *userRepository) FindByEmail(ctx context.Context, email string) (*entity.User, error) {
    var user entity.User
    result := r.db.WithContext(ctx).Where("email = ?", email).First(&user)
    if result.Error != nil {
        return nil, mapGORMError(result.Error, "FindByEmail")
    }
    return &user, nil
}

func (r *userRepository) List(ctx context.Context, offset, limit int) ([]*entity.User, int64, error) {
    var users []*entity.User
    var total int64

    if err := r.db.WithContext(ctx).Model(&entity.User{}).Count(&total).Error; err != nil {
        return nil, 0, fmt.Errorf("List count: %w", err)
    }

    result := r.db.WithContext(ctx).
        Order("created_at DESC").
        Offset(offset).
        Limit(limit).
        Find(&users)
    if result.Error != nil {
        return nil, 0, fmt.Errorf("List find: %w", result.Error)
    }
    return users, total, nil
}

// Compile-time check
var _ repository.UserRepository = (*userRepository)(nil)
```

### 3.3 DB Error Mapping

```go
// internal/repository/postgres/errors.go
package postgres

import (
    "errors"
    "fmt"
    "strings"

    "github.com/jackc/pgx/v5/pgconn"
    "gorm.io/gorm"

    domainerr "github.com/yourorg/app/internal/domain/errors"
)

// mapGORMError chuyển GORM/Postgres errors thành domain errors
// → UseCase layer chỉ biết domain errors, không biết postgres
func mapGORMError(err error, op string) error {
    if err == nil {
        return nil
    }

    // GORM Not Found
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return fmt.Errorf("%s: %w", op, domainerr.ErrNotFound)
    }

    // PostgreSQL error codes
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        switch pgErr.Code {
        case "23505": // unique_violation
            // Trích xuất field bị duplicate từ constraint name
            field := extractFieldFromConstraint(pgErr.ConstraintName)
            return fmt.Errorf("%s: %w: %s already exists", op, domainerr.ErrConflict, field)
        case "23503": // foreign_key_violation
            return fmt.Errorf("%s: %w: invalid reference", op, domainerr.ErrBadRequest)
        case "23502": // not_null_violation
            return fmt.Errorf("%s: %w: %s is required", op, domainerr.ErrBadRequest, pgErr.ColumnName)
        }
    }

    return fmt.Errorf("%s: %w", op, err)
}

func extractFieldFromConstraint(constraint string) string {
    // "users_email_key" → "email"
    // "idx_users_email" → "email"
    parts := strings.Split(constraint, "_")
    if len(parts) >= 2 {
        return parts[len(parts)-2]
    }
    return "field"
}
```

```go
// internal/domain/errors/errors.go
package errors

import "errors"

var (
    ErrNotFound     = errors.New("not found")
    ErrConflict     = errors.New("conflict")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
    ErrBadRequest   = errors.New("bad request")
    ErrInternal     = errors.New("internal error")
)
```

---

## Phần 4: User Use Cases

### 4.1 Register User

```go
// internal/usecase/auth_usecase.go
package usecase

import (
    "context"
    "errors"
    "fmt"

    "golang.org/x/crypto/bcrypt"

    "github.com/yourorg/app/internal/domain/entity"
    domainerr "github.com/yourorg/app/internal/domain/errors"
    "github.com/yourorg/app/internal/domain/repository"
)

type RegisterRequest struct {
    Name     string `validate:"required,min=2,max=100"`
    Email    string `validate:"required,email"`
    Password string `validate:"required,min=8"`
}

type LoginRequest struct {
    Email    string `validate:"required,email"`
    Password string `validate:"required"`
}

type AuthResponse struct {
    User        *entity.User `json:"user"`
    AccessToken string       `json:"access_token"`
}

type authUseCase struct {
    userRepo repository.UserRepository
    jwtSvc   JWTService
}

func (uc *authUseCase) Register(ctx context.Context, req RegisterRequest) (*entity.User, error) {
    // 1. Kiểm tra email đã tồn tại chưa
    existing, err := uc.userRepo.FindByEmail(ctx, req.Email)
    if err != nil && !errors.Is(err, domainerr.ErrNotFound) {
        return nil, fmt.Errorf("Register: check email: %w", err)
    }
    if existing != nil {
        return nil, fmt.Errorf("Register: %w: email already registered", domainerr.ErrConflict)
    }

    // 2. Hash password (bcrypt cost=12 cho production)
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), 12)
    if err != nil {
        return nil, fmt.Errorf("Register: hash password: %w", err)
    }

    // 3. Tạo user entity
    user := &entity.User{
        Name:     req.Name,
        Email:    req.Email,
        Password: string(hashedPassword),
        Role:     entity.RoleUser,
    }

    if err := uc.userRepo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("Register: create: %w", err)
    }

    return user, nil
}

func (uc *authUseCase) Login(ctx context.Context, req LoginRequest) (*AuthResponse, error) {
    // 1. Tìm user theo email
    user, err := uc.userRepo.FindByEmail(ctx, req.Email)
    if err != nil {
        if errors.Is(err, domainerr.ErrNotFound) {
            // Trả về cùng lỗi cho cả wrong email và wrong password
            // → tránh user enumeration attack
            return nil, fmt.Errorf("Login: %w", domainerr.ErrUnauthorized)
        }
        return nil, fmt.Errorf("Login: find user: %w", err)
    }

    // 2. Verify password
    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
        return nil, fmt.Errorf("Login: %w", domainerr.ErrUnauthorized)
    }

    // 3. Tạo JWT token
    token, err := uc.jwtSvc.Generate(user)
    if err != nil {
        return nil, fmt.Errorf("Login: generate token: %w", err)
    }

    return &AuthResponse{User: user, AccessToken: token}, nil
}
```

---

## Phần 5: JWT Authentication

### 5.1 JWT Service

```go
// pkg/jwt/jwt.go
package jwt

import (
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"

    "github.com/yourorg/app/internal/domain/entity"
)

type Claims struct {
    UserID string          `json:"user_id"`
    Email  string          `json:"email"`
    Role   entity.UserRole `json:"role"`
    jwt.RegisteredClaims
}

type Service struct {
    secretKey []byte
    ttl       time.Duration
}

func NewService(secretKey string, ttl time.Duration) *Service {
    return &Service{
        secretKey: []byte(secretKey),
        ttl:       ttl,
    }
}

func (s *Service) Generate(user *entity.User) (string, error) {
    claims := Claims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(s.ttl)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(s.secretKey)
    if err != nil {
        return "", fmt.Errorf("sign token: %w", err)
    }
    return signed, nil
}

func (s *Service) Validate(tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return s.secretKey, nil
    })

    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, fmt.Errorf("%w: token expired", ErrInvalidToken)
        }
        return nil, fmt.Errorf("%w: %v", ErrInvalidToken, err)
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, ErrInvalidToken
    }
    return claims, nil
}

var ErrInvalidToken = errors.New("invalid token")
```

### 5.2 Auth Middleware

```go
// internal/delivery/http/middleware/auth.go
package middleware

import (
    "errors"
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"

    "github.com/yourorg/app/internal/domain/entity"
    "github.com/yourorg/app/pkg/jwt"
)

const (
    ContextKeyUserID = "user_id"
    ContextKeyUser   = "user_claims"
)

// RequireAuth — middleware xác thực JWT
func RequireAuth(jwtSvc *jwt.Service) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Lấy token từ header: "Authorization: Bearer <token>"
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "authorization header required",
            })
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "invalid authorization format, expected: Bearer <token>",
            })
            return
        }

        claims, err := jwtSvc.Validate(parts[1])
        if err != nil {
            if errors.Is(err, jwt.ErrInvalidToken) {
                c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": err.Error()})
                return
            }
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "token validation failed"})
            return
        }

        // Lưu claims vào context — handler dùng sau
        c.Set(ContextKeyUserID, claims.UserID)
        c.Set(ContextKeyUser, claims)
        c.Next()
    }
}

// RequireRole — middleware phân quyền theo role
func RequireRole(roles ...entity.UserRole) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims, exists := c.Get(ContextKeyUser)
        if !exists {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "not authenticated"})
            return
        }

        userClaims := claims.(*jwt.Claims)
        for _, role := range roles {
            if userClaims.Role == role {
                c.Next()
                return
            }
        }

        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
            "error": "insufficient permissions",
        })
    }
}

// Helper lấy claims từ context trong handler
func GetClaims(c *gin.Context) *jwt.Claims {
    v, _ := c.Get(ContextKeyUser)
    claims, _ := v.(*jwt.Claims)
    return claims
}
```

### 5.3 Route Registration Với Middleware

```go
// internal/delivery/http/router.go
func NewRouter(
    authHandler *AuthHandler,
    userHandler *UserHandler,
    jwtSvc *jwt.Service,
) *gin.Engine {
    r := gin.New()
    r.Use(gin.Recovery())
    r.Use(middleware.Logger())
    r.Use(middleware.CORS())

    // Public routes — không cần auth
    public := r.Group("/api/v1")
    {
        public.POST("/auth/register", authHandler.Register)
        public.POST("/auth/login",    authHandler.Login)
        public.GET("/health",         healthHandler)
    }

    // Protected routes — cần JWT
    protected := r.Group("/api/v1")
    protected.Use(middleware.RequireAuth(jwtSvc))
    {
        protected.GET("/users/me",       userHandler.GetMe)
        protected.PUT("/users/me",       userHandler.UpdateMe)
        protected.POST("/auth/logout",   authHandler.Logout)
    }

    // Admin routes — cần JWT + role admin
    admin := r.Group("/api/v1/admin")
    admin.Use(middleware.RequireAuth(jwtSvc))
    admin.Use(middleware.RequireRole(entity.RoleAdmin))
    {
        admin.GET("/users",        userHandler.ListUsers)
        admin.DELETE("/users/:id", userHandler.DeleteUser)
    }

    return r
}
```

---

## Phần 6: Request Validation

```go
// internal/delivery/http/handler/auth_handler.go
package handler

import (
    "errors"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"

    domainerr "github.com/yourorg/app/internal/domain/errors"
    "github.com/yourorg/app/internal/usecase"
)

var validate = validator.New()

type AuthHandler struct {
    authUC usecase.AuthUseCase
}

type RegisterRequest struct {
    Name     string `json:"name"     validate:"required,min=2,max=100"`
    Email    string `json:"email"    validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

func (h *AuthHandler) Register(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid JSON: " + err.Error()})
        return
    }

    // Validate fields
    if err := validate.Struct(req); err != nil {
        var valErrs validator.ValidationErrors
        if errors.As(err, &valErrs) {
            c.JSON(http.StatusBadRequest, gin.H{
                "error":  "validation failed",
                "fields": formatValidationErrors(valErrs),
            })
            return
        }
    }

    user, err := h.authUC.Register(c.Request.Context(), usecase.RegisterRequest{
        Name:     req.Name,
        Email:    req.Email,
        Password: req.Password,
    })
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, gin.H{"data": user})
}

// formatValidationErrors chuyển validation errors thành JSON thân thiện
func formatValidationErrors(errs validator.ValidationErrors) map[string]string {
    fields := make(map[string]string, len(errs))
    for _, e := range errs {
        fields[e.Field()] = validationMessage(e)
    }
    return fields
}

func validationMessage(e validator.FieldError) string {
    switch e.Tag() {
    case "required":
        return "field is required"
    case "email":
        return "must be a valid email address"
    case "min":
        return fmt.Sprintf("must be at least %s characters", e.Param())
    case "max":
        return fmt.Sprintf("must be at most %s characters", e.Param())
    default:
        return fmt.Sprintf("failed validation: %s", e.Tag())
    }
}

// handleError map domain errors → HTTP status codes
func handleError(c *gin.Context, err error) {
    switch {
    case errors.Is(err, domainerr.ErrNotFound):
        c.JSON(http.StatusNotFound, gin.H{"error": "resource not found"})
    case errors.Is(err, domainerr.ErrConflict):
        c.JSON(http.StatusConflict, gin.H{"error": err.Error()})
    case errors.Is(err, domainerr.ErrUnauthorized):
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
    case errors.Is(err, domainerr.ErrForbidden):
        c.JSON(http.StatusForbidden, gin.H{"error": "access denied"})
    case errors.Is(err, domainerr.ErrBadRequest):
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    default:
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
    }
}
```

---

## Phần 7: Testing — Unit Test + Integration Test + Fixtures

### 7.1 Unit Test UseCase Với Mock

```go
// internal/usecase/auth_usecase_test.go
package usecase_test

import (
    "context"
    "testing"

    "go.uber.org/mock/gomock"
    "golang.org/x/crypto/bcrypt"

    domainerr "github.com/yourorg/app/internal/domain/errors"
    "github.com/yourorg/app/internal/usecase"
    "github.com/yourorg/app/mocks"
)

func TestAuthUseCase_Register(t *testing.T) {
    tests := []struct {
        name      string
        req       usecase.RegisterRequest
        setupMock func(*mocks.MockUserRepository)
        wantErr   error
    }{
        {
            name: "success",
            req: usecase.RegisterRequest{
                Name: "Alice", Email: "alice@example.com", Password: "password123",
            },
            setupMock: func(m *mocks.MockUserRepository) {
                // FindByEmail trả về ErrNotFound (email chưa tồn tại)
                m.EXPECT().FindByEmail(gomock.Any(), "alice@example.com").
                    Return(nil, domainerr.ErrNotFound)
                // Create được gọi với user có password đã hash
                m.EXPECT().Create(gomock.Any(), gomock.MatchedBy(func(u any) bool {
                    // Validate password được hash (không phải plaintext)
                    // return true nếu u có password field là bcrypt hash
                    return true
                })).Return(nil)
            },
        },
        {
            name: "email already exists",
            req: usecase.RegisterRequest{
                Name: "Alice", Email: "alice@example.com", Password: "password123",
            },
            setupMock: func(m *mocks.MockUserRepository) {
                m.EXPECT().FindByEmail(gomock.Any(), "alice@example.com").
                    Return(&entity.User{Email: "alice@example.com"}, nil)
            },
            wantErr: domainerr.ErrConflict,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            mockRepo := mocks.NewMockUserRepository(ctrl)
            tt.setupMock(mockRepo)

            uc := usecase.NewAuthUseCase(mockRepo, fakeJWTSvc{})
            _, err := uc.Register(context.Background(), tt.req)

            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("got error %v, want %v", err, tt.wantErr)
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
        })
    }
}
```

### 7.2 Test Fixtures Pattern

```go
// testutil/fixtures/user_fixture.go
package fixtures

import (
    "context"
    "testing"
    "time"

    "golang.org/x/crypto/bcrypt"
    "gorm.io/gorm"

    "github.com/yourorg/app/internal/domain/entity"
)

// UserFixture — factory tạo test data nhất quán
type UserFixture struct {
    db *gorm.DB
}

func NewUserFixture(db *gorm.DB) *UserFixture {
    return &UserFixture{db: db}
}

// Defaults — user với giá trị mặc định, có thể override
type UserOptions struct {
    Name     string
    Email    string
    Password string
    Role     entity.UserRole
    IsActive bool
}

func (f *UserFixture) CreateUser(t *testing.T, opts ...func(*UserOptions)) *entity.User {
    t.Helper()

    // Giá trị mặc định
    o := &UserOptions{
        Name:     "Test User",
        Email:    fmt.Sprintf("user-%d@test.com", time.Now().UnixNano()),
        Password: "password123",
        Role:     entity.RoleUser,
        IsActive: true,
    }
    for _, opt := range opts {
        opt(o)
    }

    hash, err := bcrypt.GenerateFromPassword([]byte(o.Password), 4) // cost=4 cho test (nhanh hơn)
    if err != nil {
        t.Fatalf("fixture: hash password: %v", err)
    }

    user := &entity.User{
        Name:     o.Name,
        Email:    o.Email,
        Password: string(hash),
        Role:     o.Role,
        IsActive: o.IsActive,
    }

    if err := f.db.WithContext(context.Background()).Create(user).Error; err != nil {
        t.Fatalf("fixture: create user: %v", err)
    }

    // Dọn dẹp sau mỗi test
    t.Cleanup(func() {
        f.db.Unscoped().Delete(user)
    })

    return user
}

// Convenience functions
func WithEmail(email string) func(*UserOptions) {
    return func(o *UserOptions) { o.Email = email }
}

func WithRole(role entity.UserRole) func(*UserOptions) {
    return func(o *UserOptions) { o.Role = role }
}

func WithPassword(password string) func(*UserOptions) {
    return func(o *UserOptions) { o.Password = password }
}
```

### 7.3 Integration Test Với Fixtures

```go
// internal/repository/postgres/user_repository_integration_test.go
//go:build integration

package postgres_test

import (
    "context"
    "testing"

    "gorm.io/gorm"

    "github.com/yourorg/app/internal/domain/entity"
    domainerr "github.com/yourorg/app/internal/domain/errors"
    repopostgres "github.com/yourorg/app/internal/repository/postgres"
    "github.com/yourorg/app/testutil/fixtures"
    "github.com/yourorg/app/testutil/testdb"
)

func TestUserRepository_Integration(t *testing.T) {
    db := testdb.Setup(t) // kết nối test DB, tự cleanup
    fix := fixtures.NewUserFixture(db)
    repo := repopostgres.NewUserRepository(db)
    ctx := context.Background()

    t.Run("Create và FindByID thành công", func(t *testing.T) {
        user := fix.CreateUser(t, fixtures.WithEmail("find@test.com"))

        found, err := repo.FindByID(ctx, user.ID)
        if err != nil {
            t.Fatalf("FindByID: %v", err)
        }
        if found.Email != user.Email {
            t.Errorf("got email %s, want %s", found.Email, user.Email)
        }
    })

    t.Run("FindByEmail không tồn tại → ErrNotFound", func(t *testing.T) {
        _, err := repo.FindByEmail(ctx, "notexist@test.com")
        if !errors.Is(err, domainerr.ErrNotFound) {
            t.Errorf("expected ErrNotFound, got %v", err)
        }
    })

    t.Run("Create email trùng → ErrConflict", func(t *testing.T) {
        fix.CreateUser(t, fixtures.WithEmail("duplicate@test.com"))

        duplicate := &entity.User{
            Name:     "Another",
            Email:    "duplicate@test.com",
            Password: "hash",
            Role:     entity.RoleUser,
        }
        err := repo.Create(ctx, duplicate)
        if !errors.Is(err, domainerr.ErrConflict) {
            t.Errorf("expected ErrConflict, got %v", err)
        }
    })
}
```

```go
// testutil/testdb/testdb.go
package testdb

import (
    "os"
    "testing"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "github.com/yourorg/app/internal/domain/entity"
)

func Setup(t *testing.T) *gorm.DB {
    t.Helper()

    dsn := os.Getenv("TEST_DB_DSN")
    if dsn == "" {
        dsn = "host=localhost port=5432 user=testuser password=testpass dbname=testdb sslmode=disable"
    }

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("testdb: connect: %v", err)
    }

    // Auto migrate cho test
    if err := db.AutoMigrate(&entity.User{}); err != nil {
        t.Fatalf("testdb: migrate: %v", err)
    }

    t.Cleanup(func() {
        sqlDB, _ := db.DB()
        sqlDB.Close()
    })

    return db
}
```

---

## Phần 8: Bài Tập

### Bài 1 — Product Table

Tạo `products` table và implement CRUD:
- Entity: `Product{ID, Name, Description, Price, Stock, CategoryID, CreatedAt}`
- Migration với foreign key → `categories` table
- Repository với GORM  
- UseCase: Create, GetByID, List (với pagination), Update, Delete
- Unit test với mock + integration test với fixture

### Bài 2 — Refresh Token

Mở rộng auth với refresh token:
- `POST /auth/refresh` → nhận refresh token, trả về access token mới
- Lưu refresh token trong Redis với TTL 7 ngày
- Blacklist refresh token khi logout
- Access token TTL: 15 phút
- Unit test đầy đủ

### Bài 3 — Change Password

Implement endpoint:
- `PUT /users/me/password`
- Body: `{current_password, new_password, confirm_password}`
- Validate: new_password === confirm_password
- Verify current_password với bcrypt
- Hash new_password và update

### Bài 4 — Admin User Management

Implement admin endpoints (cần role=admin):
- `GET /admin/users?page=1&limit=20` → list users có pagination
- `GET /admin/users/:id` → get user detail
- `PUT /admin/users/:id/role` → thay đổi role
- `DELETE /admin/users/:id` → soft delete (set is_active=false)
- Viết integration test với fixture: tạo admin user và regular user

---

## Tổng Kết Lesson 05

```
╔══════════════════════════════════════════════════════════════════════╗
║  LESSON 05 — KEY TAKEAWAYS                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  GORM & Database:                                                   ║
║  + Entity struct với GORM tags → tự động mapping                   ║
║  + Migrations: file-based với golang-migrate (có version control)   ║
║  + Connection pool settings quan trọng ở production                 ║
║  + Map DB errors → domain errors ở repository layer                ║
║                                                                      ║
║  Authentication & Authorization:                                    ║
║  + Authen = "Bạn là ai?" (JWT verify identity)                     ║
║  + Author = "Bạn được làm gì?" (Role-based access control)         ║
║  + bcrypt cost=12 cho production (chậm có chủ ý → brute-force)    ║
║  + JWT: không store server-side, stateless, có expiry              ║
║  + Cùng thông báo lỗi cho wrong email và wrong password            ║
║    → tránh user enumeration attack                                  ║
║                                                                      ║
║  Validation:                                                        ║
║  + Validate ở Delivery layer (handler), không ở UseCase            ║
║  + Trả về lỗi có field name cụ thể → UX tốt hơn                   ║
║  + handleError() map domain → HTTP một nơi duy nhất                ║
║                                                                      ║
║  Testing:                                                           ║
║  + Unit test: mock UserRepository, test từng use case riêng biệt   ║
║  + Integration test: DB thật, fixtures tạo test data nhất quán     ║
║  + Fixture pattern: factory + cleanup tự động (t.Cleanup)          ║
║  + bcrypt cost=4 trong test → chạy nhanh hơn                       ║
╚══════════════════════════════════════════════════════════════════════╝
```
