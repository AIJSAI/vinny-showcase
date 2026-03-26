# Technical Decisions — Vinny

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-004: 512-Dimensional Embeddings

**Status**: Accepted  
**Context**: Supabase free tier imposes storage limits. The default output of `text-embedding-3-large` is 1536 dimensions per vector. With 107K+ wine records, each embedding row consumes significant storage, and the HNSW index memory footprint grows proportionally.

**Decision**: Use OpenAI's native `dimensions: 512` parameter on `text-embedding-3-large` to reduce embedding size by 3x. Rebuild the HNSW index (migrated from `ivfflat` to `hnsw` for better recall at lower probe counts).

**Consequences**:
- 3x storage reduction — fits comfortably within free-tier limits.
- Minimal recall degradation: measured via evaluation framework (ADR-008), precision@5 dropped <2%.
- All downstream consumers (search functions, reranking pipeline) updated to 512-dim.
- Cannot mix 1536-dim and 512-dim vectors — migration required full re-embedding.

---

## ADR-007: Hybrid Search (pgvector + tsvector + RRF)

**Status**: Accepted  
**Context**: Pure vector search fails on exact-match queries. A user searching "2019 Caymus Cabernet Sauvignon" expects that exact wine, but vector similarity may rank semantically similar wines higher. Conversely, pure keyword search fails on conceptual queries like "bold Italian red under $30."

**Decision**: Implement a hybrid search pipeline:
1. **pgvector** cosine similarity search (semantic understanding)
2. **tsvector** full-text search (exact keyword matching, with stemming and ranking)
3. **Reciprocal Rank Fusion** (RRF, k=60) to merge and de-duplicate results from both paths
4. **Cohere rerank-v3.5** to re-score the fused candidate list by query relevance

Both search paths execute in parallel via a single Supabase RPC function that returns the fused, reranked results.

**Consequences**:
- Exact wine name queries now reliably surface the correct wine.
- Semantic queries ("something like Barolo but cheaper") still work via vector path.
- Marginal latency increase (~50ms) from dual search + reranking — acceptable for conversational UX.
- RRF k=60 parameter tuned via evaluation framework to balance recall vs. precision.

---

## ADR-008: Automated Evaluation Framework

**Status**: Accepted  
**Context**: Search quality is critical — bad recommendations destroy user trust instantly. Manual testing doesn't scale and can't catch regressions when changing embedding dimensions, search parameters, or reranking models.

**Decision**: Build an automated evaluation suite with:
- Test query corpus (exact-match, semantic, cross-category, edge cases)
- Expected result sets per query
- Scored metrics: precision@k, recall@k, MRR (Mean Reciprocal Rank)
- Regression gates: CI blocks merges if metrics drop below thresholds

**Consequences**:
- ADR-004 (512-dim migration) validated via evaluation before deployment.
- Search parameter changes (RRF k, rerank model, embedding model) can be A/B tested quantitatively.
- Test corpus maintenance required as wine catalog evolves.

---

## ADR-009: Multi-Tenant Data Model

**Status**: Accepted  
**Context**: B2B SaaS model requires per-restaurant data isolation. Each restaurant has its own wine catalog, pricing, and customer interactions. A shared database with row-level filtering must prevent cross-tenant data leakage at every layer.

**Decision**:
- `tenant_id` column on all tenant-scoped tables (wines, conversations, analytics)
- Supabase Row-Level Security (RLS) policies on all tenant tables
- Connection-level tenant context via `set_config('app.tenant_id', ...)` at request start
- Hybrid search function applies `tenant_id` filter *within* the search query, not as a post-filter (critical for pgvector HNSW which returns candidates before WHERE clauses)

**Consequences**:
- Database-level isolation — even raw SQL access can't cross tenant boundaries with RLS enabled.
- HNSW index is global (not per-tenant) — acceptable for current scale; per-tenant indexes evaluated if catalog sizes diverge significantly.
- All API routes must set tenant context before any database operation.

---

## ADR-011: Consumer Anonymous Access

**Status**: Accepted  
**Context**: Requiring authentication before trying a wine recommendation tool creates unnecessary friction. Most users want to ask one or two questions before deciding if the tool is useful.

**Decision**: Allow anonymous guest access with:
- Rate limiting via Upstash Redis (X queries per hour per IP)
- No conversation persistence for anonymous users
- Soft conversion prompts after N interactions ("Save your preferences — create an account")
- Full access (history, preferences, saved wines) behind optional authentication

**Consequences**:
- Lower friction → higher trial rate.
- Rate limiting prevents abuse without authentication.
- Anonymous usage data (aggregated, not per-user) feeds evaluation framework.

---

## ADR-012: Staff Mode

**Status**: Accepted  
**Context**: Restaurant staff need capabilities beyond what customers use — inventory management, analytics dashboards, catalog curation. These features must be access-controlled to prevent customers from modifying restaurant data.

**Decision**: Implement role-based staff mode:
- `role` column on user profiles: `guest`, `customer`, `staff`, `admin`
- Staff-only API routes gated by role middleware
- UI conditionally renders staff features (inventory search, analytics, bulk operations)
- Staff actions logged to audit trail

**Consequences**:
- Single codebase serves both customer and staff experiences.
- Role escalation requires admin approval (not self-service).
- Staff features developed incrementally without affecting customer UX.
