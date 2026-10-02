# DOMAIN GLOSSARY v0.3

**Status:** CANONICAL for Architecture Baseline v0.3. May be extended during contract design without changing accepted semantics.

**Scenario** — authored reusable game configuration/content independent of a specific playthrough.

**Scenario Version** — immutable published version of a Scenario.

**CompiledScenarioBundle** — immutable, validated, content-addressed runtime representation of a Scenario Version.

**Session** — one concrete playthrough; proposed canonical consistency boundary.

**Single Canonical Frontier** — invariant that only one canonical state-changing progression/resolution operation advances a Session at a time.

**Participant** — human or synthetic controller participating in a Session.

**Principal** — server-authoritative identity/system authority attached to an EngineCommand.

**Character** — in-world Entity controlled by a Participant or engine.

**Role** — scenario-defined permissions, capabilities, knowledge scope, resources, authority, objectives and restrictions.

**EngineCommand** — idempotent authoritative request/input to Session application logic; a command is not itself a canonical occurrence.

**Command Journal** — durable record used to deduplicate/retry command processing.

**Action Slot** — window-defined opportunity/requirement for one actor/participant action.

**ActionSubmission** — durable pending player input for an ActionSlot; not canonical world truth until selected and resolved.

**ActionDefinition** — immutable scenario authority defining action input schema, constraints, claims, timing and resolver/effect references.

**ExpandedAction** — deterministic server-derived resolution input created from a selected ActionSubmission, ActionDefinition and frozen state.

**AdmissionResult** — result of validating whether an ActionSubmission enters world resolution.

**ActionOutcome** — in-world result of one admitted ExpandedAction.

**Claim** — engine-derived contention/access/demand description over a target/scope.

**Interaction Graph** — representation of meaningful interactions among admitted actions.

**Interaction Group/Component** — connected set of actions that must be resolved jointly, including aggregate capacity effects.

**Resolution Strategy** — bounded versioned algorithm for a class of interactions.

**ResolutionPlan** — deterministic pre-commit plan containing outcomes, random evidence and canonical transitions.

**ResolutionRecord** — immutable audit/debug record of selected inputs, pinned versions, resolution evidence and results.

**Entity** — uniquely identified in-world thing composed from typed Components.

**Component** — typed versioned state attached to an Entity.

**SessionState** — synchronous rebuildable current projection at a known stream revision/hash.

**Canonical Event Stream** — append-only ordered historical source for canonical Session state/progression.

**CanonicalEventContent** — deterministic hash/replay-relevant semantic event content.

**EventRecordMetadata** — operational event persistence metadata such as database id/recorded timestamp, excluded from semantic replay equality.

**Domain Event** — immutable canonical semantic occurrence.

**Event Family** — finite engine-level event category.

**Event Code** — precise core/scenario semantic event identifier.

**State Mutation IR** — typed internal representation of validated canonical projection mutations.

**Scenario Progression** — deterministic canonical subsystem that activates DecisionPoints/windows/progression/ending state.

**Narrative Direction** — presentation-only selection/emphasis over already-authorized player-safe content.

**PresentationPlan** — structured player-safe presentation input to realization.

**Narrative Realization** — conversion of PresentationPlan into prose/dialogue/media; not canonical world truth.

**PresentationRecord** — immutable record of what was actually delivered to a player, retained because it can causally influence later human choices.

**Proposition** — structured referencable statement with predicate/arguments/value.

**Fact** — explicit canonical truth assertion/provenance/validity record for a Proposition when epistemic reference is needed.

**ObservationRecord** — record that a Character perceived an occurrence/evidence/proposition.

**KnowledgeRecord** — deterministic granted knowledge of a Proposition with provenance.

**BeliefRecord** — Character belief about a Proposition/value, independent from canonical truth.

**SuspicionRecord** — tentative weaker attitude toward a Proposition.

**CommunicationClaim** — Proposition/value asserted by a speaker to a recipient/channel; does not become truth by being stated.

**Secret** — visibility/access/narrative classification over information/evidence; not its own truth category.

**Evidence** — canonical world object/relation that supports or contradicts a Proposition.

**Addressability** — whether an actor may refer to/select/control a target for an ActionDefinition, independently from raw target existence.

**Logical Time** — deterministic in-world integer/tick time used by mechanics.

**Wall Clock** — real-world time used for UX/deadline triggers; operational rather than replay ordering.

**Stream Revision** — authoritative technical order of canonical events.

**Scheduler** — deterministic canonical system for logical delayed consequences.

**Named Random Draw** — seeded/versioned draw identified by stable key so unrelated random calls cannot perturb it.

**Canonical Serialization** — deterministic versioned representation used before hashing.

**Transactional Outbox** — durable records inserted in the same transaction as canonical commit so required post-commit work can be retried idempotently.

**State Replay** — rebuild historical SessionState from pinned initial state + canonical events.

**Resolver Verification Replay** — re-execute historical resolution under pinned runtime/bundle to compare semantic output.

**Forensic Replay** — reconstruct/investigate a historical Session using stored resolution/random/event/presentation evidence when old executable runtime is unavailable.

**Narrative Replay** — display exact historical PresentationRecords rather than regenerating output.


## Contract-design additions — Domain Contracts v0.2

**CommandKey** — semantic idempotency identity for one logical EngineCommand across retries/duplicate delivery.

**CommandProcessingRecord** — durable application record tracking command idempotency and resumable processing; not canonical world state.

**CanonicalTransitionRecord** — generic immutable record owning one committed canonical event batch, regardless of whether the transition was caused by player resolution, progression, scheduler, lifecycle, deadline or compensation.

**WindowInputGate** — durable authoritative pending-input gate for a ResolutionWindow; controls whether ActionSubmissions can still be accepted without itself being canonical world state.

**SlotInputRevision** — compare-and-set revision for one ActionSlot's pending input.

**FrozenInputSet** — immutable selected-submission snapshot captured when a ResolutionWindow stops accepting input.

**ResolutionAttempt** — operational/audit state for resolver execution; PREPARING/FROZEN/RESOLVING-like phases are not canonical Window states.

**Submission Eligibility** — condition that must hold for an input to be a valid attempt and cannot be enabled by another same-window action.

**Resolution Requirement** — world/resource condition evaluated during joint resolution and allowed to be enabled, invalidated or transformed by other admitted actions.

**CanonicalStateContent** — hashable canonical gameplay/progression state excluding projection metadata such as its own hash.

**MechanicsVersionManifest** — Session-pinned versions required for mechanical replay.

**PresentationGenerationContext** — immutable per-presentation versions/configuration used by presentation workers and retries.

**PredicateDefinition** — Scenario Bundle definition of a predicate's argument/value/temporal schema.

**PropositionValue** — immutable runtime statement value derived from a PredicateDefinition and runtime arguments/value.

**PropositionKey** — deterministic content-derived identity of a PropositionValue.

**BoundaryOrderingPolicy** — explicit rule defining the relative order of due scheduled effects and action resolution at a canonical frontier.

**ProgressionEvaluation** — bounded deterministic repeated evaluation of Scenario Progression transitions until stable/ended or a step budget is exhausted.

**PlayerInteractionView** — deterministic audience-safe projection containing both visible information and mechanically authorized interaction affordances before Narrative Direction.

**InteractionSurface** — mechanically authorized action slots/action types/targets derived from current canonical and authorized pending-input state; Narrative Direction may not alter it.


## Domain-model additions — v0.2

**SessionGenesis** — immutable per-Session bootstrap record that owns deterministic revision-0 reconstruction metadata, including the Session RNG seed once Domain Contracts v0.3 is accepted.

**ParticipationState** — canonical gameplay binding between participant slot, opaque participant reference, Character and Role; distinct from realtime connectivity.

**PresenceState** — operational connection/presence information; not canonical gameplay state by itself.

**SessionDerivedIndexes** — rebuildable lookup/acceleration structures excluded from canonical state hash.

**InformationClassification** — Secret/access classification over a proposition/evidence/entity-information/thread reference; distinct from Fact or Knowledge.

**GoalInstance** — canonical runtime instance of a GoalDefinition, storing only irreducible status/progress state.

**CommitmentInstance** — mechanically established canonical obligation; proposals/offers are not automatically commitments.

**NarrativeThreadInstance** — canonical runtime narrative-thread state only when rules consume it.

**Predicate Epistemic Cardinality** — compiled policy such as SINGLE_VALUE or MULTI_HYPOTHESIS controlling allowed simultaneous active belief/suspicion values for one subject/scope.


## Persistence additions — v0.3

**SessionRuntime** — mutable synchronous current SessionState projection row and canonical concurrency anchor.

**Canonical Event Bytes** — exact immutable serialized CanonicalEventContent stored as the authoritative event representation.

**Event Index Metadata** — non-authoritative relational columns derived from canonical event bytes for query/index acceleration.

**Processing Generation** — monotonically increasing fencing token for CommandProcessingRecord lease ownership.

**Lease Generation** — monotonically increasing fencing token for Outbox work claims; stale generations cannot finalize task results.

**Submission Content Hash** — hash of canonicalized mechanical ActionSubmission content, excluding operational timestamps.

**Freeze Owner Command Key** — semantic CommandKey that owns a FrozenInputSet/resolution frontier after gate freeze.

**Presentation Task Identity** — stable presentation_id/dedup identity used across async generation and delivery retries.

**Delivery Guard** — check preventing a presentation built for a superseded PlayerInteractionView from being delivered as current gameplay.


## PostgreSQL schema additions — v0.5

**Canonical Event Bytes** — immutable application-canonical serialized Domain Event content stored in PostgreSQL BYTEA; event index columns are non-authoritative.

**Presentation Hash Framing** — explicit canonical object whose hash identifies a PlayerInteractionView, PresentationPlan, or rendered presentation artifact.

**Composite Context FK** — relational foreign key that validates not merely that an artifact exists, but that it belongs to the same Session/audience/view/hash context.

**Presentation Lifecycle Trigger** — narrow PostgreSQL trigger enforcing persistence immutability/state transitions of PresentationRecord without implementing narrative/game logic.

**Migration Owner** — dedicated database role owning DDL/functions; never used as normal application runtime credentials.

**Provider-neutral Migration Archive** — canonical GitHub SQL migration path under `db/migrations/`, independent from a particular hosting provider's CLI layout.


## Implementation-platform additions — v0.3

**AuthSubject** — external authenticated security subject (initially Supabase Auth user/JWT subject); operational identity, never canonical world identity.

**Session Principal Binding** — noncanonical access record mapping one verified AuthSubject to one ParticipantRef for one Session.

**Access Schema** — internal PostgreSQL schema containing operational authorization/invite state; separate from canonical engine world state.

**Engine Runtime Role** — least-privilege PostgreSQL role used by the authoritative engine API for accepted command/current-state transactions.

**Engine Worker Role** — least-privilege PostgreSQL role used by the Outbox worker for presentation/realtime/secondary work; cannot mutate canonical Session state.

**Persistent API Runtime** — long-running service exposing authoritative HTTP commands/views while keeping canonical authority in PostgreSQL.

**Persistent Outbox Worker** — long-running service that claims durable PostgreSQL Outbox work using locks, leases and fencing; process uptime is not work durability.

**Realtime Invalidation** — content-free private notification that a player's authoritative view may have changed; never carries canonical/private gameplay state itself.

**Provider-specific Glue** — deployment/runtime integration such as Supabase Realtime policies/functions or Railway service configuration that may not redefine provider-neutral engine semantics.


## Access-schema additions — v0.3

**AuthSubject** — Supabase Auth user/JWT subject used only for operational authentication; not canonical gameplay identity.

**Session Principal Binding** — operational record authorizing one AuthSubject to act/read as one ParticipantRef in one Session.

**Binding Authorization Lock** — short PostgreSQL row lock on an ACTIVE binding used to linearize authorized requests against access revocation.

**Participation Origin Transition** — canonical transition referenced by an access binding as historical evidence of the ParticipantRef's canonical establishment; semantic equality is verified outside SQL FK semantics.

**Session Invite** — one-time operational capability for claiming a canonical participant slot; stores only a cryptographic token hash.

**Invite Claim Linearization Point** — locked database wall-clock evaluation point at which invite status, expiry and canonical target availability are authoritatively checked.

**Access Recovery/Rebind** — future privileged feature intentionally unsupported by the MVP access schema; adding it requires relaxing lifetime binding uniqueness.
