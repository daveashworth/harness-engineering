---
name: harness-engineering
description: "Set up and maintain agent-first repositories using harness engineering principles. Use when bootstrapping a new repo for agent-driven development, adding AGENTS.md / ARCHITECTURE.md / docs structure, enforcing architectural layers, or migrating an existing repo to agent-first workflows."
version: "1.1.0"
---

# Harness Engineering

Set up and work in repositories optimized for agent-first software development. Based on OpenAI's harness engineering approach: humans steer, agents execute. The harness is the repository-resident control system around agents: maps, tools, constraints, runtime evidence, review loops, and recurring cleanup.

## Core Principles

1. **The repo is the system of record.** Anything not in the repo is invisible to agents. Push all context—decisions, specs, architecture, conventions—into versioned files.
2. **AGENTS.md is a map, not a manual.** Keep it ~100 lines. It tells agents where to find deeper truth, not everything itself.
3. **Enforce invariants, not implementations.** Define strict boundaries and let agents have freedom within them.
4. **Failures reveal missing harness.** When agents repeatedly fail, do not ask them to "try harder." Add the missing doc, command, lint, test, tool, template, or workflow so the fix compounds.
5. **Make the application legible to agents.** UI state, logs, metrics, traces, screenshots, spans, and runtime health must be directly observable from isolated task environments.
6. **Agent legibility over human aesthetics.** Optimize for what agents can reason about. Prefer boring technology that is composable, stable, and well-represented in training data.
7. **Fast feedback loops.** Tests, lints, CI, and local runtime checks must run fast enough to be used repeatedly in a single task.
8. **Progressive disclosure.** Agents start with a small, stable entry point and are taught where to look next.

## Failure-to-Harness Loop

Whenever an agent gets stuck, produces the same defect twice, or needs repeated human correction, convert the lesson into the harness:

1. Name the failure mode in plain language.
2. Decide the cheapest durable control: documentation, structural lint, test fixture, generated reference, runtime probe, script, PR checklist, or skill.
3. Put that control in the repo, not in a private prompt or chat memory.
4. Add remediation guidance to failures. A lint error should say how to fix the problem.
5. Verify the control catches or prevents the observed failure.

The goal is not more instructions. The goal is fewer repeated human interventions.

## Bootstrapping a New Repo

When asked to set up a new agent-first repo, follow these steps in order:

### Step 1: Create AGENTS.md (~100 lines)

This is the table of contents. Structure it like:

```markdown
# AGENTS.md

## Start here
- Architecture overview: ARCHITECTURE.md
- Plans and execution plans: docs/PLANS.md
- Design system: docs/DESIGN.md
- Security requirements: docs/SECURITY.md
- Reliability expectations: docs/RELIABILITY.md
- Product specs index: docs/product-specs/index.md
- Active execution plans: docs/exec-plans/active/

## Workflows
- Run tests: <command>
- Lint/format: <command>
- Start dev server: <command>

## Key conventions
- Parse at boundaries: all external input validated into typed structures immediately
- Structured logging only (no console.log)
- File size guideline: aim for < 500 lines per file; when a file grows near this, pause and consider whether parts should become modules, components, helpers, or tests instead of continuing to pile content into one file
- Every PR must include evidence of behavior (test output, screenshots, traces)
```

For monorepos or large subprojects, add nested `AGENTS.md` files only where local guidance differs. Root guidance should cover repo-wide purpose, commands, and navigation; nested guidance should cover package-specific stack, constraints, and tests. Do not duplicate global rules at every level.

### Step 2: Create ARCHITECTURE.md

A stable, high-level codemap of the codebase. It should answer "where is the thing that does X?" and "what does this area do?" without implementation details.

```markdown
# Architecture

Brief description of what this system does.

## Domain map
- <domain>/
  - types/: domain types (branded types, schemas)
  - config/: configuration and environment
  - repo/: database access and queries
  - service/: business logic and workflows
  - runtime/: schedulers and background jobs
  - ui/: user-facing components

Cross-cutting concerns live in platform/ (auth, telemetry, feature flags).

## Dependency direction
types -> config -> repo -> service -> runtime -> ui

UI cannot import from repo. Services import from repo and config, never from UI.

## Cross-cutting concerns
- Auth: <where policy is enforced>
- Telemetry: <required logs, metrics, traces>
- Feature flags: <where flags are defined and evaluated>
```

Keep `ARCHITECTURE.md` short and stable. Name important modules, types, and boundaries, but avoid brittle deep links when symbol search is better. Explicitly document invariants and important absences, such as "models never depend on views" or "UI never imports database repositories."

### Step 3: Create the docs/ knowledge base

```
docs/
  DESIGN.md           # Design system and patterns
  FRONTEND.md         # Frontend conventions
  SECURITY.md         # Security requirements
  RELIABILITY.md      # Reliability expectations
  QUALITY_SCORE.md    # Quality grades per domain/layer
  PLANS.md            # How plans work, linking to exec-plans/
  PRODUCT_SENSE.md    # Product principles and user context
  design-docs/
    index.md          # Catalog of design docs with verification status
    core-beliefs.md   # Agent-first operating principles
  product-specs/
    index.md          # Index of product specs
  exec-plans/
    active/           # In-progress execution plans
    completed/        # Done plans (kept for history)
    tech-debt-tracker.md
  references/         # LLM-friendly docs for key dependencies
  generated/          # Auto-generated docs (e.g., db-schema.md)
```

Each index file should list available docs and when to read them.

### Step 4: Make the app legible and runnable per task

Set up a local task environment an agent can start, inspect, and tear down without colliding with other work:

- **Ephemeral worktrees:** one task per worktree.
- **Isolated runtime resources:** unique ports, databases, caches, queues, storage buckets, and temp dirs per worktree.
- **Single start command:** one documented command boots the app and required local services.
- **Browser/UI access:** for UI products, expose DOM snapshots, screenshots, navigation, and interaction tooling.
- **Observability access:** expose local logs, metrics, traces, and spans in queryable form.
- **Runtime acceptance:** define measurable checks such as startup under 800ms, no critical journey span over 2s, or a health endpoint returning a specific payload.
- **Cleanup:** provide a documented teardown command that removes task-local resources.

Agents should be able to reproduce bugs, validate fixes, and gather evidence without a human manually clicking through the app or copying logs.

### Step 5: Set up architectural enforcement

Create lint rules or structural tests that enforce:

1. **Layer dependency direction:** `types -> config -> repo -> service -> runtime -> ui`
2. **Parse at boundaries:** external input must be parsed into typed structures immediately
3. **Semantic domain types:** prefer `InvoiceId`, `WorkspaceSlug`, `UserId`, or language-appropriate equivalents over raw primitives
4. **Taste invariants:**
   - Structured logging only (no `console.log`)
   - Schema naming conventions (e.g., `BillingEventSchema`)
   - File size guidelines (aim for < 500 lines; use this as a prompt to consider extracting modules, components, helpers, or tests, not as an absolute rule)
   - Mandatory request tracing in critical flows

Write custom lint error messages that include remediation instructions—these inject fix guidance into agent context.

### Step 6: Set up fast CI and documentation checks

- Format, lint, typecheck, and unit tests must all run fast
- CI should fail fast with clear, actionable error messages
- No uncovered lines in new or modified files
- Critical workflows require end-to-end tests
- Documentation checks should verify that:
  - links resolve
  - referenced files exist
  - documented commands still run
  - generated docs match source schemas
  - architecture docs match actual module/layer layout
  - stale decisions are marked superseded
  - completed plans move out of active
  - root `AGENTS.md` remains small

### Step 7: Set up agent workflow and review loops

- **PR evidence:** agents must include test output, logs, traces, screenshots in PRs
- **One task per loop:** keep autonomous loops narrow enough to verify
- **Local self-review:** the authoring agent reviews its own diff before handoff
- **Agent-to-agent review:** one agent writes, another reviews with a focused brief
- **Review iteration loop:** the authoring agent responds to review feedback and repeats until deterministic checks and reviewers are satisfied
- **Human review routing:** reserve humans for ambiguity, high-risk changes, security boundaries, persistence, public APIs, and product judgment
- **Minimal merge gates:** fast pass/fail checks + evidence. Prioritize throughput while preserving correctness

## Working in an Existing Harness Repo

When working in a repo that already follows harness engineering:

1. **Read AGENTS.md first.** It's the map. Follow its pointers to find what you need.
2. **Check ARCHITECTURE.md** to understand domain layout and dependency rules.
3. **Read relevant docs/** files before making changes in that domain.
4. **Check active exec-plans/** if there's a plan for the work you're doing.
5. **Follow the layer rules.** Never violate dependency direction.
6. **Parse at boundaries.** All external data gets parsed into typed structures at entry points.
7. **Use the task-local runtime.** Reproduce, inspect, and validate behavior in the isolated environment when the change is user-facing or runtime-sensitive.
8. **Include evidence in PRs.** Test output, screenshots, traces—make reviews objective.
9. **Treat ~500 lines as a design checkpoint, not a hard cap.** When a file approaches it, consider whether modules, components, helpers, or tests would make the work clearer before adding more content.
10. **Use structured logging.** No `console.log`.
11. **If you repeat a correction, improve the harness.** Add the missing rule, doc, test, or tool rather than relying on memory.

## Migrating an Existing Repo

Follow these maturity levels incrementally:

**Level 1 — Navigable:**
Add `AGENTS.md`, `ARCHITECTURE.md`, and `docs/` structure so agents can orient themselves.

**Level 2 — Verifiable:**
Document exact format, lint, typecheck, test, build, and start commands. Add fast CI and doc link/file checks.

**Level 3 — Constrained:**
Define layer rules, ownership, boundary parsing, semantic types, and enforce them mechanically with lints or structural tests.

**Level 4 — Observable:**
Make the app runnable per worktree and expose logs, metrics, traces, screenshots, DOM snapshots, and runtime health checks.

**Level 5 — Autonomous:**
Enable agent PR creation, self-review, agent-to-agent review, recurring cleanup, and small safe merges with evidence.

## Continuous Maintenance ("Garbage Collection")

Schedule recurring background tasks that:
- Update stale documentation
- Enforce architectural rules across the full codebase
- Refactor style drift or repeated patterns into shared utilities
- Refresh quality scores in `docs/QUALITY_SCORE.md`
- Check doc links, referenced files, generated docs, and active/completed plan hygiene
- Convert repeated agent failures into durable harness improvements
- Open targeted refactoring PRs (most reviewable in under a minute)

Think of this as garbage collection: pay down tech debt continuously in small increments rather than painful bursts. Prefer shared utility packages over hand-rolled helpers. Never probe data "YOLO-style"—validate at boundaries or use typed SDKs.

## Key Patterns

### Semantic Domain Types

Use language-appropriate domain types instead of raw primitives when values cross layers or boundaries.

TypeScript example:

```typescript
type InvoiceId = string & { readonly brand: "InvoiceId" }
type WorkspaceSlug = string & { readonly brand: "WorkspaceSlug" }
```

### Boundary Parsing

Parse and validate external data at entry points. Internal services should receive already-validated domain structures.

TypeScript example:

```typescript
// At the HTTP handler boundary
const payload = parseWebhookPayload(req.body)
await processWebhook(payload)
// processWebhook assumes correctness — no validation inside
```

### Execution Plans

For complex work, use execution plans checked into `docs/exec-plans/active/`. An ExecPlan must be self-contained enough for a stateless agent or novice contributor to restart from only that file.

Each ExecPlan should include:

- Purpose / big picture: what user-visible behavior changes and how to see it working
- Context and orientation: relevant files, modules, terms, and current state
- Progress: updated checklist with timestamps at stopping points
- Surprises & discoveries: unexpected findings with evidence
- Decision log: choices and rationale
- Plan of work: concrete edits and sequence
- Concrete steps: exact commands and working directories
- Validation and acceptance: observable behavior, expected outputs, tests, screenshots, traces
- Idempotence and recovery: safe retry, cleanup, rollback guidance
- Interfaces and dependencies: specific APIs, types, services, or libraries involved
- Outcomes & retrospective: what shipped, what remains, what changed in the harness

Move to `docs/exec-plans/completed/` when done.

### Review Loops

Use one task per loop. A good loop is:

1. Agent implements the smallest useful change.
2. Agent runs deterministic checks and gathers evidence.
3. Agent self-reviews its diff.
4. Separate reviewer agent checks the diff against the goal and harness rules.
5. Author agent responds to feedback and repeats until checks and reviewers are satisfied.
6. Human reviews only where judgment or risk warrants it.

### Quality Scores
Grade each domain and layer in `docs/QUALITY_SCORE.md`. Track gaps over time. Use this to prioritize cleanup work.

### Merge Policy by Risk

Fast merging is safe only when failures are cheap: small PRs, narrow blast radius, fast detection, strong observability, feature flags, rollback paths, and cleanup loops.

- Local pure function + tests: fast path
- Documentation update with link checks: fast path
- UI copy/layout change: screenshot evidence
- Public API change: API diff, docs, tests
- Dependency addition: supply-chain review
- Security boundary change: high-trust human approval
- Migration/persistence change: rollback plan
- Deployment path change: explicit human signoff
