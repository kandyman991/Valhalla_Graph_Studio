# Chat handoff - Valhalla Graph Studio

_Last curated: 2026-11-04_

## Recovery procedure
When asked to pick up Valhalla Graph Studio: read this file, `AGENTS.md`, `docs/DESIGN.md`, `docs/ROADMAP.md`, `docs/DECISIONS.md`, `docs/OPEN_QUESTIONS.md`, then check live GitHub state before acting.

## Role
Design and tooling for an ID-stable Valhalla graph builder (identity ledger + tile rewriter + tile-level delta updates). Separate from the RoadPilot production pipeline.

## Current state
- Documentation only. No code, no experiments run, no Valhalla source reviewed yet.
- Direction chosen by the owner (2026-11-04): region-by-region builds with persistent IDs; IDs change only where a road really changed; when a change affects two tiles, rebuild both and allocate new IDs only for what is new. A single world build and tile-splitting of one were rejected because users would have to re-download stored maps.
- The direction conflicts with RoadPilot contract clauses (see `docs/CONTRACT_INTERACTION.md`). No production effect.

## Next exact action
Phase 0 of `docs/ROADMAP.md`: pin the Valhalla version (3.6.3 in production), then read its source and answer `docs/OPEN_QUESTIONS.md` Q1-Q13, starting with Q2, Q3 and Q4 (they decide whether the dead-slot approach is possible at all). Then Phase 1 (determinism) on Andorra.

## Open owner decisions
- Relationship to `valhalla-graph-lab` (continue there for research, or treat this repository as its successor).
- License.
