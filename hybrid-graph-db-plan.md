# Queryable Knowledge-Graph Explorer — Scoped Architecture & Phased Learning Plan

> **What this is:** a portfolio-grade project layered on PostgreSQL that stores data
> relationally, lets a user explore and edit it as an interactive node graph, and answers
> natural-language questions by routing them to SQL or graph traversals and lighting the
> answer up on the graph.
>
> **Two goals this serves:** (1) a strong, defensible portfolio piece for hiring in 2027;
> (2) deliberate system-architecture upskilling via real decision points you can defend in
> interviews.
>
> **What this is NOT:** a new database engine, a startup, or a generic "hybrid DB". You are
> building a *layer*, not a DBMS. Scope discipline is the whole game.

---

## 0. Guiding principles (read before building)

1. **Stand on existing engines.** Postgres is the single source of truth. Do not write a
   storage engine — it is a multi-year distraction that teaches you less than the layer above.
2. **Go deep in ONE place.** The query-planning + safety layer is where the real engineering
   and the best interview stories live. Lean on libraries for storage, visualization, and the
   raw LLM call.
3. **One domain, not generic.** A generic "hybrid DB" demos poorly. A named domain
   ("fraud-ring explorer you can talk to") demos brilliantly and gives you a narrative.
4. **The design doc is a deliverable, equal to the code.** The architecture diagram + written
   tradeoff rationale is the artifact that proves you think like an architect.
5. **Every phase ends in (a) a demonstrable milestone and (b) a named concept you learned.**

---

## 1. The decision points that make this an *architecture* project

These are the things you will be asked about in a system-design interview. Building this means
you will have lived answers, not memorized ones. Keep a running `DECISIONS.md` and write your
rationale for each as you hit it.

| # | Decision | Why it's a senior-signal topic |
|---|----------|-------------------------------|
| D1 | Store the graph as native tables, a Postgres extension (AGE), or a separate graph DB synced to Postgres | Data consistency / dual-write / single-source-of-truth |
| D2 | How to decide a NL question is a SQL aggregate vs. a graph traversal | Query routing / planning |
| D3 | How to stop the LLM from running destructive or expensive queries | Safety, sandboxing, read-only boundaries, cost guards |
| D4 | How to keep the visual graph in sync with the tables without re-fetching everything | Caching, diffing, real-time updates |
| D5 | How to map a query result back onto graph nodes/edges to "highlight the answer" | Result-to-view projection, the closed loop |

---

## 2. Recommended stack (per layer)

| Layer | Choice | Rationale (short) |
|-------|--------|-------------------|
| **Store** | PostgreSQL + Apache AGE **or** a plain `nodes`/`edges` table pair | One instance serves relational + graph. Start with plain tables (simpler, teaches the model); adopt AGE only if you want to write Cypher. |
| **Backend** | Python (FastAPI) or Node (Express/Nest) | Pick the one you interview in. FastAPI pairs naturally with the LLM/data ecosystem. |
| **Graph view** | React + Cytoscape.js *or* React Flow | Cytoscape = graph-theory strong (layouts, algorithms). React Flow = editing/UX strong. Pick Cytoscape if the domain is analytical. |
| **NL layer** | LLM via REST API (OpenAI/Gemini/Anthropic), schema-grounded prompt | No SDK lock-in needed; you feed schema + few-shot examples and get SQL/Cypher back. |
| **Query safety** | Read-only DB role + statement timeout + allowlist parser | This is the layer you go DEEP on. |

> **Depth vs. lean:** GO DEEP on the query-planning + safety layer. LEAN on storage,
> visualization, and the raw LLM call.

---

## 3. Pick the domain first (do this before phase 1)

Choose ONE. Each has a ready hiring narrative and real-world use cases:

- **Fraud / AML ring explorer** — accounts + transactions; "who is 2 hops from this flagged
  account?" Strong, visual, industry-relevant.
- **Org / knowledge graph** — people, teams, projects, documents; feeds the RAG narrative.
- **Supply-chain / dependency graph** — "what breaks if this supplier/service fails?"
- **Movie/actor or codebase graph** — easy sample data, good for a first cut.

Recommendation: **fraud-ring explorer** for maximum "wow", or **movie/actor graph** if you want
a low-friction first build and will swap the domain later.

---

## 4. Phased plan — each phase = milestone + concept learned

### Phase 1 — Data model & store  *(≈ 2–3 weeks)*
- **Build:** model the chosen domain in Postgres with relational tables + an `edges` table
  (`from_id`, `to_id`, `edge_type`, `props JSONB`). Seed with realistic sample data.
- **Learn:** schema design, when a graph beats a join, indexing edges for traversal.
- **Deliverable:** an ERD diagram + a `DECISIONS.md` entry for **D1** (why plain tables vs. AGE
  vs. separate graph DB).
- **Gate:** you can answer "give me all X within N hops of Y" with a recursive SQL CTE.

### Phase 2 — Visual layer  *(≈ 2–3 weeks)*
- **Build:** render rows/edges as an interactive graph; clicking a node shows its row;
  editing writes back to Postgres.
- **Learn:** mapping relational data ↔ graph representation, frontend state sync.
- **Deliverable:** the live interactive-graph demo (record a GIF for the README).
- **Gate:** an edit in the UI is visible in the table, and vice versa, without a full reload.

### Phase 3 — Natural-language query layer  *(≈ 3–4 weeks)*
- **Build:** English question → LLM (given schema + few-shot examples) → query → result table.
- **Learn:** schema-grounded prompting, text-to-SQL, the **routing decision (D2)** — relational
  aggregate vs. graph traversal.
- **Deliverable:** the ask-a-question feature returning correct results on 10+ sample questions.
- **Gate:** at least one question routes to a graph traversal and one to a SQL aggregate,
  correctly, chosen by your router — not hardcoded.

### Phase 4 — The closed loop + hardening  *(≈ 3–4 weeks — GO DEEP HERE)*
- **Build:** the answer highlights on the live graph (D5). Harden the query path: read-only DB
  role, statement timeout, a query allowlist/parser that rejects writes and expensive scans (D3),
  and incremental graph sync so you don't refetch everything (D4).
- **Learn:** the safety/consistency engineering that separates a toy from a system.
- **Deliverable:** the **"wow" interaction** (ask → query → visual answer) + a complete
  **design doc**: architecture diagram, all five decisions with rationale, and a
  "what I'd change at scale" section.
- **Gate:** a malicious or destructive NL prompt cannot mutate or hang the database.

---

## 5. The "wow" moment (protect this above all)

> Natural-language question → the planner decides relational vs. traversal → generates a
> **read-only, sandboxed** query → runs it → **highlights the answer live on the visual graph.**

Almost no portfolio has this closed loop. It is the 30-second demo that gets you the callback.
If you cut scope under time pressure, cut breadth (fewer domains, fewer question types) — never
cut this loop.

---

## 6. What goes in the repo (the hiring artifacts)

- `README.md` with the GIF of the wow moment at the top.
- `docs/architecture.md` — the diagram + the five decisions with rationale.
- `docs/DECISIONS.md` — an ADR-style log written as you go (not after).
- Clean, tested query-safety layer — the part you go deep on.
- A "run it yourself" quickstart (docker-compose up).

---

## 7. Interview talking points this project earns you

- "I chose a single Postgres source of truth over a synced graph DB to avoid dual-write
  inconsistency — here's the tradeoff I accepted." (D1)
- "My NL layer routes to SQL or graph based on the question shape; here's how I keep the LLM
  from generating unsafe queries." (D2, D3)
- "I project query results back onto the visual layer with an incremental diff instead of a
  full refetch." (D4, D5)

These are architecture answers grounded in something you actually built.

---

## 8. Timeline summary

| Phase | Focus | Rough time |
|-------|-------|-----------|
| 1 | Data model & store | 2–3 wk |
| 2 | Visual layer | 2–3 wk |
| 3 | NL query layer | 3–4 wk |
| 4 | Closed loop + hardening + design doc | 3–4 wk |

**Total: ~3–4 months part-time**, ending with one deep, defensible project rather than three
shallow ones.
