# Open questions about Valhalla internals

Status of the assumptions the design relies on. **Source reading done 2026-11-04 on Valhalla tag `3.6.3` (commit `e2f017b1`)**; paths below are relative to that tree. "Answered" means answered by reading the code; nothing has been executed yet, so every conclusion that the design depends on still needs the experiment named in `ROADMAP.md`.

| # | Question | Status |
|---|---|---|
| Q1 | Ordering rule for node and edge indices | Answered |
| Q2 | Are a node's edges contiguous? | Answered: yes, required |
| Q3 | Limits of per-node packed data | Answered |
| Q4 | Tolerance for dead nodes and edges | Partly: no blocker found in code, needs an experiment |
| Q5 | What references an edge or node | Answered |
| Q6 | Shortcuts and their references | Answered, with a design correction |
| Q7 | Determinism of the stock build | Open (needs a build) |
| Q8 | Header fields that vary per build | Partly |
| Q9 | Can the tile builder write arbitrary layouts? | Partly |
| Q10 | Format stability across versions | Answered |
| Q11 | Tile bins | Partly |
| Q12 | Other stable identifiers | Answered |
| Q13 | Restartable build stages | Answered |

## Answers

**Q1 - ordering.** Graph nodes are sorted by tile, then by a spatial grid cell within the tile (`grid_divisions_within_tile`, default 32 in `scripts/valhalla_build_config`), then by OSM node ID; the node's index in the tile is its rank in that order (`src/mjolnir/graphbuilder.cc:43-100`). Consequence: adding one node renumbers every later node in that tile, which in turn changes every reference to them. This confirms that stock builds are not ID-stable and that a ledger is needed.

**Q2 - contiguity.** Yes. A node stores only `edge_index_` (21 bits) and `edge_count_` (7 bits, at most 127: `kMaxEdgesPerNode`) (`valhalla/baldr/nodeinfo.h:508-509`, `:17`), and all search code loops `for i < nodeinfo->edge_count()` over a contiguous run (`src/thor/unidirectional_astar.cc:98`). The edge array position is the edge's ID, so a node's edges cannot be grown in place. A node's **slot** can stay where it is while its `edge_index_` is pointed at a new block at the end of the array.

**Q3 - packed per-node data.** Several fields are indexed by a *local edge index* of at most `kMaxLocalEdgeIndex = 7` (`nodeinfo.h:22`): node `headings_` (8 x 8 bits), `local_driveability_` (16 bits), `local_edge_count_` (3 bits); per edge `turntype_` (24 bits), `stopimpact_` (24 bits), `edge_to_left_/right_` (8-bit masks), `name_consistency_` (8 bits), and the `restrictions_` mask, which is indexed by the local index at the *end* node (`valhalla/baldr/directededge.h:1210-1307`, `nodeinfo.h:506-545`). `localedgeidx_` is the edge's position among the node's regular (non-shortcut) edges, assigned at build time in the original edge order (`src/mjolnir/directededgebuilder.cc:85`), and shared across hierarchy levels of the same intersection. Consequences: there is no room for dead edges *inside* a node's regular edges; if the edge set or edge order at an intersection changes, the local indices of all its levels change together, so an intersection must be treated as one unit across levels (see `DESIGN.md` section 5).

**Q4 - dead nodes and edges (partial).** Reading the search code:
- Reverse expansion skips an edge early when `reverseaccess` does not match the mode (`unidirectional_astar.cc:186`); `dijkstras.cc:132` does the same for both directions.
- Forward expansion still looks up the end node and computes the edge cost *before* the costing's access check (`unidirectional_astar.cc:212-237`). So a dead edge must keep a **structurally valid** `endnode_`, `leaves_tile_` and sane speed and length values, not just cleared access bits.
- Hence the proposed encoding: a dead slot is the old record with forward and reverse access cleared; dead nodes stay in the array so dead edges' end nodes remain valid.
- Not yet checked: loki (`src/loki`), meili (map matching), isochrone/matrix code paths, and any code that sums or iterates over all edges of a tile. An experiment (Phase 4) must route and match through a tile with dead slots in all costing profiles.

**Q5 - references to edges and nodes.** Edge `endnode_` is a full 46-bit graph ID (`directededge.h:1210`); `opp_index_` is the opposing edge's index *within the end node's block* (7 bits) and is recomputed in the validate stage (`src/mjolnir/graphvalidator.cc:396-403`); node transitions hold graph IDs of the same intersection on other levels (`valhalla/baldr/nodetransition.h`); complex restrictions hold from/to/via edge graph IDs (`complexrestriction.h`); bins hold edge graph IDs (`graphtileheader.h:693-694`); access restrictions, turn lanes, signs and lane connectivity are keyed by directed edge index inside the tile (`accessrestriction.h:82`, `turnlanes.h:203`). Predicted speeds and external traffic data are keyed by edge position (not used here yet).

**Q6 - shortcuts.** Shortcut edges are real entries in the node's edge block. The builder writes a node's **shortcuts first, then the base edges** (`src/mjolnir/shortcutbuilder.cc:643-659`). References are node-local masks (`shortcut_`, `superseded_`, 7 bits each) and at most `kMaxShortcutsFromNode = 7` per node (`graphconstants.h:201`). **Design correction:** because shortcuts come first, any change in the number of shortcuts at an upper-level node shifts the positions, hence the IDs, of its base edges. The rewriter must either reserve a fixed number of shortcut slots at the front of each upper-level node block, or relocate the block when the count changes.

**Q7 - determinism.** Open. The build is multithreaded; it can only be settled by building twice (Phase 1).

**Q8 - header fields.** The header carries a version string (16 bytes), `dataset_id_` (maximum OSM changeset ID of the input) and `date_created_` (the build date in days) (`graphtileheader.h:25,486,619-622,691`; `graphtilebuilder.cc:1111`). `date_created_` changes with the build day, so it must be normalized or excluded from tile hashes. Other varying fields not yet inspected.

**Q9 - tile builder.** `GraphTileBuilder::Update(nodes, directededges)` replaces the node and directed-edge arrays (`valhalla/mjolnir/graphtilebuilder.h:70`); the other tables (edge info, signs, restrictions, names) have their own add methods, and bins can only be added after the tile is stored (`graphtilebuilder.cc:358`). Whether arbitrary re-layout is possible through this API, or the rewriter must write the tile format itself, is still open.

**Q10 - format stability.** The reader only warns when the *major* version differs and carries on (`src/baldr/graphtile.cc:320-322`). Tile layout changes inside a major version are therefore not detected; the build version must be recorded in our own manifest and checked by us.

**Q11 - bins (partial).** Each tile has a 5 x 5 grid of bins holding graph IDs of edges that intersect each bin, including edges from other tiles passing through ("tweeners") (`graphtileheader.h:36-38`; `graphtilebuilder.cc:1114-1173`). Bins are built after the tile is stored. The exact rule for which tile holds which tweener still has to be traced.

**Q12 - other identifiers.** With `mjolnir.keep_osm_node_ids: true` the OSM node IDs of graph nodes are stored in each edge's edge info, and `keep_all_osm_node_ids` stores all shape nodes (`graphbuilder.cc:506-529`, `scripts/valhalla_build_config:143-144`). Edge info also stores the OSM way ID. This gives exact node keys inside the tiles and removes the need for coordinate matching in most cases. To verify: the C++ accessor for these tags, and the tile size cost.

**Q13 - build stages.** `valhalla_build_tiles` takes `--start` and `--end` stages: initialize, parseways, parserelations, parsenodes, constructedges, build, enhance, filter, transit, bss, hierarchy, shortcuts, restrictions, elevation, validate, cleanup (`valhalla/mjolnir/util.h:21-39`). The validate stage computes opposing indices. Bin building and the other late stages happen after hierarchy and shortcuts.

## Consequences for the design (applied in `DESIGN.md`)

1. Node **slots** stay fixed; a changed node gets a new edge **block**, not a new node ID.
2. Treat an intersection as one unit across hierarchy levels when its edge set or edge order changes.
3. Reserve shortcut slots in upper-level nodes.
4. Dead slot = old record with access cleared and a valid end node; dead nodes are never removed until compaction.
5. Run stock builds with `keep_osm_node_ids: true` to get exact node keys.
6. Record the Valhalla version in our manifest; the tiles do not enforce it.
7. Normalize the header build date before hashing.
