# Repository Pattern — Deep Dive

Quick overview: [SKILL.md](../SKILL.md).

## Table of Contents

1. [Interface Design](#1-interface-design)
2. [SQLC Implementation](#2-sqlc-implementation)
3. [GORM Implementation](#3-gorm-implementation)
4. [Transactions](#4-transactions)
5. [Connection Pooling](#5-connection-pooling)
6. [Query Patterns](#6-query-patterns)
7. [Testing](#7-testing)

---

## 1. Interface Design

Defined in `internal/domain/`. One interface per aggregate root. Domain-verb methods only. No DB types in signatures.

```go
// internal/domain/product.go
package domain

import ("context"; "time"; "github.com/google/uuid")

type Product struct {
    ID          uuid.UUID
    Name, Description string
    Price       int64 // cents
    Stock       int32
    CreatedAt, UpdatedAt time.Time
}

type ProductFilter struct {
    Name           string
    MinPrice, MaxPrice int64
    Page, Limit    int
    SortBy, SortDir string
}

type ProductRepository interface {
    FindByID(ctx context.Context, id uuid.UUID) (*Product, error)
    FindAll(ctx context.Context, filter ProductFilter) ([]Product, error)
    Create(ctx context.Context, p *Product) error
    Update(ctx context.Context, p *Product) error
    Delete(ctx context.Context, id uuid.UUID) error
}
```

---

## 2. SQLC Implementation

Workflow: write SQL → `sqlc generate` → implement interface with generated `*sqlcgen.Queries`. Each query needs a `-- name: QueryName :one|:many|:exec` annotation.

**sqlc.yaml:**
```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "internal/repository/sqlc/queries/"
    schema:  "migrations/"
    gen:
      go: { package: "sqlcgen", out: "internal/repository/sqlc/sqlcgen", emit_json_tags: true }
```

**Repository:**
```go
// internal/repository/product_repository_sqlc.go
package repository

import ("context"; "database/sql"; "errors"; "fmt"
    "github.com/google/uuid"; "myapp/internal/domain"; "myapp/internal/repository/sqlc/sqlcgen")

type sqlcProductRepo struct{ db *sql.DB; q *sqlcgen.Queries }

func NewSQLCProductRepository(db *sql.DB) domain.ProductRepository {
    return &sqlcProductRepo{db: db, q: sqlcgen.New(db)}
}
func toDomain(r sqlcgen.Product) domain.Product {
    return domain.Product{ID: r.ID, Name: r.Name, Description: r.Description,
        Price: r.Price, Stock: r.Stock, CreatedAt: r.CreatedAt, UpdatedAt: r.UpdatedAt}
}
func (r *sqlcProductRepo) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    row, err := r.q.GetProduct(ctx, id)
    if errors.Is(err, sql.ErrNoRows) { return nil, fmt.Errorf("product %s: %w", id, domain.ErrNotFound) }
    if err != nil { return nil, fmt.Errorf("get product: %w", err) }
    p := toDomain(row); return &p, nil
}
func (r *sqlcProductRepo) FindAll(ctx context.Context, f domain.ProductFilter) ([]domain.Product, error) {
    if f.Limit <= 0 { f.Limit = 20 }
    rows, err := r.q.ListProducts(ctx, sqlcgen.ListProductsParams{
        Limit: int32(f.Limit), Offset: int32((f.Page - 1) * f.Limit)})
    if err != nil { return nil, fmt.Errorf("list products: %w", err) }
    out := make([]domain.Product, len(rows))
    for i, row := range rows { out[i] = toDomain(row) }
    return out, nil
}
func (r *sqlcProductRepo) Create(ctx context.Context, p *domain.Product) error {
    return r.q.CreateProduct(ctx, sqlcgen.CreateProductParams{ID: p.ID, Name: p.Name,
        Description: p.Description, Price: p.Price, Stock: p.Stock, CreatedAt: p.CreatedAt, UpdatedAt: p.UpdatedAt})
}
func (r *sqlcProductRepo) Update(ctx context.Context, p *domain.Product) error {
    return r.q.UpdateProduct(ctx, sqlcgen.UpdateProductParams{ID: p.ID, Name: p.Name,
        Description: p.Description, Price: p.Price, Stock: p.Stock, UpdatedAt: p.UpdatedAt})
}
func (r *sqlcProductRepo) Delete(ctx context.Context, id uuid.UUID) error { return r.q.DeleteProduct(ctx, id) }
```

---

## 3. GORM Implementation

Separate model with GORM tags; map to/from domain. Never annotate the domain struct.

```go
// internal/repository/product_repository_gorm.go
package repository

import ("context"; "errors"; "fmt"; "time"
    "github.com/google/uuid"; "github.com/lib/pq"; "gorm.io/gorm"; "myapp/internal/domain")

type productModel struct {
    ID uuid.UUID `gorm:"type:uuid;primaryKey"`; Name string `gorm:"not null"`
    Description string; Price int64 `gorm:"not null"`; Stock int32 `gorm:"not null;default:0"`
    CreatedAt, UpdatedAt time.Time
}
func (productModel) TableName() string { return "products" }
type gormProductRepo struct{ db *gorm.DB }
func NewGORMProductRepository(db *gorm.DB) domain.ProductRepository { return &gormProductRepo{db} }
func toModel(p *domain.Product) productModel {
    return productModel{ID: p.ID, Name: p.Name, Description: p.Description,
        Price: p.Price, Stock: p.Stock, CreatedAt: p.CreatedAt, UpdatedAt: p.UpdatedAt}
}
func fromModel(m productModel) domain.Product {
    return domain.Product{ID: m.ID, Name: m.Name, Description: m.Description,
        Price: m.Price, Stock: m.Stock, CreatedAt: m.CreatedAt, UpdatedAt: m.UpdatedAt}
}
func (r *gormProductRepo) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    var m productModel
    if err := r.db.WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) { return nil, fmt.Errorf("product %s: %w", id, domain.ErrNotFound) }
        return nil, fmt.Errorf("get product: %w", err)
    }
    p := fromModel(m); return &p, nil
}
func (r *gormProductRepo) Create(ctx context.Context, p *domain.Product) error {
    m := toModel(p)
    if err := r.db.WithContext(ctx).Create(&m).Error; err != nil {
        var pqErr *pq.Error
        if errors.As(err, &pqErr) && pqErr.Code == "23505" {
            return fmt.Errorf("product %q: %w", p.Name, domain.ErrConflict)
        }
        return fmt.Errorf("insert product: %w", err)
    }
    return nil
}
func (r *gormProductRepo) Update(ctx context.Context, p *domain.Product) error {
    m := toModel(p)
    result := r.db.WithContext(ctx).Save(&m)
    if result.Error != nil { return fmt.Errorf("update product: %w", result.Error) }
    if result.RowsAffected == 0 { return fmt.Errorf("product %s: %w", p.ID, domain.ErrNotFound) }
    return nil
}
func (r *gormProductRepo) Delete(ctx context.Context, id uuid.UUID) error {
    result := r.db.WithContext(ctx).Delete(&productModel{}, "id = ?", id)
    if result.Error != nil { return fmt.Errorf("delete product: %w", result.Error) }
    if result.RowsAffected == 0 { return fmt.Errorf("product %s: %w", id, domain.ErrNotFound) }
    return nil
}
func (r *gormProductRepo) FindAll(ctx context.Context, f domain.ProductFilter) ([]domain.Product, error) {
    var ms []productModel
    err := r.db.WithContext(ctx).Limit(f.Limit).Offset((f.Page-1)*f.Limit).Find(&ms).Error
    if err != nil { return nil, fmt.Errorf("list products: %w", err) }
    out := make([]domain.Product, len(ms))
    for i, m := range ms { out[i] = fromModel(m) }
    return out, nil
}
```

> For associations, hooks, soft-deletes — see the **gin-database** skill.

---

## 4. Transactions

`execTx` handles begin/commit/rollback; callback receives a tx-backed `*sqlcgen.Queries`.

```go
// internal/repository/tx.go
package repository

import ("context"; "database/sql"; "fmt"
    "myapp/internal/domain"; "myapp/internal/repository/sqlc/sqlcgen")

func execTx(ctx context.Context, db *sql.DB, fn func(*sqlcgen.Queries) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return fmt.Errorf("begin tx: %w", err) }
    if err := fn(sqlcgen.New(tx)); err != nil {
        if rb := tx.Rollback(); rb != nil { return fmt.Errorf("rollback: %w (original: %v)", rb, err) }
        return err
    }
    return tx.Commit()
}
func TransferStock(ctx context.Context, db *sql.DB, fromID, toID string, qty int32) error {
    return execTx(ctx, db, func(q *sqlcgen.Queries) error {
        src, err := q.GetProductForUpdate(ctx, fromID)
        if err != nil { return fmt.Errorf("get source: %w", err) }
        if src.Stock < qty { return fmt.Errorf("insufficient stock: %w", domain.ErrConflict) }
        if err := q.DecrementStock(ctx, sqlcgen.DecrementStockParams{ID: fromID, Amount: qty}); err != nil {
            return fmt.Errorf("decrement: %w", err)
        }
        return q.IncrementStock(ctx, sqlcgen.IncrementStockParams{ID: toID, Amount: qty})
    })
}
```

GORM: `r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })` — commits on `nil`, rolls back on error.

---

## 5. Connection Pooling

Configure once in `main.go`, not inside constructors.

```go
db.SetMaxOpenConns(25)                  // max concurrent connections
db.SetMaxIdleConns(10)                  // idle connections kept alive
db.SetConnMaxLifetime(5 * time.Minute)  // recycle to avoid stale TCP
db.SetConnMaxIdleTime(2 * time.Minute)  // release idle sooner
if err := db.PingContext(ctx); err != nil { return fmt.Errorf("db ping: %w", err) }
```

Tune `MaxOpenConns` to your DB plan's connection limit.

---

## 6. Query Patterns

**Pagination:** clamp `f.Limit` (default 20, max 100), then `offset := (f.Page-1) * f.Limit`.

**Dynamic filtering** — parameterised WHERE; never interpolate user strings:
```go
clauses, args, n := []string{"1=1"}, []any{}, 1
if f.Name != "" { clauses = append(clauses, fmt.Sprintf("name ILIKE $%d", n)); args = append(args, "%"+f.Name+"%"); n++ }
if f.MinPrice > 0 { clauses = append(clauses, fmt.Sprintf("price >= $%d", n)); args = append(args, f.MinPrice); n++ }
// append LIMIT $n OFFSET $n+1 last
```

**Safe sort whitelist** — prevents ORDER BY injection:
```go
var allowedSort = map[string]string{"name": "name", "price": "price", "created_at": "created_at"}
func safeOrderBy(col, dir string) string {
    if allowedSort[col] == "" { col = "created_at" }
    if dir != "asc" { dir = "desc" }
    return fmt.Sprintf("%s %s", col, dir)
}
```

---

## 7. Testing

**Unit — mock interface (testify/mock):**
```go
type MockProductRepository struct{ mock.Mock }
func (m *MockProductRepository) FindByID(ctx context.Context, id uuid.UUID) (*domain.Product, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil { return nil, args.Error(1) }
    return args.Get(0).(*domain.Product), args.Error(1)
}
// Remaining methods: return m.Called(...).Error(0) or (Get(0), Error(1))

func TestGetProduct_NotFound(t *testing.T) {
    repo, id := new(MockProductRepository), uuid.New()
    repo.On("FindByID", mock.Anything, id).Return(nil, domain.ErrNotFound)
    _, err := usecase.NewProductUsecase(repo).GetProduct(context.Background(), id)
    assert.True(t, errors.Is(err, domain.ErrNotFound))
    repo.AssertExpectations(t)
}
```

**Integration — testcontainers (`-tags=integration`):**
```go
//go:build integration
func TestProductRepository_CreateAndFind(t *testing.T) {
    ctx := context.Background()
    pgc, err := postgres.RunContainer(ctx,
        postgres.WithDatabase("testdb"), postgres.WithUsername("test"), postgres.WithPassword("test"))
    require.NoError(t, err)
    t.Cleanup(func() { _ = pgc.Terminate(ctx) })
    connStr, _ := pgc.ConnectionString(ctx, "sslmode=disable")
    db, _ := sql.Open("postgres", connStr); defer db.Close()
    repo := repository.NewProductRepository(db) // run migration DDL first
    p := &domain.Product{ID: uuid.New(), Name: "Widget", Price: 999, Stock: 10,
        CreatedAt: time.Now(), UpdatedAt: time.Now()}
    require.NoError(t, repo.Create(ctx, p))
    found, err := repo.FindByID(ctx, p.ID)
    require.NoError(t, err); assert.Equal(t, p.Name, found.Name)
}
```

Run: `go test -tags=integration ./internal/repository/...`

---

## See Also

- **[error-handling.md](error-handling.md)** — repo error propagation to HTTP
- **[testing-by-layer.md](testing-by-layer.md)** — full mock patterns and coverage goals
- **[dependency-injection.md](dependency-injection.md)** — wiring repositories into usecases
- **gin-database** skill — GORM associations, migrations, sqlx
