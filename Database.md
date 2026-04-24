# database.md — Database Architecture & Management

> **Cursor instruction**: When this file is referenced in a prompt, treat it as the authoritative source of truth for *all data persistence decisions* — SQL schemas, NoSQL document structures, cache strategies, query patterns, migration rules, and connection management. Before writing any query, migration, cache call, or data access layer code, check this file. Never introduce a new database, collection, or cache key namespace without updating it here first.

---

## 0. Decision Map — Which Store to Use

> Use this map every time you need to persist or retrieve data. When in doubt, reference this before writing any storage code.

```
Is the data relational (users, orders, invoices)?
  └── YES → PostgreSQL (primary SQL store)

Is the data a flexible document / rapidly changing shape?
  └── YES → MongoDB (NoSQL document store)

Does the data need to be read extremely fast, repeatedly, and is it OK if it's slightly stale?
  └── YES → Redis (cache layer)

Is it a user session or ephemeral token?
  └── YES → Redis (TTL-keyed)

Is it a large binary (image, video, PDF)?
  └── YES → Object Storage (S3 / R2) — NOT the database

Is it a time-series / event log / analytics event?
  └── YES → [ClickHouse / TimescaleDB / dedicated analytics store]

Everything else?
  └── Default to PostgreSQL
```

---

## 1. SQL — PostgreSQL

### When to Use PostgreSQL
- Structured, relational data with clear foreign key relationships
- Data requiring ACID transactions (payments, inventory, orders)
- Data requiring complex joins, aggregations, or reporting queries
- Any data that is the **system of record** for users, accounts, and billing

### Connection & Pooling

```typescript
// lib/db/postgres.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const db = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
})

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

> **Rule**: Never instantiate `PrismaClient` more than once. Use the singleton above in all server-side code.

### Connection String Format
```
DATABASE_URL="postgresql://[user]:[password]@[host]:[port]/[dbname]?schema=public&connection_limit=10&pool_timeout=20"
```

| Parameter | Recommended | Notes |
|-----------|-------------|-------|
| `connection_limit` | `10` | Per instance; scale with replicas |
| `pool_timeout` | `20` | Seconds before timeout error |
| `statement_timeout` | `30000` | ms — set at DB level |
| `idle_in_transaction_session_timeout` | `60000` | ms — prevents hung transactions |

### Schema Conventions (Prisma)

```prisma
// schema.prisma conventions — follow these exactly

model User {
  id        String   @id @default(cuid())   // Always cuid() or uuid(), never serial int
  email     String   @unique
  name      String?
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  // Relations — always define both sides
  posts     Post[]

  @@map("users")   // DB table is snake_case, model is PascalCase
}

model Post {
  id        String   @id @default(cuid())
  title     String   @db.VarChar(255)
  content   String?  @db.Text
  status    Status   @default(DRAFT)
  authorId  String   @map("author_id")
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@index([authorId])          // Index all foreign keys
  @@index([status, createdAt]) // Compound index for common filter+sort combos
  @@map("posts")
}

enum Status {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

**Prisma Schema Rules:**
- IDs are always `cuid()` (collision-safe, URL-friendly) — never auto-increment integers as public IDs
- Every model has `createdAt` and `updatedAt`
- DB column names are `snake_case` via `@map()`; Prisma model fields are `camelCase`
- DB table names are `snake_case` via `@@map()`; Prisma model names are `PascalCase`
- Always index foreign key fields
- Use `@db.VarChar(n)` for bounded strings, `@db.Text` for unbounded

### Migration Workflow

```bash
# 1. Edit schema.prisma
# 2. Generate and apply migration
pnpm db:migrate        # = prisma migrate dev --name [description]

# Migration name conventions:
prisma migrate dev --name add_priority_to_tasks
prisma migrate dev --name create_notifications_table
prisma migrate dev --name drop_legacy_sessions

# 3. After schema change, regenerate client
pnpm db:generate       # = prisma generate

# Production migrations (CI/CD only — never run manually)
prisma migrate deploy
```

> **Rule**: Never edit a migration file after it has been committed. Create a new migration instead. Migration files are immutable history.

### Query Patterns

```typescript
// ✅ Paginated list query — always paginate
const tasks = await db.task.findMany({
  where: { workspaceId, status: { not: 'DELETED' } },
  orderBy: { createdAt: 'desc' },
  take: limit,          // max 50
  skip: (page - 1) * limit,
  select: {             // always select only needed fields
    id: true,
    title: true,
    status: true,
    assignee: { select: { id: true, name: true, avatarUrl: true } },
  },
})

// ✅ Transaction — for multi-step operations
const result = await db.$transaction(async (tx) => {
  const order = await tx.order.create({ data: orderData })
  await tx.inventory.update({ where: { id: itemId }, data: { stock: { decrement: 1 } } })
  return order
})

// ❌ Never do N+1 queries
// Bad:
const posts = await db.post.findMany()
for (const post of posts) {
  const author = await db.user.findUnique({ where: { id: post.authorId } }) // N queries!
}

// Good:
const posts = await db.post.findMany({ include: { author: true } }) // 1 query
```

### Indexing Strategy

```sql
-- Always index foreign keys (Prisma does NOT auto-index these)
CREATE INDEX idx_posts_author_id ON posts(author_id);

-- Compound index for common filter + sort
CREATE INDEX idx_tasks_workspace_status_created ON tasks(workspace_id, status, created_at DESC);

-- Partial index for filtered queries (more efficient than full index)
CREATE INDEX idx_tasks_active ON tasks(workspace_id, created_at DESC)
  WHERE status != 'DELETED';

-- Unique constraint (use @@unique in Prisma)
CREATE UNIQUE INDEX idx_memberships_user_workspace ON memberships(user_id, workspace_id);
```

> **Rule**: Add indexes in the Prisma schema via `@@index()`, not as raw SQL. This keeps migrations source-controlled.

---

## 2. NoSQL — MongoDB

### When to Use MongoDB
- Flexible or frequently changing document shapes (CMS content, user-generated forms)
- Hierarchical or nested data that doesn't map cleanly to relational tables
- Rapid iteration phases where the schema isn't settled
- High write-throughput event logs (if not using a dedicated analytics store)

> **Team rule**: MongoDB is a **secondary store**. PostgreSQL is the system of record for users, accounts, billing, and anything that requires relational integrity. Use MongoDB only when the document model is genuinely a better fit.

### Connection

```typescript
// lib/db/mongo.ts
import { MongoClient, ServerApiVersion } from 'mongodb'

const uri = process.env.MONGODB_URI!
const options = { serverApi: { version: ServerApiVersion.v1, strict: true } }

const globalForMongo = globalThis as unknown as { mongoClient: MongoClient }

export const mongoClient = globalForMongo.mongoClient ?? new MongoClient(uri, options)
if (process.env.NODE_ENV !== 'production') globalForMongo.mongoClient = mongoClient

export const mongoDB = mongoClient.db(process.env.MONGODB_DB_NAME!)
```

### Collection Naming Conventions
```
snake_case, plural nouns

✅  content_blocks
✅  form_submissions
✅  audit_logs
❌  ContentBlocks
❌  formSubmission
```

### Document Structure Standards

```typescript
// Every document must have this base shape
type BaseDocument = {
  _id: ObjectId                  // MongoDB auto-generated
  publicId: string               // cuid() — use this in APIs, never expose _id
  createdAt: Date
  updatedAt: Date
  schemaVersion: number          // increment when shape changes — enables migrations
}

// Example: CMS content block
type ContentBlock = BaseDocument & {
  workspaceId: string            // reference to PostgreSQL workspace.id
  type: 'text' | 'image' | 'video' | 'embed'
  title: string
  body: Record<string, unknown>  // flexible payload — varies by type
  tags: string[]
  publishedAt: Date | null
}
```

**MongoDB Document Rules:**
- Always include `publicId` (cuid) — never expose `_id` (ObjectId) in APIs
- Always include `schemaVersion` — this enables forward-compatible migrations
- Store cross-store references as strings (the PostgreSQL `id` value), never as ObjectIds
- Keep documents flat when possible — deep nesting makes querying hard
- Arrays inside documents are fine for small, bounded lists; for unbounded lists, use a separate collection

### Indexes (MongoDB)

```javascript
// Run once during setup or in a migration script
// lib/db/mongo-indexes.ts

await mongoDB.collection('content_blocks').createIndexes([
  { key: { workspaceId: 1, createdAt: -1 } },            // list by workspace
  { key: { publicId: 1 }, unique: true },                  // public ID lookup
  { key: { tags: 1 } },                                    // tag filter
  { key: { publishedAt: 1 }, sparse: true },               // published filter (sparse = skips nulls)
])
```

### Schema Evolution (Versioning)

```typescript
// When a document shape changes, increment schemaVersion
// and handle old versions in the read path

function normalizeContentBlock(raw: Record<string, unknown>): ContentBlock {
  const version = (raw.schemaVersion as number) ?? 1

  if (version === 1) {
    // v1 had `content` instead of `body`
    return { ...raw, body: raw.content, schemaVersion: 2 } as ContentBlock
  }

  return raw as ContentBlock
}
```

---

## 3. Cache — Redis

### When to Use Redis
| Use Case | TTL | Key Pattern |
|----------|-----|-------------|
| User session | 24h | `session:{sessionId}` |
| Auth token | 15min | `token:{userId}:{type}` |
| API response cache | 5min | `cache:{route}:{params-hash}` |
| Rate limit counter | 1min | `ratelimit:{ip}:{endpoint}` |
| Background job queue | no TTL | `queue:{jobType}` |
| Feature flags | 1h | `flags:{userId}` |
| Computed aggregates | 1h | `agg:{type}:{id}` |

### Connection

```typescript
// lib/db/redis.ts
import { Redis } from '@upstash/redis'   // or 'ioredis' for self-hosted

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
})

// OR — ioredis for self-hosted Redis
import IORedis from 'ioredis'
export const redis = new IORedis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: 3,
  enableReadyCheck: false,
})
```

### Key Design Rules

```
Format:    {namespace}:{entity}:{id}:{qualifier}
Separator: colon (:) — always

✅  session:usr_abc123
✅  cache:tasks:list:ws_xyz:page1
✅  ratelimit:192.168.1.1:api/auth/login
✅  agg:workspace:ws_abc:task_count
❌  sessionuserabc123           (no separator)
❌  Session:User:abc            (no uppercase)
❌  cache-tasks-list            (wrong separator)
```

**Rules:**
- All key components are lowercase
- IDs come from the canonical store (PostgreSQL cuid or MongoDB publicId)
- Always set a TTL — never store in Redis without expiry unless it's a queue
- Namespace every key — never store bare values like `set('userId', '123')`

### Cache Patterns

```typescript
// Pattern 1: Cache-aside (most common)
async function getWorkspaceTasks(workspaceId: string, page: number) {
  const cacheKey = `cache:tasks:${workspaceId}:page${page}`

  const cached = await redis.get<Task[]>(cacheKey)
  if (cached) return cached

  const tasks = await db.task.findMany({
    where: { workspaceId },
    take: 50,
    skip: (page - 1) * 50,
  })

  await redis.set(cacheKey, tasks, { ex: 300 }) // 5 min TTL
  return tasks
}

// Pattern 2: Cache invalidation on mutation
async function updateTask(taskId: string, workspaceId: string, data: UpdateTaskInput) {
  const task = await db.task.update({ where: { id: taskId }, data })

  // Invalidate related cache keys
  await redis.del(`cache:tasks:${workspaceId}:page1`)
  // For broad invalidation, use key scanning (use sparingly):
  // const keys = await redis.keys(`cache:tasks:${workspaceId}:*`)
  // if (keys.length) await redis.del(...keys)

  return task
}

// Pattern 3: Rate limiting
async function rateLimit(ip: string, endpoint: string, limit = 100): Promise<boolean> {
  const key = `ratelimit:${ip}:${endpoint}`
  const count = await redis.incr(key)
  if (count === 1) await redis.expire(key, 60) // 1 min window
  return count <= limit
}

// Pattern 4: Session storage
async function createSession(userId: string, sessionData: SessionPayload) {
  const sessionId = cuid()
  await redis.set(`session:${sessionId}`, JSON.stringify(sessionData), { ex: 86400 }) // 24h
  return sessionId
}
```

### What NOT to Cache
- User financial data or PII without encryption
- Data that must always be real-time (e.g. payment status mid-transaction)
- Data with no natural TTL — if you can't decide when it expires, it shouldn't be in Redis
- Large blobs (>1MB per key) — use object storage instead

---

## 4. Cross-Store Data Integrity

### Reference Pattern (SQL ↔ NoSQL)

When a MongoDB document needs to reference a PostgreSQL record:

```typescript
// ✅ Store the PostgreSQL ID as a plain string field
type ContentBlock = {
  workspaceId: string   // = PostgreSQL workspace.id (cuid string)
  authorId: string      // = PostgreSQL user.id (cuid string)
  ...
}

// ✅ Validate at the application layer — not the DB layer
async function createContentBlock(input: CreateContentBlockInput, userId: string) {
  // Verify the workspace exists in PostgreSQL before writing to MongoDB
  const workspace = await db.workspace.findUnique({ where: { id: input.workspaceId } })
  if (!workspace) throw new Error('WORKSPACE_NOT_FOUND')

  return mongoDB.collection('content_blocks').insertOne({
    ...input,
    authorId: userId,
    publicId: cuid(),
    schemaVersion: 2,
    createdAt: new Date(),
    updatedAt: new Date(),
  })
}
```

### Deletion Consistency

When deleting a PostgreSQL record that MongoDB documents reference:

```typescript
// Always clean up cross-store references in a single service function
async function deleteWorkspace(workspaceId: string) {
  // 1. Delete MongoDB documents first (no FK constraints)
  await mongoDB.collection('content_blocks').deleteMany({ workspaceId })

  // 2. Invalidate Redis cache
  const keys = await redis.keys(`cache:*:${workspaceId}:*`)
  if (keys.length) await redis.del(...keys)

  // 3. Delete PostgreSQL record last (CASCADE handles related SQL tables)
  await db.workspace.delete({ where: { id: workspaceId } })
}
```

### Cache Invalidation Ownership

Each service module owns the cache keys for its own data. No module may invalidate another module's keys directly — it must call that module's invalidation function.

```typescript
// tasks/cache.ts — TaskService owns these keys
export const taskCache = {
  key: (workspaceId: string, page: number) => `cache:tasks:${workspaceId}:page${page}`,
  invalidate: async (workspaceId: string) => {
    const keys = await redis.keys(`cache:tasks:${workspaceId}:*`)
    if (keys.length) await redis.del(...keys)
  },
}
```

---

## 5. Local Development Setup

```bash
# Start all local data stores via Docker
docker compose up -d

# docker-compose.yml provides:
#   PostgreSQL on localhost:5432
#   MongoDB    on localhost:27017
#   Redis      on localhost:6379
```

```yaml
# docker-compose.yml (reference)
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: appdb
    ports: ["5432:5432"]
    volumes: ["postgres_data:/var/lib/postgresql/data"]

  mongo:
    image: mongo:7
    ports: ["27017:27017"]
    volumes: ["mongo_data:/data/db"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    command: redis-server --appendonly yes
    volumes: ["redis_data:/data"]

volumes:
  postgres_data:
  mongo_data:
  redis_data:
```

### Environment Variables (database-related)
```bash
# PostgreSQL
DATABASE_URL="postgresql://dev:dev@localhost:5432/appdb"

# MongoDB
MONGODB_URI="mongodb://localhost:27017"
MONGODB_DB_NAME="appdb"

# Redis (local)
REDIS_URL="redis://localhost:6379"

# Redis (Upstash — production)
UPSTASH_REDIS_REST_URL="https://[your-endpoint].upstash.io"
UPSTASH_REDIS_REST_TOKEN="[your-token]"
```

---

## 6. Production Checklist

### Before Going Live
- [ ] Connection pooling configured (not default Prisma settings)
- [ ] `statement_timeout` and `idle_in_transaction_session_timeout` set at DB level
- [ ] All tables have `created_at` / `updated_at`
- [ ] All foreign keys are indexed
- [ ] No queries without `WHERE` clauses on large tables
- [ ] No `findMany()` without pagination (`take` / `skip`)
- [ ] Redis keys all have TTLs set
- [ ] MongoDB indexes created via setup script
- [ ] Backups configured and tested (restore drill done)
- [ ] DB credentials are different per environment (dev ≠ staging ≠ prod)
- [ ] Read replicas configured for heavy read workloads
- [ ] Slow query logging enabled

### Backup Strategy
| Store | Method | Frequency | Retention |
|-------|--------|-----------|-----------|
| PostgreSQL | `pg_dump` / managed snapshots | Daily + continuous WAL | 30 days |
| MongoDB | `mongodump` / Atlas backups | Daily | 14 days |
| Redis | RDB snapshots (AOF for critical data) | Hourly | 7 days |

---

## 7. Cursor Usage Notes

These prompt patterns work best with this file as context:

```
# Design a new data model
@database.md @design.md
I need to store user notification preferences.
Based on the decision map, which store should I use?
Design the schema/document structure following our conventions.

# Write a query
@database.md
Write a paginated Prisma query to fetch tasks for a workspace,
filtered by status, sorted by createdAt desc.
Follow the query patterns and always select only needed fields.

# Add caching to an endpoint
@database.md
Add Redis cache-aside caching to the getWorkspaceTasks function.
Follow the key naming conventions and cache invalidation ownership rules.

# Write a migration
@database.md
Write a Prisma migration to add a compound index on (workspace_id, status, created_at)
to the tasks table. Follow our indexing conventions.

# Review for database anti-patterns
@database.md
Review this code for N+1 queries, missing pagination, unindexed fields,
or incorrect cache key naming:
[paste code]

# Cross-store deletion
@database.md
Write a deleteUser service function that correctly cleans up
PostgreSQL records, MongoDB documents, and Redis cache keys
for a given userId.
```
