# Chat handoff - Valhalla Graph Studio

_Last curated: 2026-11-04_

## Recovery procedure
When asked to pick up Valhalla Graph Studio: read this file, `AGENTS.md`, `docs/DESIGN.md`, `docs/ROADMAP.md`, `docs/DECISIONS.md`, `docs/OPEN_QUESTIONS.md`, then check live GitHub state before acting.

## Role
Design and tooling for an ID-stable Valhalla graph builder (identity ledger + tile rewriter + tile-level delta updates). Separate from the RoadPilot production pipeline.

## Current state
- Design docs plus an imported desktop app and build pipeline (see `docs/IMPORT_PROVENANCE.md`). No ledger / rewriter code and no experiments run yet. Valhalla 3.6.3 source was read on 2026-11-04 (answers in `docs/OPEN_QUESTIONS.md`). The imported app compiles and type-checks but was never launched here.
- Direction chosen by the owner (2026-11-04): region-by-region builds with persistent IDs; IDs change only where a road really changed; when a change affects two tiles, rebuild both and allocate new IDs only for what is new. A single world build and tile-splitting of one were rejected because users would have to re-download stored maps.

## Next exact action
Phase 1 of `docs/ROADMAP.md`: obtain a Valhalla 3.6.3 build (binaries or `pyvalhalla`), freeze the Andorra extract, build twice and compare tile hashes (answers Q7 and shows the header fields that vary, Q8). Source reading for Phase 0 is done; see `docs/OPEN_QUESTIONS.md`. Q4 (dead slots) is the make-or-break experiment and comes right after.

## Open owner decisions
- Keep or drop the imported `visual/` and `search/` pipelines (RoadPilot data products, not graph tooling).
- Relationship to `valhalla-graph-lab` (continue there for research, or treat this repository as its successor).
- License.
