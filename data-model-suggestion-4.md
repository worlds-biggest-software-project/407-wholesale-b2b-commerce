# Data Model Suggestion 4: PostgreSQL + Apache AGE Graph Layer for Relationship-Centric B2B Commerce

> Project: Wholesale & B2B Commerce (407) | Generated: 2026-05-25

## Summary

A specialty approach that combines PostgreSQL relational tables for transactional data with Apache AGE (A Graph Extension) for modeling the complex web of relationships that define B2B wholesale commerce: company hierarchies, customer-group-to-price-list assignments, catalogue visibility rules, approval chains, and contract-pricing entitlements. In B2B commerce, the question "what price does this buyer see for this product?" requires traversing a graph of relationships -- buyer belongs to company, company is in customer group, customer group is assigned price list, price list contains product price, and the buyer's catalogue restricts which products are visible. Graph queries express this traversal naturally in Cypher, avoiding the multi-join SQL queries that become unwieldy as relationship complexity grows. Apache AGE runs inside PostgreSQL as an extension, so transactional data (orders, invoices, inventory) stays in standard relational tables while the relationship graph coexists in the same database.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     PostgreSQL 16+                           │
│                                                             │
│  ┌──────────────────────┐    ┌────────────────────────────┐ │
│  │   Relational Tables  │    │   Apache AGE Graph Layer   │ │
│  │                      │    │                            │ │
│  │  orders              │    │  (:Company)-[:CHILD_OF]->  │ │
│  │  order_lines         │    │  (:Company)                │ │
│  │  products            │    │                            │ │
│  │  product_variants    │    │  (:Company)-[:MEMBER_OF]-> │ │
│  │  inventory           │    │  (:CustomerGroup)          │ │
│  │  invoices            │    │                            │ │
│  │  ai_tasks            │    │  (:CustomerGroup)-[:HAS]-> │ │
│  │  integration_events  │    │  (:PriceList)              │ │
│  │                      │    │                            │ │
│  │                      │    │  (:Company)-[:SEES]->      │ │
│  │                      │    │  (:Catalogue)              │ │
│  │                      │    │                            │ │
│  │                      │    │  (:Buyer)-[:WORKS_FOR]->   │ │
│  │                      │    │  (:Company)                │ │
│  │                      │    │                            │ │
│  │                      │    │  (:Buyer)-[:APPROVES_FOR]->│ │
│  │                      │    │  (:Buyer)                  │ │
│  └──────────────────────┘    └────────────────────────────┘ │
│                                                             │
│            Joined via shared UUIDs (entity IDs)             │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Entities and Relationships

### Graph Schema (Apache AGE / Cypher)

#### Node Types

```cypher
-- Company hierarchy nodes
CREATE (:Company {
    id: 'comp_001',
    tenant_id: 'tenant_001',
    name: 'Acme Wholesale',
    status: 'active',
    credit_limit: 50000.00,
    payment_terms: 'net_30',
    currency: 'USD',
    industry: 'food_service'
})

-- Buyer nodes (linked to companies)
CREATE (:Buyer {
    id: 'buyer_001',
    email: 'jane@acme.com',
    name: 'Jane Smith',
    role: 'approver',
    spending_limit: 10000.00
})

-- Customer groups
CREATE (:CustomerGroup {
    id: 'cg_gold',
    tenant_id: 'tenant_001',
    name: 'Gold Tier',
    discount_level: 'tier_1'
})

-- Price lists
CREATE (:PriceList {
    id: 'pl_gold_2026',
    tenant_id: 'tenant_001',
    name: 'Gold Tier 2026',
    currency: 'USD',
    priority: 10,
    valid_from: '2026-01-01',
    valid_to: '2026-12-31'
})

-- Catalogues
CREATE (:Catalogue {
    id: 'cat_food',
    tenant_id: 'tenant_001',
    name: 'Food Service Catalogue',
    status: 'active'
})

-- Sales rep nodes
CREATE (:SalesRep {
    id: 'rep_001',
    name: 'Bob Williams',
    territory: 'Northeast'
})

-- Approval workflow nodes
CREATE (:ApprovalRule {
    id: 'rule_5k',
    threshold_type: 'order_total',
    threshold_value: 5000.00,
    description: 'Orders over $5,000 require approval'
})
```

#### Relationship Types

```cypher
-- Company hierarchy
CREATE (child:Company {id:'comp_002'})-[:CHILD_OF {since: '2020-01-15'}]->(parent:Company {id:'comp_001'})

-- Buyer-company membership
CREATE (b:Buyer {id:'buyer_001'})-[:WORKS_FOR {role: 'approver', since: '2023-06-01'}]->(c:Company {id:'comp_001'})

-- Customer group membership
CREATE (c:Company {id:'comp_001'})-[:MEMBER_OF {joined: '2024-01-01', tier: 'gold'}]->(cg:CustomerGroup {id:'cg_gold'})

-- Price list assignment
CREATE (cg:CustomerGroup {id:'cg_gold'})-[:HAS_PRICE_LIST {priority: 10}]->(pl:PriceList {id:'pl_gold_2026'})

-- Direct company price list (contract pricing)
CREATE (c:Company {id:'comp_001'})-[:HAS_CONTRACT_PRICING {
    contract_id: 'contract_2026_001',
    valid_from: '2026-01-01',
    valid_to: '2026-12-31',
    priority: 20  -- higher priority than group pricing
}]->(pl:PriceList {id:'pl_acme_contract'})

-- Catalogue visibility
CREATE (c:Company {id:'comp_001'})-[:SEES_CATALOGUE {assigned: '2024-01-01'}]->(cat:Catalogue {id:'cat_food'})
CREATE (cg:CustomerGroup {id:'cg_gold'})-[:SEES_CATALOGUE]->(cat:Catalogue {id:'cat_food'})

-- Sales rep assignment
CREATE (rep:SalesRep {id:'rep_001'})-[:MANAGES {since: '2023-01-01'}]->(c:Company {id:'comp_001'})

-- Approval chains
CREATE (b1:Buyer {id:'buyer_002'})-[:APPROVED_BY {
    for_rule: 'rule_5k',
    step_order: 1
}]->(b2:Buyer {id:'buyer_001'})

-- Company approves via rule
CREATE (c:Company {id:'comp_001'})-[:USES_APPROVAL]->(r:ApprovalRule {id:'rule_5k'})
```

### Core Graph Queries (Cypher via AGE)

#### Resolve all applicable price lists for a buyer

```cypher
-- Find all price lists applicable to buyer_001, ordered by priority
MATCH (b:Buyer {id: 'buyer_001'})-[:WORKS_FOR]->(c:Company)
OPTIONAL MATCH (c)-[:HAS_CONTRACT_PRICING]->(contract_pl:PriceList)
OPTIONAL MATCH (c)-[:MEMBER_OF]->(cg:CustomerGroup)-[:HAS_PRICE_LIST]->(group_pl:PriceList)
WITH b, c,
     COLLECT(DISTINCT {
         price_list: contract_pl,
         source: 'contract',
         priority: contract_pl.priority
     }) + COLLECT(DISTINCT {
         price_list: group_pl,
         source: 'group',
         priority: group_pl.priority
     }) AS all_price_lists
UNWIND all_price_lists AS pl
WHERE pl.price_list IS NOT NULL
RETURN pl.price_list.id, pl.price_list.name, pl.source, pl.priority
ORDER BY pl.priority DESC
```

#### Determine visible catalogue for a buyer

```cypher
-- All products visible to buyer through company or group catalogue assignments
MATCH (b:Buyer {id: 'buyer_001'})-[:WORKS_FOR]->(c:Company)
OPTIONAL MATCH (c)-[:SEES_CATALOGUE]->(direct_cat:Catalogue)
OPTIONAL MATCH (c)-[:MEMBER_OF]->(cg:CustomerGroup)-[:SEES_CATALOGUE]->(group_cat:Catalogue)
WITH COLLECT(DISTINCT direct_cat) + COLLECT(DISTINCT group_cat) AS catalogues
UNWIND catalogues AS cat
RETURN cat.id, cat.name
```

#### Find approval chain for an order

```cypher
-- Walk the approval chain for buyer_002 under rule_5k
MATCH path = (buyer:Buyer {id: 'buyer_002'})-[:APPROVED_BY*1..5]->(approver:Buyer)
WHERE ALL(r IN relationships(path) WHERE r.for_rule = 'rule_5k')
RETURN [node IN nodes(path) | node.name] AS approval_chain,
       length(path) AS steps
ORDER BY steps
```

#### Traverse company hierarchy

```cypher
-- Find all subsidiaries of a parent company (recursive)
MATCH path = (subsidiary:Company)-[:CHILD_OF*1..10]->(parent:Company {id: 'comp_001'})
RETURN subsidiary.id, subsidiary.name, length(path) AS depth
ORDER BY depth
```

#### Find sales rep for a buyer (through company)

```cypher
MATCH (b:Buyer {id: 'buyer_001'})-[:WORKS_FOR]->(c:Company)<-[:MANAGES]-(rep:SalesRep)
RETURN rep.id, rep.name, rep.territory
```

### Relational Tables (Standard PostgreSQL)

```sql
-- Products (relational -- high volume, stable schema)
CREATE TABLE products (
    product_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    sku             TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    product_type    TEXT NOT NULL DEFAULT 'simple',
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    case_pack_qty   INTEGER NOT NULL DEFAULT 1,
    gtin            TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

-- Prices (relational -- high volume, numeric precision critical)
CREATE TABLE prices (
    price_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    price_list_id   UUID NOT NULL,   -- references PriceList graph node ID
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID,
    min_quantity    INTEGER NOT NULL DEFAULT 1,
    price           NUMERIC(12,4) NOT NULL,
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prices_lookup ON prices (price_list_id, product_id, min_quantity);

-- Catalogue product membership (relational junction)
CREATE TABLE catalogue_products (
    catalogue_id    UUID NOT NULL,  -- references Catalogue graph node ID
    product_id      UUID NOT NULL REFERENCES products(product_id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (catalogue_id, product_id)
);

-- Orders (relational -- transactional, ACID-critical)
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    company_id      UUID NOT NULL,   -- references Company graph node ID
    buyer_id        UUID NOT NULL,   -- references Buyer graph node ID
    order_number    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    po_number       TEXT,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total           NUMERIC(12,2) NOT NULL DEFAULT 0,
    payment_terms   TEXT,
    source          TEXT NOT NULL DEFAULT 'storefront',
    source_metadata JSONB NOT NULL DEFAULT '{}',
    shipping_address JSONB NOT NULL DEFAULT '{}',
    history         JSONB NOT NULL DEFAULT '[]',
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, order_number)
);

CREATE TABLE order_lines (
    line_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID,
    sku             TEXT NOT NULL,
    product_name    TEXT NOT NULL,
    quantity        INTEGER NOT NULL,
    unit_price      NUMERIC(12,4) NOT NULL,
    line_total      NUMERIC(12,2) NOT NULL,
    price_list_id   UUID,
    metadata        JSONB NOT NULL DEFAULT '{}'
);

-- Quotes (relational)
CREATE TABLE quotes (
    quote_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    company_id      UUID NOT NULL,
    buyer_id        UUID,
    sales_rep_id    UUID,
    quote_number    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2),
    valid_until     TIMESTAMPTZ,
    negotiations    JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, quote_number)
);

CREATE TABLE quote_lines (
    line_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quotes(quote_id),
    product_id      UUID NOT NULL REFERENCES products(product_id),
    quantity        INTEGER NOT NULL,
    list_price      NUMERIC(12,4) NOT NULL,
    offered_price   NUMERIC(12,4) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}'
);

-- Inventory
CREATE TABLE inventory (
    product_id      UUID NOT NULL REFERENCES products(product_id),
    variant_id      UUID,
    warehouse_id    UUID NOT NULL,
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (product_id, COALESCE(variant_id, '00000000-0000-0000-0000-000000000000'), warehouse_id)
);

-- AI tasks
CREATE TABLE ai_tasks (
    task_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    task_type       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    input           JSONB NOT NULL,
    output          JSONB,
    confidence      NUMERIC(3,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

### Bridging Graph and Relational Data

```sql
-- Example: Get the best price for a product for a given buyer
-- Step 1: Use Cypher (via AGE) to get applicable price list IDs in priority order
-- Step 2: Use SQL to find the best price from those lists

-- Combined query using AGE's SQL integration
SELECT p.price, p.min_quantity, p.price_list_id
FROM prices p
WHERE p.product_id = $product_id
  AND p.price_list_id IN (
      -- Subquery via AGE: get price list IDs for this buyer
      SELECT (pl->>'id')::UUID
      FROM cypher('b2b_graph', $$
          MATCH (b:Buyer {id: $buyer_id})-[:WORKS_FOR]->(c:Company)
          OPTIONAL MATCH (c)-[:HAS_CONTRACT_PRICING]->(cp:PriceList)
          OPTIONAL MATCH (c)-[:MEMBER_OF]->(cg:CustomerGroup)-[:HAS_PRICE_LIST]->(gp:PriceList)
          WITH COLLECT(cp) + COLLECT(gp) AS pls
          UNWIND pls AS pl
          RETURN pl
      $$) AS (pl agtype)
  )
  AND p.min_quantity <= $requested_quantity
  AND (p.valid_from IS NULL OR p.valid_from <= now())
  AND (p.valid_to IS NULL OR p.valid_to >= now())
ORDER BY p.price ASC
LIMIT 1;
```

---

## Pros

- **Natural relationship modeling**: B2B commerce is fundamentally about relationships: buyer-to-company, company-to-group, group-to-price-list, company-to-catalogue. Graph queries express "what can this buyer see and at what price?" as a natural path traversal rather than a 6-table JOIN.
- **Flexible hierarchy depth**: Company hierarchies (parent -> subsidiary -> division -> department) are variable-depth trees. Recursive CTEs in SQL are awkward and slow; graph traversal handles arbitrary depth naturally with `[:CHILD_OF*1..N]`.
- **Approval chain modeling**: Multi-step, branching approval chains with escalation paths are graph problems. Modeling them as graph edges (`:APPROVED_BY`, `:ESCALATES_TO`) is more intuitive and queryable than relational adjacency lists.
- **Single database engine**: Apache AGE runs inside PostgreSQL. No separate Neo4j instance to deploy, backup, or keep in sync. Transactional data stays in standard tables with ACID guarantees.
- **Impact analysis**: "If I change Gold Tier pricing, which companies and buyers are affected?" is a simple graph traversal. In relational, this requires joining through multiple association tables.
- **Sales territory and rep assignment**: Territory structures, rep-to-company assignments, and coverage gaps are naturally visualized and queried as graphs.
- **Schema evolution for relationships**: Adding a new relationship type (e.g., `:REFERRED_BY` for referral tracking) requires no schema migration -- just create new edges. In relational, this means a new junction table.
- **AI-friendly structure**: Graph embeddings can feed recommendation and anomaly detection models. The relationship graph provides rich context for AI features like "suggest similar buyers" or "predict which accounts need attention."

## Cons

- **Apache AGE maturity**: AGE is a younger extension compared to core PostgreSQL features. The ecosystem of tools, ORMs, and documentation is smaller than mature graph databases like Neo4j.
- **Operational complexity**: Teams need Cypher expertise in addition to SQL. Debugging involves two query languages. Monitoring and query optimization for AGE graphs is less mature than for relational PostgreSQL.
- **Data synchronization**: Entity IDs must be consistent between graph nodes and relational tables. Creating a Company requires inserting into both the graph and potentially relational tables, adding transactional coordination.
- **Limited cloud support**: AGE may not be available on all managed PostgreSQL services (e.g., AWS RDS supports limited extensions). Self-hosted or specific providers (e.g., Azure Database for PostgreSQL) may be required.
- **Graph anti-patterns**: Not everything should be in the graph. High-volume, write-heavy data (order lines, price rows, inventory) belongs in relational tables. Putting too much in the graph degrades performance and complicates transactions.
- **Query planning**: The PostgreSQL query planner does not optimize AGE Cypher queries the same way it optimizes SQL. Complex graph queries may require manual optimization.
- **Backup and migration**: Graph data in AGE uses internal PostgreSQL schemas but requires AGE-aware backup and restore procedures. Standard pg_dump works but requires AGE extension on the target.
- **Hiring**: Finding developers comfortable with both SQL and Cypher is harder than finding SQL-only developers. Training investment is required.
- **No standard ORM support**: Prisma, Drizzle, and other popular ORMs do not have AGE graph support. The graph layer requires raw Cypher queries or a custom abstraction.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| **Database** | PostgreSQL 16+ with Apache AGE extension |
| **Graph queries** | Cypher via AGE `cypher()` function; custom TypeScript/Python wrapper |
| **Relational ORM** | Prisma or Drizzle for relational tables |
| **Graph client** | apache-age-client (Node.js), age-python, or custom SQL wrapper |
| **Visualization** | Apache AGE Viewer for development; D3.js or Cytoscape.js for admin UI |
| **Search** | Meilisearch or Typesense for product catalogue search |
| **Caching** | Redis for resolved price lists and catalogue visibility (cache the graph traversal results) |
| **API layer** | GraphQL (natural fit: graph DB backing a graph API) |
| **Hosting** | Self-hosted PostgreSQL with AGE, Azure Database for PostgreSQL (AGE supported), or Aiven |
| **Alternative** | If AGE maturity is a concern, consider Neo4j alongside PostgreSQL with sync via change-data-capture (Debezium) |

---

## Migration and Scaling Considerations

- **Start small with the graph**: Begin by modeling only the relationship-heavy domains in the graph (company hierarchy, customer-group assignments, catalogue visibility, approval chains). Keep products, orders, and prices fully relational. Expand the graph only if traversal queries prove valuable.
- **Graph caching strategy**: Graph traversal results (e.g., "buyer X can see catalogues A, B, C with price lists P1, P2") change infrequently. Cache resolved entitlements in Redis with TTL and invalidate on relationship changes. This avoids running graph queries on every page load.
- **Hybrid queries**: Use a service layer that first resolves entitlements via Cypher (which price lists? which catalogues?), then queries relational tables with the resolved IDs. This keeps the graph layer focused on relationship traversal and the relational layer on data retrieval.
- **AGE alternatives**: If AGE extension availability is a blocker, the same graph model can be implemented in:
  - **Neo4j** alongside PostgreSQL (separate database, synced via CDC)
  - **Recursive CTEs** in pure PostgreSQL (less elegant but no extension required)
  - **Adjacency list tables** with materialized path columns (classic relational graph pattern)
- **Data consistency**: Wrap graph node creation and relational inserts in the same PostgreSQL transaction (AGE operates within PostgreSQL transactions). This ensures atomicity when creating a Company node and its associated relational records.
- **Performance benchmarking**: Before committing to this architecture, benchmark the key queries (price resolution, catalogue visibility, approval chain traversal) against the equivalent relational JOINs in Suggestion 1. The graph approach wins decisively for deep hierarchies (5+ levels) and complex entitlement rules, but may not justify the added complexity for flat structures.
- **Migration path from relational**: If starting with Suggestion 1 or 3 and evolving toward this model, the graph layer can be added incrementally: import existing company, group, and price-list relationships into AGE, update the entitlement resolution service to use Cypher, and keep all other code unchanged.
- **Scaling the graph**: AGE graphs scale with PostgreSQL. For very large graphs (millions of nodes/edges), ensure adequate shared_buffers and work_mem. Partition the graph by tenant if needed using separate AGE graph names per tenant.
