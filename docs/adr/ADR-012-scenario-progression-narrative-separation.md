# ADR-012 — Scenario Progression, Narrative Direction and Realization Separation

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The architecture must separate what canonically happens from what is presented. A prior design allowed the Narrative Director to activate decision points/endings, accidentally making presentation logic a second canonical authority.

## Decision

Use three distinct layers.

### Scenario Progression — canonical

Deterministically owns:
- trigger evaluation;
- progression state;
- DecisionPoint activation;
- ResolutionWindow creation/closure policy;
- canonical pacing gates;
- goal/objective progression where applicable;
- ending activation;
- Session completion/pause transitions.

Progression emits canonical events.

### Narrative Direction — presentation-only

May choose:
- ordering/emphasis of already-authorized information;
- grouping;
- recap emphasis;
- nonmechanical presentation sequencing.

It cannot:
- grant knowledge;
- add/remove legal actions;
- activate decision points;
- alter world/resources;
- choose endings.

### Narrative Realization — noncanonical wording/media

Produces prose/dialogue/media from a player-safe PresentationPlan. LLM use is optional.

Delivered output that can causally influence later human choices is stored in an immutable PresentationRecord.

## Alternatives Considered

1. LLM/Narrative Director as Game Master.
2. Deterministic Director owning both mechanics and presentation.
3. Canonical Progression separated from presentation layers.

## Why Rejected

The first two create ambiguous canonical authority and weaken deterministic replay.

## Consequences

Mechanical progression remains testable with LLM disabled.

Presentation can evolve independently.

## Risks

Some authoring concepts such as "scene" may straddle progression and presentation and require precise contracts.

## Revisit Conditions

Revisit only if real scenario authoring reveals a boundary that cannot be expressed without reintroducing a second canonical authority.
