# Standards & API Reference

> Project: SMB ERP Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001 — Information Security Management Systems**
- URL: https://www.iso.org/isoiec-27001-information-security.html
- The primary international standard for ISMS. More than 65% of ERP clients now require ISO 27001 certification from vendors as a contractual prerequisite. Any cloud-hosted SMB ERP platform will need to demonstrate conformance to attract mid-market customers.

**ISO/IEC 27018 — Protection of PII in Public Cloud**
- URL: https://www.iso.org/standard/76559.html
- Extends ISO 27001 with specific controls for Personally Identifiable Information in cloud environments. Directly relevant to multi-tenant SaaS ERP deployments that store employee, payroll, and customer data.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- The global quality management standard that many SMB customers are required to adhere to. ERP platforms often must support audit trails and document control workflows aligned with ISO 9001 requirements. A 2026 revision is in progress emphasising risk-based thinking and digital data management.

**ISO 20022 — Financial Data Messaging Standard**
- URL: https://www.iso20022.org/
- The global standard for electronic data interchange between financial institutions. The US Federal Reserve adopted full ISO 20022 for domestic wires in 2025. SMB ERP platforms that process bank payments or integrate with treasury systems must produce ISO 20022-compliant messages.

**ISO/IEC 19845 — Universal Business Language (UBL)**
- URL: https://www.iso.org/standard/66370.html
- The ISO ratification of OASIS UBL. UBL is an open library of XML business document schemas (purchase orders, invoices, despatch advice, etc.) that are widely used in e-procurement, particularly in Europe and Asia-Pacific. SMB ERPs that sell into public sector supply chains must support UBL.

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP methods (GET, POST, PUT, DELETE, PATCH), status codes, and content negotiation. Forms the foundation for every REST API exposed by or consumed by an ERP platform.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization framework used by every major ERP and accounting SaaS. Required for third-party integrations, marketplace apps, and user SSO flows.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Specifies how OAuth 2.0 bearer tokens are transmitted in API requests. Directly applicable to the platform's REST API authentication layer.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- An identity layer built on OAuth 2.0. Required for federated SSO with identity providers such as Azure AD, Google Workspace, and Okta — standard expectations in mid-market ERP buyers.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- The token format used in OAuth 2.0 and OpenID Connect flows. ERP API sessions and service-to-service calls typically use JWTs for stateless authentication.

**SAML 2.0 (OASIS)**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- Still the dominant SSO protocol in enterprise and legacy ERP environments (SAP, Oracle). The platform should support SAML 2.0 SP-initiated flows for customers with existing SAML identity providers.

### Data Model & API Specifications

**OData v4 (OASIS/ISO/IEC 20802)**
- URL: https://www.odata.org/ · OASIS Spec: https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html
- An ISO/IEC-approved OASIS standard for building queryable, interoperable RESTful APIs. OData is heavily used in enterprise ERP integrations (SAP Business One Service Layer, Microsoft Dynamics 365, SuccessFactors). Supporting OData v4 dramatically reduces integration friction for customers coming from Microsoft ecosystems.

**OpenAPI Specification 3.1 (formerly Swagger)**
- URL: https://spec.openapis.org/oas/v3.1.0.html · Initiative: https://www.openapis.org/
- The industry-standard format for describing HTTP APIs in a machine-readable, language-agnostic way. All modern ERP API documentation (NetSuite, Business Central, Sage Intacct) publishes OpenAPI 3.x specs. The platform's public API should ship with a conformant OpenAPI 3.1 document from day one.

**JSON:API v1.1**
- URL: https://jsonapi.org/format/
- A specification for building JSON-based APIs with consistent document structure, relationship handling, sparse fieldsets, and pagination. Reduces client/server coupling and enables generic tooling and caching.

**OASIS UBL 2.1**
- URL: https://docs.oasis-open.org/ubl/UBL-2.1.html
- XML schemas for common supply chain documents: orders, invoices, despatch advice, credit notes. Foundation for PEPPOL e-invoicing. Required for European public procurement and increasingly for global B2B trade.

**ANSI X12 (EDI)**
- URL: https://x12.org/
- The dominant North American EDI standard. Key transaction sets for ERP: 850 (Purchase Order), 810 (Invoice), 856 (Advance Ship Notice), 820 (Payment Order). SMB ERPs that serve manufacturing, distribution, or retail customers need EDI X12 connectivity.

**UN/EDIFACT**
- URL: https://unece.org/trade/uncefact/introducing-unedifact
- The equivalent global (UN) EDI standard, dominant in Europe and Asia. Required for SMBs trading internationally with European partners.

**XBRL (Extensible Business Reporting Language)**
- URL: https://xbrl.us/ · SEC guide: https://www.sec.gov/files/edgar/filer-information/specifications/xbrl-guide-2024-07-08.pdf
- The global standard for machine-readable financial reporting. Mandatory for US public-company SEC filings (Inline XBRL). SMBs growing toward IPO or dealing with public-company customers/suppliers benefit from XBRL-export capability in their ERP.

### Security & Authentication Standards

**ISO/IEC 27001 & SOC 2 Type II**
- URL (SOC 2): https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/aicpasoc2report.html
- The primary security certifications cloud SaaS buyers require. SOC 2 Type II evaluates an organisation's security controls over 6–12 months; ISO 27001 covers the same ground but as an internationally recognised standard. There is ~80% overlap between the two. The platform should target both certifications within 18 months of launch.

**GDPR (EU General Data Protection Regulation)**
- URL: https://gdpr.eu/
- Mandatory for any ERP platform serving EU-based customers. Impacts data residency, right-to-erasure, consent management, and data processor agreements. The platform architecture must support per-tenant data isolation and documented data flows.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-ten/
- The definitive checklist of web application security risks. ERP platforms are high-value targets (financial data, employee records, supply chain). Development must apply OWASP controls around injection, broken authentication, IDOR, and insecure direct object references.

**PEPPOL (Pan-European Public Procurement OnLine)**
- URL: https://peppol.eu/ · BIS Billing 3.0: https://docs.peppol.eu/poacc/billing/3.0/bis/
- The European e-invoicing network with 2.5M+ registered participants in 111 countries (as of 2026). Uses the eDelivery AS4 transport protocol and UBL document format. Required for selling to EU public sector and increasingly mandated for B2B invoicing across EU member states. Compliance with EU Directive 2014/55/EU.

**IFRS & US GAAP**
- URL (IFRS): https://www.ifrs.org/ · URL (GAAP/FASB): https://fasb.org/
- The two dominant financial reporting frameworks globally. IFRS is principles-based and required in 140+ countries; US GAAP is rules-based and mandatory for US public companies. An SMB ERP must support multi-GAAP parallel ledgers (especially for lease accounting under ASC 842 / IFRS 16 and revenue recognition under ASC 606 / IFRS 15).

---

## Similar Products — Developer Documentation & APIs

### Oracle NetSuite
- **Description:** The dominant cloud ERP for growth-stage and mid-market SMBs. Comprehensive financials, inventory, CRM, and e-commerce modules in a single platform.
- **API Documentation:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/book_1559132836.html
- **REST API Browser:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_157373386674.html
- **SDKs/Libraries:** SuiteScript (server-side JS), SuiteTalk SOAP SDK, REST Records (OpenAPI 3.0 metadata)
- **Developer Guide:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/preface_1531238762.html
- **Standards:** REST/JSON, OpenAPI 3.0, SOAP, SuiteQL (SQL-style query language)
- **Authentication:** OAuth 2.0 (Token-Based Authentication), TBA

### Microsoft Dynamics 365 Business Central
- **Description:** Microsoft's SMB ERP offering, tightly integrated with Microsoft 365 and Azure. Strong in manufacturing, distribution, and professional services verticals.
- **API Documentation:** https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/api-reference/v2.0/
- **SDKs/Libraries:** AL language (Business Central), Power Platform connectors, .NET/C# libraries
- **Developer Guide:** https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/webservices/api-overview
- **Custom API Guide:** https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-develop-custom-api
- **Standards:** REST/JSON, OData v4, OpenAPI 3.0, Microsoft Graph integration
- **Authentication:** OAuth 2.0 (Azure AD / Entra ID), API Key (on-premises)

### SAP Business One
- **Description:** SAP's ERP for small businesses (10–500 employees). Strong manufacturing, inventory, and multi-currency capabilities with a large global partner network.
- **API Documentation:** https://help.sap.com/docs/SAP_BUSINESS_ONE_VERSION_FOR_SAP_HANA/686100cb1bc34346b2bc6642685bab43/b1bbebd32ff940c786c76315a8dfa270.html
- **Service Layer API Reference:** https://help.sap.com/doc/056f69366b5345a386bb8149f1700c19/10.0/en-US/Service%20Layer%20API%20Reference.html
- **SDKs/Libraries:** DI API (COM-based), Service Layer (REST), B1if (integration framework)
- **Developer Guide:** https://apiworx.com/diy-developer-guide-building-custom-integrations-for-sap-business-one/
- **Standards:** REST/JSON, OData v4 (OData v3 deprecated in FP 2405), SOAP
- **Authentication:** OAuth 2.0 (Service Layer), session cookies

### Odoo
- **Description:** Open-source ERP/CRM with a large app marketplace. Popular with tech-savvy SMBs due to its flexibility, self-hosting option, and modular pricing.
- **API Documentation:** https://www.odoo.com/documentation/18.0/developer/reference/external_api.html
- **JSON-2 API (v19+):** https://www.odoo.com/documentation/19.0/developer/reference/external_api.html
- **SDKs/Libraries:** Official Python, JavaScript clients; community libs for PHP, Ruby, Go
- **Developer Guide:** https://www.odoo.com/documentation/18.0/developer/howtos/web_services.html
- **Standards:** XML-RPC, JSON-RPC, REST controllers (Odoo 17+); migrating to JSON-2 API (XML-RPC and JSON-RPC deprecated in Odoo 22, 2028)
- **Authentication:** API key, session token

### QuickBooks Online (Intuit)
- **Description:** Dominant accounting/SMB finance platform with a massive ecosystem. Primary target for micro and small business (<50 employees).
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **API Reference:** https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/account
- **SDKs/Libraries:** Java SDK (github.com/intuit/QuickBooks-V3-Java-SDK), PHP SDK v4, Python (unofficial)
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/get-started
- **Standards:** REST/JSON, OpenAPI, Webhooks
- **Authentication:** OAuth 2.0 (OpenID Connect); Core API calls (writes) free; CorePlus (reads) metered since 2025

### Xero
- **Description:** Cloud accounting platform strong in UK, Australia, and New Zealand. Competes with QuickBooks Online in English-speaking markets.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **OAuth 2.0 Guide:** https://developer.xero.com/documentation/guides/oauth2/overview/
- **SDKs/Libraries:** Official SDKs for .NET, Java, Node.js, PHP, Python, Ruby
- **Developer Guide:** https://developer.xero.com/
- **Standards:** REST/JSON, XML (some endpoints), Webhooks; six distinct API domains (Accounting, Assets, Files, Projects, Payroll, Bankfeeds)
- **Authentication:** OAuth 2.0 (PKCE); OAuth 1.0a deprecated; rate limits: 60 req/min per tenant, 5,000/day per tenant

### Zoho Books
- **Description:** Cloud accounting platform from Zoho's ecosystem. Strong in India and Southeast Asia. Part of the broader Zoho One suite covering CRM, HR, and project management.
- **API Documentation:** https://www.zoho.com/books/api/v3/introduction/
- **OAuth Guide:** https://www.zoho.com/books/api/v3/oauth/
- **SDKs/Libraries:** Zoho SDK for Java, Python, PHP (shared across Zoho platform)
- **Developer Guide:** https://www.zoho.com/developer/rest-api.html
- **Standards:** REST/JSON, OpenAPI; multi-org support via `organization_id` parameter
- **Authentication:** OAuth 2.0; rate limit: 100 requests/minute per organisation

### Sage Intacct
- **Description:** Cloud ERP for mid-market companies ($10M–$500M revenue). Strong in multi-entity, multi-currency, and professional services verticals.
- **API Documentation:** https://developer.sage.com/intacct/docs/
- **XML API Reference:** https://developer.intacct.com/api/
- **OpenAPI Spec:** https://developer.sage.com/intacct/apis/intacct/1/intacct-openapi
- **SDKs/Libraries:** Official PHP SDK, Python SDK, .NET SDK
- **Developer Guide:** https://developer.intacct.com/
- **Standards:** REST (GA since 2025 R1), XML/SOAP (legacy, still documented); OpenAPI 3.0
- **Authentication:** OAuth 2.0 (REST); sender ID + user credentials (legacy XML)

### Stripe (Payments)
- **Description:** The market-leading payments API. Relevant for ERP platforms implementing customer billing, subscriptions, and AR automation.
- **API Documentation:** https://docs.stripe.com/api
- **Webhooks Guide:** https://docs.stripe.com/webhooks
- **SDKs/Libraries:** Official SDKs for Node.js, Python, Ruby, PHP, Java, Go, .NET
- **Developer Guide:** https://docs.stripe.com/
- **Standards:** REST/JSON, form-encoded requests, Webhook signatures (HMAC-SHA256)
- **Authentication:** API Key (Bearer); up to 16 webhook endpoints

### Plaid (Open Banking / Bank Connectivity)
- **Description:** The leading API for connecting to bank accounts. Used in ERP/accounting platforms to automate bank feed imports, cash flow analysis, and payment verification.
- **API Documentation:** https://plaid.com/docs/api/
- **SDKs/Libraries:** Official SDKs for Node.js, Python, Ruby, Java, Go
- **Developer Guide:** https://plaid.com/docs/
- **Standards:** REST/JSON, OpenAPI; FDX-aligned API specification; covers 8,000+ financial institutions
- **Authentication:** client_id + secret (API key); OAuth 2.0 for end-user bank account linking

---

## Notes

### Emerging & Evolving Standards

- **AI / MCP Integration:** The Model Context Protocol (MCP) is an emerging open standard (Anthropic, 2024) for connecting AI agents to data sources and tools. ERP platforms that expose MCP servers would allow AI copilots to query financial data, create purchase orders, and generate reports without bespoke integration code. Early-mover advantage is significant.

- **FDX (Financial Data Exchange):** The US equivalent of open banking. Plaid and major US banks are converging on the FDX API standard for permissioned financial data sharing. SMB ERPs with bank reconciliation features should track FDX adoption.

- **e-Invoicing Mandates:** EU member states are progressively mandating B2B e-invoicing (France: 2026, Germany: 2025). PEPPOL compliance will shift from optional to required across most of Europe within the platform's relevant horizon.

- **ISO 20022 Full Adoption:** The US Federal Reserve completed ISO 20022 migration for Fedwire in 2025. SMB ERP platforms that produce payment files (ACH, wire) need ISO 20022-compliant message generation.

- **XBRL Inline Expansion:** SEC continues expanding XBRL Inline filing mandates. SMBs approaching public markets or working with PE-backed portfolios increasingly need structured financial report exports.
