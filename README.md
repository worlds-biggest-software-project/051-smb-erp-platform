# SMB ERP Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source ERP for small and medium businesses — purchasing, inventory, finance, and HR in one system, configured by conversation rather than consulting hours.

The SMB ERP Platform is a core ERP for businesses in the $5M–$50M revenue range that today are stuck choosing between expensive seat-based SaaS (NetSuite, Business Central, SAP Business One) and feature-light open-source alternatives without modern AI. It targets CFOs, controllers, and operations leaders replacing QuickBooks, spreadsheets, or first-generation ERPs, and aims to collapse the multi-week setup and per-user pricing model that causes 40–60% of SMB ERP projects to fail.

---

## Why SMB ERP Platform?

- **The two open-source ERPs that exist have a known AI gap.** ERPNext is genuinely MIT-licensed but has no AI features and a roadmap that does not indicate AI investment. Dolibarr stops scaling beyond ~50 users. There is no open-source AI-native option for SMBs that have outgrown them.
- **Commercial AI ERP costs $50K–$250K+/year.** Business Central runs $70–$100/user/month, NetSuite $40K–$150K+/year, SAP Business One $95–$150/user/month with $50K–$200K implementations. AI features (Copilot, agentic automation) are gated behind those tiers.
- **Per-user licensing punishes operational coverage.** Warehouse staff, field reps, and approvers all need access; per-user pricing makes broad rollout cost-prohibitive. Acumatica's consumption-based pricing solves this commercially but is opaque and proprietary.
- **Setup is the failure point, not features.** SMBs lose weeks configuring chart of accounts, tax rules, and workflows; 40–60% of SMB ERP projects fail at this stage (Panorama Consulting, 2025).
- **Open-source ERP adoption grew 32% globally in 2025–2026** (Tirnav, 2026), driven by rising proprietary licence costs — the demand signal for an MIT-licensed AI-native alternative is already in the data.

---

## Key Features

### Core Financial Accounting

- Double-entry accounting: chart of accounts, general ledger, accounts payable, accounts receivable
- Bank reconciliation and period closing
- Standard financial reports: P&L, balance sheet, cash flow statement
- Role-based access controls with full audit trail
- Multi-currency support for international transactions

### Inventory and Purchasing

- Stock movements, purchase orders, sales orders, and goods receipt
- Vendor management and purchase requisition workflow
- Mobile interface for warehouse and field-staff capture
- Open API for eCommerce and third-party inventory integrations

### Conversational ERP

- Natural-language transaction creation for common operations — PO entry, invoice entry, expense submission — without navigating screen hierarchies
- Plain-language commands map to full workflows (e.g. "create a PO for 500 units of SKU-123 from Acme at $12 each, net-30")
- Designed to remove ERP-specialist training as a prerequisite for SMB staff

### AI-Assisted Close and Cash Flow

- AI bank reconciliation with confidence scores and explainable match reasoning for unmatched items
- Proactive 30/60/90-day cash flow forecasts driven by AP/AR aging and payment patterns
- Early-warning alerts on emerging shortfalls and trigger conditions for early-payment discounts

### Zero-Configuration Onboarding

- Auto-configuration engine infers chart of accounts, tax rules, and approval workflows from a business description, bank statements, and industry type
- Industry-specific configuration templates (planned) for professional services, retail, and light manufacturing
- Goal: collapse the multi-week setup that drives most SMB ERP project failures

---

## AI-Native Advantage

Existing SMB ERPs treat AI as an add-on to legacy data-entry workflows: Odoo 19 added OCR invoice scanning and ML demand forecasting, Business Central Copilot does bank-reconciliation auto-matching, and Odoo 20 (September 2026) is introducing an agentic layer — but all of these sit on top of screen-driven UIs and per-user pricing. This project inverts that: natural-language transaction creation, AI-driven reconciliation with explainable confidence, proactive cash flow intelligence, and inferred configuration are first-class workflows, not add-ons. The result is meaningful for cash-constrained SMBs that cannot justify a dedicated controller or a $50K implementation partner.

---

## Tech Stack & Deployment

Self-hosted-first, with managed cloud as a deployment option (mirroring the ERPNext / Frappe Cloud model where per-site pricing replaces per-user pricing). Open API surface for accounting, inventory, and HR enables integration with payroll, eCommerce, CRM, and BI tools. Relevant standards include GAAP and IFRS for financial reporting, ISO 20022 for bank messaging, PSD2 / Open Banking for bank feeds, EDI (EDIFACT / X12) for trading partner exchange, GS1 / GTIN for product identification, and XBRL for regulatory filings. GDPR and CCPA compliance applies to employee and customer personal data.

---

## Market Context

The global ERP market is ~$72.6B in 2025 and projected to reach $225.4B by 2035 at ~12% CAGR (Fortune Business Insights, 2025); the SMB software segment is ~$77.3B in 2026 with SMEs the highest-growth sub-segment at 11.7% CAGR (Business Research Insights, 2026). Incumbent annual platform costs span $0–$5K (self-hosted open source), $25K–$100K (Odoo Enterprise, Acumatica), and $50K–$250K+ (Business Central, NetSuite, SAP Business One). Primary buyers are CFOs and controllers at $5M–$50M businesses replacing QuickBooks, IT and operations leaders at SMB manufacturers and distributors, and founders of fast-growing services companies needing multi-currency and revenue recognition.

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

Licence to be determined. Given the goal of serving SMBs currently priced out of commercial AI ERP, an MIT-style permissive licence (matching ERPNext) is the most likely direction; GPL v3 (matching Odoo Community / Dolibarr) is the alternative. See [discussion](#) for context.
