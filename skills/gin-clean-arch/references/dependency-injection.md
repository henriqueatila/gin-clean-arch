# Dependency Injection — Manual DI and Constructor Patterns

Manual constructor wiring, the DI container pattern, constructor variants, and testing with DI.

Companion: [dependency-injection-alternatives.md](dependency-injection-alternatives.md) — scaling, Wire, Fx, common mistakes.

## Table of Contents

1. [Manual DI — The Recommended Approach](#manual-di--the-recommended-approach)
2. [DI Container Pattern](#di-container-pattern)
3. [Constructor Patterns](#constructor-patterns)
4. [Testing with DI](#testing-with-di)

---

## Manual DI — The Recommended Approach

Wire all dependencies by hand in `main.go`. No code generation, no reflection. Every dependency is explicit and visible.

| Property | Manual DI | Wire/Fx |
|----------|-----------|---------|
| Readable at a glance | Yes | Requires learning tool |
| Compile-time safety | Yes | Yes (Wire) / No (Fx) |
| Debuggable step-by-step | Yes | Harder |
| Suitable for | All sizes | Large codebases |

### Constructor Rule — Return Interface, Not Struct

```go
// CORRECT — callers receive the domain interface; can be swapped for a mock
func NewProductRepository(db *sql.DB) domain.ProductRepository {
    return &postgresProductRepository{db: db}
}
func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
    return &productUsecase{repo: repo}
}

// WRONG — concrete return type leaks the implementation; mocking breaks
func NewProductRepository(db *sql.DB) *postgresProductRepository { ... }
```

### Bootstrap Order

```
config → database → repository → usecase → handler → router → server
```

Each step depends only on the step above. Concrete types appear **only** in `main.go`.

```go
// cmd/api/main.go — wiring section (full server in project-scaffolding.md)
package main

import (
    "context"
    "database/sql"
    "log/slog"
    "os"

    _ "github.com/lib/pq"
    "myapp/config"
    delivery "myapp/internal/delivery/http"
    "myapp/internal/repository"
    "myapp/internal/usecase"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    cfg, err := config.Load()
    if err != nil {
        logger.Error("failed to load config", "error", err)
        os.Exit(1)
    }
    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil {
        logger.Error("failed to open database", "error", err)
        os.Exit(1)
    }
    defer func() {
        if err := db.Close(); err != nil {
            logger.Error("close db", "error", err)
        }
    }()
    if err := db.PingContext(context.Background()); err != nil {
        logger.Error("ping database", "error", err)
        os.Exit(1)
    }

    // Concrete types appear here and nowhere else.
    productRepo    := repository.NewProductRepository(db)    // → domain.ProductRepository
    productUC      := usecase.NewProductUsecase(productRepo) // → domain.ProductUsecase
    productHandler := delivery.NewProductHandler(productUC, logger)

    // ... gin setup + graceful shutdown — see project-scaffolding.md
}
```

---

## DI Container Pattern

When the app exceeds ~5 handlers, group them into a `Dependencies` struct rather than listing individual variables.

```go
// internal/delivery/http/dependencies.go
package http

// Dependencies groups all wired handlers. Add one field per aggregate.
type Dependencies struct {
    Product *ProductHandler
    // Order *OrderHandler
}

// internal/delivery/http/router.go
func RegisterRoutes(r *gin.Engine, deps Dependencies) {
    api := r.Group("/api/v1")
    RegisterProductRoutes(api, deps.Product)
    // RegisterOrderRoutes(api, deps.Order)
}

// cmd/api/main.go — wiring with container
deps := delivery.Dependencies{
    Product: delivery.NewProductHandler(productUC, logger),
}
delivery.RegisterRoutes(r, deps)
```

Switch when: more than 5 handlers, or the wiring block exceeds ~30 lines.

---

## Constructor Patterns

All variants return interfaces. Choose the simplest that fits the need.

```go
// 1. Basic — use by default
func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
    return &productUsecase{repo: repo}
}

// 2. With config — use when behaviour varies by environment
type ProductUsecaseConfig struct {
    DefaultPageSize int   // 0 → 20
    MaxPageSize     int   // 0 → 100
}

func NewProductUsecaseWithConfig(repo domain.ProductRepository, cfg ProductUsecaseConfig) domain.ProductUsecase {
    if cfg.DefaultPageSize <= 0 {
        cfg.DefaultPageSize = 20
    }
    if cfg.MaxPageSize <= 0 {
        cfg.MaxPageSize = 100
    }
    return &productUsecase{repo: repo, cfg: cfg}
}

// 3. With validation — use when nil args must be caught at startup
func NewProductRepository(db *sql.DB) (domain.ProductRepository, error) {
    if db == nil {
        return nil, errors.New("product repository: db must not be nil")
    }
    return &postgresProductRepository{db: db}, nil
}

// 4. Lazy prepared statements — use for hot-path query caching only
type postgresProductRepository struct {
    db   *sql.DB
    once sync.Once
    stmt *sql.Stmt
}

func (r *postgresProductRepository) prepareStatements(ctx context.Context) error {
    var prepErr error
    r.once.Do(func() {
        r.stmt, prepErr = r.db.PrepareContext(ctx,
            `SELECT id, name, description, price, stock, created_at, updated_at
             FROM products WHERE id = $1`)
    })
    return prepErr
}
```

---

## Testing with DI

Swap real dependencies with mocks at the constructor call — no production code changes required.

### Usecase Test — Mock Repository

```go
// internal/usecase/product_usecase_test.go
package usecase_test

import (
    "context"
    "errors"
    "testing"

    "github.com/google/uuid"
    "myapp/internal/domain"
    "myapp/internal/usecase"
)

type mockProductRepository struct {
    createFunc func(context.Context, *domain.Product) error
}

func (m *mockProductRepository) Create(ctx context.Context, p *domain.Product) error              { return m.createFunc(ctx, p) }
func (m *mockProductRepository) FindByID(_ context.Context, _ uuid.UUID) (*domain.Product, error) { return nil, nil }
func (m *mockProductRepository) FindAll(_ context.Context, _ domain.ProductFilter) ([]domain.Product, error) { return nil, nil }
func (m *mockProductRepository) Update(_ context.Context, _ *domain.Product) error { return nil }
func (m *mockProductRepository) Delete(_ context.Context, _ uuid.UUID) error       { return nil }

func TestCreateProduct_Success(t *testing.T) {
    repo := &mockProductRepository{createFunc: func(_ context.Context, _ *domain.Product) error { return nil }}
    got, err := usecase.NewProductUsecase(repo).CreateProduct(context.Background(),
        domain.CreateProductInput{Name: "Widget", Price: 999, Stock: 10})
    if err != nil { t.Fatalf("unexpected error: %v", err) }
    if got.Name != "Widget" { t.Errorf("expected Widget, got %s", got.Name) }
}

func TestCreateProduct_InvalidPrice(t *testing.T) {
    _, err := usecase.NewProductUsecase(&mockProductRepository{}).CreateProduct(
        context.Background(), domain.CreateProductInput{Name: "Widget", Price: 0})
    if !errors.Is(err, domain.ErrValidation) {
        t.Errorf("expected ErrValidation, got %v", err)
    }
}
```

### Handler Test — Mock Usecase

```go
// internal/delivery/http/product_handler_test.go
package http_test

import (
    "bytes"
    "context"
    "encoding/json"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    delivery "myapp/internal/delivery/http"
    "myapp/internal/domain"
)

type mockProductUsecase struct {
    createProductFunc func(context.Context, domain.CreateProductInput) (*domain.Product, error)
}
func (m *mockProductUsecase) CreateProduct(ctx context.Context, in domain.CreateProductInput) (*domain.Product, error) { return m.createProductFunc(ctx, in) }
func (m *mockProductUsecase) GetProduct(_ context.Context, _ uuid.UUID) (*domain.Product, error)                    { return nil, nil }
func (m *mockProductUsecase) ListProducts(_ context.Context, _ domain.ProductFilter) ([]domain.Product, error)      { return nil, nil }
func (m *mockProductUsecase) UpdateProduct(_ context.Context, _ uuid.UUID, _ domain.UpdateProductInput) (*domain.Product, error) { return nil, nil }
func (m *mockProductUsecase) DeleteProduct(_ context.Context, _ uuid.UUID) error                                    { return nil }

func TestProductHandler_Create_Returns201(t *testing.T) {
    gin.SetMode(gin.TestMode)
    uc := &mockProductUsecase{
        createProductFunc: func(_ context.Context, in domain.CreateProductInput) (*domain.Product, error) {
            return &domain.Product{ID: uuid.New(), Name: in.Name, Price: in.Price}, nil
        },
    }
    r := gin.New()
    r.POST("/products", delivery.NewProductHandler(uc, slog.Default()).Create)

    body, err := json.Marshal(map[string]any{"name": "Widget", "price": int64(999)})
    if err != nil {
        t.Fatalf("marshal: %v", err)
    }
    req := httptest.NewRequest(http.MethodPost, "/products", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()
    r.ServeHTTP(rec, req)

    if rec.Code != http.StatusCreated {
        t.Errorf("expected 201, got %d: %s", rec.Code, rec.Body.String())
    }
}
```

Each layer tests in isolation — no running database or HTTP server needed for usecase tests.

---

## See Also

- [dependency-injection-alternatives.md](dependency-injection-alternatives.md) — Scaling, Wire, Fx, common mistakes
- [layer-separation.md](layer-separation.md) — Why interfaces live in domain, not outer layers
- [testing-by-layer.md](testing-by-layer.md) — Full mock-per-layer testing patterns
- [project-scaffolding.md](project-scaffolding.md) — Complete `main.go` with graceful shutdown
