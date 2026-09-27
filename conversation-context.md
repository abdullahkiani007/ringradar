# Conversation Context & Decision Log

> Purpose: capture the full reasoning of the planning conversation so a future session — or a
> fresh agent with none of this context — can resume the project without re-deciding settled
> questions. This is the "source of context" for the project. Append to it as decisions evolve.

**Project:** Queryable Knowledge-Graph Explorer (fraud/AML ring explorer)
**Owner:** vr1254
**Started:** 2026-09-27
**Location:** `C:\Users\vr1254\.kiro\crew\workspace\hybrid-graph-db-project\`

---

## The original idea (user's words, paraphrased)

A database that is a **hybrid of relational + graph**: data stored in tables, but the user
interacts with it in visual form as **nodes connected to each other**, and it **integrates NLP**
to answer user queries about the data — "a source of info for the system and AI."

Questions the user asked: is it worth building for landing a job in 2027, is it possible to
build, how does it benefit them, does it have real-life use cases.

## The user's actual goals (this drives every decision)

1. **Make their portfolio stronger.**
2. **Upskill in knowledge and system architecture.**

> This is NOT a startup or a product for market. It is a portfolio + learning project. Scope
> decisions are made to maximize depth of learning and hiring signal, NOT feature breadth.

---

## Key conclusions from the conversation (settled)

### Is the idea novel?
- Hybrid relational+graph store: **not novel** (Postgres+AGE, SurrealDB, ArangoDB, Neo4j,
  TigerGraph, Neptune all exist). Do NOT try to out-engineer these.
- Visual node interaction: moderately common (Retool, Neo4j Bloom, Gephi).
- NLP over data (text-to-SQL / "chat with your DB"): hot but crowded (LangChain SQL agents,
  Vanna.ai).
- **Verdict:** none of the three pieces is new, but the *combination as one coherent thing*
  (same data = table to the engine, live graph to the user, question to an LLM) is a genuinely
  strong **portfolio project**. Not a moat — but that's not what's needed for a job.

### Is it worth building / possible / beneficial?
- **Worth it: yes**, as a portfolio piece with scope cut hard. Touches in-demand skills:
  data modeling, query translation, graph algorithms, LLM integration, visualization.
- **Possible solo: yes** — as a *layer* on existing engines, ~3–4 months part-time. Do NOT
  write a storage engine.
- **Real use cases:** fraud/AML, knowledge graphs/MDM, OSINT/investigative journalism,
  recommendation/identity resolution, infra impact analysis.

### Architectural stance (settled)
- **Stand on PostgreSQL** as the single source of truth. Build a layer, not a DBMS.
- **Go DEEP on ONE layer:** the query-planning + safety layer (NL question → route relational
  vs graph → read-only sandboxed query → run → map result back to visual).
- **Lean on libraries** for storage, visualization (Cytoscape.js / React Flow), and the raw
  LLM call.
- The **"wow" moment** to protect above all: ask a question in English → planner routes →
  sandboxed query → **answer highlights live on the visual graph.**

### Domain decision (settled)
- **Chosen: fraud / AML ring explorer.** Reason: multi-hop traversal is the *reason the
  product exists*, making the graph + NL layers load-bearing (not decoration). Strongest 2027
  hiring pull. Accepted tradeoff: transaction data must be generated (reproducible seed script).
- Alternatives considered and rejected for now: org/knowledge graph, supply-chain, movie/actor.

### Data model decision (settled)
- Entities (`customer`, `account`) + shared `attribute` (device/phone/ip/address) +
  `txn` money flow.
- Two edge types: **money edges** (account→account) and **link edges** (account↔attribute) —
  the link edges expose rings even when accounts never transact directly.
- **Graph projection via VIEWS over the real tables (Option A)** to start: single source of
  truth, no dual-write bug. Materialized tables / Apache AGE is the documented "at scale"
  answer, not necessarily built.
- Traversal results are deliberately **sets of node ids**, so Phase-4 highlighting is a direct
  projection.
- Full DDL, indexes, the two example query classes (relational aggregate + recursive-CTE
  traversal), and the seed strategy are in `hybrid-graph-db-datamodel.md`.

---

## The 4 phases (from the plan)

1. **Data model & store** (2–3 wk) — Postgres schema + edges + seed. Gate: recursive-CTE
   traversal finds the planted ring; aggregate returns correct totals.
2. **Visual layer** (2–3 wk) — interactive graph, edits write back.
3. **NL query layer** (3–4 wk) — English → route (D2) → query → result.
4. **Closed loop + hardening** (3–4 wk, GO DEEP) — highlight answer on graph, read-only role,
   timeouts, query allowlist, incremental sync, + the design doc.

## The 5 decision points to log in DECISIONS.md as you build
- D1 native tables vs AGE vs separate graph DB (started: views, Option A)
- D2 NL routing: relational aggregate vs graph traversal
- D3 stopping the LLM from running destructive/expensive queries
- D4 keeping the visual graph in sync without full refetch
- D5 mapping query result → highlighted nodes/edges

---

## Open items / next steps discussed (not yet done)
- Write the reproducible seed script (planted rings + noise).
- Sketch the query-safety / sandboxing layer in detail.
- Design the ask→route→visualize NL loop in detail.
- Deep-dive the Postgres-views vs Apache AGE tradeoff.

## Files in this directory
- `README.md` — index of the project docs.
- `hybrid-graph-db-plan.md` — scoped architecture & phased learning plan.
- `hybrid-graph-db-datamodel.md` — fraud/AML data model (DDL, views, queries, seed strategy).
- `conversation-context.md` — this file (decision log + resume context).
