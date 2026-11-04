# Roadmap

Rule for every phase: change one variable at a time, record inputs and hashes, keep the experiment reproducible. Heavy builds run locally (self-hosted machine), not in cloud CI. Documentation-only commits use `[skip ci]`.

## Phase 0 - Environment and source verification
**Status 2026-11-04:** source reading done for Valhalla 3.6.3 (`OPEN_QUESTIONS.md`). Still to do: pin the build recipe, set up a Valhalla binary and the Andorra extract. Q4, Q7 and Q9 need experiments and move to Phases 1 and 4.
- Pin the Valhalla version (production: 3.6.3) and record compiler, OS and config.
- Read the pinned Valhalla source and answer every item in `OPEN_QUESTIONS.md`. Record answers with file and line references.
- Freeze the test dataset: Andorra first (small, borders, tunnels, roundabouts), a second dataset later (two adjacent small regions for seam work).
- **Pass:** every open question is answered or turned into a concrete experiment.

## Phase 1 - Determinism baseline
- Build the same input twice; compare file lists, sizes, SHA-256 per tile.
- Classify any difference as meaningful, metadata-only (e.g. timestamps), ordering, or unexplained.
- Same experiment with different thread counts.
- **Pass:** every difference explained; a reproducible build recipe exists (or a documented normalization step).
- Reuse the experiment protocol from `valhalla-graph-lab` (`docs/EXPERIMENTS.md`, `experiments/001-identical-builds`).

## Phase 2 - Entity extractor and matcher
- C++ tool (libvalhalla, read-only) that dumps every node and directed edge with graph ID, coordinates, incident way IDs, end node, opposing index.
- Python matcher that computes stable keys and matches two builds of the same area before and after one controlled OSM edit.
- Report: retained / new / deleted / changed, with the unmatched share.
- **Pass:** more than 99% of entities match on an unchanged area; every unmatched entity is explained.

## Phase 3 - Renumber-only rewriter
- Rewrite tiles so a second build adopts the IDs of the first on an *unchanged* network (pure renumbering, no dead slots).
- **Pass:** the output equals the first build's tiles byte for byte, and the integrity checker passes.

## Phase 4 - Single edit
- One new intersection, then one deleted road, then one attribute change (Experiments 002/003 of the lab).
- Rules from `DESIGN.md` section 5: append, dead slot, relocate node.
- **Pass:** only the expected tiles differ, the integrity checker passes, and local routing matches a stock full build.

## Phase 5 - Routing equivalence
- Several thousand random routes (all costing profiles in use, both directions) on the stock full build versus the rewritten graph.
- **Pass:** no cost/time/distance regressions beyond a documented tolerance; every deviation explained.

## Phase 6 - Two regions and a seam
- Build region A, then region B with pinned seam IDs; then update A and verify the seam.
- Measure the delta (tile count and bytes) for realistic updates.
- **Pass:** cross-border routes work natively in one tile directory and the measured delta supports the "small delta" claim, or the claim is revised with data.

## Phase 7 - Manifest and update protocol
- Tile-level manifest, seam dependencies, atomic apply, runtime consistency check, behaviour on mismatched seams.
- Reuse the R2 immutable-publication pattern.

## Phase 8 - Integration
- Client ingestion of tile-level updates, Graph Studio UI for ledger and dirty-set inspection.

## Out of scope for now
Planet-scale builds, client-side graph generation, traffic data, transit.
