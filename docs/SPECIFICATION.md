# SPECIFICATION.md

Content requirements and quality checklist for the `gin-clean-arch` skill.

## Shared Domain Model

All examples use the `Product` entity — **not** `User` (which is used by gin-best-practices). This intentional differentiation avoids confusion when both skills are installed.

### Product Entity

```go
type Product struct {
    ID          uuid.UUID
    Name        string
    Description string
    Price       int64  // cents
    Stock       int32
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

### Product Interfaces

```go
type ProductRepository interface {
    FindByID(ctx context.Context, id uuid.UUID) (*Product, error)
    FindAll(ctx context.Context, filter ProductFilter) ([]Product, error)
    Create(ctx context.Context, p *Product) error
    Update(ctx context.Context, p *Product) error
    Delete(ctx context.Context, id uuid.UUID) error
}

type ProductUsecase interface {
    GetProduct(ctx context.Context, id uuid.UUID) (*Product, error)
    ListProducts(ctx context.Context, filter ProductFilter) ([]Product, error)
    CreateProduct(ctx context.Context, input CreateProductInput) (*Product, error)
    UpdateProduct(ctx context.Context, id uuid.UUID, input UpdateProductInput) (*Product, error)
    DeleteProduct(ctx context.Context, id uuid.UUID) error
}
```

## Content Requirements

1. **All code must compile** — no pseudo-code, no `...` placeholders in function bodies
2. **All errors handled** — no `_` for error values
3. **Context propagated** — `context.Context` passed through all layers
4. **Structured logging** — `log/slog` only (no `fmt.Println`, `log.Println`)
5. **Gin conventions** — `ShouldBind*`, `gin.New()`, `c.Copy()` for goroutines
6. **Go 1.24+** — use stdlib features (`errors.As`, `log/slog`)

## Quality Checklist

- [ ] Layer boundaries respected (no Gin imports in usecase/domain)
- [ ] DI via constructors (no global state, no init())
- [ ] No business logic in HTTP handlers
- [ ] Interfaces defined in domain layer, implemented in outer layers
- [ ] Error types defined in domain, mapped to HTTP in delivery layer
- [ ] `Product` entity used (not `User`)
- [ ] SKILL.md under 500 lines
- [ ] Each reference file under 300 lines
- [ ] TOC added if reference file exceeds 200 lines

## Reference File Guidelines

- Each reference file focuses on one topic
- Include compilable code examples
- Show anti-patterns with corrections when relevant
- Cross-reference gin-best-practices by skill name (not file paths)
- End with "See also" section linking related references
