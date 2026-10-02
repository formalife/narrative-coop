# DOMAIN GLOSSARY v0.2

**Status:** PROPOSED terminology aligned to Architecture Baseline v0.2.

**Scenario** — authored reusable game configuration/content independent of a specific playthrough.

**Scenario Version** — immutable published version of a Scenario.

**CompiledScenarioBundle** — immutable, validated, content-addressed runtime representation of one Scenario Version, including schemas/rules/version manifest.

**Session** — one concrete playthrough shared by participants; proposed primary runtime consistency boundary.

**Participant** — human or synthetic controller participating in a Session.

**Character** — in-world Entity controlled by a Participant or engine.

**Role** — scenario-defined set of permissions, capabilities, knowledge scope, resources, authority, objectives and restrictions associated with a Character/Participant.

**Entity** — uniquely identified in-world thing composed from typed Components.

**Component** — typed, versioned state attached to an Entity.

**SessionState** — rebuildable current-state projection for a Session at a known stream revision/hash.

**Canonical Event Stream** — append-only ordered history of committed canonical Domain Events for one Session.

**Projection** — read/current-state model derived from canonical events; not an independent historical source of truth.

**ResolutionWindow** — bounded collection of action slots/submissions resolved under one temporal/resolution policy.

**Action Slot** — scenario/window-defined opportunity or requirement for a Participant/actor to submit an action.

**ActionSubmission** — minimal client/input request containing action type and allowed parameters/targets; it does not contain authoritative claims/effects.

**ActionDefinition** — immutable scenario definition that owns schemas, constraints, claim derivation, timing and resolver/effect references for an action type.

**ExpandedAction** — server-derived, deterministic resolution input produced from an ActionSubmission + ActionDefinition + frozen base state.

**Action Type** — scenario/core semantic identifier selecting an ActionDefinition.

**Action Family / Action Tag** — optional classification for authoring/UI/analytics; does not dispatch canonical mechanics.

**AdmissionResult** — result of validating whether an ActionSubmission is allowed to enter world resolution.

**ActionOutcome** — canonical in-world result of resolving an admitted action.

**Claim** — contention-relevant access/demand declaration derived by the engine, with a target, mode and optional quantity/time scope.

**Interaction Graph** — graph of admitted actions connected by typed interaction edges such as contention, exclusion, interference, dependency, complementarity or order sensitivity.

**Resolution Strategy** — versioned bounded engine algorithm used to resolve a class of interacting actions.

**ResolutionPlan** — deterministic pre-commit result containing ActionOutcomes, random evidence and intended canonical events/mutations.

**ResolutionRecord** — immutable audit/debug record of resolver inputs, versions, rules, random draws, plan and result.

**Domain Event** — immutable canonical semantic record that an occurrence happened in the Session.

**Event Family** — finite engine-level classification of a Domain Event.

**Event Code** — precise core or scenario-specific semantic identifier of a Domain Event.

**Event Batch** — ordered atomic collection of Domain Events committed by one resolution/engine operation.

**State Mutation IR** — small typed internal intermediate representation describing validated projection mutations associated with canonical events.

**Fact** — canonical truth assertion about a Proposition, optionally time-bounded and provenance-linked.

**Proposition** — structured scenario-defined statement that can be true/false/valued and referenced by epistemic records.

**Observation** — explicit record that a Character perceived an occurrence, proposition or evidence.

**Knowledge** — engine-established epistemic stance indicating that a Character knows a Proposition according to scenario rules.

**Belief** — Character-specific stance treating a Proposition as plausible/true without canonical truth equivalence.

**Suspicion** — weaker/uncertain Character-specific stance toward a Proposition.

**Communication Claim** — Proposition asserted by a speaker to a recipient/channel; the assertion does not make it Fact.

**Secret** — visibility/access or narrative classification over information/evidence; not a separate category of truth.

**Evidence** — canonical world Entity/relation that can support or contradict Propositions.

**Addressability** — whether an actor is permitted to reference/select/control a target for a particular ActionDefinition; distinct from target existence.

**Narrative Thread** — stateful narrative concern such as a threat, promise, mystery or unresolved conflict.

**Logical Time** — in-world narrative time used by rules/scheduler.

**Wall Clock** — real-world time used for UX/deadlines/infrastructure triggers; not historical replay ordering.

**Engine Order / Stream Revision** — authoritative total technical ordering of canonical Session events.

**Scheduler** — canonical mechanism for logical delayed consequences.

**Named Random Draw** — versioned seeded random operation identified by a stable draw key so unrelated draws do not perturb each other.

**Simulation Layer** — determines canonical world outcomes.

**Narrative Director** — deterministic/versioned layer selecting what scene/information/decision structure should be presented and when.

**PresentationPlan** — player-safe structured output from the Narrative Director given to realization/UI.

**Narrative Realization** — noncanonical conversion of a PresentationPlan into prose/dialogue/media presentation.

**State Replay** — reconstruction of SessionState from pinned initial state/bundle + canonical event stream.

**Resolver Replay** — re-execution of historical resolution inputs/versions/random evidence to verify the generated event batch.

**Canonical Serialization** — versioned deterministic representation of state/events used before hashing.
