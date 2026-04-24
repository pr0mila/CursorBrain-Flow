# development.md — Engineering Standards & Workflow

> **Cursor instruction**: When this file is referenced in a prompt, treat it as the authoritative rulebook for *how we write code*. All generated code must conform to the naming conventions, file structure, testing standards, and patterns defined here. When in doubt about style or convention, this file is the tiebreaker. For all database queries, schema design, cache patterns, and migration rules, defer to `database.md` — that file owns the data layer.

---

## 1. Environment Setup

### Prerequisites
```bash
node >= [20.x]
pnpm >= [8.x]          # package manager (not npm, not yarn)
docker >= [24.x]       # for local database
```

### First-Time Setup
```bash
git clone [repo-url]
cd [project]

# Install dependencies
pnpm install

# Set up environment variables
cp .env.example .env.local
# Fill in required values (see Environment Variables section)

# Start local database
docker compose up -d

# Run migrations
pnpm db:migrate

# Seed development data
pnpm db:seed

# Start development server
pnpm dev
```

### Environment Variables
| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | ✅ | PostgreSQL connection string — format defined in `database.md §1` |
| `MONGODB_URI` | ✅/⬜ | MongoDB connection string — required if MongoDB is in use |
| `MONGODB_DB_NAME` | ✅/⬜ | MongoDB database name |
| `REDIS_URL` | ✅/⬜ | Redis connection URL (local dev) |
| `UPSTASH_REDIS_REST_URL` | ✅/⬜ | Upstash Redis URL (production) |
| `UPSTASH_REDIS_REST_TOKEN` | ✅/⬜ | Upstash Redis token (production) |
| `NEXTAUTH_SECRET` | ✅ | Random secret for session signing |
| `NEXTAUTH_URL` | ✅ | App base URL (e.g., http://localhost:3000) |
| `[SERVICE]_API_KEY` | ✅ | `[What service, what for]` |
| `[OPTIONAL_VAR]` | ⬜ | `[Description]` |

> **Rule**: Never commit `.env.local`. Never hardcode secrets. Always add new variables to `.env.example` with a placeholder value and document them here. Full DB connection string formats are in `database.md`.

---

## 2. Scripts Reference

```bash
pnpm dev            # Start dev server (localhost:3000)
pnpm build          # Production build
pnpm start          # Start production server
pnpm lint           # Run ESLint
pnpm format         # Run Prettier
pnpm typecheck      # Run TypeScript compiler check
pnpm test           # Run unit tests (Vitest)
pnpm test:e2e       # Run end-to-end tests (Playwright)

# PostgreSQL (Prisma) — conventions in database.md §1
pnpm db:migrate     # Apply pending migrations
pnpm db:generate    # Generate Prisma client after schema change
pnpm db:seed        # Seed development database
pnpm db:studio      # Open Prisma Studio (database GUI)
pnpm db:reset       # ⚠️  Drop and recreate database (dev only)

# All stores (local Docker)
pnpm infra:up       # docker compose up -d (postgres + mongo + redis)
pnpm infra:down     # docker compose down
pnpm infra:logs     # docker compose logs -f
```

> All query patterns, migration rules, and cache conventions are in `database.md`. This file lists the scripts — `database.md` defines what those scripts should produce.

---

## 3. Code Conventions

### TypeScript
- **Strict mode is always on** — `strict: true` in `tsconfig.json`
- Prefer `type` over `interface` for object shapes unless you need declaration merging
- Never use `any` — use `unknown` and narrow, or define the type explicitly
- Always type function return values explicitly for public/exported functions
- Use `satisfies` for config objects when you want type inference without widening

```typescript
// ✅ Good
type User = { id: string; name: string; email: string }
const getUser = async (id: string): Promise<User | null> => { ... }

// ❌ Bad
const getUser = async (id: any) => { ... }
```

### Naming Conventions
| Thing | Convention | Example |
|-------|-----------|---------|
| Variables | `camelCase` | `userName`, `isLoading` |
| Functions | `camelCase`, verb-first | `getUser`, `handleSubmit`, `formatDate` |
| Components | `PascalCase` | `UserCard`, `TaskList` |
| Types / Interfaces | `PascalCase` | `UserProfile`, `ApiResponse<T>` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_FILE_SIZE`, `API_BASE_URL` |
| Files (components) | `PascalCase.tsx` | `UserCard.tsx` |
| Files (utils/hooks) | `camelCase.ts` | `useAuth.ts`, `formatDate.ts` |
| Files (API routes) | `route.ts` (Next.js App Router) | `app/api/users/route.ts` |
| DB columns | `snake_case` | `created_at`, `user_id` |
| Env variables | `SCREAMING_SNAKE_CASE` | `DATABASE_URL` |

### File Structure per Feature
```
features/[feature-name]/
├── components/           # Feature-specific components
│   └── [Component].tsx
├── hooks/                # Feature-specific hooks
│   └── use[Feature].ts
├── types.ts              # Feature-specific types
├── utils.ts              # Feature-specific utilities
└── index.ts              # Barrel export
```

### Import Order
Enforce with ESLint import plugin. Order:
```typescript
// 1. Node built-ins
import path from 'path'

// 2. External packages
import { z } from 'zod'
import { useQuery } from '@tanstack/react-query'

// 3. Internal absolute imports
import { Button } from '@/components/ui/button'
import { useAuth } from '@/hooks/useAuth'

// 4. Relative imports
import { TaskCard } from './components/TaskCard'
import type { Task } from './types'
```

---

## 4. Component Patterns

### Server vs Client Components (Next.js)
```typescript
// Server Component (default) — no 'use client', can be async
export default async function TaskList() {
  const tasks = await db.task.findMany()  // direct DB access
  return <ul>{tasks.map(t => <TaskItem key={t.id} task={t} />)}</ul>
}

// Client Component — add 'use client' only when you need:
// - useState / useEffect / event handlers / browser APIs
'use client'
export function TaskItem({ task }: { task: Task }) {
  const [open, setOpen] = useState(false)
  ...
}
```

> **Rule**: Default to Server Components. Add `'use client'` only when necessary. Never fetch data in client components directly — pass data as props from server components or use React Query.

### Custom Hook Pattern
```typescript
// hooks/useTask.ts
export function useTask(taskId: string) {
  return useQuery({
    queryKey: ['task', taskId],
    queryFn: () => api.tasks.getById(taskId),
  })
}
```

### Error Boundaries
Wrap all major page sections with an error boundary. Use the shared `<ErrorBoundary>` component from `components/ui/`.

---

## 5. API & Data Fetching

### API Client
All API calls go through the centralized client in `lib/api.ts`. Never use raw `fetch` directly in components.

```typescript
// lib/api.ts — typed API client
export const api = {
  tasks: {
    list: () => fetchJson<Task[]>('/api/v1/tasks'),
    getById: (id: string) => fetchJson<Task>(`/api/v1/tasks/${id}`),
    create: (data: CreateTaskInput) => fetchJson<Task>('/api/v1/tasks', { method: 'POST', body: data }),
    update: (id: string, data: UpdateTaskInput) => fetchJson<Task>(`/api/v1/tasks/${id}`, { method: 'PATCH', body: data }),
    delete: (id: string) => fetchJson<void>(`/api/v1/tasks/${id}`, { method: 'DELETE' }),
  }
}
```

### Validation
- All API inputs are validated with **Zod** at the route handler boundary
- Validation schemas live in `lib/validators/[resource].ts`
- The same schema is reused for client-side form validation

```typescript
// lib/validators/task.ts
export const createTaskSchema = z.object({
  title: z.string().min(1).max(255),
  description: z.string().optional(),
  priority: z.enum(['low', 'medium', 'high']),
})
export type CreateTaskInput = z.infer<typeof createTaskSchema>
```

---

## 6. Testing Strategy

### Testing Pyramid
```
          /‾‾‾‾‾‾‾‾‾\
         /  E2E Tests  \      ~10% — Playwright
        /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
       /  Integration Tests \  ~30% — Vitest + DB
      /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
     /     Unit Tests        \  ~60% — Vitest
    /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
```

### What to Test
| Layer | What to test | Tool |
|-------|-------------|------|
| Utils / helpers | Pure functions, edge cases | Vitest |
| API handlers | Request/response contracts, validation | Vitest + supertest |
| React components | User interactions, rendering | Vitest + Testing Library |
| Critical user flows | Login, signup, core actions | Playwright |

### Test File Conventions
```
src/
├── lib/
│   ├── formatDate.ts
│   └── formatDate.test.ts    # Unit test next to source file
├── app/api/tasks/
│   ├── route.ts
│   └── route.test.ts         # Integration test next to route
tests/
└── e2e/
    └── auth.spec.ts           # E2E tests in separate directory
```

### Test Writing Rules
- Tests describe **behavior**, not implementation — test what a function does, not how
- One assertion concept per test (can have multiple expect calls if they test the same thing)
- Use `describe` blocks to group related tests
- Use `it('should ...')` naming convention
- Mock external services (DB, email, Stripe) in unit tests; use real DB in integration tests

```typescript
describe('createTask', () => {
  it('should create a task with valid input', async () => { ... })
  it('should throw validation error for empty title', async () => { ... })
  it('should return 401 for unauthenticated requests', async () => { ... })
})
```

---

## 7. Git Workflow — Trunk-Based Development

> We use **Trunk-Based Development (TBD)**. There is one trunk: `main`. It is always green, always deployable, and always production-ready. No long-lived branches. No `develop`. No GitFlow. Every engineer integrates into `main` at least once per day.

### The Golden Rule
```
main is sacred. If main is broken, everything stops until it is fixed.
No feature is worth a broken trunk.
```

### Branch Model
```
main  ← trunk (protected, always deployable, CI must be green)
  ├── [yourname]/TASK-123-short-description   ← short-lived feature branch (< 2 days)
  ├── [yourname]/fix-login-redirect           ← short-lived fix branch (hours, not days)
  └── release/v1.4.0                          ← release branch (cut from main, read-only after cut)
```

**Rules:**
- `main` is the only permanent branch
- Feature branches live for **≤ 2 days** — if it's taking longer, the task needs to be broken down smaller
- Release branches are cut from `main` at release time, receive only critical hotfixes, and are **never merged back** — hotfixes cherry-pick to `main` instead
- No `develop`, `staging`, `integration`, or any other long-lived integration branch

### Branch Naming
```
Format: [author]/[ticket-id]-[2-4-word-description]

✅  alice/TASK-123-user-auth
✅  bob/fix-login-redirect
✅  carol/TASK-456-add-rate-limit
❌  feature/user-authentication          (no author prefix)
❌  alice/TASK-123-implement-the-full-user-authentication-flow-with-oauth  (too long)
❌  develop                              (forbidden long-lived branch)
```

### Daily Integration Flow
```
1. Pull latest main
   git checkout main && git pull origin main

2. Create short-lived branch
   git checkout -b alice/TASK-123-task-pagination

3. Work in small commits (see commit rules below)
   git commit -m "feat(tasks): add cursor-based pagination to task list"

4. Keep branch in sync — rebase onto main daily, not merge
   git fetch origin && git rebase origin/main

5. Open PR → CI runs → 1 approval → Squash and merge to main
   Branch is deleted immediately after merge

6. main auto-deploys to production (see CI/CD section)
```

### Commit Message Format (Conventional Commits)

```
<type>(<scope>): <short description in imperative mood>

[optional body — explain WHY, not WHAT]

[optional footer: Closes TASK-123]
```

**Types:**
| Type | When to Use |
|------|------------|
| `feat` | New user-facing feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting — zero logic change |
| `refactor` | Restructure without feature or fix |
| `test` | Adding or fixing tests |
| `chore` | Tooling, deps, config — nothing users see |
| `perf` | Measurable performance improvement |
| `revert` | Reverting a previous commit |

**Commit rules:**
- **Imperative mood**: "add pagination" not "added pagination" or "adding pagination"
- **One logical change per commit** — if your commit message needs "and", split the commit
- **Commits must pass CI locally** before push — never push a broken commit to any branch
- Body explains *why*, not *what* — the diff shows what changed

```bash
# ✅ Good commits
feat(tasks): add cursor-based pagination to task list
fix(auth): prevent session token from expiring mid-request
refactor(api): extract error handler into shared middleware
chore: upgrade Prisma from 5.7 to 5.8

# ❌ Bad commits
WIP
fix stuff
feat: implement the entire authentication system with OAuth Google and GitHub
updated files
```

### Feature Flags — Enabling Large Features in Trunk

When a feature is too large to ship in one PR but must live in `main` before it's complete, wrap it in a **feature flag**. This lets code be merged to trunk without being exposed to users.

```typescript
// lib/flags.ts — central feature flag registry
export const flags = {
  newTaskBoard:      process.env.FLAG_NEW_TASK_BOARD === 'true',
  aiSuggestions:     process.env.FLAG_AI_SUGGESTIONS === 'true',
  bulkImport:        process.env.FLAG_BULK_IMPORT === 'true',
} as const

// Usage in a component
import { flags } from '@/lib/flags'

export default function Dashboard() {
  return (
    <main>
      <TaskList />
      {flags.newTaskBoard && <NewTaskBoard />}   {/* hidden until flag is enabled */}
    </main>
  )
}
```

**Feature flag rules:**
- Every flag must be in `lib/flags.ts` — no inline `process.env` checks scattered in code
- Flags are **temporary** — they are removed when the feature fully ships or is killed
- A flag with no removal date is a bug — every flag PR must note its removal milestone
- Flags are toggled via environment variables — no database or UI flag management for now

### Pull Request Rules

- **PRs are small by design** — TBD requires small tasks → small PRs. Target: ≤ 200 lines changed, ≤ 400 hard limit. If over 400 lines, it must be split.
- Every PR references a ticket: `Closes TASK-123` in the description
- **Squash and merge** — all commits on a PR are squashed into one clean commit on `main`
- 1 approval required before merge — no self-merge
- All CI gates must be green before merge (typecheck, lint, test, build)
- Branch is deleted immediately after merge — no exceptions
- PRs that have been open for more than 2 days without merge are a red flag — escalate or break down the task

### PR Description Template
```markdown
## What
[One paragraph: what changed and what it does]

## Why
[Why is this needed? Link to ticket: Closes TASK-123]

## How
[Brief explanation of the approach — what was the key decision?]

## Testing
- [ ] Unit tests added/updated
- [ ] Tested locally — describe what you clicked/triggered
- [ ] Edge cases considered: [list them]

## Feature Flag
[Flag name if applicable, and the milestone it will be removed in]

## Screenshots (UI changes only)
```

### Hotfix Process

When a critical bug hits production, the hotfix goes directly to `main` — never to a release branch first.

```
1. Cut a branch from main immediately
   git checkout -b alice/hotfix-payment-crash

2. Fix, test, commit
   git commit -m "fix(payments): prevent crash on missing stripe webhook signature"

3. Emergency PR — 1 approval, expedited CI
   Merge to main → auto-deploys to production

4. If a release branch is active, cherry-pick the fix there
   git checkout release/v1.4.0
   git cherry-pick <commit-sha>
```

### Release Process

```
1. Cut release branch from main at the agreed release commit
   git checkout -b release/v1.4.0 <commit-sha>

2. Tag the release
   git tag -a v1.4.0 -m "Release v1.4.0"
   git push origin v1.4.0

3. Release branch is now READ-ONLY — only cherry-picked hotfixes allowed
   No features. No refactors. No "just one more thing".

4. After release, delete the release branch — main continues
```

---

## 8. Code Review Standards

### Reviewer Checklist
- [ ] Does the code follow naming conventions in this file?
- [ ] Is the TypeScript types correct? No `any`?
- [ ] Is new behavior covered by tests?
- [ ] Are there any security concerns? (auth checks, input validation)
- [ ] Are there performance concerns? (N+1 queries, large bundles)
- [ ] Does it introduce new patterns not in `design.md`?
- [ ] Is error handling complete?

### Feedback Tone
- Prefix blocking comments with `blocking:` or `nit:` to signal severity
- Explain *why* something is a problem, not just that it is
- Suggest solutions, don't just point out issues

---

## 9. CI/CD Pipeline — Trunk-Based

> The pipeline is the enforcement mechanism of Trunk-Based Development. Every merge to `main` triggers an automatic, gated pipeline that takes code from commit to production. There is no manual deploy step in normal operation.

### Pipeline Overview

```
 PR opened                         Merge to main                    Production
     │                                   │                               │
     ▼                                   ▼                               ▼
┌─────────┐   ┌──────────┐   ┌─────────────────┐   ┌──────────┐   ┌─────────┐
│ PR Gate │──▶│  Squash  │──▶│  Trunk Pipeline │──▶│ Staging  │──▶│  Prod   │
│  (CI)   │   │ & Merge  │   │  (full suite)   │   │  Smoke   │   │  Live   │
└─────────┘   └──────────┘   └─────────────────┘   └──────────┘   └─────────┘
  ~3 min          auto             ~8 min               ~2 min        auto
```

### Stage 1 — PR Gate (runs on every PR, every push to branch)

Must be **fast** — engineers should not wait more than 3–4 minutes for PR feedback.

```yaml
# .github/workflows/pr-gate.yml
name: PR Gate
on: [pull_request]

jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile

      - name: Type check
        run: pnpm typecheck

      - name: Lint
        run: pnpm lint

      - name: Unit tests
        run: pnpm test --coverage
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

      - name: Build check
        run: pnpm build
```

**PR Gate rules:**
- All four checks must pass — any failure blocks the PR
- No exceptions, no "I'll fix the tests in the next PR"
- If the gate is slow (>5 min), fix the tests — do not skip them
- Preview deploy is created automatically for visual review (Vercel / Railway)

### Stage 2 — Trunk Pipeline (runs on every merge to `main`)

Runs the full suite including integration and E2E. This is the authoritative pipeline.

```yaml
# .github/workflows/trunk.yml
name: Trunk Pipeline
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: testdb }
        options: --health-cmd pg_isready
      redis:
        image: redis:7-alpine
        options: --health-cmd "redis-cli ping"

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile

      - name: Run migrations
        run: pnpm db:migrate
        env: { DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb }

      - name: Type check
        run: pnpm typecheck

      - name: Lint
        run: pnpm lint

      - name: Unit + Integration tests
        run: pnpm test --coverage
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      - name: Build
        run: pnpm build

  deploy-staging:
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          # Deploy to staging environment
          # Run db migrations on staging
          # Trigger staging smoke tests

  smoke-staging:
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    steps:
      - name: Smoke test staging
        run: pnpm test:e2e --project=staging
        env: { BASE_URL: ${{ secrets.STAGING_URL }} }

  deploy-production:
    needs: [smoke-staging]
    runs-on: ubuntu-latest
    environment: production          # requires manual approval gate in GitHub
    steps:
      - name: Deploy to Production
        run: |
          # Deploy to production
          # Run db migrations on production
          # Notify Slack
```

### Stage 3 — Deployment Gates

```
main merge
    │
    ▼
Staging auto-deploy ──▶ Staging smoke tests pass?
                              │ YES                │ NO
                              ▼                    ▼
                    Manual approval gate      Pipeline fails,
                    (GitHub Environments)     Slack alert,
                              │               no prod deploy
                              ▼
                    Production deploy
                              │
                              ▼
                    Production smoke tests
                              │ FAIL
                              ▼
                    Auto-rollback + Slack alert
```

### Smoke Tests

Smoke tests run against real deployed environments — they test the critical paths only, not exhaustive coverage. They must complete in under 2 minutes.

```typescript
// tests/e2e/smoke.spec.ts
test('health check returns 200', async ({ request }) => {
  const res = await request.get('/api/health')
  expect(res.status()).toBe(200)
})

test('user can log in', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name=email]', process.env.SMOKE_TEST_EMAIL!)
  await page.fill('[name=password]', process.env.SMOKE_TEST_PASSWORD!)
  await page.click('[type=submit]')
  await expect(page).toHaveURL('/dashboard')
})

test('core API responds correctly', async ({ request }) => {
  const res = await request.get('/api/v1/tasks', {
    headers: { Authorization: `Bearer ${process.env.SMOKE_TEST_TOKEN}` }
  })
  expect(res.status()).toBe(200)
})
```

### Rollback Strategy

```
Automatic rollback triggers when:
  - Production smoke tests fail after deploy
  - Error rate spikes >5% within 5 minutes of deploy (Sentry/Datadog alert)

Manual rollback:
  git revert <merge-commit-sha>     # creates a revert commit
  git push origin main              # triggers pipeline, deploys revert
  
  # NEVER use git reset on main — it rewrites shared history
  # NEVER force-push to main — ever
```

### Deployment Environments

| Environment | Trigger | URL | DB Migrations | Approval |
|-------------|---------|-----|---------------|----------|
| Preview | PR opened | auto-generated URL | ❌ none | ❌ auto |
| Staging | Merge to `main` | `[staging-url]` | ✅ auto | ❌ auto |
| Production | Staging smoke pass | `[prod-url]` | ✅ auto | ✅ manual gate |

### Branch Protection Rules (GitHub Settings)

Configure these on the `main` branch — they are non-negotiable:

```
✅ Require a pull request before merging
✅ Require 1 approval
✅ Dismiss stale reviews when new commits are pushed
✅ Require status checks to pass (PR Gate workflow)
✅ Require branches to be up to date before merging
✅ Require linear history (enforces squash merge)
❌ Allow force pushes — NEVER
❌ Allow deletions — NEVER
```

### Pipeline Health Rules

- **If `main` is red, it is the team's top priority** — no new PRs merge until it's green
- Pipeline failures send immediate Slack alerts to `#deployments`
- A pipeline that takes >15 minutes is a performance bug — fix it
- Flaky tests must be fixed within the same sprint they're detected — quarantine with `test.skip` and file a ticket immediately, never leave flaky tests in place

---

## 10. Cursor Usage Notes

These prompt patterns work best with this file as context:

```
# Write a new feature following our conventions
@development.md @design.md @database.md
Implement the create task API endpoint.
Follow all naming conventions, validation patterns, and error handling from development.md.
Use the Task schema from design.md. Follow the query patterns from database.md.

# Generate a test file
@development.md
Write a full test suite for the createTask function in lib/tasks.ts.
Follow our testing conventions — unit tests with Vitest, describe/it pattern.

# Review code for convention violations
@development.md @database.md
Review this code and flag anything that violates our conventions —
naming, TypeScript strictness, and any database anti-patterns (N+1, missing pagination, bad cache keys):
[paste code]

# Generate a PR description
@development.md
Write a PR description for these changes: [describe changes]
Follow our trunk-based PR description template.

# Set up a new feature module
@development.md
Create the folder structure and boilerplate for a new "notifications" feature module.
Follow our feature folder structure and export conventions.

# Add caching to an existing feature
@development.md @database.md
Add Redis caching to the getWorkspaceTasks function.
Follow the cache-aside pattern and key naming conventions from database.md.

# Wrap a large feature in a feature flag
@development.md
I need to merge the new AI suggestions feature to main before it's complete.
Create the feature flag entry in lib/flags.ts and wrap the relevant component.
Follow our feature flag rules.

# Write a GitHub Actions workflow
@development.md
Generate the PR Gate GitHub Actions workflow for this project.
Follow our trunk-based pipeline stages — fast gate for PRs, full suite on main merge.

# Break down a large task for trunk-based development
@development.md
This task is too large to merge in 2 days: [describe task].
Break it down into sub-tasks that each produce a mergeable PR under 400 lines,
using feature flags where the intermediate states should not be user-visible.
```
