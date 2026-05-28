# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: SMB ERP Platform · Created: 2026-05-12

## Philosophy

The Event-Sourced / Audit-First model treats every business action as an immutable event appended to a write-ahead log. The current state of any entity (an invoice, a stock level, a bank balance) is derived by replaying its event stream from the beginning. Separate read-optimised projections (materialised views or denormalised tables) serve queries. This is the CQRS (Command Query Responsibility Segregation) pattern applied to ERP.

This approach mirrors how accounting actually works: an accountant never erases a ledger entry; corrections are made by appending compensating entries. The event store IS the audit trail. There is no separate `audit_log` table bolted on after the fact — every state change is an event, and the complete history of every entity is preserved by design. Square's "Books" system, Stripe's internal ledger, and most modern fintech platforms use this architecture because financial regulators require the ability to reconstruct state at any past point in time.

For an AI-native ERP, event sourcing has a second powerful advantage: the event stream is a rich, structured dataset for training and inference. An AI cash flow predictor can analyse the temporal pattern of `PaymentReceived`, `InvoiceIssued`, and `PurchaseOrderApproved` events far more effectively than it can query a flat table of current balances. The event store makes temporal queries ("what was the AP aging on March 15?") trivial, which is foundational for the proactive cash flow intelligence described in the project specification.

**Best for:** SMBs in regulated industries requiring complete audit trails, temporal queries ("what was the state on date X?"), and rich AI training data from historical event streams.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — no data loss, no retroactive edits
- (+) Temporal queries are trivial — replay events to any point in time
- (+) Rich AI training data — structured event streams are ideal for pattern recognition
- (+) Natural alignment with double-entry accounting — events never mutate, only append
- (+) Easy to add new projections — new reporting views without schema migration
- (-) Higher write amplification — every mutation is an event + projection update
- (-) Event schema evolution is complex — changing event shapes requires versioning
- (-) Read queries hit projections, not source of truth — eventual consistency between writes and reads
- (-) Steeper learning curve for developers unfamiliar with event sourcing
- (-) Rebuilding projections from scratch can be slow for large event stores

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GAAP / IFRS | Events naturally model the append-only nature of financial ledgers; replaying events produces GAAP/IFRS-compliant financial statements at any historical point |
| ISO 20022 | Payment events include ISO 20022-aligned structured fields; outbound payment events can be directly serialised to ISO 20022 PAIN.001 messages |
| ISO 3166-1/2 | Jurisdiction codes embedded in entity registration events |
| ISO 4217 | Currency codes in all monetary event payloads |
| GS1 / GTIN | Product registration events include GTIN identifiers |
| OASIS UBL 2.1 | Invoice and purchase order events map to UBL document lifecycle events |
| XBRL | Financial report projections tagged with XBRL taxonomy elements for regulatory filing |
| PEPPOL | Invoice events include PEPPOL participant IDs; `InvoiceSentViaPeppol` events track compliance |
| SOC 2 / ISO 27001 | Immutable event store inherently satisfies audit trail requirements for both certifications |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE — THE SINGLE SOURCE OF TRUTH
-- ============================================================
-- All business actions are captured as immutable events.
-- Current state is DERIVED from events, never stored directly
-- in the event store.

CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,       -- 'Organisation', 'Account', 'Invoice', 'PurchaseOrder',
                                         -- 'Product', 'StockLevel', 'Party', 'Payment', 'BankAccount'
    stream_id       UUID NOT NULL,       -- The aggregate/entity ID this event belongs to
    event_type      TEXT NOT NULL,       -- 'InvoiceCreated', 'PaymentReceived', 'StockAdjusted', etc.
    event_version   INTEGER NOT NULL,    -- Monotonically increasing per stream (optimistic concurrency)
    event_data      JSONB NOT NULL,      -- The full event payload
    metadata        JSONB NOT NULL DEFAULT '{}',
                                         -- {"user_id": "...", "ip": "...", "ai_generated": true,
                                         --  "correlation_id": "...", "causation_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Optimistic concurrency: no two events in the same stream can have the same version
    UNIQUE (tenant_id, stream_type, stream_id, event_version)
);

-- Primary query patterns: read all events for a stream, read events after a version
CREATE INDEX idx_event_stream ON event_store(tenant_id, stream_type, stream_id, event_version);

-- Global ordering for projections that need to process all events in order
CREATE INDEX idx_event_global ON event_store(tenant_id, created_at, id);

-- Event type filtering for analytics ("show me all PaymentReceived events this month")
CREATE INDEX idx_event_type ON event_store(tenant_id, event_type, created_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store PARTITION BY RANGE (created_at);
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

COMMENT ON TABLE event_store IS
'Immutable event store. The single source of truth for all business state.
Events are never updated or deleted. Corrections are modeled as compensating events.';
```

### Event Payload Examples

```sql
-- Example: InvoiceCreated event
-- event_type = 'InvoiceCreated'
-- event_data =
-- {
--   "invoice_number": "INV-2026-000142",
--   "invoice_type": "customer_invoice",
--   "party_id": "a1b2c3d4-...",
--   "party_name": "Acme Corp",
--   "org_id": "e5f6g7h8-...",
--   "invoice_date": "2026-05-12",
--   "due_date": "2026-06-11",
--   "currency_code": "USD",
--   "exchange_rate": 1.0,
--   "lines": [
--     {
--       "line_number": 1,
--       "product_id": "i9j0k1l2-...",
--       "description": "Widget Model A",
--       "quantity": 100,
--       "unit_price": 25.00,
--       "tax_rate": 8.25,
--       "line_total": 2500.00
--     }
--   ],
--   "subtotal": 2500.00,
--   "tax_amount": 206.25,
--   "total": 2706.25,
--   "payment_terms": "NET30"
-- }
-- metadata =
-- {
--   "user_id": "m3n4o5p6-...",
--   "ip": "192.168.1.42",
--   "ai_generated": false,
--   "correlation_id": "q7r8s9t0-..."
-- }

-- Example: PaymentReceived event
-- event_type = 'PaymentReceived'
-- event_data =
-- {
--   "payment_number": "PMT-2026-000089",
--   "payment_type": "incoming",
--   "payment_method": "bank_transfer",
--   "party_id": "a1b2c3d4-...",
--   "bank_account_id": "u1v2w3x4-...",
--   "amount": 2706.25,
--   "currency_code": "USD",
--   "reference": "Wire ref 4829103",
--   "allocations": [
--     {"invoice_id": "y5z6a7b8-...", "amount": 2706.25}
--   ]
-- }

-- Example: StockAdjusted event
-- event_type = 'StockAdjusted'
-- event_data =
-- {
--   "product_id": "i9j0k1l2-...",
--   "warehouse_id": "c9d0e1f2-...",
--   "adjustment_type": "receipt",
--   "quantity_change": 500,
--   "unit_cost": 12.00,
--   "reference_type": "purchase_order",
--   "reference_id": "g3h4i5j6-...",
--   "reason": "Goods receipt from Acme supplier"
-- }

-- Example: AICommandProcessed event (NL transaction creation)
-- event_type = 'AICommandProcessed'
-- event_data =
-- {
--   "input_text": "create a PO for 500 units of SKU-123 from Acme at $12 each, net-30",
--   "parsed_intent": "create_purchase_order",
--   "confidence_score": 0.97,
--   "result_stream_type": "PurchaseOrder",
--   "result_stream_id": "g3h4i5j6-...",
--   "model_version": "erp-nl-v2.1"
-- }
```

## Command Handlers (Application Layer)

```sql
-- ============================================================
-- COMMAND PROCESSING SUPPORT
-- ============================================================

-- Idempotency: prevent duplicate command processing
CREATE TABLE processed_commands (
    command_id      UUID PRIMARY KEY,               -- Client-generated idempotency key
    tenant_id       UUID NOT NULL,
    command_type    TEXT NOT NULL,                   -- 'CreateInvoice', 'ApproveOrder', etc.
    result_event_id UUID REFERENCES event_store(id),
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_processed_cmd_tenant ON processed_commands(tenant_id, processed_at);

-- Snapshot store: avoid replaying full event history for long-lived aggregates
CREATE TABLE event_snapshot (
    tenant_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,               -- Event version at snapshot time
    snapshot_data   JSONB NOT NULL,                  -- Serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, stream_type, stream_id)
);

COMMENT ON TABLE event_snapshot IS
'Periodic snapshots of aggregate state to avoid replaying full event history.
Snapshots are rebuilt by replaying events from version 0 to snapshot_version.
New events after snapshot_version are replayed on top of the snapshot.';
```

## Read Projections (Query Side)

```sql
-- ============================================================
-- READ PROJECTIONS — MATERIALISED FROM EVENTS
-- ============================================================
-- These tables are the "query side" of CQRS. They are populated
-- by projection handlers that consume events from the event store.
-- They can be rebuilt from scratch at any time by replaying all events.

-- Projection tracking: which events have been processed by each projection
CREATE TABLE projection_checkpoint (
    projection_name TEXT NOT NULL,
    tenant_id       UUID NOT NULL,
    last_event_id   UUID NOT NULL REFERENCES event_store(id),
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, tenant_id)
);

-- ---- Organisation Projection ----

CREATE TABLE proj_organisation (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    parent_org_id   UUID,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    lei             TEXT,
    country_code    CHAR(2) NOT NULL,
    jurisdiction    TEXT,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    fiscal_year_start_month SMALLINT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_org_tenant ON proj_organisation(tenant_id);

-- ---- Account (Chart of Accounts) Projection ----

CREATE TABLE proj_account (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    org_id          UUID NOT NULL,
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL,
    parent_id       UUID,
    account_standard TEXT DEFAULT 'GAAP',
    xbrl_element    TEXT,
    is_reconcilable BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    current_balance NUMERIC(19,4) NOT NULL DEFAULT 0,  -- Running balance, updated on each journal event
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_account_org ON proj_account(tenant_id, org_id);

-- ---- Party Projection ----

CREATE TABLE proj_party (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    org_id          UUID NOT NULL,
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
    country_code    CHAR(2),
    email           TEXT,
    phone           TEXT,
    payment_terms_days SMALLINT DEFAULT 30,
    credit_limit    NUMERIC(19,4),
    currency_code   CHAR(3) DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Denormalised aggregates (updated on payment/invoice events)
    total_invoiced  NUMERIC(19,4) NOT NULL DEFAULT 0,
    total_paid      NUMERIC(19,4) NOT NULL DEFAULT 0,
    outstanding_balance NUMERIC(19,4) NOT NULL DEFAULT 0,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_party_tenant ON proj_party(tenant_id, org_id);
CREATE INDEX idx_proj_party_type ON proj_party(tenant_id, party_type);

-- ---- Invoice Projection ----

CREATE TABLE proj_invoice (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    org_id          UUID NOT NULL,
    invoice_number  TEXT NOT NULL,
    invoice_type    TEXT NOT NULL,
    party_id        UUID NOT NULL,
    party_name      TEXT NOT NULL,                  -- Denormalised for fast listing
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(19,4) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(19,4) NOT NULL DEFAULT 0,
    days_overdue    INTEGER GENERATED ALWAYS AS (
        CASE WHEN status NOT IN ('paid', 'cancelled') AND due_date < CURRENT_DATE
             THEN CURRENT_DATE - due_date ELSE 0 END
    ) STORED,
    line_items      JSONB NOT NULL DEFAULT '[]',    -- Denormalised lines for fast read
    peppol_sent     BOOLEAN NOT NULL DEFAULT FALSE,
    edi_sent        BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_invoice_tenant ON proj_invoice(tenant_id, org_id);
CREATE INDEX idx_proj_invoice_party ON proj_invoice(tenant_id, party_id);
CREATE INDEX idx_proj_invoice_status ON proj_invoice(tenant_id, status);
CREATE INDEX idx_proj_invoice_overdue ON proj_invoice(tenant_id, due_date)
    WHERE status NOT IN ('paid', 'cancelled');

-- ---- Purchase Order Projection ----

CREATE TABLE proj_purchase_order (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    org_id          UUID NOT NULL,
    po_number       TEXT NOT NULL,
    vendor_id       UUID NOT NULL,
    vendor_name     TEXT NOT NULL,
    order_date      DATE NOT NULL,
    expected_date   DATE,
    status          TEXT NOT NULL DEFAULT 'draft',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(19,4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(19,4) NOT NULL DEFAULT 0,
    total           NUMERIC(19,4) NOT NULL DEFAULT 0,
    line_items      JSONB NOT NULL DEFAULT '[]',
    ai_created      BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_po_tenant ON proj_purchase_order(tenant_id, org_id);
CREATE INDEX idx_proj_po_vendor ON proj_purchase_order(tenant_id, vendor_id);
CREATE INDEX idx_proj_po_status ON proj_purchase_order(tenant_id, status);

-- ---- Stock Level Projection ----

CREATE TABLE proj_stock_level (
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    warehouse_id    UUID NOT NULL,
    product_sku     TEXT NOT NULL,
    product_name    TEXT NOT NULL,
    quantity_on_hand NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_reserved NUMERIC(12,3) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(12,3) NOT NULL DEFAULT 0,
    avg_unit_cost   NUMERIC(19,4) NOT NULL DEFAULT 0,
    total_value     NUMERIC(19,4) NOT NULL DEFAULT 0,
    last_movement_at TIMESTAMPTZ,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id, warehouse_id)
);

CREATE INDEX idx_proj_stock_warehouse ON proj_stock_level(tenant_id, warehouse_id);

-- ---- Cash Flow Projection (AI Analytics) ----

CREATE TABLE proj_cash_flow_daily (
    tenant_id       UUID NOT NULL,
    org_id          UUID NOT NULL,
    flow_date       DATE NOT NULL,
    inflows         NUMERIC(19,4) NOT NULL DEFAULT 0,   -- Total payments received
    outflows        NUMERIC(19,4) NOT NULL DEFAULT 0,   -- Total payments made
    net_flow        NUMERIC(19,4) NOT NULL DEFAULT 0,   -- inflows - outflows
    running_balance NUMERIC(19,4) NOT NULL DEFAULT 0,   -- Cumulative balance
    ar_outstanding  NUMERIC(19,4) NOT NULL DEFAULT 0,   -- Total AR at end of day
    ap_outstanding  NUMERIC(19,4) NOT NULL DEFAULT 0,   -- Total AP at end of day
    invoice_count   INTEGER NOT NULL DEFAULT 0,
    payment_count   INTEGER NOT NULL DEFAULT 0,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, org_id, flow_date)
);

COMMENT ON TABLE proj_cash_flow_daily IS
'Daily cash flow aggregation built from PaymentReceived and PaymentMade events.
Used by the AI cash flow predictor for 30/60/90-day forecasting.
Can be rebuilt from scratch by replaying all payment events.';

-- ---- Bank Reconciliation Projection ----

CREATE TABLE proj_bank_transaction (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    bank_account_id UUID NOT NULL,
    transaction_date DATE NOT NULL,
    amount          NUMERIC(19,4) NOT NULL,
    description     TEXT,
    counterparty_name TEXT,
    is_reconciled   BOOLEAN NOT NULL DEFAULT FALSE,
    matched_type    TEXT,
    matched_id      UUID,
    match_method    TEXT,
    confidence_score NUMERIC(3,2),
    ai_explanation  TEXT,
    version         INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_bank_txn_account ON proj_bank_transaction(tenant_id, bank_account_id);
CREATE INDEX idx_proj_bank_txn_unreconciled ON proj_bank_transaction(tenant_id, bank_account_id)
    WHERE is_reconciled = FALSE;
```

## Temporal Queries

```sql
-- ============================================================
-- TEMPORAL QUERY EXAMPLES
-- ============================================================

-- "What was the balance of account 1000 on March 15, 2026?"
-- Replay all JournalEntryPosted events up to that date:
--
-- SELECT
--     SUM(
--         CASE WHEN line->>'account_id' = '<account-id>'
--              THEN (line->>'debit')::NUMERIC - (line->>'credit')::NUMERIC
--              ELSE 0
--         END
--     ) AS balance_at_date
-- FROM event_store,
--      jsonb_array_elements(event_data->'lines') AS line
-- WHERE tenant_id = '<tenant>'
--   AND event_type = 'JournalEntryPosted'
--   AND created_at <= '2026-03-15 23:59:59+00'
-- ;

-- "What was the AP aging on April 1, 2026?"
-- Replay InvoiceCreated and PaymentReceived events up to that date:
--
-- WITH invoices_at_date AS (
--     SELECT
--         stream_id,
--         event_data->>'party_id' AS vendor_id,
--         (event_data->>'total')::NUMERIC AS total,
--         (event_data->>'due_date')::DATE AS due_date,
--         event_data->>'invoice_number' AS invoice_number
--     FROM event_store
--     WHERE tenant_id = '<tenant>'
--       AND event_type = 'InvoiceCreated'
--       AND event_data->>'invoice_type' = 'vendor_bill'
--       AND created_at <= '2026-04-01 23:59:59+00'
-- ),
-- payments_at_date AS (
--     SELECT
--         alloc->>'invoice_id' AS invoice_id,
--         SUM((alloc->>'amount')::NUMERIC) AS paid
--     FROM event_store,
--          jsonb_array_elements(event_data->'allocations') AS alloc
--     WHERE tenant_id = '<tenant>'
--       AND event_type = 'PaymentMade'
--       AND created_at <= '2026-04-01 23:59:59+00'
--     GROUP BY alloc->>'invoice_id'
-- )
-- SELECT
--     i.vendor_id,
--     i.invoice_number,
--     i.total - COALESCE(p.paid, 0) AS outstanding,
--     '2026-04-01'::DATE - i.due_date AS days_overdue
-- FROM invoices_at_date i
-- LEFT JOIN payments_at_date p ON p.invoice_id = i.stream_id::TEXT
-- WHERE i.total - COALESCE(p.paid, 0) > 0
-- ORDER BY days_overdue DESC;
```

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REFERENCE
-- ============================================================
-- This table documents all known event types. It is a reference
-- table, not enforced as a FK constraint on event_store (to allow
-- new event types without migration).

CREATE TABLE event_type_catalogue (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    payload_schema  JSONB,                          -- JSON Schema for the event_data field
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core event types:
INSERT INTO event_type_catalogue (event_type, stream_type, description) VALUES
-- Organisation
('OrganisationCreated', 'Organisation', 'A new legal entity/organisation was registered'),
('OrganisationUpdated', 'Organisation', 'Organisation details were modified'),
-- Account
('AccountCreated', 'Account', 'A new GL account was added to the chart of accounts'),
('AccountDeactivated', 'Account', 'A GL account was deactivated'),
-- Journal
('JournalEntryDrafted', 'JournalEntry', 'A journal entry was created in draft status'),
('JournalEntryPosted', 'JournalEntry', 'A journal entry was posted to the general ledger'),
('JournalEntryReversed', 'JournalEntry', 'A posted journal entry was reversed via a compensating entry'),
-- Party
('PartyCreated', 'Party', 'A new customer, vendor, or employee was registered'),
('PartyUpdated', 'Party', 'Party details were modified'),
('PartyDeactivated', 'Party', 'A party was deactivated'),
-- Product
('ProductCreated', 'Product', 'A new product/service was added to the catalogue'),
('ProductUpdated', 'Product', 'Product details were modified'),
-- Stock
('StockReceived', 'StockLevel', 'Goods were received into a warehouse'),
('StockShipped', 'StockLevel', 'Goods were shipped from a warehouse'),
('StockTransferred', 'StockLevel', 'Goods were transferred between warehouses'),
('StockAdjusted', 'StockLevel', 'Manual stock adjustment (count correction, damage, etc.)'),
('StockReserved', 'StockLevel', 'Stock was reserved for a sales order'),
('StockReservationReleased', 'StockLevel', 'A stock reservation was released'),
-- Purchase Order
('PurchaseOrderCreated', 'PurchaseOrder', 'A new purchase order was created'),
('PurchaseOrderSubmitted', 'PurchaseOrder', 'A PO was submitted for approval'),
('PurchaseOrderApproved', 'PurchaseOrder', 'A PO was approved'),
('PurchaseOrderReceived', 'PurchaseOrder', 'Goods were received against a PO'),
('PurchaseOrderCancelled', 'PurchaseOrder', 'A PO was cancelled'),
-- Sales Order
('SalesOrderCreated', 'SalesOrder', 'A new sales order was created'),
('SalesOrderConfirmed', 'SalesOrder', 'A sales order was confirmed'),
('SalesOrderShipped', 'SalesOrder', 'Goods were shipped for a sales order'),
('SalesOrderCancelled', 'SalesOrder', 'A sales order was cancelled'),
-- Invoice
('InvoiceCreated', 'Invoice', 'A new invoice or vendor bill was created'),
('InvoiceSent', 'Invoice', 'An invoice was sent to the customer'),
('InvoiceSentViaPeppol', 'Invoice', 'An invoice was transmitted via the PEPPOL network'),
('InvoiceSentViaEdi', 'Invoice', 'An invoice was transmitted via EDI (X12/EDIFACT)'),
('InvoicePartiallyPaid', 'Invoice', 'A partial payment was applied to an invoice'),
('InvoicePaid', 'Invoice', 'An invoice was fully paid'),
('InvoiceCancelled', 'Invoice', 'An invoice was cancelled'),
('CreditNoteIssued', 'Invoice', 'A credit note was issued against an invoice'),
-- Payment
('PaymentReceived', 'Payment', 'An incoming payment was received'),
('PaymentMade', 'Payment', 'An outgoing payment was made'),
('PaymentVoided', 'Payment', 'A payment was voided'),
-- Bank
('BankTransactionImported', 'BankAccount', 'A bank transaction was imported (Plaid/CSV/OFX)'),
('BankTransactionReconciled', 'BankAccount', 'A bank transaction was matched to an internal record'),
('BankReconciliationSuggested', 'BankAccount', 'AI suggested a reconciliation match'),
-- AI
('AICommandProcessed', 'AIInteraction', 'A natural-language command was processed by the AI layer'),
('AIReconciliationSuggested', 'AIInteraction', 'AI suggested a bank reconciliation match'),
('AICashFlowForecastGenerated', 'AIInteraction', 'AI generated a cash flow forecast'),
-- Fiscal
('FiscalPeriodOpened', 'FiscalPeriod', 'A new fiscal period was opened'),
('FiscalPeriodClosed', 'FiscalPeriod', 'A fiscal period was closed');
```

## Multi-Tenancy & User Management

```sql
-- ============================================================
-- TENANT & USER (REFERENCE TABLES — NOT EVENT-SOURCED)
-- ============================================================
-- These tables are mutable reference data, not event-sourced,
-- because they are infrastructure concerns, not business domain.

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_plan TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    org_id          UUID NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, org_id)
);

-- Reference data
CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,
    name            TEXT NOT NULL,
    symbol          TEXT NOT NULL,
    decimal_places  SMALLINT NOT NULL DEFAULT 2
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
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | `event_store` — the single source of truth |
| Command Infrastructure | 2 | `processed_commands`, `event_snapshot` |
| Projection Infrastructure | 1 | `projection_checkpoint` |
| Read Projections | 8 | `proj_organisation`, `proj_account`, `proj_party`, `proj_invoice`, `proj_purchase_order`, `proj_stock_level`, `proj_cash_flow_daily`, `proj_bank_transaction` |
| Reference Catalogue | 1 | `event_type_catalogue` |
| Tenant & Users (mutable) | 6 | `tenant`, `app_user`, `role`, `permission`, `role_permission`, `user_role` |
| Reference Data (mutable) | 2 | `currency`, `tax_rate` |
| **Total** | **21** | Plus projections can be added without schema migration |

---

## Key Design Decisions

1. **Single event store table for all aggregate types.** Rather than a separate event table per entity, all events flow into one table partitioned by `created_at`. This simplifies infrastructure, enables cross-aggregate analytics, and makes global ordering straightforward. The `stream_type` + `stream_id` composite identifies which aggregate an event belongs to.

2. **Optimistic concurrency via `event_version`.** The `UNIQUE (tenant_id, stream_type, stream_id, event_version)` constraint ensures that two concurrent commands on the same aggregate will not both succeed — the second will receive a constraint violation and must retry. This replaces pessimistic locking.

3. **Projections are disposable.** Every `proj_*` table can be dropped and rebuilt from scratch by replaying the event store. This means new reporting needs can be met by adding new projections without modifying the source of truth. The `projection_checkpoint` table tracks which events each projection has consumed.

4. **Corrections as compensating events, not mutations.** If an invoice amount is wrong, the correction is modeled as an `InvoiceCancelled` event followed by a new `InvoiceCreated` event — never an UPDATE to the original event. This mirrors how real-world accounting corrections work.

5. **Event metadata captures AI provenance.** The `metadata` JSONB field on every event includes `ai_generated`, `correlation_id`, and `causation_id`, enabling full traceability from a natural-language command through to the resulting financial events. This is critical for the AI-native ERP's compliance story.

6. **Cash flow projection is purpose-built for AI.** The `proj_cash_flow_daily` table aggregates daily inflows, outflows, and running balances — precisely the format an AI cash flow predictor needs for 30/60/90-day forecasting. This projection rebuilds automatically from payment events.

7. **Event schema versioning.** The `event_type_catalogue` table tracks a `schema_version` for each event type and an optional `payload_schema` (JSON Schema). When event shapes evolve, projection handlers must support all historical versions — the events themselves are never modified.

8. **Reference data is NOT event-sourced.** Tenant, user, role, currency, and tax rate tables use standard mutable CRUD. Event sourcing is reserved for business domain aggregates where audit trails and temporal queries provide value. Over-applying event sourcing to infrastructure tables adds complexity without benefit.

9. **Snapshot store for performance.** Long-lived aggregates (organisations with thousands of events) can use periodic snapshots. The `event_snapshot` table stores a serialised aggregate state at a known event version; reconstruction loads the snapshot then replays only events after that version.

10. **Time-range partitioning ready.** The event store schema includes commented-out partition definitions. For production deployments exceeding millions of events, monthly partitioning by `created_at` enables efficient pruning and archival of old event data.
