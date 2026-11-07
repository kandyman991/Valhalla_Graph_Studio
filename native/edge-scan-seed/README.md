# edge-scan-seed

Unwired seed code copied from `RoadPilot-Offline-Data/tools/native-f8-edge-scan` (commit `f29efad`). Read-only C++ programs that open a Valhalla tile set through `baldr::GraphReader` and walk directed edges.

They contain F8-specific eligibility predicates and a window scan; **do not treat them as the entity extractor**. They are here as working examples of: linking libvalhalla via pkg-config, binding the compile-time Valhalla version, tile hierarchy iteration and directed-edge access. Phase 2 of `docs/ROADMAP.md` replaces them with an extractor that dumps every node and directed edge (graph ID, coordinates, incident way IDs, end node, opposing index).

Not built or tested in this repository (no libvalhalla available in the import environment).
