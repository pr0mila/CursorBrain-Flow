# planning.md — Product Planning & Scope

> **Cursor instruction**: When this file is referenced in a prompt, treat it as the authoritative source of truth for *what* this product does, who it serves, and what is in or out of scope. Never generate features or flows that contradict this document.

---

## 1. Product Overview

### Product Name
`[Your product name]`

### One-Line Description
`[What does it do? For whom? What's the core value?]`
> Example: "A Kanban-style task manager for remote engineering teams that auto-prioritizes work based on deadline proximity and blocker status."

### Problem Statement
```
[Describe the pain point in 3–5 sentences. Be specific — who feels this pain, when, and what does it cost them?]
```

### Solution Summary
```
[Describe your solution in 3–5 sentences. What does your product do differently? What is the core insight?]
```

---

## 2. Users & Personas

### Primary Persona
| Field         | Details |
|---------------|---------|
| Name          | `[e.g., "Alex the Engineering Lead"]` |
| Role          | `[Job title / context]` |
| Goals         | `[What are they trying to achieve?]` |
| Pain Points   | `[What frustrates them today?]` |
| Tech Comfort  | `[Low / Medium / High]` |

### Secondary Persona (optional)
| Field         | Details |
|---------------|---------|
| Name          | `[e.g., "Sam the Stakeholder"]` |
| Role          |  |
| Goals         |  |
| Pain Points   |  |

---

## 3. Goals & Success Metrics

### Business Goals
- [ ] `[Goal 1 — e.g., Reach 500 paying customers within 6 months]`
- [ ] `[Goal 2]`
- [ ] `[Goal 3]`

### Product Goals
- [ ] `[e.g., Core task flow completable in under 60 seconds]`
- [ ] `[e.g., Mobile-responsive for field users]`

### Success Metrics (KPIs)
| Metric             | Target         | Measured How |
|--------------------|----------------|--------------|
| Activation rate    | `[e.g., >60%]` | `[e.g., Mixpanel]` |
| Weekly Active Users| `[e.g., 200+]` | |
| NPS                | `[e.g., >40]`  | |
| Churn rate         | `[e.g., <5%/mo]`| |

---

## 4. Feature Scope

### MVP Features (Must Have)
> These are the minimum features required to solve the core problem and validate the product.

| # | Feature | User Story | Priority |
|---|---------|------------|----------|
| 1 | `[Feature name]` | As a `[persona]`, I want to `[action]` so that `[outcome]`. | P0 |
| 2 | | | P0 |
| 3 | | | P1 |

### Phase 2 Features (Should Have)
> Post-MVP enhancements that increase retention or expand the audience.

| # | Feature | Rationale |
|---|---------|-----------|
| 1 | `[Feature name]` | `[Why it matters]` |
| 2 | | |

### Out of Scope (Won't Have — This Version)
> Explicitly listing what is NOT being built prevents scope creep and keeps Cursor focused.

- `[Feature X]` — Reason: `[e.g., too complex for MVP, different audience]`
- `[Feature Y]` — Reason:
- `[Feature Z]` — Reason:

---

## 5. User Flows

### Core Flow: [Primary Action Name]
```
[Step 1] User lands on → [Page/Screen]
[Step 2] User does → [Action]
[Step 3] System responds → [Response]
[Step 4] User sees → [Result]
[Step 5] Success state → [What success looks like]
```

### Edge Cases & Error States
- What happens if `[edge case 1]`? → `[Expected behavior]`
- What happens if `[edge case 2]`? → `[Expected behavior]`
- What happens if the user is not authenticated? → `[Redirect / error / gate]`

---

## 6. Constraints & Assumptions

### Technical Constraints
- `[e.g., Must work offline — PWA required]`
- `[e.g., Must integrate with existing Salesforce CRM]`
- `[e.g., Cannot store PII in third-party services — GDPR]`

### Business Constraints
- `[e.g., Launch deadline: Q3 2025]`
- `[e.g., Budget cap: $50k for MVP build]`
- `[e.g., Team: 2 engineers, 1 designer]`

### Assumptions
- `[e.g., Users have a stable internet connection]`
- `[e.g., Primary device is desktop — mobile is secondary]`
- `[e.g., Users are comfortable with Kanban-style UX]`

---

## 7. Risks & Open Questions

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `[e.g., Users won't change existing workflow]` | Medium | High | `[Early user interviews, onboarding flow]` |
| `[e.g., Integration with third-party API fails]` | Low | High | `[Build abstraction layer, fallback plan]` |

### Open Questions
- [ ] `[Question 1 — who resolves this and by when?]`
- [ ] `[Question 2]`
- [ ] `[Question 3]`

---

## 8. Competitive Landscape

| Competitor | Strengths | Weaknesses | Our Differentiator |
|------------|-----------|------------|--------------------|
| `[Name]`   | `[...]`   | `[...]`    | `[...]`            |
| `[Name]`   | `[...]`   | `[...]`    | `[...]`            |

---

## 9. Timeline & Milestones

| Milestone | Description | Target Date | Owner |
|-----------|-------------|-------------|-------|
| Discovery complete | planning.md finalized, design.md started | `[Date]` | `[Name]` |
| Architecture complete | design.md finalized, development starts | `[Date]` | `[Name]` |
| MVP internal build | All P0 features working in staging | `[Date]` | `[Name]` |
| Beta launch | 20 invited users, feedback loop open | `[Date]` | `[Name]` |
| Public launch | Full release, monitoring live | `[Date]` | `[Name]` |

---

## 10. Cursor Usage Notes

When using this file with Cursor, here are the most effective prompt patterns:

```
# Generate a feature spec
@planning.md
Write a detailed feature specification for the [Feature Name] listed in the MVP scope.
Include acceptance criteria, edge cases, and a proposed API surface.

# Validate a new feature idea
@planning.md
I'm considering adding [new feature]. Does this conflict with any out-of-scope decisions
or contradict the core user persona's goals?

# Generate user stories
@planning.md
Generate a full set of user stories for the MVP features, formatted for a Jira/Linear import.
```
