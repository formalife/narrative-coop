# Narrative Co-op

Browser-first asymmetric cooperative narrative game/engine for two players.

## Core product invariant

Two players inhabit the same canonical world, may know different facts and take different actions, and their actions are resolved into **one canonical timeline**.

The distinguishing mechanic is **Asymmetric Agency + Causal Merge**.

## Current phase

**Phase 0 architecture accepted — Contract & Domain Model Design**

No production implementation should begin until the governing architecture is sufficiently defined and accepted.

Start with:

- [PROJECT_STATE.md](PROJECT_STATE.md) — current phase, active baseline, open work and next step.
- [Architecture Baseline v0.3](docs/architecture/ARCHITECTURE_BASELINE_v0.3.md) — accepted Phase-0 architecture.
- [Project Genesis](docs/product/PROJECT_GENESIS.md) — founding specification and original requirements.
- [Domain Glossary](docs/domain/GLOSSARY.md) — canonical terminology.
- [ADR registry](docs/adr/README.md) — structural decisions and status.
- [Project Instructions](PROJECT_INSTRUCTIONS.md) — operating rules for ChatGPT/project work.

## Sources of truth

- **GitHub** — canonical technical source for accepted architecture, ADRs, schemas, code, scenario definitions, tests and technical documentation.
- **Google Drive** — canonical store for research, source material, references, recordings, media, business material and playtest artefacts.
- **ChatGPT** — working environment. Chat discussion is not itself an accepted architectural decision.

Google Drive project root:
https://drive.google.com/drive/folders/12AztRoR0p4jRRkvlOseExRdqFrytCJnK

## Decision rule

Structural decisions move through:

`PROPOSED → ACCEPTED`

or become `REJECTED`, `SUPERSEDED`, or `DEPRECATED`.

Accepted structural decisions must be captured in an ADR and reflected in `PROJECT_STATE.md` and the active architecture baseline.

## Repository status

The Phase-0 architecture is accepted. The repository is now moving into **contract and domain/data model design**. Application code and infrastructure remain intentionally unstarted until those contracts are sufficiently stable.
