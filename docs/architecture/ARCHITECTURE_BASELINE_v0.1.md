ARCHITECTURE BASELINE v0.1
Status: PROPOSED
Date: 2026-10-02
ARCHITECTURAL THESIS
The system is a deterministic two-player narrative simulation with asymmetric information, concurrent semantic actions and a single canonical event stream. It is not a branching-story tree, a two-player chatbot, or a universal simulator.
CORE PIPELINE
Player Input → Action Proposal → Semantic Action → Validation → Resolution Window → Claim/Interaction Graph → Deterministic Resolver → Domain Events → State Reduction → Projections → Narrative Director → Narrative Realization
BOUNDED CONTEXTS
1. Scenario Definition — roles, entity/component definitions, action/rule definitions, scenes, decisions, metrics, narrative threads, endings, communication policy and assets. Publishes immutable CompiledScenarioBundle versions.
2. Session & Participation — sessions, participants, character assignment, invites and synchronization lifecycle.
3. Canonical Simulation — actions, windows, resolver, domain events, scheduler, state reduction and canonical event stream.
4. Epistemics & Visibility — observations, knowledge, beliefs, suspicions, evidence relations, visibility and secrets.
5. Narrative Presentation — scene/presentation selection, decision presentation, narrative realization, localization and recap projections.
6. Identity & Entitlements — anonymous/account identity and later purchases/entitlements; outside simulation core.
7. Product Analytics & Operations — telemetry, funnels, performance and experiments; must not own or receive complete canonical world state.
AGGREGATE AND EVENT SOURCING
- Session is the primary runtime consistency boundary for v0.1.
- Each Session has one ordered canonical event stream with monotonically increasing sequence/revision.
- Event Source canonical session progression, world-changing resolutions, resources/inventory, locations, epistemic changes, relationships, commitments, goals, threads, scheduled consequences and endings.
- Keep ActionProposal, ResolutionRecord, RandomDraw and NarrativeGenerationRecord append-only for audit/debug when useful, conceptually separate from canonical Domain Events.
- Accounts, analytics, drafts, UI state, media and technical logs use normal CRUD/storage patterns.
STATE MODEL
World state is structured and prose-independent. Entities use a Componentized Entity Model: Entity → typed versioned Components → capabilities/state.
SessionState contains at least metadata/revision, entities/components, relationships, epistemics, goals, commitments, narrative threads, scheduler, scene state and scenario metrics.
Scenario metrics use a registry rather than a globally hardcoded psychological vector.
SEMANTIC ACTION MODEL
Each Action Proposal contains actor/submitting participant, action family, scenario-specific action type, targets/objects, parameters, optional intended outcome, temporal spec, preconditions, claims, visibility intent and source metadata.
Core action families remain small and stable; scenario action types provide precision.
CLAIMS
Actions may claim resources, entities, state fields, locations, authority, capabilities, exclusive control, communication channels and time slots. Intersecting claims create an interaction graph; connected actions resolve together.
RESOLUTION PIPELINE
1. Freeze ResolutionWindow and base revision.
2. Validate each action independently.
3. Expand claims.
4. Construct interaction graph.
5. Group interacting actions.
6. Apply physical/state invariants.
7. Apply capability/permission/authority/resource/temporal rules.
8. Apply scenario-specific rules.
9. Apply deterministic tie-break rules.
10. Use seeded randomness only when explicitly declared.
11. Produce ResolutionRecord and Domain Events.
12. Reduce to next state.
13. Validate resulting state.
14. Commit events + projection atomically using optimistic concurrency.
Resolution outcomes may include ACCEPTED, REJECTED, PARTIAL, TRANSFORMED, INTERRUPTED and SUPERSEDED.
EFFECT ALGEBRA
The engine should converge on a finite, versioned primitive state-operation algebra such as SET, INCREMENT, ADD, REMOVE, TRANSFER, MOVE, CREATE, DEACTIVATE, GRANT, REVOKE, OPEN, CLOSE, SCHEDULE and CANCEL_SCHEDULE. Exact primitives remain PROPOSED and must be red-teamed before v0.2.
EPISTEMIC MODEL
Fact, Observation, Knowledge, Belief and Suspicion are distinct. Communication creates assertions/claims, not automatic truth. Secrets are visibility classifications. Evidence exists in-world and can support or contradict propositions.
TEMPORAL MODEL
Keep separate:
- Wall Clock — real-world time/UX/deadlines.
- Logical Game Time — in-world chronology.
- Engine Sequence — canonical technical ordering.
ResolutionWindows support simultaneous, sequential, secret-simultaneous, conditional, single-actor and automatic modes. Delayed consequences use canonical ScheduledEffects.
NARRATIVE MODEL
Strictly separate Simulation Layer (what actually happened), Narrative Director (what to show and when), and Narrative Realization (how to express it).
The deterministic visibility engine creates player-specific views before any LLM receives context.
LLM BOUNDARY
LLMs may interpret optional free text into candidate actions, realize structured events as prose, assist authoring, power synthetic players and create noncanonical reflections. They may never directly mutate canonical world state or bypass the resolver. Historical narrative output is stored and replayed rather than regenerated.
SCENARIO AUTHORING
Prefer author-friendly structured source compiled/validated into immutable versioned bundles. Runtime sessions consume compiled bundles, not mutable authoring source. Scenario structure should be state/condition-driven rather than a combinatorial branch tree. Acts/chapters are authoring metadata, not engine primitives.
VERSIONING AND REPLAY
Every Session pins engine/resolver version, source commit, scenario ID/version/hash, state/event/action/effect schema versions, rule bundle, narrative/prompt versions, RNG algorithm and base seed. Published events are immutable.
Two replay modes:
- state replay: events → reducers → state/hash;
- resolver replay: prior state + actions + versions + seed → events.
Record before/after state hashes and event-batch hashes per resolution.
PERSISTENCE AND CONCURRENCY
Baseline: PostgreSQL. Canonical events are authoritative; current-state/read projections are rebuildable. Flexible component state may use versioned schema-validated JSONB where appropriate.
Use optimistic concurrency per Session revision and atomically append event batch + update projections.
TECHNICAL BASELINE UNDER EVALUATION
- React + TypeScript browser/PWA client.
- PostgreSQL/Supabase as MVP database/auth/realtime/API baseline.
- Realtime as notification, never source of truth.
- Cloudflare for static hosting/media where useful.
- R2 for media, not canonical state.
- PostHog for deliberately designed product analytics without full world state, secret content or unnecessary free text.
- Monorepo later, with pure domain/resolver packages isolated from framework/provider SDKs.
No Redis, broker, graph DB, vector DB, Durable Objects or microservices until demonstrated need.
VALIDATION
Build-time: schema, references, rules, graph reachability, scenes/endings, temporal cycles, effects, visibility and assets.
Runtime pre-resolution: action, capability, permission, resource and temporal validation.
Runtime post-resolution: state, resource, epistemic, relationship, timeline and canon invariants.
Replay: reconstructed state hash must equal historical state hash.
TESTING
Unit/state-transition tests, property-based resolver tests, deterministic replay tests, scenario validation, regression playthroughs and later synthetic-agent playtesting. A common PlayerController interface should support human, scripted, random and agent controllers without changing the engine.
KEY RED-TEAM RISKS
Universal-simulator scope creep; infinite action ontology; untyped generic JSON; HTTP arrival order affecting simultaneous choices; LLM Game Master mutating canon; secret leakage; branch explosion; meaningless numerical psychology; analytics contaminating canonical events; generic StateChanged events; schema evolution breaking replay; projections becoming accidental truth; resolver logic drifting into database-specific code; multiple authoritative stateful backends; unmanageable scenario authoring; DSL becoming arbitrary programming language; disconnect/timeouts left to UI; delayed effects forgotten; private state leaking into logs; AI work distracting from proving causal merge is fun.
DECISIONS INTENTIONALLY PREMATURE
Exact core action families/effect primitives, psychological metrics, fixed act structure, free-text input policy, LLM provider, vector/graph databases, Redis/Durable Objects, microservices, final DSL syntax, Studio UI, payment provider, retention periods and recap-video pipeline.
NEXT TARGET
Red-team effect algebra, semantic action/claims, event envelope/taxonomy and resolver invariants to produce Architecture Baseline v0.2. Only after v0.2 proceed to concrete schema → implementation repo skeleton → test engine → tiny disposable technical scenario.
