# Manufacturing ERP Platform

A full-stack manufacturing execution system (MES) built for biotech/pharmaceutical production environments. Manages the complete lifecycle of work orders — from BOM configuration and document templating through QR-scanned workflow execution to NetSuite ERP synchronization — all with sub-second UI updates powered by a custom realtime caching infrastructure.

> **Tech Stack:** Next.js 15 (App Router) · React 19 · Fastify · Supabase (Postgres + Realtime) · Upstash Redis · NetSuite MCP · TypeScript · Tailwind CSS · TanStack Query

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Architecture Diagram](#system-architecture-diagram)
- [Data Flow](#data-flow)
  - [Realtime Cache Sync Pipeline](#1-realtime-cache-sync-pipeline)
  - [Work Order Lifecycle](#2-work-order-lifecycle)
  - [LLM + MCP Agentic Loop](#3-llm--mcp-agentic-loop)
- [Core Features](#core-features)
  - [Inventory Management](#inventory-management)
  - [Bill of Materials (BOM)](#bill-of-materials-bom)
  - [Work Orders](#work-orders)
  - [Document Builder](#document-builder)
  - [Workflow Execution Engine](#workflow-execution-engine)
  - [QR Code System](#qr-code-system)
  - [AI Chat with RAG + MCP](#ai-chat-with-rag--mcp)
  - [NetSuite Integration](#netsuite-integration)
  - [Admin Panel](#admin-panel)
- [Realtime Caching Infrastructure](#realtime-caching-infrastructure)
  - [Cache Architecture Diagram](#cache-architecture-diagram)
  - [Relationship Index System](#relationship-index-system)
  - [Cascade SSE Propagation](#cascade-sse-propagation)
- [Server Architecture](#server-architecture)
  - [API Layer](#api-layer)
  - [Background Workers](#background-workers)
  - [Authentication](#authentication)
  - [Message Queue (PGMQ)](#message-queue-pgmq)
- [Frontend Architecture](#frontend-architecture)
  - [Component System](#component-system)
  - [Feature Module Pattern](#feature-module-pattern)
  - [Provider Stack](#provider-stack)
- [Database Schema](#database-schema)
  - [Entity Relationship Diagram](#entity-relationship-diagram)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)

---

## Architecture Overview

Biolog is a two-process application:

| Process | Framework | Port | Role |
|---------|-----------|------|------|
| **Next.js App** | Next.js 15 (Turbopack) | 3000 | SSR/RSC pages, API routes for Supabase auth, static assets, service worker |
| **Fastify API Server** | Fastify 5 | 4000 | Business logic API, SSE streaming, background workers, NetSuite integration, LLM orchestration |

Both processes share the same TypeScript codebase and type definitions. The Fastify server runs as a standalone Node process (`src/server/server.ts`) with its own middleware stack, route registry, and worker pool.

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                    │
│                                                                         │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
│   │   Browser     │    │  Mobile PWA  │    │  LLM Client  │             │
│   │  (React 19)   │    │  (Serwist)   │    │  (AI Chat)   │             │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘             │
│          │                   │                   │                      │
└──────────┼───────────────────┼───────────────────┼──────────────────────┘
           │                   │                   │
           ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          NEXT.JS 15 (PORT 3000)                         │
│                                                                         │
│   ┌─────────────┐  ┌──────────────┐  ┌─────────────┐                  │
│   │ App Router   │  │ Server Comps │  │ Supabase    │                  │
│   │ (RSC + SSR)  │  │ (Data Fetch) │  │ SSR Auth    │                  │
│   └─────────────┘  └──────────────┘  └─────────────┘                  │
│                                                                         │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       FASTIFY API SERVER (PORT 4000)                    │
│                                                                         │
│   ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌───────────────┐       │
│   │ REST API │  │ SSE Stream│  │ LLM + MCP │  │  Background   │       │
│   │ Routes   │  │ /api/sse  │  │ Orchestr. │  │   Workers     │       │
│   └────┬─────┘  └─────┬─────┘  └─────┬─────┘  └───────┬───────┘       │
│        │               │              │                 │               │
│   ┌────┴───────────────┴──────────────┴─────────────────┴────┐         │
│   │              Middleware Stack                              │         │
│   │  CORS · Rate Limiting · JWT Auth · API Key Auth           │         │
│   └──────────────────────────┬────────────────────────────────┘         │
│                              │                                          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
┌──────────────────┐ ┌─────────────────┐ ┌──────────────────┐
│    Supabase      │ │  Upstash Redis  │ │    NetSuite      │
│  (PostgreSQL)    │ │   (Cache)       │ │   (ERP via MCP)  │
│                  │ │                 │ │                  │
│ • Row-Level Sec. │ │ • Entity Cache  │ │ • OAuth 2.0      │
│ • Realtime       │ │ • FK Indexes    │ │ • JSON-RPC MCP   │
│ • PGMQ Queues   │ │ • Session Store │ │ • SuiteQL        │
│ • Auth           │ │                 │ │ • Work Orders    │
└──────────────────┘ └─────────────────┘ └──────────────────┘
```

---

## Data Flow

### 1. Realtime Cache Sync Pipeline

This is the core innovation — a transparent caching layer where **no manual cache invalidation is ever needed**. Database writes automatically propagate to Redis and then to connected browser tabs via SSE.

```
Database Write (any client)
       │
       ▼
┌──────────────────────────────────┐
│   Supabase Realtime              │
│   (postgres_changes event)       │
│   Subscribed tables:             │
│   work_order, inventory_item,    │
│   assembly_build, doc_template,  │
│   + 12 more tables               │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│   RealtimeCacheSyncWorker        │
│   (src/server/workers/)          │
│                                  │
│   1. Parse event type            │
│      (INSERT/UPDATE/DELETE)      │
│   2. Handle soft deletes:        │
│      deleted_at null→val = DEL   │
│      deleted_at val→null = INS   │
│   3. Update Redis entity cache   │
│      Key: {schema}:{table}:{id}  │
│   4. Maintain FK index sets      │
│      Key: fk:{parent}:{id}:{child}│
│   5. Emit SSE CacheEvent         │
│   6. Cascade SSE to parent       │
│      entities (recursive FK walk) │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│   SSE Cache Events Service       │
│   (EventEmitter pub/sub)         │
│                                  │
│   event: cache:{table}           │
│   data: { schema, table, id,    │
│           eventType, data,       │
│           timestamp }            │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│   Browser (SSEProvider)          │
│                                  │
│   1. Parse SSE event             │
│   2. Invalidate TanStack Query   │
│      cache (mark stale)          │
│   3. Notify entity subscribers   │
│      → Detail pages refetch      │
│   4. Notify table subscribers    │
│      → List pages refetch        │
└──────────────────────────────────┘
```

### 2. Work Order Lifecycle

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  Create WO  │────▶│  Released   │────▶│  Execute     │────▶│  Complete     │
│  (from BOM) │     │  (Pending)  │     │  Workflow    │     │  Work Order   │
└─────────────┘     └─────────────┘     └──────────────┘     └───────┬───────┘
                                                                     │
                                         ┌───────────────────────────┤
                                         ▼                           ▼
                                  ┌──────────────┐          ┌───────────────┐
                                  │ Assembly     │          │ NetSuite Sync │
                                  │ Build        │          │ (via PGMQ)    │
                                  │ Created      │          │               │
                                  └──────┬───────┘          │ Job queued →  │
                                         │                  │ Worker polls →│
                                         ▼                  │ API call →    │
                                  ┌──────────────┐          │ Status update │
                                  │ Inventory    │          └───────────────┘
                                  │ Ledger       │
                                  │ Transactions │
                                  │ (Debit/Credit│
                                  │  per comp.)  │
                                  └──────────────┘
```

**Workflow execution steps** (defined in Doc Builder):
1. **Text steps** — Read-only instructions for operators
2. **Input steps** — User-entered data, system auto-fills, QR scans, or manager verification signatures
3. **Component scan steps** — QR code scanning with lot/instance-level tracking and quantity confirmation
4. **Component list steps** — Batch scan all BOM components with progress tracking

### 3. LLM + MCP Agentic Loop

The AI chat feature connects a locally-hosted LLM (Mistral 7B via vLLM) to the application database via RAG and to NetSuite via the Model Context Protocol (MCP).

```
┌──────────────┐
│  User Query   │
│  "What's the  │
│  stock of     │
│  PM5?"        │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────┐
│   RAG Orchestrator               │
│   (src/llm/server/rag/)          │
│                                  │
│   1. Extract entities            │
│      (item numbers, WO numbers,  │
│       quantities)                │
│   2. Detect query type           │
│      (inventory/BOM/work order)  │
│   3. Query Supabase for context  │
│   4. Manage session history      │
│   5. Build system prompt with    │
│      domain glossary + context   │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│   MCP Execution Loop             │
│   (src/llm/server/mcp/)          │
│                                  │
│   IF user connected to NetSuite: │
│                                  │
│   1. Initialize MCP client       │
│   2. Discover available tools    │
│      via JSON-RPC tools/list     │
│   3. Convert MCP → OpenAI format │
│   4. Call LLM with tool defs     │
│   5. Execute tool calls via MCP  │
│      (ns_runCustomSuiteQL,       │
│       ns_getRecord, etc.)        │
│   6. Return results to LLM      │
│   7. Loop until LLM finishes    │
│      (max 10 iterations)         │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│   NetSuite MCP Server            │
│   (AI Connector endpoint)        │
│                                  │
│   JSON-RPC 2.0 over HTTPS       │
│   OAuth 2.0 per-user tokens      │
│   Endpoint:                      │
│   {acct}.suitetalk.api.netsuite  │
│   .com/services/mcp/v1/all      │
└──────────────────────────────────┘
```

---

## Core Features

### Inventory Management
- Multi-level inventory tracking: **Item → Lot → Instance**
- Real-time stock quantities via cached ledger balances
- Category-based navigation with search
- Detail pages with lot breakdown, ledger history, and NetSuite sync status
- Cycle count support for physical inventory reconciliation

### Bill of Materials (BOM)
- Hierarchical BOM structures (assembly → components)
- Multi-level BOM resolution (e.g., product → solutions → raw chemicals)
- Scaled quantity calculations for batch production
- BOM-linked document templates
- Component cost rollup

### Work Orders
- Full lifecycle management: **Pending → Released → In Progress → Completed**
- Created from BOM with auto-populated component lists
- Linked assembly builds with inventory ledger postings (debit consumed components, credit produced assembly)
- Parent/child work order hierarchies
- Mobile-responsive execution interface
- Detail pages with components, documents, assembly builds, and status timeline

### Document Builder
- Visual drag-and-drop template editor with a **48×66 grid canvas**
- Block types: Text, Input (user/system/QR/verification), Image, Table, Component Scan
- **Data binding system** — bind template fields to BOM variables, component data, user inputs, and document metadata via a variable registry
- Canvas settings: margins, grid scale, wireframe toggle
- Properties panel for block styling, positioning, and configuration
- Save/overwrite/save-as flow for template versioning

### Workflow Execution Engine
- Step-by-step execution of document template workflows on the manufacturing floor
- Step types:
  - **Text** — Display instructions to operators
  - **Input** — Capture user data, auto-fill system fields (date, operator, lot numbers), or verification signatures
  - **Component Scan** — QR scan with lot/instance-level validation against inventory
  - **Component List** — Batch scan all BOM components with progress tracking and quantity confirmation
- System field resolution (UUID input keys → semantic keys)
- Completion triggers assembly build creation and optional NetSuite sync
- Progress tracking with step completion state

### QR Code System
- QR code generation for inventory items, lots, and instances
- Camera-based QR scanning via `html5-qrcode` with URL-format parsing
- QR validation against expected item/lot/instance hierarchy
- Deep linking: `/qr/[code]` routes resolve QR codes to item detail pages

### AI Chat with RAG + MCP
- **RAG pipeline**: Entity extraction → query type detection → Supabase context retrieval → domain glossary injection
- **MCP integration**: NetSuite AI Connector for live ERP queries (SuiteQL, record lookup, saved searches)
- **Agentic tool loop**: LLM can autonomously call NetSuite tools, receive results, and reason over them (up to 10 iterations)
- Session-based conversation history with entity tracking (last item, last quantity, last work order)
- Per-user NetSuite OAuth tokens with encrypted storage
- Conditional MCP enablement based on user's NetSuite connection status

### NetSuite Integration
- **OAuth 2.0 flow**: Init → redirect to NetSuite → callback with token exchange → encrypted storage
- **Sync queue** (PGMQ): Work orders and assembly builds queued for async sync
- **Background worker**: Polls queue, processes one job at a time (respects NetSuite's concurrency limit), updates sync status
- **Sync states**: `queued` → `syncing` → `synced` | `error`
- **Polymorphic integration tracking**: `integration_netsuite_data` table with `entity_type` + `entity_id` pattern
- Inventory sync from NetSuite for item/lot/instance data

### Admin Panel
- Employee management (CRUD, role assignment)
- Role & permission management (RBAC)
- Integration management (NetSuite connection status, OAuth flows)
- NetSuite inventory sync controls
- Admin dashboard statistics

---

## Realtime Caching Infrastructure

### Cache Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       Redis (Upstash)                           │
│                                                                 │
│   ENTITY CACHE                     RELATIONSHIP INDEXES         │
│   ─────────────                    ────────────────────         │
│   public:work_order:{uuid}         fk:work_order:{uuid}:       │
│     → { full row JSON }              work_order_component       │
│                                      → SET { comp_id_1,        │
│   public:inventory_item:{uuid}         comp_id_2, ... }        │
│     → { full row JSON }                                        │
│                                    fk:inventory_item:{uuid}:    │
│   public:assembly_build:{uuid}       inventory_item_lot         │
│     → { full row JSON }              → SET { lot_id_1, ... }   │
│                                                                 │
│   16 tables × all active records   Auto-maintained via          │
│   (~20ms read latency)             Realtime event handler       │
└─────────────────────────────────────────────────────────────────┘
```

### Relationship Index System

The cache maintains **Redis SET indexes** for every foreign key relationship defined in `types/schema-relationships.ts`. This file is auto-generated from the database schema (`npm run types`).

```typescript
// Example: Hydrating a work order with all its children
const { parent, children } = await worker.hydrateWithChildren(
  'work_order',
  workOrderId,
  ['work_order_component', 'work_order_document', 'assembly_build']
);
// Returns parent entity + arrays of related entities, all from Redis
```

**Index operations on every Realtime event:**
- **INSERT**: `SADD` — Add child ID to parent's index set
- **UPDATE**: `SREM` old parent + `SADD` new parent (if FK changed)
- **DELETE**: `SREM` — Remove child ID from parent's index set

### Cascade SSE Propagation

When a child entity changes, the system recursively walks up the FK chain and emits SSE events for every ancestor. This means a detail page for a work order will automatically refresh when one of its components is updated, even though the work_order row itself didn't change.

```
inventory_ledger_balance changes
  └─▶ CASCADE to inventory_item (via item_id FK)
  └─▶ CASCADE to inventory_item_lot (via lot_id FK)
        └─▶ CASCADE to inventory_item (via lot's item_id FK)
```

Special handling for polymorphic tables (`integration_netsuite_data`) which use `entity_type` + `entity_id` instead of standard foreign keys.

---

## Server Architecture

### API Layer

The Fastify server registers domain-scoped route groups:

| Route Group | Prefix | Purpose |
|------------|--------|---------|
| **NetSuite** | `/api/netsuite/*` | OAuth flow, work order sync, assembly build sync, inventory preview |
| **Work Orders** | `/api/app/work-order/*` | Create, delete, verify, complete, release |
| **Workflows** | `/api/app/workflow/*` | Save step data, complete workflows |
| **LLM** | `/api/app/llm/*` | Chat completions, models, sessions, MCP debug |
| **SSE** | `/api/sse/*` | Cache update stream |
| **Cache** | `/api/cache/*` | Cache management endpoints |

### Background Workers

Two singleton workers run alongside the Fastify server:

1. **RealtimeCacheSyncWorker** — Subscribes to Supabase Realtime for 16 tables, syncs to Redis, emits SSE events
2. **NetSuiteSyncWorker** — Polls PGMQ queue, processes NetSuite sync jobs sequentially

Both workers implement graceful shutdown via `SIGTERM` handler.

### Authentication

Dual authentication strategy:
- **JWT** (internal): Supabase auth tokens validated via `supabase.auth.getUser()`
- **API Key** (remote/LLM): `X-API-Key` header checked against configured keys

```typescript
// Middleware chain
CORS → Rate Limiting → JWT/API Key Auth → Route Handler
```

### Message Queue (PGMQ)

NetSuite operations are queued via [PGMQ](https://github.com/tembo-io/pgmq) (PostgreSQL-native message queue) to handle:
- Async processing without blocking the HTTP response
- Single-concurrency processing (NetSuite API limit)
- Visibility timeouts for job retry
- Metrics and monitoring

```typescript
// Queue a NetSuite sync job
await pgmq.send({
  jobType: 'work_order',
  entityId: workOrderId,
  employeeId: currentUser.id,
});
```

---

## Frontend Architecture

### Component System

Three-tier UI component architecture:

```
src/app/components/
├── shadcn/          # Base primitives (Radix UI + shadcn/ui)
│   ├── ui/          # 50+ components (Dialog, Select, Tabs, etc.)
│   ├── hooks/       # useIsMobile, useSidebar
│   └── lib/         # cn() utility
└── ui/              # Application design system
    ├── primitives/  # Button, Input, Card, Heading, Text, etc.
    ├── layouts/     # PageLayout, Grid, Stack, Flex, Section
    ├── cards/       # StatCard, DataCard, ListCard, InteractiveCard
    ├── navigation/  # BackLink, TabNavigation, NavbarCard
    ├── common/      # Modal, StatusBadge, ConfirmationDialog
    ├── forms/       # SearchAutocomplete
    ├── tables/      # DataTable
    └── progress/    # LinearProgress, CircularProgress
```

### Feature Module Pattern

Each feature is a self-contained module with its own components, data layer, and types:

```
src/app/features/
├── work-orders/
│   ├── create-work-order-modal/   # Modal + form + data hooks
│   ├── work-order-detail/         # Detail page feature
│   ├── work-order-active-list/    # Filtered list + data hooks
│   ├── workflow-execution/        # Step-by-step engine
│   │   ├── index.tsx              # Main component
│   │   ├── types.ts               # Feature-scoped types
│   │   ├── data/                  # Server actions + API calls
│   │   └── helpers/               # System field resolution
│   └── ...
├── doc-builder/
│   ├── DocBuilder.tsx             # Workspace orchestrator
│   ├── template-canvas/           # Grid canvas renderer
│   ├── block-palette/             # Drag source for blocks
│   ├── properties-panel/          # Block config editor
│   ├── workflow-builder/          # Step sequence editor
│   ├── workflow-preview/          # Live preview
│   └── data/                      # Save actions + variable registry
├── inventory/
├── bom/
├── qr/
├── admin/
└── navbar/
```

### Provider Stack

```
<TanStackProvider>           # React Query client + devtools
  <ThemeProvider>             # Dark/light mode (next-themes)
    <ToastProvider>           # Sonner toast notifications
      <SSEProvider>           # Global SSE connection + subscriptions
        <NavBar />
        {children}            # Page content
      </SSEProvider>
    </ToastProvider>
  </ThemeProvider>
</TanStackProvider>
```

The **SSEProvider** maintains a single SSE connection per browser tab with:
- Automatic reconnection (exponential backoff, max 10 attempts)
- Tab visibility-based reconnection
- Two subscription tiers:
  - `subscribeToEntity(table, id, callback)` — For detail page refetch
  - `subscribeToTable(table, callback)` — For list page refetch
- TanStack Query cache invalidation on every event

---

## Database Schema

### Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│     department      │       │      location        │
└──────────┬──────────┘       └──────────┬───────────┘
           │                             │
           │  ┌──────────────────────────┤
           │  │                          │
           ▼  ▼                          ▼
┌─────────────────────┐       ┌─────────────────────┐
│   inventory_item    │◄──────│  unit_of_measurement │
│                     │       └─────────────────────┘
│  item_number        │
│  display_name       │
│  quantity_on_hand   │
│  cost               │
│  attributes (JSON)  │
└──────────┬──────────┘
           │
    ┌──────┼──────────────────────┐
    │      │                      │
    ▼      ▼                      ▼
┌────────────────┐  ┌──────────────────────┐  ┌────────────────────────┐
│inventory_item  │  │  bill_of_materials   │  │     work_order         │
│    _lot        │  │                      │  │                        │
│                │  │  assembly_item_id ──▶│  │  assembly_item_id ──▶  │
│  lot_number    │  │  doc_template_id     │  │  bom_id ──▶            │
│  item_id ──▶   │  └──────────┬───────────┘  │  status (enum)         │
└──────┬─────────┘             │              │  planned_quantity      │
       │                       ▼              └──────────┬─────────────┘
       ▼              ┌────────────────────┐             │
┌────────────────┐    │  bill_of_materials │      ┌──────┼───────┐
│inventory_item  │    │    _component      │      │      │       │
│ _lot_instance  │    │                    │      ▼      ▼       ▼
│                │    │  component_item_id  │  ┌──────┐ ┌──────┐ ┌──────────┐
│  lot_id ──▶    │    │  quantity_per       │  │ WO   │ │ WO   │ │ assembly │
│  instance_num  │    │  uom_id            │  │ comp │ │ doc  │ │ _build   │
│  qr_code       │    └────────────────────┘  └──────┘ └──────┘ └────┬─────┘
└────────────────┘                                                    │
                                                                      ▼
       ┌──────────────────────────────────┐                 ┌─────────────────┐
       │      inventory_ledger            │                 │ assembly_build  │
       │                                  │                 │   _component    │
       │  item_id, lot_id, lot_instance_id│◄────────────────│                 │
       │  quantity (+ or -)               │                 │  quantity_used  │
       │  assembly_build_id               │                 └────────┬────────┘
       │  transaction_type                │                          │
       └──────────────────────────────────┘                          ▼
                                                            ┌─────────────────┐
       ┌──────────────────────────────────┐                 │ assembly_build  │
       │      doc_template                │                 │   _component    │
       │                                  │                 │   _assignment   │
       │  name, category, settings        │                 │                 │
       │  (page size, margins, grid)      │                 │  lot_instance_id│
       └──────────┬───────────────────────┘                 │  quantity       │
                  │                                         └─────────────────┘
           ┌──────┴──────┐
           ▼             ▼
   ┌──────────────┐ ┌──────────────────┐
   │doc_template  │ │doc_template      │
   │  _block      │ │ _workflow_step   │
   │              │ │                  │
   │ blockType    │ │ step_number      │
   │ blockData    │ │ step_type        │
   │ gridPosition │ │ template_block_id│
   └──────────────┘ │ config (JSON)    │
                    └──────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  RBAC & AUTH                                                      │
│                                                                    │
│  employee ──▶ employee_role ──▶ role ──▶ role_permission          │
│                 │                                                  │
│                 ├──▶ department                                    │
│                 ├──▶ location                                      │
│                 └──▶ subsidiary                                    │
│                                                                    │
│  employee ──▶ integration_netsuite_user ──▶ integration_netsuite  │
│                (encrypted OAuth tokens)       ──▶ integration     │
│                                                                    │
│  integration_netsuite_data (polymorphic: entity_type + entity_id) │
│    sync_status: queued | syncing | synced | error                 │
│    netsuite_id, data (JSON)                                        │
│                                                                    │
│  audit_log (employee_id, action, entity_type, entity_id, etc.)    │
└──────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- **Soft deletes** (`deleted_at` column) — Records are never hard-deleted; the cache layer detects soft delete transitions and treats them as DELETE events
- **JSONB columns** for flexible data: `attributes` on inventory items, `blockData` on template blocks, `config` on workflow steps, `data` on NetSuite sync records
- **Ledger-based inventory** — All inventory changes are immutable ledger entries (debits and credits), balances are materialized views
- **Auto-generated schema relationships** — `npm run types` generates both the TypeScript database types and the FK relationship map used by the cache system

---

## Project Structure

```
biolog/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── (pages)/
│   │   │   ├── (public)/             # Login, unauthenticated routes
│   │   │   └── (private)/            # Auth-required routes
│   │   │       ├── home/             # Dashboard
│   │   │       ├── inventory/        # Inventory management
│   │   │       ├── bom/              # Bill of materials
│   │   │       ├── work-order/       # Work orders + execution
│   │   │       ├── doc-builder/      # Document template editor
│   │   │       ├── ai-chat/          # LLM chat interface
│   │   │       ├── cycle-count/      # Inventory counting
│   │   │       ├── qr/[code]/        # QR code resolution
│   │   │       ├── admin/            # Admin panel
│   │   │       └── unauthorized/     # Access denied
│   │   ├── components/
│   │   │   ├── shadcn/               # Base UI primitives
│   │   │   └── ui/                   # Application design system
│   │   ├── features/                 # Feature modules (see above)
│   │   ├── providers/                # React context providers
│   │   │   ├── sse/                  # SSE connection + subscriptions
│   │   │   ├── supabase/             # Browser + server clients
│   │   │   ├── tanstack/             # React Query setup
│   │   │   ├── theme/                # Dark/light mode
│   │   │   └── toast/                # Notifications
│   │   └── sw.ts                     # Service worker (Serwist PWA)
│   │
│   ├── server/                       # Fastify API server
│   │   ├── server.ts                 # Entry point
│   │   ├── api/                      # Route handlers
│   │   │   ├── netsuite/             # NetSuite integration routes
│   │   │   ├── app/                  # App business logic routes
│   │   │   ├── sse/                  # SSE streaming
│   │   │   └── cache/                # Cache management
│   │   ├── config/                   # Env validation (Zod), client factory
│   │   ├── middleware/               # CORS, rate limit, JWT auth
│   │   ├── workers/                  # Background workers
│   │   ├── services/                 # PGMQ, SSE event bus
│   │   └── utils/                    # Encryption, audit context
│   │
│   └── llm/                          # LLM integration
│       ├── server/
│       │   ├── client.ts             # vLLM API client
│       │   ├── rag/                  # RAG pipeline
│       │   │   ├── index.ts          # Orchestrator
│       │   │   ├── queries.ts        # Supabase data queries
│       │   │   ├── context.ts        # Prompt building
│       │   │   ├── glossary.ts       # Domain terminology
│       │   │   └── history.ts        # Session management
│       │   └── mcp/                  # NetSuite MCP integration
│       │       ├── netsuite-client.ts # MCP JSON-RPC client
│       │       ├── tools.ts          # MCP → OpenAI tool conversion
│       │       └── execution-loop.ts # Agentic tool calling loop
│       ├── hooks/                    # useLLM React hook
│       └── api.ts                    # Client-side LLM API
│
├── types/
│   ├── database.ts                   # Auto-generated Supabase types
│   ├── index.ts                      # Re-exported DB types
│   └── schema-relationships.ts       # Auto-generated FK map
│
├── supabase/
│   ├── config.toml                   # Supabase project config
│   └── migrations/                   # SQL migrations
│
├── scripts/
│   └── generate-schema-relationships.ts  # FK map generator
│
├── seed/                             # Database seed data
├── public/                           # Static assets
├── next.config.ts                    # Next.js + Serwist PWA config
├── vercel.json                       # Vercel deployment config
└── package.json                      # Dependencies + scripts
```

---

## Getting Started

### Prerequisites

- Node.js 20+
- npm
- Supabase project (with PGMQ extension enabled)
- Upstash Redis instance
- (Optional) NetSuite account with AI Connector enabled
- (Optional) vLLM server running Mistral 7B

### Environment Setup

Create `.env.app` for Next.js:
```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
NEXT_PUBLIC_API_URL=http://localhost:4000
```

Create `.env.server` for Fastify:
```env
PORT=4000
NODE_ENV=development
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
UPSTASH_REDIS_REST_URL=https://your-redis.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-redis-token
LLM_BASE_URL=http://localhost:8000
MODEL_NAME=mistralai/Mistral-7B-Instruct-v0.2
```

### Development

```bash
# Install dependencies
npm install

# Run both Next.js and Fastify server concurrently
npm run dev:all

# Or run individually:
npm run dev          # Next.js on :3000
npm run dev:server   # Fastify on :4000

# Generate types from Supabase schema
npm run types
```

### Production

```bash
# Build Next.js
npm run build

# Build and start Fastify server
npm run start:server:prod

# Start Next.js
npm run start
```

Deployed on **Vercel** (Next.js frontend) with the Fastify server running separately.
