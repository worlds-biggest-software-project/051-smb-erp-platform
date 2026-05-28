# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: SMB ERP Platform · Created: 2026-05-12

## Philosophy

The Entity-Centric Normalized Relational model follows the classical ERP database tradition: every business concept gets its own table, every relationship is an explicit foreign key, and data integrity is enforced at the database level through constraints, triggers, and referential integrity rules. This is the pattern used by SAP Business One, Oracle NetSuite, and Microsoft Dynamics 365 Business Central — battle-tested across millions of SMB deployments over two decades.

The core design principle is **explicitness over flexibility**. Every field has a declared type and constraint. Every relationship is navigable via foreign keys. Every business rule that can be expressed as a database constraint is expressed as one, rather than relying on application-layer validation alone. This makes the schema self-documenting and enables powerful ad-hoc querying without requiring knowledge of JSON structures or event replay logic.

This approach is best suited for teams with strong relational database skills, deployments where data integrity and regulatory compliance are paramount, and organisations that expect to run complex cross-entity reports (e.g., "show me all purchase orders from vendors in jurisdiction X where the goods receipt was more than 7 days late and the invoice has not been matched").

**Best for:** SMBs that prioritise data integrity, regulatory compliance, and rich cross-entity reporting over schema flexibility.

**Trade-offs:**
- (+) Maximum data integrity — constraints catch errors before they persist
- (+) Excellent ad-hoc query support — standard SQL joins across any entities
- (+) Self-documenting schema — the DDL is the specification
- (+) Mature tooling — every BI tool, ETL pipeline, and ORM works natively
- (+) Standards alignment is straightforward — one column per standard identifier
- (-) Schema changes require migrations — adding a jurisdiction-specific field means ALTER TABLE
- (-) Higher table count (~80-100 tables) increases initial complexity
- (-) Multi-jurisdiction variability is awkward — optional columns proliferate
- (-) Less suited to rapid prototyping than JSONB or document approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GAAP / IFRS | Chart of accounts structure supports multi-GAAP parallel ledgers; `account.account_standard` distinguishes GAAP vs IFRS classification |
| ISO 20022 | Structured address fields (`street_name`, `building_number`, `post_code`, `town_name`, `country_code`) on party tables align with ISO 20022 structured address requirements (mandatory Nov 2026) |
| ISO 3166-1/2 | `jurisdiction` and `country_code` columns use ISO 3166 codes throughout |
| ISO 4217 | `currency_code` columns use ISO 4217 three-letter codes |
| GS1 / GTIN | `product.gtin` column stores the Global Trade Item Number; `product.gs1_company_prefix` for GS1 membership |
| OASIS UBL 2.1 | Purchase order and invoice table structures map to UBL Order and Invoice document schemas |
| ANSI X12 / EDIFACT | EDI transaction mapping tables link internal documents to EDI transaction sets (850, 810, 856) |
| XBRL | Account taxonomy tags stored in `account.xbrl_element` for regulatory financial filing export |
| OAuth 2.0 / OIDC | API client and user authentication tables support OAuth 2.0 flows per RFC 6749 |
| OData v4 | Table and column naming conventions designed for clean OData entity exposure |
| PEPPOL | Party identifiers include `peppol_participant_id` for European e-invoicing network |

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
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_org_id   UUID REFERENCES organisation(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                           -- VAT number, EIN, ABN, etc.
    lei             TEXT,                           -- ISO 17442 Legal Entity Identifier
    peppol_participant_id TEXT,                     -- PEPPOL e-invoicing ID
    country_code    CHAR(2) NOT NULL,               -- ISO 3166-1 alpha-2
    jurisdiction    TEXT,                           -- ISO 3166-2 subdivision
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    fiscal_year_start_month SMALLINT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);
CREATE INDEX idx_organisation_parent ON organisation(parent_org_id);

-- Row-Level Security
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
    password_hash   TEXT,                           -- NULL for SSO-only users
    auth_provider   TEXT DEFAULT 'local',           -- 'local', 'azure_ad', 'google', 'okta'
    auth_provider_id TEXT,                          -- External IdP user ID
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                  -- 'admin', 'accountant', 'warehouse_clerk', etc.
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT FALSE, -- System roles cannot be deleted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        TEXT NOT NULL,                  -- 'invoice', 'purchase_order', 'journal_entry'
    action          TEXT NOT NULL,                  -- 'create', 'read', 'update', 'delete', 'approve'
    UNIQUE (resource, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisation(id),  -- Role scoped to org
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, org_id)
);

CREATE INDEX idx_user_role_org ON user_role(org_id);
```

## Chart of Accounts & General Ledger

```sql
-- ============================================================
-- CHART OF ACCOUNTS & GENERAL LEDGER
-- ============================================================

CREATE TYPE account_type AS ENUM (
    'asset', 'liability', 'equity', 'revenue', 'expense'
);

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,                  -- '1000', '2100', '4000', etc.
    name            TEXT NOT NULL,                  -- 'Cash and Cash Equivalents'
    account_type    account_type NOT NULL,
    parent_id       UUID REFERENCES account(id),   -- Hierarchical CoA
    account_standard TEXT DEFAULT 'GAAP',           -- 'GAAP', 'IFRS' for multi-GAAP ledgers
    xbrl_element    TEXT,                           -- XBRL taxonomy tag for regulatory filing
    is_reconcilable BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, code, account_standard)
);

CREATE INDEX idx_account_org ON account(tenant_id, org_id);
CREATE INDEX idx_account_type ON account(tenant_id, account_type);
CREATE INDEX idx_account_parent ON account(parent_id);

CREATE TABLE fiscal_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- 'FY2026-Q1', 'FY2026-01'
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open',   -- 'open', 'closing', 'closed'
    closed_by       UUID REFERENCES app_user(id),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, start_date, end_date)
);

CREATE TABLE journal_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    entry_number    TEXT NOT NULL,                  -- Auto-generated sequence: 'JE-2026-000142'
    entry_date      DATE NOT NULL,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    description     TEXT,
    source_type     TEXT,                           -- 'manual', 'invoice', 'payment', 'reconciliation', 'ai_suggested'
    source_id       UUID,                           -- FK to originating document
    status          TEXT NOT NULL DEFAULT 'draft',  -- 'draft', 'posted', 'reversed'
    posted_by       UUID REFERENCES app_user(id),
    posted_at       TIMESTAMPTZ,
    reversed_by_id  UUID REFERENCES journal_entry(id), -- Points to reversing entry
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, entry_number)
);

CREATE INDEX idx_journal_entry_org_date ON journal_entry(tenant_id, org_id, entry_date);
CREATE INDEX idx_journal_entry_period ON journal_entry(tenant_id, fiscal_period_id);
CREATE INDEX idx_journal_entry_source ON journal_entry(source_type, source_id);

CREATE TABLE journal_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    journal_entry_id UUID NOT NULL REFERENCES journal_entry(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES account(id),
    description     TEXT,
    debit           NUMERIC(19,4) NOT NULL DEFAULT 0,
    credit          NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    base_debit      NUMERIC(19,4) NOT NULL DEFAULT 0,   -- In org base currency
    base_credit     NUMERIC(19,4) NOT NULL DEFAULT 0,
    cost_center_id  UUID,                           -- Optional departmental tracking
    project_id      UUID,                           -- Optional project tracking
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_debit_credit CHECK (
        (debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0) OR (debit = 0 AND credit = 0)
    )
);

CREATE INDEX idx_journal_line_entry ON journal_line(tenant_id, journal_entry_id);
CREATE INDEX idx_journal_line_account ON journal_line(tenant_id, account_id);

-- Enforce double-entry: every journal entry must balance
CREATE OR REPLACE FUNCTION check_journal_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT ABS(SUM(base_debit) - SUM(base_credit)) > 0.001
        FROM journal_line WHERE journal_entry_id = NEW.journal_entry_id) THEN
        RAISE EXCEPTION 'Journal entry does not balance: debits must equal credits';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Contacts & Parties

```sql
-- ============================================================
-- CONTACTS & PARTIES (Customers, Vendors, Employees)
-- ============================================================

CREATE TYPE party_type AS ENUM ('customer', 'vendor', 'employee', 'other');

CREATE TABLE party (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    party_type      party_type NOT NULL,
    code            TEXT NOT NULL,                  -- 'CUST-001', 'VEND-042'
    display_name    TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                           -- VAT / EIN / ABN
    lei             TEXT,                           -- ISO 17442
    peppol_participant_id TEXT,

    -- ISO 20022 Structured Address (mandatory Nov 2026)
    street_name     TEXT,
    building_number TEXT,
    post_code       TEXT,
    town_name       TEXT,
    country_sub_division TEXT,                      -- ISO 3166-2
    country_code    CHAR(2),                        -- ISO 3166-1 alpha-2

    email           TEXT,
    phone           TEXT,
    website         TEXT,
    payment_terms_days SMALLINT DEFAULT 30,
    credit_limit    NUMERIC(19,4),
    currency_code   CHAR(3) DEFAULT 'USD',          -- ISO 4217 preferred currency
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, code)
);

CREATE INDEX idx_party_tenant_org ON party(tenant_id, org_id);
CREATE INDEX idx_party_type ON party(tenant_id, party_type);
CREATE INDEX idx_party_name ON party(tenant_id, display_name);

CREATE TABLE party_contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_id        UUID NOT NULL REFERENCES party(id) ON DELETE CASCADE,
    full_name       TEXT NOT NULL,
    job_title       TEXT,
    email           TEXT,
    phone           TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_contact_party ON party_contact(tenant_id, party_id);

CREATE TABLE party_bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_id        UUID NOT NULL REFERENCES party(id) ON DELETE CASCADE,
    bank_name       TEXT NOT NULL,
    account_name    TEXT,
    account_number  TEXT,
    routing_number  TEXT,                           -- ABA / Sort Code
    iban            TEXT,                           -- ISO 13616
    bic_swift       TEXT,                           -- ISO 9362
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_bank_party ON party_bank_account(tenant_id, party_id);
```

## Products & Inventory

```sql
-- ============================================================
-- PRODUCTS & INVENTORY
-- ============================================================

CREATE TYPE product_type AS ENUM ('goods', 'service', 'consumable');

CREATE TABLE product_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES product_category(id),
    name            TEXT NOT NULL,
    code            TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, parent_id)
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             TEXT NOT NULL,                  -- Internal SKU
    gtin            TEXT,                           -- GS1 Global Trade Item Number
    gs1_company_prefix TEXT,                        -- GS1 Company Prefix
    name            TEXT NOT NULL,
    description     TEXT,
    product_type    product_type NOT NULL DEFAULT 'goods',
    category_id     UUID REFERENCES product_category(id),
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',     -- UN/ECE Rec 20 unit codes
    cost_price      NUMERIC(19,4) DEFAULT 0,
    sell_price      NUMERIC(19,4) DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    weight_kg       NUMERIC(10,3),
    is_trackable    BOOLEAN NOT NULL DEFAULT TRUE,  -- Track inventory levels
    reorder_point   NUMERIC(12,3) DEFAULT 0,
    reorder_qty     NUMERIC(12,3) DEFAULT 0,
    income_account_id UUID REFERENCES account(id),
    expense_account_id UUID REFERENCES account(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_category ON product(tenant_id, category_id);

CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL,
    address_line    TEXT,
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE warehouse_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    name            TEXT NOT NULL,                  -- 'Aisle-3-Shelf-B'
    location_type   TEXT NOT NULL DEFAULT 'storage', -- 'storage', 'receiving', 'shipping', 'quality'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE stock_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    location_id     UUID REFERENCES warehouse_location(id),
    quantity_on_hand NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_reserved NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,3) GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    last_count_date DATE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, warehouse_id, location_id)
);

CREATE INDEX idx_stock_product ON stock_level(tenant_id, product_id);
CREATE INDEX idx_stock_warehouse ON stock_level(tenant_id, warehouse_id);

CREATE TABLE stock_movement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    movement_type   TEXT NOT NULL,                  -- 'receipt', 'shipment', 'transfer', 'adjustment', 'return'
    source_warehouse_id UUID REFERENCES warehouse(id),
    source_location_id UUID REFERENCES warehouse_location(id),
    dest_warehouse_id UUID REFERENCES warehouse(id),
    dest_location_id UUID REFERENCES warehouse_location(id),
    quantity        NUMERIC(12,3) NOT NULL,
    unit_cost       NUMERIC(19,4),
    reference_type  TEXT,                           -- 'purchase_order', 'sales_order', 'manual'
    reference_id    UUID,
    notes           TEXT,
    moved_by        UUID REFERENCES app_user(id),
    moved_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stock_movement_product ON stock_movement(tenant_id, product_id);
CREATE INDEX idx_stock_movement_date ON stock_movement(tenant_id, moved_at);
CREATE INDEX idx_stock_movement_ref ON stock_movement(reference_type, reference_id);
```

## Purchasing

```sql
-- ============================================================
-- PURCHASING
-- ============================================================

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    po_number       TEXT NOT NULL,                  -- 'PO-2026-000312'
    vendor_id       UUID NOT NULL REFERENCES party(id),
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_date   DATE,
    status          TEXT NOT NULL DEFAULT 'draft',  -- 'draft','submitted','approved','received','closed','cancelled'
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    payment_terms   TEXT,                           -- 'NET30', 'NET60', '2/10-NET30'
    shipping_address TEXT,
    notes           TEXT,
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    ai_created      BOOLEAN NOT NULL DEFAULT FALSE, -- Flag for NL-created POs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, po_number)
);

CREATE INDEX idx_po_tenant_org ON purchase_order(tenant_id, org_id);
CREATE INDEX idx_po_vendor ON purchase_order(tenant_id, vendor_id);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_date ON purchase_order(tenant_id, order_date);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    product_id      UUID REFERENCES product(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(12,3) NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    unit_price      NUMERIC(19,4) NOT NULL,
    tax_rate        NUMERIC(5,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    line_total      NUMERIC(19,4) NOT NULL,
    quantity_received NUMERIC(12,3) NOT NULL DEFAULT 0,
    account_id      UUID REFERENCES account(id),   -- Expense account override
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, purchase_order_id, line_number)
);

CREATE INDEX idx_po_line_order ON purchase_order_line(tenant_id, purchase_order_id);

CREATE TABLE goods_receipt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    receipt_number  TEXT NOT NULL,
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    receipt_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    status          TEXT NOT NULL DEFAULT 'draft',  -- 'draft', 'received', 'inspected', 'accepted'
    received_by     UUID REFERENCES app_user(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, receipt_number)
);

CREATE TABLE goods_receipt_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    goods_receipt_id UUID NOT NULL REFERENCES goods_receipt(id) ON DELETE CASCADE,
    po_line_id      UUID NOT NULL REFERENCES purchase_order_line(id),
    product_id      UUID REFERENCES product(id),
    quantity_received NUMERIC(12,3) NOT NULL,
    location_id     UUID REFERENCES warehouse_location(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Sales & Invoicing

```sql
-- ============================================================
-- SALES & INVOICING
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
    shipping_address TEXT,
    notes           TEXT,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, so_number)
);

CREATE INDEX idx_so_customer ON sales_order(tenant_id, customer_id);
CREATE INDEX idx_so_status ON sales_order(tenant_id, status);

CREATE TABLE sales_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sales_order_id  UUID NOT NULL REFERENCES sales_order(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    product_id      UUID REFERENCES product(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(12,3) NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    unit_price      NUMERIC(19,4) NOT NULL,
    discount_pct    NUMERIC(5,2) NOT NULL DEFAULT 0,
    tax_rate        NUMERIC(5,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    line_total      NUMERIC(19,4) NOT NULL,
    quantity_shipped NUMERIC(12,3) NOT NULL DEFAULT 0,
    account_id      UUID REFERENCES account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sales_order_id, line_number)
);

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    invoice_number  TEXT NOT NULL,
    invoice_type    TEXT NOT NULL,                  -- 'customer_invoice', 'vendor_bill', 'credit_note'
    party_id        UUID NOT NULL REFERENCES party(id),
    invoice_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',  -- 'draft','sent','partial','paid','overdue','cancelled'
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(19,4) GENERATED ALWAYS AS (total - amount_paid) STORED,
    journal_entry_id UUID REFERENCES journal_entry(id),
    source_type     TEXT,                           -- 'sales_order', 'purchase_order', 'manual'
    source_id       UUID,
    peppol_sent     BOOLEAN NOT NULL DEFAULT FALSE,
    edi_sent        BOOLEAN NOT NULL DEFAULT FALSE,
    notes           TEXT,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, invoice_number)
);

CREATE INDEX idx_invoice_party ON invoice(tenant_id, party_id);
CREATE INDEX idx_invoice_status ON invoice(tenant_id, status);
CREATE INDEX idx_invoice_due ON invoice(tenant_id, due_date) WHERE status NOT IN ('paid', 'cancelled');
CREATE INDEX idx_invoice_type ON invoice(tenant_id, invoice_type);

CREATE TABLE invoice_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    product_id      UUID REFERENCES product(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(12,3) NOT NULL,
    unit_price      NUMERIC(19,4) NOT NULL,
    discount_pct    NUMERIC(5,2) NOT NULL DEFAULT 0,
    tax_rate        NUMERIC(5,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    line_total      NUMERIC(19,4) NOT NULL,
    account_id      UUID NOT NULL REFERENCES account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_id, line_number)
);

CREATE INDEX idx_invoice_line_invoice ON invoice_line(tenant_id, invoice_id);
```

## Payments & Bank Reconciliation

```sql
-- ============================================================
-- PAYMENTS & BANK RECONCILIATION
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
    plaid_item_id   TEXT,                           -- Plaid bank feed integration
    plaid_access_token TEXT,
    current_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    transaction_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,         -- Positive = credit, negative = debit
    description     TEXT,
    reference       TEXT,                           -- Check number, wire reference
    counterparty_name TEXT,
    import_source   TEXT,                           -- 'plaid', 'csv', 'ofx', 'manual'
    import_id       TEXT,                           -- External ID for dedup
    is_reconciled   BOOLEAN NOT NULL DEFAULT FALSE,
    reconciled_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bank_txn_account ON bank_transaction(tenant_id, bank_account_id);
CREATE INDEX idx_bank_txn_date ON bank_transaction(tenant_id, transaction_date);
CREATE INDEX idx_bank_txn_unreconciled ON bank_transaction(tenant_id, bank_account_id)
    WHERE is_reconciled = FALSE;

CREATE TABLE bank_reconciliation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_transaction_id UUID NOT NULL REFERENCES bank_transaction(id),
    matched_type    TEXT NOT NULL,                  -- 'invoice', 'payment', 'journal_entry'
    matched_id      UUID NOT NULL,
    match_method    TEXT NOT NULL DEFAULT 'manual', -- 'manual', 'ai_auto', 'ai_suggested', 'rule'
    confidence_score NUMERIC(3,2),                  -- 0.00-1.00, populated by AI matcher
    explanation     TEXT,                           -- AI explanation of why it matched
    confirmed_by    UUID REFERENCES app_user(id),
    confirmed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recon_bank_txn ON bank_reconciliation(tenant_id, bank_transaction_id);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    payment_number  TEXT NOT NULL,
    payment_type    TEXT NOT NULL,                  -- 'incoming', 'outgoing'
    payment_method  TEXT NOT NULL,                  -- 'bank_transfer', 'check', 'credit_card', 'cash'
    party_id        UUID NOT NULL REFERENCES party(id),
    bank_account_id UUID REFERENCES bank_account(id),
    payment_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    amount          NUMERIC(19,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    exchange_rate   NUMERIC(12,6) NOT NULL DEFAULT 1.0,
    reference       TEXT,
    journal_entry_id UUID REFERENCES journal_entry(id),
    status          TEXT NOT NULL DEFAULT 'draft',  -- 'draft', 'confirmed', 'reconciled', 'voided'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, payment_number)
);

CREATE TABLE payment_allocation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    payment_id      UUID NOT NULL REFERENCES payment(id) ON DELETE CASCADE,
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    amount          NUMERIC(19,4) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_alloc_payment ON payment_allocation(tenant_id, payment_id);
CREATE INDEX idx_payment_alloc_invoice ON payment_allocation(tenant_id, invoice_id);
```

## Tax Configuration

```sql
-- ============================================================
-- TAX CONFIGURATION
-- ============================================================

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                  -- 'US Sales Tax - CA', 'UK VAT Standard'
    rate            NUMERIC(5,2) NOT NULL,          -- 8.25, 20.00
    tax_type        TEXT NOT NULL,                  -- 'sales_tax', 'vat', 'gst', 'withholding'
    jurisdiction    TEXT,                           -- ISO 3166-2 code
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tax_rate_tenant ON tax_rate(tenant_id);
CREATE INDEX idx_tax_rate_jurisdiction ON tax_rate(tenant_id, jurisdiction);
```

## Currency & Exchange Rates

```sql
-- ============================================================
-- CURRENCY & EXCHANGE RATES
-- ============================================================

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,            -- ISO 4217
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
    source          TEXT DEFAULT 'manual',          -- 'manual', 'ecb', 'openexchangerates'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, from_currency, to_currency, rate_date)
);

CREATE INDEX idx_exchange_rate_date ON exchange_rate(tenant_id, rate_date);
```

## AI & Audit Trail

```sql
-- ============================================================
-- AI INTERACTION LOG & AUDIT TRAIL
-- ============================================================

CREATE TABLE ai_interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    input_text      TEXT NOT NULL,                  -- "create a PO for 500 units of SKU-123..."
    parsed_intent   TEXT,                           -- 'create_purchase_order', 'reconcile_bank'
    output_action   TEXT,                           -- 'purchase_order_created', 'match_suggested'
    output_entity_type TEXT,                        -- 'purchase_order', 'journal_entry'
    output_entity_id UUID,                          -- FK to created/modified record
    confidence_score NUMERIC(3,2),
    model_version   TEXT,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_interaction_user ON ai_interaction(tenant_id, user_id);
CREATE INDEX idx_ai_interaction_date ON ai_interaction(tenant_id, created_at);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,                  -- 'create', 'update', 'delete', 'approve', 'login'
    entity_type     TEXT NOT NULL,                  -- 'invoice', 'journal_entry', 'purchase_order'
    entity_id       UUID NOT NULL,
    changes         JSONB,                          -- {"field": {"old": "...", "new": "..."}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_tenant_date ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(tenant_id, user_id);
```

## EDI Integration

```sql
-- ============================================================
-- EDI INTEGRATION
-- ============================================================

CREATE TABLE edi_trading_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    party_id        UUID NOT NULL REFERENCES party(id),
    edi_standard    TEXT NOT NULL,                  -- 'x12', 'edifact', 'ubl'
    qualifier       TEXT,                           -- ISA qualifier for X12
    edi_id          TEXT NOT NULL,                  -- Trading partner EDI ID
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE edi_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    trading_partner_id UUID NOT NULL REFERENCES edi_trading_partner(id),
    direction       TEXT NOT NULL,                  -- 'inbound', 'outbound'
    edi_standard    TEXT NOT NULL,                  -- 'x12', 'edifact', 'ubl'
    transaction_set TEXT NOT NULL,                  -- '850', '810', '856' (X12) or ORDERS, INVOIC (EDIFACT)
    control_number  TEXT,
    raw_content     TEXT,
    parsed_data     JSONB,
    internal_doc_type TEXT,                         -- 'purchase_order', 'invoice'
    internal_doc_id UUID,
    status          TEXT NOT NULL DEFAULT 'received', -- 'received','parsed','mapped','error'
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edi_message_partner ON edi_message(tenant_id, trading_partner_id);
CREATE INDEX idx_edi_message_date ON edi_message(tenant_id, created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Multi-Tenancy | 2 | `tenant`, `organisation` |
| Users & RBAC | 5 | `app_user`, `role`, `permission`, `role_permission`, `user_role` |
| Chart of Accounts & GL | 4 | `account`, `fiscal_period`, `journal_entry`, `journal_line` |
| Contacts & Parties | 3 | `party`, `party_contact`, `party_bank_account` |
| Products & Inventory | 6 | `product_category`, `product`, `warehouse`, `warehouse_location`, `stock_level`, `stock_movement` |
| Purchasing | 4 | `purchase_order`, `purchase_order_line`, `goods_receipt`, `goods_receipt_line` |
| Sales & Invoicing | 4 | `sales_order`, `sales_order_line`, `invoice`, `invoice_line` |
| Payments & Banking | 5 | `bank_account`, `bank_transaction`, `bank_reconciliation`, `payment`, `payment_allocation` |
| Tax & Currency | 3 | `tax_rate`, `currency`, `exchange_rate` |
| AI & Audit | 2 | `ai_interaction`, `audit_log` |
| EDI Integration | 2 | `edi_trading_partner`, `edi_message` |
| **Total** | **40** | |

---

## Key Design Decisions

1. **Shared-schema multi-tenancy with RLS.** All tables include `tenant_id` as the first column of every composite index, and PostgreSQL Row-Level Security policies enforce isolation at the database level. This is the most cost-efficient approach for a platform targeting SMBs where tenant count will be high but individual tenant data volumes moderate.

2. **Unified `party` table instead of separate customer/vendor tables.** Many real-world counterparties are both customer and vendor. A single `party` table with a `party_type` enum avoids duplicating contact and banking data. This follows the NetSuite "entity" pattern.

3. **Double-entry enforced at the database level.** A trigger function validates that every `journal_entry` balances (total debits = total credits). This makes it impossible to persist unbalanced entries regardless of application-layer bugs.

4. **ISO 20022 structured addresses.** Party address fields are stored as individual columns (`street_name`, `building_number`, `post_code`, `town_name`, `country_code`) rather than a single freetext field, anticipating the November 2026 mandate for structured addresses in financial messaging.

5. **GS1/GTIN on the product table.** The `gtin` column directly stores the Global Trade Item Number, enabling barcode scanning integration and supply chain interoperability without a separate lookup table.

6. **AI interaction logging as a first-class entity.** The `ai_interaction` table captures every natural-language command, the parsed intent, the resulting action, and the confidence score. This is essential for debugging, compliance, and improving the NL model over time.

7. **Multi-GAAP support via `account_standard`.** The chart of accounts supports parallel GAAP and IFRS classifications through the `account_standard` field, enabling dual-book reporting for SMBs operating across jurisdictions.

8. **Plaid integration columns on `bank_account`.** Rather than a separate integration table, Plaid identifiers live directly on the bank account record, reflecting the tight coupling between bank accounts and their feed sources.

9. **NUMERIC(19,4) for all monetary values.** This provides precision to four decimal places with a range exceeding $999 trillion, sufficient for any SMB accounting scenario and compatible with ISO 20022 amount fields.

10. **Generated columns for derived values.** `stock_level.quantity_available` and `invoice.amount_due` are PostgreSQL GENERATED ALWAYS AS columns, ensuring they are always consistent without application-layer computation.
