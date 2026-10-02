PROJECT STATE
Project: Narrative Co-op Engine
Phase: Phase 0 — Architecture Discovery
Architecture baseline: v0.1 — PROPOSED
Implementation status: Not started
Last updated: 2026-10-02
CURRENT OBJECTIVE
Red-team and refine Architecture Baseline v0.1 into v0.2 before concrete database schema, implementation repository skeleton, test engine, or first technical micro-scenario.
PRODUCT INVARIANT
Two players inhabit the same world, receive asymmetric information, take asymmetric actions, and produce one canonical timeline through deterministic causal merge.
CURRENT PROPOSED ARCHITECTURE
- Session is the main runtime consistency boundary.
- One ordered canonical event stream per session.
- Selective Event Sourcing for canonical session history.
- CQRS as logical write/read separation, initially inside one PostgreSQL system.
- Componentized Entity Model rather than a full ECS runtime.
- Semantic actions described by stable core families plus scenario-specific action types.
- Claim-based conflict detection/resolution.
- Deterministic resolver; seeded randomness only when explicitly declared and recorded.
- Fact / observation / knowledge / belief model kept distinct.
- Separate wall-clock time, logical narrative time, and engine sequence.
- Separate Simulation, Narrative Director, and Narrative Realization layers.
- Scenario source compiled into an immutable versioned bundle.
- LLM outside canonical state transitions.
- Complete version pinning and deterministic replay are architectural requirements.
- Baseline stack under evaluation: React/TypeScript, PostgreSQL/Supabase, Supabase Realtime, Cloudflare where useful, PostHog, R2.
These remain PROPOSALS until converted to ACCEPTED ADRs.
IMMEDIATE NEXT WORK
1. Red-team the v0.1 effect algebra.
2. Red-team semantic action + claim model.
3. Red-team canonical event envelope and event taxonomy.
4. Formalize deterministic conflict/resolution algorithm.
5. Produce Architecture Baseline v0.2.
6. Only then define concrete DB schema and implementation repository skeleton.
DECISIONS INTENTIONALLY NOT FROZEN
- exact core action families;
- exact primitive effect set;
- global psychological metrics;
- fixed three-act structure;
- free-text versus option-only player input;
- LLM provider/model;
- vector database or embeddings;
- graph database;
- Durable Objects / Redis;
- microservices;
- final Scenario DSL syntax;
- Scenario Studio UI;
- payment provider;
- final data-retention periods;
- recap video generation.
CANONICAL WORKING RULE
Chat discussion is not an accepted architectural decision by itself. Accepted structural decisions must be promoted to an ADR and reflected in the active Architecture Baseline.
