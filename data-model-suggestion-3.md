# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: SMB ERP Platform · Created: 2026-05-12

## Philosophy

The Hybrid Relational + JSONB model takes a pragmatic middle ground: core business fields that are common across all tenants and jurisdictions are stored as typed relational columns with full constraint enforcement, while variable, jurisdiction-specific, integration-specific, and extensible fields are stored in JSONB columns. This gives developers the query power and data integrity of a relational schema for the 80% of fields that are universal, combined with the flexibility of document storage for the 20% that vary.

This is the pattern used by Odoo (which stores custom fields in separate `ir_property` and `ir_model_fields` tables but reads them via JSONB-like mechanisms), by Shopify (which uses `metafields` JSONB for merchant-extensible data), and increasingly by modern SaaS platforms that need to support multi-jurisdiction variability without an ALTER TABLE migration per region. PostgreSQL's JSONB support is mature — it supports GIN indexing, containment queries, JSONPath expressions, and partial indexing on JSONB fields, making it viable for production query patterns.

For an AI-native SMB ERP, the hybrid approach is particularly compelling because multi-jurisdiction tax configurations, locale-specific compliance fields (e.g., Brazilian NFe fields vs. German GoBD fields vs. Indian GST fields), and per-tenant custom fields can all be stored in JSONB without requiring schema changes. This dramatically reduces the time-to-value for the zero-configuration onboarding goal: the AI configuration engine can write inferred chart-of-accounts structures and tax rules directly into JSONB fields without needing database migrations.

**Best for:** Rapid MVP development, multi-jurisdiction deployments where field requirements vary by country, and platforms that need tenant-extensible fields without per-tenant schema changes.

**Trade-offs:**
- (+) Fast iteration — adding new fields requires no migration, just a new key in JSONB
- (+) Multi-jurisdiction flexibility — locale-specific fields live in JSONB without cluttering the core schema
- (+) Tenant-extensible — custom fields are just JSONB keys, no ALTER TABLE
- (+) Fewer tables (~30) than a fully normalized model (~40+)
- (+) PostgreSQL JSONB is production-proven with GIN indexing and JSONPath
- (-) JSONB fields lack database-level type enforcement — validation shifts to application layer
- (-) JSONB queries are slower than typed column queries for complex aggregations
- (-) Schema documentation requires discipline — JSONB structures must be documented outside the DDL
- (-) Migration tooling (Flyway, Alembic) cannot track JSONB schema changes
- (-) Foreign key constraints cannot reference JSONB values

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GAAP / IFRS | Core account fields are relational; GAAP/IFRS-specific classification metadata stored in `account.metadata` JSONB |
| ISO 20022 | Structured address fields stored as relational columns (mandatory compliance); extended address fields in JSONB |
| ISO 3166-1/2 | `country_code` and `jurisdiction` as typed columns throughout |
| ISO 4217 | `currency_code` as CHAR(3) column throughout |
| GS1 / GTIN | `product.gtin` as indexed relational column; extended GS1 Application Identifier data in `product.attributes` JSONB |
| OASIS UBL 2.1 | UBL-specific document metadata (BuyerReference, OrderReference, etc.) stored in document `metadata` JSONB |
| ANSI X12 / EDIFACT | EDI mapping configuration in JSONB on trading partner records |
| XBRL | XBRL taxonomy tags in `account.metadata` JSONB |
| PEPPOL | PEPPOL identifiers as relational columns; PEPPOL-specific routing metadata in JSONB |
| GDPR | Data subject fields clearly identified; `party.metadata` JSONB can store consent records |

---

## Organisation & Multi-Tenancy

```sql
-- ============================================================
-- ORGANISATION & MULTI-TENANCY
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_plan TEXT NOT NULL DEFAULT 'free',
    -- Tenant-wide configuration in JSONB: locale, date format, number format,
    -- default tax rules, onboarding state, feature flags
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "locale": "en-US",
    --   "date_format": "MM/DD/YYYY",
    --   "number_format": {"decimal": ".", "thousands": ","},
    --   "features": {"ai_reconciliation": true, "edi_enabled": false},
    --   "onboarding": {"step": "chart_of_accounts", "completed": false}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_org_id   UUID REFERENCES organisation(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    lei             TEXT,                           -- ISO 17442
    peppol_participant_id TEXT,
    country_code    CHAR(2) NOT NULL,               -- ISO 3166-1
    jurisdiction    TEXT,                           -- ISO 3166-2
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    fiscal_year_start_month SMALLINT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Jurisdiction-specific configuration
    locale_config   JSONB NOT NULL DEFAULT '{}',
    -- Example locale_config:
    -- {
    --   "tax_system": "vat",                    -- or "sales_tax", "gst"
    --   "vat_return_frequency": "quarterly",
    --   "e_invoicing_mandate": "peppol",        -- or "sdi" (Italy), "nfe" (Brazil)
    --   "fiscal_retention_years": 7,
    --   "gobd_compliant": true,                 -- Germany-specific
    --   "gst_registration_number": "29AXXXX...", -- India-specific
    --   "default_payment_terms": "NET30",
    --   "approval_workflows": {
    --     "purchase_order": {"threshold": 5000, "approver_role": "manager"}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_tenant ON organisation(tenant_id);
CREATE INDEX idx_org_parent ON organisation(parent_org_id);

ALTER TABLE organisation ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON organisation
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

## Users & Access Control

```sql
-- ============================================================
-- USERS & RBAC
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    password_hash   TEXT,
    auth_provider   TEXT DEFAULT 'local',
    auth_provider_id TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- User preferences and profile in JSONB
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {
    --   "timezone": "America/New_York",
    --   "language": "en",
    --   "dashboard_layout": "finance",
    --   "notification_prefs": {"email": true, "push": false}
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    -- Permissions stored as a structured JSONB array
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- [
    --   {"resource": "invoice", "actions": ["create", "read", "update"]},
    --   {"resource": "purchase_order", "actions": ["create", "read", "update", "approve"]},
    --   {"resource": "journal_entry", "actions": ["read"]},
    --   {"resource": "report", "actions": ["read"], "scope": "own_department"}
    -- ]
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisation(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, org_id)
);
```

## Chart of Accounts & General Ledger

```sql
-- ============================================================
-- CHART OF ACCOUNTS & GENERAL LEDGER
-- ============================================================

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL,                  -- 'asset', 'liability', 'equity', 'revenue', 'expense'
    parent_id       UUID REFERENCES account(id),
    is_reconcilable BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Standards and classification metadata in JSONB
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "gaap_classification": "current_asset",
    --   "ifrs_classification": "financial_asset_amortised_cost",
    --   "xbrl_element": "us-gaap:CashAndCashEquivalentsAtCarryingValue",
    --   "tax_line": "1040_line_1",
    --   "custom_tags": ["cash", "operating"],
    --   "notes": "Primary operating account"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, code)
);

CREATE INDEX idx_account_org ON account(tenant_id, org_id);
CREATE INDEX idx_account_type ON account(tenant_id, account_type);
CREATE INDEX idx_account_parent ON account(parent_id);
-- GIN index for JSONB queries on metadata
CREATE INDEX idx_account_metadata ON account USING GIN (metadata);

CREATE TABLE fiscal_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open',
    closed_by       UUID REFERENCES app_user(id),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, start_date, end_date)
);

CREATE TABLE journal_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    entry_number    TEXT NOT NULL,
    entry_date      DATE NOT NULL,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    description     TEXT,
    source_type     TEXT,
    source_id       UUID,
    status          TEXT NOT NULL DEFAULT 'draft',
    posted_by       UUID REFERENCES app_user(id),
    posted_at       TIMESTAMPTZ,
    reversed_by_id  UUID REFERENCES journal_entry(id),
    -- Source tracking and extended attributes
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "ai_generated": true,
    --   "ai_confidence": 0.95,
    --   "ai_explanation": "Auto-generated from bank reconciliation match",
    --   "tags": ["month_end", "accrual"],
    --   "attachments": ["receipt-2026-05-12.pdf"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, entry_number)
);

CREATE INDEX idx_je_org_date ON journal_entry(tenant_id, org_id, entry_date);
CREATE INDEX idx_je_period ON journal_entry(tenant_id, fiscal_period_id);
CREATE INDEX idx_je_source ON journal_entry(source_type, source_id);

CREATE TABLE journal_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    journal_entry_id UUID NOT NULL REFERENCES journal_entry(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES account(id),
    description     TEXT,
    debit           NUMERIC(19,4) NOT NULL DEFAULT 0,
    credit          NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    base_debit      NUMERIC(19,4) NOT NULL DEFAULT 0,
    base_credit     NUMERIC(19,4) NOT NULL DEFAULT 0,
    -- Dimensional tagging in JSONB (replaces separate cost_center and project tables)
    dimensions      JSONB NOT NULL DEFAULT '{}',
    -- Example dimensions:
    -- {
    --   "cost_center": "engineering",
    --   "project": "proj-042",
    --   "department": "R&D",
    --   "custom_1": "grant-2026-NSF"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_debit_credit CHECK (
        (debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0) OR (debit = 0 AND credit = 0)
    )
);

CREATE INDEX idx_jl_entry ON journal_line(tenant_id, journal_entry_id);
CREATE INDEX idx_jl_account ON journal_line(tenant_id, account_id);
CREATE INDEX idx_jl_dimensions ON journal_line USING GIN (dimensions);
```

## Contacts & Parties

```sql
-- ============================================================
-- CONTACTS & PARTIES
-- ============================================================

CREATE TABLE party (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    party_type      TEXT NOT NULL,                  -- 'customer', 'vendor', 'employee', 'other'
    code            TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    lei             TEXT,
    peppol_participant_id TEXT,

    -- ISO 20022 structured address (relational for compliance)
    street_name     TEXT,
    building_number TEXT,
    post_code       TEXT,
    town_name       TEXT,
    country_sub_division TEXT,
    country_code    CHAR(2),

    email           TEXT,
    phone           TEXT,
    payment_terms_days SMALLINT DEFAULT 30,
    credit_limit    NUMERIC(19,4),
    currency_code   CHAR(3) DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- Extended attributes, contacts, bank accounts all in JSONB
    extended        JSONB NOT NULL DEFAULT '{}',
    -- Example extended:
    -- {
    --   "contacts": [
    --     {"name": "Jane Smith", "title": "AP Manager", "email": "jane@acme.com", "phone": "+1-555-0142", "is_primary": true},
    --     {"name": "Bob Jones", "title": "Warehouse", "email": "bob@acme.com", "phone": "+1-555-0143"}
    --   ],
    --   "bank_accounts": [
    --     {"bank_name": "Chase", "account_number": "****4521", "routing": "021000021",
    --      "iban": null, "bic_swift": "CHASUS33", "currency": "USD", "is_default": true}
    --   ],
    --   "edi_config": {
    --     "standard": "x12",
    --     "qualifier": "ZZ",
    --     "edi_id": "ACME001",
    --     "transaction_sets": ["850", "810", "856"]
    --   },
    --   "custom_fields": {
    --     "vendor_rating": "A",
    --     "payment_method_preference": "ach",
    --     "minority_owned": true
    --   },
    --   "tags": ["preferred", "domestic"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, code)
);

CREATE INDEX idx_party_org ON party(tenant_id, org_id);
CREATE INDEX idx_party_type ON party(tenant_id, party_type);
CREATE INDEX idx_party_name ON party(tenant_id, display_name);
CREATE INDEX idx_party_extended ON party USING GIN (extended);

-- Example query: Find all vendors with EDI configured for X12
-- SELECT * FROM party
-- WHERE tenant_id = '<tenant>'
--   AND party_type = 'vendor'
--   AND extended @> '{"edi_config": {"standard": "x12"}}';

-- Example query: Find all parties tagged as 'preferred'
-- SELECT * FROM party
-- WHERE tenant_id = '<tenant>'
--   AND extended @> '{"tags": ["preferred"]}';
```

## Products & Inventory

```sql
-- ============================================================
-- PRODUCTS & INVENTORY
-- ============================================================

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             TEXT NOT NULL,
    gtin            TEXT,                           -- GS1 Global Trade Item Number
    name            TEXT NOT NULL,
    description     TEXT,
    product_type    TEXT NOT NULL DEFAULT 'goods',  -- 'goods', 'service', 'consumable'
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',     -- UN/ECE Rec 20
    cost_price      NUMERIC(19,4) DEFAULT 0,
    sell_price      NUMERIC(19,4) DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_trackable    BOOLEAN NOT NULL DEFAULT TRUE,
    reorder_point   NUMERIC(12,3) DEFAULT 0,
    reorder_qty     NUMERIC(12,3) DEFAULT 0,
    income_account_id UUID REFERENCES account(id),
    expense_account_id UUID REFERENCES account(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Product attributes, categories, variants, and GS1 extended data in JSONB
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example attributes:
    -- {
    --   "categories": ["electronics", "components"],
    --   "weight_kg": 0.250,
    --   "dimensions": {"length_cm": 15, "width_cm": 10, "height_cm": 5},
    --   "gs1_data": {
    --     "company_prefix": "0614141",
    --     "application_identifiers": {"21": "SN12345", "10": "BATCH-A"}
    --   },
    --   "variants": [
    --     {"name": "Color", "value": "Red"},
    --     {"name": "Size", "value": "Large"}
    --   ],
    --   "customs": {
    --     "hs_code": "8542.31",
    --     "country_of_origin": "CN"
    --   },
    --   "custom_fields": {
    --     "supplier_part_number": "ACM-W-100R",
    --     "lead_time_days": 14
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_attrs ON product USING GIN (attributes);

CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL,
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Warehouse configuration: locations, zones, layout in JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "locations": [
    --     {"code": "A1-S1", "name": "Aisle 1 Shelf 1", "type": "storage"},
    --     {"code": "REC-1", "name": "Receiving Dock 1", "type": "receiving"},
    --     {"code": "SHIP-1", "name": "Shipping Dock 1", "type": "shipping"}
    --   ],
    --   "address": {"street": "123 Warehouse Rd", "city": "Dallas", "state": "TX", "zip": "75201"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE stock_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    location_code   TEXT,                           -- References warehouse.config->locations
    quantity_on_hand NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_reserved NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,3) GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    avg_unit_cost   NUMERIC(19,4) NOT NULL DEFAULT 0,
    last_count_date DATE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, warehouse_id, location_code)
);

CREATE INDEX idx_stock_product ON stock_level(tenant_id, product_id);
CREATE INDEX idx_stock_warehouse ON stock_level(tenant_id, warehouse_id);

CREATE TABLE stock_movement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    movement_type   TEXT NOT NULL,                  -- 'receipt', 'shipment', 'transfer', 'adjustment'
    source_warehouse_id UUID REFERENCES warehouse(id),
    dest_warehouse_id UUID REFERENCES warehouse(id),
    quantity        NUMERIC(12,3) NOT NULL,
    unit_cost       NUMERIC(19,4),
    reference_type  TEXT,
    reference_id    UUID,
    notes           TEXT,
    moved_by        UUID REFERENCES app_user(id),
    moved_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sm_product ON stock_movement(tenant_id, product_id);
CREATE INDEX idx_sm_date ON stock_movement(tenant_id, moved_at);
```

## Purchase Orders, Sales Orders & Invoices

```sql
-- ============================================================
-- PURCHASE ORDERS
-- ============================================================

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    po_number       TEXT NOT NULL,
    vendor_id       UUID NOT NULL REFERENCES party(id),
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_date   DATE,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    payment_terms   TEXT,
    -- Line items stored inline as JSONB array
    lines           JSONB NOT NULL DEFAULT '[]',
    -- Example lines:
    -- [
    --   {
    --     "line_number": 1,
    --     "product_id": "i9j0k1l2-...",
    --     "product_sku": "SKU-123",
    --     "description": "Widget Model A",
    --     "quantity": 500,
    --     "unit_of_measure": "EA",
    --     "unit_price": 12.00,
    --     "tax_rate": 8.25,
    --     "tax_amount": 495.00,
    --     "line_total": 6000.00,
    --     "quantity_received": 0,
    --     "account_id": "m3n4o5p6-..."
    --   }
    -- ]
    -- Extended metadata for approvals, shipping, etc.
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "ai_created": true,
    --   "ai_input": "create a PO for 500 units of SKU-123 from Acme at $12 each, net-30",
    --   "approval_chain": [
    --     {"user_id": "...", "action": "approved", "at": "2026-05-12T10:30:00Z"}
    --   ],
    --   "shipping_address": "123 Warehouse Rd, Dallas, TX 75201",
    --   "ubl_buyer_reference": "BR-2026-042",
    --   "edi_sent": false
    -- }
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, po_number)
);

CREATE INDEX idx_po_org ON purchase_order(tenant_id, org_id);
CREATE INDEX idx_po_vendor ON purchase_order(tenant_id, vendor_id);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_metadata ON purchase_order USING GIN (metadata);

-- ============================================================
-- SALES ORDERS
-- ============================================================

CREATE TABLE sales_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    so_number       TEXT NOT NULL,
    customer_id     UUID NOT NULL REFERENCES party(id),
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    delivery_date   DATE,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    payment_terms   TEXT,
    lines           JSONB NOT NULL DEFAULT '[]',    -- Same structure as PO lines
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, so_number)
);

CREATE INDEX idx_so_customer ON sales_order(tenant_id, customer_id);
CREATE INDEX idx_so_status ON sales_order(tenant_id, status);

-- ============================================================
-- INVOICES
-- ============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    invoice_number  TEXT NOT NULL,
    invoice_type    TEXT NOT NULL,                  -- 'customer_invoice', 'vendor_bill', 'credit_note'
    party_id        UUID NOT NULL REFERENCES party(id),
    invoice_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(19,4) GENERATED ALWAYS AS (total - amount_paid) STORED,
    journal_entry_id UUID REFERENCES journal_entry(id),
    source_type     TEXT,
    source_id       UUID,
    -- Line items inline
    lines           JSONB NOT NULL DEFAULT '[]',
    -- Example lines:
    -- [
    --   {
    --     "line_number": 1,
    --     "product_id": "i9j0k1l2-...",
    --     "description": "Widget Model A - 500 units",
    --     "quantity": 500,
    --     "unit_price": 25.00,
    --     "discount_pct": 0,
    --     "tax_rate": 8.25,
    --     "tax_amount": 1031.25,
    --     "line_total": 12500.00,
    --     "account_id": "q7r8s9t0-..."
    --   }
    -- ]
    -- Extended metadata for e-invoicing, EDI, PEPPOL
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "peppol_sent": true,
    --   "peppol_message_id": "MSG-2026-...",
    --   "edi_sent": false,
    --   "ubl_document_id": "INV-2026-000142",
    --   "payment_instructions": {
    --     "method": "bank_transfer",
    --     "iban": "DE89370400440532013000",
    --     "bic": "COBADEFFXXX"
    --   }
    -- }
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, invoice_number)
);

CREATE INDEX idx_inv_party ON invoice(tenant_id, party_id);
CREATE INDEX idx_inv_status ON invoice(tenant_id, status);
CREATE INDEX idx_inv_due ON invoice(tenant_id, due_date) WHERE status NOT IN ('paid', 'cancelled');
CREATE INDEX idx_inv_metadata ON invoice USING GIN (metadata);

-- Example query: Find all invoices sent via PEPPOL
-- SELECT * FROM invoice
-- WHERE tenant_id = '<tenant>'
--   AND metadata @> '{"peppol_sent": true}';
```

## Payments & Banking

```sql
-- ============================================================
-- PAYMENTS & BANKING
-- ============================================================

CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    bank_name       TEXT NOT NULL,
    account_number  TEXT,
    iban            TEXT,
    bic_swift       TEXT,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    gl_account_id   UUID NOT NULL REFERENCES account(id),
    current_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Integration configuration in JSONB
    integration     JSONB NOT NULL DEFAULT '{}',
    -- Example integration:
    -- {
    --   "plaid": {"item_id": "...", "access_token": "...", "last_sync": "2026-05-12T..."},
    --   "open_banking": {"consent_id": "...", "provider": "truelayer"},
    --   "csv_import": {"date_format": "DD/MM/YYYY", "delimiter": ",", "encoding": "utf-8"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    transaction_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    description     TEXT,
    reference       TEXT,
    counterparty_name TEXT,
    import_source   TEXT,
    import_id       TEXT,
    is_reconciled   BOOLEAN NOT NULL DEFAULT FALSE,
    -- Reconciliation data in JSONB
    reconciliation  JSONB,
    -- Example reconciliation:
    -- {
    --   "matched_type": "invoice",
    --   "matched_id": "y5z6a7b8-...",
    --   "match_method": "ai_auto",
    --   "confidence_score": 0.94,
    --   "explanation": "Amount matches INV-2026-000142 ($2,706.25). Counterparty 'ACME CORP' matches vendor Acme Corp. Payment date within 3 days of due date.",
    --   "confirmed_by": "m3n4o5p6-...",
    --   "confirmed_at": "2026-05-12T14:30:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_btxn_account ON bank_transaction(tenant_id, bank_account_id);
CREATE INDEX idx_btxn_date ON bank_transaction(tenant_id, transaction_date);
CREATE INDEX idx_btxn_unreconciled ON bank_transaction(tenant_id, bank_account_id)
    WHERE is_reconciled = FALSE;
CREATE INDEX idx_btxn_recon ON bank_transaction USING GIN (reconciliation)
    WHERE reconciliation IS NOT NULL;

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    payment_number  TEXT NOT NULL,
    payment_type    TEXT NOT NULL,
    payment_method  TEXT NOT NULL,
    party_id        UUID NOT NULL REFERENCES party(id),
    bank_account_id UUID REFERENCES bank_account(id),
    payment_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    reference       TEXT,
    journal_entry_id UUID REFERENCES journal_entry(id),
    status          TEXT NOT NULL DEFAULT 'draft',
    -- Allocations and ISO 20022 payment data in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "allocations": [
    --     {"invoice_id": "y5z6a7b8-...", "amount": 2706.25}
    --   ],
    --   "iso20022": {
    --     "payment_information_id": "PMT-2026-000089",
    --     "end_to_end_id": "E2E-2026-000089",
    --     "purpose_code": "SUPP",
    --     "remittance_info": "INV-2026-000142"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, payment_number)
);

CREATE INDEX idx_payment_party ON payment(tenant_id, party_id);
```

## Tax & Currency

```sql
-- ============================================================
-- TAX & CURRENCY
-- ============================================================

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,
    name            TEXT NOT NULL,
    symbol          TEXT NOT NULL,
    decimal_places  SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE exchange_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    from_currency   CHAR(3) NOT NULL REFERENCES currency(code),
    to_currency     CHAR(3) NOT NULL REFERENCES currency(code),
    rate            NUMERIC(12,6) NOT NULL,
    rate_date       DATE NOT NULL,
    source          TEXT DEFAULT 'manual',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, from_currency, to_currency, rate_date)
);

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    rate            NUMERIC(5,2) NOT NULL,
    tax_type        TEXT NOT NULL,
    jurisdiction    TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,
    -- Jurisdiction-specific tax rules in JSONB
    rules           JSONB NOT NULL DEFAULT '{}',
    -- Example rules:
    -- {
    --   "applies_to": ["goods", "services"],
    --   "exemptions": ["food", "medical"],
    --   "reverse_charge": false,
    --   "reporting_code": "BOX1",
    --   "nexus_required": true,
    --   "state_rate": 6.00,
    --   "county_rate": 1.25,
    --   "city_rate": 1.00
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## AI Interaction & Audit

```sql
-- ============================================================
-- AI INTERACTION & AUDIT
-- ============================================================

CREATE TABLE ai_interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    input_text      TEXT NOT NULL,
    parsed_intent   TEXT,
    output_action   TEXT,
    output_entity_type TEXT,
    output_entity_id UUID,
    confidence_score NUMERIC(3,2),
    -- Full AI processing details in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "model_version": "erp-nl-v2.1",
    --   "latency_ms": 342,
    --   "tokens_used": 1247,
    --   "intermediate_steps": [
    --     {"step": "entity_extraction", "result": {"vendor": "Acme", "product": "SKU-123", "qty": 500}},
    --     {"step": "vendor_resolution", "result": {"vendor_id": "a1b2c3d4-...", "confidence": 0.98}},
    --     {"step": "product_resolution", "result": {"product_id": "i9j0k1l2-...", "confidence": 0.99}}
    --   ],
    --   "user_feedback": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_user ON ai_interaction(tenant_id, user_id);
CREATE INDEX idx_ai_date ON ai_interaction(tenant_id, created_at);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                          -- {"field": {"old": "...", "new": "..."}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_date ON audit_log(tenant_id, created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Multi-Tenancy | 2 | `tenant`, `organisation` |
| Users & RBAC | 3 | `app_user`, `role`, `user_role` (permissions embedded in role JSONB) |
| Chart of Accounts & GL | 4 | `account`, `fiscal_period`, `journal_entry`, `journal_line` |
| Contacts & Parties | 1 | `party` (contacts, bank accounts, EDI config all in JSONB) |
| Products & Inventory | 4 | `product`, `warehouse`, `stock_level`, `stock_movement` |
| Purchasing | 1 | `purchase_order` (lines in JSONB) |
| Sales | 1 | `sales_order` (lines in JSONB) |
| Invoicing | 1 | `invoice` (lines in JSONB) |
| Payments & Banking | 3 | `bank_account`, `bank_transaction`, `payment` |
| Tax & Currency | 3 | `currency`, `exchange_rate`, `tax_rate` |
| AI & Audit | 2 | `ai_interaction`, `audit_log` |
| **Total** | **25** | Significantly fewer than normalized (~40) |

---

## Key Design Decisions

1. **Line items stored as JSONB arrays instead of separate tables.** Purchase order lines, sales order lines, and invoice lines are stored as JSONB arrays within their parent document. This eliminates 6 join tables, simplifies CRUD operations (one INSERT/UPDATE instead of multi-table transactions), and makes API responses naturally nested. The trade-off is that you cannot query individual line items with the same efficiency as a normalized `invoice_line` table — but for SMB transaction volumes (thousands, not millions), GIN-indexed JSONB containment queries are performant.

2. **Permissions embedded in role JSONB.** Instead of `permission` and `role_permission` junction tables, permissions are stored as a JSONB array on the `role` table. This reduces the RBAC schema from 4 tables to 2 and makes the permission model self-documenting (you can read a role's permissions in one query). The trade-off is that permission queries require JSONB containment checks rather than simple joins.

3. **Party contacts and bank accounts in JSONB.** The normalized model uses 3 tables for party data (`party`, `party_contact`, `party_bank_account`). The hybrid model stores contacts and bank accounts as JSONB arrays within `party.extended`. For the typical SMB party (1-3 contacts, 1-2 bank accounts), this is more ergonomic and eliminates join overhead.

4. **Warehouse locations in JSONB.** Rather than a separate `warehouse_location` table, locations are stored in `warehouse.config` JSONB. This works well for SMB warehouses with tens of locations, not enterprise warehouses with thousands.

5. **Reconciliation data inline on bank transactions.** Instead of a separate `bank_reconciliation` table, the match result lives directly on `bank_transaction.reconciliation` JSONB. This makes the common query ("show me all unreconciled transactions") a simple WHERE clause without a LEFT JOIN.

6. **Jurisdiction-specific config in `organisation.locale_config`.** Tax systems, e-invoicing mandates, retention periods, and approval workflows vary by jurisdiction. Storing these in JSONB means adding support for a new jurisdiction (e.g., Italian SDI e-invoicing) requires no schema migration — just a new set of JSONB keys.

7. **Tax rules in JSONB for compound tax rates.** US sales tax often combines state + county + city rates. The `tax_rate.rules` JSONB stores these components, exemptions, and reverse-charge logic without requiring separate tables for each tax jurisdiction's complexity.

8. **ISO 20022 payment data in `payment.details`.** Outbound payment ISO 20022 fields (PaymentInformationId, EndToEndId, PurposeCode) are stored in `payment.details` JSONB, enabling direct serialisation to ISO 20022 PAIN.001 messages without a separate mapping table.

9. **GIN indexes on all JSONB columns.** Every table with a JSONB column has a GIN index (`CREATE INDEX ... USING GIN`) to support efficient containment queries (`@>`). This is critical for production performance of queries like "find all vendors with X12 EDI configured."

10. **Core monetary and compliance fields remain relational.** Despite the JSONB flexibility, all monetary amounts (`total`, `amount_paid`, `debit`, `credit`), dates, currency codes, and ISO-mandated structured address fields remain typed relational columns. This ensures database-level type safety and constraint enforcement for the fields where correctness matters most.
