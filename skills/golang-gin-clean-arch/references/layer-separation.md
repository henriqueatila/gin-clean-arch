# Layer Separation — Layers in Detail

Deep-dive on the 4 clean architecture layers: dependency rule, each layer's responsibilities, and the request/response flow.

Companion: [layer-separation-antipatterns.md](layer-separation-antipatterns.md) — anti-patterns and migration guide.

## Dependency Rule Deep Dive

**Dependencies point inward only.** No inner layer may import an outer layer.

```
  ┌──────────────────────────────────────────────────┐
  │              Outer Layers (Adapters)              │
  │  ┌─────────────────┐  ┌───────────────────────┐  │
  │  │    Delivery      │  │     Repository        │  │
  │  │  (HTTP/Gin)      │  │  (Postgres/SQLC)      │  │
  │  └────────┬─────────┘  └──────────┬────────────┘  │
  │           │ imports                │ imports        │
  │  ┌────────▼────────────────────────▼────────────┐  │
  │  │           Usecase (business logic)           │  │
  │  │                  imports ↓                   │  │
  │  │  ┌────────────────────────────────────────┐  │  │
  │  │  │        Domain (entities, interfaces)   │  │  │
  │  │  └────────────────────────────────────────┘  │  │
  │  └──────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────┘

  X  Domain → any outer layer    FORBIDDEN
  X  Usecase → repository pkg    FORBIDDEN
  X  Usecase → gin / net/http    FORBIDDEN
  X  Repository → delivery       FORBIDDEN
```

| Layer | Package | Allowed imports | Forbidden |
|-------|---------|-----------------|-----------|
| Domain | `internal/domain` | stdlib only | all internal pkgs |
| Usecase | `internal/usecase` | `domain` | `repository`, `delivery`, `gin` |
| Repository | `internal/repository` | `domain`, `database/sql` | `delivery`, `gin` |
| Delivery | `internal/delivery/http` | `domain` | `repository`, `usecase` (concrete) |

**How:** Domain defines `ProductRepository` and `ProductUsecase` interfaces. Outer layers implement them. Inner layers depend on the interface — never the concrete struct.

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
    if p.Name == ""  { return fmt.Errorf("product name is required: %w", ErrValidation) }
    if p.Price <= 0  { return fmt.Errorf("product price must be positive: %w", ErrValidation) }
    if p.Stock < 0   { return fmt.Errorf("product stock cannot be negative: %w", ErrValidation) }
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
        return nil, fmt.Errorf("create product: %v: %w", err, domain.ErrValidation)
    }
    if err := u.repo.Create(ctx, p); err != nil {
        return nil, fmt.Errorf("create product: %w", err)
    }
    return p, nil
}
```

**Multi-repo usecases** inject multiple interfaces. Each dependency comes from `internal/domain` — the concrete packages are never referenced:

```go
// domain.WarehouseRepository — hypothetical interface for this example:
//   type WarehouseRepository interface {
//       RecordTransfer(ctx context.Context, from, to uuid.UUID, qty int32) error
//   }
type transferStockUsecase struct {
    productRepo   domain.ProductRepository
    warehouseRepo domain.WarehouseRepository
}

// Production: define TransferStockUsecase interface in domain and return it here.
// Returning concrete type below violates Golden Rule 6 — shown for brevity only.
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

> **Production note:** This simplified example omits transactional coordination. In production, wrap multi-repo writes in a transaction using the `execTx` pattern from [repository-pattern.md](repository-pattern.md#4-transactions) to prevent partial updates.

---

## Repository Layer in Detail

Implements domain interfaces with a specific database technology. The rest of the application never knows which database is used.

**One repository per aggregate root.** `Product` owns its own `ProductRepository`. Never mix two aggregates into one repository. Standard methods: `FindByID`, `FindAll`, `Create`, `Update`, `Delete`.

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

type postgresProductRepo struct{ db *sql.DB }

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

**Mapping tax:** `createProductRequest` → `CreateProductInput` → `Product` → `productResponse`. Justified when business rules exist between layers. For pure CRUD you may map request DTO directly to domain entity in the handler, but never skip the response DTO and never add `json`/`binding` tags to domain structs.

```go
// internal/delivery/http/product_dto.go
package http

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
    UpdatedAt   string `json:"updated_at"`
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
        ID: p.ID.String(), Name: p.Name, Description: p.Description,
        PriceCents: p.Price, Stock: p.Stock,
        CreatedAt: p.CreatedAt.Format(time.RFC3339),
        UpdatedAt: p.UpdatedAt.Format(time.RFC3339),
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

This skill uses **package-by-layer** (`internal/domain`, `internal/usecase`, `internal/repository`, `internal/delivery/http`). For larger codebases (3+ aggregates), consider **package-by-component** — group all code for one aggregate in a single package, using unexported types for encapsulation.

**Start with package-by-layer.** Migrate to package-by-component when developers bypass the dependency rule.

---

## See Also

- [layer-separation-antipatterns.md](layer-separation-antipatterns.md) — 6 anti-patterns with corrections, migration checklist
- [dependency-injection.md](dependency-injection.md) — Wiring layers together in `main.go`
- [repository-pattern.md](repository-pattern.md) — SQLC and GORM implementations
- [error-handling.md](error-handling.md) — Propagating domain errors across layers
