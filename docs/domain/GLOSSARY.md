DOMAIN GLOSSARY v0.1
Status: PROPOSED
Scenario — authored reusable game configuration/content independent of a specific playthrough.
Scenario Version — immutable published version of a Scenario used by sessions.
Session — one concrete playthrough shared by two participants; primary runtime consistency boundary in v0.1.
Participant — human or synthetic controller taking part in a Session.
Character — in-world entity controlled by a participant or engine.
Role — scenario-defined permissions, capabilities, knowledge scope, resources, authority, objectives and restrictions.
Entity — uniquely identified in-world thing composed from typed Components.
Component — typed, versioned state attached to an Entity.
Canonical State / World State — structured representation of what is objectively true in the game world at a specific engine revision.
Projection — rebuildable read model derived from canonical events.
Action Proposal — a participant's requested action before validation/resolution.
Semantic Action — normalized structured representation of an intended in-world action.
Action Family — small stable engine-level semantic category.
Action Type — scenario-specific action identifier.
Claim — resource/control/time/authority requirement an action asserts during conflict detection.
Resolution Window — set of actions evaluated against the same base state under a declared temporal policy.
Resolution Record — immutable debug/audit record of resolver inputs, rules, conflicts, randomness and output.
Domain Event — immutable canonical record that something meaningful happened in the world/session.
Effect — standardized state operation emitted/applied as part of a resolved event.
Fact — canonical proposition objectively true in the world for a validity interval.
Observation — information directly perceived by a character.
Knowledge — proposition established as known by a specific character according to engine rules.
Belief — proposition a character treats as plausible/true without canonical certainty.
Suspicion — lower-confidence epistemic stance toward a proposition.
Communication Claim — proposition asserted by a speaker; it does not become a Fact merely because it was communicated.
Secret — visibility/access classification over information, not a separate class of truth.
Evidence — world entity or relation that supports or contradicts a proposition.
Narrative Thread — stateful narrative concern such as threat, promise, unresolved conflict or mystery.
Simulation Layer — determines what canonically happens.
Narrative Director — chooses what should be presented and when, without changing canon.
Narrative Realization — converts structured presentation plans/events into readable prose/dialogue/media output.
Logical Time — in-world narrative time.
Wall Clock — real-world time used for deadlines, latency and UX.
Engine Sequence — total technical ordering of canonical events/revisions.
Replay — deterministic reconstruction of state and/or resolution from historical inputs, versions, events and seeds.
