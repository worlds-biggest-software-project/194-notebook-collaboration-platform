# Notebook Collaboration Platform — Phased Development Plan

> Project: 194-notebook-collaboration-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | TypeScript (Node.js) | Real-time collaboration via WebSocket/Yjs runs natively in the JS runtime; shared types between frontend and backend reduce impedance mismatch for the CRDT layer; the Yjs ecosystem is TypeScript-first |
| Primary language (frontend) | TypeScript (React) | React dominates the notebook UI space (JupyterLab uses Lumino/React); rich component ecosystem for code editors (CodeMirror 6), drag-and-drop layouts, and presence indicators |
| API framework | Fastify | High-performance Node.js framework with first-class TypeScript support, built-in JSON Schema validation (aligns with OpenAPI 3.1 generation via @fastify/swagger), and WebSocket plugin for Yjs sync |
| Database | PostgreSQL 16 | JSONB support enables the Hybrid Relational + JSONB data model (Suggestion 3) where notebook content is stored as native nbformat JSON; generated columns, GIN indexes, and Row-Level Security for multi-tenancy |
| Real-time sync | Yjs + y-websocket | De facto CRDT standard adopted by JupyterLab's jupyter-collaboration extension; BSD-licensed; supports WebSocket transport with awareness protocol for presence indicators |
| Code editor | CodeMirror 6 | Used by JupyterLab and multiple notebook platforms; supports Yjs binding via y-codemirror.next; language modes for Python, SQL, Markdown; extensible for AI completions |
| Task queue | BullMQ (Redis) | Handles async workloads: scheduled notebook execution, AI completion requests, export jobs, notification delivery; Redis also serves as Yjs persistence/pubsub for horizontal scaling |
| Authentication | Passport.js + OIDC | Modular auth with OAuth 2.0 / OpenID Connect strategies for enterprise SSO (Google, GitHub, SAML); JWT session tokens with refresh rotation |
| Kernel execution | Jupyter Kernel Gateway | Reuses the Jupyter kernel protocol (ZMQ) via HTTP/WebSocket; avoids reimplementing kernel management; supports Python, R, Julia kernels via standard Jupyter kernel specs |
| Container runtime | Docker + Docker Compose | Self-hosted deployment target; reproducible environments via repo2docker pattern; kernel isolation via per-user containers |
| ORM / query builder | Drizzle ORM | TypeScript-native, SQL-first ORM with excellent PostgreSQL JSONB support; generates migrations; type-safe queries without hiding SQL |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest is fast, ESM-native, and compatible with the TypeScript toolchain; Playwright tests the full notebook UI including real-time collaboration |
| Code quality | ESLint + Prettier + tsc --noEmit | Standard TypeScript toolchain; enforced via pre-commit hooks and CI |
| Package manager | pnpm | Efficient monorepo support with workspace protocol; strict dependency resolution avoids phantom deps |
| Monorepo | pnpm workspaces + Turborepo | Separate packages for server, client, shared types, and kernel gateway; Turborepo handles build/test orchestration with caching |
| Notebook format | nbformat v4 via JSONB | Notebook content stored as native nbformat v4 JSON in PostgreSQL JSONB column; zero-transformation import/export; validated against nbformat JSON Schema at the application layer |

### Project Structure

```
notebook-collab/
├── pnpm-workspace.yaml
├── turbo.json
├── package.json
├── docker-compose.yml
├── docker-compose.dev.yml
├── Dockerfile
├── .env.example
├── packages/
│   ├── shared/                          # Shared types and utilities
│   │   ├── package.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── notebook.ts          # nbformat-aligned TypeScript types
│   │       │   ├── user.ts
│   │       │   ├── workspace.ts
│   │       │   ├── execution.ts
│   │       │   ├── collaboration.ts
│   │       │   └── api.ts               # Request/response schemas
│   │       ├── constants.ts
│   │       └── validation/
│   │           └── nbformat-schema.ts   # nbformat v4 JSON Schema validator
│   ├── server/                          # Fastify backend
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── app.ts                   # Fastify app factory
│   │   │   ├── server.ts                # Entry point
│   │   │   ├── config.ts                # Env-based configuration
│   │   │   ├── db/
│   │   │   │   ├── schema.ts            # Drizzle schema definitions
│   │   │   │   ├── migrations/          # Generated SQL migrations
│   │   │   │   └── seed.ts              # Development seed data
│   │   │   ├── routes/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── workspaces.ts
│   │   │   │   ├── projects.ts
│   │   │   │   ├── notebooks.ts
│   │   │   │   ├── cells.ts
│   │   │   │   ├── execution.ts
│   │   │   │   ├── schedules.ts
│   │   │   │   ├── shares.ts
│   │   │   │   ├── comments.ts
│   │   │   │   ├── connections.ts
│   │   │   │   ├── dashboards.ts
│   │   │   │   └── ai.ts
│   │   │   ├── services/
│   │   │   │   ├── notebook.service.ts
│   │   │   │   ├── execution.service.ts
│   │   │   │   ├── kernel.service.ts
│   │   │   │   ├── scheduling.service.ts
│   │   │   │   ├── ai-completion.service.ts
│   │   │   │   ├── version.service.ts
│   │   │   │   ├── share.service.ts
│   │   │   │   ├── export.service.ts
│   │   │   │   └── notification.service.ts
│   │   │   ├── collaboration/
│   │   │   │   ├── yjs-server.ts        # Yjs WebSocket server
│   │   │   │   ├── persistence.ts       # Yjs ↔ PostgreSQL sync
│   │   │   │   └── awareness.ts         # Presence/cursor tracking
│   │   │   ├── queue/
│   │   │   │   ├── workers/
│   │   │   │   │   ├── execution.worker.ts
│   │   │   │   │   ├── ai.worker.ts
│   │   │   │   │   ├── export.worker.ts
│   │   │   │   │   └── notification.worker.ts
│   │   │   │   └── queues.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── workspace-scope.ts
│   │   │   │   ├── rate-limit.ts
│   │   │   │   └── error-handler.ts
│   │   │   └── plugins/
│   │   │       ├── swagger.ts           # OpenAPI 3.1 spec generation
│   │   │       └── websocket.ts
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── fixtures/
│   ├── client/                          # React frontend
│   │   ├── package.json
│   │   ├── vite.config.ts
│   │   ├── index.html
│   │   └── src/
│   │       ├── main.tsx
│   │       ├── App.tsx
│   │       ├── components/
│   │       │   ├── notebook/
│   │       │   │   ├── NotebookEditor.tsx
│   │       │   │   ├── Cell.tsx
│   │       │   │   ├── CodeCell.tsx
│   │       │   │   ├── MarkdownCell.tsx
│   │       │   │   ├── SqlCell.tsx
│   │       │   │   ├── CellOutput.tsx
│   │       │   │   ├── CellToolbar.tsx
│   │       │   │   └── AddCellButton.tsx
│   │       │   ├── collaboration/
│   │       │   │   ├── PresenceIndicator.tsx
│   │       │   │   ├── CursorOverlay.tsx
│   │       │   │   └── CollaboratorList.tsx
│   │       │   ├── editor/
│   │       │   │   ├── CodeEditor.tsx   # CodeMirror 6 wrapper
│   │       │   │   └── MarkdownRenderer.tsx
│   │       │   ├── sidebar/
│   │       │   │   ├── ProjectTree.tsx
│   │       │   │   ├── VersionHistory.tsx
│   │       │   │   └── CommentPanel.tsx
│   │       │   ├── dashboard/
│   │       │   │   ├── DashboardBuilder.tsx
│   │       │   │   └── DashboardViewer.tsx
│   │       │   └── layout/
│   │       │       ├── Header.tsx
│   │       │       ├── Sidebar.tsx
│   │       │       └── WorkspaceSelector.tsx
│   │       ├── hooks/
│   │       │   ├── useYjsProvider.ts
│   │       │   ├── useNotebook.ts
│   │       │   ├── usePresence.ts
│   │       │   ├── useKernel.ts
│   │       │   └── useAiCompletion.ts
│   │       ├── stores/
│   │       │   ├── auth.store.ts
│   │       │   ├── workspace.store.ts
│   │       │   └── notebook.store.ts
│   │       └── api/
│   │           └── client.ts            # Generated from OpenAPI spec
│   └── kernel-gateway/                  # Kernel management sidecar
│       ├── Dockerfile
│       ├── requirements.txt
│       └── config/
│           └── kernel_gateway_config.py
├── e2e/                                 # Playwright E2E tests
│   ├── playwright.config.ts
│   └── tests/
│       ├── notebook-editing.spec.ts
│       ├── collaboration.spec.ts
│       ├── execution.spec.ts
│       └── sharing.spec.ts
└── scripts/
    ├── dev.sh                           # Start dev environment
    ├── migrate.sh                       # Run database migrations
    └── seed.sh                          # Seed development data
```

---

## Phase 1: Foundation — Project Scaffolding, Database, and Auth

### Purpose

Establish the monorepo structure, database schema, authentication system, and development environment. After this phase, developers can start the application, authenticate via local login, and interact with workspace/user APIs. Every subsequent phase builds on this foundation.

### Tasks

#### 1.1 — Monorepo Scaffolding and Dev Environment

**What**: Set up the pnpm workspace with Turborepo, shared package, server package, and client package with all tooling configured.

**Design**:

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
  - "e2e"
```

Root `turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": {}
  }
}
```

Shared `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

`docker-compose.dev.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: notebook
      POSTGRES_PASSWORD: notebook_dev
      POSTGRES_DB: notebook_collab
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  pgdata:
```

**Testing**:
- `Unit: pnpm install succeeds with no peer dependency warnings`
- `Unit: turbo build compiles all packages without errors`
- `Unit: turbo typecheck passes across all packages`
- `Unit: turbo lint passes with zero warnings`
- `Integration: docker-compose.dev.yml starts PostgreSQL and Redis, both accept connections`

---

#### 1.2 — Database Schema and Migrations

**What**: Define the Drizzle ORM schema implementing the Hybrid Relational + JSONB data model (based on Data Model Suggestion 3) and generate the initial migration.

**Design**:

Core schema in `packages/server/src/db/schema.ts`:

```typescript
import { pgTable, uuid, varchar, text, timestamp, jsonb, integer, boolean, uniqueIndex, index } from 'drizzle-orm/pg-core';

// ─── Identity & Multi-Tenancy ───

export const workspaces = pgTable('workspaces', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull().unique(),
  plan: varchar('plan', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  avatarUrl: text('avatar_url'),
  authProvider: varchar('auth_provider', { length: 50 }).notNull().default('local'),
  authProviderId: varchar('auth_provider_id', { length: 255 }),
  passwordHash: varchar('password_hash', { length: 255 }),
  profile: jsonb('profile').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const workspaceMemberships = pgTable('workspace_memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('editor'),
  permissions: jsonb('permissions').notNull().default({}),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueMembership: uniqueIndex('unique_workspace_user').on(table.workspaceId, table.userId),
  workspaceIdx: index('idx_ws_memberships_workspace').on(table.workspaceId),
  userIdx: index('idx_ws_memberships_user').on(table.userId),
}));

export const apiTokens = pgTable('api_tokens', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  workspaceId: uuid('workspace_id').references(() => workspaces.id, { onDelete: 'cascade' }),
  tokenHash: varchar('token_hash', { length: 64 }).notNull().unique(),
  name: varchar('name', { length: 255 }).notNull(),
  scopes: jsonb('scopes').notNull().default([]),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// ─── Projects & Notebooks ───

export const projects = pgTable('projects', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  settings: jsonb('settings').notNull().default({}),
  createdBy: uuid('created_by').notNull().references(() => users.id),
  archivedAt: timestamp('archived_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceIdx: index('idx_projects_workspace').on(table.workspaceId),
}));

export const notebooks = pgTable('notebooks', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  content: jsonb('content').notNull().default({
    nbformat: 4,
    nbformat_minor: 5,
    metadata: {
      kernelspec: { display_name: 'Python 3', language: 'python', name: 'python3' },
      language_info: { name: 'python', version: '3.11' },
    },
    cells: [],
  }),
  createdBy: uuid('created_by').notNull().references(() => users.id),
  lastEditedBy: uuid('last_edited_by').references(() => users.id),
  lastEditedAt: timestamp('last_edited_at', { withTimezone: true }),
  tags: text('tags').array().notNull().default([]),
  platformMetadata: jsonb('platform_metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  projectIdx: index('idx_notebooks_project').on(table.projectId),
}));
```

Additional tables for versions, execution, comments, collaboration, sharing, connections, dashboards, and AI interactions follow the same pattern from Data Model Suggestion 3 (15 tables total).

Drizzle config (`drizzle.config.ts`):
```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './src/db/migrations',
  driver: 'pg',
  dbCredentials: {
    connectionString: process.env.DATABASE_URL!,
  },
} satisfies Config;
```

**Testing**:
- `Unit: Drizzle schema compiles without TypeScript errors`
- `Integration: drizzle-kit generate produces a valid SQL migration`
- `Integration: migration applies cleanly to a fresh PostgreSQL database`
- `Integration: migration is idempotent — running twice produces no errors`
- `Integration: seed script inserts a workspace, user, project, and empty notebook`
- `Unit: inserting a notebook with valid nbformat v4 JSON succeeds`
- `Unit: inserting a notebook with invalid JSON content fails validation`

---

#### 1.3 — Configuration and Environment Management

**What**: Create a type-safe configuration module that reads from environment variables with validation and sensible defaults.

**Design**:

```typescript
// packages/server/src/config.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default('0.0.0.0'),

  // Database
  DATABASE_URL: z.string().url(),

  // Redis
  REDIS_URL: z.string().url().default('redis://localhost:6379'),

  // Auth
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRY: z.string().default('15m'),
  REFRESH_TOKEN_EXPIRY: z.string().default('7d'),
  BCRYPT_ROUNDS: z.coerce.number().default(12),

  // OAuth (optional, for SSO)
  GOOGLE_CLIENT_ID: z.string().optional(),
  GOOGLE_CLIENT_SECRET: z.string().optional(),
  GITHUB_CLIENT_ID: z.string().optional(),
  GITHUB_CLIENT_SECRET: z.string().optional(),

  // Kernel Gateway
  KERNEL_GATEWAY_URL: z.string().url().default('http://localhost:8888'),

  // AI
  ANTHROPIC_API_KEY: z.string().optional(),
  AI_MODEL: z.string().default('claude-sonnet-4'),

  // App
  CORS_ORIGIN: z.string().default('http://localhost:5173'),
  MAX_NOTEBOOK_SIZE_MB: z.coerce.number().default(50),
  MAX_CELL_OUTPUT_SIZE_MB: z.coerce.number().default(10),
});

export type AppConfig = z.infer<typeof envSchema>;
export const config: AppConfig = envSchema.parse(process.env);
```

**Testing**:
- `Unit: valid complete env → AppConfig with all fields populated`
- `Unit: missing DATABASE_URL → ZodError with field name`
- `Unit: PORT='abc' → ZodError (coerce fails)`
- `Unit: missing optional fields → defaults applied correctly`
- `Unit: JWT_SECRET shorter than 32 chars → ZodError`

---

#### 1.4 — Authentication and User Management

**What**: Implement local email/password registration and login with JWT access tokens and refresh token rotation. Set up Passport.js middleware for future OAuth/OIDC expansion.

**Design**:

```typescript
// packages/shared/src/types/user.ts
export interface UserProfile {
  id: string;
  email: string;
  displayName: string;
  avatarUrl: string | null;
  authProvider: 'local' | 'google' | 'github' | 'saml';
}

export interface AuthTokens {
  accessToken: string;   // JWT, 15min expiry
  refreshToken: string;  // opaque, 7d expiry, stored hashed in DB
}

export interface RegisterRequest {
  email: string;
  password: string;
  displayName: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface RefreshRequest {
  refreshToken: string;
}
```

API routes:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/auth/register` | `RegisterRequest` | `{ user: UserProfile, tokens: AuthTokens }` |
| POST | `/api/auth/login` | `LoginRequest` | `{ user: UserProfile, tokens: AuthTokens }` |
| POST | `/api/auth/refresh` | `RefreshRequest` | `{ tokens: AuthTokens }` |
| POST | `/api/auth/logout` | (cookie/header) | `204 No Content` |
| GET | `/api/auth/me` | (JWT in header) | `UserProfile` |

JWT payload:
```typescript
interface JwtPayload {
  sub: string;        // user ID
  email: string;
  iat: number;
  exp: number;
}
```

Password hashing: bcrypt with configurable rounds (default 12). Refresh tokens stored as SHA-256 hashes in a `refresh_tokens` table with device/IP tracking and one-time-use enforcement (rotation).

**Testing**:
- `Unit: register with valid email/password → user created, tokens returned, password hashed`
- `Unit: register with duplicate email → 409 Conflict`
- `Unit: register with weak password (< 8 chars) → 400 Bad Request`
- `Unit: login with correct credentials → tokens returned`
- `Unit: login with wrong password → 401 Unauthorized`
- `Unit: login with non-existent email → 401 Unauthorized (no user enumeration)`
- `Unit: access protected route with valid JWT → 200`
- `Unit: access protected route with expired JWT → 401`
- `Unit: refresh with valid refresh token → new access + refresh tokens, old refresh token invalidated`
- `Unit: refresh with already-used refresh token → 401, all user sessions revoked (rotation violation detection)`
- `Integration: full register → login → access → refresh → access flow`

---

#### 1.5 — Workspace and Membership CRUD

**What**: Implement workspace creation, listing, and membership management endpoints with role-based access control.

**Design**:

```typescript
// packages/shared/src/types/workspace.ts
export interface Workspace {
  id: string;
  name: string;
  slug: string;
  plan: 'free' | 'team' | 'enterprise';
  settings: WorkspaceSettings;
  createdAt: string;
}

export interface WorkspaceSettings {
  defaultKernel?: string;
  maxConcurrentExecutions?: number;
  allowedDataConnectors?: string[];
}

export interface WorkspaceMember {
  userId: string;
  email: string;
  displayName: string;
  role: 'viewer' | 'editor' | 'admin' | 'owner';
  joinedAt: string;
}

export type WorkspaceRole = 'viewer' | 'editor' | 'admin' | 'owner';
```

API routes:

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/api/workspaces` | JWT | `Workspace` |
| GET | `/api/workspaces` | JWT | `Workspace[]` (user's memberships) |
| GET | `/api/workspaces/:slug` | JWT + member | `Workspace` |
| PATCH | `/api/workspaces/:slug` | JWT + admin | `Workspace` |
| POST | `/api/workspaces/:slug/members` | JWT + admin | `WorkspaceMember` |
| GET | `/api/workspaces/:slug/members` | JWT + member | `WorkspaceMember[]` |
| PATCH | `/api/workspaces/:slug/members/:userId` | JWT + admin | `WorkspaceMember` |
| DELETE | `/api/workspaces/:slug/members/:userId` | JWT + admin/owner | `204` |

Middleware: `workspace-scope.ts` extracts workspace from URL, verifies membership, and attaches role to the request context.

**Testing**:
- `Unit: create workspace → slug auto-generated from name, creator added as owner`
- `Unit: create workspace with duplicate slug → 409 Conflict`
- `Unit: list workspaces → returns only workspaces where user is a member`
- `Unit: add member → membership created, invitation sent`
- `Unit: viewer tries to add member → 403 Forbidden`
- `Unit: admin changes member role → role updated`
- `Unit: owner cannot be removed → 400 Bad Request`
- `Unit: remove member → membership deleted`
- `Integration: create workspace → add member → member sees workspace in list`

---

## Phase 2: Notebook CRUD and .ipynb Import/Export

### Purpose

Deliver the core notebook data model: creating, reading, updating, and deleting notebooks within projects, plus full `.ipynb` import/export. After this phase, users can manage notebooks programmatically (API-only) with content stored as native nbformat v4 JSON.

### Tasks

#### 2.1 — Project CRUD

**What**: Implement project management within workspaces (projects are the grouping unit for notebooks).

**Design**:

```typescript
// packages/shared/src/types/project.ts (subset)
export interface Project {
  id: string;
  workspaceId: string;
  name: string;
  description: string | null;
  settings: ProjectSettings;
  createdBy: string;
  archivedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface ProjectSettings {
  defaultEnvironment?: string;
  gitRepo?: { url: string; branch: string };
  sharedConnections?: string[];
}
```

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/api/workspaces/:slug/projects` | JWT + editor | `Project` |
| GET | `/api/workspaces/:slug/projects` | JWT + member | `Project[]` |
| GET | `/api/workspaces/:slug/projects/:projectId` | JWT + member | `Project` |
| PATCH | `/api/workspaces/:slug/projects/:projectId` | JWT + editor | `Project` |
| DELETE | `/api/workspaces/:slug/projects/:projectId` | JWT + admin | `204` |

**Testing**:
- `Unit: create project → project created with workspace scope`
- `Unit: viewer cannot create project → 403`
- `Unit: list projects → returns only projects in the scoped workspace`
- `Unit: delete project cascades to notebooks`
- `Integration: create project → list → verify it appears`

---

#### 2.2 — Notebook CRUD with nbformat Content

**What**: Implement notebook creation, retrieval, update, and deletion. The `content` column stores native nbformat v4 JSON. Updates replace the full content (cell-level operations come in Phase 4 via Yjs).

**Design**:

```typescript
// packages/shared/src/types/notebook.ts
export interface NotebookSummary {
  id: string;
  projectId: string;
  name: string;
  kernelName: string;
  language: string;
  cellCount: number;
  tags: string[];
  createdBy: string;
  lastEditedBy: string | null;
  lastEditedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface NotebookFull extends NotebookSummary {
  content: NbformatDocument;
  platformMetadata: Record<string, unknown>;
}

// nbformat v4 types
export interface NbformatDocument {
  nbformat: 4;
  nbformat_minor: number;
  metadata: NbformatMetadata;
  cells: NbformatCell[];
}

export interface NbformatMetadata {
  kernelspec: {
    display_name: string;
    language: string;
    name: string;
  };
  language_info: {
    name: string;
    version?: string;
    mimetype?: string;
    file_extension?: string;
  };
  [key: string]: unknown;
}

export type NbformatCell = NbformatCodeCell | NbformatMarkdownCell | NbformatRawCell;

export interface NbformatCodeCell {
  id: string;
  cell_type: 'code';
  source: string;
  metadata: Record<string, unknown>;
  execution_count: number | null;
  outputs: NbformatOutput[];
}

export interface NbformatMarkdownCell {
  id: string;
  cell_type: 'markdown';
  source: string;
  metadata: Record<string, unknown>;
}

export interface NbformatRawCell {
  id: string;
  cell_type: 'raw';
  source: string;
  metadata: Record<string, unknown>;
}

export type NbformatOutput =
  | NbformatExecuteResult
  | NbformatDisplayData
  | NbformatStreamOutput
  | NbformatErrorOutput;

export interface NbformatExecuteResult {
  output_type: 'execute_result';
  execution_count: number;
  data: Record<string, string>;     // MIME-type → content
  metadata: Record<string, unknown>;
}

export interface NbformatDisplayData {
  output_type: 'display_data';
  data: Record<string, string>;
  metadata: Record<string, unknown>;
}

export interface NbformatStreamOutput {
  output_type: 'stream';
  name: 'stdout' | 'stderr';
  text: string;
}

export interface NbformatErrorOutput {
  output_type: 'error';
  ename: string;
  evalue: string;
  traceback: string[];
}
```

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/api/workspaces/:slug/projects/:projectId/notebooks` | JWT + editor | `NotebookFull` |
| GET | `/api/workspaces/:slug/projects/:projectId/notebooks` | JWT + member | `NotebookSummary[]` |
| GET | `/api/workspaces/:slug/notebooks/:notebookId` | JWT + member | `NotebookFull` |
| PUT | `/api/workspaces/:slug/notebooks/:notebookId` | JWT + editor | `NotebookFull` |
| DELETE | `/api/workspaces/:slug/notebooks/:notebookId` | JWT + admin | `204` |

Validation: incoming `content` validated against nbformat v4 JSON Schema using Ajv.

**Testing**:
- `Unit: create notebook → empty notebook with valid nbformat structure`
- `Unit: create notebook with custom kernelspec → stored correctly`
- `Unit: update notebook content → content replaced, updatedAt bumped`
- `Unit: update with invalid nbformat → 400 with validation errors`
- `Unit: notebook summary excludes content field`
- `Unit: delete notebook → 204, subsequent GET returns 404`
- `Fixture: validate 5 real-world .ipynb files parse correctly as NbformatDocument`

---

#### 2.3 — .ipynb Import and Export

**What**: Import `.ipynb` files by uploading JSON, and export notebooks as `.ipynb` downloads. The content column IS the .ipynb, so export is trivial.

**Design**:

```typescript
// Import: POST multipart/form-data
// packages/server/src/services/notebook.service.ts

interface ImportResult {
  notebook: NotebookFull;
  warnings: string[];  // e.g., "nbformat_minor 4 upgraded to 5"
}

async function importNotebook(
  projectId: string,
  file: Buffer,
  fileName: string,
  userId: string,
): Promise<ImportResult>;

// Export: GET with Accept header
// GET /api/workspaces/:slug/notebooks/:notebookId/export
// Response: application/json with Content-Disposition: attachment; filename="notebook.ipynb"

async function exportNotebook(
  notebookId: string,
): Promise<{ content: NbformatDocument; fileName: string }>;
```

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/api/workspaces/:slug/projects/:projectId/notebooks/import` | JWT + editor | `ImportResult` |
| GET | `/api/workspaces/:slug/notebooks/:notebookId/export` | JWT + member | `.ipynb` file download |

Import flow: parse JSON → validate against nbformat schema → normalize (upgrade nbformat_minor if needed, assign cell IDs if missing) → insert as notebook.

**Testing**:
- `Unit: import valid .ipynb → notebook created, content matches input`
- `Unit: import .ipynb with missing cell IDs → IDs auto-generated (UUID)`
- `Unit: import .ipynb with nbformat 3 → rejected with 400 (only v4 supported)`
- `Unit: import non-JSON file → 400 Bad Request`
- `Unit: import oversized file (> MAX_NOTEBOOK_SIZE_MB) → 413`
- `Unit: export notebook → valid .ipynb JSON, Content-Disposition header set`
- `Fixture: round-trip test — import fixtures/sample.ipynb → export → compare, assert identical structure`
- `Fixture: import 3 real-world .ipynb files from different tools (Colab, JupyterLab, Deepnote)`

---

## Phase 3: Notebook UI — Editor, Cells, and CodeMirror

### Purpose

Build the frontend notebook editor: a cell-based UI with CodeMirror 6 for code editing, Markdown rendering, and cell management (add, delete, reorder). After this phase, users can visually create and edit notebooks in the browser, though without real-time collaboration or kernel execution.

### Tasks

#### 3.1 — React App Shell and Routing

**What**: Set up the Vite-based React app with routing, auth state management, workspace/project navigation, and a responsive layout.

**Design**:

Routes:
```typescript
// packages/client/src/App.tsx
const routes = [
  { path: '/login', element: <LoginPage /> },
  { path: '/register', element: <RegisterPage /> },
  { path: '/', element: <Navigate to="/workspaces" /> },
  { path: '/workspaces', element: <WorkspaceListPage /> },
  { path: '/w/:slug', element: <WorkspaceLayout />, children: [
    { path: '', element: <WorkspaceDashboard /> },
    { path: 'projects/:projectId', element: <ProjectPage /> },
    { path: 'notebooks/:notebookId', element: <NotebookPage /> },
    { path: 'settings', element: <WorkspaceSettingsPage /> },
  ]},
];
```

State management: Zustand stores for auth, workspace, and notebook state. API client auto-generated from the OpenAPI spec using `openapi-typescript-fetch`.

**Testing**:
- `E2E: unauthenticated user visits / → redirected to /login`
- `E2E: login → redirected to /workspaces`
- `E2E: navigate to workspace → project list displayed`
- `Unit: auth store handles login/logout/token refresh`
- `Unit: workspace store fetches and caches workspace data`

---

#### 3.2 — Notebook Editor Component

**What**: Build the main notebook editor that renders cells in order, supports cell selection, and manages cell CRUD operations.

**Design**:

```typescript
// packages/client/src/components/notebook/NotebookEditor.tsx
interface NotebookEditorProps {
  notebook: NotebookFull;
  onSave: (content: NbformatDocument) => Promise<void>;
  readOnly?: boolean;
}

// Cell state managed via React context
interface NotebookEditorState {
  cells: NbformatCell[];
  selectedCellId: string | null;
  isModified: boolean;
  addCell: (type: 'code' | 'markdown' | 'raw', position: number) => void;
  deleteCell: (cellId: string) => void;
  moveCell: (cellId: string, direction: 'up' | 'down') => void;
  updateCellSource: (cellId: string, source: string) => void;
  updateCellType: (cellId: string, type: 'code' | 'markdown' | 'raw') => void;
}
```

Cell toolbar actions: Run (disabled until Phase 5), Delete, Move Up, Move Down, Change Type, Add Cell Below.

Keyboard shortcuts:
- `Ctrl+S` / `Cmd+S`: Save notebook
- `Ctrl+Enter`: Run cell (Phase 5)
- `Shift+Enter`: Run cell and advance (Phase 5)
- `A`: Add cell above (command mode)
- `B`: Add cell below (command mode)
- `DD`: Delete cell (command mode)
- `M`: Change to Markdown (command mode)
- `Y`: Change to Code (command mode)

**Testing**:
- `E2E: open notebook → cells rendered in order`
- `E2E: click Add Cell → new empty cell appears at correct position`
- `E2E: delete cell → cell removed, remaining cells re-ordered`
- `E2E: move cell up → cell position swapped`
- `E2E: Cmd+S → save request sent, "Saved" indicator shown`
- `Unit: addCell generates a UUID for the new cell`
- `Unit: deleteCell removes cell and updates positions`
- `Unit: moveCell swaps positions correctly at boundaries (first/last cell)`

---

#### 3.3 — CodeMirror 6 Code Cell Editor

**What**: Integrate CodeMirror 6 as the code editor within code cells, with Python and SQL syntax highlighting, line numbers, and bracket matching.

**Design**:

```typescript
// packages/client/src/components/editor/CodeEditor.tsx
interface CodeEditorProps {
  value: string;
  language: 'python' | 'sql' | 'markdown';
  onChange: (value: string) => void;
  onRun?: () => void;                    // Ctrl+Enter handler
  readOnly?: boolean;
  lineNumbers?: boolean;                 // default true
  minHeight?: string;                    // default '3em'
}
```

CodeMirror extensions:
- `@codemirror/lang-python` for Python
- `@codemirror/lang-sql` for SQL
- `@codemirror/lang-markdown` for Markdown cells in edit mode
- `@codemirror/view` keymap with custom notebook-aware bindings
- Theme: light/dark mode support via CSS variables

**Testing**:
- `E2E: type Python code → syntax highlighting applied`
- `E2E: type SQL code → SQL keywords highlighted`
- `E2E: bracket autocomplete works (type '(' → ')' inserted)`
- `Unit: CodeEditor onChange fires on every keystroke`
- `Unit: readOnly mode prevents editing`
- `Unit: onRun called on Ctrl+Enter`

---

#### 3.4 — Markdown Cell Rendering

**What**: Render Markdown cells with a toggle between edit mode (CodeMirror with Markdown syntax) and rendered mode (HTML via remark/rehype).

**Design**:

```typescript
// packages/client/src/components/notebook/MarkdownCell.tsx
interface MarkdownCellProps {
  cell: NbformatMarkdownCell;
  onUpdate: (source: string) => void;
  isSelected: boolean;
}
```

Rendering pipeline: `source` → remark → rehype → sanitize (rehype-sanitize) → React elements. Supports GFM (GitHub Flavored Markdown): tables, task lists, strikethrough, autolinks.

Double-click rendered Markdown to enter edit mode. Click outside or press Escape to return to rendered mode.

LaTeX/math support via `rehype-katex` and `remark-math`.

**Testing**:
- `E2E: Markdown cell displays rendered HTML by default`
- `E2E: double-click → enters edit mode with CodeMirror`
- `E2E: Escape → returns to rendered mode with updated content`
- `Unit: renders headings, bold, italic, code blocks, links`
- `Unit: renders GFM tables`
- `Unit: renders LaTeX math expressions`
- `Unit: HTML is sanitized (script tags stripped)`

---

#### 3.5 — Cell Output Display

**What**: Render cell outputs from the nbformat structure: text/plain, text/html, image/png, error tracebacks, and stream output.

**Design**:

```typescript
// packages/client/src/components/notebook/CellOutput.tsx
interface CellOutputProps {
  outputs: NbformatOutput[];
}
```

Output rendering by type:
- `execute_result` / `display_data`: render richest MIME type available. Priority: `text/html` > `image/png` > `image/svg+xml` > `text/plain`.
- `stream` (stdout/stderr): monospace pre-formatted text, stderr in red.
- `error`: formatted traceback with ANSI color code parsing (via `ansi-to-html`).

HTML outputs sandboxed in an iframe with `sandbox="allow-scripts"` to prevent XSS from cell outputs.

**Testing**:
- `Unit: text/plain output → rendered in <pre> tag`
- `Unit: text/html output → rendered in sandboxed iframe`
- `Unit: image/png output → rendered as <img> with base64 src`
- `Unit: error output → traceback rendered with ANSI colors`
- `Unit: stream stdout → displayed as pre-formatted text`
- `Unit: stream stderr → displayed in red`
- `Unit: multiple outputs → rendered in order`
- `Fixture: render outputs from fixtures/complex-outputs.ipynb`

---

## Phase 4: Real-Time Collaboration with Yjs

### Purpose

Enable multiplayer notebook editing using Yjs CRDTs. Multiple users can simultaneously edit different cells, see each other's cursors, and have changes merged conflict-free. After this phase, the platform's core differentiating feature — real-time collaboration — is functional.

### Tasks

#### 4.1 — Yjs WebSocket Server

**What**: Implement a Yjs WebSocket provider on the server that manages Y.Doc instances for each notebook, persists CRDT state to PostgreSQL, and handles client connections.

**Design**:

```typescript
// packages/server/src/collaboration/yjs-server.ts
import * as Y from 'yjs';
import { WebSocketServer } from 'ws';

interface YjsServerConfig {
  persistenceInterval: number;   // ms between flushes to DB (default: 5000)
  gcEnabled: boolean;            // Yjs garbage collection (default: true)
  maxConnections: number;        // per document (default: 50)
}

class YjsCollaborationServer {
  private docs: Map<string, Y.Doc>;       // notebookId → Y.Doc
  private connections: Map<string, Set<WebSocket>>;

  async handleConnection(ws: WebSocket, notebookId: string, userId: string): Promise<void>;
  async loadDocument(notebookId: string): Promise<Y.Doc>;
  async persistDocument(notebookId: string): Promise<void>;
  async flushToRelational(notebookId: string): Promise<void>;  // Sync Y.Doc → notebooks.content
}
```

Yjs document structure (maps to nbformat):
```typescript
// Y.Doc shared types for a notebook
// doc.getMap('metadata')     → kernelspec, language_info
// doc.getArray('cells')      → Y.Map per cell
//   cell.get('id')           → string
//   cell.get('cell_type')    → 'code' | 'markdown' | 'raw'
//   cell.get('source')       → Y.Text (for collaborative text editing)
//   cell.get('outputs')      → Y.Array (replaced on execution, not collaboratively edited)
//   cell.get('metadata')     → Y.Map
```

Persistence flow: Yjs binary state (`Y.encodeStateAsUpdate`) stored in `collaboration_sessions.yjs_state`. Periodic flush converts Y.Doc to nbformat JSON and updates `notebooks.content`.

**Testing**:
- `Integration: connect two WebSocket clients → both receive initial document state`
- `Integration: client A types in cell → client B sees the update within 100ms`
- `Integration: client disconnects and reconnects → document state preserved`
- `Integration: server restart → document loaded from PostgreSQL, clients reconnect`
- `Unit: flushToRelational produces valid nbformat JSON`
- `Unit: loadDocument from empty DB → creates new Y.Doc with default notebook structure`
- `Unit: maxConnections exceeded → new connection rejected with 429`

---

#### 4.2 — Client-Side Yjs Integration

**What**: Integrate the Yjs provider into the React notebook editor, binding Y.Text to CodeMirror instances for each cell and syncing cell list operations (add, delete, reorder) through Y.Array.

**Design**:

```typescript
// packages/client/src/hooks/useYjsProvider.ts
interface YjsProviderState {
  doc: Y.Doc | null;
  provider: WebsocketProvider | null;
  connected: boolean;
  synced: boolean;
}

function useYjsProvider(notebookId: string): YjsProviderState;

// packages/client/src/hooks/useNotebook.ts (updated for Yjs)
interface UseNotebookReturn {
  cells: NbformatCell[];
  addCell: (type: CellType, position: number) => void;
  deleteCell: (cellId: string) => void;
  moveCell: (cellId: string, direction: 'up' | 'down') => void;
  // Source updates now go through Y.Text binding, not direct setState
}
```

CodeMirror binding: `y-codemirror.next` extension binds each cell's `Y.Text` to a CodeMirror `EditorView`. Undo/redo scoped per cell via `Y.UndoManager`.

**Testing**:
- `E2E: two browser tabs open same notebook → typing in tab A appears in tab B`
- `E2E: add cell in tab A → cell appears in tab B`
- `E2E: delete cell in tab A → cell removed in tab B`
- `E2E: reorder cell in tab A → order updated in tab B`
- `E2E: undo in tab A → only tab A's changes undone`
- `Unit: useYjsProvider connects on mount, disconnects on unmount`
- `Integration: network partition → local edits buffered, synced on reconnect`

---

#### 4.3 — Presence and Cursor Tracking

**What**: Show collaborator avatars, names, and cursor positions in the notebook using Yjs Awareness protocol.

**Design**:

```typescript
// packages/client/src/hooks/usePresence.ts
interface PresenceState {
  user: {
    id: string;
    name: string;
    avatarUrl: string | null;
    color: string;          // deterministic from user ID
  };
  cursor: {
    cellId: string;
    offset: number;
  } | null;
}

function usePresence(provider: WebsocketProvider, currentUser: UserProfile): {
  peers: Map<number, PresenceState>;  // clientID → state
  updateCursor: (cellId: string, offset: number) => void;
};
```

Components:
- `CollaboratorList`: shows avatars of all connected users in the notebook header.
- `CursorOverlay`: renders colored cursor lines and name labels within CodeMirror cells.
- `PresenceIndicator`: colored dot on cells being edited by other users.

Color assignment: deterministic hash of user ID to one of 12 preset colors (avoiding red/green for colorblind accessibility).

**Testing**:
- `E2E: two users open notebook → both see each other in collaborator list`
- `E2E: user A clicks in cell 3 → user B sees A's cursor in cell 3`
- `E2E: user A disconnects → removed from collaborator list within 5 seconds`
- `Unit: color assignment is deterministic for same user ID`
- `Unit: 12 preset colors are WCAG AA contrast-compliant against white and dark backgrounds`

---

## Phase 5: Kernel Execution

### Purpose

Connect the notebook to a Jupyter Kernel Gateway for cell execution. Users can run Python and SQL code cells and see outputs rendered inline. After this phase, the notebook is a fully functional computational environment.

### Tasks

#### 5.1 — Kernel Gateway Integration

**What**: Implement a service that manages kernel lifecycle (start, interrupt, restart, shutdown) via the Jupyter Kernel Gateway REST API.

**Design**:

```typescript
// packages/server/src/services/kernel.service.ts
interface KernelInfo {
  kernelId: string;
  name: string;          // 'python3'
  state: 'starting' | 'idle' | 'busy' | 'dead';
  lastActivity: string;
  connections: number;
}

interface KernelService {
  startKernel(notebookId: string, kernelName: string): Promise<KernelInfo>;
  getKernel(notebookId: string): Promise<KernelInfo | null>;
  interruptKernel(notebookId: string): Promise<void>;
  restartKernel(notebookId: string): Promise<void>;
  shutdownKernel(notebookId: string): Promise<void>;
  executeCode(notebookId: string, code: string): AsyncIterable<KernelMessage>;
}

type KernelMessage =
  | { type: 'execute_result'; data: Record<string, string>; executionCount: number }
  | { type: 'display_data'; data: Record<string, string> }
  | { type: 'stream'; name: 'stdout' | 'stderr'; text: string }
  | { type: 'error'; ename: string; evalue: string; traceback: string[] }
  | { type: 'status'; state: 'busy' | 'idle' };
```

Communication: WebSocket to Kernel Gateway for execution (ZMQ over WebSocket). REST API for lifecycle management. Each notebook gets one kernel instance. Kernels are idle-reaped after configurable timeout (default: 30 minutes).

**Testing**:
- `Integration (real kernel): startKernel → kernel ID returned, state is 'idle'`
- `Integration (real kernel): execute 'print("hello")' → stream output 'hello\n'`
- `Integration (real kernel): execute '1+1' → execute_result with text/plain '2'`
- `Integration (real kernel): execute invalid syntax → error output with traceback`
- `Integration (real kernel): interrupt during sleep(60) → KeyboardInterrupt`
- `Integration (real kernel): restart → execution count resets, variables cleared`
- `Integration (mocked): kernel gateway unavailable → meaningful error returned`
- `Unit: idle reap after timeout → shutdownKernel called`

---

#### 5.2 — Cell Execution API and Output Streaming

**What**: API endpoint to execute a cell and stream outputs back to the client via Server-Sent Events (SSE). Outputs are stored in the notebook's JSONB content.

**Design**:

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/api/workspaces/:slug/notebooks/:notebookId/cells/:cellId/execute` | JWT + editor | SSE stream of `KernelMessage` |
| POST | `/api/workspaces/:slug/notebooks/:notebookId/execute-all` | JWT + editor | SSE stream (sequential cell execution) |
| POST | `/api/workspaces/:slug/notebooks/:notebookId/kernel/interrupt` | JWT + editor | `204` |
| POST | `/api/workspaces/:slug/notebooks/:notebookId/kernel/restart` | JWT + editor | `KernelInfo` |

Execution flow:
1. Extract cell source from Y.Doc (or notebooks.content if not collaborating).
2. Start kernel if not running.
3. Send execute_request to kernel.
4. Stream outputs via SSE to requesting client.
5. Broadcast outputs to all collaborators via Yjs (update cell's `outputs` Y.Array).
6. Update `notebooks.content` with final outputs on execution complete.
7. Increment `execution_count` on the cell.

**Testing**:
- `E2E: click Run on code cell → output appears below cell`
- `E2E: run cell with print() → stdout appears incrementally`
- `E2E: run cell with error → traceback displayed`
- `E2E: run cell producing plot → image rendered`
- `E2E: user A runs cell → user B sees output appear (via Yjs sync)`
- `E2E: click Interrupt → execution stops, KeyboardInterrupt shown`
- `E2E: Run All → cells execute sequentially, outputs accumulate`
- `Integration: SSE stream delivers messages in order with correct event types`

---

#### 5.3 — SQL Cell Execution

**What**: Execute SQL cells against configured data connections, rendering results as tables.

**Design**:

SQL cells use `cell_type: 'code'` with cell metadata `{ "sql": true, "connection_id": "..." }`. The server detects SQL cells and routes execution to the data connection rather than the Python kernel.

```typescript
// packages/server/src/services/execution.service.ts
interface SqlExecutionResult {
  columns: string[];
  rows: unknown[][];
  rowCount: number;
  truncated: boolean;       // true if result exceeds max rows (default: 1000)
  durationMs: number;
}
```

Results converted to nbformat output:
```json
{
  "output_type": "execute_result",
  "data": {
    "text/html": "<table>...</table>",
    "application/json": { "columns": [...], "rows": [...] }
  }
}
```

**Testing**:
- `Integration (real DB): SELECT 1 → result table with one row, one column`
- `Integration (real DB): SELECT with multiple columns → table rendered with headers`
- `Integration (real DB): invalid SQL → error output with database error message`
- `Integration (real DB): result exceeds 1000 rows → truncated flag set, first 1000 returned`
- `Unit: connection_id not configured → 400 with helpful error`
- `Unit: SQL injection via cell metadata → connection_id validated as UUID`

---

## Phase 6: Version Control and Git Integration

### Purpose

Add version history (internal snapshots) and Git integration (commit, branch, diff) so users can track changes, revert, and collaborate via existing Git workflows. Includes the AI-powered semantic diff feature identified as a key differentiator.

### Tasks

#### 6.1 — Internal Version History

**What**: Automatic and manual version snapshots stored in `notebook_versions`, with a version browser in the UI.

**Design**:

```typescript
// packages/shared/src/types/version.ts
export interface NotebookVersion {
  id: string;
  notebookId: string;
  versionNumber: number;
  message: string | null;
  diffSummary: string | null;     // AI-generated plain-language summary
  diffStats: DiffStats | null;
  createdBy: string;
  createdAt: string;
}

export interface DiffStats {
  cellsAdded: number;
  cellsDeleted: number;
  cellsModified: number;
  linesAdded: number;
  linesDeleted: number;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/versions` | `NotebookVersion` (manual save) |
| GET | `.../notebooks/:notebookId/versions` | `NotebookVersion[]` |
| GET | `.../notebooks/:notebookId/versions/:versionId` | `NotebookFull` (at that version) |
| POST | `.../notebooks/:notebookId/versions/:versionId/restore` | `NotebookFull` (restored) |
| GET | `.../notebooks/:notebookId/versions/:v1/diff/:v2` | `VersionDiff` |

Auto-save: version created every 5 minutes during active editing (if content changed). Manual save: user clicks "Save Version" with optional message.

**Testing**:
- `Unit: save version → snapshot stored, version_number incremented`
- `Unit: auto-save skipped when content unchanged`
- `Unit: restore version → notebook.content replaced with snapshot`
- `Unit: diff between versions → correct cells_added, cells_modified counts`
- `E2E: version history panel shows chronological list with authors`
- `E2E: click version → read-only view of notebook at that point in time`
- `E2E: restore version → notebook content reverted, new version created`

---

#### 6.2 — Git Integration

**What**: Connect projects to Git repositories for commit, branch, and diff operations via the UI. Uses `isomorphic-git` for serverless Git operations or shells out to `git` CLI.

**Design**:

```typescript
// packages/shared/src/types/git.ts
export interface GitConnection {
  id: string;
  projectId: string;
  provider: 'github' | 'gitlab' | 'bitbucket';
  repoUrl: string;
  branch: string;
  lastSyncedAt: string | null;
}

export interface GitCommit {
  sha: string;
  message: string;
  author: string;
  date: string;
}

export interface GitDiff {
  files: GitFileDiff[];
}

export interface GitFileDiff {
  path: string;
  status: 'added' | 'modified' | 'deleted';
  hunks: Array<{
    header: string;
    lines: Array<{ type: 'add' | 'remove' | 'context'; content: string }>;
  }>;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../projects/:projectId/git/connect` | `GitConnection` |
| POST | `.../projects/:projectId/git/commit` | `GitCommit` |
| GET | `.../projects/:projectId/git/log` | `GitCommit[]` |
| GET | `.../projects/:projectId/git/diff` | `GitDiff` |
| POST | `.../projects/:projectId/git/pull` | `{ status: 'up-to-date' \| 'merged' \| 'conflict' }` |
| POST | `.../projects/:projectId/git/push` | `{ status: 'pushed' }` |

Notebook-to-Git mapping: each notebook is a `.ipynb` file in the repository. The `content` JSONB column is serialized to pretty-printed JSON for readable Git diffs.

**Testing**:
- `Integration (mocked Git): connect to repo → connection saved, initial clone`
- `Integration (mocked Git): commit → .ipynb files written to repo, commit created`
- `Integration (mocked Git): diff → shows changed notebooks as file diffs`
- `Integration (mocked Git): pull with no conflicts → notebooks updated from remote`
- `Unit: notebook content serialized with sorted keys for stable diffs`
- `Unit: .ipynb files in repo round-trip without data loss`

---

#### 6.3 — Semantic Diff Summaries (AI)

**What**: Generate plain-language summaries of what changed between notebook versions using an LLM, addressing the "semantic version diffing" gap identified in the features survey.

**Design**:

```typescript
// packages/server/src/services/version.service.ts
interface SemanticDiffRequest {
  notebookId: string;
  fromVersionId: string;
  toVersionId: string;
}

interface SemanticDiffResult {
  summary: string;        // "Added feature engineering step using one-hot encoding for categorical variables. Removed the manual label encoding that was causing data leakage."
  changes: SemanticChange[];
}

interface SemanticChange {
  cellId: string;
  changeType: 'added' | 'modified' | 'deleted';
  description: string;   // per-cell plain-language description
}
```

Prompt template (sent to Claude):
```
You are a data science notebook analyst. Compare two versions of a Jupyter notebook and describe the semantic changes in plain language. Focus on what changed conceptually (not line-by-line diffs). Identify:
1. What analytical steps were added, modified, or removed
2. How the data processing pipeline changed
3. Whether results or conclusions changed
4. Any potential issues introduced

Version A cells: <cells_json>
Version B cells: <cells_json>
```

**Testing**:
- `Integration (mocked LLM): two versions with added cell → summary mentions the new analysis step`
- `Integration (mocked LLM): modified code cell → summary describes the functional change`
- `Unit: diff with no changes → "No semantic changes detected"`
- `Unit: large notebook (>50 cells) → only changed cells sent to LLM (context optimization)`
- `Unit: LLM timeout → graceful fallback to statistical diff summary`

---

## Phase 7: Scheduled Execution and Notifications

### Purpose

Enable scheduled notebook runs with cron expressions, parameter injection, and notifications on completion/failure. After this phase, notebooks can run automatically on a schedule, making the platform viable for recurring data pipelines.

### Tasks

#### 7.1 — Schedule Management

**What**: CRUD for scheduled run configurations with cron expression validation and timezone support.

**Design**:

```typescript
// packages/shared/src/types/execution.ts
export interface ScheduledRun {
  id: string;
  notebookId: string;
  name: string;
  cronExpression: string;
  timezone: string;
  isActive: boolean;
  config: ScheduleConfig;
  nextRunAt: string;          // computed from cron + timezone
  lastRunAt: string | null;
  lastRunStatus: string | null;
  createdBy: string;
  createdAt: string;
}

export interface ScheduleConfig {
  parameters: Record<string, unknown>;
  compute: {
    cpu: string;
    memory: string;
    timeoutSeconds: number;
  };
  notifications: NotificationChannel[];
  retry: { maxAttempts: number; delaySeconds: number };
}

export interface NotificationChannel {
  on: 'success' | 'failure';
  channel: 'email' | 'webhook';
  target: string;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/schedules` | `ScheduledRun` |
| GET | `.../notebooks/:notebookId/schedules` | `ScheduledRun[]` |
| PATCH | `.../notebooks/:notebookId/schedules/:scheduleId` | `ScheduledRun` |
| DELETE | `.../notebooks/:notebookId/schedules/:scheduleId` | `204` |
| POST | `.../notebooks/:notebookId/schedules/:scheduleId/trigger` | `ExecutionRun` (manual trigger) |

Cron validation: use `cron-parser` library. Support standard 5-field cron syntax plus `@daily`, `@hourly`, `@weekly` shortcuts.

**Testing**:
- `Unit: create schedule with valid cron → schedule saved, nextRunAt computed correctly`
- `Unit: create schedule with invalid cron → 400 with parsing error`
- `Unit: toggle isActive → schedule paused/resumed`
- `Unit: nextRunAt computed correctly for different timezones`
- `Unit: @daily shortcut → 0 0 * * * cron expression`
- `Integration: manual trigger → execution run created and queued`

---

#### 7.2 — Execution Worker

**What**: BullMQ worker that processes scheduled and manual notebook execution jobs: starts a kernel, runs all cells sequentially, captures outputs, and updates the execution run record.

**Design**:

```typescript
// packages/server/src/queue/workers/execution.worker.ts
interface ExecutionJob {
  executionRunId: string;
  notebookId: string;
  parameters: Record<string, unknown>;
  timeoutSeconds: number;
}

// Worker flow:
// 1. Load notebook content (snapshot at execution time)
// 2. Start kernel with parameters injected as variables
// 3. Execute cells sequentially, capturing outputs
// 4. On each cell complete: update execution_runs.result JSONB
// 5. On completion: set status = 'completed', record duration
// 6. On failure: set status = 'failed', record error, trigger retry if configured
// 7. Send notifications based on schedule config
// 8. Shut down kernel
```

Execution run status lifecycle: `queued → running → completed | failed | cancelled | timed_out`

**Testing**:
- `Integration (real kernel): execute 3-cell notebook → all outputs captured, status = completed`
- `Integration (real kernel): cell 2 errors → status = failed, error in result, cells 3+ skipped`
- `Integration (real kernel): timeout exceeded → status = timed_out, kernel killed`
- `Unit: parameters injected as Python variables before first cell`
- `Unit: retry on failure → re-queued up to maxAttempts`
- `Unit: concurrent execution limit enforced per workspace`

---

#### 7.3 — Notification Delivery

**What**: Send email and webhook notifications on execution completion/failure.

**Design**:

```typescript
// packages/server/src/services/notification.service.ts
interface NotificationService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
  sendWebhook(url: string, payload: WebhookPayload): Promise<void>;
}

interface WebhookPayload {
  event: 'execution.completed' | 'execution.failed';
  executionRunId: string;
  notebookId: string;
  notebookName: string;
  status: string;
  durationMs: number;
  error?: string;
  triggeredAt: string;
  completedAt: string;
}
```

Email: use `nodemailer` with configurable SMTP transport (or SendGrid/SES in production). Webhook: HTTP POST with HMAC-SHA256 signature in `X-Webhook-Signature` header for verification.

**Testing**:
- `Integration (mocked SMTP): successful execution → email sent with notebook name and duration`
- `Integration (mocked SMTP): failed execution → email includes error summary`
- `Integration (mocked HTTP): webhook POST sent with correct payload and signature`
- `Unit: webhook signature verification → valid signature accepted, invalid rejected`
- `Unit: webhook delivery retry on 5xx → up to 3 attempts with exponential backoff`
- `Unit: notification skipped if channel config empty`

---

## Phase 8: Sharing, Comments, and Permissions

### Purpose

Enable granular notebook sharing (link-based with optional password protection), cell-level comments with threading, and fine-grained permission enforcement. After this phase, teams can share work securely with configurable access levels.

### Tasks

#### 8.1 — Notebook Sharing

**What**: Share notebooks via direct user invitation or link with configurable permissions, optional password, and expiry.

**Design**:

```typescript
// packages/shared/src/types/share.ts
export interface NotebookShare {
  id: string;
  notebookId: string;
  type: 'user' | 'link';
  permission: 'view' | 'comment' | 'edit';
  // Link-specific
  linkToken?: string;
  passwordProtected?: boolean;
  expiresAt?: string;
  allowDownload?: boolean;
  // User-specific
  userId?: string;
  userEmail?: string;
  createdBy: string;
  createdAt: string;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/shares` | `NotebookShare` |
| GET | `.../notebooks/:notebookId/shares` | `NotebookShare[]` |
| DELETE | `.../notebooks/:notebookId/shares/:shareId` | `204` |
| GET | `/api/shared/:linkToken` | `NotebookFull` (public, with password check) |

Link tokens: 32-byte cryptographically random hex strings. Password: optional bcrypt hash stored in `share_config` JSONB. Expiry: checked on access; expired links return 410 Gone.

**Testing**:
- `Unit: create user share → user can access notebook`
- `Unit: create link share → linkToken generated, accessible via /shared/:token`
- `Unit: link with password → 401 without password, 200 with correct password`
- `Unit: expired link → 410 Gone`
- `Unit: view permission → cannot edit cells`
- `Unit: comment permission → can add comments but not edit cells`
- `Unit: delete share → access revoked immediately`
- `E2E: share notebook → recipient sees it in their workspace`

---

#### 8.2 — Cell-Level Comments

**What**: Add threaded comments on individual cells (Google Docs-style), with resolve/unresolve and @mention notifications.

**Design**:

```typescript
// packages/shared/src/types/comment.ts
export interface Comment {
  id: string;
  notebookId: string;
  cellId: string | null;    // null = notebook-level comment
  parentCommentId: string | null;
  author: {
    id: string;
    displayName: string;
    avatarUrl: string | null;
  };
  body: string;
  resolved: boolean;
  resolvedBy: string | null;
  resolvedAt: string | null;
  createdAt: string;
  updatedAt: string;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/comments` | `Comment` |
| GET | `.../notebooks/:notebookId/comments` | `Comment[]` (threaded) |
| PATCH | `.../notebooks/:notebookId/comments/:commentId` | `Comment` |
| DELETE | `.../notebooks/:notebookId/comments/:commentId` | `204` |
| POST | `.../notebooks/:notebookId/comments/:commentId/resolve` | `Comment` |

@mentions: parse `@user-email` in comment body, send notification to mentioned user. Real-time: broadcast new comments to collaborators via Yjs Awareness or a dedicated WebSocket channel.

**Testing**:
- `E2E: add comment on cell → comment indicator appears on cell, comment panel shows comment`
- `E2E: reply to comment → threaded reply appears indented`
- `E2E: resolve comment → comment marked as resolved with resolver name`
- `E2E: @mention → notification sent to mentioned user`
- `Unit: comment on non-existent cell ID → 400`
- `Unit: delete comment by non-author → 403 (unless admin)`
- `Unit: comments broadcast to collaborators in real time`

---

## Phase 9: Data Connections and SQL Integration

### Purpose

Allow workspaces to configure database connections that SQL cells can query against, with schema discovery for AI-powered completions. After this phase, the platform supports the mixed Python + SQL workflow that Deepnote, Hex, and Briefer offer.

### Tasks

#### 9.1 — Data Connection Management

**What**: CRUD for workspace-level database connections with encrypted credential storage and connection testing.

**Design**:

```typescript
// packages/shared/src/types/connection.ts
export interface DataConnection {
  id: string;
  workspaceId: string;
  name: string;
  connectorType: 'postgresql' | 'mysql' | 'snowflake' | 'bigquery' | 'redshift' | 'sqlite';
  config: Record<string, unknown>;  // varies by connector type
  schemaCache: DatabaseSchema | null;
  lastTestedAt: string | null;
  createdBy: string;
  createdAt: string;
}

export interface DatabaseSchema {
  tables: TableSchema[];
  cachedAt: string;
}

export interface TableSchema {
  name: string;
  schema: string;
  columns: ColumnSchema[];
  rowCountApprox?: number;
}

export interface ColumnSchema {
  name: string;
  type: string;
  nullable: boolean;
  isPrimaryKey: boolean;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../connections` | `DataConnection` |
| GET | `.../connections` | `DataConnection[]` |
| PATCH | `.../connections/:connectionId` | `DataConnection` |
| DELETE | `.../connections/:connectionId` | `204` |
| POST | `.../connections/:connectionId/test` | `{ success: boolean; error?: string }` |
| POST | `.../connections/:connectionId/refresh-schema` | `DatabaseSchema` |

Credential encryption: AES-256-GCM with a key derived from `ENCRYPTION_KEY` env var. Credentials never returned in API responses (write-only).

**Testing**:
- `Integration (real PostgreSQL): create connection → test succeeds`
- `Integration (real PostgreSQL): refresh schema → tables and columns discovered`
- `Unit: credentials encrypted at rest, not returned in GET`
- `Unit: test with invalid credentials → success: false with error message`
- `Unit: delete connection → referenced SQL cells show "connection removed" warning`
- `Unit: schema cache includes column types and primary keys`

---

#### 9.2 — Schema Browser UI

**What**: Sidebar panel showing connected database schemas with table/column browsing and click-to-insert for SQL cells.

**Design**:

```typescript
// packages/client/src/components/sidebar/SchemaExplorer.tsx
interface SchemaExplorerProps {
  connections: DataConnection[];
  onInsertTable: (tableName: string) => void;
  onInsertColumn: (tableName: string, columnName: string) => void;
}
```

Tree structure: Connection → Schema → Table → Columns (with type badges). Click table name → inserts `SELECT * FROM schema.table LIMIT 100` into active SQL cell. Click column → inserts column name at cursor.

**Testing**:
- `E2E: open schema browser → connections listed with tables`
- `E2E: expand table → columns displayed with type badges`
- `E2E: click table → SQL inserted into active cell`
- `E2E: search filter → tables/columns filtered by name`
- `Unit: schema browser handles no connections gracefully`

---

## Phase 10: AI-Native Features

### Purpose

Implement the AI capabilities that differentiate this platform from incumbents: context-aware code completion, automated narrative generation, and output anomaly detection. These features leverage the full notebook state (variables, schemas, prior outputs) as context for the LLM.

### Tasks

#### 10.1 — AI Code Completion with Notebook Context

**What**: Inline code suggestions powered by Claude that understand the full notebook state: defined variables, imported libraries, dataframe schemas, and prior cell outputs.

**Design**:

```typescript
// packages/server/src/services/ai-completion.service.ts
interface CompletionRequest {
  notebookId: string;
  cellId: string;
  cursorPosition: number;
  maxTokens?: number;
}

interface NotebookContext {
  cells: Array<{
    cellType: string;
    source: string;
    outputs: string[];       // summarized output representations
  }>;
  variables: Variable[];     // from kernel introspection
  schemas: TableSchema[];    // from connected data sources
  currentCellSource: string;
  cursorPosition: number;
}

interface Variable {
  name: string;
  type: string;
  shape?: string;           // for DataFrames: "(1000, 5)"
  columns?: string[];       // for DataFrames: column names
  sampleValue?: string;     // truncated string repr
}

interface CompletionResponse {
  suggestion: string;
  confidence: number;
}
```

Context assembly: kernel introspection via `%who_ls` and custom magics to extract variable names, types, and DataFrame schemas. Combined with connected database schemas and prior cell source/outputs.

System prompt:
```
You are an AI code assistant embedded in a Jupyter-compatible notebook. You have access to the user's full notebook context including defined variables, their types and shapes, connected database schemas, and all prior code and outputs.

Generate a code completion that continues naturally from the cursor position. Be concise, contextually relevant, and type-aware. If the user is working with a DataFrame, suggest operations consistent with its schema.
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/ai/complete` | `CompletionResponse` |

**Testing**:
- `Integration (mocked LLM): cursor after 'df.' → suggestion includes column names from DataFrame`
- `Integration (mocked LLM): SQL cell with schema context → suggests valid table/column names`
- `Integration (mocked LLM): empty cell after data loading → suggests analysis code`
- `Unit: context assembly includes last 10 cell outputs (not all, to fit token budget)`
- `Unit: variable introspection extracts DataFrame schema correctly`
- `Unit: completion request with no kernel → falls back to static context only`
- `Unit: ai_interactions record created with acceptance tracking`

---

#### 10.2 — Narrative Gap Detection and Auto-Generation

**What**: Detect gaps in notebook narrative (code blocks without explanatory Markdown) and suggest prose to insert between cells.

**Design**:

```typescript
// packages/server/src/services/ai-completion.service.ts
interface NarrativeAnalysis {
  gaps: NarrativeGap[];
}

interface NarrativeGap {
  afterCellId: string;
  suggestion: string;        // Markdown prose to insert
  confidence: number;
  reason: string;            // "Three consecutive code cells with no explanation"
}
```

Detection heuristics:
1. Three or more consecutive code cells with no Markdown between them.
2. Code cell that defines a function or class with no preceding documentation.
3. Code cell that produces a visualization with no following interpretation.
4. Transition between data loading and analysis with no explanation of the dataset.

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/ai/narrative-analysis` | `NarrativeAnalysis` |

**Testing**:
- `Integration (mocked LLM): notebook with 5 consecutive code cells → gaps detected`
- `Integration (mocked LLM): well-documented notebook → no gaps`
- `Unit: gap after function definition → suggests docstring-style explanation`
- `Unit: gap after visualization → suggests interpretation prompt`

---

#### 10.3 — Output Anomaly Detection

**What**: Compare cell outputs across execution runs and alert when values deviate significantly from historical patterns.

**Design**:

```typescript
// packages/server/src/services/ai-completion.service.ts
interface AnomalyDetectionConfig {
  notebookId: string;
  cellId: string;
  enabled: boolean;
  thresholds: {
    numericDeviationStdDev: number;   // default: 3.0
    schemaChangeAlert: boolean;       // default: true
    rowCountDeviationPercent: number; // default: 20
  };
}

interface AnomalyAlert {
  id: string;
  executionRunId: string;
  cellId: string;
  alertType: 'distribution_shift' | 'value_out_of_range' | 'schema_change' | 'row_count_change';
  severity: 'info' | 'warning' | 'critical';
  description: string;
  expected: Record<string, unknown>;
  actual: Record<string, unknown>;
  createdAt: string;
}
```

Detection runs post-execution as a background job. Compares current output against the rolling average of the last 10 runs.

**Testing**:
- `Unit: numeric output shifted by 4 std devs → critical anomaly detected`
- `Unit: output within 1 std dev → no anomaly`
- `Unit: DataFrame schema changed (column added) → schema_change alert`
- `Unit: row count dropped by 50% → row_count_change warning`
- `Unit: first execution (no history) → no anomaly detection possible, skip`
- `Integration: anomaly alert stored in ai_interactions table`

---

## Phase 11: Dashboard Publishing

### Purpose

Enable one-click publishing of notebook outputs as interactive dashboards for non-technical stakeholders. After this phase, the platform covers the Hex/Deepnote differentiator of notebook-to-dashboard publishing.

### Tasks

#### 11.1 — Dashboard Builder

**What**: UI for selecting notebook cell outputs and arranging them in a grid layout as a publishable dashboard.

**Design**:

```typescript
// packages/shared/src/types/dashboard.ts
export interface Dashboard {
  id: string;
  notebookId: string;
  slug: string;
  title: string;
  config: DashboardConfig;
  publishedBy: string;
  publishedAt: string;
  updatedAt: string;
}

export interface DashboardConfig {
  layout: DashboardPanel[];
  inputs: DashboardInput[];
  autoRefreshSeconds: number | null;
  theme: 'light' | 'dark' | 'auto';
}

export interface DashboardPanel {
  cellId: string;
  type: 'chart' | 'table' | 'metric' | 'markdown';
  x: number; y: number; w: number; h: number;  // grid coordinates (12-column grid)
  title?: string;
}

export interface DashboardInput {
  name: string;
  type: 'text' | 'number' | 'date_picker' | 'dropdown';
  label: string;
  default: unknown;
  options?: string[];    // for dropdown
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `.../notebooks/:notebookId/dashboards` | `Dashboard` |
| GET | `.../dashboards/:slug` | Dashboard HTML (public, no auth required) |
| PATCH | `.../dashboards/:dashboardId` | `Dashboard` |
| DELETE | `.../dashboards/:dashboardId` | `204` |

Grid layout: `react-grid-layout` for drag-and-drop panel arrangement. Dashboard viewer is a standalone route that renders cell outputs in the defined layout, re-executing the notebook on input changes.

**Testing**:
- `E2E: drag cell outputs onto dashboard grid → layout saved`
- `E2E: publish dashboard → accessible via public slug URL`
- `E2E: dashboard with auto-refresh → outputs update on interval`
- `E2E: dashboard input (dropdown) → notebook re-executed with parameter, outputs update`
- `Unit: invalid slug (duplicate) → 409 Conflict`
- `Unit: dashboard references deleted cell → panel shows "Cell not found" placeholder`

---

## Phase 12: Production Readiness

### Purpose

Prepare the platform for production deployment: Docker containerisation, OpenAPI documentation, rate limiting, security hardening, monitoring, and CI/CD pipeline. After this phase, the platform can be deployed and operated by teams.

### Tasks

#### 12.1 — Docker Containerisation

**What**: Production Dockerfile and docker-compose.yml for the complete stack (server, client, PostgreSQL, Redis, Kernel Gateway).

**Design**:

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/ packages/
RUN corepack enable && pnpm install --frozen-lockfile && pnpm turbo build

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/packages/server/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3001
CMD ["node", "dist/server.js"]
```

Production `docker-compose.yml`:
```yaml
services:
  server:
    build: .
    ports: ["3001:3001"]
    environment:
      DATABASE_URL: postgresql://notebook:${DB_PASSWORD}@postgres:5432/notebook_collab
      REDIS_URL: redis://redis:6379
    depends_on: [postgres, redis]
  client:
    build:
      context: .
      dockerfile: packages/client/Dockerfile
    ports: ["3000:80"]
  postgres:
    image: postgres:16-alpine
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
  kernel-gateway:
    build: packages/kernel-gateway
    ports: ["8888:8888"]
volumes:
  pgdata:
```

**Testing**:
- `Integration: docker-compose up → all services start, health checks pass`
- `Integration: client accessible on port 3000, API on port 3001`
- `Integration: kernel gateway responds to /api/kernelspecs`
- `Unit: Docker image size < 500MB`
- `Unit: non-root user in runtime container`

---

#### 12.2 — OpenAPI Specification and REST API Documentation

**What**: Auto-generate an OpenAPI 3.1 specification from Fastify route schemas, served at `/api/docs`.

**Design**:

Fastify plugin configuration:
```typescript
// packages/server/src/plugins/swagger.ts
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';

export async function registerSwagger(app: FastifyInstance) {
  await app.register(swagger, {
    openapi: {
      info: {
        title: 'Notebook Collaboration Platform API',
        version: '1.0.0',
      },
      components: {
        securitySchemes: {
          bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
          apiKey: { type: 'apiKey', in: 'header', name: 'X-API-Key' },
        },
      },
    },
  });
  await app.register(swaggerUi, { routePrefix: '/api/docs' });
}
```

Every route must have JSON Schema definitions for request body, query parameters, and response. These drive both runtime validation and OpenAPI generation.

**Testing**:
- `Unit: /api/docs returns Swagger UI HTML`
- `Unit: /api/docs/json returns valid OpenAPI 3.1 JSON`
- `Unit: all routes have request/response schemas defined`
- `Integration: generated OpenAPI spec validates against OpenAPI 3.1 schema`

---

#### 12.3 — Security Hardening

**What**: Implement rate limiting, CORS configuration, CSP headers, input sanitisation, and kernel execution sandboxing per OWASP Top 10 guidelines.

**Design**:

Security measures:
- **Rate limiting**: `@fastify/rate-limit` — 100 req/min for authenticated, 20 req/min for unauthenticated, 10 req/min for auth endpoints (brute-force protection).
- **CORS**: strict origin allowlist from `CORS_ORIGIN` env var.
- **CSP**: `Content-Security-Policy` header restricting script sources; cell output iframes sandboxed.
- **Input validation**: all inputs validated via Fastify JSON Schema (Ajv); notebook content validated against nbformat schema.
- **SQL injection**: parameterised queries via Drizzle ORM; SQL cell execution uses connection pool with read-only role.
- **Secrets in notebooks**: scan cell source for common secret patterns (API keys, passwords) and warn users.
- **Kernel isolation**: each kernel runs in a separate Docker container with resource limits (CPU, memory, no network by default).

**Testing**:
- `Unit: 101st request within 1 minute → 429 Too Many Requests`
- `Unit: CORS request from unlisted origin → blocked`
- `Unit: CSP header present on all responses`
- `Unit: notebook with AWS key in cell source → warning returned on save`
- `Integration: kernel container has no outbound network access by default`
- `Unit: SQL cell execution uses read-only connection role`

---

#### 12.4 — CI/CD Pipeline

**What**: GitHub Actions workflow for linting, type checking, testing, building Docker images, and deploying.

**Design**:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint
      - run: pnpm turbo typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_USER: test, POSTGRES_PASSWORD: test, POSTGRES_DB: test }
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo test

  docker:
    needs: [lint-and-typecheck, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:latest
```

**Testing**:
- `Unit: CI pipeline passes on a clean checkout`
- `Unit: failing test → pipeline fails, PR blocked`
- `Unit: Docker image pushed only on main branch`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                          ─── required by everything
    │
Phase 2: Notebook CRUD & Import/Export       ─── requires Phase 1
    │
    ├── Phase 3: Notebook UI                 ─── requires Phase 2
    │       │
    │       ├── Phase 4: Real-Time Collab    ─── requires Phase 3
    │       │       │
    │       │       └── Phase 5: Kernel Exec ─── requires Phase 4
    │       │               │
    │       │               ├── Phase 6: Version Control & Git ─── requires Phase 5
    │       │               │
    │       │               ├── Phase 7: Scheduling            ─── requires Phase 5
    │       │               │
    │       │               └── Phase 9: Data Connections       ─── requires Phase 5
    │       │
    │       └── Phase 8: Sharing & Comments  ─── requires Phase 3, can parallel with 4-5
    │
    └── Phase 10: AI-Native Features         ─── requires Phases 5, 9
         │
         └── Phase 11: Dashboard Publishing  ─── requires Phase 5
              │
              └── Phase 12: Production       ─── requires all above

Parallelism opportunities:
  - Phases 6, 7, 9 can be developed concurrently after Phase 5
  - Phase 8 can be developed concurrently with Phases 4-5 (after Phase 3)
  - Phase 11 can be developed concurrently with Phase 10 (after Phase 5)
```

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`pnpm turbo test`).
3. All integration tests pass (with test database and Redis).
4. ESLint passes with zero errors and zero warnings (`pnpm turbo lint`).
5. TypeScript type checking passes (`pnpm turbo typecheck`).
6. Docker build succeeds (`docker build .`).
7. The feature works end-to-end when tested manually in the dev environment.
8. New environment variables documented in `.env.example`.
9. New API endpoints have JSON Schema request/response definitions (auto-included in OpenAPI spec).
10. Database schema changes have a Drizzle migration file generated.
11. No secrets, credentials, or API keys committed to the repository.
12. Code reviewed — no TODO markers left in production code paths (TODOs allowed only for future-phase features).
