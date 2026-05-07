# Wholesale & B2B Commerce — Feature & Functionality Survey

> Candidate #407 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Shopify B2B | SaaS, hosted | Commercial (Shopify plans incl. B2B; Plus historically required, now broader) | https://www.shopify.com/plus/solutions/b2b-ecommerce |
| BigCommerce B2B Edition | SaaS, hosted | Commercial (B2B Edition tier) | https://www.bigcommerce.com/solutions/b2b-ecommerce-platform/ |
| OroCommerce | Open-core / commercial | OSL-3.0 community edition; commercial Enterprise | https://oroinc.com/b2b-ecommerce/ |
| Adobe Commerce (Magento) | Enterprise platform | Commercial (Adobe Commerce); Open Source edition (OSL-3.0) | https://business.adobe.com/products/magento/magento-commerce.html |
| Salesforce Commerce Cloud B2B | SaaS, enterprise | Commercial (subscription, GMV-based) | https://www.salesforce.com/commerce/b2b/ |
| SAP Commerce Cloud (Hybris) | Enterprise platform | Commercial (subscription) | https://www.sap.com/products/crm/commerce-cloud.html |
| commercetools | Headless / MACH | Commercial (subscription, API-call based) | https://commercetools.com/ |
| Spryker | Headless / composable | Commercial (subscription); Spryker OS components | https://spryker.com/ |
| RepSpark | Wholesale specialist (fashion/CPG) | Commercial SaaS | https://www.repspark.com/ |
| WizCommerce | Wholesale specialist (distributors) | Commercial SaaS | https://wizcommerce.com/ |
| Handshake (Shopify) | Wholesale marketplace | Commercial (free for buyers/sellers) | https://www.handshake.com/ |
| Pepperi | B2B sales / e-commerce / DSD | Commercial SaaS | https://www.pepperi.com/ |

## Feature Analysis by Solution

### Shopify B2B

**Core features**
- Company profiles with multiple buyers and locations
- Custom catalogues per company / location
- Price lists with fixed prices, percentage discounts, and quantity breaks
- Payment terms (Net 7/15/30/60/90) and vaulted payment methods
- Draft orders and quote-style workflows
- Same-storefront B2B & DTC, or B2B-only stores
- Order minimums and quantity rules (case packs)

**Differentiating features**
- Native B2B inside the same Shopify admin used for DTC — single product catalogue, single inventory
- 2026 expansion: B2B features extended below the Plus tier
- Tight integration with Shop Pay, Shopify Functions, and the Shopify App ecosystem

**UX patterns**
- Buyer-facing storefront uses standard Online Store / Hydrogen themes with B2B sections
- Admin-style draft order flow for sales-assisted selling
- Customer Account experience with company switcher

**Integration points**
- REST and GraphQL Admin APIs; Storefront API; Hydrogen/Oxygen
- Shopify Flow for approval automations
- App ecosystem for EDI, ERP, and punchout

**Known gaps**
- Punchout (cXML/OCI) requires third-party apps
- Limited approval workflow depth out of the box
- Less suited to deeply hierarchical enterprise org structures

**Licence / IP notes**
- Proprietary; APIs covered by Shopify API ToS

### BigCommerce B2B Edition

**Core features**
- Buyer Portal (account management, quotes, invoices, lists)
- Customer groups with price lists
- Quote management with sales rep tools
- Invoice/A-R portal and payment-on-account
- Shared shopping lists and quick order pad
- Stencil and headless (Catalyst) storefront options

**Differentiating features**
- B2B Edition acquired from BundleB2B; tightly integrated as first-party module
- Strong quote-to-order workflow with sales-rep impersonation

**UX patterns**
- Distinct Buyer Portal app separate from storefront
- Sales rep "masquerade" / order-on-behalf-of UX

**Integration points**
- REST/GraphQL Storefront and Management APIs
- Webhooks; native ERP connectors (NetSuite, SAP, MS Dynamics) via partners
- Headless via Catalyst (Next.js) or BigCommerce for WordPress

**Known gaps**
- Punchout still typically third-party
- Multi-warehouse / complex inventory weaker than enterprise platforms

**Licence / IP notes**
- Proprietary SaaS

### OroCommerce

**Core features**
- Multi-organisation, multi-website, multi-language out of the box
- Corporate account hierarchies with buyer roles and permissions
- Workflows engine (purchase approvals, RFQ-to-order)
- Price lists with rules, schedules, and currencies
- Built-in CRM (OroCRM) and quote management
- Punchout (Level 1 & 2) and EDI through partners
- Open-source community edition (OSL-3.0)

**Differentiating features**
- Purpose-built for B2B from day one — not retrofitted from B2C
- Workflow engine handles complex approval chains natively
- Strong vendor neutrality / on-premise option

**UX patterns**
- Back-office UX shared with OroCRM and OroPlatform
- Storefront supports configurable org-tree navigation

**Integration points**
- Symfony-based PHP framework, extensible via bundles
- REST and GraphQL APIs
- Akeneo, ERP, and PIM connectors

**Known gaps**
- Storefront UX is older; many adopters go headless
- Smaller community than Magento

**Licence / IP notes**
- Open Software Licence 3.0 (community); commercial Enterprise edition

### Adobe Commerce (Magento)

**Core features**
- Customer segments, shared catalogues, tiered pricing, quote management (Adobe Commerce)
- Requisition lists, quick order, company accounts with hierarchies
- Negotiable quotes between buyer and seller
- Page Builder, B2B-aware promotions, gift card / store credit

**Differentiating features**
- Mature enterprise platform with deep customisation
- Adobe Experience Cloud integration (Target, Analytics, AEM)
- Magento Open Source community

**UX patterns**
- Luma / Hyvä / PWA Studio storefronts; classic Magento admin

**Integration points**
- REST, GraphQL, SOAP APIs; message queues (RabbitMQ); Adobe IO Events
- Massive extension marketplace

**Known gaps**
- Operational complexity / hosting cost
- Performance tuning required at scale
- Adobe focus shifting toward composable Edge Delivery Services

**Licence / IP notes**
- OSL-3.0 (Magento Open Source); commercial Adobe Commerce subscription
- Some B2B features only in Adobe Commerce, not Magento OS

### Salesforce Commerce Cloud B2B

**Core features**
- Account hierarchies and contact roles
- Entitlements-driven catalogues and price books
- Quote-to-cash via Salesforce CPQ + Commerce
- Reorder portal, contract pricing, approval flows
- Composable storefronts via PWA Kit / LWR

**Differentiating features**
- Native integration with Salesforce CRM, Service Cloud, Data Cloud
- Einstein AI for recommendations and search
- B2B & B2C unified data model (B2B2C Commerce)

**UX patterns**
- Lightning-style admin; LWR storefront components

**Integration points**
- Connect REST APIs; SCAPI; MuleSoft for ERP
- AppExchange ecosystem

**Known gaps**
- Cost; complex licensing (per-org, per-user, GMV)
- Heavyweight for SMB

**Licence / IP notes**
- Proprietary; some patents around CPQ pricing engines

### SAP Commerce Cloud (Hybris)

**Core features**
- B2B Accelerator with org hierarchy, cost centres, budgets
- Multi-catalogue, multi-price-list, contract pricing
- Approval workflows tied to budgets/cost centres
- Punchout (cXML, OCI), EDI integration
- Spartacus / Composable Storefront (Angular, MIT licensed)

**Differentiating features**
- Tight S/4HANA, ECC, and Ariba integration
- Cost-centre / budget tracking that mirrors enterprise procurement

**UX patterns**
- Backoffice (ZK-based) admin; Spartacus storefront

**Integration points**
- OCC REST API; SAP Cloud Platform Integration; OData
- SAP Customer Data Cloud, Sales Cloud

**Known gaps**
- Long implementation timelines
- High TCO

**Licence / IP notes**
- Proprietary; Spartacus storefront is Apache-2.0

### commercetools

**Closer to MACH-native B2B**
- Business Units, Associate Roles, B2B carts and quotes APIs
- Customer Groups, channels, and product selections
- Price tiers per customer group / channel
- Composable: bring your own storefront

**Differentiating features**
- API-first, headless from day one; MACH Alliance founder
- Multi-tenant cloud with global edge
- Strong evidence in MACH adoption (B2B brands like Audi, Lego B2B, John Deere)

**UX patterns**
- No prescriptive UX; Merchant Center for back office; storefront BYO (Next.js, Remix, etc.)

**Integration points**
- REST and GraphQL APIs; webhooks; subscriptions to Pub/Sub
- Frontend SDKs for TypeScript, Java, .NET, PHP

**Known gaps**
- No out-of-the-box storefront; build/buy decision required
- Pricing per API call can surprise at scale

**Licence / IP notes**
- Proprietary SaaS; SDKs MIT-licensed

### Spryker

**Core features**
- Modular packages for B2B, B2C, marketplace, IoT commerce
- Company accounts, business unit hierarchies, approval rules
- Shopping lists, quick order, RFQ, punchout
- Headless via Glue API; Yves storefront optional

**Differentiating features**
- Modular feature catalogue (1,000+ modules) selected per project
- Built-in marketplace capability for B2B distributors

**UX patterns**
- Backoffice based on Zed (Symfony); composable storefront

**Integration points**
- REST Glue APIs; PHP module ecosystem
- Pre-built ERP/PIM connectors

**Known gaps**
- Steep learning curve; module licensing complexity

**Licence / IP notes**
- Proprietary commercial licence; SDK pieces MIT

### RepSpark

**Core features**
- Line sheets, lookbook-style ordering for fashion/CPG
- Sales rep mobile app for in-field ordering
- Bulk order grids by size/colour
- ATS (available-to-sell) and pre-book / at-once orders
- Integration with ERP/3PL (NetSuite, Full Circle, AIMS360)

**Differentiating features**
- Vertical specialisation in apparel/footwear/accessories
- Strong sales-rep-led workflows

**Known gaps**
- Limited beyond fashion/CPG verticals
- Less developer extensibility

**Licence / IP notes**
- Proprietary SaaS

### WizCommerce

**Core features**
- Distributor-focused order taking, in-person and online
- AI-assisted product search and order entry
- Customer-specific pricing and credit
- Mobile field-rep app with offline mode
- ERP connectors (QuickBooks, NetSuite, SAP B1)

**Differentiating features**
- Trade-show & in-person ordering modes
- Marketed AI features for product discovery

**Known gaps**
- Mid-market only; limited at enterprise complexity

**Licence / IP notes**
- Proprietary SaaS

### Handshake (Shopify)

**Core features**
- Curated wholesale marketplace; buyers discover Shopify-hosted brands
- Free for both sides; Shopify monetises via merchant subscriptions

**Differentiating features**
- Marketplace dynamic rather than per-merchant storefront

**Known gaps**
- Limited platform features beyond discovery

**Licence / IP notes**
- Proprietary

### Pepperi

**Core features**
- Field sales, B2B e-commerce, DSD (direct store delivery)
- Catalogue, price lists, promotions, retail execution
- Configurable forms and surveys for field reps
- Offline-first mobile app

**Differentiating features**
- Combined B2B commerce + retail execution + DSD
- Strong CPG / consumer goods fit

**Licence / IP notes**
- Proprietary SaaS

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Account / company entities with multiple buyers and configurable roles
- Customer-group price lists with quantity breaks and contract pricing
- Custom / restricted catalogues per account or segment
- Quick reorder, saved lists, requisition lists
- Order minimums, case-pack and multiple-of rules
- Payment terms (Net X), purchase order method, credit limit checks
- Sales rep "order on behalf of" / impersonation
- Quote / RFQ workflow
- Approval workflows for orders above thresholds
- Headless API access (REST and/or GraphQL)
- ERP and tax-engine integration hooks (Avalara, Vertex, NetSuite, SAP)

### Differentiating Features
- Native punchout (cXML, OCI) and EDI for procurement integration
- Multi-org hierarchies with cost centres and budgets
- Quote negotiation with audit trail and counter-offers
- Channel/region-aware pricing and inventory
- Built-in workflow / business-process engine
- Marketplace-style multi-vendor capability
- Embedded CPQ for configurable / engineered products
- Offline-capable field-rep mobile apps
- Vertical-specific UX (line sheets for fashion, grids for distributors)

### Underserved Areas / Opportunities
- Conversational ordering — natural-language reorder, "send the usual order plus 20% on SKUs trending up"
- Document / PDF order ingestion — extract line items from emailed POs and faxed forms into structured orders
- AI-assisted price-list maintenance — detect drift between contract terms and active price lists
- Procurement-side bridges — punchout/EDI is fragmented, expensive, and rarely turn-key for SMB sellers
- Credit risk and dynamic terms — most platforms enforce static credit limits; real-time risk scoring is rare
- Buyer-side AI agents purchasing autonomously — emerging requirement few platforms address
- Translation of complex ERP pricing rules into commerce price lists without manual mapping

### AI-Augmentation Candidates
- Reorder prediction from purchase history and seasonality
- Anomaly detection on order patterns (fraud, mistakes, demand shifts)
- Automated parsing of inbound POs (PDF/email/EDI 850) into platform orders
- Natural-language search and product discovery for long catalogues
- AI-generated negotiation drafts and quote responses for sales reps
- Auto-classification of new SKUs into customer-specific catalogues
- Translation of ERP pricing logic to platform price lists
- Buyer-facing copilot that explains contract terms, applicable discounts, lead times

## Legal & IP Summary

The B2B commerce space is dominated by proprietary SaaS platforms (Shopify, BigCommerce, Salesforce, SAP, commercetools, Spryker, RepSpark, WizCommerce, Pepperi, Handshake). Two notable open-source-licensed entries exist: OroCommerce (OSL-3.0, with a commercial Enterprise tier) and Magento Open Source (OSL-3.0; many B2B features however are gated to the commercial Adobe Commerce edition). SAP's Spartacus storefront is Apache-2.0 and freely usable independently of SAP Commerce. OSL-3.0 is a copyleft licence and is incompatible with the GPL family and with permissive distribution; an AI-native B2B project should not bundle OSL code into a permissively-licensed core. No specific patents were identified in this scan as blockers, but CPQ pricing engines (Salesforce, Oracle, PROS) and EDI translation layers have known patent estates and should be reviewed before reimplementing equivalent algorithms. Trademarks (Shopify, BigCommerce, Adobe Commerce, etc.) must not be used in branding. EDI X12 specifications are owned by ASC X12; subscription is required for reproduction of the standards documents, though implementation of the protocol itself is unencumbered.

## Recommended Feature Scope

**Must-have (MVP)**
- Company / account model with multiple buyers and role-based permissions
- Customer-group price lists with fixed prices and quantity breaks
- Custom catalogues restricted by customer group
- Quick reorder and saved order lists
- Order minimums and case-pack rules
- Net-terms and purchase-order payment methods (with manual A/R reconciliation)
- REST and GraphQL APIs with webhooks; headless-first storefront
- Sales rep "order on behalf of" mode

**Should-have (v1.1)**
- Approval workflows with configurable thresholds and chains
- Quote / RFQ workflow with negotiation history
- AI-powered reorder prediction and natural-language ordering
- PDF / email purchase-order ingestion into draft orders
- Integration adapters for NetSuite, QuickBooks, and a generic CSV ERP bridge
- Avalara / Vertex tax integration

**Nice-to-have (backlog)**
- Punchout (cXML, OCI) and EDI 850/810/856 connectors
- Multi-org hierarchies with cost centres and budgets
- Real-time credit-risk scoring and dynamic terms
- Buyer-side AI agent / copilot for catalogue navigation
- Marketplace / multi-vendor mode
- Vertical-specific UX templates (fashion line sheets, distributor grids)
- Offline-first field-sales mobile app
