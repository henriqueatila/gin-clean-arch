# AGENTS.md

Multi-agent guidance for the `gin-clean-arch` repository.

## Repository Overview

1 Agent Skill for Clean Architecture with Go/Gin, published on skills.sh. Skill lives under `skills/gin-clean-arch/`, build docs under `docs/`.

## Agent Roles

### Skill Author
- Writes or updates `SKILL.md` and `references/*.md` files
- Must read `docs/SPECIFICATION.md` for content requirements
- Keeps `SKILL.md` under 500 lines; moves overflow to reference files
- Uses `Product` domain model in all examples

### Reviewer
- Verifies all Gin API calls against official documentation
- Checks `ShouldBind*` usage (not `Bind*`), `gin.New()` (not `gin.Default()`), `c.Copy()` for goroutines
- Validates YAML frontmatter fields in `SKILL.md`
- Runs quality checklist from `docs/SPECIFICATION.md`

### Packager
- Generates zip file under `skills/` for the skill
- Updates `metadata.json` version fields
- Ensures `README.md` paths match actual structure

## File Ownership

| Path | Owner |
|------|-------|
| `skills/gin-clean-arch/SKILL.md` | Skill Author |
| `skills/gin-clean-arch/references/*.md` | Skill Author |
| `skills/gin-clean-arch/metadata.json` | Packager |
| `skills/gin-clean-arch/README.md` | Packager |
| `skills/gin-clean-arch.zip` | Packager |
| `docs/*` | Skill Author |
| `README.md` | Packager |

## Conventions

- All code examples must compile, handle errors, use `context.Context`, and use `log/slog`
- Use the `Product` domain model across all examples (defined in `docs/SPECIFICATION.md`)
- Cross-references to gin-best-practices use skill names (e.g., "see the **gin-api** skill"), not file paths
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
