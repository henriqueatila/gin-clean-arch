# Layer Separation — Anti-Patterns and Migration Guide

Anti-patterns to avoid and a step-by-step guide for refactoring a flat Gin project into clean architecture layers.

Companion file: [layer-separation.md](layer-separation.md) — dependency rule, each layer in detail.

---

## Anti-Patterns

### 1. Business Logic in Handlers

```go
// BAD — price floor is a business rule; it belongs in the usecase
// Also BAD: err.Error() leaks internal struct names to the client
func (h *ProductHandler) Create(c *gin.Context) {
    var req createProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()}) // never expose raw errors
        return
    }
    if req.Price < 100 { // business rule leaking into delivery layer
        c.JSON(http.StatusBadRequest, gin.H{"error": "minimum price is $1.00"})
        return
    }
    // ...
}

// GOOD — handler delegates unconditionally; usecase enforces the rule
func (u *productUsecase) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    if input.Price < 100 {
        return nil, fmt.Errorf("minimum price is 100 cents: %w", domain.ErrValidation)
    }
    // ...
}
```

### 2. GORM Model Embedded in Domain Entity

```go
// BAD — domain entity imports gorm; framework concern leaks inward
import "gorm.io/gorm"

type Product struct {
    gorm.Model        // embeds ID, CreatedAt, DeletedAt — ties core to ORM
    Name  string
    Price int64
}

// GOOD — pure domain struct; GORM mapping happens only in the repository
type Product struct {
    ID        uuid.UUID
    Name      string
    Price     int64
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### 3. Repository Returning HTTP Errors

```go
// BAD — repository knows about HTTP status codes
func (r *postgresProductRepository) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    // ...
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("404 not found") // HTTP concept in data layer
    }
}

// GOOD — repository returns domain errors; delivery layer maps to HTTP
func (r *postgresProductRepository) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    // ...
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("product %s: %w", id, domain.ErrNotFound)
    }
}
```

### 4. Usecase Importing Gin

```go
// BAD — usecase is coupled to the HTTP framework; untestable without HTTP
import "github.com/gin-gonic/gin"

func (u *productUsecase) CreateProduct(c *gin.Context) {
    name := c.PostForm("name") // reads directly from gin context
    // ...
}

// GOOD — usecase accepts plain Go types; context only for cancellation/tracing
func (u *productUsecase) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    // testable without a running HTTP server
}
```

### 5. Usecase Importing Repository Package Directly

```go
// BAD — usecase imports the outer repository package; violates dependency rule
import "myapp/internal/repository"

func NewProductUsecase(repo *repository.PostgresProductRepository) *productUsecase {
    return &productUsecase{repo: repo}
}

// GOOD — usecase depends only on the domain interface
import "myapp/internal/domain"

func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
    return &productUsecase{repo: repo}
}
```

### 6. God Usecase

```go
// BAD — one struct handles unrelated aggregates; grows without bound
type GodUsecase struct {
    productRepo domain.ProductRepository
    orderRepo   domain.OrderRepository
    userRepo    domain.UserRepository
    paymentRepo domain.PaymentRepository
}

func (u *GodUsecase) CreateProduct(...) {}
func (u *GodUsecase) PlaceOrder(...) {}
func (u *GodUsecase) ProcessPayment(...) {}

// GOOD — one usecase per aggregate or bounded business concern
type productUsecase struct{ repo domain.ProductRepository }
type orderUsecase struct {
    productRepo domain.ProductRepository
    orderRepo   domain.OrderRepository
}
type paymentUsecase struct{ paymentRepo domain.PaymentRepository }
```

---

## Migration Guide

Refactor a flat Gin project into clean architecture layers incrementally — one step per deploy cycle.

### Step 1 — Create the directory structure

```bash
mkdir -p internal/domain
mkdir -p internal/usecase
mkdir -p internal/repository
mkdir -p internal/delivery/http
```

### Step 2 — Extract domain types

Move all structs that represent business concepts to `internal/domain`. Strip every JSON tag, GORM embedding, and HTTP status code from them. Domain structs are pure Go — stdlib imports only.

### Step 3 — Define interfaces in domain

For every database operation currently in the codebase, write a method signature in `ProductRepository`. For every business operation a handler currently calls directly, write a method in `ProductUsecase`. Place both interfaces in `internal/domain`.

### Step 4 — Move database code to repository

Create `internal/repository/postgres_product_repository.go`. Implement `domain.ProductRepository`. Wrap `sql.ErrNoRows` → `domain.ErrNotFound` here. Return `domain.ProductRepository` from the constructor, not the concrete struct.

### Step 5 — Extract business logic to usecase

Identify any logic in handlers beyond "bind, call, respond" — price rules, stock checks, computed fields, conditional branching on domain state. Move it to `internal/usecase/product_usecase.go`. Constructor returns `domain.ProductUsecase`.

### Step 6 — Slim down handlers

Each handler should reduce to: `ShouldBindJSON` → map to domain input → call usecase → map to response DTO → `c.JSON`. A handler over ~40 lines almost certainly contains extractable business logic.

### Migration checklist

- [ ] Domain structs have no `gorm.Model`, no JSON tags, no HTTP package imports
- [ ] Interfaces defined in `internal/domain`, implemented in outer layers
- [ ] Repository constructor returns `domain.ProductRepository` (interface)
- [ ] Usecase constructor returns `domain.ProductUsecase` (interface)
- [ ] Usecase has zero imports from `gin`, `net/http`, `database/sql`
- [ ] Handlers contain zero business rules — all logic delegated to usecase
- [ ] All dependencies wired in `cmd/api/main.go` — no `init()`, no globals
- [ ] `go build ./...` passes with no errors

---

## See Also

- [layer-separation.md](layer-separation.md) — Dependency rule, each layer in detail
- [dependency-injection.md](dependency-injection.md) — Wiring layers together in `main.go`
- [error-handling.md](error-handling.md) — Propagating domain errors across layers
- [testing-by-layer.md](testing-by-layer.md) — Testing each layer in isolation
