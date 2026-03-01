# CLAUDE.md

Agent guidance for the `gin-clean-arch` repository.

## What This Repo Is

A single Agent Skill for building Go/Gin APIs with Clean Architecture. Published on [skills.sh](https://skills.sh). The skill lives under `skills/gin-clean-arch/` and follows the [Agent Skills open standard](https://agentskills.io).

## Repository Structure

```
gin-clean-arch/
├── skills/
│   └── gin-clean-arch/
│       ├── SKILL.md              # Primary skill file (auto-loaded)
│       ├── metadata.json
│       ├── README.md
│       └── references/
│           ├── layer-separation.md
│           ├── dependency-injection.md
│           ├── repository-pattern.md
│           ├── error-handling.md
│           ├── testing-by-layer.md
│           └── project-scaffolding.md
├── CLAUDE.md             # This file (agent guidance)
├── AGENTS.md             # Multi-agent guidance
├── README.md             # Project overview
└── LICENSE               # MIT
```

## Skill Format

The skill directory contains:
- `SKILL.md` — Primary skill file with YAML frontmatter (`name`, `description`, `license`, `metadata`). Under 500 lines. Loaded by agents automatically.
- `references/*.md` — Deep-dive reference files loaded on demand. Each under 300 lines.
- `metadata.json` — Version, author, abstract, external references.
- `README.md` — Brief description and structure overview.

## Conventions

- **Domain model**: Use `Product` entity across all examples (not `User` — differentiates from gin-best-practices).
- **Gin API calls**: Must match official Gin documentation exactly. Never invent methods.
- **Binding**: Use `ShouldBind*` (not `Bind*`) in all examples.
- **Server setup**: Use `gin.New()` + explicit `r.Use(...)` (not `gin.Default()`).
- **Goroutine safety**: Call `c.Copy()` before passing `*gin.Context` to goroutines.
- **Logging**: Use `log/slog` (not `fmt.Println` or `log.Println`).
- **Context**: Pass `c.Request.Context()` to all downstream blocking calls.
- **Handlers**: Thin — bind input, call usecase, format response. No DB calls in handlers.
- **DI**: Manual constructor-based injection. No Wire/Fx unless explicitly teaching alternatives.

## Complementarity

This skill is standalone. It is enhanced by the **gin-best-practices** skills (`gin-api`, `gin-auth`, `gin-database`, `gin-testing`, `gin-deploy`) for Gin-specific patterns. Cross-references use skill names (e.g., "see the **gin-api** skill"), not file paths.

## Modifying Skills

1. Read official Gin documentation before writing any Gin code
2. Keep `SKILL.md` under 500 lines — move detail to `references/`
3. Keep reference files under 300 lines — add TOC if over 200 lines
4. Use `ShouldBind*`, `gin.New()`, `c.Copy()`, `log/slog`, and `context.Context` consistently
5. All code examples must compile and handle errors
