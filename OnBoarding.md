---
name: codebase-onboarding
description: Analyze an unfamiliar application codebase in any mainstream language and produce an evidence-backed onboarding brief covering entrypoints, detected stack, core domain entities, one representative request/data-flow trace, evidence-ranked critical modules, and a prioritized reading order. Use this skill whenever the user asks how a codebase works, where to start reading a repo, wants an architecture overview or audit, is onboarding onto a new project, or asks to "explain this repo", "map this codebase", or "what should I read first" — even if they don't say "onboarding". Works across Python, JS/TS, Ruby, Java/Kotlin, Go, Rust, C#, PHP and others; uses an ecosystem-specific adapter skill (e.g. fastapi-onboarding-adapter) when one is available and degrades gracefully when not.
---

# Codebase Onboarding & Architecture Analyzer (framework-agnostic)

## Mission
Help a competent engineer understand an unfamiliar codebase and know what to read first. Produce a concise, verified map, not an exhaustive inventory. Analyze what exists; do not assume a directory layout, framework version, or layers such as `services/`.

Three phases: **Discover** → **Reconstruct** → **Verify and explain**.

## Operating Rules
1. Inspect before asserting. Every path, symbol, and flow edge must come from source you opened.
2. Cite `path/file.ext:LINE`. If line numbers are unavailable, cite path + symbol and say so.
3. Separate facts from inferences. "Not found" is not "absent": dynamic registration hides things.
4. Adapt to available tools (Glob / Grep / Read / shell / IDE). If one is unavailable, use the closest method and disclose limits. Signal patterns below are typical, not required.
5. Never print secrets, tokens, keys, or `.env` contents. Key names and loading files are fine.
6. Do not modify files, install dependencies, or execute code unless explicitly asked and permitted.
7. Be proportionate: a small app gets a small report; a monorepo gets a scoped one.
8. Do not force a stack conclusion. If the language or framework is unclear, say so and stop framework-specific analysis.

## Exclusions
Default-exclude:
- VCS (`.git`)
- package caches and virtualenvs (`.venv/`, `venv/`, `env/`)
- dependency trees (`node_modules/`)
- build output (`build/`, `dist/`, `target/`, `bin/`, `obj/`, `out/`, `_build/`, `deps/`, `vendor/`)
- generated code (`*.pyi`, `*_pb2.*`, `*.g.cs`, `*_generated.*`)
- migration history, unless relevant
- `tests/`, unless requested or needed to verify a flow

Note meaningful exclusions in the scope line.

## Step 0: Detect the ecosystem
Read manifests before anything else; they determine which signals to use.

| Manifest present | Ecosystem |
|---|---|
| `pyproject.toml`, `setup.py`, `requirements*.txt`, `Pipfile` | Python |
| `package.json`, `tsconfig.json` | JavaScript / TypeScript |
| `Gemfile`, `*.gemspec` | Ruby |
| `pom.xml`, `build.gradle(.kts)` | Java / Kotlin |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `*.csproj`, `*.sln` | C# / .NET |
| `composer.json` | PHP |
| `Package.swift`, `*.xcodeproj` | Swift |
| `mix.exs` | Elixir |
| `build.sbt` | Scala |
| `pubspec.yaml` | Dart |
| `CMakeLists.txt`, `meson.build`, `vcpkg.json` | C / C++ |
| `deps.edn`, `project.clj` | Clojure |

If several manifests coexist, this is a monorepo or mixed-language repo: pick the package containing the target entrypoint and state the scope. If none match, ask for the target package or fall back to the README and top-level directories.

## Step 0.5: Load an adapter if one is available
If an ecosystem-specific adapter skill is available (e.g. `fastapi-onboarding-adapter`, `rails-onboarding-adapter`, `spring-boot-onboarding-adapter`), use its signal tables, trace shape, and version idioms to inform Steps 1-5. If none exists, proceed in **degraded mode** and say so in the scope line.

### Generic fallback signals (degraded mode)
- **Entrypoint**: the file the runtime actually launches. Look for `main()` declarations, framework bootstrap (`application.*`, `main.*`, `Program.cs`, `cmd/*/main.go`), and container/deployment config (`Dockerfile`, `Procfile`, `docker-compose.yml`, systemd unit, k8s manifest). Read it in full.
- **Routing / handler registration**: once the framework is known, grep its registration idiom (`app.get(`, `@router.`, `@RequestMapping`, `app.use(`, `Router(`, `#route`, `#[get(`, `[HttpGet]`, `http.HandleFunc`). If registration is dynamic, note the indirection instead of guessing.
- **Domain entities**: the persistence model declaration idiom and the DTO/validation idiom of the ecosystem. If there is no dominant ORM, use migration files or SQL as evidence of entity shape.
- **Request-scoped resources**: the mechanism that supplies DB session, auth, or request context (`Depends`, DI container, `before_action`, middleware, filters, `layer`). **Read the implementation**; never assume the name describes the lifecycle.
- **Version-sensitive idioms**: record which era of a library is in use from imports, not directory names.

## Phase A: Discover

### 1. Find the entrypoint and boot sequence
Identify the runtime launch target; config, mounts, middleware, and lifecycle; and list registered handlers/routes. Read `main` / `app` / `index` / bootstrap in full. If multiple apps exist and the target isn't obvious, scope to the main backend and state the assumption.

### 2. Detect the real stack (from imports and usage)
Determine: HTTP framework or application type (service / CLI / library / worker / job); persistence; validation/serialization; auth; lifecycle; sync vs async. Use dependency metadata for versions; otherwise mark uncertain.

## Phase B: Reconstruct

### 3. Map domain entities and contracts
Cover persistence models (identity, keys, relationships **from declarations**, purpose grounded in fields/usage), contracts/DTOs (shapes, inheritance, validation), and domain enums. **Group, don't dump.** Target ~5-12 core entities; for large schemas, name aggregate roots from real references and summarize the rest by category and count.

### 4. Map request entry, dependencies, and business logic
Inventory registered handlers/routes. Build the dependency graph via the framework's wiring mechanism. Establish where work happens: inline, service, repository, external client, or queue. **Do not assume layers.** Note transaction boundaries and error handling only where they affect the traced flow.

**Rank critical modules by multiple signals:** runtime centrality (bootstrap/mounting/lifecycle/wiring), cross-module reuse, business centrality, API surface, security/persistence, workflow presence. A small orchestrator can outrank a large service file. If evidence is weak, label the ranking provisional.

### 5. Trace one representative flow
Prefer a common create/read or a central operation; choose login only if auth is the best illustration.

```
Entry (HTTP method + path / CLI command / job trigger / consumer topic)
  → handler / controller / resolver      (file:line)
  → middleware / filter / interceptor    (file:line)
  → dependencies / auth / session        (file:line)
  → service / use case                   (file:line)
  → persistence or side effect           (file:line)
  → response / serialization             (file:line)
```

At each hop, cite `file:line` and confirm the edge by following the actual call or registration, **not the name**. **Stop at the last verified hop and name the gap.** Never fabricate a chain. If no flow can be traced, say why and show the closest verified partial chain.

## Phase C: Verify and explain

### 6. Risks and open questions (three classes)
- **Observed**: a concrete pattern in source (duplicated roles, unclear transaction ownership, blocking calls on an async hot path, business logic in handlers).
- **Potential risk**: depends on context you can't verify.
- **Ask the team**: dynamic registration, deployment assumptions, missing tests on a critical path (only if tests were inspected).

Do not label unconventional patterns as bugs. Do not pattern-match into security findings.

### 7. Verify, then write
- [ ] Every cited path exists; the symbol/line supports the claim.
- [ ] Selected app and mounted routes are actually connected.
- [ ] Relationships match declarations or FK/migration evidence.
- [ ] Each flow hop is verified from source.
- [ ] Generated code and migration history haven't displaced first-party code.
- [ ] No secrets reproduced.
- [ ] Entity count is grouped, not exhaustive.
- [ ] Inferences and uncertainties are labeled.
- [ ] Report is scoped and prioritized.

**Completion criteria.** You have enough evidence when: the entrypoint is verified; at least one registered route/flow and its request-scoped resource are verified; and at least 3 critical files are supported by multiple signals.

**Stop, disclose, or ask the user when:** no entrypoint is found; tool/context limits block verification of the entrypoint or route connection; the repo is too large and no target is specified; or the language/framework is unclear and no adapter is available.

If a claim can't be verified, remove it or mark it `Not verified`.

---

## Output Template

```
## Assumptions & Scope
<target package, exclusions, mode (full / degraded / scoped), meaningful limits>

## Project Overview
<3-4 sentences: purpose, language, framework, persistence, validation, auth, sync/async. Only as verified.>
- Entrypoint: path:line
- Configuration: path:line (key names only)
- Runtime wiring / DI / session: path:line
- Route / handler map: main groups + source files

## Core Domain Entities
| Entity | Persistence model | Key contracts | Relationships / role |

## Representative Flow
<arrow chain with file:line per hop; mark incomplete links>

## Critical Files to Read First
| # | File | Why it matters | Read to understand… |

## Recommended Reading Order
| Step | File(s) | Focus | ~Time |

## Gotchas & Open Questions
- Observed: …
- Potential risk: …
- Ask the team: …
```

Target ~900 words excluding tables.

## Anti-Patterns
- Listing every model, route, or endpoint instead of domain entities and main flows.
- Quoting large code blocks; cite `file:line` instead.
- Inferring a relationship, version, or purpose from a name.
- Treating a request-scoped dependency (`get_db`, `@Autowired`, `before_action`) as doing what its name suggests without reading it.
- Starting the reading order with a router/controller.
- Claiming completeness when scoped or tool-limited.
- Turning this into a security audit, code review, or refactor plan unless asked.
- Reporting "Not found" that is really "Not searched for."
- **Running in degraded mode without saying so.**

---

## Appendix: Adapter contract (for authors of ecosystem adapters)
An adapter is a small, separate skill that supplies only what this general skill cannot reason about. It must not duplicate the protocol, rules, exclusions, completion criteria, anti-patterns, or output template above.

```markdown
---
name: <framework>-onboarding-adapter
description: Framework-specific signals for codebase-onboarding when the detected stack is <framework>.
parent: codebase-onboarding
---

# <Framework> Adapter

## When this adapter applies
<manifest + import signals>

## Signal table
| Concern | Typical patterns | Notes |

## Request / flow trace shape
<entry → ... → response, specific to the framework>

## Version-sensitive idioms
| Signal | Version |

## Common pitfalls
<framework-specific inspection reminders, e.g. session/transaction lifecycle>

## Output deltas
<only if the framework warrants a different table or section>
```

Examples of trace shapes an adapter encodes (not interchangeable):
- Rails: `routes.rb → before_action → controller → model`
- Spring: `@RequestMapping → filter chain → @ControllerAdvice → @Service → @Repository`
- Axum: `Router → layer → handler → extractor → State`
- FastAPI: `router → Depends → service → ORM → schema`
