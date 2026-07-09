# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Paperclip** is an open-source orchestration platform for managing teams of AI agents. It's a full control plane that handles org charts, budgeting, governance, task management, heartbeat execution, and cost tracking across multiple agent types (Claude Code, Codex, Cursor, OpenClaw, HTTP agents, etc.).

The server is a Node.js Express API with an embedded PostgreSQL database. The UI is a React + Vite board application. Everything is managed as a pnpm monorepo with TypeScript throughout.

## Quick Start

```bash
pnpm install
pnpm dev                          # Full dev: API + UI (watch mode)
pnpm dev:once                     # Single run, auto-applies migrations
pnpm dev:server                   # Server only
pnpm dev:ui                       # UI only
```

This starts:
- API server: `http://localhost:3100`
- UI: served by the API server in dev middleware mode
- Embedded PostgreSQL: automatically created in `~/.paperclip/instances/default/db/`

Reset local dev data: `rm -rf ~/.paperclip/instances/default/db/`

## Repository Structure

**Top-level directories:**
- `server/` — Express REST API, orchestration services, heartbeat execution
- `ui/` — React + Vite board UI, Storybook stories under `ui/storybook/`
- `packages/db/` — Drizzle ORM schema, migrations, database layer
- `packages/shared/` — shared types, constants, validators, API path constants
- `packages/adapters/` — agent adapter implementations (Claude, Codex, Cursor, Gemini, Grok, OpenClaw, etc.)
- `packages/adapter-utils/` — shared adapter utilities and base classes
- `packages/plugins/` — plugin system core and plugin implementations
- `packages/skills-catalog/` — shipped skills catalog
- `packages/teams-catalog/` — example agent team configurations
- `packages/mcp-server/` — MCP (Model Context Protocol) server implementation
- `cli/` — `paperclipai` CLI tool for onboarding, auth, configuration
- `doc/` — comprehensive product and technical documentation
- `scripts/` — utility scripts for dev, release, migration, testing
- `tests/` — E2E (Playwright) and smoke tests
- `skills/` — per-skill implementation folders (e.g. `skills/paperclip/` for Paperclip integration)

**Key docs to read first:**
1. `doc/GOAL.md` — mission and success criteria
2. `doc/PRODUCT.md` — feature overview
3. `doc/SPEC-implementation.md` — V1 build contract
4. `doc/DEVELOPING.md` — detailed dev guide
5. `doc/DATABASE.md` — database setup and migrations
6. `AGENTS.md` — contributor expectations and rules
7. `DESIGN.md` — design system and token layer governance

## Common Commands

### Development

```bash
pnpm dev                          # Full dev (API + UI, watch)
pnpm dev:once                     # Single run without watch
pnpm dev:watch                    # Server watch mode (alias for dev)
pnpm dev:list                     # List running dev servers
pnpm dev:stop                     # Stop the current dev server
pnpm dev --bind lan               # Authenticated/private dev mode
pnpm storybook                    # UI Storybook on port 6006
```

### Testing

```bash
pnpm test                         # Vitest suite (default, fast)
pnpm test:watch                   # Vitest watch mode
pnpm test:run                     # Full stable test run
pnpm test:e2e                     # Playwright browser tests
pnpm test:e2e:headed             # E2E with visible browser
pnpm test:release-smoke          # Release verification tests
pnpm test:storybook-visual       # Visual regression checks
```

Do NOT run browser suites by default—use `pnpm test` (Vitest only) for normal development. Start with the smallest check that proves your change. Reserve repo-wide `pnpm -r typecheck`, `pnpm test:run`, and `pnpm build` for PR-ready handoff or when the change scope is broad.

### Building & Type Checking

```bash
pnpm typecheck                    # TypeScript check (all packages)
pnpm typecheck:build-gaps        # Check build vs typecheck gaps
pnpm build                        # Build all packages
pnpm build:npm                    # Build for npm release
```

### Database

```bash
pnpm db:generate                 # Generate migration from schema changes
pnpm db:migrate                  # Apply pending migrations
pnpm db:backup                   # Backup current database
pnpm issue-references:backfill   # Backfill issue reference mentions
```

### Release & Deployment

```bash
pnpm release:canary              # Release canary version to npm
pnpm release:stable              # Release stable version to npm
pnpm release:github              # Create GitHub release
```

## Architecture Essentials

### Control Plane Subsystems

Paperclip is organized around these interlocking systems (see README under "What's Under the Hood"):

1. **Identity & Access** — deployment modes (trusted local, authenticated), board users, agent API keys, company memberships
2. **Org Chart & Agents** — agent roles, titles, reporting lines, permissions, budgets; adapter framework
3. **Work & Task System** — issues/tasks with company/project/goal links, atomic checkout, dependencies, comments, attachments
4. **Heartbeat Execution** — scheduled wakeup queue with coalescing, budget enforcement, workspace resolution, adapter invocation
5. **Workspaces & Runtime** — project workspaces, git worktrees (execution isolation), dev servers, preview URLs
6. **Governance & Approvals** — approval workflows, execution policies, audit logging
7. **Budget & Cost Control** — token/cost tracking by company/agent/project/issue/provider/model; scoped policies with hard stops
8. **Routines & Schedules** — recurring tasks with cron/webhook/API triggers
9. **Plugins** — extensible plugin system with out-of-process workers and capability gating
10. **Secrets & Storage** — encrypted local storage, provider-backed object storage, attachments
11. **Activity & Events** — immutable audit trail of mutations and state changes
12. **Company Portability** — export/import orgs with secret scrubbing and collision handling

### Core API Patterns

- **Base path:** `/api`
- **Authorization:** bearer token for agents (`Authorization: Bearer agent_<key>`), session for board users
- **Company scoping:** every endpoint must enforce company access checks; no cross-company data leakage
- **Atomic task checkout:** checkout is single-assignee and atomic; the database row lock prevents double-work
- **Budget enforcement:** hard stops when spent >= budget; agents are paused and runs are cancelled

**Key route patterns:**
- `/api/companies/` — company management, multi-tenancy
- `/api/agents/` — agent registration, lifecycle
- `/api/issues/` — task/issue CRUD and state management
- `/api/runs/` — heartbeat execution logs and results
- `/api/approvals/` — governance workflows
- `/api/adapters/` — agent adapter discovery and configuration

### Database Layer (Drizzle)

- Schema is in `packages/db/src/schema/*.ts` (split by entity type)
- Migrations are auto-generated: `pnpm db:generate`
- Embedded PostgreSQL in dev; point to external Postgres in prod
- All entities are company-scoped (critical invariant)
- Schema is compiled to `dist/schema/*.js` before migration generation

### Adapter Framework

Adapters implement the `AgentAdapter` interface and handle runtime execution for different agent types:
- **Local adapters** (e.g., `adapter-claude-local`) — run agents locally as processes
- **Cloud adapters** (e.g., `adapter-cursor-cloud`) — invoke cloud agents via APIs
- Each adapter is a workspace package under `packages/adapters/`

Adapters receive structured task context (company, project, goal ancestry, budget, secrets) and return structured logs and cost events.

### UI Architecture

- React 19 + Vite build
- Tailwind v4 with CSS custom properties (no Tailwind config file)
- shadcn UI components via Radix UI
- Storybook for component review (stories under `ui/storybook/stories/`)
- Design tokens defined in `ui/src/index.css` (semantic, brand, and domain tiers)

## Design System & Tokens

**Governance rule:** All visual values (color, spacing, radius, type, shadow, motion) must come from the token layer at `ui/src/index.css`. No hardcoded hex, raw `px`, ad-hoc Tailwind arbitrary values (`p-[13px]`), or palette classes (`bg-red-500`) in components—extract them as tokens first.

**Token tiers already in place:**
- **Semantic:** `--background`, `--foreground`, `--card`, `--primary`, `--secondary`, `--muted`, `--accent`, `--destructive`, `--border`, `--input`, `--ring`, `--sidebar-*`, `--chart-1..5`
- **Brand:** agent gradients `--agent-1a/1b..10a/10b`, status hues `--status-task-*` / `--status-agent-*`
- **Domain:** match chips, doc annotations, motion, typography

When adding new UI values, add a token to `ui/src/index.css` and use it via CSS variables or Tailwind `@theme`.

**Storybook:** stories live in `ui/storybook/stories/`; run `pnpm storybook` to review components.

## Key Conventions

### Company Scoping (Critical)

Every entity is company-scoped. Routes, services, and database queries must enforce company boundaries:
```typescript
// ✅ Correct: check company access
const company = await requireCompanyAccess(req, targetCompanyId);
const issue = await db.query.issues.findOne({ 
  where: (i) => and(eq(i.companyId, company.id), eq(i.id, issueId))
});

// ❌ Wrong: no company check
const issue = await db.query.issues.findOne({ where: { id: issueId } });
```

### Contract Synchronization

When changing data model or API behavior, update all impacted layers:
1. **`packages/db`** — schema definitions and exports
2. **`packages/shared`** — types, constants, validators
3. **`server`** — routes, services, business logic
4. **`ui`** — API clients and pages consuming the data

Missing a layer means broken contracts and runtime errors.

### Preserving Control-Plane Invariants

Do not weaken these:
- Single-assignee task model (no multi-owner tasks)
- Atomic issue checkout (no double-work)
- Approval gates for governed actions
- Budget hard-stop auto-pause on overspend
- Activity logging for all mutations

### No Internal Issue References

In PRs, commit messages, and comments, only reference public GitHub issues and PRs (`#123`, `Fixes #123`, full `https://github.com/paperclipai/paperclip/...` URLs). Do NOT reference internal Paperclip ticket IDs (`PAPA-123`), instance UI links (`/PAP/issues/...`), or localhost URLs—they're meaningless to reviewers.

### Branch Naming

Use short, kebab-case descriptive names, optionally with conventional prefixes:
```bash
git branch -m fix/sandbox-secret-resolution
git branch -m feat/adapter-retry-backoff
git branch -m docs/no-internal-issue-references
```

Do not include internal ticket IDs in branch names.

### Database Changes

Workflow:
1. Edit schema in `packages/db/src/schema/*.ts`
2. Ensure new tables are exported from `packages/db/src/schema/index.ts`
3. Generate migration: `pnpm db:generate`
4. Validate compile: `pnpm -r typecheck`

Migration applies automatically on next `pnpm dev` or via `pnpm db:migrate`.

## Testing Philosophy

**Default:** Use `pnpm test` (Vitest suite only) for normal development. It's fast and covers the majority of changes.

**When to use browser suites:**
- You're working on UI/browser flows specifically
- You're explicitly verifying CI or release paths
- A targeted change to E2E surface requires end-to-end verification

**Strategy:** For normal issue work, start with the smallest relevant check. The full stack check (`pnpm -r typecheck && pnpm test:run && pnpm build`) is for PR-ready handoff or broad changes only. If anything cannot be run, explicitly state what was not run and why.

## Release & CI

- `pnpm-lock.yaml` is owned by GitHub Actions; do not commit it in PRs
- PR CI validates dependency resolution when manifests change
- Pushes to `master` regenerate lockfile, commit it back, and verify with `--frozen-lockfile`
- Greptile automated code review must achieve 5/5 score (no P2+ comments)
- All Paperclip CI gates (lint, typecheck, tests, build) must pass before merge
- Every PR must include a **Model Used** section specifying AI model or "None — human-authored"

## Contributing Path

**Small, focused changes (fastest):** one clear thing to fix, smallest files, targeted, tests pass, Greptile 5/5.

**Bigger changes:** discuss in Discord #dev first, build, include full description/proof/testing/Greptile score, follow PR template.

See `CONTRIBUTING.md` for full details. Use the PR template at `.github/PULL_REQUEST_TEMPLATE.md`; every PR requires it.

## Telemetry & Observability

Telemetry is **enabled by default** (anonymous usage data, no PII/prompts/secrets). Disable with:
- `PAPERCLIP_TELEMETRY_DISABLED=1`
- `DO_NOT_TRACK=1`
- CI automatically disables when `CI=true`
- Or set `telemetry.enabled: false` in config

OpenTelemetry tracing is optional—activates when `OTEL_EXPORTER_OTLP_ENDPOINT` is set.

## Skillsets & Integration

- **Paperclip skill** — interact with Paperclip API for work coordination (assignments, status updates, comments, routines)
- **Design guide** — Paperclip UI design system guidance
- **Deep research** — multi-source fact-checked research
- **Code review** — automated code quality checks
- **Verify** — end-to-end verification of changes

## Local Development Tips

- **Environment:** Node.js 20+, pnpm 9.15+
- **No Docker required** — everything runs bare-metal in dev
- **Embedded DB:** zero setup, automatic; data persists in `~/.paperclip/instances/default/db/`
- **Dev idioms:** watch mode is default; use `pnpm dev:stop` to kill the runner
- **UI fonts:** Inter v4.1 is bundled in `ui/public/fonts/`; don't rely on system fonts
- **Storybook visual tests:** Linux-only for now (pixel-exact Playwright; macOS/Windows get subpixel rasterization diffs)

## Customization & Extension

Since you've forked this repo:

- Preference for **plugins** over forking core—use the plugin system for custom integrations
- Keep `doc/SPEC.md` and `doc/SPEC-implementation.md` aligned if you diverge from upstream
- Preserve control-plane invariants when adding features
- If adding an adapter, use the same patterns under `packages/adapters/`
- For team templates or skills, follow the patterns in `packages/teams-catalog/` and `packages/skills-catalog/`
