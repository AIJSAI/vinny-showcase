# Technical Decisions: Vinny

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-004: Keep Vintage Years in Wine Embedding Text

**Status**: Accepted  
**Context**: The original wine review dataset is a snapshot from around 2017, and wine titles carry vintage years ("Château Margaux 2015") that end up in the embedded text. The concern was that the years could bias search toward specific bottles no longer on shelves, or lead the assistant to recommend "the 2014 Barolo" when the 2021 is what's available.

**Decision**: Keep vintage years in the embedding text as-is. Handle how vintages are presented in the prompt layer, not the data layer.

**Consequences**:
- A four-digit year is about one token out of 50 to 150 per wine, so it barely moves cosine similarity; searches like "bold red for steak" match on flavor, region and variety, not year.
- Vintage character (a hot or a cool year) is real taste information, and descriptions mention vintages too, so stripping years would remove valid signal.
- The system prompt tells Vinny to recommend by producer, style and region rather than specific vintage bottles.
- If vintage-aware recommendations are ever wanted ("best 2015 Bordeaux"), the data already supports them without re-ingestion.

---

## ADR-007: Hybrid Search (pgvector + tsvector + RRF)

**Status**: Accepted  
**Context**: Vector-only search failed on exact name searches during wine bar field testing. "Caymus Cabernet" returned semantically similar Cabernet Sauvignons rather than the specific Caymus wine, because the embedding captures the *concept* of a full-bodied Napa Cab rather than the *name*. Pure keyword search has the opposite problem on conceptual queries like "bold Italian red under $30."

**Decision**: Implement a hybrid search pipeline:
1. **pgvector** cosine similarity search (semantic understanding)
2. **tsvector** full-text search (exact keyword matching, with stemming and ranking)
3. **Reciprocal Rank Fusion** (RRF, default k=50) to merge and de-duplicate results from both paths, with tunable keyword and semantic weights
4. **Cohere reranking** (added in a later phase, now rerank-v4.0-fast) to re-score the fused candidate list by query relevance

A single Supabase RPC runs both searches and returns the fused list, and the reranker re-scores it in the app. Metadata filters (variety, country, region, points, price) run as SQL `WHERE` clauses, so a constraint like "under $50" holds exactly.

**Consequences**:
- Exact wine name queries now reliably surface the correct wine.
- Semantic queries ("something like Barolo but cheaper") still work via vector path.
- Both tsvector and pgvector are built into Supabase Postgres, so no external search service is needed.
- The keyword weight, semantic weight and RRF constant need empirical tuning, starting from Supabase's recommended defaults.

---

## ADR-008: Evaluation Framework (SommBench-Inspired)

**Status**: Accepted  
**Context**: Wine AI models are prone to hallucination and positivity bias, recommending wines confidently without grounded evidence. The SommBench benchmark (March 2026) defined three evaluation axes for wine AI (theory knowledge, food and wine pairing, and wine feature prediction) but published no code or data. Vinny needed its own evaluation to quantify recommendation quality and catch regressions.

**Decision**: Build a dedicated Vitest evaluation suite (`npm run test:eval`), kept out of the standard test run because it calls live models and data. As built, it checks:
- **Retrieval quality**: hit rate and mean reciprocal rank on wine queries whose correct answers come straight from the catalog, never from the search path under test.
- **Wine recall gate**: fails when recall at 10 drops more than 3 points below a committed baseline.
- **Grounded recommendations**: every specific bottle the model names must match a real catalog row on name, producer, ABV, vintage and price; an honest decline passes.
- **Pairing judgment**: the production chat model rates good and bad pairings against authored labels, including deliberately bad ones, to surface approval bias.

**Consequences**:
- Search and model changes are compared on numbers instead of impressions.
- The suite runs against the configured production model, so a model swap is measured, not assumed.
- Live model calls vary between runs, so the gates compare against a baseline with a tolerance instead of exact matches.
- The ground-truth data has to grow as Vinny's categories and features do.

---

## ADR-009: Separate Per-Venue Menu Table

**Status**: Accepted  
**Context**: The `wines` table is a global knowledge base: public reviews, tasting notes, embeddings and metadata that serve every user equally. Each venue also needs its own wine menu, with its own pricing, by-the-glass status and availability. The question was whether to add a `tenant_id` to `wines` or create a separate table. Vector search adds a second problem: pgvector HNSW indexes return approximate nearest neighbors *before* SQL `WHERE` filters apply, so a venue with a short menu searching a large shared index may get only a handful of results after filtering.

**Decision**:
- Keep `wines` as the global, public catalog, and create a separate `tenant_wines` table for each venue's menu, linked to the catalog by `wine_id`.
- Search the global index, join to the venue's menu, and set `hnsw.iterative_scan = strict_order` so pgvector keeps fetching candidates when the join removes too many.
- Protect venue tables with row-level security through a security-definer function that resolves the signed-in user's venues from `tenant_members`.

**Consequences**:
- No embedding duplication: menus carry no vectors, so there is no per-venue re-indexing.
- `wines` stays public; only venue-scoped tables need RLS, and their `tenant_id` columns are indexed for policy performance.
- Iterative index scans require pgvector 0.8.0 or later.
- Anonymous guests take a separate path through the API (ADR-011), so the API route must scope every query to the right venue.

---

## ADR-011: Two-Path Access (Anonymous QR Guests, Signed-In Admins)

**Status**: Accepted  
**Context**: Vinny serves two kinds of users with opposite needs. Restaurant guests scan a QR code at the table and should not need an account, since any friction kills adoption, yet they need data scoped to that restaurant's menu. Restaurant admins manage the wine list, steering and analytics, and need full authentication with row-level security. Supabase RLS identifies users by `auth.uid()`, which an anonymous guest does not have.

**Decision**: Two paths, with the API route as the trust boundary.
- **Guests (QR code)**: the venue's slug in the URL is the only input. The API route validates it, resolves the venue id on the server, and passes that id explicitly to every database function. Guests are rate-limited through Upstash Redis.
- **Admins (dashboard)**: Supabase Auth sign-in, with row-level security policies that resolve the user's venues through a security-definer function (ADR-009).

**Consequences**:
- No sign-up friction for guests, the first product requirement.
- The slug is public and the venue id is not; the server resolves one to the other.
- Guest queries rely on the API route rather than RLS, so strict slug validation and an explicit venue id in every database function are mandatory.
- The service key stays server-side only.

---

## ADR-012: Staff Mode (Dual-Persona Prompt with Role Detection)

**Status**: Proposed in March 2026, then built  
**Context**: The venue membership table already defined `owner`, `staff` and `viewer` roles, but every prompt, tool and screen was guest-facing. Servers are the real distribution channel: a server who uses Vinny before and during service carries that knowledge to the table, and staff adoption is easier to secure than a change in guest behavior.

**Decision**: Detect the signed-in user's role for the venue and switch personas within the same chat:
- **Staff and owners** get a concise, service-ready persona and staff-only tools: `generate_talking_points` (a few sentences a server can say at the table about a wine) and `shift_prep` (a pre-shift briefing on featured wines, pairings to suggest and wines that are 86'd).
- **Guests** keep the warm, exploratory consumer persona.

**Consequences**:
- One codebase and one deployment serve both personas; the switch is a prompt-level change, not an architecture change.
- Two persona variants add prompt maintenance and testing surface.
- No schema change was needed, since the staff role already existed.

---

## ADR-013: Integration Hub Strategy (Clean API Surfaces, Not Custom Middleware)

**Status**: Accepted
**Context**: As Vinny's integration surface grows (Toast POS, Provi distributor ordering, future Zapier/Make connections), the same N×M integration problem that enterprise iPaaS platforms (MuleSoft, Boomi, Workato) solve at scale could emerge. The naive instinct is to build a central hub. But Vinny's domain has well-established players (Olo with the Omnivore API, Deliverect, Chowly) that already solve the restaurant-POS hub problem.

**Decision**: Do **not** build integration hub middleware. Design clean API surfaces so Vinny plugs *into* existing hubs when partners come calling. The surfaces it commits to, ahead of the first partner integrations on the roadmap:
- **OpenAPI spec** for all API routes, so partners can integrate without bespoke work
- **Webhook events** for state changes (wine 86'd, steering updated, menu changed)
- **OAuth2 scopes** for partner access (Toast, Provi, POS integrations)

What Vinny uses when needed:
- **Zapier/Make** for no-code connections (free to start)
- **Merge.dev or Apideck** if many CRM/POS integrations are needed fast
- **Nango** for an open-source unified-API self-hosted option

**Consequences**:
- Vinny stays focused on beverage intelligence, not middleware engineering.
- Clean API surfaces mean Olo, Toast, and other hubs can integrate Vinny without bespoke work on either side.
- The decision is revisited past 100 venues or once three or more partner integrations are live; the OpenAPI surface and webhook events are the foundation a hub would sit on.

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
- Cross-category queries (e.g., "what pairs with steak?") require fan-out, through a `search_beverage_pairings` RPC over a `food_pairings` table unified by a `beverage_domain` column.
- The unified `search_beverages` tool absorbs complexity: the LLM sees one tool with a category discriminator; the backend dispatches to the appropriate RPC.
- A per-venue `enabledCategories` setting controls which categories each venue exposes.
