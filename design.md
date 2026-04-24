# design.md — System Architecture & Technical Design

> **Cursor instruction**: When this file is referenced in a prompt, treat it as the authoritative source of truth for *how* this system is structured. All generated code must conform to the data models, API contracts, component hierarchy, and architectural patterns defined here. Never introduce new schemas, services, or patterns without updating this file first.

---

## 1. Tech Stack

### Core Technologies
| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Frontend | `[e.g., Next.js]` | `[e.g., 14.x]` | `[e.g., SSR + App Router for SEO]` |
| Backend | `[e.g., Node.js / Express]` | `[e.g., 20.x]` | |
| Database | `[e.g., PostgreSQL]` | `[e.g., 15]` | |
| ORM | `[e.g., Prisma]` | `[e.g., 5.x]` | |
| Auth | `[e.g., NextAuth.js / Clerk]` | | |
| File Storage | `[e.g., AWS S3 / Cloudflare R2]` | | |
| Cache | `[e.g., Redis / Upstash]` | | |
| Email | `[e.g., Resend / Postmark]` | | |
| Payments | `[e.g., Stripe]` | | |
| Deployment | `[e.g., Vercel / Railway / AWS]` | | |
| Monitoring | `[e.g., Sentry / Datadog]` | | |

### Why This Stack
```
[2–3 sentences explaining the core reasoning behind these choices. 
What trade-offs were made? What was rejected and why?]
```

---

## 2. System Architecture

### High-Level Architecture
```
[Draw or describe the system architecture here. Example:]

┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Browser   │────▶│  Next.js App │────▶│  PostgreSQL │
│  (React)    │     │  (API Routes)│     │  (Primary)  │
└─────────────┘     └──────┬───────┘     └─────────────┘
                           │
                    ┌──────▼───────┐
                    │    Redis     │
                    │  (Sessions / │
                    │   Cache)     │
                    └──────────────┘
```

### Service Boundaries
| Service/Module | Responsibility | Owns |
|----------------|---------------|------|
| `[e.g., auth]` | `[User authentication and session management]` | `[Users table, sessions]` |
| `[e.g., billing]` | `[Stripe integration, subscription state]` | `[Subscriptions table]` |
| `[e.g., notifications]` | `[Email, in-app alerts]` | `[Notifications table]` |
| `[e.g., core]` | `[Primary product domain logic]` | `[Main domain tables]` |

---

## 3. Database Schema

> All data models are defined here. Cursor should NEVER create new database tables or fields without updating this section.

### Entity Relationship Overview
```
[Describe the key relationships in plain text or ASCII diagram]

User ──< Workspace ──< Project ──< Task
                              └──< Comment
User ──< Session
Workspace ──< Member (User)
```

### Core Models

#### `users`
```sql
id            UUID          PRIMARY KEY DEFAULT gen_random_uuid()
email         VARCHAR(255)  UNIQUE NOT NULL
name          VARCHAR(255)
avatar_url    TEXT
created_at    TIMESTAMPTZ   DEFAULT now()
updated_at    TIMESTAMPTZ   DEFAULT now()
```

#### `[model_2]`
```sql
id            UUID          PRIMARY KEY DEFAULT gen_random_uuid()
-- Add fields here
created_at    TIMESTAMPTZ   DEFAULT now()
updated_at    TIMESTAMPTZ   DEFAULT now()
```

#### `[model_3]`
```sql
id            UUID          PRIMARY KEY DEFAULT gen_random_uuid()
-- Add fields here
```

### Indexes
```sql
-- Add indexes for frequently queried fields
CREATE INDEX idx_[table]_[field] ON [table]([field]);
```

### Enums
```typescript
// Shared TypeScript enums — mirrors DB enum types
export enum UserRole { OWNER = 'owner', ADMIN = 'admin', MEMBER = 'member' }
export enum Status   { ACTIVE = 'active', ARCHIVED = 'archived', DELETED = 'deleted' }
```

---

## 4. API Design

> RESTful by default. Use the patterns here consistently. Cursor must follow these conventions when generating new endpoints.

### API Conventions
- **Base URL**: `/api/v1/`
- **Auth**: Bearer token in `Authorization` header
- **Response envelope**:
```json
{
  "data": {},
  "error": null,
  "meta": { "page": 1, "total": 100 }
}
```
- **Error shape**:
```json
{
  "data": null,
  "error": { "code": "NOT_FOUND", "message": "Resource not found" }
}
```

### HTTP Status Code Conventions
| Status | When to Use |
|--------|------------|
| `200`  | Successful GET, PATCH, DELETE |
| `201`  | Successful POST (resource created) |
| `400`  | Validation error / malformed request |
| `401`  | Unauthenticated |
| `403`  | Authenticated but unauthorized |
| `404`  | Resource not found |
| `409`  | Conflict (duplicate resource) |
| `422`  | Unprocessable entity |
| `500`  | Server error |

### Endpoint Definitions

#### Auth
```
POST   /api/v1/auth/register        Register new user
POST   /api/v1/auth/login           Login with email + password
POST   /api/v1/auth/logout          Invalidate session
POST   /api/v1/auth/refresh         Refresh access token
GET    /api/v1/auth/me              Get current user
```

#### [Resource 1]
```
GET    /api/v1/[resource]           List (paginated)
POST   /api/v1/[resource]           Create
GET    /api/v1/[resource]/:id       Get by ID
PATCH  /api/v1/[resource]/:id       Partial update
DELETE /api/v1/[resource]/:id       Soft delete
```

#### [Resource 2]
```
GET    /api/v1/[resource]           List
POST   /api/v1/[resource]           Create
...
```

---

## 5. Frontend Architecture

### Folder Structure
```
src/
├── app/                    # Next.js App Router pages
│   ├── (auth)/             # Auth route group
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/        # Authenticated route group
│   │   └── [module]/
│   └── api/                # API route handlers
├── components/
│   ├── ui/                 # Primitive UI components (Button, Input, Modal)
│   ├── [feature]/          # Feature-specific components
│   └── layout/             # Layout components (Header, Sidebar, Footer)
├── hooks/                  # Custom React hooks
├── lib/                    # Utilities, API clients, helpers
├── stores/                 # Global state (Zustand / Redux)
├── types/                  # Shared TypeScript types
└── styles/                 # Global CSS / design tokens
```

### Component Architecture
```
Page (data fetching, routing)
  └── Feature Component (business logic)
        └── UI Component (pure display, no business logic)
```

**Rules:**
- UI components in `components/ui/` must be **stateless and generic**
- Business logic lives in hooks (`hooks/use[Feature].ts`)
- Data fetching happens in page-level server components or React Query hooks
- Global state (auth, UI state) lives in Zustand stores

### State Management
| State Type | Solution | Location |
|-----------|---------|---------|
| Server/async state | `[e.g., React Query / SWR]` | `hooks/` |
| Global UI state | `[e.g., Zustand]` | `stores/` |
| Form state | `[e.g., React Hook Form]` | component-local |
| URL state | Next.js `useSearchParams` | page-level |

---

## 6. Authentication & Authorization

### Auth Strategy
`[Describe your auth approach: JWT, session-based, OAuth providers, etc.]`

### Session Flow
```
1. User submits credentials
2. Server validates → creates session / issues JWT
3. Token stored in → [httpOnly cookie / localStorage]
4. Protected routes check → [middleware / layout guard]
5. Token refresh → [when / how]
```

### Permission Model
```
[Describe your RBAC or permission model]

Roles: owner > admin > member > viewer
Resource-level permissions: [who can do what]
```

---

## 7. Third-Party Integrations

| Integration | Purpose | Trigger | Notes |
|------------|---------|---------|-------|
| `[e.g., Stripe]` | Payments | User upgrades plan | Webhook handler at `/api/webhooks/stripe` |
| `[e.g., Resend]` | Transactional email | User signup, password reset | Templates in `/emails/` |
| `[e.g., Cloudflare R2]` | File uploads | User uploads asset | Signed URL pattern |

---

## 8. Performance & Scalability Considerations

### Caching Strategy
- `[e.g., Redis caches user sessions for 24h]`
- `[e.g., React Query staleTime: 5 min for read-heavy data]`
- `[e.g., CDN caching for static assets]`

### Database Performance
- Paginate all list endpoints — max `[50]` items per page
- Use cursor-based pagination for high-volume collections
- Index all foreign keys and commonly filtered fields

### Known Bottlenecks & Mitigations
| Potential Bottleneck | Mitigation |
|----------------------|-----------|
| `[e.g., Large file uploads]` | `[Multipart upload + direct-to-S3]` |
| `[e.g., Complex queries]` | `[Materialized views / denormalization]` |

---

## 9. Security Considerations

- **Input validation**: All inputs validated with `[e.g., Zod]` at the API boundary
- **SQL injection**: Prevented by ORM parameterized queries (never raw SQL with user input)
- **XSS**: React escapes output by default; CSP headers configured
- **CSRF**: `[e.g., SameSite cookie / CSRF tokens]`
- **Secrets**: All secrets in environment variables, never hardcoded
- **Rate limiting**: `[e.g., 100 req/min per IP on auth endpoints]`

---

## 10. Cursor Usage Notes

When using this file with Cursor, here are the most effective prompt patterns:

```
# Generate a new API endpoint
@design.md
Create a new PATCH /api/v1/projects/:id endpoint.
Follow the API conventions, error shapes, and auth patterns defined in design.md.
Use Prisma with the schema defined in the database section.

# Generate a new component
@design.md
Create a TaskCard component. 
It should be a UI component (stateless), using the data model from design.md.
Follow the component architecture rules.

# Generate a database migration
@design.md
Write a Prisma migration to add a `priority` field (enum: low, medium, high) 
to the tasks model. Update the schema in design.md accordingly.

# Validate a new design decision
@design.md @planning.md
I want to add real-time collaboration using WebSockets.
Does this conflict with our current architecture?
What changes to design.md would be required?
```
