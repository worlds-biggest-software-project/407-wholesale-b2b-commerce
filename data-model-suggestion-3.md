# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Wholesale & B2B Commerce (407) | Generated: 2026-05-25

## Summary

A pragmatic hybrid architecture that uses normalized relational tables for core transactional entities (orders, companies, pricing) while leveraging PostgreSQL JSONB columns for flexible, schema-variable data (product attributes, workflow configurations, integration payloads, AI metadata). This approach captures the referential integrity benefits of relational modeling where it matters most (financial transactions, pricing enforcement, approval chains) while avoiding the rigidity that makes pure 3NF painful for product catalogues, custom fields, and evolving B2B requirements. It treats PostgreSQL as both a relational database and a document store -- a pattern that has gained significant traction with PostgreSQL 16+ JSONB improvements and is the approach used by modern headless commerce platforms like Medusa and Saleor.

---

## Key Entities and Relationships

### Organization & Account Hierarchy (Relational Core)

```sql
CREATE TABLE companies (
    company_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    parent_id       UUID REFERENCES companies(company_id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    -- Structured financial fields (must be queryable, auditable)
    credit_limit    NUMERIC(12,2),
    payment_terms   TEXT NOT NULL DEFAULT 'net_30',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    tax_id          TEXT,
    -- Flexible fields for industry-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"industry": "food_service", "annual_revenue": "5M-10M",
    --        "preferred_carrier": "FedEx", "delivery_days": ["Tue","Thu"],
    --        "custom_fields": {"region_code": "NE", "sales_territory": "T4"}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_companies_tenant ON companies (tenant_id);
CREATE INDEX idx_companies_metadata ON companies USING GIN (metadata);

CREATE TABLE buyers (
    buyer_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    email           TEXT NOT NULL UNIQUE,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'buyer',
    spending_limit  NUMERIC(12,2),
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- e.g. ["place_order", "view_invoices", "manage_saved_lists"]
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"locale": "en-US", "notifications": {"order_updates": true}}
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Product Catalogue (Hybrid: Relational + JSONB Attributes)

```sql
CREATE TABLE products (
    product_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    sku             TEXT NOT NULL,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    -- Fixed, frequently-queried fields stay as columns
    product_type    TEXT NOT NULL DEFAULT 'simple',  -- 'simple', 'variable', 'bundle'
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    case_pack_qty   INTEGER NOT NULL DEFAULT 1,
    min_order_qty   INTEGER NOT NULL DEFAULT 1,
    weight_kg       NUMERIC(10,3),
    gtin            TEXT,
    brand           TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    -- Variable product attributes live in JSONB
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Examples by industry:
    -- Food: {"allergens": ["gluten","dairy"], "shelf_life_days": 90,
    --        "storage_temp": "refrigerated", "kosher": true, "organic": true}
    -- Fashion: {"color": "Navy", "size": "XL", "material": "100% Cotton",
    --           "season": "SS26", "style_number": "FW-3421"}
    -- Industrial: {"voltage": "220V", "certification": ["UL","CE"],
    --              "hazmat_class": "3", "msds_url": "https://..."}
    -- Searchable tags (extracted from attributes for filtering)
    tags            TEXT[] NOT NULL DEFAULT '{}',
    -- AI-generated metadata
    ai_metadata     JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"auto_category": "Beverages > Coffee", "embedding_id": "emb_...",
    --        "suggested_reorder_interval_days": 14}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_products_tenant ON products (tenant_id, status);
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_products_gtin ON products (gtin) WHERE gtin IS NOT NULL;

CREATE TABLE product_variants (
    variant_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    sku             TEXT NOT NULL,
    name            TEXT NOT NULL,
    -- Variant-specific overrides
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"color": "Red", "size": "M"} — overrides parent
    weight_kg       NUMERIC(10,3),
    gtin            TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE product_images (
    image_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    variant_id      UUID REFERENCES product_variants(variant_id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    alt_text        TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}'
    -- e.g. {"width": 1200, "height": 800, "format": "webp"}
);

CREATE TABLE categories (
    category_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    parent_id       UUID REFERENCES categories(category_id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}'
);

CREATE TABLE product_categories (
    product_id      UUID NOT NULL REFERENCES products(product_id),
    category_id     UUID NOT NULL REFERENCES categories(category_id),
    PRIMARY KEY (product_id, category_id)
);
```

### Custom Catalogues

```sql
CREATE TABLE catalogues (
    catalogue_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    -- Flexible rules for catalogue composition
    rules           JSONB NOT NULL DEFAULT '{}',
    -- Static: {"type": "static"} — manual product list
    -- Dynamic: {"type": "dynamic", "filters": {"brand": ["Acme","Globex"],
    --           "attributes.organic": true, "tags": {"contains": "seasonal"}}}
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
    target_type     TEXT NOT NULL,  -- 'company', 'customer_group', 'region'
    target_id       UUID NOT NULL,
    PRIMARY KEY (catalogue_id, target_type, target_id)
);
```

### Price Lists (Relational Core + JSONB Rules)

```sql
CREATE TABLE customer_groups (
    group_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    -- Dynamic membership rules (alternative to explicit member table)
    membership_rules JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"type": "manual"} or
    -- {"type": "dynamic", "criteria": {"metadata.industry": "food_service",
    --  "metadata.annual_revenue_min": 1000000}}
    metadata        JSONB NOT NULL DEFAULT '{}'
);

CREATE TABLE customer_group_members (
    group_id        UUID NOT NULL REFERENCES customer_groups(group_id),
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    PRIMARY KEY (group_id, company_id)
);

CREATE TABLE price_lists (
    price_list_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    -- Pricing rules that don't fit into individual price rows
    rules           JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"global_discount_pct": 5, "rounding": "nearest_0.05",
    --        "exclude_categories": ["clearance"], "apply_after_quantity_breaks": true}
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE price_list_assignments (
    price_list_id   UUID NOT NULL REFERENCES price_lists(price_list_id),
    target_type     TEXT NOT NULL,  -- 'company', 'customer_group', 'channel'
    target_id       UUID NOT NULL,
    PRIMARY KEY (price_list_id, target_type, target_id)
);

CREATE TABLE prices (
    price_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    price_list_id   UUID NOT NULL REFERENCES price_lists(price_list_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    min_quantity    INTEGER NOT NULL DEFAULT 1,
    price           NUMERIC(12,4) NOT NULL,
    -- Optional per-price overrides as JSONB
    overrides       JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"discount_pct": 10, "valid_from": "2026-01-01", "valid_to": "2026-12-31",
    --        "note": "Annual contract renewal"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prices_lookup ON prices (price_list_id, product_id, min_quantity);
```

### Orders (Relational Core + JSONB for Flexibility)

```sql
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    buyer_id        UUID NOT NULL REFERENCES buyers(buyer_id),
    order_number    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    po_number       TEXT,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    shipping_total  NUMERIC(12,2) NOT NULL DEFAULT 0,
    total           NUMERIC(12,2) NOT NULL DEFAULT 0,
    payment_terms   TEXT,
    -- Source and AI metadata
    source          TEXT NOT NULL DEFAULT 'storefront',
    -- 'storefront', 'sales_rep', 'ai_agent', 'po_ingestion', 'edi_850', 'api'
    source_metadata JSONB NOT NULL DEFAULT '{}',
    -- For PO ingestion: {"original_file": "po_12345.pdf", "confidence": 0.94,
    --                     "extracted_lines": 12, "manual_corrections": 1}
    -- For EDI: {"isa_control": "000012345", "gs_control": "1234",
    --           "trading_partner_id": "TP-0042"}
    -- Shipping and fulfillment
    shipping_address JSONB NOT NULL DEFAULT '{}',
    billing_address  JSONB NOT NULL DEFAULT '{}',
    -- Audit trail as JSONB array (lightweight alternative to full event sourcing)
    history         JSONB NOT NULL DEFAULT '[]',
    -- e.g. [{"at": "2026-05-20T10:30:00Z", "action": "created", "by": "buyer_123"},
    --        {"at": "2026-05-20T10:35:00Z", "action": "submitted", "by": "buyer_123"},
    --        {"at": "2026-05-20T11:00:00Z", "action": "approved", "by": "approver_456",
    --         "comment": "Looks good"}]
    notes           TEXT,
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, order_number)
);
CREATE INDEX idx_orders_company ON orders (company_id, status, created_at DESC);
CREATE INDEX idx_orders_tenant ON orders (tenant_id, status, created_at DESC);

CREATE TABLE order_lines (
    line_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    sku             TEXT NOT NULL,
    product_name    TEXT NOT NULL,  -- denormalized for historical accuracy
    quantity        INTEGER NOT NULL,
    unit_price      NUMERIC(12,4) NOT NULL,
    line_total      NUMERIC(12,2) NOT NULL,
    price_list_id   UUID REFERENCES price_lists(price_list_id),
    -- Line-level metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"original_po_line": 3, "buyer_note": "Ship separately",
    --        "price_source": "contract", "discount_applied": "qty_break_10pct"}
    sort_order      INTEGER NOT NULL DEFAULT 0
);
```

### Approval Workflows (JSONB-Configured)

```sql
CREATE TABLE approval_workflows (
    workflow_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Workflow definition as structured JSONB
    definition      JSONB NOT NULL,
    -- e.g. {
    --   "trigger": {"type": "order_total", "operator": "gte", "value": 5000},
    --   "steps": [
    --     {"step": 1, "role": "department_approver", "timeout_hours": 24,
    --      "escalation": {"to": "company_admin", "after_hours": 48}},
    --     {"step": 2, "role": "finance_approver", "required_for": {"total_gte": 25000}}
    --   ],
    --   "on_reject": "return_to_buyer",
    --   "on_timeout": "auto_escalate"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_requests (
    request_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(workflow_id),
    current_step    INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'pending',
    step_history    JSONB NOT NULL DEFAULT '[]',
    -- e.g. [{"step": 1, "approver_id": "...", "decision": "approved",
    --         "at": "2026-05-20T11:00:00Z", "comment": "OK"}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Quotes & RFQ (Hybrid)

```sql
CREATE TABLE quotes (
    quote_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    company_id      UUID NOT NULL REFERENCES companies(company_id),
    buyer_id        UUID REFERENCES buyers(buyer_id),
    sales_rep_id    UUID,
    quote_number    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2),
    valid_until     TIMESTAMPTZ,
    -- Full negotiation history as JSONB array
    negotiations    JSONB NOT NULL DEFAULT '[]',
    -- e.g. [
    --   {"round": 1, "actor": "buyer", "at": "2026-05-18T09:00:00Z",
    --    "lines": [{"product_id":"...","qty":100,"price":12.50}],
    --    "comment": "Can you do better on volume?"},
    --   {"round": 2, "actor": "sales_rep", "at": "2026-05-18T14:30:00Z",
    --    "lines": [{"product_id":"...","qty":100,"price":11.75}],
    --    "comment": "Best I can do for 100 units", "ai_assisted": true},
    --   {"round": 3, "actor": "buyer", "at": "2026-05-19T10:00:00Z",
    --    "action": "accept"}
    -- ]
    converted_order_id UUID REFERENCES orders(order_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, quote_number)
);

CREATE TABLE quote_lines (
    line_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quotes(quote_id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID REFERENCES product_variants(variant_id),
    quantity        INTEGER NOT NULL,
    list_price      NUMERIC(12,4) NOT NULL,
    offered_price   NUMERIC(12,4) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}'
);
```

### Integration & ERP Sync Log

```sql
CREATE TABLE integration_events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    integration     TEXT NOT NULL,  -- 'netsuite', 'quickbooks', 'edi', 'avalara'
    direction       TEXT NOT NULL,  -- 'inbound', 'outbound'
    entity_type     TEXT NOT NULL,  -- 'order', 'invoice', 'product', 'price_list'
    entity_id       UUID NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    -- Full payload stored for debugging and replay
    request_payload  JSONB,
    response_payload JSONB,
    error_details   JSONB,
    attempts        INTEGER NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integration_events ON integration_events (tenant_id, integration, status);
```

### AI Workflow State

```sql
CREATE TABLE ai_tasks (
    task_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    task_type       TEXT NOT NULL,
    -- 'po_ingestion', 'reorder_prediction', 'price_anomaly',
    -- 'quote_draft', 'sku_classification', 'nl_order'
    status          TEXT NOT NULL DEFAULT 'pending',
    input           JSONB NOT NULL,
    -- PO ingestion: {"file_url": "s3://...", "file_type": "pdf", "sender_email": "..."}
    -- NL order: {"utterance": "send the usual order plus 20% on trending SKUs",
    --            "buyer_id": "...", "company_id": "..."}
    output          JSONB,
    -- PO ingestion: {"lines": [...], "confidence": 0.92, "warnings": [...]}
    -- Reorder: {"recommendations": [{"sku": "...", "qty": 50, "reason": "..."}]}
    confidence      NUMERIC(3,2),
    human_review    BOOLEAN NOT NULL DEFAULT false,
    reviewed_by     UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_ai_tasks ON ai_tasks (tenant_id, task_type, status);
```

---

## Pros

- **Best of both worlds**: Core transactional data (pricing, orders, financials) benefits from relational integrity while flexible domains (product attributes, workflow configs, integration payloads) avoid schema-migration friction.
- **Single database engine**: PostgreSQL serves as both relational DB and document store, eliminating the operational overhead of a separate MongoDB or Elasticsearch instance for flexible data.
- **Industry-agnostic product model**: The JSONB `attributes` column on products handles any vertical -- food service allergens, fashion size/color matrices, industrial certifications -- without schema changes. New attribute types require zero migrations.
- **Queryable flexibility**: PostgreSQL GIN indexes on JSONB enable efficient filtering: `WHERE attributes @> '{"organic": true}'` or `WHERE attributes->>'color' = 'Navy'`. This supports dynamic faceted search without a separate search engine for basic needs.
- **Lightweight audit trail**: The `history` JSONB array on orders provides 80% of the audit-trail value of full event sourcing at a fraction of the implementation cost.
- **Integration-friendly**: Raw integration payloads (EDI, ERP responses, AI model outputs) stored as JSONB can be inspected, debugged, and replayed without separate logging infrastructure.
- **Familiar to teams**: The relational core is standard SQL; JSONB columns are a well-documented PostgreSQL feature. No exotic technologies required.
- **Gradual schema hardening**: Start with flexible JSONB for new domains, and promote frequently-queried paths to proper columns via simple `ALTER TABLE ADD COLUMN` + backfill as patterns stabilize.

## Cons

- **JSONB lacks referential integrity**: Foreign keys cannot point into JSONB structures. If `attributes.category_id` references a category, that relationship is not enforced by the database. Application-layer validation is required.
- **Schema discipline required**: Without governance, JSONB columns become dumping grounds for unstructured data, making querying and reporting unreliable. A JSON Schema validation layer (enforced in the application or via PostgreSQL CHECK constraints) is advisable.
- **GIN index overhead**: GIN indexes on large JSONB columns consume significant storage and slow write operations. Selective indexing (expression indexes on specific paths) is preferable to blanket GIN indexes.
- **Reporting complexity**: BI tools and SQL reporting work best with flat columns. JSONB data requires `jsonb_extract_path_text()` or `->>` operators in queries, which are less intuitive for analysts.
- **Migration ambiguity**: The boundary between "should be a column" and "should be JSONB" is a judgment call. Inconsistent decisions across the codebase create confusion.
- **Partial audit trail**: The JSONB `history` array on orders is append-only by convention, not by database constraint. A bug or careless UPDATE can overwrite history. Full event sourcing (Suggestion 2) provides stronger guarantees.
- **JSONB size limits**: Very large JSONB documents (e.g., an order with 500+ history entries) can degrade read performance. Periodic archival or extraction to separate tables may be needed for long-lived entities.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| **Database** | PostgreSQL 16+ (JSONB, GIN indexes, generated columns, RLS) |
| **ORM** | Prisma (good JSONB support in TypeScript), Drizzle ORM, or MikroORM |
| **Validation** | Zod (TypeScript) or JSON Schema for JSONB field validation at the application layer |
| **Search** | PostgreSQL full-text search + GIN for basic catalogue search; Meilisearch for rich faceted search |
| **Caching** | Redis for price resolution and session caching |
| **API layer** | GraphQL (custom resolvers handle JSONB fields naturally) + REST for integrations |
| **Migrations** | Prisma Migrate with custom SQL for JSONB index management |
| **Analytics** | dbt for transforming JSONB data into flat reporting tables; Metabase for dashboards |
| **Hosting** | Supabase (built-in RLS, JSONB support) or AWS Aurora PostgreSQL |

---

## Migration and Scaling Considerations

- **Column promotion pattern**: Track which JSONB paths are queried most frequently. When a path becomes critical for filtering or joins, promote it to a proper column with a generated column or migration:
  ```sql
  -- Add a generated column extracted from JSONB
  ALTER TABLE products ADD COLUMN brand_indexed TEXT
    GENERATED ALWAYS AS (attributes->>'brand') STORED;
  CREATE INDEX idx_products_brand ON products (tenant_id, brand_indexed);
  ```
- **JSON Schema enforcement**: Use PostgreSQL CHECK constraints with `jsonb_matches_schema()` (pg_jsonschema extension) or application-layer validation (Zod) to prevent schema drift in JSONB columns.
- **Partitioning**: Partition `orders` by `created_at` range and `integration_events` by `created_at` for large tenants. JSONB columns partition with the table naturally.
- **JSONB to event sourcing migration**: If audit requirements grow beyond what JSONB history arrays provide, the `history` column serves as a natural migration path -- extract history entries into a proper event store table for the Order aggregate while keeping the hybrid model for other entities.
- **Read replicas**: Route catalogue browsing (heavy JSONB queries with GIN index scans) to read replicas. Keep transactional writes (orders, pricing) on the primary.
- **Archival**: Move completed orders older than N years to an archive table (same schema). Use `pg_partman` for automatic partition management. Integration event payloads can be moved to S3/GCS after 90 days, retaining only a summary row.
- **Multi-tenant scaling**: Start with shared-schema RLS. If a tenant grows large enough to warrant isolation, migrate them to a dedicated schema or database -- the hybrid model's schema is self-contained and portable.
