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
