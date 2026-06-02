# Technical Decisions: Vinny

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-004: Embedding Dimensions (512-dim, later halfvec(1536))

**Status**: Accepted, later revised  
**Context**: Supabase free tier imposes storage limits. `text-embedding-3-small` produces 1536 dimensions per vector natively. Across the wine catalog, each embedding row consumes significant storage, and the HNSW index memory footprint grows proportionally with dimensionality.

**Decision**: Initially used OpenAI's native `dimensions: 512` parameter on `text-embedding-3-small` to reduce embedding size by 3x. The catalog was later re-embedded at the full native 1536 dimensions, stored as Postgres `halfvec(1536)`: half-precision (2 bytes per component) keeps the larger vectors affordable while restoring full retrieval fidelity.

**Consequences**:
- The original 512-dim build cut storage 3x with minimal recall loss (precision@5 dropped <2%, measured via the evaluation framework in ADR-008).
- The later halfvec(1536) build carries full native dimensionality without breaking the free-tier storage budget.
- Changing dimensions requires a full re-embedding, since vectors of different widths cannot be mixed.
- All downstream consumers (search functions, reranking pipeline) track the current `halfvec(1536)` representation.

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
- Marginal latency increase (~50ms) from dual search + reranking, acceptable for conversational UX.
- RRF k=60 parameter tuned via evaluation framework to balance recall vs. precision.

---

## ADR-008: Automated Evaluation Framework

**Status**: Accepted  
**Context**: Search quality is critical: bad recommendations destroy user trust instantly. Manual testing doesn't scale and can't catch regressions when changing embedding dimensions, search parameters, or reranking models.

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
- Database-level isolation: even raw SQL access can't cross tenant boundaries with RLS enabled.
- HNSW index is global (not per-tenant), acceptable for current scale; per-tenant indexes evaluated if catalog sizes diverge significantly.
- All API routes must set tenant context before any database operation.

---

## ADR-011: Consumer Anonymous Access

**Status**: Accepted  
**Context**: Requiring authentication before trying a wine recommendation tool creates unnecessary friction. Most users want to ask one or two questions before deciding if the tool is useful.

**Decision**: Allow anonymous guest access with:
- Rate limiting via Upstash Redis (X queries per hour per IP)
- No conversation persistence for anonymous users
- Soft conversion prompts after N interactions ("Save your preferences, create an account")
- Full access (history, preferences, saved wines) behind optional authentication

**Consequences**:
- Lower friction → higher trial rate.
- Rate limiting prevents abuse without authentication.
- Anonymous usage data (aggregated, not per-user) feeds evaluation framework.

---

## ADR-012: Staff Mode

**Status**: Accepted  
**Context**: Restaurant staff need capabilities beyond what customers use: inventory management, analytics dashboards, catalog curation. These features must be access-controlled to prevent customers from modifying restaurant data.

**Decision**: Implement role-based staff mode:
- `role` column on user profiles: `guest`, `customer`, `staff`, `admin`
- Staff-only API routes gated by role middleware
- UI conditionally renders staff features (inventory search, analytics, bulk operations)
- Staff actions logged to audit trail

**Consequences**:
- Single codebase serves both customer and staff experiences.
- Role escalation requires admin approval (not self-service).
- Staff features developed incrementally without affecting customer UX.

---

## ADR-013: Integration Hub Strategy (Clean API Surfaces, Not Custom Middleware)

**Status**: Accepted
**Context**: As Vinny's integration surface grows (Toast POS, Provi distributor ordering, future Zapier/Make connections), the same N×M integration problem that enterprise iPaaS platforms (MuleSoft, Boomi, Workato) solve at scale could emerge. The naive instinct is to build a central hub. But Vinny's domain has well-established players (Olo with the Omnivore API, Deliverect, Chowly) that already solve the restaurant-POS hub problem.

**Decision**: Do **not** build integration hub middleware. Design clean API surfaces so Vinny plugs *into* existing hubs when partners come calling.

What Vinny builds day one:
- **OpenAPI spec** for all API routes, enables partner integrations without bespoke work
- **Webhook events** for state changes (wine 86'd, steering updated, menu changed)
- **OAuth2 scopes** for partner access (Toast, Provi, POS integrations)

What Vinny uses when needed:
- **Zapier/Make** for no-code connections (free to start)
- **Merge.dev or Apideck** if many CRM/POS integrations are needed fast
- **Nango** for an open-source unified-API self-hosted option

**Consequences**:
- Vinny stays focused on beverage intelligence, not middleware engineering.
- Clean API surfaces mean Olo, Toast, and other hubs can integrate Vinny without bespoke work on either side.
- If middleware ever becomes the right move (10+ integrations, enterprise-tier customers), the decision is reversible: the OpenAPI surface and webhook events are the foundation a hub would sit on top of.

---

## ADR-014: Multi-Category Schema (Separate Tables per Beverage Category)

**Status**: Accepted (Phase 17)
**Context**: Phase 17 expands Vinny from wine-only to a unified beverage intelligence platform covering beer, spirits, cocktails, and cross-category food pairings. The core schema question: store catalog data for four fundamentally different beverage categories how?

Two options evaluated:
1. **Polymorphic `beverages` table**: single table with a `category` discriminator, shared columns, category-specific data in JSONB or nullable columns.
2. **Separate tables per category**: `beers`, `spirits`, `cocktails` alongside `wines`, each with typed columns and dedicated HNSW indexes.

**Decision**: Separate tables per category. Each gets typed columns, its own HNSW vector index, its own GIN FTS index, and its own hybrid search RPC.

**Rationale**:
- **Column divergence is fundamental, not incidental**: wine has 15+ wine-specific columns; beer needs `ibu`/`srm`/`style`/`substyle`; spirits need `proof`/`age_statement`/`cask_type`/`botanicals`; cocktails need `ingredients` JSONB, `technique`, `glassware`, `family`, `ice_type`. A polymorphic table ends up with 50+ mostly-NULL columns and degraded index efficiency.
- **Vector space coherence**: separate HNSW indexes produce better recall. A query for "hoppy IPA" won't pull wine vectors into the candidate set.
- **RPC type safety**: existing `hybrid_search_wines` uses typed filter parameters (`filter_variety`, `filter_min_body`). Same pattern extends cleanly to `hybrid_search_beers(filter_style, filter_min_ibu)` and so on. A polymorphic approach would require either a single RPC with 30+ nullable parameters or runtime dispatch logic inside the RPC.
- **Additive migration**: the `wines` table and `hybrid_search_wines` RPC are never modified. Zero regression risk to existing wine functionality.

**Consequences**:
- More tables and RPCs to maintain (3 new tables, 3 new hybrid search RPCs, 3 new index sets), manageable because they follow identical patterns.
- Cross-category queries (e.g., "what pairs with steak?") require fan-out, handled by `search_beverage_pairings` against a `food_pairings` table unified by a `beverage_domain` column.
- The unified `search_beverages` tool absorbs complexity: the LLM sees one tool with a category discriminator; the backend dispatches to the appropriate RPC.
- Tenant-scoped `enabledCategories` config controls which categories each tenant exposes, which supports future product forks (e.g., a standalone "Beer Expert" deployment is the same codebase with `enabledCategories: ['beer']`).
