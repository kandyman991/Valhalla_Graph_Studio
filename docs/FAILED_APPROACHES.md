# Rejected, superseded and inconclusive approaches

Purpose: preserve *why* an approach was dropped, so it is not repeated by accident. Check this file before proposing a significant design, and update it in the same PR that rejects or supersedes something.

## Status definitions (never conflate)
- **CONFIRMED_FAILURE:** executed attempt with a reproducible negative result.
- **REJECTED_BY_DESIGN:** consciously declined for stated constraints; not necessarily built or tested.
- **SUPERSEDED:** replaced by a decision; does not prove it failed.
- **INCONCLUSIVE:** insufficient evidence; the next discriminating test is named.

Unexecuted ideas are REJECTED_BY_DESIGN or INCONCLUSIVE, never reported as tested failures.

## R-001 - One world (or continent) build, shipped as region packs
- Status: REJECTED_BY_DESIGN
- Recorded: 2026-11-04
- Idea: build everything in one Valhalla run so all IDs are consistent.
- Reason: updating one region would produce new tiles from a new build, and devices holding other regions from the old build would have broken cross-region links unless everything is re-downloaded.
- Evidence: reasoning only; nothing was built or measured.
- Reconsider only if: a measured tile-level delta between consecutive world builds is small (Phase 6 data), which would make this simple option viable.

## R-002 - Split a monolithic build by tile list
- Status: REJECTED_BY_DESIGN
- Recorded: 2026-11-04
- Same reason as R-001: unchanged shipping unit would be the region, and a rebuild renumbers across regions.
- Reconsider only if: as R-001.

## R-003 - Independent regional builds joined by a RoadPilot-owned layer (F8) only
- Status: not rejected; this is the current production approach and the fallback here.
- Limitation motivating this project: cross-region routing depends on runtime or precomputed discovery of crossings instead of native graph links.
- See `RoadPilot-Offline-Data` `docs/ARCHITECTURE_CONTRACT.md` and `docs/FAILED_APPROACHES.md`.

## VGL-001 - Recomposition of independently built modules (from valhalla-graph-lab)
- Status: INCONCLUSIVE (recorded there on 2026-10-24)
- That entry states the research plan was not executed as a production validation. This project continues the question with a ledger-based method and Phases 0-5 as the discriminating experiments.
