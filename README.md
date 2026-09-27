# Hybrid Graph DB Project — Portfolio + Architecture Upskilling

A **queryable knowledge-graph explorer** layered on PostgreSQL: data stored relationally,
explored and edited as an interactive node graph, and questioned in natural language — with the
NL answer highlighted live on the graph.

**Domain:** fraud / AML ring explorer.
**Goals:** strengthen portfolio for 2027 hiring + upskill in system architecture.
**Status:** planning complete; Phase 1 (data model) designed, not yet implemented.

## Read in this order

1. [`conversation-context.md`](./conversation-context.md) — **start here.** Decision log +
   full reasoning so any session can resume with context.
2. [`hybrid-graph-db-plan.md`](./hybrid-graph-db-plan.md) — scoped architecture & the 4-phase
   learning plan (milestone + concept learned per phase).
3. [`hybrid-graph-db-datamodel.md`](./hybrid-graph-db-datamodel.md) — the Phase-1 data model:
   DDL, graph views, the two query classes, and the seed strategy.

## The one thing to protect
The "wow" moment: **ask a question in English → planner routes to SQL or graph traversal →
runs a read-only sandboxed query → the answer lights up on the live graph.** If scope must be
cut, cut breadth — never this loop.

## Next steps (see conversation-context.md → Open items)
- Reproducible seed script (planted rings + noise)
- Query-safety / sandboxing layer
- The ask→route→visualize NL loop
- Postgres-views vs Apache AGE tradeoff deep-dive
