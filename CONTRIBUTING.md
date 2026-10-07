# Contributing to Manchester Metrolink Routing

## Git Workflow

> ⚠️ **NEVER commit directly to `main`.** Always use feature branches.
>
> **PR base:** target the `main` branch

### Branch Naming

```text
type/short-description
```

Common types:

| Prefix      | Purpose                   |
| ----------- | ------------------------- |
| `feat/`     | New features              |
| `fix/`      | Bug fixes                 |
| `refactor/` | Code restructuring        |
| `docs/`     | Documentation changes     |
| `test/`     | Test additions/fixes      |
| `chore/`    | Tooling, CI, dependencies |

Examples:

```text
feat/gtfs-data-cleaner
fix/overnight-trip-time
refactor/astar-heuristic
docs/build-guide
```

Use lowercase English words separated by `-`.

---

### Commit Messages

Use Conventional Commit format:
[https://www.conventionalcommits.org](https://www.conventionalcommits.org)

```text
type(scope): short description
```

Common types:

- `feat` — new feature
- `fix` — bug fix
- `docs` — documentation
- `test` — tests
- `refactor` — refactoring without changing behavior
- `chore` — maintenance or tooling

Scopes used in this project:

| Scope      | Covers                                                     |
| ---------- | ---------------------------------------------------------- |
| `pipeline` | `data_pipeline/` — GTFS cleaning and graph building        |
| `backend`  | `backend/` — C++ A\* core, Express API, `evaluate.py`      |
| `frontend` | `frontend/` — React web app                                |
| `db`       | `database.sql`, `metrolink.db`, MySQL schema               |
| `report`   | `report/` — project report notebook                        |
| `plan`     | `plan/` — weekly plans                                     |

Examples:

```text
feat(pipeline): add GTFS data cleaner notebook
fix(backend): keep times above 24h for overnight trips
docs: update build guide
feat(frontend): center map on Manchester
refactor(backend): extract heuristic function
chore: add .gitignore
```

`scope` is optional.

Avoid vague commit messages such as:

```text
fix
update
change
test
final
```

---

### Pull Request Name

PR titles should follow the same format as commit messages:

```text
type(scope): short description
```

Examples:

```text
feat(pipeline): build graph database from cleaned GTFS
fix(frontend): remove duplicate station names in suggestions
refactor(backend): extract peak-hour multiplier
```

---

### Pull Request Description

Use this template:

```markdown
## Changes

- What was changed
- What was added or removed

## Why

Briefly explain why this change is needed.

## How to test

1. Step to test the change
2. Run the affected part (notebook, `node server.js`, `npm run dev`)
3. Verify expected behavior

## Checklist

- [ ] Code works as expected
- [ ] Affected journeys in Step 9 of `HUONG_DAN_BUILD.md` still give the expected result
- [ ] No hardcoded secrets (e.g. the MySQL password in `backend/server.js`)
- [ ] Documentation updated if needed
```
