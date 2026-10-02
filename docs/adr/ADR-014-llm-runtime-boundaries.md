# ADR-014 — LLM Runtime Boundaries

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

LLMs are useful for natural-language interpretation and narrative realization but are non-deterministic, provider-dependent and vulnerable to leakage/incorrect authority if treated as the world simulation.

## Decision

LLMs are outside canonical mechanics.

LLMs may:
- interpret optional natural-language input into candidate structured actions;
- realize player-safe PresentationPlans as prose/dialogue;
- assist scenario authoring/validation;
- drive synthetic players;
- create explicitly noncanonical reflections/summaries.

LLMs may not directly:
- mutate canonical world/progression state;
- spend resources;
- resolve conflicts;
- grant knowledge;
- bypass permissions/addressability;
- decide canonical randomness;
- activate endings/DecisionPoints outside deterministic Scenario Progression.

Player-specific visibility projection occurs before any runtime LLM receives context.

The core Session must remain mechanically playable/correct with runtime LLM generation disabled.

Historical narrative replay uses stored delivered output rather than rerunning a current model.

## Alternatives Considered

1. LLM Game Master as canonical authority.
2. Hybrid resolver where LLM adjudicates exceptional conflicts.
3. LLM constrained to interpretation/realization/authoring/noncanonical roles.

## Why Rejected

Canonical LLM authority breaks deterministic replay, increases leakage risk and couples game correctness to model availability/behavior.

## Consequences

Structured contracts/rules must carry the actual simulation burden.

LLM quality affects presentation/input convenience, not canonical correctness.

## Risks

Over-constraining LLM use may reduce flexibility for highly open-ended future scenarios.

## Revisit Conditions

Revisit only if a future product mode explicitly abandons deterministic canonical simulation; that would require a superseding architecture rather than silent expansion.
