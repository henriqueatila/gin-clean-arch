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

## Extended Sanitization

### Strip DEL and High Control Characters

`SanitizeString` above strips ASCII control chars (0–31) but misses the DEL character (127) and Unicode control categories. For stricter sanitization:

```go
// StrictSanitizeString strips all control characters including DEL (127)
// and Unicode control categories (Cc, Cf), then trims whitespace.
func StrictSanitizeString(s string) string {
    return strings.Map(func(r rune) rune {
        if r < 32 && r != '\n' && r != '\r' && r != '\t' {
            return -1
        }
        if r == 127 { // DEL character
            return -1
        }
        if unicode.IsControl(r) {
            return -1
        }
        return r
    }, strings.TrimSpace(s))
}
```

> Import `"unicode"` when using `StrictSanitizeString`. Use this variant for fields displayed in admin panels or exported to CSV/PDF.

### Post-Sanitization Length Validation

Binding tags validate the raw input length. After sanitization, the string may be shorter (stripped characters) or remain the same. Always re-check length constraints after sanitization:

```go
func (h *ProductHandler) Create(c *gin.Context) {
    var req createProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "validation failed"})
        return
    }
    // Sanitize first, then validate business length constraints
    name := SanitizeString(req.Name)
    if len(name) < 2 {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "name too short after sanitization"})
        return
    }
    // ...
}
```

### Header-Safe Values

For values used in HTTP headers (e.g., correlation IDs, filenames in Content-Disposition), strip ALL control characters — do not preserve newlines:

```go
// SanitizeHeader strips all control characters for safe use in HTTP headers.
func SanitizeHeader(s string) string {
    return strings.Map(func(r rune) rune {
        if r < 32 || r == 127 { return -1 }
        return r
    }, strings.TrimSpace(s))
}
```

### LIKE Pattern Safety

When user input is used in SQL `LIKE`/`ILIKE` queries, escape wildcard characters to prevent pattern abuse. See [repository-pattern.md](repository-pattern.md#6-query-patterns) for the `escapeLike` function.

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
