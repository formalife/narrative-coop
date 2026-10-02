# Domain Model v0.2 Acceptance

**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted model:** DOMAIN_MODEL_v0.2  
**Supersedes:** DOMAIN_MODEL_v0.1  
**Governing contracts:** DOMAIN_CONTRACTS_v0.3 — ACCEPTED

## Accepted ownership model

- CompiledScenarioBundle owns immutable definitions.
- SessionGenesis owns deterministic revision-0 bootstrap.
- SessionEventStream owns canonical historical gameplay/progression truth.
- SessionStateProjection is rebuildable current state.
- PendingInputAggregate owns noncanonical authoritative pending input.
- TransitionEvidenceStore owns replay/debug evidence.
- PlayerInteraction/Presentation owns player-facing interaction evidence.
- Presence/Outbox/Analytics remain operational, not canonical.

## Accepted corrections

- participant binding separated from connection presence;
- derived indexes excluded from canonical state hash;
- explicit InformationClassification/Secret state;
- CommitmentInstance represents established obligation, not proposals;
- epistemic current-state/cardinality invariants;
- only irreducible Goal/Thread progress is canonical;
- terminal ending closes active Window/input gate atomically.

## Evidence

- `RED_TEAM_DOMAIN_MODEL_v0.2.md`
- `POSTMORTEM_DOMAIN_MODEL_v0.1.md`
