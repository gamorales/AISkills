# codebase-onboarding

A skill family for Claude that produces evidence-backed onboarding briefs for
unfamiliar codebases — entrypoints, detected stack, core domain entities, a
representative end-to-end request trace, evidence-ranked critical modules, and
a prioritized reading order.

The brief is designed to answer five questions for a new engineer:

1. What does this backend do, and how does it start?
2. What are its core domain entities, and how does data move between them?
3. How does a real request travel through the system?
4. Which files should I read first, and why?
5. What architectural risks or open questions should I investigate next?

Every claim in the brief is cited to `path/file.ext:LINE`. Inferences are
labeled. Gaps are named, not fabricated.

---

## The problem

Most "onboarding" skills are well-written checklists that describe what a
human *should* do — not protocols a model can *execute*. They fail in
predictable ways on real repositories:

- **Plausible fabrication.** Confident paths, relationships, and version
  numbers that were inferred from filenames.
- **Framework blindness.** A skill written for FastAPI confidently misdescribes
  a Rails app.
- **Noise dominance.** Migration history and vendored code outnumber
  hand-written models 10:1 and dominate the "important files" list.
- **Arbitrary ranking.** "The 3–5 most important scripts" with no criterion.
- **Unverifiable output.** No template, no citation discipline, no self-check.

This family fixes those by separating **protocol** (how to analyze) from
**signals** (what to look for in a given framework).

---

## Architecture

Three layers. The protocol is shared; the signals are per-ecosystem.
```
┌───────────────────────────────────────────────────────────┐
│  Layer 1 · codebase-onboarding                            │
│  protocol · phases · verification · output · ranking      │
│  (language-agnostic)                                      │
└──────────────────────┬────────────────────────────────────┘
                       │ detects ecosystem from manifests,
                       │ loads a matching adapter if installed
                       ▼
┌───────────────────────────────────────────────────────────┐
│  Layer 2 · adapters                                       │
│  fastapi-onboarding-adapter                               │
│  rails-onboarding-adapter                                 │
│  (add spring-boot, django, nestjs, … the same way)        │
└──────────────────────┬────────────────────────────────────┘
                       │ applies adapter signals for
                       │ Steps 1–5 of the protocol
                       ▼
┌───────────────────────────────────────────────────────────┐
│  Layer 3 · the report                                     │
│  one output template for every ecosystem                  │
└───────────────────────────────────────────────────────────┘
```

**Why split.** Roughly 65% of a good onboarding skill is framework-independent:
evidence discipline, verification, ranking, output shape. The remaining 35% —
signal tables, request-trace shapes, version idioms, pitfalls — is exactly what
makes the output reliable, and it does not transfer between frameworks. A
single "any language" skill is possible but degrades to the principled-checklist
failure mode above. A layered family keeps both properties.

---

## Skills in this repository

| Skill | Layer | Purpose | Status |
|---|---|---|---|
| [`codebase-onboarding`](./codebase-onboarding/SKILL.md) | 1 | Framework-agnostic protocol, verification, output template | ✅ Stable |
| [`fastapi-onboarding-adapter`](./fastapi-onboarding-adapter/SKILL.md) | 2 | FastAPI / SQLAlchemy / SQLModel / Pydantic signals | ✅ Stable |

---

## Installation

Clone into your Claude skills directory:

```bash
git clone git@github.com:gamorales/AISkills.git ~/.claude/skills/codebase-onboarding
```

Or copy the individual skill directories. Final layout:

```
~/.claude/skills/
├── codebase-onboarding/
│   └── SKILL.md
├── fastapi-onboarding-adapter/
│   └── SKILL.md
```

The skills are discovered independently. The general skill finds a
matching adapter at runtime by description — you don't have to invoke them in
order.

> **Platform note.** Sibling-skill discovery works in Claude Code and the
> Skills SDK, which scan the filesystem. On platforms that don't discover
> adjacent skills, the general skill still runs — it just runs in **degraded
> mode** (see *Runtime behavior* below).

---

## Usage

Invoke the general skill by describing the task. It detects the stack and
loads the right adapter automatically.

```
> Onboard me onto this FastAPI repo. I've never seen it before.

> Audit this backend's architecture and give me a reading order.
```

There is no need to invoke an adapter directly. Adapters are not standalone
onboarding skills — they supply signals to the general protocol.

---

## Runtime behavior

| Situation | What happens |
|---|---|
| FastAPI repo, adapter installed | General skill + FastAPI signals; standard template |
| Rails repo, adapter installed | General skill + Rails signals; standard template |
| Django repo, no adapter | General skill runs **degraded mode**, states so in the scope line |
| Monorepo, FastAPI service + React frontend | Step 0 scopes to the service package; FastAPI adapter applies within scope |
| Unrecognized language / mixed manifests | Ask for the target package, or fall back to README + top-level dirs |
| Tool or context limits block verification | Disclose the limitation; never claim completeness |

The scope line at the top of every report names the mode (`full` / `degraded`
/ `scoped`) and the adapter used.

---

## What the protocol guarantees

Every report produced by this family:

- **Cites.** `path/file.ext:LINE` for every important claim. Line numbers
  verified by re-reading, not inferred from structure.
- **Separates facts from inferences.** Deductions labeled `Inferred`;
  unknowns labeled `Not found` or `Not verified`.
- **Traces one real flow end to end.** `entry → handler → middleware →
  dependencies → service → persistence → response`, each hop verified from
  source. Fabricated hops are forbidden; a broken chain is reported as a gap.
- **Ranks by evidence.** Critical files selected by runtime centrality,
  cross-module reuse, business centrality, API surface, security/persistence,
  and workflow presence — never file size alone.
- **Groups, doesn't dump.** 5–12 core domain entities; the long tail is
  summarized by category and count.
- **Protects secrets.** No `.env` contents, credentials, tokens, keys, or
  connection strings, ever. Configuration *key names* and loading files are
  allowed.
- **Self-checks before writing.** A verification checklist runs before the
  report is emitted; unverifiable claims are removed or labeled.
- **Knows when to stop.** Completion criteria define when evidence is
  sufficient; the skill discloses, stops, or asks the user to expand scope
  when it isn't.

The full protocol, operating rules, and output template live in
[`codebase-onboarding/SKILL.md`](./codebase-onboarding/SKILL.md).

---

## Design decisions

A few choices that are deliberate and worth knowing before you modify these
skills.

**Signals, not recipes.** The general skill says "grep the framework's
registration idiom (`app.get(`, `@router.`, `@RequestMapping`, `#route`, …)"
rather than hard-coding one. Adapters supply the specific patterns. This keeps
the protocol portable and still gives Claude concrete anchors.

**Degraded mode is a first-class outcome.** Running with no adapter is
allowed. Running with no adapter *silently* is not. This is enforced by an
anti-pattern ("Running in degraded mode without saying so") and by the scope
line in the output template.

**The request trace is mandatory.** Everything else in the report can be
provisional; the trace cannot. It is the single highest-value artifact for a
new engineer, and it is the part most likely to expose a skill that is
reasoning from conventions rather than reading source.

**Adapters are cheap.** Each adapter is 40–80 lines. Adding Django or Spring
Boot costs an hour and touches zero lines in the general skill.

**No hidden orchestration.** Adapters are ordinary skills. There is no
loader, no registry, no config file. Runtime dispatch is by description match,
which is how skills already work.

---

## Verification status

Honest accounting of what has been validated and what hasn't.

| Claim | Status | Notes |
|---|---|---|
| General skill protocol is framework-independent | ✅ By inspection | No framework names in Steps 1–5 except as examples |
| FastAPI adapter is a faithful extraction of the merged skill | ✅ By construction | Signal table, `table=True` split, `get_db` reminder, trace shape, version idioms preserved |
| Rails adapter signals are correct | ⚠️ **Untested in a real repo** | Written from domain knowledge; smoke-test against a known project (GitLab, Discourse, Forem) before relying on it |
| Sibling-skill discovery works on the target platform | ⚠️ **Platform-dependent** | Works where skills are scanned from the filesystem; otherwise degraded mode |
| Output beats a single mega-skill | ⚠️ **Hypothesis** | The theoretical case is that concrete signals drive execution. A 10-repo A/B would settle it |

The README will be updated as these are validated.

---

## Limitations

- **Not a security audit.** The skill deliberately avoids security findings
  from superficial pattern matching. It surfaces observed issues, potential
  risks, and questions for the team — nothing more.
- **Not a code review or refactor plan** unless the user asks.
- **Not a completeness guarantee.** Scoped analysis (large repos, limited
  tools) is disclosed, never disguised.
- **Not a replacement for `bin/rails routes` / `python -c "import app"` / a
  running service.** The skill reads source statically. When a runtime
  command would be more authoritative, it says so.
- **Not immune to misleading structure.** A directory named `services/` that
  contains no services will be reported as such, not as a services layer.

---

## Roadmap

1. **Smoke-test the Rails adapter** against a real repo. This is the
   strongest proof-of-concept for the layered design — if the signal table
   survives contact, the architecture is validated.
2. **Add a third contrasting adapter.** Spring Boot is the best choice: its
   DI / filter-chain / `@ControllerAdvice` shape is a third distinct pattern.
   Three adapters that share only the protocol is the strongest argument for
   the split.
3. **Consider a generator.** Once three adapters exist, a template that emits
   a starter adapter from a framework name and a signal list is worth building.
   Premature before then — hand-writing is faster than debugging a generator
   with no examples to test against.
4. **Measure.** A small A/B (layered family vs. single mega-skill, N repos)
   would replace the current "hypothesis" status with data.

---

## Adding a new adapter

Create `~//.claude/skills/<framework>-onboarding-adapter/SKILL.md` with:

```yaml
---
name: <framework>-onboarding-adapter
description: Framework-specific signals for codebase-onboarding when the
  detected stack is <framework>. Supplies detection signals, request-trace
  shape, version idioms, and pitfalls. Used by the codebase-onboarding
  skill — not a standalone onboarding skill.
parent: codebase-onboarding
---
```

Then supply exactly these sections, and nothing else:

1. **Detection signals** — table of framework-specific idioms
2. **Request trace shape** — ASCII arrow chain for one representative flow
3. **Framework-specific pitfalls** — the things newcomers stumble on
4. **Ranking emphasis** — which files matter most in this framework, and why
5. **Version idioms** — how to tell framework/library versions apart from source

Do **not** restate the protocol, operating rules, verification, or output
template. Those live in `codebase-onboarding` and must stay consistent across
adapters.

Start from `fastapi-onboarding-adapter` or `rails-onboarding-adapter` as a
template.

---

## License

GNU GENERAL PUBLIC LICENSE　
```

---

**Notes on the README itself:**

- **The Verification status section is the point.** Every claim about a skill in this space tends to be stronger than reality; keeping the ⚠️ rows visible is what makes the README trustworthy rather than a marketing page. Update them as you smoke-test — that's how the README earns its status column.
- **Adapter contract is written down.** A contributor who's never seen this conversation can build a Spring Boot adapter by following *Adding a new adapter* — that was the whole reason for the layered split, and the README should make that promise executable.
- **No invocation examples that won't work.** The *Usage* section shows prompts a user would actually type, not fictional CLI commands. If your platform surfaces skills differently, adjust that one block.
- **The License placeholder is deliberate.** Pick your license; nothing else in the file depends on which one.

Want a `CONTRIBUTING.md` that expands the adapter contract into a step-by-step guide with a filled-in example (say, a hypothetical Spring Boot skeleton), or a `docs/architecture.md` that goes deeper on the protocol-vs-signals split?
