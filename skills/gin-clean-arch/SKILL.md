---
name: gin-clean-arch
description: "Clean Architecture for Go Gin APIs. Covers layer separation (entity, usecase, repository, delivery), dependency injection, repository pattern, error handling, and testable project structure. Use when organizing Go/Gin projects, implementing clean/hexagonal architecture, setting up dependency injection, or structuring code for testability."
license: MIT
metadata:
  author: henriqueatila
  version: "1.0.0"
---

# gin-clean-arch — Clean Architecture for Go/Gin

Build Go/Gin APIs with strict layer separation, dependency injection, and testable structure. Covers the 80% of clean architecture patterns you need daily. Works standalone; enhanced with **gin-best-practices** for Gin-specific patterns.

## When to Use

- Starting or refactoring a Go/Gin project with clean architecture
- Setting up dependency injection without frameworks
- Separating business logic from framework code for testability

## 5 Golden Rules

**These are non-negotiable. Every code generation must follow them.**

1. **Gin is a detail — isolate it completely.** The package `github.com/gin-gonic/gin` and `*gin.Context` can ONLY exist in the outermost layer (`internal/delivery/http`). Entities and UseCases MUST NOT import Gin or know they are building a web API. If you swap Gin for Fiber, net/http, or a CLI, business logic requires zero changes.

2. **The Database is a detail — don't pollute UseCases.** No SQL, GORM, SQLC, or database driver code inside Entities or UseCases. UseCases call only repository interfaces (e.g., `ProductRepository`). The concrete implementation with queries lives in `internal/repository/`.

3. **Follow the Dependency Rule strictly.** Source code depends only on inner layers. Delivery calls UseCases. UseCases coordinate Entities. Entities know nothing about the outside world.

4. **Separate Request/Response models from Domain models.** Never pass Gin request structs (with `json`/`binding` tags) or DB models to UseCases. The Controller receives the HTTP request, maps it to a plain DTO, and passes that DTO to the UseCase.

5. **`main.go` is the only dirty component.** Dependency injection happens exclusively in `cmd/api/main.go`. It instantiates the DB, creates the concrete repository, injects it into the UseCase, injects the UseCase into the Gin Controller, and starts the server. It is the only file allowed to know the entire system.

## Project Structure

```
myapp/
├── cmd/
│   └── api/
│       └── main.go              # Entry point — DI wiring
├── internal/
│   ├── domain/                  # Entities, interfaces, errors (innermost)
│   ├── usecase/                 # Business logic (depends on domain only)
│   ├── repository/              # Data access — SQLC or GORM (implements domain interfaces)
│   └── delivery/
│       └── http/                # Gin handlers + routes (outermost)
├── pkg/
│   └── httpserver/              # Reusable HTTP server wrapper
├── config/                      # Configuration (YAML + env)
├── migrations/                  # SQL migration files
└── go.mod
```

## The 4 Layers

```
┌─────────────────────────────────────────┐
│           Delivery (HTTP/Gin)           │  ← outermost
│  ┌─────────────────────────────────┐    │
│  │         Repository (DB)         │    │
│  │  ┌─────────────────────────┐    │    │
│  │  │     Usecase (Logic)     │    │    │
│  │  │  ┌─────────────────┐    │    │    │
│  │  │  │  Domain (Core)  │    │    │    │
│  │  │  └─────────────────┘    │    │    │
│  │  └─────────────────────────┘    │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

| Layer | Package | Can Import | Never Imports |
|-------|---------|------------|---------------|
| Domain | `internal/domain` | stdlib only | usecase, repository, delivery |
| Usecase | `internal/usecase` | domain | repository, delivery, gin |
| Repository | `internal/repository` | domain | delivery, gin |
| Delivery | `internal/delivery/http` | domain | repository, usecase (concrete) |

## Layer 1: Domain (Entities + Interfaces)

Pure Go types. No framework dependencies. Defines what the system does, not how.

```go
// internal/domain/product.go
package domain

import (
    "context"
    "time"

    "github.com/google/uuid"
)

type Product struct {
    ID          uuid.UUID
    Name        string
    Description string
    Price       int64 // cents
    Stock       int32
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

type ProductFilter struct {
    Name   string
    MinPrice int64
    MaxPrice int64
    Page     int
    Limit    int
}

type ProductRepository interface {
    FindByID(ctx context.Context, id uuid.UUID) (*Product, error)
    FindAll(ctx context.Context, filter ProductFilter) ([]Product, error)
    Create(ctx context.Context, p *Product) error
    Update(ctx context.Context, p *Product) error
    Delete(ctx context.Context, id uuid.UUID) error
}

type ProductUsecase interface {
    GetProduct(ctx context.Context, id uuid.UUID) (*Product, error)
    ListProducts(ctx context.Context, filter ProductFilter) ([]Product, error)
    CreateProduct(ctx context.Context, input CreateProductInput) (*Product, error)
    UpdateProduct(ctx context.Context, id uuid.UUID, input UpdateProductInput) (*Product, error)
    DeleteProduct(ctx context.Context, id uuid.UUID) error
}

type CreateProductInput struct {
    Name        string
    Description string
    Price       int64
    Stock       int32
}

type UpdateProductInput struct {
    Name        *string
    Description *string
    Price       *int64
    Stock       *int32
}
```

Domain errors — see [Error Flow](#error-flow-across-layers) below.

## Layer 2: Usecase (Business Logic)

Orchestrates domain operations. Depends only on domain interfaces.

```go
// internal/usecase/product_usecase.go
package usecase

import (
    "context"
    "fmt"
    "time"

    "github.com/google/uuid"
    "myapp/internal/domain"
)

type productUsecase struct {
    repo domain.ProductRepository
}

func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
    return &productUsecase{repo: repo}
}

func (u *productUsecase) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    if input.Price <= 0 {
        return nil, fmt.Errorf("create product: %w", domain.ErrValidation)
    }

    product := &domain.Product{
        ID:          uuid.New(),
        Name:        input.Name,
        Description: input.Description,
        Price:       input.Price,
        Stock:       input.Stock,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }

    if err := u.repo.Create(ctx, product); err != nil {
        return nil, fmt.Errorf("create product: %w", err)
    }

    return product, nil
}

func (u *productUsecase) GetProduct(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    product, err := u.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get product %s: %w", id, err)
    }
    return product, nil
}

// ListProducts, UpdateProduct, DeleteProduct follow the same pattern.
```

**Key:** Constructor returns interface (`domain.ProductUsecase`), hides struct. Usecase never imports `gin`, `sql`, or `gorm`.

## Layer 3: Repository (Data Access)

Implements domain interfaces with concrete database technology.

```go
// internal/repository/product_repository.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "github.com/google/uuid"
    "myapp/internal/domain"
)

type postgresProductRepo struct {
    db *sql.DB
}

func NewProductRepository(db *sql.DB) domain.ProductRepository {
    return &postgresProductRepo{db: db}
}

func (r *postgresProductRepo) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    var p domain.Product
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name, description, price, stock, created_at, updated_at
         FROM products WHERE id = $1`, id,
    ).Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Stock, &p.CreatedAt, &p.UpdatedAt)

    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("product %s: %w", id, domain.ErrNotFound)
    }
    if err != nil {
        return nil, fmt.Errorf("query product %s: %w", id, err)
    }
    return &p, nil
}

func (r *postgresProductRepo) Create(ctx context.Context, p *domain.Product) error {
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO products (id, name, description, price, stock, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        p.ID, p.Name, p.Description, p.Price, p.Stock, p.CreatedAt, p.UpdatedAt,
    )
    if err != nil {
        return fmt.Errorf("insert product: %w", err)
    }
    return nil
}
```

Repository wraps `sql.ErrNoRows` → `domain.ErrNotFound`. Database errors never leak to usecase.

For SQLC and GORM implementations, see [references/repository-pattern.md](references/repository-pattern.md).

## Layer 4: Delivery (HTTP/Gin Handlers)

Thin handlers: bind → call usecase → respond. No business logic.

```go
// internal/delivery/http/product_handler.go
package http

import (
    "errors"
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "myapp/internal/domain"
)

type ProductHandler struct {
    uc     domain.ProductUsecase
    logger *slog.Logger
}

func NewProductHandler(uc domain.ProductUsecase, logger *slog.Logger) *ProductHandler {
    return &ProductHandler{uc: uc, logger: logger}
}

type createProductRequest struct {
    Name        string `json:"name"        binding:"required,min=2,max=200"`
    Description string `json:"description" binding:"max=1000"`
    Price       int64  `json:"price"       binding:"required,gt=0"`
    Stock       int32  `json:"stock"       binding:"min=0"`
}

type productResponse struct {
    ID          string `json:"id"`
    Name        string `json:"name"`
    Description string `json:"description"`
    PriceCents  int64  `json:"price_cents"`
    Stock       int32  `json:"stock"`
    CreatedAt   string `json:"created_at"`
}

func toProductResponse(p *domain.Product) productResponse {
    return productResponse{
        ID: p.ID.String(), Name: p.Name, Description: p.Description,
        PriceCents: p.Price, Stock: p.Stock,
        CreatedAt: p.CreatedAt.Format("2006-01-02T15:04:05Z"),
    }
}

func (h *ProductHandler) Create(c *gin.Context) {
    var req createProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "validation failed"})
        return
    }

    product, err := h.uc.CreateProduct(c.Request.Context(), domain.CreateProductInput{
        Name:        req.Name,
        Description: req.Description,
        Price:       req.Price,
        Stock:       req.Stock,
    })
    if err != nil {
        handleError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusCreated, toProductResponse(product))
}

func RegisterProductRoutes(rg *gin.RouterGroup, h *ProductHandler) {
    products := rg.Group("/products")
    {
        products.POST("", h.Create)
        products.GET("/:id", h.GetByID)
        products.GET("", h.List)
        products.PUT("/:id", h.Update)
        products.DELETE("/:id", h.Delete)
    }
}
```

## Dependency Injection Bootstrap

Wire everything in `main.go` — no framework, explicit and debuggable.

```go
// cmd/api/main.go
package main

import (
    "context"
    "database/sql"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
    _ "github.com/lib/pq"

    "myapp/config"
    delivery "myapp/internal/delivery/http"
    "myapp/internal/repository"
    "myapp/internal/usecase"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    cfg, err := config.Load()
    if err != nil { logger.Error("load config", "error", err); os.Exit(1) }

    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil { logger.Error("open db", "error", err); os.Exit(1) }
    defer db.Close()
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)

    // DI wiring: repo → usecase → handler
    productRepo := repository.NewProductRepository(db)
    productUC := usecase.NewProductUsecase(productRepo)
    productHandler := delivery.NewProductHandler(productUC, logger)

    r := gin.New()
    r.Use(gin.Recovery())
    delivery.RegisterProductRoutes(r.Group("/api/v1"), productHandler)

    srv := &http.Server{Addr: cfg.Port, Handler: r}
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Error("server error", "error", err); os.Exit(1)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil { logger.Error("shutdown", "error", err) }
}
```

**DI flow:** `config → db → repository → usecase → handler → router → server`

For DI container pattern and scaling, see [references/dependency-injection.md](references/dependency-injection.md).

## Error Flow Across Layers

```
Repository          Usecase             Delivery
sql.ErrNoRows   →   domain.ErrNotFound  →  404 JSON
unique violation →   domain.ErrConflict  →  409 JSON
business rule    →   domain.ErrValidation → 422 JSON
unknown          →   wrapped error       →  500 JSON (logged)
```

```go
// internal/domain/errors.go — NO Gin imports here (Rule 1)
package domain

type AppError struct {
    Code    int
    Message string
    Detail  string
}

func (e *AppError) Error() string { return e.Message }

var (
    ErrNotFound   = &AppError{Code: 404, Message: "not found"}
    ErrConflict   = &AppError{Code: 409, Message: "already exists"}
    ErrValidation = &AppError{Code: 422, Message: "validation failed"}
    ErrForbidden  = &AppError{Code: 403, Message: "forbidden"}
)
```

Error-to-HTTP mapping lives in the **delivery layer** (not domain — Gin is a detail):
```go
// internal/delivery/http/errors.go — Gin allowed here (outermost layer)
package http

func handleError(c *gin.Context, err error, logger *slog.Logger) {
    var appErr *domain.AppError
    if errors.As(err, &appErr) {
        resp := gin.H{"error": appErr.Message}
        if appErr.Detail != "" { resp["detail"] = appErr.Detail }
        c.JSON(appErr.Code, resp)
        return
    }
    logger.ErrorContext(c.Request.Context(), "unhandled error", "error", err)
    c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
}
```

For detailed error patterns, see [references/error-handling.md](references/error-handling.md).

## Input Sanitization

Binding tags validate structure but **do not sanitize content**. Sanitize free-text strings (`SanitizeString`) at the delivery boundary after `ShouldBind*` succeeds — before mapping to domain input. SQL parameters (`$1, $2...`) prevent injection. See [references/input-sanitization.md](references/input-sanitization.md).

## Quick Reference

| Question | Answer |
|----------|--------|
| New entity? | `internal/domain/` |
| Business logic? | `internal/usecase/` |
| SQL queries? | `internal/repository/` |
| HTTP handlers? | `internal/delivery/http/` |
| Interfaces? | `internal/domain/` (always) |
| DI wiring? | `cmd/api/main.go` (only dirty place) |

## Reference Files

- **[references/layer-separation.md](references/layer-separation.md)** — Layer responsibilities, dependency rule enforcement
- **[references/layer-separation-antipatterns.md](references/layer-separation-antipatterns.md)** — Anti-patterns (bad→good), migration guide
- **[references/dependency-injection.md](references/dependency-injection.md)** — Manual DI, DI container pattern, testing with DI
- **[references/dependency-injection-alternatives.md](references/dependency-injection-alternatives.md)** — Scaling DI, Wire/Fx alternatives
- **[references/repository-pattern.md](references/repository-pattern.md)** — SQLC, GORM, transactions, query patterns
- **[references/error-handling.md](references/error-handling.md)** — Domain errors, propagation, HTTP mapping, validation errors
- **[references/input-sanitization.md](references/input-sanitization.md)** — Sanitize untrusted strings at delivery boundary
- **[references/testing-by-layer.md](references/testing-by-layer.md)** — Mock-per-layer, testcontainers, table-driven tests, coverage
- **[references/project-scaffolding.md](references/project-scaffolding.md)** — From-scratch setup, Makefile, configuration, graceful shutdown

## Cross-Skill References (gin-best-practices)

Each skill from `henriqueatila/gin-best-practices` complements a specific layer:

- **gin-api** → Delivery layer. `ShouldBind*` variants (query, header, form), custom validators, file uploads, route versioning, CORS/rate-limit middleware, `AppError` system. Load `references/routing.md` for pagination and wildcard routes, `references/middleware.md` for request ID and timeout middleware.
- **gin-auth** → Delivery layer. JWT middleware for protected route groups, login handler, token refresh/blacklist (Redis), RBAC with `RequireRole`. Load `references/jwt-patterns.md` for RS256 vs HS256, `references/rbac.md` for multi-tenant permissions.
- **gin-database** → Repository layer. GORM associations, hooks, soft deletes, sqlx struct scanning, `NamedExec`, migrations with golang-migrate. Load `references/gorm-patterns.md` for GORM CRUD, `references/migrations.md` for zero-downtime migrations.
- **gin-testing** → All layers. `httptest.NewRecorder` patterns, table-driven handler tests, testcontainers for integration, e2e flows with docker-compose. Load `references/unit-tests.md` for mock generation, `references/e2e.md` for full CI pipeline tests.
- **gin-deploy** → Infrastructure. Multi-stage Dockerfile (distroless), docker-compose with Air hot-reload, Kubernetes manifests (Deployment, HPA, Ingress), OpenTelemetry tracing. Load `references/dockerfile.md` for layer caching, `references/kubernetes.md` for Helm charts.
