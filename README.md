# CursorBrain-Flow

> This repository follows a structured three-phase workflow optimized for AI-assisted development with Cursor. Every product decision, architecture choice, and implementation task is documented across three companion files that Cursor uses as persistent context.

---

## How This System Works

Cursor is most powerful when it has **rich, structured context** about what you're building and why. These three files act as a living brain for your project — feed them to Cursor at the right phase and it will generate code, catch architectural mistakes, and maintain consistency across your entire codebase.

```
planning.md   →   design.md   →   development.md
   "What"            "How"            "Build"
```

Each file serves a distinct purpose and maps to a distinct phase of your product lifecycle. You never mix concerns across files.

---

## The Three Files

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
- Data models are defined in design.md — never create new schemas without checking there first
- Features must align with the scope in planning.md
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

3. DEVELOP   Keep development.md open in every Cursor session
             → Use Cursor to write, refactor, test, and review code

4. ITERATE   Update all three files when things change
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
| Understand the database schema        | `design.md`                 | `design.md`                 |
| Know which tech stack we use          | `design.md`                 | `design.md`                 |
| Set up my local dev environment       | `development.md`            | —                           |
| Know how to name a variable/function  | `development.md`            | `development.md`            |
| Understand Git branching strategy     | `development.md`            | `development.md`            |
| Write a Cursor prompt for a feature   | All three                   | —                           |

---

## File Maintenance Checklist

Run this checklist at the start of every sprint:

- [ ] `planning.md` reflects current priorities and approved scope
- [ ] `design.md` matches what's actually been built (schemas, APIs, components)
- [ ] `development.md` reflects current tooling and conventions
- [ ] `.cursorrules` is up to date
- [ ] No contradictions exist between the three files
