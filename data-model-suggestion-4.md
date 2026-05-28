# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: SMB ERP Platform · Created: 2026-05-12

## Philosophy

The Graph-Relational Hybrid model maintains conventional relational tables for transactional CRUD operations (invoices, purchase orders, journal entries) but adds a property graph layer for relationship-heavy queries. In an ERP system, many of the most valuable analytical questions are fundamentally graph questions: "who are our single-source-of-supply vendors?", "what is the approval chain for this PO?", "which customers are connected to which vendors through shared contacts?", "what is the full lifecycle of this product from PO through receipt, stock, sale, and invoice?"

Rather than using a separate graph database (Neo4j, Amazon Neptune), this model implements the graph layer in PostgreSQL using `graph_node` and `graph_edge` tables with JSONB properties, combined with PostgreSQL's recursive CTEs for traversal queries. This keeps the entire system in one database engine, eliminates sync issues between a relational DB and a separate graph store, and leverages PostgreSQL's mature transaction and RLS capabilities.

The graph layer is particularly powerful for the AI features described in the project specification. An AI cash flow predictor benefits from understanding the graph of relationships between vendors, products, purchase orders, and payments — not just flat table data. An AI-driven conflict-of-interest detector can traverse the graph to find hidden relationships. Supply chain risk analysis (single-source detection, vendor dependency mapping) is a natural graph traversal problem.

**Best for:** SMBs with complex supply chains, multi-entity structures, or regulatory requirements for relationship tracing (conflict-of-interest, supply chain transparency, beneficial ownership).

**Trade-offs:**
- (+) Powerful relationship queries — supply chain mapping, approval chains, entity relationship analysis
- (+) AI-friendly — graph structures are natural inputs for GNN-based anomaly detection and recommendation
- (+) Single database engine — no sync issues between relational and graph databases
- (+) Flexible relationship types — new edge types added without schema migration
- (+) Enables supply chain transparency and conflict-of-interest analysis
- (-) Graph queries via recursive CTEs are less intuitive than native graph query languages (Cypher, Gremlin)
- (-) Deep traversals (>5 hops) on large graphs can be slow in PostgreSQL
- (-) Graph layer adds ~3 tables and additional complexity over a pure relational model
- (-) Developers need to maintain both relational and graph representations
- (-) Less mature ecosystem — fewer ORMs and tools support this hybrid pattern

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GAAP / IFRS | Relational accounting tables follow standard double-entry structure; graph edges link journal entries to their source documents for full traceability |
| ISO 20022 | Structured address fields on party nodes; payment edges carry ISO 20022 remittance references |
| ISO 3166-1/2 | Jurisdiction nodes in the graph enable geographic traversal queries |
| ISO 4217 | Currency codes on all monetary fields in both relational and graph layers |
| GS1 / GTIN | Product nodes include GTIN properties; supply chain edges link products to vendors |
| OASIS UBL 2.1 | Document lifecycle modeled as graph edges (PO -> GoodsReceipt -> Invoice -> Payment) |
| ISO 17442 (LEI) | Legal Entity Identifier stored on organisation and party nodes for entity resolution |
| PEPPOL | PEPPOL participant IDs as node properties; e-invoicing interactions as graph edges |
| GDPR | Relationship graph enables complete data subject impact analysis — "show me everything connected to this person" |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH LAYER — PROPERTY GRAPH IN POSTGRESQL
-- ============================================================
-- Implements a property graph model using two tables:
-- graph_node: vertices with typed properties
-- graph_edge: directed edges with typed properties
--
-- All business entities have both a relational table (for CRUD)
-- and a corresponding graph node (for relationship queries).
-- The graph node's entity_id references the relational table's PK.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL,
    -- Node types:
    -- 'Organisation', 'Party', 'Product', 'Warehouse', 'Account',
    -- 'PurchaseOrder', 'SalesOrder', 'Invoice', 'Payment',
    -- 'BankAccount', 'User', 'JournalEntry', 'StockMovement'
    entity_id       UUID NOT NULL,                  -- FK to the relational table
    label           TEXT NOT NULL,                  -- Human-readable: 'Acme Corp', 'SKU-123', 'INV-2026-000142'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties for a Party node:
    -- {
    --   "party_type": "vendor",
    --   "country_code": "US",
    --   "jurisdiction": "US-CA",
    --   "lei": "529900T8BM49AURSDO55",
    --   "credit_limit": 50000,
    --   "risk_score": 0.15
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE INDEX idx_gn_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gn_entity ON graph_node(tenant_id, entity_id);
CREATE INDEX idx_gn_label ON graph_node(tenant_id, label);
CREATE INDEX idx_gn_properties ON graph_node USING GIN (properties);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    from_node_id    UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    to_node_id      UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
    -- Edge types:
    -- 'BELONGS_TO'        — Organisation -> Parent Organisation
    -- 'SUPPLIES'          — Party(vendor) -> Product
    -- 'PURCHASES_FROM'    — Organisation -> Party(vendor)
    -- 'SELLS_TO'          — Organisation -> Party(customer)
    -- 'ORDERED'           — PurchaseOrder -> Product (via lines)
    -- 'INVOICED'          — Invoice -> Party
    -- 'PAID_BY'           — Payment -> Invoice
    -- 'RECEIVED_AT'       — StockMovement -> Warehouse
    -- 'DEBITS'            — JournalEntry -> Account
    -- 'CREDITS'           — JournalEntry -> Account
    -- 'APPROVED_BY'       — PurchaseOrder -> User
    -- 'CREATED_BY'        — any entity -> User
    -- 'CONTACT_OF'        — User/Contact -> Party
    -- 'STORED_IN'         — Product -> Warehouse
    -- 'FULFILLS'          — Invoice -> SalesOrder
    -- 'DERIVED_FROM'      — Invoice -> PurchaseOrder
    -- 'RECONCILED_WITH'   — BankTransaction -> Payment/Invoice
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties for a SUPPLIES edge:
    -- {
    --   "since": "2024-01-15",
    --   "lead_time_days": 14,
    --   "unit_price": 12.00,
    --   "currency": "USD",
    --   "is_primary": true,
    --   "last_order_date": "2026-05-01"
    -- }
    -- Example properties for a PAID_BY edge:
    -- {
    --   "amount": 2706.25,
    --   "payment_date": "2026-05-12",
    --   "method": "bank_transfer",
    --   "iso20022_e2e_id": "E2E-2026-000089"
    -- }
    weight          NUMERIC(12,4),                  -- Optional weight for ranking algorithms
    valid_from      TIMESTAMPTZ DEFAULT now(),       -- Temporal: when this relationship started
    valid_to        TIMESTAMPTZ,                    -- Temporal: when this relationship ended (NULL = current)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_from ON graph_edge(tenant_id, from_node_id);
CREATE INDEX idx_ge_to ON graph_edge(tenant_id, to_node_id);
CREATE INDEX idx_ge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_ge_from_type ON graph_edge(tenant_id, from_node_id, edge_type);
CREATE INDEX idx_ge_to_type ON graph_edge(tenant_id, to_node_id, edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING GIN (properties);
-- Temporal index: find current relationships
CREATE INDEX idx_ge_current ON graph_edge(tenant_id, edge_type)
    WHERE valid_to IS NULL;

-- Prevent duplicate active edges of the same type between the same nodes
CREATE UNIQUE INDEX idx_ge_unique_active ON graph_edge(tenant_id, from_node_id, to_node_id, edge_type)
    WHERE valid_to IS NULL;

-- Materialised view for common graph statistics
CREATE TABLE graph_node_stats (
    node_id         UUID PRIMARY KEY REFERENCES graph_node(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    in_degree       INTEGER NOT NULL DEFAULT 0,     -- Number of incoming edges
    out_degree      INTEGER NOT NULL DEFAULT 0,     -- Number of outgoing edges
    total_degree    INTEGER GENERATED ALWAYS AS (in_degree + out_degree) STORED,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gns_tenant ON graph_node_stats(tenant_id);
```

### Graph Query Examples

```sql
-- ============================================================
-- GRAPH TRAVERSAL QUERIES (RECURSIVE CTEs)
-- ============================================================

-- 1. Full document lifecycle: PO -> Goods Receipt -> Invoice -> Payment
-- "Show me the complete lifecycle of purchase order PO-2026-000312"
--
-- WITH RECURSIVE lifecycle AS (
--     -- Start from the PO node
--     SELECT
--         gn.id AS node_id,
--         gn.node_type,
--         gn.label,
--         ge.edge_type,
--         0 AS depth,
--         ARRAY[gn.id] AS path
--     FROM graph_node gn
--     WHERE gn.tenant_id = '<tenant>'
--       AND gn.node_type = 'PurchaseOrder'
--       AND gn.label = 'PO-2026-000312'
--
--     UNION ALL
--
--     -- Follow edges forward
--     SELECT
--         gn2.id,
--         gn2.node_type,
--         gn2.label,
--         ge.edge_type,
--         lc.depth + 1,
--         lc.path || gn2.id
--     FROM lifecycle lc
--     JOIN graph_edge ge ON ge.from_node_id = lc.node_id
--         AND ge.tenant_id = '<tenant>'
--         AND ge.valid_to IS NULL
--     JOIN graph_node gn2 ON gn2.id = ge.to_node_id
--     WHERE lc.depth < 10
--       AND NOT (gn2.id = ANY(lc.path))  -- Prevent cycles
-- )
-- SELECT node_type, label, edge_type, depth FROM lifecycle ORDER BY depth;

-- 2. Single-source-of-supply risk analysis
-- "Which products have only one vendor?"
--
-- SELECT
--     p.label AS product,
--     p.properties->>'sku' AS sku,
--     COUNT(DISTINCT ge.from_node_id) AS vendor_count,
--     ARRAY_AGG(v.label) AS vendors
-- FROM graph_node p
-- JOIN graph_edge ge ON ge.to_node_id = p.id
--     AND ge.edge_type = 'SUPPLIES'
--     AND ge.valid_to IS NULL
-- JOIN graph_node v ON v.id = ge.from_node_id
-- WHERE p.tenant_id = '<tenant>'
--   AND p.node_type = 'Product'
-- GROUP BY p.id, p.label, p.properties->>'sku'
-- HAVING COUNT(DISTINCT ge.from_node_id) = 1;

-- 3. Vendor dependency analysis
-- "How much of our spending goes through each vendor?"
--
-- SELECT
--     v.label AS vendor,
--     COUNT(DISTINCT po.id) AS order_count,
--     SUM((ge.properties->>'total')::NUMERIC) AS total_spend,
--     COUNT(DISTINCT prod_edge.to_node_id) AS products_supplied
-- FROM graph_node v
-- JOIN graph_edge ge ON ge.to_node_id = v.id
--     AND ge.edge_type = 'PURCHASES_FROM'
--     AND ge.valid_to IS NULL
-- JOIN graph_node po ON po.id = ge.from_node_id
--     AND po.node_type = 'PurchaseOrder'
-- LEFT JOIN graph_edge prod_edge ON prod_edge.from_node_id = v.id
--     AND prod_edge.edge_type = 'SUPPLIES'
--     AND prod_edge.valid_to IS NULL
-- WHERE v.tenant_id = '<tenant>'
--   AND v.node_type = 'Party'
-- GROUP BY v.id, v.label
-- ORDER BY total_spend DESC;

-- 4. Approval chain traversal
-- "Who approved PO-2026-000312 and what else have they approved recently?"
--
-- SELECT
--     u.label AS approver,
--     po.label AS document,
--     po.node_type AS doc_type,
--     ge.properties->>'approved_at' AS approved_at,
--     (ge.properties->>'amount')::NUMERIC AS amount
-- FROM graph_node u
-- JOIN graph_edge ge ON ge.from_node_id = u.id
--     AND ge.edge_type = 'APPROVED_BY'
--     AND ge.valid_to IS NULL
-- JOIN graph_node po ON po.id = ge.to_node_id
-- WHERE u.tenant_id = '<tenant>'
--   AND u.node_type = 'User'
--   AND ge.created_at > now() - INTERVAL '30 days'
-- ORDER BY ge.created_at DESC;

-- 5. Organisation hierarchy traversal (multi-entity)
-- "Show the complete organisational tree"
--
-- WITH RECURSIVE org_tree AS (
--     SELECT gn.id, gn.label, gn.properties, 0 AS depth, ARRAY[gn.label::TEXT] AS path
--     FROM graph_node gn
--     LEFT JOIN graph_edge ge ON ge.from_node_id = gn.id
--         AND ge.edge_type = 'BELONGS_TO'
--         AND ge.valid_to IS NULL
--     WHERE gn.tenant_id = '<tenant>'
--       AND gn.node_type = 'Organisation'
--       AND ge.id IS NULL  -- Root orgs have no BELONGS_TO edge
--
--     UNION ALL
--
--     SELECT gn.id, gn.label, gn.properties, ot.depth + 1, ot.path || gn.label::TEXT
--     FROM org_tree ot
--     JOIN graph_edge ge ON ge.to_node_id = ot.id
--         AND ge.edge_type = 'BELONGS_TO'
--         AND ge.valid_to IS NULL
--     JOIN graph_node gn ON gn.id = ge.from_node_id
--     WHERE ot.depth < 10
-- )
-- SELECT REPEAT('  ', depth) || label AS org_name, properties->>'country_code' AS country
-- FROM org_tree ORDER BY path;

-- 6. GDPR: "Show me everything connected to this person"
--
-- WITH RECURSIVE connected AS (
--     SELECT gn.id, gn.node_type, gn.label, 0 AS depth
--     FROM graph_node gn
--     WHERE gn.tenant_id = '<tenant>'
--       AND gn.entity_id = '<person-uuid>'
--
--     UNION ALL
--
--     SELECT gn2.id, gn2.node_type, gn2.label, c.depth + 1
--     FROM connected c
--     JOIN graph_edge ge ON (ge.from_node_id = c.id OR ge.to_node_id = c.id)
--         AND ge.tenant_id = '<tenant>'
--     JOIN graph_node gn2 ON gn2.id = CASE
--         WHEN ge.from_node_id = c.id THEN ge.to_node_id
--         ELSE ge.from_node_id END
--     WHERE c.depth < 3
-- )
-- SELECT DISTINCT node_type, label FROM connected ORDER BY node_type, label;
```

## Relational Tables (Transactional CRUD)

```sql
-- ============================================================
-- TENANT & USERS (same as other models)
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
    tax_id          TEXT,
    lei             TEXT,
    peppol_participant_id TEXT,
    country_code    CHAR(2) NOT NULL,
    jurisdiction    TEXT,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    fiscal_year_start_month SMALLINT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_tenant ON organisation(tenant_id);

ALTER TABLE organisation ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON organisation
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    password_hash   TEXT,
    auth_provider   TEXT DEFAULT 'local',
    auth_provider_id TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
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
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        TEXT NOT NULL,
    action          TEXT NOT NULL,
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
    org_id          UUID NOT NULL REFERENCES organisation(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, org_id)
);
```

## Accounting

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
    account_type    TEXT NOT NULL,
    parent_id       UUID REFERENCES account(id),
    account_standard TEXT DEFAULT 'GAAP',
    xbrl_element    TEXT,
    is_reconcilable BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, code, account_standard)
);

CREATE INDEX idx_account_org ON account(tenant_id, org_id);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, entry_number)
);

CREATE INDEX idx_je_org_date ON journal_entry(tenant_id, org_id, entry_date);

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
    cost_center_id  UUID,
    project_id      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_debit_credit CHECK (
        (debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0) OR (debit = 0 AND credit = 0)
    )
);

CREATE INDEX idx_jl_entry ON journal_line(tenant_id, journal_entry_id);
CREATE INDEX idx_jl_account ON journal_line(tenant_id, account_id);
```

## Parties, Products & Inventory

```sql
-- ============================================================
-- PARTIES
-- ============================================================

CREATE TABLE party (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    party_type      TEXT NOT NULL,
    code            TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    lei             TEXT,
    peppol_participant_id TEXT,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, code)
);

CREATE INDEX idx_party_org ON party(tenant_id, org_id);
CREATE INDEX idx_party_type ON party(tenant_id, party_type);

-- ============================================================
-- PRODUCTS & INVENTORY
-- ============================================================

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             TEXT NOT NULL,
    gtin            TEXT,
    name            TEXT NOT NULL,
    description     TEXT,
    product_type    TEXT NOT NULL DEFAULT 'goods',
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    cost_price      NUMERIC(19,4) DEFAULT 0,
    sell_price      NUMERIC(19,4) DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_trackable    BOOLEAN NOT NULL DEFAULT TRUE,
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

CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL,
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE stock_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    quantity_on_hand NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_reserved NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,3) GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    avg_unit_cost   NUMERIC(19,4) NOT NULL DEFAULT 0,
    last_count_date DATE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, warehouse_id)
);

CREATE TABLE stock_movement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    movement_type   TEXT NOT NULL,
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

## Purchasing, Sales & Invoicing

```sql
-- ============================================================
-- PURCHASING
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
    notes           TEXT,
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    ai_created      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, po_number)
);

CREATE INDEX idx_po_vendor ON purchase_order(tenant_id, vendor_id);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);

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
    account_id      UUID REFERENCES account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, purchase_order_id, line_number)
);

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
    notes           TEXT,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, so_number)
);

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

-- ============================================================
-- INVOICES
-- ============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    invoice_number  TEXT NOT NULL,
    invoice_type    TEXT NOT NULL,
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
    notes           TEXT,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, org_id, invoice_number)
);

CREATE INDEX idx_inv_party ON invoice(tenant_id, party_id);
CREATE INDEX idx_inv_status ON invoice(tenant_id, status);
CREATE INDEX idx_inv_due ON invoice(tenant_id, due_date) WHERE status NOT IN ('paid', 'cancelled');

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
    reconciled_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_btxn_account ON bank_transaction(tenant_id, bank_account_id);
CREATE INDEX idx_btxn_unreconciled ON bank_transaction(tenant_id, bank_account_id)
    WHERE is_reconciled = FALSE;

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
```

## Reference Data & AI

```sql
-- ============================================================
-- REFERENCE DATA
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AI & AUDIT
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
    model_version   TEXT,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_user ON ai_interaction(tenant_id, user_id);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_date ON audit_log(tenant_id, created_at);
```

## Graph Maintenance Functions

```sql
-- ============================================================
-- GRAPH MAINTENANCE — TRIGGER FUNCTIONS
-- ============================================================
-- These functions keep the graph layer in sync with the
-- relational tables. When a relational record is created,
-- updated, or deleted, the corresponding graph node and
-- edges are maintained automatically.

-- Function to create/update a graph node when a relational entity changes
CREATE OR REPLACE FUNCTION sync_graph_node()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        TG_ARGV[0],     -- node_type passed as trigger argument
        NEW.id,
        COALESCE(NEW.display_name, NEW.name, NEW.po_number, NEW.invoice_number, NEW.payment_number, NEW.sku, NEW.code),
        jsonb_build_object(
            'status', COALESCE(NEW.status, 'active'),
            'type', COALESCE(NEW.party_type, NEW.product_type, NEW.invoice_type, NEW.payment_type, NULL)
        )
    )
    ON CONFLICT (tenant_id, node_type, entity_id)
    DO UPDATE SET
        label = EXCLUDED.label,
        properties = graph_node.properties || EXCLUDED.properties,
        updated_at = now();

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply triggers to key tables
-- CREATE TRIGGER trg_party_graph AFTER INSERT OR UPDATE ON party
--     FOR EACH ROW EXECUTE FUNCTION sync_graph_node('Party');
-- CREATE TRIGGER trg_product_graph AFTER INSERT OR UPDATE ON product
--     FOR EACH ROW EXECUTE FUNCTION sync_graph_node('Product');
-- CREATE TRIGGER trg_po_graph AFTER INSERT OR UPDATE ON purchase_order
--     FOR EACH ROW EXECUTE FUNCTION sync_graph_node('PurchaseOrder');
-- CREATE TRIGGER trg_invoice_graph AFTER INSERT OR UPDATE ON invoice
--     FOR EACH ROW EXECUTE FUNCTION sync_graph_node('Invoice');
-- CREATE TRIGGER trg_payment_graph AFTER INSERT OR UPDATE ON payment
--     FOR EACH ROW EXECUTE FUNCTION sync_graph_node('Payment');

-- Function to create a directed edge between two graph nodes
CREATE OR REPLACE FUNCTION create_graph_edge(
    p_tenant_id UUID,
    p_from_type TEXT,
    p_from_entity_id UUID,
    p_to_type TEXT,
    p_to_entity_id UUID,
    p_edge_type TEXT,
    p_properties JSONB DEFAULT '{}'
) RETURNS UUID AS $$
DECLARE
    v_from_node_id UUID;
    v_to_node_id UUID;
    v_edge_id UUID;
BEGIN
    SELECT id INTO v_from_node_id FROM graph_node
        WHERE tenant_id = p_tenant_id AND node_type = p_from_type AND entity_id = p_from_entity_id;
    SELECT id INTO v_to_node_id FROM graph_node
        WHERE tenant_id = p_tenant_id AND node_type = p_to_type AND entity_id = p_to_entity_id;

    IF v_from_node_id IS NULL OR v_to_node_id IS NULL THEN
        RAISE EXCEPTION 'Graph nodes not found for edge creation';
    END IF;

    INSERT INTO graph_edge (tenant_id, from_node_id, to_node_id, edge_type, properties)
    VALUES (p_tenant_id, v_from_node_id, v_to_node_id, p_edge_type, p_properties)
    RETURNING id INTO v_edge_id;

    -- Update node statistics
    INSERT INTO graph_node_stats (node_id, tenant_id, out_degree)
    VALUES (v_from_node_id, p_tenant_id, 1)
    ON CONFLICT (node_id)
    DO UPDATE SET out_degree = graph_node_stats.out_degree + 1, updated_at = now();

    INSERT INTO graph_node_stats (node_id, tenant_id, in_degree)
    VALUES (v_to_node_id, p_tenant_id, 1)
    ON CONFLICT (node_id)
    DO UPDATE SET in_degree = graph_node_stats.in_degree + 1, updated_at = now();

    RETURN v_edge_id;
END;
$$ LANGUAGE plpgsql;

-- Usage examples:
-- When a PO is created for a vendor:
-- SELECT create_graph_edge(
--     '<tenant-id>',
--     'Organisation', '<org-id>',        -- from
--     'Party', '<vendor-id>',            -- to
--     'PURCHASES_FROM',
--     '{"po_id": "<po-id>", "total": 6000.00}'::JSONB
-- );
--
-- When a payment is applied to an invoice:
-- SELECT create_graph_edge(
--     '<tenant-id>',
--     'Payment', '<payment-id>',         -- from
--     'Invoice', '<invoice-id>',         -- to
--     'PAID_BY',
--     '{"amount": 2706.25, "payment_date": "2026-05-12"}'::JSONB
-- );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | `graph_node`, `graph_edge`, `graph_node_stats` |
| Organisation & Multi-Tenancy | 2 | `tenant`, `organisation` |
| Users & RBAC | 5 | `app_user`, `role`, `permission`, `role_permission`, `user_role` |
| Chart of Accounts & GL | 4 | `account`, `fiscal_period`, `journal_entry`, `journal_line` |
| Parties | 1 | `party` |
| Products & Inventory | 4 | `product`, `warehouse`, `stock_level`, `stock_movement` |
| Purchasing | 2 | `purchase_order`, `purchase_order_line` |
| Sales | 2 | `sales_order`, `sales_order_line` |
| Invoicing | 2 | `invoice`, `invoice_line` |
| Payments & Banking | 4 | `bank_account`, `bank_transaction`, `payment`, `payment_allocation` |
| Reference Data | 3 | `currency`, `exchange_rate`, `tax_rate` |
| AI & Audit | 2 | `ai_interaction`, `audit_log` |
| **Total** | **34** | 31 relational + 3 graph |

---

## Key Design Decisions

1. **Graph layer as an overlay, not a replacement.** The relational tables handle all transactional CRUD (creating invoices, posting journal entries, receiving goods). The graph layer mirrors entity relationships and enables traversal queries that would be expensive or impossible with multi-table joins. This means the graph can be rebuilt from relational data at any time.

2. **Temporal edges with `valid_from`/`valid_to`.** Graph edges include temporal validity, enabling historical relationship queries ("who supplied this product in Q1 2025?"). The `UNIQUE INDEX ... WHERE valid_to IS NULL` constraint prevents duplicate active relationships while allowing historical records.

3. **Property graph over labeled property graph.** Both nodes and edges carry JSONB `properties` bags rather than fixed typed columns. This provides the flexibility to add domain-specific attributes to any relationship type (lead time on SUPPLIES edges, approval timestamp on APPROVED_BY edges) without schema changes.

4. **Graph maintenance via application-layer triggers.** Trigger functions automatically sync graph nodes when relational records are created or updated. Edge creation is handled via a stored function `create_graph_edge()` called from the application layer at key workflow points (PO creation, invoice posting, payment allocation).

5. **Node statistics table for analytics.** The `graph_node_stats` table maintains in-degree and out-degree counts, enabling fast queries like "which vendor has the most purchase orders?" or "which product is ordered from the most vendors?" without counting edges at query time.

6. **Supply chain analysis as a first-class capability.** The SUPPLIES edge type with properties (lead time, unit price, primary flag) enables supply chain risk queries (single-source detection, vendor concentration analysis) that are natural graph problems but extremely awkward as relational joins.

7. **GDPR data subject traversal.** The bidirectional graph traversal ("show me everything connected to this person") makes GDPR data subject access requests and right-to-erasure impact analysis straightforward — a recursive CTE traverses all connected nodes within a depth limit.

8. **Relational tables remain the source of truth.** The graph layer is a derived view. If graph data becomes inconsistent, it can be rebuilt by scanning all relational tables and re-creating nodes and edges. This eliminates the risk of graph-relational divergence causing data integrity issues.

9. **Document lifecycle traceability.** The graph naturally models the ERP document lifecycle: PurchaseOrder --ORDERED--> Product, PurchaseOrder --RECEIVED_AT--> Warehouse, Invoice --DERIVED_FROM--> PurchaseOrder, Payment --PAID_BY--> Invoice. This chain is traversable in a single recursive CTE, whereas the relational model requires multiple joins across different tables with different foreign key patterns.

10. **AI-ready graph structure.** Graph neural networks (GNNs) and knowledge graph embeddings can operate directly on the `graph_node`/`graph_edge` tables for anomaly detection (unusual payment patterns), recommendation (vendor suggestions for new products), and risk scoring (vendor reliability based on delivery history). The graph structure is a natural fit for these AI workloads.
