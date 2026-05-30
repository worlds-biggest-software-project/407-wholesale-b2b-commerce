# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Wholesale & B2B Commerce (407) | Generated: 2026-05-25

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS), where every state change in the system is captured as an immutable event rather than overwriting current state. The write side appends events to an event store; the read side builds optimized projections (materialized views) from those events for querying. This approach excels at the audit-heavy, negotiation-rich, and compliance-sensitive nature of B2B wholesale commerce -- where knowing *how* a price was agreed, *who* approved an order, and *what changed* in a contract is as important as knowing the current state.

---

## Core Architecture

```
                  ┌─────────────────┐
  Commands ──────>│  Command Handler │──────> Event Store (append-only)
                  └─────────────────┘              │
                                                   │ events
                                                   ▼
                                          ┌──────────────┐
                                          │  Projectors   │
                                          └──────────────┘
                                                   │
                              ┌────────────────────┼────────────────────┐
                              ▼                    ▼                    ▼
                     ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
                     │ Catalogue DB │   │  Order Views  │   │ Pricing View │
                     │ (read model) │   │ (read model)  │   │ (read model) │
                     └──────────────┘   └──────────────┘   └──────────────┘
                              │                    │                    │
  Queries <───────────────────┴────────────────────┴────────────────────┘
```

---

## Key Entities as Event Streams

### Event Store Schema

```sql
-- Core event store table (PostgreSQL-backed, similar to Marten's approach)
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,         -- aggregate root ID
    stream_type     TEXT NOT NULL,          -- 'Order', 'Quote', 'PriceList', 'Company'
    event_type      TEXT NOT NULL,          -- 'OrderPlaced', 'PriceUpdated', etc.
    event_data      JSONB NOT NULL,         -- full event payload
    metadata        JSONB NOT NULL DEFAULT '{}',  -- tenant_id, user_id, correlation_id, causation_id
    sequence_number BIGINT NOT NULL,        -- per-stream ordering
    global_position BIGSERIAL NOT NULL,     -- global ordering for projections
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

CREATE INDEX idx_event_store_stream ON event_store (stream_id, sequence_number);
CREATE INDEX idx_event_store_global ON event_store (global_position);
CREATE INDEX idx_event_store_type   ON event_store (stream_type, event_type);
CREATE INDEX idx_event_store_tenant ON event_store ((metadata->>'tenant_id'));
```

### Domain Events by Aggregate

#### Company / Account Events

```typescript
// Company lifecycle events
interface CompanyRegistered {
  companyId: string;
  tenantId: string;
  name: string;
  taxId?: string;
  parentCompanyId?: string;
}

interface CreditLimitSet {
  companyId: string;
  creditLimit: number;
  currency: string;
  approvedBy: string;
  previousLimit?: number;
}

interface PaymentTermsUpdated {
  companyId: string;
  terms: 'net_30' | 'net_60' | 'net_90' | 'cod';
  previousTerms?: string;
  effectiveDate: string;
}

interface BuyerAddedToCompany {
  companyId: string;
  buyerId: string;
  email: string;
  role: 'buyer' | 'approver' | 'admin';
  spendingLimit?: number;
}
```

#### Price List Events

```typescript
interface PriceListCreated {
  priceListId: string;
  tenantId: string;
  name: string;
  currencyCode: string;
  priority: number;
  validFrom?: string;
  validTo?: string;
}

interface PriceSet {
  priceListId: string;
  productId: string;
  variantId?: string;
  minQuantity: number;
  price: number;
  previousPrice?: number;
  reason?: string;  // 'contract_renewal', 'cost_increase', 'ai_recommendation'
}

interface PriceListAssignedToGroup {
  priceListId: string;
  customerGroupId: string;
}

interface PriceListExpired {
  priceListId: string;
  expiredAt: string;
}
```

#### Order Lifecycle Events

```typescript
interface OrderDraftCreated {
  orderId: string;
  tenantId: string;
  companyId: string;
  buyerId: string;
  currencyCode: string;
  source: 'storefront' | 'sales_rep' | 'ai_agent' | 'po_ingestion' | 'edi';
}

interface OrderLineAdded {
  orderId: string;
  lineId: string;
  productId: string;
  sku: string;
  quantity: number;
  unitPrice: number;
  priceListId: string;
}

interface OrderSubmitted {
  orderId: string;
  poNumber?: string;
  subtotal: number;
  taxTotal: number;
  total: number;
  submittedBy: string;
}

interface OrderApprovalRequested {
  orderId: string;
  workflowId: string;
  stepId: string;
  approverId: string;
  reason: string;  // e.g. 'order_total_exceeds_1000'
}

interface OrderApproved {
  orderId: string;
  approvedBy: string;
  stepId: string;
  comment?: string;
}

interface OrderRejected {
  orderId: string;
  rejectedBy: string;
  stepId: string;
  reason: string;
}

interface OrderFulfillmentStarted {
  orderId: string;
  warehouseId: string;
  estimatedShipDate: string;
}

interface OrderShipped {
  orderId: string;
  shipmentId: string;
  trackingNumber: string;
  carrier: string;
  lines: { lineId: string; quantityShipped: number }[];
}
```

#### Quote / RFQ Negotiation Events

```typescript
interface QuoteRequested {
  quoteId: string;
  tenantId: string;
  companyId: string;
  buyerId: string;
  lines: { productId: string; quantity: number; requestedPrice?: number }[];
}

interface QuoteResponseDrafted {
  quoteId: string;
  salesRepId: string;
  lines: { productId: string; quantity: number; offeredPrice: number }[];
  validUntil: string;
  aiAssisted: boolean;  // flag if AI generated the draft
}

interface QuoteCounterOffered {
  quoteId: string;
  actorType: 'buyer' | 'sales_rep';
  actorId: string;
  lines: { productId: string; quantity: number; counterPrice: number }[];
  comment?: string;
}

interface QuoteAccepted {
  quoteId: string;
  acceptedBy: string;
  convertedOrderId?: string;
}
```

### Read Model Projections

```sql
-- Projection: Current order state (rebuilt from events)
CREATE TABLE order_projections (
    order_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    company_id      UUID NOT NULL,
    company_name    TEXT NOT NULL,
    buyer_id        UUID NOT NULL,
    buyer_email     TEXT NOT NULL,
    order_number    TEXT NOT NULL,
    status          TEXT NOT NULL,
    po_number       TEXT,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(12,2),
    tax_total       NUMERIC(12,2),
    total           NUMERIC(12,2),
    line_count      INTEGER NOT NULL DEFAULT 0,
    source          TEXT NOT NULL,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    shipped_at      TIMESTAMPTZ,
    last_event_pos  BIGINT NOT NULL,      -- track projection position
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Effective prices per customer group (denormalized for fast lookup)
CREATE TABLE effective_price_projections (
    tenant_id       UUID NOT NULL,
    customer_group_id UUID NOT NULL,
    product_id      UUID NOT NULL,
    variant_id      UUID,
    currency_code   CHAR(3) NOT NULL,
    min_quantity    INTEGER NOT NULL,
    effective_price NUMERIC(12,4) NOT NULL,
    price_list_id   UUID NOT NULL,
    price_list_name TEXT NOT NULL,
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    PRIMARY KEY (tenant_id, customer_group_id, product_id, COALESCE(variant_id, '00000000-0000-0000-0000-000000000000'), min_quantity)
);

-- Projection: Quote negotiation timeline (flattened for UI)
CREATE TABLE quote_timeline_projections (
    quote_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    event_sequence  INTEGER NOT NULL,
    event_type      TEXT NOT NULL,
    actor_type      TEXT NOT NULL,
    actor_id        UUID NOT NULL,
    actor_name      TEXT,
    summary         TEXT NOT NULL,     -- human-readable summary
    line_snapshot   JSONB,             -- lines at this point in negotiation
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (quote_id, event_sequence)
);

-- Projection: Company credit usage (running balance)
CREATE TABLE credit_usage_projections (
    company_id      UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    credit_limit    NUMERIC(12,2) NOT NULL DEFAULT 0,
    outstanding     NUMERIC(12,2) NOT NULL DEFAULT 0,
    available       NUMERIC(12,2) NOT NULL DEFAULT 0,
    last_order_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Aggregate Root Example (Order)

```typescript
class OrderAggregate {
  private events: DomainEvent[] = [];
  private state: OrderState;

  static create(cmd: CreateOrderDraft): OrderAggregate {
    const order = new OrderAggregate();
    order.apply(new OrderDraftCreated({
      orderId: generateId(),
      tenantId: cmd.tenantId,
      companyId: cmd.companyId,
      buyerId: cmd.buyerId,
      currencyCode: cmd.currencyCode,
      source: cmd.source,
    }));
    return order;
  }

  addLine(cmd: AddOrderLine): void {
    this.assertStatus('draft');
    this.assertCasePackMultiple(cmd.quantity, cmd.casePackQty);
    this.apply(new OrderLineAdded({ ... }));
  }

  submit(cmd: SubmitOrder): void {
    this.assertStatus('draft');
    this.assertMinimumOrderValue(cmd.minimumOrderValue);
    this.assertCreditAvailable(cmd.creditAvailable, this.state.total);
    this.apply(new OrderSubmitted({ ... }));

    if (this.requiresApproval(cmd.approvalRules)) {
      this.apply(new OrderApprovalRequested({ ... }));
    }
  }

  approve(cmd: ApproveOrder): void {
    this.assertStatus('pending_approval');
    this.assertIsApprover(cmd.approverId);
    this.apply(new OrderApproved({ ... }));
  }

  private apply(event: DomainEvent): void {
    this.events.push(event);
    this.evolve(event);
  }

  private evolve(event: DomainEvent): void {
    switch (event.type) {
      case 'OrderDraftCreated':
        this.state = { status: 'draft', lines: [], total: 0, ... };
        break;
      case 'OrderLineAdded':
        this.state.lines.push(event.data);
        this.state.total += event.data.unitPrice * event.data.quantity;
        break;
      case 'OrderSubmitted':
        this.state.status = 'pending_approval';
        break;
      case 'OrderApproved':
        this.state.status = 'approved';
        break;
      // ...
    }
  }
}
```

---

## Pros

- **Complete audit trail by design**: Every state change is an immutable event. For B2B commerce, this means full traceability of price changes, approval decisions, quote negotiations, credit limit adjustments, and order modifications -- invaluable for compliance, dispute resolution, and regulatory requirements (e.g., GDPR right-to-explanation, e-invoicing mandates).
- **Rich negotiation history**: Quote counter-offers, approval chains, and pricing discussions are naturally captured as event sequences, not lossy status columns. The quote timeline is a first-class artifact.
- **Temporal queries**: "What was this customer's price list on March 15th?" or "Show the order state before the last modification" are trivial -- replay events up to that point.
- **AI integration friendly**: Events are ideal training data for AI models (reorder prediction, anomaly detection, price optimization). The event stream is a natural source for ML feature stores. AI-generated actions (PO ingestion, quote drafts) are recorded as first-class events with `source: 'ai_agent'`.
- **Independent read scaling**: Read models (projections) can be tailored per use case -- fast catalogue browsing, order dashboards, pricing lookups -- and scaled independently of the write side.
- **Domain alignment**: B2B order lifecycle (draft -> submit -> approve -> fulfill -> ship -> invoice) maps naturally to an event stream. Business rules live in aggregate roots, making pricing enforcement and approval logic testable.
- **Integration event bus**: The event store doubles as the source for outbound integration events (webhooks, EDI, ERP sync) via CloudEvents-formatted subscriptions.

## Cons

- **Complexity**: Event sourcing is significantly more complex to implement and operate than CRUD. Teams need to understand aggregates, projections, idempotency, and eventual consistency.
- **Eventual consistency**: Read models are updated asynchronously. A buyer who places an order may not see it in their order list for a few hundred milliseconds. This is usually acceptable but requires careful UX design (optimistic UI updates).
- **Projection maintenance**: Each new read model requires building and maintaining a projector. Schema changes to projections require replay from the event store, which can be slow for large streams.
- **Event schema evolution**: As the domain evolves, event schemas must be versioned. Old events must remain parseable forever. Upcasting strategies add complexity.
- **Storage growth**: Storing every event accumulates significant data over time. A high-volume B2B platform processing thousands of orders daily with dozens of events per order generates substantial event store growth.
- **Debugging difficulty**: Current state is derived, not directly visible. Debugging requires replaying event streams or inspecting projections, which is less intuitive than querying a single table.
- **Tooling maturity**: Fewer off-the-shelf tools compared to relational. ORMs, admin panels, and reporting tools assume mutable state.
- **Team skill requirements**: Requires developers with event sourcing experience, which is less common than relational database skills.

---

## Technology Recommendations

| Layer | Recommendation |
|-------|---------------|
| **Event store** | PostgreSQL + Marten (.NET) or custom event table (Node.js/TypeScript); EventStoreDB for dedicated event store |
| **Event bus** | Apache Kafka or NATS JetStream for durable event streaming; CloudEvents envelope format |
| **Read-model DB** | PostgreSQL for transactional projections; Redis for hot caches (price lookups); Elasticsearch for catalogue search |
| **Framework** | Axon Framework (Java/Kotlin), Marten (C#/.NET), or custom with TypeScript (Emmett library) |
| **Projection engine** | Custom projectors subscribing to event store; or Kafka Streams / Flink for complex event processing |
| **API layer** | GraphQL for storefront reads (projections); REST for command submission and integrations |
| **Monitoring** | Track projection lag (time between event publish and projection update); alert on lag > 1s |
| **Schema registry** | Confluent Schema Registry or custom JSON Schema registry for event versioning |

---

## Migration and Scaling Considerations

- **Incremental adoption**: Start with event sourcing for the highest-value aggregates (Orders, Quotes, PriceLists) and use a traditional relational model for simpler entities (Products, Categories, Addresses). This "CQRS lite" approach captures 80% of the audit/history benefits with lower complexity.
- **Event store partitioning**: Partition events by `stream_type` or by tenant for large multi-tenant deployments. Archive old events to cold storage (S3 + Parquet) while keeping recent events hot.
- **Projection rebuild**: Design projections so they can be rebuilt from scratch by replaying the event store. This is essential for schema changes and bug fixes. Target rebuild time of < 1 hour for full dataset.
- **Snapshot optimization**: For long-lived aggregates (e.g., a Company with thousands of events over years), store periodic snapshots of aggregate state to avoid replaying the full event history on every command.
- **Idempotency**: All command handlers must be idempotent. Use `correlation_id` and `causation_id` metadata to detect and deduplicate retried commands, which is critical for reliable PO ingestion and EDI processing.
- **Migration from relational**: If migrating from Suggestion 1, create an "initial state" event for each existing aggregate (e.g., `OrderMigratedFromLegacy`) containing the full current state, then switch to event-sourced writes going forward.
- **Multi-region**: Event stores support multi-region replication via Kafka MirrorMaker or EventStoreDB's built-in replication. Projections can be rebuilt per-region for low-latency reads.
