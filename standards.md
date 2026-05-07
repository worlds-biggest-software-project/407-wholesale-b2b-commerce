# Standards & API Reference

> Project: Wholesale & B2B Commerce · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO 8601 — Date and time format**
  https://www.iso.org/iso-8601-date-and-time-format.html
  Universally used for order dates, delivery windows, and audit timestamps in B2B APIs.

- **ISO 4217 — Currency codes**
  https://www.iso.org/iso-4217-currency-codes.html
  Required for multi-currency price lists and invoices.

- **ISO 3166-1 / 3166-2 — Country and subdivision codes**
  https://www.iso.org/iso-3166-country-codes.html
  Used in shipping, tax, and account address normalisation.

- **ISO 6166 / GS1 GTIN — Global Trade Item Number**
  https://www.gs1.org/standards/id-keys/gtin
  Core product identifier for B2B catalogues; underpins barcode and EDI item identification.

- **ISO/IEC 27001:2022 — Information security management systems**
  https://www.iso.org/standard/27001
  Common procurement requirement for enterprise B2B vendors.

- **ISO/IEC 27017 / 27018 — Cloud security and PII protection**
  https://www.iso.org/standard/43757.html
  Frequently requested by enterprise buyers in vendor-security questionnaires.

- **ISO 20022 — Financial-services messaging**
  https://www.iso20022.org/
  Increasing relevance for B2B payment instructions, remittance, and account-to-account flows.

### W3C & IETF Standards

- **RFC 9110 / 9111 / 9112 — HTTP Semantics, Caching, HTTP/1.1**
  https://www.rfc-editor.org/rfc/rfc9110
  Basis for all REST APIs.

- **RFC 8259 — JSON Data Interchange Format**
  https://www.rfc-editor.org/rfc/rfc8259
  Default payload format for modern commerce APIs.

- **RFC 6749 / 6750 — OAuth 2.0 and Bearer Tokens**
  https://www.rfc-editor.org/rfc/rfc6749
  Standard authorization framework for buyer and partner integrations.

- **RFC 9068 — JWT Profile for OAuth 2.0 Access Tokens**
  https://www.rfc-editor.org/rfc/rfc9068
  Common token format used by commerce IdPs.

- **RFC 8288 — Web Linking**
  https://www.rfc-editor.org/rfc/rfc8288
  Pagination and HATEOAS conventions in REST commerce APIs.

- **RFC 7519 — JSON Web Token (JWT)**
  https://www.rfc-editor.org/rfc/rfc7519
  Identity propagation between storefront, BFF, and commerce backend.

- **RFC 5321 / 5322 — SMTP and Internet Message Format**
  https://www.rfc-editor.org/rfc/rfc5322
  Order confirmations, PO acknowledgements, and email-based PO ingestion.

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines**
  https://www.w3.org/TR/WCAG22/
  Procurement RFPs increasingly require AA conformance for buyer-facing interfaces.

- **W3C ActivityStreams 2.0 / Webhooks (WICG webhooks-discovery)**
  https://www.w3.org/TR/activitystreams-core/
  Inspiration for event payload shapes; many commerce platforms use webhook conventions aligned to CloudEvents.

- **CloudEvents 1.0 (CNCF)**
  https://cloudevents.io/
  Becoming a de-facto standard for commerce event payloads (OrderPlaced, OrderShipped).

### Data Model & API Specifications

- **OpenAPI Specification 3.1**
  https://spec.openapis.org/oas/v3.1.0
  De-facto standard for documenting REST commerce APIs.

- **GraphQL Specification (October 2021)**
  https://spec.graphql.org/October2021/
  Used by Shopify Storefront/Admin, BigCommerce, commercetools, and Salesforce Commerce.

- **JSON Schema 2020-12**
  https://json-schema.org/draft/2020-12
  Validation of catalogue, price-list, and order payloads.

- **AsyncAPI 3.0**
  https://www.asyncapi.com/
  Documentation of webhook / event streams alongside OpenAPI.

- **Schema.org Product / Offer / Order**
  https://schema.org/Product
  SEO and structured-data baseline; many commerce platforms emit JSON-LD using these types.

- **GS1 EDI / GS1 XML**
  https://www.gs1.org/standards/edi
  Trade-document standards built atop GTIN/GLN identifiers; widely used for grocery and CPG B2B.

- **Open Applications Group OAGIS 10**
  https://oagi.org/oagis-10/
  Canonical XML/JSON business object documents (BODs) for commerce/ERP integration.

### B2B Procurement & Trade Standards

- **ANSI ASC X12 EDI** (notably 850 Purchase Order, 855 PO Ack, 856 ASN, 810 Invoice, 832 Price/Sales Catalog)
  https://x12.org/
  North American EDI standard; foundational to enterprise B2B procurement.

- **UN/EDIFACT** (ORDERS, ORDRSP, DESADV, INVOIC, PRICAT)
  https://unece.org/trade/uncefact/introducing-unedifact
  International EDI standard, widely used outside North America.

- **cXML (Commerce XML)**
  https://cxml.org/
  Ariba-originated XML protocol; powers punchout, OrderRequest, and ConfirmationRequest flows.

- **OCI (Open Catalog Interface) 5.0**
  https://help.sap.com/docs/SAP_ERP/c4f3d71d70d44e3a8b4a9b1d37eb3a89/9b1c8fb7378b4b4d9c0c7a3a8d4d8f47.html
  SAP-originated punchout standard used by SAP SRM and Coupa.

- **PEPPOL BIS 3.0 (e-invoicing)**
  https://docs.peppol.eu/poacc/billing/3.0/
  Standardised European e-invoicing and procurement network; mandatory in many EU public-procurement contexts.

- **UBL 2.4 (Universal Business Language)**
  https://docs.oasis-open.org/ubl/UBL-2.4.html
  XML library covering Order, Invoice, Catalogue, Despatch Advice, etc. Underlies PEPPOL.

- **Factur-X / ZUGFeRD**
  https://www.ferd-net.de/standards/zugferd
  Hybrid PDF/A-3 + XML invoice format mandated for B2B in France (2026 rollout).

- **CII (Cross Industry Invoice, UN/CEFACT)**
  https://unece.org/trade/uncefact/xml-schemas
  XML schemas for trade documents used inside Factur-X and PEPPOL profiles.

### Security & Authentication Standards

- **OAuth 2.1 (Draft)**
  https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
  Consolidated OAuth profile recommended for new commerce APIs.

- **OpenID Connect Core 1.0**
  https://openid.net/specs/openid-connect-core-1_0.html
  Buyer SSO and federated identity for company users.

- **FIDO2 / WebAuthn Level 3**
  https://www.w3.org/TR/webauthn-3/
  Phishing-resistant buyer authentication.

- **OWASP ASVS 4.0 / API Security Top 10 (2023)**
  https://owasp.org/www-project-application-security-verification-standard/
  Baseline API hardening checklist for commerce surfaces.

- **NIST SP 800-63B — Digital Identity Guidelines**
  https://pages.nist.gov/800-63-3/sp800-63b.html
  Authenticator assurance for buyer accounts.

- **PCI DSS 4.0**
  https://www.pcisecuritystandards.org/document_library/
  Required for any path that touches cardholder data.

- **SOC 2 Type II (AICPA TSC)**
  https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2
  Standard enterprise-procurement vendor-security expectation.

- **GDPR / UK GDPR / CCPA**
  https://gdpr.eu/ · https://oag.ca.gov/privacy/ccpa
  Apply to buyer contact data even in B2B contexts.

### MCP Server Specifications

- **Model Context Protocol Specification**
  https://modelcontextprotocol.io/
  Anthropic-led standard for connecting AI agents to data and tools; relevant for AI buyer copilots and agentic ordering.

- **MCP TypeScript & Python SDKs**
  https://github.com/modelcontextprotocol/typescript-sdk
  https://github.com/modelcontextprotocol/python-sdk
  Reference implementations for exposing catalogue, pricing, and order tools to AI agents.

- **Agent2Agent (A2A) Protocol**
  https://github.com/google/A2A
  Emerging standard for agent-to-agent procurement workflows; relevant as buyer-side AI agents start placing orders.

## Similar Products — Developer Documentation & APIs

### Shopify (B2B on Shopify)
- **Description:** Hosted commerce platform with native B2B features — companies, catalogues, price lists, payment terms, draft orders.
- **API Documentation:** https://shopify.dev/docs/api
- **B2B Reference:** https://shopify.dev/docs/apps/build/b2b
- **SDKs/Libraries:** JS Buy SDK, Hydrogen (Remix), Storefront API client (TS/JS), Ruby/Python admin clients, Shopify Functions (Rust/JS).
- **Developer Guide:** https://shopify.dev/docs/api/usage
- **Standards:** REST + GraphQL; OAuth 2.0; webhooks (HMAC-signed JSON).
- **Authentication:** OAuth 2.0 (online & offline tokens); API keys for private apps via Shopify CLI.

### BigCommerce
- **Description:** SaaS commerce with B2B Edition (Buyer Portal, quotes, A-R, customer groups).
- **API Documentation:** https://developer.bigcommerce.com/docs/rest-management
- **B2B Edition API:** https://developer.bigcommerce.com/b2b-edition
- **SDKs/Libraries:** Catalyst (Next.js), Stencil CLI, Node.js & PHP SDKs, GraphQL Storefront client.
- **Developer Guide:** https://developer.bigcommerce.com/docs/start
- **Standards:** REST + GraphQL Storefront; OpenAPI; OAuth 2.0; webhooks.
- **Authentication:** OAuth 2.0 (server-to-server), Storefront customer JWTs.

### Adobe Commerce / Magento
- **Description:** Enterprise/open-source commerce platform with deep B2B feature set in Adobe Commerce.
- **API Documentation:** https://developer.adobe.com/commerce/webapi/
- **B2B Reference:** https://experienceleague.adobe.com/en/docs/commerce-admin/b2b/introduction
- **SDKs/Libraries:** PWA Studio, Adobe Commerce SDK, Magento OS PHP framework, GraphQL/REST clients.
- **Developer Guide:** https://developer.adobe.com/commerce/
- **Standards:** REST, GraphQL, SOAP; AMQP message queues; Adobe IO Events (CloudEvents-aligned).
- **Authentication:** OAuth 1.0a (legacy), OAuth 2.0 via Adobe IMS, integration tokens.

### Salesforce Commerce Cloud B2B
- **Description:** Enterprise commerce on Salesforce platform with native CRM, CPQ, and Service Cloud integration.
- **API Documentation:** https://developer.salesforce.com/docs/commerce/commerce-api/
- **SDKs/Libraries:** PWA Kit (Node/React), Composable Storefront (LWR), Commerce SDK (Node), Connect REST API.
- **Developer Guide:** https://developer.salesforce.com/docs/commerce/
- **Standards:** REST (SCAPI), GraphQL (limited), OpenAPI; B2B and B2B2C data model.
- **Authentication:** SLAS (Shopper Login & API Access Service), OAuth 2.0, JWT bearer.

### SAP Commerce Cloud
- **Description:** Enterprise B2B commerce with deep S/4HANA, ERP, and Ariba integration.
- **API Documentation:** https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD
- **OCC API:** https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace
- **SDKs/Libraries:** Spartacus / Composable Storefront (Angular, Apache-2.0), JS Storefront SDK.
- **Developer Guide:** https://www.sap.com/products/crm/commerce-cloud.html
- **Standards:** OCC REST, OData; SAP CPI integration; CloudEvents-style event mesh.
- **Authentication:** OAuth 2.0; SAP IAS / customer IdP federation.

### commercetools
- **Description:** API-first MACH commerce platform with native B2B (Business Units, Associates, B2B Quote/Cart APIs).
- **API Documentation:** https://docs.commercetools.com/api
- **B2B Reference:** https://docs.commercetools.com/api/projects/business-units
- **SDKs/Libraries:** TypeScript SDK, Java, .NET, PHP, Python; Frontend SDKs; commercetools Frontend (Frontastic).
- **Developer Guide:** https://docs.commercetools.com/
- **Standards:** REST + GraphQL; OpenAPI; webhooks; subscriptions (Pub/Sub, SQS, EventGrid).
- **Authentication:** OAuth 2.0 (client credentials, password, anonymous, refresh).

### Spryker
- **Description:** Modular commerce platform supporting B2B, B2C, marketplace, and IoT scenarios.
- **API Documentation:** https://docs.spryker.com/docs/scos/dev/glue-api-guides/glue-api-tutorials.html
- **SDKs/Libraries:** Glue API (REST), PHP modules (Spryker Commerce OS), JS storefront kits.
- **Developer Guide:** https://docs.spryker.com/
- **Standards:** REST Glue API; JSON:API conventions; webhooks; events via RabbitMQ.
- **Authentication:** OAuth 2.0; JWT.

### OroCommerce
- **Description:** Open-source B2B commerce platform with built-in workflow engine and CRM.
- **API Documentation:** https://doc.oroinc.com/api/
- **SDKs/Libraries:** Symfony PHP framework, REST + GraphQL APIs, Akeneo connector.
- **Developer Guide:** https://doc.oroinc.com/
- **Standards:** REST (JSON:API), GraphQL; OpenAPI; webhooks; OAuth 2.0.
- **Authentication:** OAuth 2.0; WSSE (legacy); SSO via SAML / OIDC bundles.

### Medusa (B2B Starter)
- **Description:** Open-source headless commerce framework (MIT) with a community B2B starter.
- **API Documentation:** https://docs.medusajs.com/
- **B2B Starter:** https://github.com/medusajs/b2b-starter-medusa
- **SDKs/Libraries:** Medusa JS SDK, Next.js storefront starter.
- **Developer Guide:** https://docs.medusajs.com/learn
- **Standards:** REST APIs; OpenAPI; webhooks; modular module system.
- **Authentication:** JWT, API tokens; OAuth via providers.

### Saleor
- **Description:** Open-source GraphQL-first commerce platform (BSD-3); B2B features delivered via apps and price-list configuration.
- **API Documentation:** https://docs.saleor.io/api-reference
- **SDKs/Libraries:** Saleor App SDK (TS), Storefront SDK, Next.js storefront.
- **Developer Guide:** https://docs.saleor.io/
- **Standards:** GraphQL only; OpenID Connect; webhook subscriptions.
- **Authentication:** JWT; OAuth 2.0 / OIDC; App tokens.

### Coupa (procurement-side, target integration partner)
- **Description:** Enterprise procurement / spend-management suite — primary buyer-side counterpart for B2B sellers.
- **API Documentation:** https://developer.coupa.com/
- **Punchout Reference:** cXML 1.2.x via supplier-hosted catalogue.
- **Standards:** REST/JSON, cXML 1.2, OAuth 2.0.
- **Authentication:** OAuth 2.0 client credentials.

### SAP Ariba (procurement-side, target integration partner)
- **Description:** Largest enterprise procurement network; suppliers integrate via cXML and Ariba Network APIs.
- **API Documentation:** https://help.sap.com/docs/ariba
- **Standards:** cXML 1.2, OCI, REST APIs.
- **Authentication:** OAuth 2.0; cXML shared-secret credentials.

## Notes

- **EDI vs. modern APIs:** Most enterprise B2B integrations still depend on X12 / EDIFACT, with modernisation slow due to entrenched VAN providers and trading-partner inertia. Expect to support both EDI and modern REST/GraphQL for the foreseeable future.
- **E-invoicing mandates:** France (Factur-X 2026), Italy (FatturaPA), Poland (KSeF), Spain, and Germany (XRechnung) are progressively mandating structured e-invoicing. PEPPOL BIS 3.0 + UBL is the converging cross-border profile.
- **Punchout fragmentation:** cXML and OCI cover most enterprise punchout scenarios but require trading-partner-specific tuning; SMB sellers rarely have turn-key support — an opportunity area.
- **Agentic commerce:** MCP and emerging agent-to-agent protocols (A2A) point toward AI agents acting as buyers. Designing the API surface so an MCP server can expose catalogue/pricing/order tools cleanly is a forward-looking differentiator.
- **CloudEvents adoption:** A growing share of commerce platforms emit webhook payloads aligned to CloudEvents 1.0; standardising on that envelope is a low-cost interoperability win.
- **Standards behind paywalls:** ISO documents and ANSI X12 specifications require paid subscription. Open alternatives (UBL, PEPPOL, OAGIS, GS1 free reference profiles) cover most practical needs without licence cost.
