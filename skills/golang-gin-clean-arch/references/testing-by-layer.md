# Testing by Layer

Testing strategies per layer for Go/Gin clean architecture. All examples use the `Product` entity.

**Contents:** [Mock Strategy](#mock-strategy) · [Test Organization](#test-organization) · [Usecase Tests](#usecase-unit-tests) · [Handler Tests](#handler-unit-tests) · [Integration Tests](#repository-integration-tests) · [Mock Generation](#mock-generation) · [Fixtures](#test-fixtures) · [Coverage](#coverage-goals) · [Anti-Patterns](#testing-anti-patterns)

## Mock Strategy

**Mocks belong at system boundaries only.** Over-mocking creates fragile tests.

| Layer | Test with | Mock? |
|-------|-----------|-------|
| Domain | Pure unit tests | **Never** — pure logic, no deps |
| Usecase | Unit tests | **Yes** — mock `ProductRepository` |
| Handler | Unit tests | **Yes** — mock `ProductUsecase` |
| Repository | Integration tests | **Never** — use real DB (testcontainers) |

Only mock interfaces at external boundaries. Mocking something that isn't a boundary interface indicates an architecture leak.

---

## Test Organization

```
internal/domain/mocks/              # generated mocks (ProductRepository, ProductUsecase)
internal/usecase/product_usecase_test.go     # unit — mock repo
internal/repository/product_repository_test.go  # //go:build integration
internal/delivery/http/product_handler_test.go  # unit — mock usecase
internal/testutil/fixtures.go       # NewTestProduct, NewTestCreateInput
internal/testutil/db.go             # RunMigrations (integration only)
```

Helpers in `internal/testutil/` only. `//go:build integration` must be the **first line** (before `package`).

---

## Usecase Unit Tests

```go
// internal/usecase/product_usecase_test.go
package usecase_test

import (
	"context"
	"errors"
	"testing"

	"github.com/google/uuid"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/require"
	"myapp/internal/domain"
	"myapp/internal/domain/mocks"
	"myapp/internal/usecase"
)

func TestCreateProduct(t *testing.T) {
	valid := domain.CreateProductInput{Name: "Widget", Price: 999, Stock: 10}
	tests := []struct {
		name    string
		input   domain.CreateProductInput
		repoErr error
		wantErr error
	}{
		{name: "success", input: valid},
		{name: "duplicate → conflict", input: valid,
			repoErr: domain.ErrConflict, wantErr: domain.ErrConflict},
		{name: "zero price → validation",
			input: domain.CreateProductInput{Name: "W"}, wantErr: domain.ErrValidation},
		{name: "db error → internal", input: valid, repoErr: errors.New("conn reset")},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			repo := mocks.NewProductRepository(t)
			if tc.input.Price > 0 { // repo not reached when validation fails
				repo.On("Create", context.Background(),
					mock.MatchedBy(func(*domain.Product) bool { return true })).
					Return(tc.repoErr).Maybe()
			}
			product, err := usecase.NewProductUsecase(repo).
				CreateProduct(context.Background(), tc.input)
			if tc.wantErr != nil {
				require.ErrorIs(t, err, tc.wantErr)
				assert.Nil(t, product)
				return
			}
			if tc.repoErr != nil {
				require.Error(t, err)
				var appErr *domain.AppError
				assert.False(t, errors.As(err, &appErr), "db errors must not be AppError")
				return
			}
			require.NoError(t, err)
			assert.Equal(t, int64(999), product.Price)
			assert.NotEqual(t, uuid.Nil, product.ID)
		})
	}
}
```

Constructor returns the **interface**. Use `require` for fatal checks, `assert` for non-fatal.

---

## Handler Unit Tests

```go
// internal/delivery/http/product_handler_test.go
package http_test

import (
	"bytes"
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/require"
	delivery "myapp/internal/delivery/http"
	"myapp/internal/domain"
	"myapp/internal/domain/mocks"
)

func init() { gin.SetMode(gin.TestMode) }

func newTestRouter(uc domain.ProductUsecase) *gin.Engine {
	r := gin.New()
	h := delivery.NewProductHandler(uc, slog.New(slog.NewTextHandler(os.Stderr, nil)))
	delivery.RegisterProductRoutes(r.Group("/api/v1"), h)
	return r
}

func TestCreateProductHandler(t *testing.T) {
	validBody := map[string]any{"name": "Widget", "price": 999, "stock": 5}
	tests := []struct {
		name       string
		body       any
		setup      func(*mocks.ProductUsecase)
		wantStatus int
	}{
		{name: "success → 201", body: validBody, wantStatus: http.StatusCreated,
			setup: func(m *mocks.ProductUsecase) {
				m.On("CreateProduct", mock.Anything, mock.Anything).
					Return(&domain.Product{ID: uuid.New(), Name: "Widget", Price: 999}, nil)
			}},
		{name: "bad JSON → 422", body: `{bad`,
			setup: func(*mocks.ProductUsecase) {}, wantStatus: http.StatusUnprocessableEntity},
		{name: "conflict → 409", body: validBody, wantStatus: http.StatusConflict,
			setup: func(m *mocks.ProductUsecase) {
				m.On("CreateProduct", mock.Anything, mock.Anything).
					Return(nil, domain.ErrConflict)
			}},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			uc := mocks.NewProductUsecase(t)
			tc.setup(uc)
			body, err := json.Marshal(tc.body)
			require.NoError(t, err)
			req, err := http.NewRequest(http.MethodPost, "/api/v1/products", bytes.NewReader(body))
			require.NoError(t, err)
			req.Header.Set("Content-Type", "application/json")
			w := httptest.NewRecorder()
			newTestRouter(uc).ServeHTTP(w, req)
			assert.Equal(t, tc.wantStatus, w.Code)
		})
	}
}
```

`httptest.NewRecorder()` + `ServeHTTP` — no TCP. Test observable HTTP behavior only; never test `handleError` internals.

---

## Repository Integration Tests

```go
//go:build integration

package repository_test

import (
	"context"
	"database/sql"
	"fmt"
	"os"
	"testing"
	"time"

	_ "github.com/lib/pq"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
	"github.com/testcontainers/testcontainers-go/wait"
	"myapp/internal/domain"
	"myapp/internal/repository"
	"myapp/internal/testutil"
)

var testDB *sql.DB

func TestMain(m *testing.M) {
	ctx := context.Background()
	pgc, err := postgres.RunContainer(ctx,
		testcontainers.WithImage("postgres:16-alpine"),
		postgres.WithDatabase("testdb"), postgres.WithUsername("test"), postgres.WithPassword("test"),
		testcontainers.WithWaitStrategy(wait.ForLog("database system is ready to accept connections").
			WithOccurrence(2).WithStartupTimeout(30*time.Second)),
	)
	if err != nil { fmt.Fprintf(os.Stderr, "start postgres: %v\n", err); os.Exit(1) }
	defer pgc.Terminate(ctx) //nolint:errcheck
	dsn, _ := pgc.ConnectionString(ctx, "sslmode=disable")
	testDB, _ = sql.Open("postgres", dsn)
	defer testDB.Close()
	testutil.RunMigrations(testDB)
	os.Exit(m.Run())
}

func TestProductRepository_Create(t *testing.T) {
	t.Cleanup(func() { testDB.Exec("DELETE FROM products") }) //nolint:errcheck
	repo := repository.NewProductRepository(testDB)
	p := testutil.NewTestProduct(nil)
	require.NoError(t, repo.Create(context.Background(), p))
	found, err := repo.FindByID(context.Background(), p.ID)
	require.NoError(t, err)
	assert.Equal(t, p.Name, found.Name)
	// duplicate must surface ErrConflict, not a raw *pq.Error
	assert.ErrorIs(t, repo.Create(context.Background(), p), domain.ErrConflict)
}
```

Run: `go test -tags integration ./internal/repository/...` — `TestMain` owns the container (one per package); `t.Cleanup` truncates rows for isolation.

---

## Mock Generation

Add `go:generate` directives above each interface in `internal/domain/product.go`:

```go
//go:generate mockery --name=ProductRepository --outpkg=mocks --output=./mocks
type ProductRepository interface { /* ... */ }

//go:generate mockery --name=ProductUsecase --outpkg=mocks --output=./mocks
type ProductUsecase interface { /* ... */ }
```

```bash
go generate ./internal/domain/...   # regenerates all mocks
```

---

## Test Fixtures

Factory functions in `internal/testutil/fixtures.go`:

```go
func NewTestProduct(overrides *domain.Product) *domain.Product {
	p := &domain.Product{ID: uuid.New(), Name: "Test Widget", Price: 999, Stock: 10,
		CreatedAt: time.Now().UTC(), UpdatedAt: time.Now().UTC()}
	if overrides != nil {
		if overrides.Name != "" { p.Name = overrides.Name }
		if overrides.Price != 0 { p.Price = overrides.Price }
	}
	return p
}

func NewTestCreateInput() domain.CreateProductInput {
	return domain.CreateProductInput{Name: "Test Widget", Price: 999, Stock: 10}
}
```

---

## Coverage Goals

| Layer | Target | Rationale |
|---|---|---|
| Domain | 95%+ | Pure logic, no external deps |
| Usecase | 90%+ | Core business logic |
| Handler | 80%+ | Cover binding + all status-code paths |
| Repository | 70%+ | Integration tests count toward total |

---

## Testing Anti-Patterns

**Never mock:** stdlib types, domain logic, repositories in integration tests. **Never share:** mutable fixtures across sub-tests. **Always:** gate integration tests with `-tags integration`, include error-path cases, check `rows.Err()` after iteration.

---

**See also:** [layer-separation.md](layer-separation.md) · [dependency-injection.md](dependency-injection.md) · [error-handling.md](error-handling.md) · [project-scaffolding.md](project-scaffolding.md)
