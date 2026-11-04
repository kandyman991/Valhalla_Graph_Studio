# Design: ID-stable Valhalla graph builder

**Status:** proposal. Statements about Valhalla internals are working assumptions to be verified against the pinned Valhalla source; each is listed in [`OPEN_QUESTIONS.md`](OPEN_QUESTIONS.md). Nothing here is proven.

## 1. Goals and non-goals

Goals
- Regions built at different times connect natively at their borders in one tile directory.
- A data update changes only the tiles that really changed; unchanged tiles stay byte-identical.
- Devices download only changed tiles (tile-level delta), not whole region packs.
- Routing behaviour stays identical to stock Valhalla.

Non-goals
- Replacing the Valhalla router or forking its costing logic.
- A monolithic world build, or splitting one world build by tile (rejected, see `DECISIONS.md` D-002).
- Changing the RoadPilot cross-region runtime (F8) in this repository.

## 2. Why separate builds do not connect

Working assumptions (to verify):
- A graph ID is level + tile + index. Nodes and directed edges are dense arrays inside a tile.
- A node's directed edges are contiguous (node edge index + edge count). Each edge stores its end node (a graph ID, possibly in another tile) and the index of its opposing edge at that end node.
- Nodes on different hierarchy levels are linked by transitions holding the other level's node ID.
- Tiles also hold restrictions, bins (edge lists for spatial lookup, including edges that cross in from neighbouring tiles), and per-node data packed for a small number of local edges.
- IDs are assigned by the build, so two builds of different data (or of the same data in a different order) number things differently.

Consequence: copying tiles between builds does not connect roads, and any renumbering inside a tile invalidates every reference to it.

## 3. Pipeline

```
source prep -> stock build -> match to ledger -> dirty set -> rewrite -> validate -> publish
```

1. **Source prep.** Fetch and verify OSM extracts, merge neighbours, clip to the region polygon plus a border buffer, record SHA-256 of every input. Tile ownership follows the global tile grid, not the clip polygon: a tile must be built with complete data around it.
2. **Stock build.** Run the pinned Valhalla build. Output: *raw tiles* with provisional IDs. These are an intermediate product, never published.
3. **Match.** Extract every node and directed edge from the raw tiles, compute its stable key, and look it up in the ledger. Result: retained, new, deleted and changed (attributes only) entities.
4. **Dirty set.** Tiles that contain any new, deleted or changed entity, plus tiles that reference a relocated node or an affected cross-tile edge (found through a reverse index in the ledger). All other tiles are carried over from the previous release byte for byte and are not rewritten.
5. **Rewrite.** For dirty tiles only, produce the final tile applying the ID rules in section 5.
6. **Validate.** Integrity checks, byte-identity of untouched tiles, routing comparison against a stock full build (section 9).
7. **Publish.** Per-tile content hashes in a manifest; ledger and manifest are committed atomically (section 7).

## 4. Identity ledger

Lives on the build server only; never shipped to devices.

Stable keys (initial proposal, refine experimentally):
- **Node:** coordinates (quantized as in the tile) plus the set of incident OSM way IDs. Matching tolerates a small displacement, so a mapper nudging a node by a few metres does not look like delete + create.
- **Directed edge:** OSM way ID + start node key + end node key + direction + segment position (for ways split into several edges).

The ledger stores, per key, the assigned graph ID and a revision. It also stores a reverse index (node -> tiles that reference it) used to compute the dirty set cheaply.

The ledger is the source of truth for ID stability. If it is lost, IDs cannot be preserved and the fallback is a full epoch reset (all users re-download). It must be versioned, backed up and published atomically with each release.

## 5. Tile rewrite rules

For a dirty tile:
- **Retained** node/edge: keeps its ID and array position; data fields (speed, names, shape, access) are updated in place.
- **New** node/edge: appended at the end of the arrays.
- **Deleted** node/edge: becomes a *dead slot* (no traversable access, excluded from bins and search) so later entries do not shift.
- **Node whose set of edges changed** (an edge added or removed at an intersection): contiguity of its edges forbids growing in place, so it gets a **new node ID** appended at the end and the old node becomes a dead slot. Every edge ending at the old node is patched to the new ID, and opposing-edge indices pointing at it are recomputed. This is the only source of ripple beyond the tile itself.
- **Always patched in dirty tiles:** end-node references, opposing-edge indices, hierarchy transitions, restrictions, bins, offsets and the tile header counts.

Tile growth from dead slots is bounded by periodic per-tile **compaction** (section 8).

## 6. Hierarchy, shortcuts and seams

- **Upper levels.** Shortcuts are derived data and can be recomputed on every build; only upper-level *node* IDs must be stable because lower levels reference them through transitions. Upper-level tiles will still change often (they cover large areas) but local tiles do not need to change just because an upper tile did. To be verified.
- **Seams.** A seam is where a road crosses a tile owned by another region. Regions are built in sequence. The first build assigns the IDs of seam nodes; later builds **pin** those IDs (match by coordinates and way IDs) instead of allocating new ones. Seam tiles have one authoritative copy, with a revision recorded in the manifest.
- **Seam dependencies.** A change at a seam node touches tiles on both sides. The manifest records, for each seam tile, which neighbour revisions it requires.

## 7. Update and distribution protocol

- Manifest: tile path -> content hash, size, revision, plus seam dependencies and the build-set identifier.
- A device keeps a local manifest. An update downloads exactly the tiles whose hash differs, for **all** installed regions together and applied atomically (swap on success).
- Adding a region brings the device to the current build set first.
- Runtime check: before cross-region routing over a seam, verify that both sides are at compatible revisions. If not, fall back to the existing cross-region logic (F8) for that seam instead of routing over a mismatched pair.
- Publication: immutable per-tile objects plus a small `latest` pointer written last (same pattern as the existing R2 publication, see `REUSE.md`).

## 8. Epochs, compaction, version bumps

- **Compaction:** rebuilding a tile without dead slots changes IDs inside it and in its referrers; do it rarely and per tile.
- **Epoch reset:** a full rebuild with new IDs, needed if the ledger is lost, the key scheme changes, or Valhalla's tile format changes. Users re-download once.
- **Valhalla version:** pin it (production currently uses 3.6.3). A version bump is a migration event and an epoch boundary unless the rewriter is ported and proven.

## 9. Validation

1. **Determinism:** same input, same version and config produce identical tile bytes (precondition for hash-based deltas).
2. **Byte identity:** tiles outside the dirty set are identical to the previous release.
3. **Integrity checker:** every end node exists; opposing edges are symmetric; transitions are consistent; bins contain only live edges; no reference to a dead slot from a live entity.
4. **Routing oracle:** a stock full build of the same data is the reference. Compare routes (cost, time, distance) on a large random sample, including cross-seam routes, in both directions and for each costing profile in use.
5. **Cross-region parity with F8:** final acceptance evidence, compared on the same graph fingerprints.

## 10. Main risks

- Valhalla may not tolerate dead slots or non-canonical ordering somewhere in loki/thor/meili code paths.
- The rewriter depends on internal tile structures; maintenance cost on each Valhalla upgrade.
- Non-deterministic stock builds would make hash deltas useless; this is the first thing to measure.
- Upper-level tile size may make "small deltas" less small than hoped.
- Key matching quality (splits, merges, retagging, nudged nodes) decides how much ripple real updates cause.
