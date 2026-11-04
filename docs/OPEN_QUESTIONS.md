# Open questions (verify in the pinned Valhalla source)

These are assumptions the design relies on. None is verified yet. Answer each with a source reference (file, function, version) or an experiment, then record the result here.

| # | Question | Why it matters | Status |
|---|---|---|---|
| Q1 | How are node and edge indices assigned inside a tile (ordering rule)? Does inserting one node shift all later indices? | Decides how far a small change ripples in stock builds | open |
| Q2 | Are a node's directed edges required to be contiguous, and are other layouts accepted by the readers? | Basis of the "relocate node" rule | open |
| Q3 | Which per-node data is packed for a limited number of local edges (stop impact, turn types, headings, restriction masks)? What is the limit? | Dead slots inside a node's edge list may be impossible | open |
| Q4 | Does Valhalla (loki, thor, meili, costing) tolerate dead nodes and dead edges (no access, not binned)? Any code that iterates all edges of a tile or assumes density? | Core of the tombstone approach | open |
| Q5 | What exactly references an edge or node by ID: opposing index, end node, transitions, complex restrictions (from/via/to), bins, traffic/predicted-speed data, OSMLR associations? | Defines the full patch list | open |
| Q6 | Are shortcuts referenced from outside their own node (superseded masks, other tiles)? Can they be regenerated without touching lower-level tiles? | Decides whether upper-level changes ripple down | open |
| Q7 | Is the stock build deterministic for identical input, config, version and thread count? Across thread counts? Across machines? | Precondition for hash-based deltas | open |
| Q8 | Which parts of the tile header (build date, version, checksums) change on every build and must be normalized? | Hash stability | open |
| Q9 | Can `GraphTileBuilder` (libvalhalla) produce arbitrary node/edge layouts, or must the rewriter write the tile format itself? | Effort estimate for the rewriter | open |
| Q10 | Is the tile format stable across the Valhalla versions we may use (3.6.x)? What changes in newer releases? | Migration and epoch policy | open |
| Q11 | How are tile bins computed and which tiles hold bins for edges crossing a tile boundary? | Dirty-set computation at seams | open |
| Q12 | Are there other stable identifiers worth using as keys (way ID in edge info, OSM node ID in intermediate build files)? | Quality of ledger keys | open |
| Q13 | Which Valhalla build stages can be rerun or restricted to a tile subset? Does any stage write outside the tiles it owns? | Whether partial stock builds are possible | open |
