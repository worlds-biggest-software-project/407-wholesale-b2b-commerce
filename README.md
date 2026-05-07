# Wholesale & B2B Commerce

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, API-first wholesale commerce platform that enforces negotiated pricing, custom catalogues, and procurement workflows across every sales channel.

Wholesale and B2B commerce is fundamentally different from consumer ecommerce: transactions involve negotiated price lists, customer-group-specific catalogues, order minimums, approval chains, and trade credit terms. Existing platforms are either expensive enterprise monoliths or consumer platforms with B2B features bolted on. This project builds a purpose-built, headless-first B2B commerce engine that treats AI-powered ordering, pricing enforcement, and procurement integration as first-class capabilities rather than afterthoughts.

---

## Why Wholesale & B2B Commerce?

- **Enterprise platforms carry prohibitive TCO.** Mid-market B2B platforms carry 3-year total costs ranging from $100K to over $1M depending on customisation requirements. Adobe Commerce, SAP Commerce Cloud, and Salesforce Commerce Cloud demand significant implementation investment and complex licensing (per-org, per-user, or GMV-based).
- **Consumer platforms retrofit B2B poorly.** Shopify and BigCommerce have extended B2B features to broader tiers, but lack depth in approval workflows, multi-org hierarchies, and procurement integration (punchout/EDI) without third-party add-ons.
- **The only open-source option is limited.** OroCommerce (OSL-3.0) is purpose-built for B2B but has an aging storefront UX, a smaller community than Magento, and higher operational overhead. Magento Open Source gates most B2B features behind the commercial Adobe Commerce licence.
- **Procurement integration remains fragmented.** Punchout (cXML/OCI) and EDI connectors are expensive, rarely turnkey for SMB sellers, and typically require third-party middleware even on enterprise platforms.
- **No platform addresses AI-native B2B workflows.** Conversational reordering, automated PO ingestion from PDFs and emails, AI-assisted price-list maintenance, and real-time credit-risk scoring are underserved across the entire market.

---

## Key Features

### Accounts & Pricing

- Company/account model with multiple buyers and role-based permissions
- Customer-group price lists with fixed prices, percentage discounts, and quantity breaks
- Contract pricing enforced across storefront, sales rep portal, and order management
- Net-30/60/90 payment terms, credit limits, and purchase-order payment methods
- Order minimums, case-pack multiples, and minimum line quantities at checkout

### Catalogues & Ordering

- Custom catalogues restricted by customer group, geography, or channel
- Quick reorder and saved order lists for regular replenishment
- Sales rep "order on behalf of" mode with account history access
- Quote/RFQ workflow with negotiation history and counter-offers
- Approval workflows with configurable thresholds and chains

### AI-Powered Capabilities

- Reorder prediction from purchase history and seasonality patterns
- PDF/email purchase-order ingestion into structured draft orders
- Natural-language ordering and product discovery for long catalogues
- Anomaly detection on order patterns (fraud, mistakes, demand shifts)
- AI-generated quote responses and negotiation drafts for sales reps
- Auto-classification of new SKUs into customer-specific catalogues

### Integration & Extensibility

- REST and GraphQL APIs with webhooks; headless-first architecture
- ERP integration adapters (NetSuite, QuickBooks, SAP B1, generic CSV bridge)
- Tax engine integration hooks (Avalara, Vertex)
- Punchout (cXML, OCI) and EDI 850/810/856 connectors (backlog)
- Same-storefront B2B and DTC or B2B-only deployment

---

## AI-Native Advantage

Current B2B platforms treat AI as a bolt-on recommendation engine at best. This project embeds AI at the workflow level: buyers can reorder using natural language ("send the usual order plus 20% on SKUs trending up"), inbound purchase orders from PDFs and emails are automatically parsed into platform orders, and price-list maintenance is continuously audited for drift between contract terms and active prices. These capabilities address real friction points -- manual PO re-keying, stale price lists, and missed reorder windows -- that existing platforms leave to human operators.

---

## Tech Stack & Deployment

The platform is designed as a headless, API-first engine supporting self-hosted, cloud, and hybrid deployment. The storefront layer is decoupled, allowing teams to build with frameworks like Next.js or Remix. EDI X12 protocol implementation is unencumbered, though ASC X12 specification documents require a subscription for reproduction. The OSL-3.0 licence used by OroCommerce and Magento Open Source is copyleft and incompatible with permissive distribution, so this project will not bundle OSL-licensed code into its core.

---

## Market Context

The B2B ecommerce market continues to grow rapidly, with platforms at every price point competing for mid-market and enterprise buyers. Three-year TCOs for mid-market solutions range from $100K to over $1M, while enterprise platforms like Salesforce and SAP carry complex subscription and GMV-based licensing. Primary buyers are manufacturers, distributors, and wholesalers selling to business accounts who need negotiated pricing, procurement integration, and trade credit -- capabilities that consumer ecommerce platforms only partially address.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
