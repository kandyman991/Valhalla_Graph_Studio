# Decision log

Supersede decisions with a dated entry; do not delete history. "Owner" means the repository owner (kandyman991).

## D-001 - Build graphs region by region with a persistent identity ledger
**Status:** Accepted by owner (2026-11-04) as the direction to investigate.
Build the first region, then build further regions around it renumbered to match; fix IDs once assigned and keep them in later builds. IDs may change only in tiles where a road actually changed. When a change affects two tiles, rebuild both, generating new IDs only for what is new.

## D-002 - Reject single world build and tile-splitting of a monolithic build
**Status:** Accepted by owner (2026-11-04).
Reason given: a world build (or a build split by tile afterwards) forces users to re-download maps they already store when one region is updated. See `FAILED_APPROACHES.md` R-001, R-002.

## D-003 - Keep stock Valhalla as the source of routing logic
**Status:** Proposed.
Implement a post-build tile rewriter that applies the ledger, rather than a new graph builder, so costing, turn logic and restrictions stay identical to upstream. Reconsider if the rewriter proves impractical (Phase 3/4).

## D-004 - Pin the Valhalla version; upgrades are migrations
**Status:** Proposed.
Start from the version used in production (3.6.3). A version bump is an epoch boundary unless the rewriter is ported and proven.

## D-005 - Tile-level manifests and delta updates
**Status:** Proposed.
Distribute per-tile content hashes; devices fetch only changed tiles, applied atomically across all installed regions, with a seam consistency check at runtime.

## D-006 - Validate against a stock full build
**Status:** Proposed (inherited from valhalla-graph-lab D-003).
A conventional full Valhalla build of the same data is the routing oracle.

## D-007 - Separate repository, no production effect yet
**Status:** Accepted by owner (2026-11-04), by creating this repository.
This project does not change `RoadPilot-Offline-Data` or the RoadPilot app. Integration needs the approval described in `CONTRACT_INTERACTION.md`.
