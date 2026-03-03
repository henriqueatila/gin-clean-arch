# Input Sanitization

Binding tags (`binding:"required,min=2,max=200"`) validate structure but do not sanitize content. Treat all user-supplied strings as untrusted data.

---

## Sanitize at the Delivery Boundary

Strip control characters and trim whitespace **after** `ShouldBind*` succeeds, **before** mapping to domain input.

```go
// internal/delivery/http/sanitize.go
package http

import "strings"

// SanitizeString strips control characters and trims whitespace.
// Call on every free-text field after ShouldBind* succeeds.
func SanitizeString(s string) string {
    return strings.Map(func(r rune) rune {
        if r < 32 && r != '\n' && r != '\r' && r != '\t' {
            return -1 // drop control chars
        }
        return r
    }, strings.TrimSpace(s))
}
```

> **Note:** `SanitizeString` preserves `\n`, `\r`, `\t` for multi-line text fields. For values used in HTTP headers, strip all control characters: `strings.Map(func(r rune) rune { if r < 32 { return -1 }; return r }, s)`.

Usage in handler:

```go
// In handler — sanitize after binding, before mapping to domain input
product, err := h.uc.CreateProduct(c.Request.Context(), domain.CreateProductInput{
    Name:        SanitizeString(req.Name),
    Description: SanitizeString(req.Description),
    Price:       req.Price,
    Stock:       req.Stock,
})
```

---

## Rules

- **Where:** Sanitize free-text strings (`Name`, `Description`) at the delivery boundary — never inside domain or usecase.
- **Numerics:** Numeric and enum fields validated by binding tags are already safe.
- **HTML output:** Apply `html.EscapeString` at render time (not at ingestion).
- **SQL:** Parameterised queries (`$1, $2...`) prevent injection — never interpolate user strings into queries.

---

## See Also

- **[error-handling.md](error-handling.md)** — Validation error formatting
- **[layer-separation.md](layer-separation.md)** — Why sanitization lives in delivery, not domain
- **golang-gin-api** skill — `ShouldBind*` variants, custom validators, file upload sanitization
