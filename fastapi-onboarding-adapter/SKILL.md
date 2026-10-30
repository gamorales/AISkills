---
name: fastapi-codebase-onboarding
description: Analyze an unfamiliar Python FastAPI repository and produce an evidence-backed onboarding brief covering application entrypoints, framework and ORM stack, core domain models, routing and dependency chains, a representative end-to-end request trace, critical modules, and a prioritized reading order. Use for onboarding, architecture audits, or questions such as "how does this backend work?" and "where should I start?"
---

# FastAPI Codebase Onboarding & Architecture Analyzer

## Mission

Help a Python engineer understand the backend's architecture and know what to read first. Prefer a concise, verified map over an exhaustive inventory.

Analyze the repository as it exists. Do not assume a conventional directory layout, a particular framework version, or the existence of layers such as `services/` or `repositories/`.

The workflow runs in three phases: **Discover** (scope, entrypoint, stack), **Reconstruct** (entities, routing, one request trace), **Verify and explain** (check claims, rank by evidence, emit the brief).

## Operating Rules

1. **Inspect before asserting.** Every path, symbol, route, relationship, or call-flow edge must come from source you opened. Never infer file contents from a filename.
2. **Cite navigable evidence.** Use `path/to/file.py:LINE`. If line numbers aren't available, cite path + symbol and say so.
3. **Separate fact from inference.** Label deductions `Inferred` and give the evidence. Use `Not found` / `Not verified` instead of turning failed searches into proof of absence — dynamic registration, aliases, and decorators hide things.
4. **Adapt to available tools.** Use `Glob` / `Grep` / `Read` / shell / IDE tools or equivalents. If one isn't available, use the closest method and disclose limits. The signal patterns below are *typical*, not required syntax.
5. **Protect secrets.** Never print `.env` contents, credential values, tokens, keys, or connection strings. You may report configuration *key names* and the files that load them.
6. **Respect scope.** Do not modify files, install dependencies, run migrations, start services, or execute code unless explicitly asked and permitted.
7. **Be proportionate.** A small API gets a small report; a monorepo gets a scoped one with disclosed limits. Target ~900 words excluding compact tables.
8. **Do not force FastAPI conclusions.** If the repo isn't demonstrably FastAPI, say what you found and stop FastAPI-specific analysis. For a mixed monorepo, scope to the relevant backend package.

## Exclusions and Scope

Exclude by default: `.git/`, `.venv/`, `venv/`, `env/`, `__pycache__/`, `node_modules/`, `build/`, `dist/`, caches; generated code (`*.pyi`, `*_pb2.py`, generated OpenAPI clients); migration history (`alembic/versions/`, `migrations/`) unless the question concerns schema evolution; `tests/` unless requested or needed to verify a flow.

Do not exclude config, deployment, or scripts wholesale when they explain startup or request wiring. Note meaningful exclusions in the scope line.

**Scale strategy**
- Small (< ~300 Python files): inspect all relevant first-party files.
- Medium: start at entrypoints and router registration, expand along real call paths.
- Large (≥ ~1,500 files): scope the first pass to the package containing the entrypoint; state the scope and offer to go deeper.
- Never read files > ~800 lines wholesale when targeted inspection is possible — read imports and definitions first, then focused ranges.
- If tools or context limits block verification, disclose the limitation; never imply completeness.

---

## Phase A — Discover

### Step 1. Find the entrypoint

Typical signals (adapt as needed):
- `FastAPI(` — direct construction vs. factory.
- `create_app`, `def lifespan`, `@app.on_event` — boot and lifecycle.
- `include_router`, `app.mount` — mounted routers and sub-apps.
- Packaging (`pyproject.toml`, `requirements*.txt`, `Dockerfile`, `Procfile`) — clues, not proof of runtime use.

Read `main.py` / `asgi.py` / factory in full — they're small and always first. If multiple apps exist and the target isn't obvious, scope to the main backend and state the assumption.

### Step 2. Detect the real stack

Determine from **imports and usage**, not directory names. Record only what source supports:

| Signal | Indicates |
|---|---|
| `from sqlalchemy.orm import DeclarativeBase`, `Mapped[`, `mapped_column` | SQLAlchemy 2.x |
| `declarative_base()`, `Column(`, `relationship(` | SQLAlchemy 1.x |
| `from sqlmodel import SQLModel` + `table=True` | SQLModel table |
| `from sqlmodel import SQLModel` (no `table=True`) | SQLModel data contract — not a table |
| `Tortoise`, `Beanie`, `Motor`, `ODMantic` | Async/document ORM |
| `from pydantic_settings import BaseSettings` | pydantic-settings v2 |
| `OAuth2PasswordBearer`, `HTTPBearer`, `jwt.decode`, `passlib` | Auth stack |

Also record:
- **Sync vs async** endpoints and dependencies.
- **DB session dependency** — read the actual implementation, not just its name. Does it yield, close, commit, roll back, or delegate? Do not assume `get_db` does anything in particular.
- **Lifespan / middleware / exception handlers** that affect a normal request.

Don't pin a version from one syntax clue if compatibility layers are visible. Use dependency metadata where available; otherwise report version as uncertain.

---

## Phase B — Reconstruct

### Step 3. Map domain models and contracts

Typical signals: `__tablename__`, `class \w+\(Base\)`, `class \w+\(SQLModel\)`; Pydantic `class \w+\((BaseModel|SQLModel)\)` under schemas/dto or inline.

For **persistence models**: class, table/collection, PK, important FKs, explicit relationships (direction/cardinality from declarations), and a one-line purpose grounded in fields and usage.

For **contracts**: `Create` / `Update` / `Read` per entity, shared bases, validation rules, aliases, `orm_mode`/`from_attributes`, and whether they're distinct from or combined with persistence models.

Note domain-relevant enums, dataclasses, TypedDicts.

**Group, don't dump.** Target ~5–12 core entities. For large schemas, identify aggregate roots from *actual references* in routes and services — not from names. Summarize the long tail by category and count.

### Step 4. Map routes, dependencies, and business logic

- **Router inventory**: HTTP method, mounted path/prefix, handler, source. Distinguish *registered* routes from route functions that exist but aren't demonstrably mounted.
- **Dependency graph**: follow `Depends(...)`, `Security(...)`, middleware, decorators — map auth, authz, DB injection, cross-cutting concerns.
- **Handler work location**: inline vs service vs CRUD/repository vs external client vs task queue. **Do not assume a layered architecture exists.**
- Note transaction boundaries and error handling only where they materially affect the traced flow.
- Note background jobs, event handlers, consumers when central to a main workflow.

#### Rank critical modules by evidence

Select ~3–5 files using **multiple signals** — a small orchestration file can outrank a large service file:

1. **Runtime centrality** — app construction, router mounting, lifespan, middleware, shared dependency wiring.
2. **Cross-module reuse** — imports/references from many first-party modules (when measurable).
3. **Business centrality** — modules coordinating important domain operations, called from several key routes.
4. **API surface** — routers exposing main user-facing workflows.
5. **Security and persistence** — auth and DB/session config.
6. **Workflow presence** — modules in the traced request.

For every selected file, state *why* it matters and *what a developer learns*. File size alone is not evidence. If evidence is weak, label the ranking provisional.

### Step 5. Trace one representative request

Mandatory when a real route can be traced. Prefer a common resource create/read or a central operation; choose login only if auth is the best illustration or the user asks.

```
HTTP method + mounted path
  → router handler              (file:line)
  → dependencies / auth / DB    (file:line)
  → service / use case          (file:line)
  → persistence or side effect  (file:line)
  → response schema             (file:line)
```

At each hop: cite file:line; confirm the edge by following imports and actual calls, **not** naming conventions; note branching/validation/transaction/error only when relevant; distinguish direct calls from inferred framework behavior. **Stop at the last verified hop and name the gap** — do not fabricate a chain. If no route can be traced, explain why and show the closest verified partial flow.

---

## Phase C — Verify and explain

### Step 6. Identify risks and open questions

Report only specific, source-grounded observations. Separate three classes:

- **Observed issue** — a concrete pattern visible in source (e.g. duplicated schema/model roles, unclear transaction ownership, mixed sync/async on a hot path, business logic embedded in a router).
- **Potential risk** — plausible concern that depends on context you can't fully verify.
- **Open question** — something a newcomer should ask the team (dynamic registration limiting static analysis, deployment assumptions, missing tests on a critical path — *only if tests were actually inspected*).

Do not label unconventional patterns as bugs. Do not pattern-match into "security findings."

### Step 7. Verify, then write

- [ ] Every cited path exists; the cited symbol/line supports the adjacent claim.
- [ ] The selected app and mounted routers are actually connected.
- [ ] Model relationships match declarations or FKs.
- [ ] Each request-trace hop is verified from source.
- [ ] Generated files and migrations haven't displaced first-party code.
- [ ] No secrets, tokens, or `.env` contents are reproduced.
- [ ] Entity count is grouped, not exhaustive.
- [ ] Inferences and uncertainties are labeled.
- [ ] The report is scoped, prioritized, and useful.

**Completion criteria — you have enough evidence when:**
- Entrypoint, at least one mounted router, and its DB dependency are all verified.
- At least one core request is traceable end-to-end, *or* the gap is explicitly identified.
- At least 3 critical files are supported by multiple ranking signals.

**Stop, disclose, or ask the user to expand scope when:**
- No FastAPI app can be found.
- Tool or context limits prevent verifying the entrypoint or a router connection.
- The repo is too large to scan and the user hasn't specified a target package.

If a claim can't be verified, remove it or mark it `Not verified`. Silence beats fabrication.

---

## Output Template

Omit a subsection only when it genuinely doesn't apply — and say so if omission could confuse.

## Assumptions & Scope
<target package, exclusions, size limits, meaningful limits — 1–2 lines>

## Project Overview
<3–4 sentences: purpose, framework, persistence, validation/settings, auth, sync/async — only as far as verified.>

- **Entrypoint:** `path:line` — direct vs factory; router/lifecycle role.
- **Configuration:** `path:line` — settings source, notable key names only.
- **Database/session:** `path:line` — engine/session dependency and verified lifecycle.
- **Router map:** main mounted prefixes + source files.

## Core Domain Entities
| Entity | Persistence model | Key contracts | Relationships / role |
|---|---|---|---|
| User | `models/user.py:8` | `UserCreate`, `UserRead` | 1-N → Order (verified) |

<State how many additional models were summarized, if counted.>

## Representative Request Flow
`POST /api/v1/items` → `routers/items.py:41` → `get_current_user` / `get_db`
→ `services/items.py:22` → `Item` (models/item.py:8) → `ItemRead`

<Notes on transaction ownership, validation, serialization. Mark any incomplete link.>

## Critical Files to Read First
| # | File | Why it matters | Read it to understand… |
|---|---|---|---|
| 1 | `app/main.py` | App construction, router mounting | Boot sequence |

<3–5 files. Explain the ranking; don't claim metrics that weren't measured.>

## Recommended Reading Order
| Step | File(s) | Focus | ~Time |
|---|---|---|---|
| 1 | `main.py`, `core/config.py` | Startup, wiring | 5–10 min |
| 2 | `db/session.py`, `models/` | Data core | 15–20 min |
| 3 | auth/security module | Dependency chain | 10 min |
| 4 | top service modules | Business rules | 15–20 min |
| 5 | primary routers | Entry surface | 10–15 min |

## Gotchas & Open Questions
- **Observed:** …
- **Potential risk:** …
- **Ask the team:** …

<If nothing stands out, say the scan didn't reveal a clear issue — don't imply a full audit.>

Target ~900 words for a normal project, excluding compact tables. End with the most useful next investigation if a meaningful one remains.

## Anti-Patterns

- Listing every model, route, or endpoint instead of the domain entities and main flows.
- Quoting large code blocks — cite `file:line` instead.
- Inferring a relationship or version from a name.
- Treating `get_db` (or any named dependency) as doing what its name suggests without reading it.
- Starting the reading order with a router.
- Claiming completeness when analysis was scoped or tools were limited.
- Turning this into a security audit, code review, or refactor plan unless asked.
- Reporting "Not found" that is really "Not searched for."

