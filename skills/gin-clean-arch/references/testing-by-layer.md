# Testing by Layer

Testing strategies per layer for Go/Gin clean architecture. All examples use the `Product` entity.

**Contents:** [Mock Strategy](#mock-strategy) · [Test Organization](#test-organization) · [Usecase Tests](#usecase-unit-tests) · [Handler Tests](#handler-unit-tests) · [Integration Tests](#repository-integration-tests) · [Mock Generation](#mock-generation) · [Fixtures](#test-fixtures) · [Coverage](#coverage-goals) · [Anti-Patterns](#testing-anti-patterns)

---

## Mock Strategy

**Mocks belong at system boundaries only** — not at every layer. Over-mocking creates fragile tests that break on every refactor.

| Layer | Test with | Mock? | Why |
|-------|-----------|-------|-----|
| Domain | Pure unit tests | **Never** | Pure logic, no dependencies — test input/output directly |
| Usecase | Unit tests | **Yes — mock repository interfaces** | Repository is an external boundary (DB) |
| Handler | Unit tests | **Yes — mock usecase interfaces** | Usecase is the layer boundary |
| Repository | Integration tests | **Never mock — use real DB** | Must answer "does this actually work?" |

**Domain tests are pure — zero mocks:**
```go
func TestProduct_DeductStock(t *testing.T) {
    p := &domain.Product{Stock: 10}
    require.NoError(t, p.DeductStock(5))
    assert.Equal(t, int32(5), p.Stock)
    assert.ErrorIs(t, p.DeductStock(20), domain.ErrValidation) // insufficient
}

func TestProduct_Validate(t *testing.T) {
    assert.Error(t, (&domain.Product{Name: "", Price: 0}).Validate())
    assert.NoError(t, (&domain.Product{Name: "Widget", Price: 999, Stock: 5}).Validate())
}
```

**What to mock:** Only interfaces that represent external system boundaries (`ProductRepository`, third-party API clients). If you find yourself mocking something that isn't a boundary interface, your architecture has a leak.

---

## Test Organization

```
internal/
├── domain/mocks/
│   ├── product_repository.go   # implements domain.ProductRepository
│   └── product_usecase.go      # implements domain.ProductUsecase
├── usecase/product_usecase_test.go     # unit — mock repo
├── repository/product_repository_test.go  # //go:build integration
├── delivery/http/product_handler_test.go  # unit — mock usecase
└── testutil/
    ├── fixtures.go   # NewTestProduct, NewTestCreateInput, AnyProduct
    └── db.go         # RunMigrations (integration only)
```

- Helpers in `internal/testutil/` only — never inside a layer package.
- Mocks in `internal/domain/mocks/` — generated against domain interfaces.
- `//go:build integration` must be the **first line** (before `package`).

---

## Usecase Unit Tests

```go
// internal/usecase/product_usecase_test.go
package usecase_test

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

- Constructor returns the **interface** — swap real repo for mock without changing call sites.
- `require` for fatal checks, `assert` for non-fatal.

---

## Handler Unit Tests

```go
// internal/delivery/http/product_handler_test.go
package http_test

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
		{
			name: "success → 201",
			body: validBody,
			setup: func(m *mocks.ProductUsecase) {
				m.On("CreateProduct", mock.Anything, mock.Anything).
					Return(&domain.Product{ID: uuid.New(), Name: "Widget", Price: 999}, nil)
			},
			wantStatus: http.StatusCreated,
		},
		{name: "bad JSON → 400", body: `{bad`,
			setup: func(*mocks.ProductUsecase) {}, wantStatus: http.StatusBadRequest},
		{
			name: "conflict → 409",
			body: validBody,
			setup: func(m *mocks.ProductUsecase) {
				m.On("CreateProduct", mock.Anything, mock.Anything).
					Return(nil, domain.ErrConflict)
			},
			wantStatus: http.StatusConflict,
		},
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

- `httptest.NewRecorder()` + `ServeHTTP` — no real TCP connection.
- Test observable HTTP behavior only; never test `handleError` internals.

---

## Repository Integration Tests

```go
//go:build integration

package repository_test

var testDB *sql.DB

func TestMain(m *testing.M) {
	ctx := context.Background()
	pgc, err := postgres.RunContainer(ctx,
		testcontainers.WithImage("postgres:16-alpine"),
		postgres.WithDatabase("testdb"),
		postgres.WithUsername("test"),
		postgres.WithPassword("test"),
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").
				WithOccurrence(2).WithStartupTimeout(30*time.Second)),
	)
	if err != nil { fmt.Fprintf(os.Stderr, "start postgres: %v\n", err); os.Exit(1) }
	defer pgc.Terminate(ctx) //nolint:errcheck

	dsn, err := pgc.ConnectionString(ctx, "sslmode=disable")
	if err != nil { fmt.Fprintf(os.Stderr, "dsn: %v\n", err); os.Exit(1) }

	testDB, err = sql.Open("postgres", dsn)
	if err != nil { fmt.Fprintf(os.Stderr, "open db: %v\n", err); os.Exit(1) }
	defer testDB.Close()

	testutil.RunMigrations(testDB)
	os.Exit(m.Run())
}

func TestProductRepository_Create(t *testing.T) {
	t.Cleanup(func() {
		_, err := testDB.Exec("DELETE FROM products")
		require.NoError(t, err)
	})

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

Run: `go test -tags integration ./internal/repository/...`

- `TestMain` owns the container — one per package, shared across all tests.
- `t.Cleanup` truncates rows; tests stay independent without container restart.

---

## Mock Generation

Add `go:generate` directives directly above each interface in `internal/domain/product.go`:

```go
//go:generate mockery --name=ProductRepository --outpkg=mocks --output=./mocks
type ProductRepository interface { /* ... */ }

//go:generate mockery --name=ProductUsecase --outpkg=mocks --output=./mocks
type ProductUsecase interface { /* ... */ }
```

```bash
go generate ./internal/domain/...   # regenerates all mocks
```

Makefile: `make mocks` → `go generate ./internal/domain/...`

For a manual mock, embed `mock.Mock`, implement each interface method with `m.Called(...)`, and call `m.AssertExpectations(t)` in a `t.Cleanup` closure. Prefer `go generate` for anything beyond 2–3 methods.

---

## Test Fixtures

Factory functions in `internal/testutil/fixtures.go` prevent coupling test data to layer internals.

```go
// NewTestProduct returns a valid Product. Pass non-nil overrides for specific fields.
func NewTestProduct(overrides *domain.Product) *domain.Product {
	p := &domain.Product{ID: uuid.New(), Name: "Test Widget", Price: 999, Stock: 10,
		CreatedAt: time.Now(), UpdatedAt: time.Now()}
	if overrides != nil {
		if overrides.Name != "" { p.Name = overrides.Name }
		if overrides.Price != 0 { p.Price = overrides.Price }
	}
	return p
}

func NewTestCreateInput() domain.CreateProductInput {
	return domain.CreateProductInput{Name: "Test Widget", Price: 999, Stock: 10}
}

func AnyProduct() any { return mock.MatchedBy(func(*domain.Product) bool { return true }) }
```

- Only import `domain` from `testutil` — no cross-layer coupling.
- `*T` overrides (nil = all defaults) keeps function signatures stable.

---

## Coverage Goals

| Layer | Target | Rationale |
|---|---|---|
| Domain | 95%+ | Pure logic, no external deps |
| Usecase | 90%+ | Core business logic |
| Handler | 80%+ | Cover binding + all status-code paths |
| Repository | 70%+ | Integration tests count toward total |

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html
# Fail CI below 80%:
go tool cover -func=coverage.out | grep total | awk '{if ($3+0 < 80) exit 1}'
```

---

## Testing Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Mocking stdlib types (`*sql.DB`, `io.Reader`) | Mock only boundary interfaces (Repository, API client) |
| Mocking domain logic | Domain is pure — test with real inputs, zero mocks |
| Mocking repository in integration tests | Use real DB (testcontainers) — must prove it works |
| Sharing mutable fixtures across sub-tests | Fresh fixture per `t.Run` |
| `t.Skip()` on integration tests locally | Gate with `-tags integration`; run in CI |
| Asserting on unexported struct fields | Assert on returned value or HTTP response body |
| Table tests with zero error-path cases | Every exported func needs ≥1 error case |
| Ignoring `rows.Err()` after iteration | Always check — silent data loss otherwise |

---

**See also:** [layer-separation.md](layer-separation.md) · [dependency-injection.md](dependency-injection.md) · [error-handling.md](error-handling.md) · [project-scaffolding.md](project-scaffolding.md)
