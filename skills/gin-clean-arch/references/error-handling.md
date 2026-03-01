# Error Handling — Deep Dive

Quick overview: [SKILL.md](../SKILL.md).

## Table of Contents

1. [Domain Error Types](#1-domain-error-types)
2. [Error Propagation Rules](#2-error-propagation-rules)
3. [Repository Error Wrapping](#3-repository-error-wrapping)
4. [Usecase Error Enrichment](#4-usecase-error-enrichment)
5. [HTTP Error Mapping Middleware](#5-http-error-mapping-middleware)
6. [Validation Errors](#6-validation-errors)
7. [Logging Strategy](#7-logging-strategy)
8. [Panic Recovery](#8-panic-recovery)

---

## 1. Domain Error Types

All error types in `internal/domain/errors.go`. Stdlib only — no framework imports, no `encoding/json`.

```go
// internal/domain/errors.go
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

Check errors: `errors.As(err, &appErr)` to extract; `errors.Is(err, domain.ErrNotFound)` for sentinel identity.
Anti-pattern: `if err.Error() == "not found"` (breaks with wrapping).

**Adding detail to errors** — use `fmt.Errorf` to wrap the sentinel with context. Detail stays in the error chain, not in the struct:

```go
// In usecase — adds context without changing AppError struct
return nil, fmt.Errorf("price must be greater than zero: %w", domain.ErrValidation)
```

---

## 2. Error Propagation Rules

| Layer | Receives | Must do | Must NOT do |
|-------|----------|---------|-------------|
| Repository | raw DB errors (`sql.ErrNoRows`, driver) | wrap into domain errors | return `sql.ErrNoRows` or driver types |
| Usecase | domain errors from repo | add context with `%w` | create HTTP codes or call gin |
| Delivery | domain errors from usecase | map to HTTP status + JSON | contain business logic or DB awareness |

**Flow:** `sql.ErrNoRows` → repo wraps → `domain.ErrNotFound` → usecase wraps → delivery `errors.As` → 404 JSON.

`%w` preserves the error chain; `errors.As` unwraps through any depth.

---

## 3. Repository Error Wrapping

Translate every DB-specific error to a domain error before returning.

```go
// internal/repository/product_repository.go
package repository

import ("context"; "database/sql"; "errors"; "fmt"; "strings"
    "github.com/google/uuid"; "github.com/lib/pq"; "myapp/internal/domain")

type postgresProductRepo struct{ db *sql.DB }

func NewProductRepository(db *sql.DB) domain.ProductRepository { return &postgresProductRepo{db: db} }

func (r *postgresProductRepo) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    var p domain.Product
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name, description, price, stock, created_at, updated_at FROM products WHERE id = $1`, id,
    ).Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Stock, &p.CreatedAt, &p.UpdatedAt)
    if errors.Is(err, sql.ErrNoRows) { return nil, fmt.Errorf("product %s: %w", id, domain.ErrNotFound) }
    if err != nil { return nil, fmt.Errorf("query product: %w", err) }
    return &p, nil
}
func (r *postgresProductRepo) Create(ctx context.Context, p *domain.Product) error {
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO products (id, name, description, price, stock, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        p.ID, p.Name, p.Description, p.Price, p.Stock, p.CreatedAt, p.UpdatedAt)
    if err != nil { return wrapDBError("insert product", err) }
    return nil
}

func wrapDBError(op string, err error) error {
    var pqErr *pq.Error
    if errors.As(err, &pqErr) {
        switch pqErr.Code {
        case "23505": return fmt.Errorf("%s: %w", op, domain.ErrConflict)  // unique_violation
        case "23503": return fmt.Errorf("%s: %w", op, domain.ErrNotFound)  // foreign_key_violation
        }
    }
    if strings.Contains(err.Error(), "connection refused") || strings.Contains(err.Error(), "timeout") {
        return fmt.Errorf("%s: db unavailable: %w", op, err)
    }
    return fmt.Errorf("%s: %w", op, err)
}
```

---

## 4. Usecase Error Enrichment

Add business context with `fmt.Errorf("...: %w", err)`. Create domain errors for business rule violations.

```go
// internal/usecase/product_usecase.go
package usecase

import ("context"; "fmt"; "time"; "github.com/google/uuid"; "myapp/internal/domain")

type productUsecase struct{ repo domain.ProductRepository }

func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
    return &productUsecase{repo: repo}
}
func (u *productUsecase) GetProduct(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    p, err := u.repo.FindByID(ctx, id)
    if err != nil { return nil, fmt.Errorf("get product %s: %w", id, err) }
    return p, nil
}
func (u *productUsecase) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    if input.Price <= 0 {
        return nil, fmt.Errorf("price must be greater than zero: %w", domain.ErrValidation)
    }
    p := &domain.Product{ID: uuid.New(), Name: input.Name, Description: input.Description,
        Price: input.Price, Stock: input.Stock, CreatedAt: time.Now(), UpdatedAt: time.Now()}
    if err := u.repo.Create(ctx, p); err != nil { return nil, fmt.Errorf("create product: %w", err) }
    return p, nil
}
func (u *productUsecase) UpdateProduct(ctx context.Context, id uuid.UUID, input domain.UpdateProductInput) (*domain.Product, error) {
    p, err := u.repo.FindByID(ctx, id)
    if err != nil { return nil, fmt.Errorf("update product: %w", err) }
    if input.Name != nil { p.Name = *input.Name }
    if input.Price != nil {
        if *input.Price <= 0 { return nil, fmt.Errorf("price must be greater than zero: %w", domain.ErrValidation) }
        p.Price = *input.Price
    }
    if input.Stock != nil { p.Stock = *input.Stock }
    p.UpdatedAt = time.Now()
    if err := u.repo.Update(ctx, p); err != nil { return nil, fmt.Errorf("update product: %w", err) }
    return p, nil
}
// DeleteProduct and ListProducts follow the same pattern: wrap repo errors with fmt.Errorf("op: %w", err).
```

---

## 5. HTTP Error Mapping Middleware

Handlers call `c.Error(err)` and return. Middleware extracts `AppError` and writes JSON.

```go
// internal/delivery/http/middleware/error_middleware.go
package middleware

import ("errors"; "log/slog"; "net/http"
    "github.com/gin-gonic/gin"; "myapp/internal/domain")

func ErrorMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        if len(c.Errors) == 0 { return }
        err := c.Errors.Last().Err
        var appErr *domain.AppError
        if errors.As(err, &appErr) {
            if appErr.Code >= 500 {
                logger.ErrorContext(c.Request.Context(), "internal error", "error", err, "path", c.Request.URL.Path)
            } else {
                logger.WarnContext(c.Request.Context(), "client error", "status", appErr.Code, "error", err, "path", c.Request.URL.Path)
            }
            resp := gin.H{"error": appErr.Message}
            if appErr.Detail != "" { resp["detail"] = appErr.Detail }
            c.JSON(appErr.Code, resp)
            return
        }
        logger.ErrorContext(c.Request.Context(), "unhandled error", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
    }
}
```

**Thin handler — no mapping logic:**
```go
func (h *ProductHandler) GetByID(c *gin.Context) {
    id, err := uuid.Parse(c.Param("id"))
    if err != nil { _ = c.Error(fmt.Errorf("invalid product ID: %w", domain.ErrValidation)); return }
    product, err := h.uc.GetProduct(c.Request.Context(), id)
    if err != nil { _ = c.Error(err); return }
    c.JSON(http.StatusOK, product)
}
```

**Router setup:** `r.Use(middleware.RecoveryMiddleware(logger))` then `r.Use(middleware.ErrorMiddleware(logger))`.

---

## 6. Validation Errors

Two kinds: struct-tag binding (`ShouldBindJSON`) and business rule (usecase).

```go
type createProductRequest struct {
    Name        string `json:"name"        binding:"required,min=2,max=200"`
    Description string `json:"description" binding:"max=1000"`
    Price       int64  `json:"price"       binding:"required,gt=0"`
    Stock       int32  `json:"stock"       binding:"min=0"`
}

func (h *ProductHandler) Create(c *gin.Context) {
    var req createProductRequest
    if err := c.ShouldBindJSON(&req); err != nil { _ = c.Error(formatBindError(err)); return }
    product, err := h.uc.CreateProduct(c.Request.Context(), domain.CreateProductInput{
        Name: req.Name, Description: req.Description, Price: req.Price, Stock: req.Stock})
    if err != nil { _ = c.Error(err); return }
    c.JSON(http.StatusCreated, product)
}

func formatBindError(err error) *domain.AppError {
    var ve validator.ValidationErrors
    if errors.As(err, &ve) {
        fields := make([]string, len(ve))
        for i, fe := range ve { fields[i] = fe.Field() + ": " + fe.Tag() }
        return &domain.AppError{Code: 422, Message: "validation failed", Detail: strings.Join(fields, "; ")}
    }
    return &domain.AppError{Code: 422, Message: "validation failed"}
}
```

Response: `{"error":"validation failed","detail":"Name: min; Price: gt"}`

---

## 7. Logging Strategy

Log at the delivery layer only. Repo and usecase propagate — they never call logger.

- 2xx/3xx: no log (success)
- 4xx: `logger.WarnContext` (client mistake, not an outage)
- 5xx: `logger.ErrorContext` (needs operator attention)

Centralised in `ErrorMiddleware` (section 5). Use `slog` structured calls only — never `fmt.Println` or `log.Println`.

---

## 8. Panic Recovery

```go
// internal/delivery/http/middleware/recovery_middleware.go
package middleware

import ("log/slog"; "net/http"; "github.com/gin-gonic/gin")

func RecoveryMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return gin.RecoveryWithWriter(nil, func(c *gin.Context, recovered any) {
        logger.ErrorContext(c.Request.Context(), "panic recovered",
            "panic", recovered, "path", c.Request.URL.Path)
        c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
    })
}
```

Use `gin.New()` (not `gin.Default()`) — `gin.Default()` adds its own plaintext logger and recovery. Register `RecoveryMiddleware` before all other middleware.

---

## See Also

- **[repository-pattern.md](repository-pattern.md)** — DB error wrapping (SQLC, GORM)
- **[testing-by-layer.md](testing-by-layer.md)** — testing error paths, mock error injection
- **[layer-separation.md](layer-separation.md)** — why errors must not cross layer boundaries raw
- **gin-api** skill — `ShouldBind*` variants, header parsing, request lifecycle
