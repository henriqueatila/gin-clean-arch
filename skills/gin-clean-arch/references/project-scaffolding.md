# Project Scaffolding

Complete guide for bootstrapping a Go/Gin clean architecture project from scratch.

**Contents:** [Quick Start](#quick-start) · [Directory Structure](#directory-structure) · [Domain](#domain-layer) · [Usecase](#usecase-layer) · [Repository](#repository-layer) · [Delivery](#delivery-layer) · [main.go](#maingo) · [Configuration](#configuration) · [Makefile](#makefile) · [Dependencies](#recommended-dependencies)

## Quick Start

```bash
go mod init myapp

mkdir -p cmd/api \
         internal/{domain/mocks,usecase,repository,delivery/http,testutil} \
         pkg/httpserver config migrations

go get github.com/gin-gonic/gin \
       github.com/google/uuid \
       github.com/lib/pq \
       github.com/ilyakaznacheev/cleanenv
```

Add files in dependency order: `domain` → `usecase` → `repository` → `delivery` → `main`.

## Directory Structure

```
myapp/
├── cmd/api/main.go
├── config/config.go
├── internal/
│   ├── domain/
│   │   ├── product.go      # Entity, ProductRepository, ProductUsecase, input types
│   │   ├── errors.go       # AppError + sentinel vars
│   │   └── mocks/          # Generated testify mocks
│   ├── usecase/product_usecase.go
│   ├── repository/product_repository.go
│   ├── delivery/http/
│   │   ├── router.go       # gin.New(), middleware, /healthz, route groups
│   │   └── product_handler.go
│   └── testutil/
│       ├── fixtures.go     # NewTestProduct, NewTestCreateInput
│       └── db.go           # RunMigrations (integration tests)
├── migrations/
│   ├── 000001_create_products.up.sql
│   └── 000001_create_products.down.sql
├── .env.example
├── .air.toml
├── Makefile
└── go.mod
```

## Domain Layer

`internal/domain/product.go` — entity + interfaces. See `SKILL.md` for full listing.

`internal/domain/errors.go` — sentinel errors used across all layers:

```go
package domain

// No net/http import — domain must not know about HTTP (Rule 1)
type AppError struct {
	Code    int
	Message string
}

func (e *AppError) Error() string { return e.Message }

var (
	ErrNotFound   = &AppError{Code: 404, Message: "not found"}
	ErrConflict   = &AppError{Code: 409, Message: "already exists"}
	ErrValidation = &AppError{Code: 422, Message: "validation failed"}
	ErrForbidden  = &AppError{Code: 403, Message: "forbidden"}
)
```

Rule: `domain` never imports `gin`, `database/sql`, or any outer layer.

## Usecase Layer

```go
// internal/usecase/product_usecase.go
package usecase

// Constructor returns the interface — hides the struct, enables mock injection.
func NewProductUsecase(repo domain.ProductRepository) domain.ProductUsecase {
	return &productUsecase{repo: repo}
}

func (u *productUsecase) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
	if input.Price <= 0 {
		return nil, fmt.Errorf("create product: %w", domain.ErrValidation)
	}
	p := &domain.Product{
		ID: uuid.New(), Name: input.Name, Description: input.Description,
		Price: input.Price, Stock: input.Stock,
		CreatedAt: time.Now().UTC(), UpdatedAt: time.Now().UTC(),
	}
	if err := u.repo.Create(ctx, p); err != nil {
		return nil, fmt.Errorf("create product: %w", err)
	}
	return p, nil
}
// GetProduct, ListProducts, UpdateProduct, DeleteProduct follow the same pattern.
```

Rule: usecase never imports `gin`, `database/sql`, or `repository`.

## Repository Layer

```go
// internal/repository/product_repository.go
package repository

func NewProductRepository(db *sql.DB) domain.ProductRepository {
	return &postgresProductRepo{db: db}
}

func (r *postgresProductRepo) Create(ctx context.Context, p *domain.Product) error {
	_, err := r.db.ExecContext(ctx,
		`INSERT INTO products (id,name,description,price,stock,created_at,updated_at)
		 VALUES ($1,$2,$3,$4,$5,$6,$7)`,
		p.ID, p.Name, p.Description, p.Price, p.Stock, p.CreatedAt, p.UpdatedAt)
	if err != nil {
		var pqErr *pq.Error
		if errors.As(err, &pqErr) && pqErr.Code == "23505" {
			return fmt.Errorf("product %q: %w", p.Name, domain.ErrConflict)
		}
		return fmt.Errorf("insert product: %w", err)
	}
	return nil
}

// FindByID, FindAll, Update, Delete follow the same pattern: wrap sql.ErrNoRows →
// domain.ErrNotFound, unique violations → domain.ErrConflict. See SKILL.md.
```

## Delivery Layer

```go
// internal/delivery/http/router.go
func NewRouter(productUC domain.ProductUsecase, logger *slog.Logger) *gin.Engine {
	r := gin.New()
	r.Use(gin.Recovery(), requestLogger(logger))
	r.GET("/healthz", func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"status": "ok"}) })
	RegisterProductRoutes(r.Group("/api/v1"), NewProductHandler(productUC, logger))
	return r
}
```

`handleError` in `product_handler.go` — maps `*domain.AppError` to HTTP, logs the rest:

```go
func handleError(c *gin.Context, err error, logger *slog.Logger) {
	var appErr *domain.AppError
	if errors.As(err, &appErr) {
		c.JSON(appErr.Code, gin.H{"error": appErr.Message})
		return
	}
	logger.ErrorContext(c.Request.Context(), "unhandled error", "error", err)
	c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
}
```

Handler struct, `RegisterProductRoutes`, and CRUD methods: see `SKILL.md`.

## main.go

```go
// cmd/api/main.go — imports: context, database/sql, log/slog, net/http, os, os/signal, syscall, time
package main

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))

	cfg, err := config.Load()
	if err != nil { logger.Error("load config", "error", err); os.Exit(1) }

	db, err := sql.Open("postgres", cfg.DatabaseURL)
	if err != nil { logger.Error("open db", "error", err); os.Exit(1) }
	defer db.Close()
	db.SetMaxOpenConns(cfg.DBMaxOpenConns)
	db.SetMaxIdleConns(cfg.DBMaxIdleConns)
	db.SetConnMaxLifetime(cfg.DBConnMaxLifetime)
	if err := db.PingContext(context.Background()); err != nil {
		logger.Error("ping db", "error", err); os.Exit(1)
	}

	// DI: repo → usecase → handler → router (one line per layer)
	router := delivery.NewRouter(
		usecase.NewProductUsecase(repository.NewProductRepository(db)), logger)

	srv := &http.Server{Addr: cfg.Addr, Handler: router,
		ReadTimeout: 10 * time.Second, WriteTimeout: 30 * time.Second, IdleTimeout: 60 * time.Second}
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("server error", "error", err); os.Exit(1)
		}
	}()
	logger.Info("server starting", "addr", cfg.Addr)

	// graceful shutdown: HTTP first (drains requests), then DB pool
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil { logger.Error("shutdown", "error", err) }
	logger.Info("server stopped")
}
```

## Configuration

```go
// config/config.go
package config

type Config struct {
	Addr              string        `env:"SERVER_ADDR"          env-default:":8080"`
	DatabaseURL       string        `env:"DATABASE_URL"          env-required:"true"`
	DBMaxOpenConns    int           `env:"DB_MAX_OPEN_CONNS"    env-default:"25"`
	DBMaxIdleConns    int           `env:"DB_MAX_IDLE_CONNS"    env-default:"10"`
	DBConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" env-default:"5m"`
	Env               string        `env:"APP_ENV"              env-default:"development"`
}

func Load() (*Config, error) {
	var cfg Config
	if err := cleanenv.ReadEnv(&cfg); err != nil {
		return nil, fmt.Errorf("read env: %w", err)
	}
	return &cfg, nil
}
```

`.env.example` (commit; copy to `.env` locally — add `.env` to `.gitignore`):

```dotenv
SERVER_ADDR=:8080
DATABASE_URL=postgres://user:password@localhost:5432/myapp?sslmode=disable
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=10
DB_CONN_MAX_LIFETIME=5m
APP_ENV=development
```

## Makefile

```makefile
BINARY := bin/api
MAIN   := ./cmd/api
MIGS   := ./migrations
DB_URL ?= $(shell grep DATABASE_URL .env | cut -d= -f2-)

.PHONY: build run dev test test-integration coverage lint \
        migrate-up migrate-down migrate-create mocks sqlc clean

build:            ; go build -o $(BINARY) $(MAIN)
run: build        ; $(BINARY)
dev:              ; air
test:             ; go test -race -count=1 ./...
test-integration: ; go test -race -count=1 -tags integration ./...
coverage:         ; go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out -o coverage.html
lint:             ; golangci-lint run ./...
migrate-up:       ; migrate -path $(MIGS) -database "$(DB_URL)" up
migrate-down:     ; migrate -path $(MIGS) -database "$(DB_URL)" down 1
migrate-create:
	@read -p "Name: " n; migrate create -ext sql -dir $(MIGS) -seq $$n
mocks:            ; go generate ./internal/domain/...
sqlc:             ; sqlc generate
clean:            ; rm -rf bin/ coverage.out coverage.html
```

## Recommended Dependencies

| Purpose | Package |
|---|---|
| HTTP framework | `github.com/gin-gonic/gin` v1.10+ |
| UUID | `github.com/google/uuid` |
| Postgres driver | `github.com/lib/pq` or `github.com/jackc/pgx/v5` |
| Config/env | `github.com/ilyakaznacheev/cleanenv` |
| SQL codegen | `github.com/sqlc-dev/sqlc` |
| ORM (optional) | `gorm.io/gorm` |
| Test assertions | `github.com/stretchr/testify` |
| Mock generation | `github.com/vektra/mockery/v2` |
| DB containers | `github.com/testcontainers/testcontainers-go` |
| Migrations | `github.com/golang-migrate/migrate/v4` |
| Live reload (dev) | `github.com/air-verse/air` |
| Linter | `github.com/golangci/golangci-lint` |

**See also:** [layer-separation.md](layer-separation.md) · [dependency-injection.md](dependency-injection.md) · [repository-pattern.md](repository-pattern.md) · [error-handling.md](error-handling.md) · [testing-by-layer.md](testing-by-layer.md)
