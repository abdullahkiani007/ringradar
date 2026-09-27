# Data Model — Fraud / AML Ring Explorer

> Companion to `hybrid-graph-db-plan.md`. This is the Phase-1 deliverable: the domain choice,
> the relational + graph schema, edge conventions, seed strategy, and the example queries that
> prove the model supports both SQL aggregates and graph traversals.

## Why fraud/AML (the domain decision — DECISIONS.md D0)

Chosen because multi-hop traversal is the **reason the product exists**, not a decoration:
"who is within N hops of a flagged account, sharing a device, phone, or address?" is a question
that is painful in pure SQL and natural on a graph. That makes the graph layer and the NL router
*load-bearing* — which is what makes the architecture story credible in interviews. It also has
the strongest hiring pull for 2027 (finance is graph-hungry: AML, sanctions, identity resolution).

**Accepted tradeoff:** transaction data must be generated. Mitigated by a small deterministic
seed script (versioned, reproducible — no hardcoded computed values).

---

## The core idea: entities + shared attributes + transactions

Fraud rings are detected through **shared attributes** (same device, phone, IP, address) and
**money flow** (transactions). So the graph has two kinds of edges:

- **Money edges:** account → transaction → account (directed, weighted by amount).
- **Link edges:** account ↔ shared attribute (device/phone/ip/address) — these are what expose
  a ring even when accounts never transact directly.

---

## Relational schema (source of truth)

```sql
-- ---------- ENTITIES ----------
CREATE TABLE customer (
    id            BIGSERIAL PRIMARY KEY,
    full_name     TEXT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    risk_flag     TEXT NOT NULL DEFAULT 'none'   -- none | watch | flagged
);

CREATE TABLE account (
    id            BIGSERIAL PRIMARY KEY,
    customer_id   BIGINT NOT NULL REFERENCES customer(id),
    account_no    TEXT NOT NULL UNIQUE,
    opened_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        TEXT NOT NULL DEFAULT 'active', -- active | frozen | closed
    balance_cents BIGINT NOT NULL DEFAULT 0
);

-- Shared attributes that link accounts (device, phone, ip, address, ...)
CREATE TABLE attribute (
    id            BIGSERIAL PRIMARY KEY,
    kind          TEXT NOT NULL,   -- device | phone | ip | address | email
    value         TEXT NOT NULL,
    UNIQUE (kind, value)
);

-- Which account has used which attribute (the link that exposes rings)
CREATE TABLE account_attribute (
    account_id    BIGINT NOT NULL REFERENCES account(id),
    attribute_id  BIGINT NOT NULL REFERENCES attribute(id),
    first_seen    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, attribute_id)
);

-- ---------- MONEY FLOW ----------
CREATE TABLE txn (
    id            BIGSERIAL PRIMARY KEY,
    src_account   BIGINT NOT NULL REFERENCES account(id),
    dst_account   BIGINT NOT NULL REFERENCES account(id),
    amount_cents  BIGINT NOT NULL CHECK (amount_cents > 0),
    occurred_at   TIMESTAMPTZ NOT NULL,
    channel       TEXT NOT NULL DEFAULT 'transfer' -- transfer | card | cash | wire
);
```

### Indexes that make traversal fast (learn: indexing for graph workloads)
```sql
CREATE INDEX idx_txn_src        ON txn(src_account);
CREATE INDEX idx_txn_dst        ON txn(dst_account);
CREATE INDEX idx_acct_customer  ON account(customer_id);
CREATE INDEX idx_aa_attribute   ON account_attribute(attribute_id);
CREATE INDEX idx_aa_account     ON account_attribute(account_id);
```

---

## The unified graph projection (the layer both the UI and the router traverse)

Rather than force the UI to know the domain tables, project everything into a canonical
node/edge shape. Two options — pick per DECISIONS.md **D1**:

**Option A (recommended to start): views over the real tables — zero dual-write.**
```sql
-- Nodes: accounts and attributes both become graph nodes
CREATE VIEW graph_node AS
    SELECT 'account:'||id   AS node_id, 'account'   AS node_type,
           account_no       AS label,  jsonb_build_object('status',status,'balance',balance_cents) AS props
    FROM account
  UNION ALL
    SELECT 'attr:'||id      AS node_id, kind        AS node_type,
           value            AS label,  '{}'::jsonb  AS props
    FROM attribute;

-- Edges: money edges (account->account) + link edges (account<->attribute)
CREATE VIEW graph_edge AS
    SELECT 'account:'||src_account AS from_id, 'account:'||dst_account AS to_id,
           'sent_money'            AS edge_type, amount_cents          AS weight,
           occurred_at             AS at
    FROM txn
  UNION ALL
    SELECT 'account:'||account_id  AS from_id, 'attr:'||attribute_id   AS to_id,
           'uses'                  AS edge_type, 1                     AS weight,
           first_seen              AS at
    FROM account_attribute;
```
> **Why views:** single source of truth, no sync bug, always consistent. The cost is that
> heavy traversals recompute the projection — fine at portfolio scale, and it's the honest
> tradeoff you defend in the design doc. Option B (materialized `nodes`/`edges` tables kept in
> sync by triggers, or Apache AGE) is the "at scale" answer you describe but don't necessarily build.

---

## The two query classes the NL router must distinguish (D2)

**Relational aggregate** — natural in SQL:
> "What's the total amount sent by flagged customers in the last 30 days?"
```sql
SELECT c.full_name, SUM(t.amount_cents)/100.0 AS total_sent
FROM txn t
JOIN account a ON a.id = t.src_account
JOIN customer c ON c.id = a.customer_id
WHERE c.risk_flag = 'flagged'
  AND t.occurred_at > now() - interval '30 days'
GROUP BY c.full_name
ORDER BY total_sent DESC;
```

**Graph traversal** — natural on the graph (recursive CTE, or Cypher via AGE):
> "Which accounts are within 2 hops of flagged account #42 through a shared device or money?"
```sql
WITH RECURSIVE reach(node_id, depth) AS (
    SELECT 'account:42'::text, 0
  UNION ALL
    SELECT e.to_id, r.depth + 1
    FROM reach r
    JOIN graph_edge e ON e.from_id = r.node_id
    WHERE r.depth < 2
)
SELECT DISTINCT node_id, MIN(depth) AS hops
FROM reach
WHERE node_id <> 'account:42'
GROUP BY node_id
ORDER BY hops;
```
> The router's job (Phase 3): read the question, decide which class it is, generate the right
> query, and — critically — the traversal result (`node_id` list) is exactly what Phase 4
> highlights on the visual graph. The model is designed so the answer *is* a set of node ids.

---

## Seed strategy (reproducible, no hardcoded values)

A `seed.py` (or `.mjs`) that:
1. Creates ~200 customers / ~300 accounts with mostly-benign behavior.
2. Plants **2–3 deliberate rings**: clusters of accounts sharing a device/phone/IP and cycling
   money among themselves, one seed account per ring set to `risk_flag='flagged'`.
3. Adds noise transactions so the ring isn't trivially obvious.

Keep the ring definitions in a versioned JSON so the "answer key" is reproducible and you can
demo "the tool found the ring I planted."

---

## Phase-1 gate (from the plan)

You are done with Phase 1 when the recursive-CTE traversal above returns the planted ring
members within 2 hops, and the relational aggregate returns correct per-customer totals — both
against seeded data. That proves the single model supports **both** query classes, which is the
entire premise of the project.
