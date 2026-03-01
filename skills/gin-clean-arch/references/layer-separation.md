# Layer Separation — Layers in Detail

Deep-dive on the 4 clean architecture layers: dependency rule, each layer's responsibilities, and the request/response flow.

Companion: [layer-separation-antipatterns.md](layer-separation-antipatterns.md) — anti-patterns and migration guide.

## Table of Contents

1. [Dependency Rule Deep Dive](#dependency-rule-deep-dive)
2. [Entity Layer in Detail](#entity-layer-in-detail)
3. [Usecase Layer in Detail](#usecase-layer-in-detail)
4. [Repository Layer in Detail](#repository-layer-in-detail)
5. [Delivery Layer in Detail](#delivery-layer-in-detail)

---

## Dependency Rule Deep Dive

**Dependencies point inward only.** No inner layer may import an outer layer.

```
  ┌────────────────────────────────────────────┐
  │  Delivery (internal/delivery/http)         │  ← outermost
  │         imports ↓                          │
  │  ┌──────────────────────────────────────┐  │
  │  │  Repository (internal/repository)   │  │
  │  │         imports ↓                   │  │
  │  │  ┌────────────────────────────────┐  │  │
  │  │  │  Usecase (internal/usecase)   │  │  │
  │  │  │         imports ↓             │  │  │
  │  │  │  ┌────────────────────────┐   │  │  │
  │  │  │  │  Domain (innermost)   │   │  │  │
  │  │  │  └────────────────────────┘   │  │  │
  │  │  └────────────────────────────────┘  │  │
  │  └──────────────────────────────────────┘  │
  └────────────────────────────────────────────┘

  X  Domain → any outer layer    FORBIDDEN
  X  Usecase → repository pkg    FORBIDDEN
  X  Usecase → gin / net/http    FORBIDDEN
  X  Repository → delivery       FORBIDDEN
```

**How interfaces enforce this:** Domain defines `ProductRepository` and `ProductUsecase`. Outer layers implement them. Inner layers hold only the interface — never the concrete struct. The dependency arrow points *toward* domain in every case.

| Layer | Package | Allowed imports | Forbidden |
|-------|---------|-----------------|-----------|
| Domain | `internal/domain` | stdlib only | all internal pkgs |
| Usecase | `internal/usecase` | `domain` | `repository`, `delivery`, `gin` |
| Repository | `internal/repository` | `domain`, `database/sql` | `delivery`, `gin` |
| Delivery | `internal/delivery/http` | `domain` | `repository`, `usecase` (concrete) |

---

## Entity Layer in Detail

Pure Go types. No framework imports. No JSON tags. No database annotations.

```go
// internal/domain/product.go
package domain

import ("errors"; "fmt"; "time"; "github.com/google/uuid")

type Product struct {
    ID          uuid.UUID
    Name        string
    Description string
    Price       int64 // cents — never float
    Stock       int32
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

func (p *Product) Validate() error {
    if p.Name == ""  { return errors.New("product name is required") }
    if p.Price <= 0  { return errors.New("product price must be positive") }
    if p.Stock < 0   { return errors.New("product stock cannot be negative") }
    return nil
}

func (p *Product) DeductStock(qty int32) error {
    if qty <= 0     { return errors.New("quantity must be positive") }
    if p.Stock < qty { return fmt.Errorf("insufficient stock: have %d, need %d: %w", p.Stock, qty, ErrValidation) }
    p.Stock -= qty
    return nil
}
```

**Value objects** wrap primitives to enforce domain invariants (e.g. `Money{amount int64, currency string}`). Immutable — operations return new values. See [error-handling.md](error-handling.md) for `AppError` and sentinel errors.

**Interface placement:** `ProductRepository` and `ProductUsecase` are defined in `internal/domain` — *used* by inner layers, *implemented* by outer layers. Full interface signatures are in SKILL.md. This is the mechanism that inverts the dependency: the repository package imports domain, never the reverse.

---

## Usecase Layer in Detail

Orchestrates domain operations. Depends only on domain interfaces — never on `gin`, `sql`, or `gorm`.

```go
// internal/usecase/product_usecase.go
package usecase

import ("context"; "fmt"; "time"; "github.com/google/uuid"; "myapp/internal/domain")

type productUsecase struct{ repo domain.ProductRepository }

// Constructor returns the interface — not the concrete struct.
func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
    return &productUsecase{repo: repo}
}

func (u *productUsecase) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    p := &domain.Product{
        ID: uuid.New(), Name: input.Name, Description: input.Description,
        Price: input.Price, Stock: input.Stock,
        CreatedAt: time.Now().UTC(), UpdatedAt: time.Now().UTC(),
    }
    if err := p.Validate(); err != nil {
        return nil, fmt.Errorf("create product: %w", domain.ErrValidation)
    }
    if err := u.repo.Create(ctx, p); err != nil {
        return nil, fmt.Errorf("create product: %w", err)
    }
    return p, nil
}
```

**Multi-repo usecases** inject multiple interfaces. Each dependency comes from `internal/domain` — the concrete packages are never referenced:

```go
type transferStockUsecase struct {
    productRepo   domain.ProductRepository
    warehouseRepo domain.WarehouseRepository
}

func NewTransferStockUsecase(p domain.ProductRepository, w domain.WarehouseRepository) *transferStockUsecase {
    return &transferStockUsecase{productRepo: p, warehouseRepo: w}
}

func (u *transferStockUsecase) Execute(ctx context.Context, src, dst uuid.UUID, qty int32) error {
    s, err := u.productRepo.FindByID(ctx, src)
    if err != nil { return fmt.Errorf("transfer: %w", err) }
    d, err := u.productRepo.FindByID(ctx, dst)
    if err != nil { return fmt.Errorf("transfer: %w", err) }
    if err := s.DeductStock(qty); err != nil { return fmt.Errorf("transfer: %w", err) }
    d.Stock += qty
    if err := u.productRepo.Update(ctx, s); err != nil { return fmt.Errorf("transfer: update src: %w", err) }
    if err := u.productRepo.Update(ctx, d); err != nil { return fmt.Errorf("transfer: update dst: %w", err) }
    return u.warehouseRepo.RecordTransfer(ctx, src, dst, qty)
}
```

---

## Repository Layer in Detail

Implements domain interfaces with a specific database technology. The rest of the application never knows which database is used.

**One repository per aggregate root.** `Product` owns its own `ProductRepository`. Never mix two aggregates into one repository.

| Method | SQL |
|--------|-----|
| `FindByID` | `SELECT ... WHERE id = $1` |
| `FindAll` | `SELECT ... LIMIT $1 OFFSET $2` |
| `Create` | `INSERT INTO ...` |
| `Update` | `UPDATE ... SET ...` |
| `Delete` | `DELETE FROM ...` or soft-delete |

```go
// internal/repository/postgres_product_repository.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "github.com/google/uuid"
    "myapp/internal/domain"
)

type postgresProductRepository struct{ db *sql.DB }

func NewProductRepository(db *sql.DB) domain.ProductRepository {
    return &postgresProductRepository{db: db}
}

func (r *postgresProductRepository) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    var p domain.Product
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name, description, price, stock, created_at, updated_at
         FROM products WHERE id = $1`, id,
    ).Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Stock, &p.CreatedAt, &p.UpdatedAt)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("product %s: %w", id, domain.ErrNotFound)
    }
    if err != nil {
        return nil, fmt.Errorf("find product: %w", err)
    }
    return &p, nil
}
```

**Rule:** Map `sql.ErrNoRows` → `domain.ErrNotFound` at the repository boundary. Raw database errors must never escape to the usecase.

---

## Delivery Layer in Detail

Handlers are thin adapters: bind → call usecase → respond. Zero business logic.

**DTOs** decouple HTTP wire format from domain model. Domain structs never carry JSON tags or `binding:` tags.

**Pragmatic note on model duplication:** Strict separation creates a "mapping tax" — `createProductRequest` → `CreateProductInput` → `Product` → `productModel` → `productResponse`. This cost is justified when business rules exist between layers. For pure CRUD endpoints with no business logic, you may skip the usecase input type and map directly from request DTO to domain entity in the handler — but never skip the response DTO (it controls what the client sees) and never add `json`/`binding` tags to domain structs.

```go
// internal/delivery/http/product_dto.go
package http

import "github.com/google/uuid"

type createProductRequest struct {
    Name        string `json:"name"        binding:"required,min=2,max=200"`
    Description string `json:"description" binding:"max=1000"`
    Price       int64  `json:"price"       binding:"required,gt=0"`
    Stock       int32  `json:"stock"       binding:"min=0"`
}

type productResponse struct {
    ID         uuid.UUID `json:"id"`
    Name       string    `json:"name"`
    PriceCents int64     `json:"price_cents"`
    Stock      int32     `json:"stock"`
    CreatedAt  string    `json:"created_at"`
}
```

```go
// internal/delivery/http/product_handler.go
package http

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
)

type ProductHandler struct {
    uc     domain.ProductUsecase
    logger *slog.Logger
}

func NewProductHandler(uc domain.ProductUsecase, logger *slog.Logger) *ProductHandler {
    return &ProductHandler{uc: uc, logger: logger}
}

func toProductResponse(p *domain.Product) productResponse {
    return productResponse{
        ID: p.ID, Name: p.Name, PriceCents: p.Price,
        Stock: p.Stock, CreatedAt: p.CreatedAt.Format("2006-01-02T15:04:05Z"),
    }
}

// Create: ShouldBindJSON → domain input → usecase → response DTO.
func (h *ProductHandler) Create(c *gin.Context) {
    var req createProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "validation failed"})
        return
    }
    product, err := h.uc.CreateProduct(c.Request.Context(), domain.CreateProductInput{
        Name: req.Name, Description: req.Description,
        Price: req.Price, Stock: req.Stock,
    })
    if err != nil {
        handleError(c, err, h.logger)
        return
    }
    c.JSON(http.StatusCreated, toProductResponse(product))
}
```

**Flow:** `ShouldBindJSON` → map to domain input → usecase → map to response DTO → `c.JSON`.

---

## Package by Layer vs Package by Component

This skill defaults to **package-by-layer** (`internal/domain`, `internal/usecase`, `internal/repository`, `internal/delivery/http`). This is the standard Go clean architecture layout and works well for small-to-medium projects.

For larger codebases with multiple bounded contexts, consider **package-by-component** (Simon Brown's "The Missing Chapter"):

```
internal/
├── product/           # All product code — private implementations
│   ├── entity.go      # Product struct (unexported details)
│   ├── usecase.go     # productUsecase (unexported)
│   ├── repository.go  # postgresProductRepo (unexported)
│   ├── handler.go     # ProductHandler (exported — entry point)
│   └── product.go     # Exported interfaces + constructor
├── order/             # Same pattern for Order
└── shared/            # Shared domain types, errors
```

**Advantages:** Stronger encapsulation — unexported types within a package prevent callers from bypassing architecture layers. A handler cannot call a repository directly because the struct is unexported outside the package.

**Trade-off:** Harder to navigate when you want to see "all repositories" at a glance. Go's package system enforces visibility boundaries, so the compiler blocks violations automatically.

**Recommendation:** Start with package-by-layer (this skill's default). Migrate to package-by-component when you have 3+ aggregate roots and find developers bypassing the dependency rule.

---

## See Also

- [layer-separation-antipatterns.md](layer-separation-antipatterns.md) — 6 anti-patterns with corrections, migration checklist
- [dependency-injection.md](dependency-injection.md) — Wiring layers together in `main.go`
- [repository-pattern.md](repository-pattern.md) — SQLC and GORM implementations
- [error-handling.md](error-handling.md) — Propagating domain errors across layers
