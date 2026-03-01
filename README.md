# gin-clean-arch

Agent Skill for Clean Architecture with Go and the Gin framework.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## What Is This

A single Agent Skill that teaches clean architecture patterns for Go/Gin APIs. Covers layer separation, dependency injection, repository pattern, error handling, testing by layer, and project scaffolding.

Works standalone. Enhanced with [gin-best-practices](https://github.com/henriqueatila/gin-best-practices) for Gin-specific patterns.

---

## Install

```bash
npx skills add henriqueatila/gin-clean-arch --skill gin-clean-arch
```

### Manual Installation

```bash
curl -L https://github.com/henriqueatila/gin-clean-arch/raw/main/skills/gin-clean-arch.zip -o gin-clean-arch.zip
unzip gin-clean-arch.zip -d .claude/skills/gin-clean-arch/
```

---

## Skill Contents

| File | Description |
|------|-------------|
| **SKILL.md** | 5 golden rules, 4-layer overview, DI bootstrap, error flow |
| **references/layer-separation.md** | Layer responsibilities, dependency rule enforcement |
| **references/layer-separation-antipatterns.md** | Anti-patterns (bad→good), migration guide |
| **references/dependency-injection.md** | Manual DI, DI container pattern, testing with DI |
| **references/dependency-injection-alternatives.md** | Scaling DI, Wire/Fx alternatives |
| **references/repository-pattern.md** | SQLC, GORM, transactions, query patterns |
| **references/error-handling.md** | Domain errors, propagation, HTTP mapping |
| **references/testing-by-layer.md** | Mock-per-layer, testcontainers, coverage goals |
| **references/project-scaffolding.md** | From-scratch setup, Makefile, configuration |

---

## Design Principles

**Opinionated patterns** — One right way per topic, chosen for production correctness.

**PostgreSQL default** — All database examples target PostgreSQL. SQLC as primary, GORM as alternative.

**Production-first** — `gin.New()` + explicit middleware, `log/slog`, `ShouldBind*`, environment variables.

**Go 1.24+** — Uses stdlib features (`log/slog`, `context`, `errors.As`).

**Manual DI** — Transparent constructor-based injection. No Wire/Fx unless teaching alternatives.

**Product domain model** — All examples use `Product` entity (not `User` — differentiates from gin-best-practices).

---

## Architecture

### Progressive Disclosure

1. **SKILL.md** — The 80% you need daily. Loaded automatically. Under 500 lines.
2. **references/*.md** — The 20% for deep dives. Loaded on demand. Each under 300 lines.

### Complementarity

This skill is standalone. It teaches clean architecture structure and patterns. For Gin-specific details (routing, middleware, JWT, deployment), see [gin-best-practices](https://github.com/henriqueatila/gin-best-practices).

---

## Directory Structure

```
gin-clean-arch/
├── README.md
├── CLAUDE.md
├── AGENTS.md
├── LICENSE
│
├── skills/
│   └── gin-clean-arch/
│       ├── SKILL.md
│       ├── metadata.json
│       ├── README.md
│       └── references/
│           ├── layer-separation.md
│           ├── layer-separation-antipatterns.md
│           ├── dependency-injection.md
│           ├── dependency-injection-alternatives.md
│           ├── repository-pattern.md
│           ├── error-handling.md
│           ├── testing-by-layer.md
│           └── project-scaffolding.md
│
├── skills/
│   └── gin-clean-arch.zip
│
└── docs/
    └── SPECIFICATION.md
```

---

## Contributing

1. Follow the design principles above — patterns must be production-ready
2. Code examples must compile and handle errors (no `_` for errors, no `fmt.Println`)
3. SKILL.md must stay under 500 lines — move detail to reference files
4. Reference files over 300 lines must be split; over 200 lines must include a TOC
5. Use `gin.New()`, `log/slog`, `ShouldBind*`, and `context.Context` consistently
6. Use `Product` entity in all examples (not `User`)

---

## License

MIT License — see [LICENSE](LICENSE) for details.
