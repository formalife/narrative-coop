# ADR-010 — Compiled Scenario Bundle and Pure Rules

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

Scenarios should normally be data/configuration-driven and independently publishable/versionable. Runtime interpretation of mutable source files or arbitrary scripts would weaken validation, determinism and replay.

## Decision

Authoring source is compiled into an immutable, validated, content-addressed **CompiledScenarioBundle**.

Runtime consumes only the compiled bundle pinned to the Session.

The bundle may contain:
- schemas;
- initial state;
- ActionDefinitions;
- progression rules;
- interaction/resolution policies;
- event/mutation templates;
- epistemic/proposition definitions;
- scheduler definitions;
- ending rules;
- version manifest;
- asset manifest.

Compiled gameplay rules are pure deterministic functions of declared inputs.

Forbidden implicit runtime dependencies include:
- current wall clock;
- network;
- filesystem;
- mutable global/environment state;
- provider APIs;
- unseeded randomness.

No arbitrary scenario-provided runtime JavaScript/code escape hatch in the MVP architecture.

Final authoring syntax remains unfrozen.

## Alternatives Considered

1. Runtime YAML/JSON interpretation.
2. Arbitrary scenario scripts/plugins.
3. Compiled immutable declarative bundle.

## Why Rejected

Mutable runtime source and scripts make validation/replay/security substantially harder.

## Consequences

Publishing creates stable scenario versions/hashes.

A compiler/validator becomes a core product component.

## Risks

The declarative model may become too restrictive or drift into a hidden programming language.

## Revisit Conditions

Revisit only after several real scenarios demonstrate a repeated requirement that cannot be expressed through validated declarative constructs.
