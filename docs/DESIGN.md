# Design: ID-stable Valhalla graph builder

**Status:** proposal, revised after reading the Valhalla 3.6.3 source (2026-11-04, see [`OPEN_QUESTIONS.md`](OPEN_QUESTIONS.md) for the answers and code references). Nothing has been executed or proven yet; Q4 (dead slots), Q7 (determinism) and Q9 (tile writing) still need experiments.

## 1. Goals and non-goals

Goals
- Regions built at different times connect natively at their borders in one tile directory.
- A data update changes only the tiles that really changed; unchanged tiles stay byte-identical.
- Devices download only changed tiles (tile-level delta), not whole region packs.
- Routing behaviour stays identical to stock Valhalla.

Non-goals
- Replacing the Valhalla router or forking its costing logic.
- A monolithic world build, or splitting one world build by tile (rejected, see `DECISIONS.md` D-002).
- Changing any client or runtime routing code in this repository.

## 2. Why separate builds do not connect

Working assumptions (to verify):
- A graph ID is level + tile + index. Nodes and directed edges are dense arrays inside a tile.
- A node's directed edges are contiguous (node edge index + edge count). Each edge stores its end node (a graph ID, possibly in another tile) and the index of its opposing edge at that end node.
- Nodes on different hierarchy levels are linked by transitions holding the other level's node ID.
- Tiles also hold restrictions, bins (edge lists for spatial lookup, including edges that cross in from neighbouring tiles), and per-node data packed for a small number of local edges.
- IDs are assigned by the build, so two builds of different data (or of the same data in a different order) number things differently.

Confirmed from the source: nodes are numbered by (spatial grid cell, OSM node ID) inside a tile, so one inserted node renumbers all later nodes; a node's edges are one contiguous run (`edge_index`, `edge_count`); per-node packed data supports at most 8 local edges; the hierarchy builder splits roads over levels by class and links the copies of an intersection with node transitions; shortcuts are written at the front of a node's edge run.

Consequence: copying tiles between builds does not connect roads, and any renumbering inside a tile invalidates every reference to it.

## 3. Pipeline

```
source prep -> stock build -> match to ledger -> dirty set -> rewrite -> validate -> publish
```

1. **Source prep.** Fetch and verify OSM extracts, merge neighbours, clip to the region polygon plus a border buffer, record SHA-256 of every input. Tile ownership follows the global tile grid, not the clip polygon: a tile must be built with complete data around it.
2. **Stock build.** Run the pinned Valhalla build with `keep_osm_node_ids: true`. Output: *raw tiles* with provisional IDs. These are an intermediate product, never published.
3. **Match.** Extract every node and directed edge from the raw tiles, compute its stable key, and look it up in the ledger. Result: retained, new, deleted and changed (attributes only) entities.
4. **Dirty set.** Tiles that contain any new, deleted or changed entity, plus tiles that reference a relocated node or an affected cross-tile edge (found through a reverse index in the ledger). All other tiles are carried over from the previous release byte for byte and are not rewritten.
5. **Rewrite.** For dirty tiles only, produce the final tile applying the ID rules in section 5.
6. **Validate.** Integrity checks, byte-identity of untouched tiles, routing comparison against a stock full build (section 9).
7. **Publish.** Per-tile content hashes in a manifest; ledger and manifest are committed atomically (section 7).

## 4. Identity ledger

Lives on the build server only; never shipped to devices.

Stable keys (initial proposal, refine experimentally):
- **Node:** the OSM node ID, read from the edge info when the stock build runs with `keep_osm_node_ids: true` (to be verified, Q12). Fallback: coordinates plus incident OSM way IDs with a small displacement tolerance, so a mapper nudging a node by a few metres does not look like delete + create.
- **Directed edge:** OSM way ID + start node key + end node key + direction + segment position (for ways split into several edges).

The ledger stores, per key, the assigned graph ID and a revision. It also stores a reverse index (node -> tiles that reference it) used to compute the dirty set cheaply.

The ledger is the source of truth for ID stability. If it is lost, IDs cannot be preserved and the fallback is a full epoch reset (all users re-download). It must be versioned, backed up and published atomically with each release.

## 5. Tile rewrite rules

Conceptual model: the **new stock build defines the content**, the **ledger defines the labels**. The rewriter relabels the new build's tile so that retained entities get their old position, and pads with dead slots where the old build had entities that no longer exist.

For a dirty tile:
- **Node slots are stable.** A node keeps its array position (its graph ID) for as long as the ledger holds it.
- **Retained edge with unchanged node block:** keeps its ID; data fields (speed, names, shape, access) come from the new build.
- **New edge:** appended at the end of the edge array.
- **Deleted edge or node:** becomes a **dead slot**: the old record with forward and reverse access cleared, keeping a structurally valid end node, `leaves_tile` flag and non-zero speed and length, because forward search computes the edge cost before the access check. Dead nodes are never removed before compaction, so dead edges keep valid end nodes.
- **Node whose outbound block changed** (an edge added or removed, or the order of its regular edges changed): the node slot stays, but its `edge_index` is pointed at a **new block** appended at the end; the old block becomes dead slots. The node's outbound edges get new IDs. Inbound edges (owned by neighbours) keep their IDs and their `endnode`, but their `opp_index` is recomputed. This is the only ripple beyond the tile itself, and it is smaller than relocating the node.
- **Intersection unit.** Local edge indices (`localedgeidx`, turn types, stop impact, headings, restriction masks) are positions among the node's regular edges and are shared by all hierarchy levels of the same intersection. When the edge set or order at an intersection changes, the node blocks of all its levels, found through the transitions, are rebuilt together, and the end-node `restrictions_` masks of the edges that arrive there are taken from the new build.
- **Always patched in dirty tiles:** `opp_index` of inbound edges, complex-restriction edge IDs, bins, tile-internal tables keyed by edge index (access restrictions, turn lanes, signs, lane connectivity), offsets and header counts.

Tile growth from dead slots is bounded by periodic per-tile **compaction** (section 8).

## 6. Hierarchy, shortcuts and seams

- **Levels.** Edges live on exactly one level by road class (level 0: 4-degree tiles, level 1: 1-degree, level 2: 0.25-degree); an intersection that touches several classes is duplicated per level and linked by node transitions. Local-level tiles reference upper levels only through those transitions.
- **Shortcuts.** Shortcut edges are real entries at the **front** of an upper-level node's edge block, so a change in the number of shortcuts shifts the positions of the base edges. The rewriter reserves a fixed number of shortcut slots per upper-level node (at most 7 are ever created, `kMaxShortcutsFromNode`), filled with the new build's shortcuts and padded with dead slots, so base edge IDs stay stable. Shortcut contents are derived data and are recomputed every build.
- **Seams.** A seam is where a road crosses into a tile owned by another region. Regions are built in sequence. The first build assigns the IDs of seam nodes; later builds **pin** those IDs (match by OSM node ID, available when the stock build runs with `keep_osm_node_ids`) instead of allocating new ones. Seam tiles have one authoritative copy, with a revision recorded in the manifest.
- **Seam dependencies.** A change at a seam node touches tiles on both sides. The manifest records, for each seam tile, which neighbour revisions it requires.

## 7. Update and distribution protocol

- Manifest: tile path -> content hash, size, revision, plus seam dependencies and the build-set identifier.
- A device keeps a local manifest. An update downloads exactly the tiles whose hash differs, for **all** installed regions together and applied atomically (swap on success).
- Adding a region brings the device to the current build set first.
- Runtime check: before cross-region routing over a seam, verify that both sides are at compatible revisions. If not, do not route natively across that seam (the client falls back to its own handling) instead of routing over a mismatched pair.
- Publication: immutable per-tile objects plus a small `latest` pointer written last (same pattern as the existing R2 publication, see `REUSE.md`).

## 8. Epochs, compaction, version bumps

- **Compaction:** rebuilding a tile without dead slots changes IDs inside it and in its referrers; do it rarely and per tile.
- **Epoch reset:** a full rebuild with new IDs, needed if the ledger is lost, the key scheme changes, or Valhalla's tile format changes. Users re-download once.
- **Valhalla version:** pin it (production currently uses 3.6.3). A version bump is a migration event and an epoch boundary unless the rewriter is ported and proven.

## 9. Validation

1. **Determinism:** same input, same version and config produce identical tile bytes (precondition for hash-based deltas). The header build date (`date_created_`) must be normalized first.
2. **Byte identity:** tiles outside the dirty set are identical to the previous release.
3. **Integrity checker:** every end node exists; opposing edges are symmetric; transitions are consistent; bins contain only live edges; no reference to a dead slot from a live entity.
4. **Routing oracle:** a stock full build of the same data is the reference. Compare routes (cost, time, distance) on a large random sample, including cross-seam routes, in both directions and for each costing profile in use.
5. **Seam acceptance:** real cross-border routes across independently built regions succeed natively and match the oracle, in both directions.

## 10. Main risks

- Valhalla may not tolerate dead slots or non-canonical ordering somewhere in loki/thor/meili code paths.
- The rewriter depends on internal tile structures; maintenance cost on each Valhalla upgrade.
- Non-deterministic stock builds would make hash deltas useless; this is the first thing to measure.
- Upper-level tile size may make "small deltas" less small than hoped.
- Key matching quality (splits, merges, retagging, nudged nodes) decides how much ripple real updates cause.
