# Wholesale & B2B Commerce — Phased Development Plan

> Project: 407-wholesale-b2b-commerce · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into an implementation specification. The platform is an **AI-native, API-first, headless-first wholesale/B2B commerce engine** whose core differentiator is reliable enforcement of negotiated pricing, custom catalogues, and procurement workflows across every channel, with AI embedded at the workflow level (conversational reorder, PO ingestion, price-drift detection).

The database design adopts **Data Model Suggestion 3 — Hybrid Relational + JSONB** as the canonical schema. Rationale: it keeps strong referential integrity and ACID guarantees on financial/order/pricing data while using PostgreSQL JSONB for variable product attributes, workflow configs, integration payloads, AI metadata, and lightweight audit history — the exact pattern used by modern headless commerce platforms (Medusa, Saleor) the README positions this project against. Where Suggestion 2's event-sourcing benefits matter most (auditable price/quote/order history), they are captured pragmatically via the `history`/`negotiations`/`step_history` JSONB arrays plus an append-only `audit_events` table, rather than full CQRS. Suggestion 4's graph traversal needs (entitlement resolution) are met with recursive CTEs and a cached resolver service.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The competitive set this project targets headless-first (commercetools, Medusa, Saleor, Catalyst) is TypeScript-centric; storefronts are Next.js/Remix. A single TS language across API, AI tooling, and the MCP server reduces context-switching and lets the OpenAPI/GraphQL/Zod schemas be shared. |
| Runtime / package manager | Node 22 + pnpm workspaces | pnpm gives fast, disk-efficient installs and first-class monorepo support for the API, worker, MCP server, and shared packages. |
| Monorepo tooling | Turborepo | Caches builds/tests across the `apps/*` and `packages/*` boundary; matches next-forge / Catalyst conventions familiar to B2B headless teams. |
| API framework | Fastify | Highest-throughput Node HTTP framework with first-class JSON Schema validation (aligns with JSON Schema 2020-12 in standards.md), native OpenAPI generation via `@fastify/swagger`, and a plugin model that cleanly separates concerns. |
| REST contract | OpenAPI 3.1 (generated) | Standards.md mandates OpenAPI 3.1 as the de-facto B2B API contract; every integration partner expects it. |
| GraphQL | Pothos (code-first) over GraphQL Yoga | Storefronts (commercetools, Saleor, Shopify) expose GraphQL Storefront APIs; Pothos gives type-safe, code-first schema generation that stays in sync with TS types. |
| Webhooks / events | CloudEvents 1.0 envelope, HMAC-SHA256 signed | Standards.md identifies CloudEvents as the converging commerce event format; HMAC signing matches Shopify/BigCommerce conventions. |
| Database | PostgreSQL 16 | Suggestion 3 hybrid model; JSONB + GIN + generated columns + RLS in one engine. Avoids operating a separate document/graph store. |
| ORM / query builder | Drizzle ORM | First-class TypeScript types, transparent SQL (critical for the hand-tuned price-resolution CTEs), JSONB column support, and SQL-level migrations — no hidden query magic that would obscure the pricing logic. |
| Migrations | drizzle-kit + raw SQL for RLS/indexes | drizzle-kit generates table migrations; RLS policies, GIN indexes, and partitioning are applied via committed raw-SQL migration files. |
| Connection pooling | PgBouncer (transaction mode) | Required for multi-tenant shared-schema workloads with many concurrent tenants. |
| Cache / queues | Redis 7 + BullMQ | Redis caches resolved entitlements and effective prices (the hot path); BullMQ runs async jobs: webhook delivery, ERP sync, AI tasks (PO ingestion, reorder prediction), email PO polling. |
| Search | PostgreSQL FTS for MVP; Meilisearch adapter behind an interface | FTS covers MVP catalogue search with zero extra infra; the search interface lets Meilisearch be dropped in for faceted search later without API changes. |
| AI / LLM | Vercel AI SDK with provider abstraction (Anthropic default) | AI features are first-class (PO ingestion, NL ordering, quote drafts, anomaly detection). The AI SDK gives structured-output (`generateObject` + Zod), tool calling, and provider failover. Prompt caching enabled on long catalogue/contract context. |
| Embeddings / vector | pgvector extension | SKU classification, NL product discovery, and similar-buyer features need vector similarity; pgvector keeps it inside Postgres. |
| Auth | OAuth 2.1 + OIDC (buyer SSO) via a pluggable IdP adapter; JWT (RFC 9068) access tokens | Standards.md mandates OAuth 2.1/OIDC; access tokens follow the JWT profile. Service-to-service uses client-credentials. |
| Authorization | Casbin (RBAC + ABAC) | Buyer roles, sales-rep impersonation scopes, and approval permissions need declarative policy. |
| Validation | Zod (shared `packages/contracts`) | One source of truth for request/response shapes, JSONB field schemas, AI structured output, and OpenAPI/GraphQL generation. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Standards.md flags MCP as a forward-looking differentiator for agentic/buyer-side ordering; expose catalogue/pricing/order tools. |
| Tax engine | Adapter interface; Avalara + Vertex + flat-rate impls | Hooks required by features.md; interface keeps providers swappable. |
| ERP | Adapter interface; NetSuite, QuickBooks, generic CSV bridge | features.md v1.1 scope; CSV bridge is the zero-cost default. |
| Containerisation | Docker + docker-compose | Self-hosted is a stated deployment mode; compose runs api + worker + postgres + redis + mailhog locally. |
| Testing | Vitest (unit/integration) + Testcontainers (real Postgres/Redis) + Supertest (HTTP) + Playwright (storefront e2e) | Vitest is fast and TS-native; Testcontainers gives real-DB integration tests for RLS and pricing; Playwright drives the reference storefront. |
| Code quality | ESLint + Prettier + TypeScript strict + `tsc --noEmit` | Standard TS toolchain; strict mode catches pricing/money type errors. |
| Money handling | `dinero.js` (integer minor units) + `NUMERIC` in DB | Never use floats for money; integer minor units in code, fixed-precision `NUMERIC` in storage. |
| Observability | OpenTelemetry traces + structured pino logs | Projection/queue lag and price-resolution latency must be observable. |
| Licence posture | Permissive (Apache-2.0); no OSL-3.0 code bundled | README/research mandate avoiding OSL-3.0 copyleft contamination. |

### Project Structure

```
wholesale-b2b-commerce/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.base.json
├── docker-compose.yml                # postgres, redis, pgbouncer, mailhog, api, worker
├── Dockerfile.api
├── Dockerfile.worker
├── .env.example
├── apps/
│   ├── api/                          # Fastify REST + GraphQL + webhooks
│   │   ├── src/
│   │   │   ├── server.ts             # app factory (testable, no listen)
│   │   │   ├── index.ts              # boot + listen
│   │   │   ├── plugins/              # auth, rls-context, error-handler, otel, rate-limit
│   │   │   ├── rest/                 # route modules grouped by domain
│   │   │   ├── graphql/              # Pothos schema, resolvers
│   │   │   ├── webhooks/             # inbound (EDI/email) + outbound dispatch
│   │   │   └── mcp/                  # MCP server transport mounted on API
│   │   └── test/
│   ├── worker/                       # BullMQ processors
│   │   └── src/jobs/                 # webhook-delivery, erp-sync, ai-*, po-email-poll
│   └── storefront/                   # Next.js reference storefront (thin; proves headless API)
├── packages/
│   ├── contracts/                    # Zod schemas + generated TS types (shared)
│   ├── db/                           # Drizzle schema, migrations, RLS SQL, seed
│   │   ├── src/schema/               # one file per domain (companies, catalogue, pricing, orders…)
│   │   ├── migrations/               # drizzle + raw SQL (rls, indexes, partitions, pgvector)
│   │   └── src/seed.ts
│   ├── domain/                       # pure business logic (no I/O)
│   │   ├── pricing/                  # price resolution engine
│   │   ├── ordering/                 # cart/order rules: minimums, case-pack, credit
│   │   ├── approvals/                # workflow evaluation
│   │   ├── quotes/                   # negotiation state machine
│   │   └── entitlements/             # catalogue/price-list visibility resolver
│   ├── services/                     # orchestration: repositories + domain + side effects
│   ├── ai/                           # LLM clients, prompt templates, structured-output schemas
│   ├── integrations/                 # tax + erp + edi + punchout adapters (interfaces + impls)
│   ├── events/                       # CloudEvents builders, webhook signing
│   └── auth/                         # OAuth/OIDC, JWT, Casbin policy
└── tools/
    └── openapi/                      # spec export + client codegen
```

---

## Phase 1: Foundation — Monorepo, Database, Multi-Tenancy, Auth Skeleton

### Purpose
Establish the runnable skeleton every later phase builds on: the pnpm/Turborepo workspace, a Dockerized Postgres+Redis stack, the Drizzle schema for core tenancy/account entities with Row-Level Security, the shared `contracts` (Zod) package, and a Fastify app that authenticates a request and pins the tenant context for RLS. After this phase the API boots, a health check passes, and a tenant-scoped query is provably isolated.

### Tasks

#### 1.1 — Workspace, tooling, and Docker stack

**What**: Bootstrap the monorepo, TypeScript strict config, lint/format, and a docker-compose dev stack.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`. Root `package.json` scripts: `dev`, `build`, `test`, `lint`, `typecheck`, `db:migrate`, `db:seed` (delegated through `turbo`).
- `tsconfig.base.json`: `strict: true`, `noUncheckedIndexedAccess: true`, `target: ES2023`, `module: NodeNext`, path aliases `@b2b/*` → `packages/*/src`.
- `docker-compose.yml` services: `postgres` (postgres:16, with `pgvector` image variant), `redis` (redis:7), `pgbouncer` (transaction mode), `mailhog` (SMTP capture for PO/email tests), `api`, `worker`. Postgres init script enables `pgcrypto`, `pgvector`.
- `.env.example` keys: `DATABASE_URL`, `DATABASE_URL_POOLED`, `REDIS_URL`, `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_PUBLIC_JWKS_URL`, `AI_PROVIDER`, `ANTHROPIC_API_KEY`, `SMTP_URL`, `WEBHOOK_SIGNING_SECRET`, `LOG_LEVEL`.

**Testing**:
- `Unit: tsconfig path alias resolves @b2b/contracts in a sample import`.
- `Integration (real): docker compose up postgres → pg connection succeeds, pgvector + pgcrypto extensions present`.
- `Smoke: pnpm build compiles all packages with zero TS errors`.

#### 1.2 — Core tenancy & account schema (Drizzle + RLS)

**What**: Implement `tenants`, `companies`, `company_addresses`, `buyers`, and RLS policies pinning all tenant-scoped tables to a session GUC.

**Design**:
Adopt Suggestion 3 hybrid tables. Drizzle schema (`packages/db/src/schema/accounts.ts`) mirrors this DDL (applied via migration):

```sql
CREATE TABLE tenants (
    tenant_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    slug        TEXT NOT NULL UNIQUE,
    settings    JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE companies (
    company_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    parent_id     UUID REFERENCES companies(company_id),
    name          TEXT NOT NULL,
    slug          TEXT NOT NULL,
    status        TEXT NOT NULL DEFAULT 'active',
    credit_limit  NUMERIC(12,2),
    payment_terms TEXT NOT NULL DEFAULT 'net_30',
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',   -- ISO 4217
    tax_id        TEXT,
    metadata      JSONB NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);
CREATE INDEX idx_companies_tenant ON companies (tenant_id, status);
CREATE INDEX idx_companies_metadata ON companies USING GIN (metadata);

CREATE TABLE company_addresses (
    address_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL,
    company_id   UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    address_type TEXT NOT NULL,            -- 'billing' | 'shipping'
    line1 TEXT NOT NULL, line2 TEXT, city TEXT NOT NULL, state TEXT,
    postal_code TEXT NOT NULL,
    country_code CHAR(2) NOT NULL,         -- ISO 3166-1
    is_default BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE buyers (
    buyer_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    company_id    UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    email         TEXT NOT NULL,
    first_name    TEXT NOT NULL,
    last_name     TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'buyer',   -- 'buyer'|'approver'|'admin'
    spending_limit NUMERIC(12,2),
    permissions   JSONB NOT NULL DEFAULT '[]',
    preferences   JSONB NOT NULL DEFAULT '{}',
    status        TEXT NOT NULL DEFAULT 'active',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- RLS: every tenant-scoped table
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON companies
  USING (tenant_id = current_setting('app.current_tenant', true)::UUID);
-- repeated for company_addresses, buyers, and all later tenant-scoped tables
```

- Connections run as a non-superuser role (`app_user`) so RLS is enforced (superusers bypass RLS).
- `payment_terms` constrained by app-level Zod enum: `net_7|net_15|net_30|net_60|net_90|cod|prepaid|po`.

**Testing**:
- `Unit: Zod CompanyCreate schema rejects invalid country_code (non ISO-3166 length)`.
- `Integration (real DB): set app.current_tenant=T1, insert company; set tenant=T2, SELECT returns zero rows (RLS isolation proven)`.
- `Integration (real DB): attempt insert with tenant_id mismatching session GUC → blocked by RLS WITH CHECK`.
- `Integration (real DB): cascade delete company removes its buyers and addresses`.

#### 1.3 — RLS request context + tenant resolution plugin

**What**: A Fastify plugin that opens a per-request DB transaction, sets `app.current_tenant` (and `app.current_buyer`) from the authenticated principal, and exposes a tenant-scoped Drizzle client on `request.db`.

**Design**:
```ts
interface RequestPrincipal {
  tenantId: string;
  subjectType: 'buyer' | 'sales_rep' | 'service' | 'ai_agent';
  subjectId: string;
  impersonatedBuyerId?: string;   // sales-rep order-on-behalf-of
  scopes: string[];
}
// plugin pseudocode
fastify.addHook('onRequest', resolvePrincipal);   // from JWT (phase 1.4)
fastify.decorateRequest('db', null);
fastify.addHook('preHandler', async (req) => {
  const tx = await pool.connect();
  await tx.query(`SET LOCAL app.current_tenant = $1`, [req.principal.tenantId]);
  req.db = drizzle(tx);                 // bound to this connection for the request
});
```
- Transaction-scoped `SET LOCAL` guarantees the GUC never leaks across pooled connections.
- Health/readiness routes bypass tenant context.

**Testing**:
- `Integration (mocked auth): request with tenant T1 token → request.db queries see only T1 rows`.
- `Integration: connection returned to pool after request; subsequent request with no tenant set sees zero rows (GUC reset)`.

#### 1.4 — Auth skeleton: JWT verification + Casbin RBAC

**What**: Verify OAuth 2.1 / OIDC bearer JWTs (RFC 9068) against a JWKS, map claims to `RequestPrincipal`, and gate routes with Casbin.

**Design**:
- `packages/auth`: `verifyAccessToken(token): Promise<RequestPrincipal>` using `jose` + cached JWKS. Required claims: `iss`, `aud`, `exp`, `tenant_id`, `sub`, `subject_type`, `scope`.
- Casbin model: RBAC with domains (tenant) + ABAC on `spending_limit`. Policy CSV seeded with roles: `buyer`(place_order, view_catalogue, manage_own_lists), `approver`(+approve_order), `admin`(+manage_users, manage_pricing), `sales_rep`(impersonate, place_order_on_behalf).
- Decorator `requireScope(scope: string)` and `requirePermission(obj, act)` preHandlers.

**Testing**:
- `Unit: valid signed JWT → principal with correct tenant/scopes`.
- `Unit: expired token → 401; wrong audience → 401`.
- `Unit: Casbin — buyer attempts approve_order → denied; approver → allowed`.
- `Integration: protected route without token → 401; with insufficient scope → 403`.

---

## Phase 2: Product Catalogue & Custom Catalogues

### Purpose
Build the product domain — products with JSONB attributes, variants, categories, images — and the custom-catalogue layer that controls which products a given company/group/region can see. This is the substrate pricing and ordering depend on. After this phase, products can be managed via REST/GraphQL and catalogue visibility rules (static and dynamic) resolve to product sets.

### Tasks

#### 2.1 — Product, variant, category, image schema

**What**: Implement the catalogue tables per Suggestion 3 with GIN-indexed JSONB attributes and a pgvector embedding column for later AI search.

**Design**:
```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    sku TEXT NOT NULL, name TEXT NOT NULL, slug TEXT NOT NULL, description TEXT,
    product_type TEXT NOT NULL DEFAULT 'simple',  -- simple|variable|bundle
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',   -- EA|CS|PK|...
    case_pack_qty INTEGER NOT NULL DEFAULT 1,
    min_order_qty INTEGER NOT NULL DEFAULT 1,
    weight_kg NUMERIC(10,3), gtin TEXT, brand TEXT,   -- GS1 GTIN
    status TEXT NOT NULL DEFAULT 'active',
    attributes JSONB NOT NULL DEFAULT '{}',
    tags TEXT[] NOT NULL DEFAULT '{}',
    ai_metadata JSONB NOT NULL DEFAULT '{}',
    embedding vector(1536),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);
CREATE INDEX idx_products_tenant ON products (tenant_id, status);
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_products_gtin ON products (gtin) WHERE gtin IS NOT NULL;
-- product_variants, product_images, categories, product_categories per Suggestion 3
```
- Zod `ProductAttributes` is an open record validated as `Record<string, JsonValue>`; per-tenant attribute schemas can be registered in `tenants.settings` and enforced via a CHECK using `pg_jsonschema` (optional, later).

**Testing**:
- `Unit: ProductCreate rejects case_pack_qty < 1`.
- `Integration: insert product with attributes {organic:true}; query WHERE attributes @> '{"organic":true}' returns it`.
- `Integration: duplicate (tenant_id, sku) → unique violation surfaced as 409`.

#### 2.2 — Product CRUD REST + GraphQL

**What**: `/v1/products` REST routes and `products`/`product` GraphQL queries + admin mutations.

**Design**:
- REST: `POST /v1/products`, `GET /v1/products/{id}`, `PATCH`, `DELETE` (soft → status=archived), `GET /v1/products?cursor=&limit=&tag=&attr.<k>=` with RFC 8288 `Link` pagination headers.
- Response envelope: `{ data, meta: { cursor, hasMore } }`. JSON per RFC 8259.
- GraphQL: `type Product`, `Query.products(filter, first, after): ProductConnection` (Relay cursor connections).
- Bulk import: `POST /v1/products/bulk` accepts CSV/NDJSON → enqueues a BullMQ import job (Phase 7 reuses).

**Testing**:
- `Integration (HTTP): POST product → 201 with id; GET → 200 matches`.
- `Integration: cursor pagination — create 25, page size 10, walk 3 pages, no dupes/omissions`.
- `Integration: filter attr.color=Navy returns only matching`.
- `E2E (GraphQL): products query returns Relay connection with pageInfo`.

#### 2.3 — Custom catalogues & visibility assignment

**What**: `catalogues`, `catalogue_products`, `catalogue_assignments`, with static and dynamic (JSONB-rule) membership.

**Design**:
- Tables per Suggestion 3. `catalogues.rules` JSONB: `{"type":"static"}` or `{"type":"dynamic","filters":{"brand":["Acme"],"attributes.organic":true,"tags":{"contains":"seasonal"}}}`.
- `assignTarget`: `{ target_type: 'company'|'customer_group'|'region', target_id }`.
- Dynamic-rule compiler: translate a `filters` object into a parameterised SQL predicate over `products` (whitelisted columns + `attributes->>` paths + `tags && ARRAY[...]`). Reject unknown operators.

**Testing**:
- `Unit: rule compiler turns {"attributes.organic":true} into attributes @> '{"organic":true}'`.
- `Unit: rule compiler rejects injection attempt in attribute key → throws`.
- `Integration: dynamic catalogue resolves to products matching filters; adding a matching product makes it appear`.

#### 2.4 — Entitlement (visibility) resolver

**What**: A cached domain service: given `(buyerId)`, return the set of catalogue IDs and thus visible product IDs.

**Design**:
```ts
interface EntitlementResolver {
  visibleCatalogues(ctx: ResolveCtx): Promise<string[]>;   // company + group + region assignments
  isProductVisible(ctx: ResolveCtx, productId: string): Promise<boolean>;
}
```
- Resolution = buyer → company (+ recursive parent companies via CTE) → group memberships → union of catalogue assignments; fall back to tenant default catalogue if none.
- Result cached in Redis key `ent:{tenant}:{company}` with TTL 300s; invalidated on assignment/membership change events.

**Testing**:
- `Integration: company assigned catalogue A; buyer sees A's products only`.
- `Integration: parent-company catalogue inherited by subsidiary buyer (recursive CTE)`.
- `Integration: cache hit avoids second DB query (spy on pool)`.
- `Integration: assignment change publishes invalidation → next resolve recomputes`.

---

## Phase 3: Pricing Engine — Price Lists, Quantity Breaks, Contract Pricing

### Purpose
Implement the **core differentiator**: reliable resolution and enforcement of negotiated pricing across channels. This is the heart of the product and ships early. After this phase, given a buyer + product + quantity + currency, the engine deterministically returns the correct unit price with full provenance, and the result is cacheable and identical regardless of channel (storefront, REST, GraphQL, sales rep).

### Tasks

#### 3.1 — Pricing schema

**What**: `customer_groups`, `customer_group_members`, `price_lists`, `price_list_assignments`, `prices` per Suggestion 3.

**Design**:
```sql
CREATE TABLE price_lists (
    price_list_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL, name TEXT NOT NULL,
    currency_code CHAR(3) NOT NULL,            -- ISO 4217
    priority INTEGER NOT NULL DEFAULT 0,       -- higher wins
    valid_from TIMESTAMPTZ, valid_to TIMESTAMPTZ,
    rules JSONB NOT NULL DEFAULT '{}',         -- global_discount_pct, rounding, exclude_categories
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE prices (
    price_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    price_list_id UUID NOT NULL REFERENCES price_lists(price_list_id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(product_id),
    variant_id UUID REFERENCES product_variants(variant_id),
    min_quantity INTEGER NOT NULL DEFAULT 1,   -- quantity break threshold
    price NUMERIC(12,4) NOT NULL,
    overrides JSONB NOT NULL DEFAULT '{}',     -- discount_pct, valid_from/to, note
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prices_lookup ON prices (price_list_id, product_id, min_quantity);
-- price_list_assignments(target_type: company|customer_group|channel)
```

**Testing**:
- `Integration: insert quantity-break rows (min_qty 1/10/100) for one product/list`.
- `Integration: assign list to customer_group and to company; both rows present`.

#### 3.2 — Price resolution engine (domain, pure)

**What**: Deterministic algorithm resolving the effective unit price for `(buyer, product/variant, quantity, currency, at)`.

**Design**:
Algorithm:
1. Gather candidate price lists for the buyer's company in priority order: direct **contract** company assignments first, then **customer_group** assignments, then **channel**, then tenant default. (Company-direct beats group; higher `priority` beats lower; ties broken by most-recent `valid_from`.)
2. Filter to lists active at `at` (valid_from/to) and matching `currency`.
3. For each list, select the `prices` row for the product/variant with the **largest `min_quantity ≤ requested_qty`** (quantity-break selection).
4. Apply per-row `overrides.discount_pct`, then list-level `rules.global_discount_pct`, then `rules.rounding`.
5. First list (by the priority ordering) that yields a price wins. Return:
```ts
interface PriceResolution {
  unitPrice: Money;            // integer minor units + currency
  priceListId: string;
  source: 'contract' | 'group' | 'channel' | 'default';
  appliedQuantityBreak: number;
  discountsApplied: string[];  // provenance for audit ("qty_break_10pct","list_global_5pct")
  resolvedAt: string;          // ISO 8601
}
```
- Implemented as one Drizzle SQL query using a CTE + `DISTINCT ON` for quantity-break selection, returning ordered candidates; tie-break and discount math done in TS for testability.
- No price found → `PriceResolution | null`; caller decides (hide product / block order).

**Testing**:
- `Unit: qty 5 with breaks at 1/10/100 → selects min_qty 1 row`.
- `Unit: qty 50 → selects min_qty 10 row`.
- `Unit: contract list priority 20 beats group list priority 10 even if group price lower`.
- `Unit: expired list skipped; list in wrong currency skipped`.
- `Unit: overrides.discount_pct=10 then list global 5% applied in order; rounding nearest_0.05 honoured`.
- `Unit: no applicable price → returns null`.
- `Property test: resolution is deterministic and channel-independent for identical inputs`.

#### 3.3 — Effective-price cache + pricing API

**What**: Cache resolutions in Redis and expose `POST /v1/pricing/resolve` (batch) + GraphQL `effectivePrice`.

**Design**:
- Cache key `price:{tenant}:{company}:{product}:{variant}:{qty}:{currency}`, TTL 600s; invalidated on price/list/assignment change (publish to a `pricing.invalidate` channel; worker clears keys by tenant prefix).
- `POST /v1/pricing/resolve` body: `{ items: [{productId, variantId?, quantity, currency}] }` → array of `PriceResolution`. Used by carts and storefront product grids (one round trip).

**Testing**:
- `Integration: resolve then mutate price → cache invalidated → new resolution reflects change`.
- `Integration (HTTP): batch resolve 50 items returns 50 resolutions, all from buyer's correct lists`.
- `Load (optional): cached resolve p99 < 10ms`.

#### 3.4 — Audit events table (price/quote/order provenance)

**What**: Append-only `audit_events` capturing every price change, approval decision, and order/quote transition (the pragmatic slice of event-sourcing from Suggestion 2).

**Design**:
```sql
CREATE TABLE audit_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    aggregate_type TEXT NOT NULL,    -- 'PriceList'|'Order'|'Quote'|'Company'
    aggregate_id UUID NOT NULL,
    event_type TEXT NOT NULL,        -- 'PriceSet'|'OrderApproved'|...
    actor_type TEXT NOT NULL, actor_id UUID,
    data JSONB NOT NULL,
    correlation_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_agg ON audit_events (tenant_id, aggregate_type, aggregate_id, created_at);
```
- Writes go through `recordEvent(tx, ...)` inside the same transaction as the state change (transactional outbox pattern; Phase 6 dispatches to webhooks).

**Testing**:
- `Integration: updating a price writes a PriceSet audit event in the same tx (rollback drops both)`.
- `Integration: query timeline for a PriceList returns events in created_at order`.

---

## Phase 4: Carts, Order Rules & Order Lifecycle

### Purpose
Turn resolved prices into orders while enforcing B2B-specific rules: order minimums, case-pack/multiple-of quantities, minimum line quantities, credit-limit checks, and payment terms. Implement the order state machine. After this phase, a buyer can build a cart, have rules enforced, and submit an order that transitions through its lifecycle with full audit history.

### Tasks

#### 4.1 — Order & cart schema

**What**: `orders`, `order_lines` per Suggestion 3 (with `source`, `source_metadata`, `history`, `shipping_address`/`billing_address` JSONB). Carts are draft orders (`status='draft'`).

**Design**: Tables exactly per Suggestion 3 §Orders. `orders.status` enum: `draft|pending_approval|approved|submitted|processing|shipped|delivered|cancelled|rejected`. `source` enum: `storefront|sales_rep|ai_agent|po_ingestion|edi_850|api`. Denormalize `order_lines.product_name`/`sku` for historical accuracy.

**Testing**:
- `Integration: create draft order, add lines, totals recomputed`.
- `Integration: cascade delete order removes lines`.

#### 4.2 — Order rules engine (domain, pure)

**What**: Validate a cart against B2B constraints before submission.

**Design**:
```ts
interface OrderRuleContext {
  lines: { productId: string; quantity: number; casePackQty: number; minOrderQty: number; unitPrice: Money }[];
  subtotal: Money;
  company: { creditLimit?: Money; creditAvailable?: Money; paymentTerms: string };
  catalogueMinimums?: { minOrderValue?: Money };
}
type RuleViolation =
  | { code: 'CASE_PACK_MULTIPLE'; productId: string; required: number; got: number }
  | { code: 'MIN_LINE_QTY'; productId: string; required: number; got: number }
  | { code: 'MIN_ORDER_VALUE'; required: Money; got: Money }
  | { code: 'CREDIT_EXCEEDED'; available: Money; required: Money };
function validateOrder(ctx: OrderRuleContext): RuleViolation[];
```
- Pure function; returns all violations (not fail-fast) so the UI shows every issue at once.
- Credit check uses live `creditAvailable` = `creditLimit − outstanding` (outstanding from a credit-usage view built in 4.4).

**Testing**:
- `Unit: qty 7 with case_pack 4 → CASE_PACK_MULTIPLE`.
- `Unit: line qty 1 with min_order_qty 5 → MIN_LINE_QTY`.
- `Unit: subtotal below catalogue min_order_value → MIN_ORDER_VALUE`.
- `Unit: order total exceeds credit available → CREDIT_EXCEEDED`.
- `Unit: compliant cart → [] (no violations)`.

#### 4.3 — Order lifecycle service + state machine

**What**: Service orchestrating cart→order transitions, re-resolving prices at submit, writing audit events, and emitting domain events.

**Design**:
State machine (allowed transitions):
```
draft → submitted            (rules pass, no approval needed)
draft → pending_approval     (approval workflow triggered — Phase 8)
pending_approval → approved | rejected
approved/submitted → processing → shipped → delivered
any(pre-shipped) → cancelled
```
- `submitOrder(orderId)`: lock order row, re-run price resolution per line (reject if any price drifted beyond a tolerance unless `acceptRepricing`), run `validateOrder`, set `submitted_at`, append to `history`, write `OrderSubmitted` audit event, emit CloudEvent `com.b2b.order.submitted`.
- `order_number` generated per tenant via a sequence (`ORD-{tenant_seq}`).
- Idempotency: `submitOrder` accepts an `Idempotency-Key` header; repeated key returns the prior result.

**Testing**:
- `Unit: illegal transition shipped→draft → throws InvalidTransition`.
- `Integration: submit compliant cart → status submitted, history appended, audit event + CloudEvent emitted`.
- `Integration: submit with stale price → 409 PRICE_CHANGED unless acceptRepricing`.
- `Integration: duplicate Idempotency-Key → same order, no double submit`.

#### 4.4 — Credit usage view + quick reorder / saved lists

**What**: `saved_lists` + `saved_list_items`, a reorder endpoint, and a credit-usage SQL view.

**Design**:
- `saved_lists`/`saved_list_items` per Suggestion 1 (with `is_shared`).
- `credit_usage` view: `SUM(total)` of orders in unpaid statuses per company → `outstanding`; `available = credit_limit − outstanding`.
- `POST /v1/saved-lists/{id}/reorder` → creates a draft order from list items, prices resolved live, returns cart with any rule violations.

**Testing**:
- `Integration: save list of 5 items, reorder → draft order with 5 priced lines`.
- `Integration: shared list visible to other buyers in same company, not other companies`.
- `Integration: credit_usage reflects two unpaid orders; paying one reduces outstanding`.

---

## Phase 5: API Surface, Reference Storefront & Sales-Rep Mode

### Purpose
Harden the public contract (OpenAPI 3.1 + GraphQL Storefront), prove the headless promise with a thin Next.js reference storefront, and implement sales-rep "order on behalf of" — a table-stakes B2B feature. After this phase the platform is demonstrably usable end-to-end by both buyers and reps through real interfaces.

### Tasks

#### 5.1 — OpenAPI 3.1 generation + typed client

**What**: Auto-generate the OpenAPI 3.1 document from Fastify route schemas (derived from Zod) and publish a typed TS client.

**Design**:
- `@fastify/swagger` + `fastify-type-provider-zod` so every route's Zod schema becomes OpenAPI. Served at `/v1/openapi.json` and `/docs`.
- `tools/openapi` exports the spec at build time and runs `openapi-typescript` to emit `packages/contracts/src/openapi-client.ts`.
- Error model standardised: RFC 9457 Problem Details (`type`, `title`, `status`, `detail`, `instance`).

**Testing**:
- `Integration: GET /v1/openapi.json validates against OpenAPI 3.1 meta-schema`.
- `Unit: generated client types compile against a sample request/response`.
- `Integration: 404/409/422 responses conform to Problem Details shape`.

#### 5.2 — GraphQL Storefront API

**What**: Pothos schema exposing read-optimized buyer storefront operations.

**Design**:
- Queries: `viewer`(buyer+company), `catalogue`(visible products with effective prices via 3.3), `product`, `order`, `orders`, `savedLists`.
- Mutations: `addToCart`, `updateCartLine`, `submitOrder`, `createSavedList`.
- DataLoader batching for prices/products to avoid N+1; effective price resolved through the cached engine.

**Testing**:
- `Integration: catalogue query returns only visible products, each with buyer-specific price`.
- `Integration: N+1 guard — 20 products → 1 batched price resolve call`.
- `E2E: addToCart then submitOrder mutation chain succeeds`.

#### 5.3 — Sales-rep "order on behalf of" (impersonation)

**What**: Allow a `sales_rep` principal to act for a buyer within assigned accounts, with every action attributed to the rep.

**Design**:
- Header `X-On-Behalf-Of: {buyerId}`; plugin verifies rep `MANAGES` the buyer's company (assignment table `sales_rep_assignments(rep_id, company_id)`), sets `impersonatedBuyerId`, and sets `app.current_buyer` to the buyer while recording `actor_type='sales_rep'`, `actor_id=rep` in audit events.
- Pricing/entitlements resolve as the **buyer**; audit attribution stays the **rep**.

**Testing**:
- `Integration: rep with assignment places order for buyer → order.buyer_id = buyer, audit actor = rep`.
- `Integration: rep without assignment → 403`.
- `Integration: impersonated cart prices match the buyer's contract pricing, not the rep's`.

#### 5.4 — Next.js reference storefront (thin)

**What**: A minimal Next.js (App Router) storefront consuming the GraphQL Storefront API: login, catalogue grid with prices, cart, reorder, order history.

**Design**:
- Server Components fetch via the GraphQL client; OIDC login via the auth adapter; company switcher for buyers in multiple companies.
- WCAG 2.2 AA baseline (semantic landmarks, labelled controls) per standards.md.
- Explicitly thin — it exists to prove the API, not to be the product.

**Testing**:
- `E2E (Playwright): login → see catalogue with my prices → add to cart → submit → appears in order history`.
- `E2E: reorder from saved list → cart pre-filled`.
- `A11y (axe): catalogue and checkout pages have zero critical violations`.

---

## Phase 6: Events, Webhooks & Integration Adapter Framework

### Purpose
Make the platform composable: emit CloudEvents-formatted webhooks reliably (transactional outbox), and define the tax/ERP adapter interfaces with a working flat-rate tax provider and CSV ERP bridge. After this phase, external systems can react to commerce events and orders can sync outward.

### Tasks

#### 6.1 — Transactional outbox + CloudEvents

**What**: An outbox table written in-transaction with state changes, drained by a worker that delivers signed CloudEvents.

**Design**:
```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    event_type TEXT NOT NULL,           -- com.b2b.order.submitted, ...
    ce_id UUID NOT NULL, ce_source TEXT NOT NULL, ce_subject TEXT,
    data JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending|delivered|failed
    attempts INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- CloudEvents 1.0 envelope: `specversion, id, source, type, subject, time, datacontenttype, data`.
- BullMQ `webhook-delivery` job: load subscriptions for `event_type`, POST envelope with `X-B2B-Signature: sha256=...` (HMAC of body), exponential backoff (max 8 attempts), record delivery attempts.
- `webhook_subscriptions(tenant_id, url, event_types[], secret, active)`.

**Testing**:
- `Integration: order submit writes outbox row in same tx`.
- `Integration (mocked HTTP): worker delivers signed CloudEvent; receiver verifies HMAC`.
- `Integration: receiver returns 500 thrice then 200 → retried with backoff, marked delivered`.
- `Unit: CloudEvent envelope conforms to spec required attributes`.

#### 6.2 — Tax adapter interface + flat-rate provider

**What**: `TaxProvider` interface and a default flat-rate/destination-table implementation; Avalara/Vertex stubs.

**Design**:
```ts
interface TaxProvider {
  quote(input: { lines: TaxLine[]; shipTo: Address; tenantId: string }): Promise<TaxQuote>;
  commit(input: { orderId: string; quote: TaxQuote }): Promise<void>;
}
```
- Selected per tenant via `tenants.settings.tax_provider`. Tax computed at submit and stored in `orders.tax_total`.

**Testing**:
- `Unit: flat-rate provider applies tenant rate by destination state`.
- `Integration: order submit populates tax_total via configured provider`.
- `Integration (mocked): Avalara stub called with correct address payload`.

#### 6.3 — ERP adapter interface + CSV bridge

**What**: `ErpAdapter` interface and a generic CSV bridge (import products/companies, export orders); `integration_events` log per Suggestion 3.

**Design**:
```ts
interface ErpAdapter {
  exportOrder(order: OrderDTO): Promise<{ externalId: string }>;
  importProducts(stream: Readable): Promise<ImportResult>;
}
```
- CSV bridge: configurable column mapping in `tenants.settings.erp.csv_mapping`. Order export drops NDJSON/CSV to a configured location; product import reuses the bulk-import job (2.2). Every call logged to `integration_events` (request/response/error JSONB, attempts) for replay.

**Testing**:
- `Integration: import 100-row product CSV → 100 products, mapping applied, errors row-reported`.
- `Integration: order submit → CSV export file produced with correct columns`.
- `Integration: failed export logged to integration_events with status=failed and retryable`.

---

## Phase 7: AI-Native Capabilities

### Purpose
Deliver the AI-native advantage that no incumbent treats as first-class: PO ingestion from PDF/email, natural-language ordering, reorder prediction, order anomaly detection, AI quote drafts, and SKU auto-classification. After this phase, AI tasks run asynchronously, write structured results with confidence scores, and route low-confidence outputs to human review.

### Tasks

#### 7.1 — AI task framework

**What**: `ai_tasks` table (Suggestion 3) + BullMQ processors + a provider-abstracted LLM client with structured output and prompt caching.

**Design**:
```sql
CREATE TABLE ai_tasks (
  task_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  task_type TEXT NOT NULL,   -- po_ingestion|nl_order|reorder|price_anomaly|quote_draft|sku_classification
  status TEXT NOT NULL DEFAULT 'pending',
  input JSONB NOT NULL, output JSONB, confidence NUMERIC(3,2),
  human_review BOOLEAN NOT NULL DEFAULT false, reviewed_by UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), completed_at TIMESTAMPTZ
);
```
- `packages/ai`: `runStructured<T>(schema: ZodSchema<T>, prompt, opts)` wrapping the Vercel AI SDK `generateObject`. Anthropic default; prompt caching on stable system/context blocks.
- Tasks below a per-type confidence threshold set `human_review=true` and surface in a review queue; never auto-commit money-affecting actions without confirmation.

**Testing**:
- `Unit: runStructured returns Zod-validated object; invalid model output → retry then fail task`.
- `Integration: task lifecycle pending→completed with output+confidence persisted`.
- `Integration: low-confidence result flags human_review`.

#### 7.2 — PO ingestion (PDF/email → draft order)

**What**: Parse inbound PDF/email purchase orders into structured draft orders with SKU matching.

**Design**:
- Inbound email captured (MailHog in dev; IMAP/SES inbound in prod) → enqueue `po_ingestion` with file ref.
- Pipeline: extract text/tables (pdf parse + vision model for scanned PDFs) → `generateObject` against `POExtraction` Zod schema `{ poNumber, lines: [{ description, sku?, gtin?, quantity, unitPrice? }], buyerHint }` → match each line to a product (exact SKU/GTIN, else pgvector similarity on description) → create draft order (`source='po_ingestion'`, `source_metadata` with original file + confidence + per-line match scores). Unmatched lines flagged for review.

**Testing**:
- `Fixture: sample PO PDF → draft order with correct line count and quantities`.
- `Integration: line with exact GTIN matches product; ambiguous description → review flag`.
- `Integration: created draft has source=po_ingestion and confidence in source_metadata`.
- `Integration (mocked LLM): deterministic fixture output → stable parse`.

#### 7.3 — Natural-language ordering

**What**: Turn an utterance ("send the usual order plus 20% on SKUs trending up") into a proposed draft order.

**Design**:
- Tool-calling agent with tools: `getOrderHistory(buyer)`, `getSavedLists(buyer)`, `getTrendingSkus(buyer)`, `resolvePrice(...)`, `proposeDraftOrder(lines)`. Bound to the buyer's entitlements/pricing so it cannot exceed visibility.
- Output is always a **proposed** draft requiring buyer confirmation; never auto-submits.

**Testing**:
- `Integration (mocked LLM+tools): "the usual" resolves to most-frequent reorder list`.
- `Integration: proposed order only contains products visible to the buyer`.
- `Integration: "+20% on trending" increases qty on flagged SKUs only`.

#### 7.4 — Reorder prediction, anomaly detection, quote drafts, SKU classification

**What**: Four scheduled/triggered AI features sharing the framework.

**Design**:
- **Reorder prediction**: scheduled job analyses order cadence per company/SKU → `reorder` task output `{ recommendations: [{sku, qty, reason}] }`; surfaced as suggested saved-list updates.
- **Anomaly detection**: on order submit, compare against the company's historical distribution (qty, value, cadence) → flag `{ anomalyType, score }`; high scores notify rep, don't block.
- **Quote drafts** (Phase 8 dependency): generate `QuoteResponseDraft` lines from list prices + negotiation context; flagged `ai_assisted=true`.
- **SKU classification**: on product create without category, embed description and assign nearest category + `ai_metadata.auto_category` with confidence.

**Testing**:
- `Integration: company ordering SKU every 14d → reorder recommendation near day 14`.
- `Integration: order 10x normal value → anomaly flagged, order not blocked`.
- `Integration: new product without category → auto_category set with confidence`.
- `Integration: quote draft lines marked ai_assisted`.

---

## Phase 8: Approvals & Quote/RFQ Workflows

### Purpose
Add the procurement workflows that distinguish real B2B from retrofitted B2C: configurable multi-step approval chains above thresholds, and a quote/RFQ negotiation flow with full counter-offer history. After this phase, orders can route through approvals and buyers/reps can negotiate quotes that convert to orders.

### Tasks

#### 8.1 — Approval workflows (JSONB-configured)

**What**: `approval_workflows` + `approval_requests` per Suggestion 3, evaluated during order submit.

**Design**:
- `approval_workflows.definition` JSONB: `{ trigger: {type:'order_total',operator:'gte',value:5000}, steps:[{step:1,role:'department_approver',timeout_hours:24,escalation:{to:'company_admin',after_hours:48}},{step:2,role:'finance_approver',required_for:{total_gte:25000}}], on_reject:'return_to_buyer', on_timeout:'auto_escalate' }`.
- On submit, the approvals domain service evaluates triggers; if matched, order → `pending_approval`, `approval_requests` row created at `current_step=1`, approvers notified (webhook/email). `approve`/`reject` advance/branch per definition, appending to `step_history`. Timeouts handled by a scheduled worker (escalation/auto-actions).

**Testing**:
- `Unit: trigger order_total gte 5000 → requires approval; 4999 → no approval`.
- `Integration: $30k order requires both steps; step 1 approve advances to step 2`.
- `Integration: reject at step 1 → on_reject return_to_buyer (status draft)`.
- `Integration: timeout escalates to company_admin after configured hours`.

#### 8.2 — Quote / RFQ negotiation

**What**: `quotes` + `quote_lines` per Suggestion 3 with `negotiations` JSONB history and a counter-offer state machine.

**Design**:
- States: `draft|submitted|counter_offer|accepted|rejected|expired|converted`.
- Buyer/rep/ai_agent append rounds to `negotiations` (each: round, actor, lines snapshot, comment, ai_assisted?). Accept converts to a draft order with the negotiated prices (`converted_order_id`), preserving negotiated unit prices over standard resolution.
- `valid_until` expiry handled by scheduled worker → status `expired`.

**Testing**:
- `Integration: submit RFQ → rep counter (AI-assisted draft from 7.4) → buyer accept → order created with negotiated prices`.
- `Integration: negotiations array records each round in order with snapshots`.
- `Integration: expired quote cannot be accepted → 409`.
- `Integration: accepting overrides standard price resolution for those lines`.

---

## Phase 9: Procurement Integration — Punchout & EDI (Backlog Differentiator)

### Purpose
Implement the procurement-side integration incumbents leave to third parties: cXML/OCI punchout and EDI 850/810/856. This is the backlog differentiator from features.md that unlocks enterprise accounts. After this phase, enterprise buyers can punch out from Ariba/Coupa and exchange EDI documents.

### Tasks

#### 9.1 — Punchout (cXML 1.2 + OCI)

**What**: Supplier-hosted punchout: accept `PunchOutSetupRequest`, return a session start URL, and post back a `PunchOutOrderMessage` cart.

**Design**:
- `POST /punchout/cxml` parses cXML `PunchOutSetupRequest` (shared-secret credential auth per trading partner in `trading_partners`), creates a punchout session bound to a company/buyer mapping, returns `StartPage` URL into the storefront in punchout mode.
- On checkout, emit `PunchOutOrderMessage` cXML back to the buyer's `BrowserFormPost` URL with line items priced via the engine.
- OCI variant: HTML-form field mapping (`NEW_ITEM-*`) for SAP SRM/Coupa.

**Testing**:
- `Fixture: valid cXML PunchOutSetupRequest → session created, StartPage returned`.
- `Integration: bad shared secret → 401 cXML fault`.
- `Integration: punchout cart checkout → well-formed PunchOutOrderMessage with engine prices`.
- `Unit: OCI field mapping produces NEW_ITEM-* fields`.

#### 9.2 — EDI 850/855/856/810

**What**: Ingest EDI 850 (PO) into orders; emit 855 (PO ack), 856 (ASN), 810 (invoice).

**Design**:
- `packages/integrations/edi`: X12 parser/generator (envelope ISA/GS/ST). 850 → draft/submitted order (`source='edi_850'`, ISA/GS controls in `source_metadata`); SKU match via GTIN/partner item IDs from `trading_partner_items`.
- Outbound: order events (Phase 6) trigger 855 on acceptance, 856 on ship, 810 on invoice. GS1 identifiers (GTIN/GLN) used for item/party identification.
- Transport: SFTP/AS2 adapter behind an interface (VAN-agnostic).

**Testing**:
- `Fixture: sample 850 → order with correct lines and PO number`.
- `Integration: order ship event → 856 ASN generated, valid X12 envelope`.
- `Integration: invalid X12 segment → parse error logged to integration_events`.
- `Unit: ISA/GS control numbers round-trip correctly`.

---

## Phase 10: MCP Server, Hardening & Production Readiness

### Purpose
Expose the platform to AI agents via MCP (the forward-looking agentic-commerce differentiator) and complete production hardening: rate limiting, security headers, observability, partitioning, and deployment artefacts. After this phase the platform is deployable, secure, observable, and agent-ready.

### Tasks

#### 10.1 — MCP server (agentic commerce)

**What**: An MCP server exposing catalogue/pricing/order tools to AI buyer agents, scoped to an authenticated principal.

**Design**:
- Tools: `search_catalogue(query)`, `get_price(productId, qty)`, `create_draft_order(lines)`, `get_order_status(orderId)`, `list_saved_lists()`. Each tool resolves through the same domain services (entitlements + pricing) so an agent can never exceed the buyer's visibility or pricing.
- Auth: MCP transport carries an OAuth token → `RequestPrincipal` with `subject_type='ai_agent'`; all agent actions audited as such; order creation is draft-only (human/policy confirmation to submit).

**Testing**:
- `Integration: search_catalogue returns only entitled products`.
- `Integration: get_price returns buyer's contract price`.
- `Integration: create_draft_order produces draft with source=ai_agent, audited as ai_agent`.
- `Integration: agent cannot submit order directly (draft-only enforced)`.

#### 10.2 — Security, rate limiting & API hardening

**What**: Apply OWASP API Security Top 10 / ASVS baseline.

**Design**:
- `@fastify/rate-limit` per tenant + per principal; `@fastify/helmet` headers; request body size limits; strict CORS for storefront origin.
- Object-level authorization checks (BOLA) enforced by RLS + Casbin on every resource route. Input validation everywhere via Zod. Secrets via env/secret manager only. Audit log immutability (append-only, no UPDATE/DELETE grant on `audit_events`).

**Testing**:
- `Integration: exceeding rate limit → 429 with Retry-After`.
- `Integration (BOLA): buyer requesting another company's order → 404/403, never leaked`.
- `Integration: oversized payload → 413`.
- `Security: dependency audit (pnpm audit) gate in CI`.

#### 10.3 — Observability, partitioning & deployment

**What**: OpenTelemetry tracing, structured logs, DB partitioning for scale, and production Docker/CI.

**Design**:
- OTel spans on price resolution, order submit, AI tasks, webhook delivery; pino structured logs with `tenant_id`/`correlation_id`. Metrics: price-resolve p99, queue depth/lag, webhook delivery success rate.
- Partition `orders`/`order_lines` by `created_at` range, `audit_events` and `integration_events` likewise; BRIN indexes on timestamps; `pg_partman` for partition management.
- `Dockerfile.api`/`Dockerfile.worker` multi-stage builds; GitHub Actions CI: lint → typecheck → unit → integration (Testcontainers) → build → OpenAPI export. Health (`/healthz`) and readiness (`/readyz`) probes.

**Testing**:
- `Integration: trace spans emitted for an order-submit request`.
- `Integration: orders partition created for current period; insert lands in correct partition`.
- `Smoke: docker compose up → /healthz 200, /readyz 200 after migrations`.
- `CI: full pipeline green on a sample PR`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (tenancy, RLS, auth)        ─── required by everything
    │
Phase 2: Catalogue & visibility                 ─── requires Phase 1
    │
Phase 3: Pricing engine (CORE differentiator)   ─── requires Phase 2
    │
Phase 4: Carts, order rules, lifecycle          ─── requires Phase 3
    │
    ├── Phase 5: API surface, storefront, rep    ─── requires Phase 4
    ├── Phase 6: Events, webhooks, adapters      ─── requires Phase 4 (parallel with 5)
    │
Phase 7: AI capabilities                        ─── requires Phase 4 + 6 (uses pgvector/2, ai_tasks)
    │                                                (7.4 quote drafts depend on Phase 8)
Phase 8: Approvals & quotes                      ─── requires Phase 4 (parallel with 7; 7.4 ↔ 8.2)
    │
Phase 9: Punchout & EDI                          ─── requires Phase 4 + 6 (can parallel with 7/8)
    │
Phase 10: MCP, hardening, production             ─── requires all prior (10.1 needs 2/3/4; 10.2/10.3 cross-cutting)
```

**Parallelism opportunities:**
- Phases **5 and 6** can be developed concurrently once Phase 4 is complete.
- Phases **7, 8, and 9** can largely proceed in parallel after Phases 4 and 6, with a soft coupling between 7.4 (AI quote drafts) and 8.2 (quote workflow) — build 8.2's data model first, then 7.4 layers on.
- Within Phase 10, **10.2 (security)** and **10.3 (observability/deploy)** are cross-cutting and can begin earlier as continuous hardening rather than a final gate.

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented and merged.
2. All unit and integration (Testcontainers) tests pass locally and in CI.
3. ESLint + Prettier pass with zero warnings; `tsc --noEmit` is clean in strict mode.
4. Docker images build; `docker compose up` brings the stack to a healthy state.
5. The phase's primary capability works end-to-end (demonstrated by an integration or E2E test).
6. New configuration keys are added to `.env.example` and documented.
7. New REST routes appear in the generated `/v1/openapi.json` (3.1) and new GraphQL fields in the schema SDL.
8. Database migrations (drizzle + raw SQL for RLS/indexes/partitions) are created, idempotent, and reversible where feasible; `db:migrate` runs clean on an empty database.
9. RLS isolation is verified for any new tenant-scoped table (cross-tenant read returns zero rows).
10. Money is handled as integer minor units / `NUMERIC` — no floats touch prices, taxes, or totals.
11. AI-produced, money-affecting actions are never auto-committed without human/policy confirmation (draft-only).
12. New domain events have a defined CloudEvents `type` and are emitted via the transactional outbox.
```
