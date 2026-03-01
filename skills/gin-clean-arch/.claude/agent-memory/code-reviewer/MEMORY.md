# Code Reviewer Memory - gin-clean-arch

## Project Type
Agent Skill (documentation with Go code examples), not a runnable project. No build/test commands to run.

## Key Conventions
- SKILL.md max 500 lines, each reference max 300 lines
- Product entity (never User) throughout all examples
- `ShouldBind*` (not `Bind*`), `gin.New()` (not `gin.Default()`), `log/slog` (not fmt/log.Println)
- BAD/anti-pattern code examples intentionally violate rules -- skip those during audits
- `fmt.Fprintf(os.Stderr, ...)` in TestMain is acceptable Go idiom (slog not yet configured)

## Known Issues (as of 2026-03-01)
- `project-scaffolding.md` L61-75: domain errors.go imports `net/http` -- contradicts Rule 1. Should use raw ints like SKILL.md does.
- `AppError` json tags inconsistent: SKILL.md uses struct tags, error-handling.md uses custom MarshalJSON. Should standardize.
- `AppError.Code` stores HTTP status codes in domain -- pragmatic trade-off, documented but debatable.

## Report Location
Reports go to: `plans/reports/code-reviewer-{YYMMDD}-{HHMM}-{slug}.md`
