# Backend + Real-time Collaboration Design

**Date:** 2026-05-16
**Status:** Draft

## Context

ChartDB is currently 100% client-side: all diagram data persists in IndexedDB via Dexie. This design adds a Laravel backend with real-time collaboration so that multiple authenticated users can edit shared diagrams simultaneously.

## Goals

- Team collaboration: multiple users edit diagrams in the same workspace
- Real-time synchronization: changes appear instantly for all connected clients
- Full authentication: unauthenticated users see only the login page
- SaaS deployment: single hosted instance, multi-tenant via workspaces

## Non-goals

- Offline support / local-first mode (Dexie is removed)
- Mobile native apps
- Billing / subscription management (separate future project)

---

## Architecture

### Overall

```
Browser (React)
────────────────────────────────────────────────
All existing ChartDB code (canvas, panels, history,
import/export) — unchanged, talks only to StorageContext

BackendStorageProvider          ← replaces StorageProvider (Dexie)
  HTTP REST calls + WebSocket listener

Unauthenticated users → redirect to /login
────────────────────────────────────────────────
          │   REST API  +  WebSocket (Reverb)
          ▼
Laravel 11 Backend
  Sanctum auth │ REST API controllers │ Laravel Reverb
          │
          ▼
     PostgreSQL
```

`StorageProvider` (Dexie) is removed. `BackendStorageProvider` becomes the sole implementation of `StorageContext`. The rest of the application is untouched.

---

## Database Schema (PostgreSQL)

```sql
users
  id              UUID PK
  name            string
  email           string UNIQUE
  password        string
  email_verified_at timestamp nullable
  created_at / updated_at

workspaces
  id              UUID PK
  name            string
  created_by      → users.id
  created_at / updated_at

workspace_users  (pivot)
  workspace_id    → workspaces.id
  user_id         → users.id
  role            enum: owner | editor | viewer
  PRIMARY KEY (workspace_id, user_id)

diagrams
  id              UUID PK  -- preserves IDs from frontend
  workspace_id    → workspaces.id
  name            string
  database_type   string
  database_edition string nullable
  created_at / updated_at

-- All child entities share the same pattern:
db_tables
  id              UUID PK
  diagram_id      → diagrams.id
  name / schema / x / y / width / color / comment
  fields          JSONB
  indexes         JSONB
  is_view / is_materialized_view / order
  created_at / updated_at

db_relationships
  id              UUID PK
  diagram_id      → diagrams.id
  name / source_schema / source_table_id / target_schema
  target_table_id / source_field_id / target_field_id
  source_cardinality / target_cardinality
  created_at

db_dependencies
  id              UUID PK
  diagram_id      → diagrams.id
  schema / table_id / dependent_schema / dependent_table_id
  created_at

areas
  id              UUID PK
  diagram_id      → diagrams.id
  name / x / y / width / height / color

db_custom_types
  id              UUID PK
  diagram_id      → diagrams.id
  schema / name / kind / values JSONB / fields JSONB

notes
  id              UUID PK
  diagram_id      → diagrams.id
  content / x / y / width / height / color

diagram_filters
  diagram_id      → diagrams.id
  user_id         → users.id   -- per-user, not per-team
  table_ids       JSONB
  schema_ids      JSONB
  PRIMARY KEY (diagram_id, user_id)

user_settings
  user_id              → users.id PK
  default_diagram_id   → diagrams.id nullable
```

> `diagram_filters` and `user_settings` are per-user (not per-workspace) to preserve personal view preferences.
> `user_settings.default_diagram_id` replaces the `config` table from Dexie.

---

## REST API

All routes require `auth:sanctum` middleware. Access to a diagram is authorized by verifying the user is a member of the diagram's workspace.

```
# Authentication
GET    /sanctum/csrf-cookie
POST   /api/auth/register     { name, email, password }
POST   /api/auth/login        { email, password }
POST   /api/auth/logout
GET    /api/user              → { user, workspaces, roles, settings }
PATCH  /api/user/settings    { default_diagram_id }

# Workspaces
GET    /api/workspaces
POST   /api/workspaces
PATCH  /api/workspaces/{id}
DELETE /api/workspaces/{id}
POST   /api/workspaces/{id}/invitations   { email, role }

# Diagrams
GET    /api/diagrams                      → listDiagrams()
POST   /api/diagrams                      → addDiagram()
GET    /api/diagrams/{id}                 → getDiagram()
PATCH  /api/diagrams/{id}                 → updateDiagram()
DELETE /api/diagrams/{id}                 → deleteDiagram()

# Tables
GET    /api/diagrams/{id}/tables          → listTables()
POST   /api/diagrams/{id}/tables          → addTable()
GET    /api/tables/{id}                   → getTable()
PATCH  /api/tables/{id}                   → updateTable()
PUT    /api/tables/{id}                   → putTable()
DELETE /api/tables/{id}                   → deleteTable()
DELETE /api/diagrams/{id}/tables          → deleteDiagramTables()

# Relationships, Dependencies, Areas, CustomTypes, Notes
# — same CRUD pattern as Tables

# Diagram filter (per user)
GET    /api/diagrams/{id}/filter          → getDiagramFilter()
PUT    /api/diagrams/{id}/filter          → updateDiagramFilter()
DELETE /api/diagrams/{id}/filter          → deleteDiagramFilter()
```

---

## Real-time Synchronization (Laravel Reverb)

### Channels

```
private-diagram.{diagramId}    -- all diagram mutations
presence-diagram.{diagramId}   -- who is currently online
```

Private channels are authenticated via `/api/broadcasting/auth` (Sanctum). Access is granted only to workspace members.

### Flow

```
User A edits a table
  → BackendStorageProvider.updateTable()
  → PATCH /api/tables/{id}
  → Laravel saves to PostgreSQL
  → broadcasts TableUpdated on private-diagram.{diagramId}
  → Reverb delivers to all connected clients
  → useRealtimeSync hook receives event
  → calls chartdb EventEmitter.emit('update_table', { table })
  → Canvas re-renders the table node
```

### Broadcasted Events

```
TableUpdated      { table }          RelationshipAdded    { relationship }
TableAdded        { table }          RelationshipUpdated  { relationship }
TableDeleted      { tableId }        RelationshipDeleted  { relationshipId }

AreaAdded / AreaUpdated / AreaDeleted
NoteAdded / NoteUpdated / NoteDeleted
DependencyAdded / DependencyDeleted
CustomTypeAdded / CustomTypeUpdated / CustomTypeDeleted
DiagramUpdated    { diagram }
```

Each event carries the **full entity object** (not a diff). The client replaces its local copy directly, no patch logic needed.

### Frontend: `useRealtimeSync` hook

New hook at `src/hooks/use-realtime-sync.ts`. Connects via `laravel-echo` + `pusher-js` (the standard Reverb client). Translates incoming events into `chartdb-context` EventEmitter calls.

```typescript
useEffect(() => {
    const channel = echo.private(`diagram.${diagramId}`);

    channel.listen('TableUpdated', ({ table }) =>
        chartdb.events.emit('update_table', { table })
    );
    channel.listen('TableAdded', ({ table }) =>
        chartdb.events.emit('add_tables', { tables: [table] })
    );
    // ... same for all entity types

    return () => echo.leave(`diagram.${diagramId}`);
}, [diagramId]);
```

### Conflict Resolution

**Last-write-wins:** the server is authoritative. If two users simultaneously edit the same field, the later HTTP request wins. Acceptable for DB diagrams where concurrent edits to the same element are rare by nature.

---

## Authentication & Workspaces

### Sanctum SPA Auth

Cookie-based session (not Bearer token). No tokens stored in localStorage. CSRF protection automatic.

Optional OAuth (Google / GitHub) via Laravel Socialite — addable without architecture changes.

### Workspaces

Each user gets a **personal workspace** on registration. They can also create team workspaces and invite others.

A diagram belongs to exactly one workspace. Moving a diagram = changing its `workspace_id`.

### Roles

| Role | View | Edit | Manage members | Delete workspace |
|------|------|------|----------------|-----------------|
| viewer | ✓ | — | — | — |
| editor | ✓ | ✓ | — | — |
| owner  | ✓ | ✓ | ✓ | ✓ |

Roles enforced via Laravel Gates on every mutation. Frontend receives the current user's role in `GET /api/user` and disables editing UI for `viewer`.

### Invitations

```
POST /api/workspaces/{id}/invitations  { email, role }
```

Laravel sends an email with a signed invitation link. The invitee registers or logs in → automatically added to the workspace with the specified role.

### Frontend Auth Guard

New routes: `/login`, `/register` (outside editor-page, no providers).

React Router loader on all editor routes: calls `GET /api/user` → if 401, redirect to `/login`. On successful login, redirect back to the original URL.

---

## Implementation Order

| # | Sub-project | Depends on |
|---|-------------|------------|
| 1 | Laravel backend: auth + REST API + PostgreSQL | — |
| 2 | Frontend: `BackendStorageProvider` + auth guard | Sub-project 1 |
| 3 | Laravel Reverb + `useRealtimeSync` hook | Sub-project 2 |
| 4 | Workspaces UI + invitation flow | Sub-project 2 |

Each sub-project is independently shippable and testable before the next begins.

---

## Key Files to Create / Modify

### New (Laravel — separate repository `chartdb-api`)
- `routes/api.php` — all API routes
- `app/Http/Controllers/Auth/` — register, login, logout
- `app/Http/Controllers/DiagramController.php`
- `app/Http/Controllers/TableController.php` — and per entity
- `app/Events/TableUpdated.php` — and per entity event
- `app/Models/` — User, Workspace, Diagram, DBTable, etc.
- `database/migrations/` — all tables above
- `config/broadcasting.php` — Reverb config

### New (Frontend)
- `src/context/storage-context/backend-storage-provider.tsx`
- `src/hooks/use-realtime-sync.ts`
- `src/lib/echo.ts` — Laravel Echo singleton
- `src/pages/login-page/login-page.tsx`
- `src/pages/register-page/register-page.tsx`
- `src/router.tsx` — add auth guard loader, `/login`, `/register` routes

### Modified (Frontend)
- `src/pages/editor-page/editor-page.tsx` — replace `StorageProvider` with `BackendStorageProvider`, mount `useRealtimeSync`
- `src/router.tsx` — auth guard
- `package.json` — add `laravel-echo`, `pusher-js`

### Removed (Frontend)
- `src/context/storage-context/storage-provider.tsx` — Dexie removed
