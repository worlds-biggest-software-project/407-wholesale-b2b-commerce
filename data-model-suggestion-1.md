# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Wholesale & B2B Commerce (407) | Generated: 2026-05-25

## Summary

A fully normalized relational schema in PostgreSQL, following classic 3NF/BCNF normalization for all core entities. This approach treats the B2B commerce domain as a set of well-defined, strongly typed tables connected by foreign keys, with multi-tenancy enforced via Row-Level Security (RLS) policies and a shared-schema, shared-database architecture. It is the most conventional and well-understood approach, aligning closely with the data models used by enterprise platforms like OroCommerce and Adobe Commerce.

---

## Key Entities and Relationships

### Organization & Account Hierarchy

```sql
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE companies (
    company_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    parent_id       UUID REFERENCES companies(company_id),
    name            TEXT NOT NULL,
    tax_id          TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    credit_limit    NUMERIC(12,2),
    payment_terms   TEXT,  -- e.g. 'net_30', 'net_60'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE company_addresses (
    address_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    address_type    TEXT NOT NULL,  -- 'billing', 'shipping'
    line1           TEXT NOT NULL,
    line2           TEXT,
    city            TEXT NOT NULL,
    state           TEXT,
    postal_code     TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,  -- ISO 3166-1
    is_default      BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE buyers (
    buyer_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    email           TEXT NOT NULL,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'buyer',  -- 'buyer', 'approver', 'admin'
    spending_limit  NUMERIC(12,2),
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE buyer_roles (
    buyer_id        UUID NOT NULL REFERENCES buyers(buyer_id),
    permission      TEXT NOT NULL,  -- 'place_order', 'approve_order', 'manage_users', etc.
    PRIMARY KEY (buyer_id, permission)
);
```

### Product Catalogue

```sql
CREATE TABLE categories (
    category_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    parent_id       UUID REFERENCES categories(category_id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE products (
    product_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    sku             TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',  -- EA, CS, PK, etc.
    case_pack_qty   INTEGER NOT NULL DEFAULT 1,
    gtin            TEXT,  -- GS1 GTIN
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE TABLE product_variants (
    variant_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    sku             TEXT NOT NULL,
    name            TEXT NOT NULL,
    weight          NUMERIC(10,3),
    dimensions      TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE product_categories (
    product_id      UUID NOT NULL REFERENCES products(product_id),
    category_id     UUID NOT NULL REFERENCES categories(category_id),
    PRIMARY KEY (product_id, category_id)
);
```

### Custom Catalogues & Visibility

```sql
CREATE TABLE catalogues (
    catalogue_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            TEXT NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE catalogue_products (
    catalogue_id    UUID NOT NULL REFERENCES catalogues(catalogue_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (catalogue_id, product_id)
);

CREATE TABLE catalogue_assignments (
    catalogue_id    UUID NOT NULL REFERENCES catalogues(catalogue_id),
    assignee_type   TEXT NOT NULL,  -- 'company', 'customer_group', 'region'
    assignee_id     UUID NOT NULL,
    PRIMARY KEY (catalogue_id, assignee_type, assignee_id)
);
```

### Price Lists & Contract Pricing

```sql
CREATE TABLE customer_groups (
    group_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            TEXT NOT NULL,
    description     TEXT
);

CREATE TABLE customer_group_members (
    group_id        UUID NOT NULL REFERENCES customer_groups(group_id),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    PRIMARY KEY (group_id, company_id)
);

CREATE TABLE price_lists (
    price_list_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            TEXT NOT NULL,
    currency_code   CHAR(3) NOT NULL,  -- ISO 4217
    priority        INTEGER NOT NULL DEFAULT 0,
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE price_list_assignments (
    price_list_id   UUID NOT NULL REFERENCES price_lists(price_list_id),
    assignee_type   TEXT NOT NULL,  -- 'company', 'customer_group'
    assignee_id     UUID NOT NULL,
    PRIMARY KEY (price_list_id, assignee_type, assignee_id)
);

CREATE TABLE prices (
    price_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    price_list_id   UUID NOT NULL REFERENCES price_lists(price_list_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    min_quantity    INTEGER NOT NULL DEFAULT 1,
    price           NUMERIC(12,4) NOT NULL,
    discount_pct    NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prices_lookup ON prices (price_list_id, product_id, min_quantity);
```

### Orders, Approvals & Quotes

```sql
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    buyer_id        UUID NOT NULL REFERENCES buyers(buyer_id),
    order_number    TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- 'draft','pending_approval','approved','submitted','processing',
    -- 'shipped','delivered','cancelled'
    po_number       TEXT,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total           NUMERIC(12,2) NOT NULL DEFAULT 0,
    payment_terms   TEXT,
    notes           TEXT,
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_lines (
    line_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    sku             TEXT NOT NULL,
    quantity        INTEGER NOT NULL,
    unit_price      NUMERIC(12,4) NOT NULL,
    line_total      NUMERIC(12,2) NOT NULL,
    price_list_id   UUID REFERENCES price_lists(price_list_id),
    notes           TEXT
);

CREATE TABLE approval_workflows (
    workflow_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            TEXT NOT NULL,
    trigger_type    TEXT NOT NULL,  -- 'order_total', 'line_count', 'product_category'
    threshold_value NUMERIC(12,2),
    status          TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE approval_steps (
    step_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(workflow_id),
    step_order      INTEGER NOT NULL,
    approver_role   TEXT NOT NULL,
    required_count  INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE approval_requests (
    request_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    step_id         UUID NOT NULL REFERENCES approval_steps(step_id),
    approver_id     UUID NOT NULL REFERENCES buyers(buyer_id),
    status          TEXT NOT NULL DEFAULT 'pending',  -- 'pending','approved','rejected'
    decided_at      TIMESTAMPTZ,
    comment         TEXT
);

CREATE TABLE quotes (
    quote_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    buyer_id        UUID REFERENCES buyers(buyer_id),
    sales_rep_id    UUID,
    quote_number    TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- 'draft','submitted','counter_offer','accepted','rejected','expired','converted'
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2),
    valid_until     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quote_lines (
    line_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quotes(quote_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    quantity        INTEGER NOT NULL,
    list_price      NUMERIC(12,4) NOT NULL,
    offered_price   NUMERIC(12,4) NOT NULL,
    notes           TEXT
);

CREATE TABLE quote_negotiations (
    negotiation_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quotes(quote_id),
    actor_type      TEXT NOT NULL,  -- 'buyer', 'sales_rep', 'ai_agent'
    action          TEXT NOT NULL,  -- 'submit', 'counter', 'accept', 'reject'
    snapshot        JSONB NOT NULL, -- snapshot of line items at this point
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Saved Lists & Reorder

```sql
CREATE TABLE saved_lists (
    list_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    buyer_id        UUID NOT NULL REFERENCES buyers(buyer_id),
    name            TEXT NOT NULL,
    is_shared       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE saved_list_items (
    list_id         UUID NOT NULL REFERENCES saved_lists(list_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    default_qty     INTEGER NOT NULL DEFAULT 1,
    PRIMARY KEY (list_id, product_id, variant_id)
);
```

### Multi-Tenancy via RLS

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON companies
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Repeat for: products, orders, price_lists, catalogues, categories,
-- customer_groups, approval_workflows, quotes, etc.
```

---

## Pros

- **Strong consistency**: ACID guarantees across all transactional operations (pricing, orders, approvals, credit). No eventual-consistency surprises during checkout or approval flows.
- **Referential integrity**: Foreign keys enforce that orders always reference valid products, buyers, and companies. Orphaned records are structurally impossible.
- **Mature tooling**: Excellent support across ORMs (Prisma, Drizzle, TypeORM, SQLAlchemy, Diesel), migration tools (Flyway, Alembic, Prisma Migrate), and monitoring (pganalyze, Datadog).
- **RLS-based multi-tenancy**: PostgreSQL Row-Level Security provides database-enforced tenant isolation without application-layer WHERE clauses, reducing the risk of cross-tenant data leakage.
- **Query flexibility**: Complex pricing resolution queries (find best price across multiple price lists for a customer group with quantity breaks) are natural in SQL with CTEs and window functions.
- **Well-understood by teams**: Relational modeling is the most common skill set; hiring and onboarding friction is minimal.
- **Proven at scale**: PostgreSQL handles multi-TB databases with proper indexing and partitioning (e.g., partition orders by year).

## Cons

- **Schema rigidity**: Adding new product attributes or pricing dimensions requires schema migrations and potentially downtime. Flexible or customer-specific attributes are awkward in strict 3NF.
- **Complex joins for reads**: Resolving a buyer's visible catalogue with applicable prices requires joining 6-8 tables (buyer -> company -> customer_group -> price_list_assignment -> price_list -> prices -> products -> catalogue_products). Read performance depends heavily on index tuning.
- **Approval workflow modeling**: Representing arbitrary workflow graphs in relational tables is verbose and requires careful recursive queries or adjacency-list patterns.
- **Audit trail bolted on**: Native relational tables only store current state. Audit logging requires either trigger-based audit tables or application-level history tracking, adding complexity.
- **Horizontal scaling limits**: While PostgreSQL scales vertically well, sharding a normalized schema with deep foreign-key relationships across nodes is complex (Citus can help but adds operational overhead).
- **EAV temptation**: Product attribute variability often pushes teams toward Entity-Attribute-Value patterns, which degrade query performance and complicate reporting.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| **Database** | PostgreSQL 16+ (with RLS, partitioning, BRIN indexes) |
| **Connection pooling** | PgBouncer or Supavisor |
| **Migrations** | Prisma Migrate, Flyway, or golang-migrate |
| **ORM / Query builder** | Prisma (TypeScript), Drizzle ORM, or SQLAlchemy (Python) |
| **Search** | PostgreSQL full-text search for basic needs; Meilisearch or Typesense for catalogue search |
| **Caching** | Redis for price-resolution caching and session state |
| **API layer** | GraphQL (Pothos/Nexus) for storefront; REST (OpenAPI 3.1) for integrations |
| **Hosting** | AWS RDS / Aurora PostgreSQL, Supabase, or Neon for serverless |

---

## Migration and Scaling Considerations

- **Partitioning**: Partition `orders` and `order_lines` by `created_at` (range partitioning) once order volume exceeds ~50M rows. Partition `prices` by `price_list_id` if price lists grow large.
- **Read replicas**: Route catalogue browsing and reporting queries to read replicas; keep writes on the primary for orders, approvals, and pricing changes.
- **Materialized views**: Pre-compute "effective price" views for high-traffic catalogue pages; refresh on price list change events.
- **Connection pooling**: Essential for multi-tenant workloads where many tenants share the same database; PgBouncer in transaction mode handles 10K+ concurrent connections.
- **Index strategy**: Composite indexes on `(tenant_id, sku)`, `(price_list_id, product_id, min_quantity)`, `(company_id, status, created_at)` are critical. Use BRIN indexes on timestamp columns for large partitioned tables.
- **Migration from monolith**: If starting here and later needing event sourcing for specific domains (e.g., order lifecycle), extract those aggregates incrementally. The relational core can coexist with an event store for selected bounded contexts.
- **Data archival**: Move completed orders older than N years to archive tables or cold storage (e.g., S3 + Parquet) to keep the hot dataset lean.
