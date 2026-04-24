# CursorBrain-Flow

> This repository follows a structured three-phase workflow optimized for AI-assisted development with Cursor. Every product decision, architecture choice, and implementation task is documented across three companion files that Cursor uses as persistent context.

---


## How This System Works

Cursor is most powerful when it has **rich, structured context** about what you're building and why. These four files act as a living brain for your project — feed them to Cursor at the right phase and it will generate code, catch architectural mistakes, and maintain consistency across your entire codebase.

```
planning.md   →   design.md   →   database.md   →   development.md
   "What"            "How"          "Where data       "Build it right"
                                     lives"
```

Each file serves a distinct purpose. You never mix concerns across files. `database.md` sits between `design.md` and `development.md` — it's informed by the architecture decisions in `design.md` and its conventions are enforced during development via `development.md`.

---

## The Four Files

### [`planning.md`](./planning.md)
**Phase: Discovery & Scoping**

The product brain. Contains everything about *what* you're building before a single line of code is written. Cursor reads this to understand goals, constraints, user personas, feature scope, and success criteria.

- Feed this to Cursor when: generating project scaffolding, writing PRDs, or scoping features
- Owner: Product / Founder
- Updated: When scope changes, priorities shift, or new features are approved

### [`design.md`](./design.md)
**Phase: Architecture & System Design**

The technical blueprint. Contains *how* the system is structured — data models, API contracts, component hierarchy, tech stack decisions, and architectural patterns. Cursor reads this to generate code that fits the system.

- Feed this to Cursor when: creating new modules, writing APIs, designing database schemas
- Owner: Lead Engineer / Architect
- Updated: When architectural decisions are made or changed

### [`database.md`](./database.md)
**Phase: Data Layer Design & Operations**

The data persistence brain. Contains everything about *where and how data is stored* — SQL schemas, NoSQL document structures, cache key conventions, query patterns, migration rules, cross-store integrity, and the decision map for choosing the right store.

- Feed this to Cursor when: writing queries, designing schemas, adding caching, writing migrations, building data access layers
- Owner: Backend Engineer / DBA
- Updated: When a new store is introduced, a schema changes, or cache patterns evolve
- **Sync rule**: Schema changes here must be reflected in `design.md`'s Entity Relationship section. Cache env vars must be added to `development.md`'s Environment Variables table.

### [`development.md`](./development.md)
**Phase: Implementation & Standards**

The engineering rulebook. Contains *how we write code* — conventions, folder structure, code style, testing strategy, Git workflow, CI/CD, and environment setup. Cursor reads this to enforce consistency.

- Feed this to Cursor when: writing features, reviewing PRs, onboarding new engineers
- Owner: Engineering Team
- Updated: When team conventions evolve or new tooling is adopted

---

## Cursor Integration Guide

### Method 1: `@file` Mentions (Recommended)
Reference files directly in your Cursor prompt:

```
@planning.md @design.md  
Build the user authentication module based on the data models and scope defined here.
```

### Method 2: `.cursorrules` (Persistent Context)
Create a `.cursorrules` file at your repo root to automatically inject context into every Cursor session:

```
# .cursorrules
- Always follow conventions in development.md
- Git workflow is Trunk-Based Development — one trunk (main), short-lived branches (≤2 days), squash merge
- All data persistence decisions follow database.md — SQL, NoSQL, and cache patterns
- Data models are defined in design.md — never create new schemas without checking there first
- database.md is the authority on query patterns, migrations, and cache key naming
- Features must align with the scope in planning.md
- Large features that can't ship in one PR must use feature flags (see development.md)
```

### Method 3: Cursor Notepads
For long-running projects, store each file as a Cursor Notepad so it's always one click away without re-uploading.

### Method 4: Context on Demand
Structure your prompts with explicit file references:

```
Context: [paste relevant section from design.md]
Task: Implement the UserProfile component following these data contracts.
Rule: Follow the code conventions in development.md.
```

---

## Workflow in Practice

```
1. PLAN      Fill out planning.md before any code
             → Use Cursor to challenge assumptions, spot gaps

2. DESIGN    Fill out design.md after planning is locked
             → Use Cursor to generate ERDs, API specs, component trees

3. DATA      Fill out database.md alongside design.md
             → Use Cursor to design schemas, pick the right store, plan cache strategy

4. DEVELOP   Keep development.md open in every Cursor session
             → Use Cursor to write, refactor, test, and review code

5. ITERATE   Update all four files when things change
             → Never let the docs drift from the codebase
```

---

## Rules for Keeping This System Healthy

- **One source of truth**: If something is in `design.md`, it is not in `development.md`. Never duplicate.
- **Docs before code**: Update the relevant `.md` file *before* asking Cursor to implement a change.
- **Version with the code**: These files live in the repo. They are not wikis or Notion pages — they are source files.
- **Review in PRs**: Changes to any of these files should go through code review like any other change.
- **Don't let them rot**: A stale `planning.md` is worse than no `planning.md`. Schedule a monthly review.

---

## Quick Reference

| I want to...                          | Read                        | Update                      |
|---------------------------------------|-----------------------------|-----------------------------|
| Understand the product goals          | `planning.md`               | `planning.md`               |
| Know what's in scope                  | `planning.md`               | `planning.md`               |
| Understand the database schema        | `database.md`               | `database.md` + `design.md` |
| Know which tech stack we use          | `design.md`                 | `design.md`                 |
| Decide SQL vs NoSQL vs Cache          | `database.md`               | `database.md`               |
| Write a query or migration            | `database.md`               | `database.md`               |
| Add caching to a feature             | `database.md`               | `database.md`               |
| Set up my local dev environment       | `development.md`            | —                           |
| Know how to name a variable/function  | `development.md`            | `development.md`            |
| Understand Git / branching strategy   | `development.md §7`         | `development.md`            |
| Understand the CI/CD pipeline         | `development.md §9`         | `development.md`            |
| Wrap a feature in a flag              | `development.md §7`         | `lib/flags.ts`              |
| Write a Cursor prompt for a feature   | All four                    | —                           |

---

## File Maintenance Checklist

Run this checklist at the start of every sprint:

- [ ] `planning.md` reflects current priorities and approved scope
- [ ] `design.md` matches what's actually been built (APIs, components, high-level ERD)
- [ ] `database.md` reflects current schemas, index decisions, and cache key patterns
- [ ] `development.md` reflects current tooling and conventions
- [ ] `.cursorrules` is up to date
- [ ] No contradictions exist between the four files — especially between `database.md` and `design.md` schema sections
