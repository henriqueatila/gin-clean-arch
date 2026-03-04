# Dependency Injection — Scaling, Alternatives, and Common Mistakes

When to scale beyond manual DI, Google Wire and Uber Fx alternatives, and mistakes to avoid.

Companion file: [dependency-injection.md](dependency-injection.md) — manual DI, constructor patterns, testing.

---

## Scaling DI

Choose the DI approach based on project size and team familiarity.

| Size | Usecases | Approach | Where wiring lives |
|------|----------|----------|--------------------|
| Small | 1–4 | Flat variables in `main.go` | `cmd/api/main.go` |
| Medium | 5–12 | `Dependencies` struct + `RegisterRoutes` | `cmd/api/main.go` |
| Large | 13+ | Sub-containers per domain or Wire/Fx | Multiple `wire.go` files |

### Decision Criteria

- Fewer than 5 usecases: flat manual DI — no ceremony needed.
- 5–12 usecases: `Dependencies` struct keeps `main.go` readable without adding a framework.
- More than 12 usecases or multiple bounded contexts: evaluate Wire (compile-time) or Fx (runtime) to eliminate repetitive boilerplate.
- Shared dependencies (logger, db, config): always pass explicitly to each constructor — never via global variable.

---

## Alternative: Google Wire

Wire is a compile-time DI code generator from Google. It reads constructor signatures and generates the wiring glue in `wire_gen.go`.

### Minimal Wire Example

```go
// cmd/api/wire.go
// The wireinject build tag prevents this file from being compiled normally.
//go:build wireinject

package main

import (
    "database/sql"
    "log/slog"

    "github.com/google/wire"
    delivery "myapp/internal/delivery/http"
    "myapp/internal/repository"
    "myapp/internal/usecase"
)

// InitializeProductHandler is the Wire injector function.
// Wire reads the wire.Build call and generates the full body in wire_gen.go.
func InitializeProductHandler(db *sql.DB, logger *slog.Logger) (*delivery.ProductHandler, error) {
    wire.Build(
        repository.NewProductRepository, // *sql.DB → domain.ProductRepository
        usecase.NewProductUsecase,        // domain.ProductRepository → domain.ProductUsecase
        delivery.NewProductHandler,       // domain.ProductUsecase, *slog.Logger → *ProductHandler
    )
    return nil, nil // Wire replaces this body at generation time
}
```

Run `wire ./cmd/api/` to generate `wire_gen.go`. Commit the generated file — it is plain, readable Go.

### Wire Pros and Cons

| Pros | Cons |
|------|------|
| Compile-time safety — missing deps are build errors | Requires `wire` CLI installed and a generation step |
| Eliminates boilerplate at scale (15+ constructors) | Generated code can be disorienting to debug |
| Provider sets group domain wiring cleanly | `//go:build wireinject` tag is easy to forget |
| Works naturally with interfaces | Learning curve for teams new to code generation |

**Use Wire when:** the manual wiring block in `main.go` has grown past ~30 lines and expanding it with each new aggregate feels like maintenance risk.

---

## Alternative: Uber Fx

Fx is a runtime DI framework from Uber. Dependencies are registered via `fx.Provide` and resolved by reflection when the application starts.

### Minimal Fx Example

```go
// cmd/api/main.go
package main

import (
    "context"
    "database/sql"
    "log/slog"
    "net/http"
    "os"

    "github.com/gin-gonic/gin"
    _ "github.com/lib/pq"
    "go.uber.org/fx"

    delivery "myapp/internal/delivery/http"
    "myapp/internal/repository"
    "myapp/internal/usecase"
)

func main() {
    app := fx.New(
        fx.Provide(
            newLogger,
            openDatabase,
            repository.NewProductRepository, // db → domain.ProductRepository
            usecase.NewProductUsecase,        // domain.ProductRepository → domain.ProductUsecase
            delivery.NewProductHandler,       // domain.ProductUsecase, *slog.Logger → *ProductHandler
        ),
        fx.Invoke(registerServer),
    )
    app.Run()
}

func newLogger() *slog.Logger {
    return slog.New(slog.NewJSONHandler(os.Stdout, nil))
}

func openDatabase() (*sql.DB, error) {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        return nil, err
    }
    return db, nil
}

func newRouter(handler *delivery.ProductHandler) *gin.Engine {
    r := gin.New()
    r.Use(gin.Recovery())
    delivery.RegisterProductRoutes(r.Group("/api/v1"), handler)
    return r
}

func registerServer(handler *delivery.ProductHandler, logger *slog.Logger, lc fx.Lifecycle) {
    r := newRouter(handler)
    srv := &http.Server{Addr: ":8080", Handler: r}

    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            logger.Info("server starting", "addr", srv.Addr)
            go func() {
                if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
                    logger.Error("server error", "error", err)
                }
            }()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            logger.Info("server stopping")
            return srv.Shutdown(ctx)
        },
    })
}
```

### Fx Pros and Cons

| Pros | Cons |
|------|------|
| Lifecycle hooks (OnStart/OnStop) built in | Runtime errors — missing deps crash at startup, not compile time |
| `fx.Option` modules group domain wiring cleanly | Reflection makes IDE navigation and stack traces harder |
| Handles graceful shutdown for all components | Heavy dependency; adds meaningful complexity |
| Scales well for microservices with many components | Interface resolution is implicit and can surprise |

**Use Fx when:** the application has complex lifecycle needs — multiple HTTP servers, background workers, message consumers — and the team accepts the reflection trade-off.

---

## Common Mistakes

### Injecting Concrete Types

```go
// BAD — caller is coupled to a struct; cannot substitute a mock
func NewProductHandler(repo *repository.PostgresProductRepository) *ProductHandler { ... }

// GOOD — caller depends on the domain interface only
func NewProductHandler(uc domain.ProductUsecase, logger *slog.Logger) *ProductHandler { ... }
```

### Circular Dependencies

```go
// BAD — A needs B, B needs A; will not compile
func NewA(b *B) *A { ... }
func NewB(a *A) *B { ... }

// GOOD — extract shared behaviour into C; both A and B depend on C
func NewC() *C     { ... }
func NewA(c *C) *A { ... }
func NewB(c *C) *B { ... }
```

### Global State as a DI Shortcut

```go
// BAD — global variables are hidden dependencies; parallel tests corrupt each other
var DB     *sql.DB
var Logger *slog.Logger

func init() {
    DB, _ = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    Logger = slog.Default()
}

// GOOD — pass all dependencies explicitly through constructors
func NewProductRepository(db *sql.DB) domain.ProductRepository { ... }
func NewProductHandler(uc domain.ProductUsecase, logger *slog.Logger) *ProductHandler { ... }
```

### Over-Abstracting Early

```go
// BAD — a custom DI container for a 3-usecase app adds zero value
type Container struct {
    mu    sync.Mutex
    cache map[string]any
}

func (c *Container) Resolve(name string) any { ... }

// GOOD — three lines of manual wiring are always clearer than reflection magic
productRepo    := repository.NewProductRepository(db)
productUC      := usecase.NewProductUsecase(productRepo)
productHandler := delivery.NewProductHandler(productUC, logger)
```

---

## See Also

- [dependency-injection.md](dependency-injection.md) — Manual DI, constructor patterns, testing with mocks
- [layer-separation.md](layer-separation.md) — Why interfaces live in domain, not outer layers
- [project-scaffolding.md](project-scaffolding.md) — Complete `main.go` with config loading and graceful shutdown
