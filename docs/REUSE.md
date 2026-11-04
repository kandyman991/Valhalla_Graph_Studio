# Reuse from the other repositories

**Update 2026-11-04:** the Graph Studio desktop app and its build pipeline have been imported into this repository (see `IMPORT_PROVENANCE.md`). The table below still describes the sources; "Status" says what happened to each.

Reviewed level: **read** = opened and read the relevant part on 2026-11-04; **listed** = seen by file name and size only, not yet reviewed. Verify before adopting anything marked *listed*.

Preferred method: copy and adapt small, self-contained files (keeping a note of the source commit); reference by link anything large or still changing. Do not create a dependency on a private repository for code that must build elsewhere.

## From `RoadPilot-Offline-Data` (public)

| Path | Size | Reviewed | What to reuse | How |
|---|---|---|---|---|
| `tools/native-f8-edge-scan/CMakeLists.txt` | 18 lines | read | libvalhalla via pkg-config, C++20, compile-time Valhalla version binding | **Copied** to `native/edge-scan-seed/` (unwired); template for the extractor and rewriter build |
| `tools/native-f8-edge-scan/native_f8_edge_scan.cpp` | 179 lines | read (head) | GraphReader / tile hierarchy usage, read-only tile iteration, version guard | Adapt into the Phase 2 entity extractor |
| `tools/native-f8-edge-scan/native_f8_directed_endpoints.cpp` | 151 lines | listed | Directed-edge endpoint extraction | Adapt for end-node / opposing-edge dumping |
| `tools/build_routing_region.py` | 715 lines | listed (header and docs read); **imported** | Buffered clip, Valhalla build driver, manifest and SHA-256 recording | Base for the "source prep" and "stock build" stages |
| `tools/source_cache.py`, `geofabrik_source_selection.py`, `osmium_source_merge.py`, `prepared_region_cache.py` | 200 / 50 / 55 / 73 lines | listed | Geofabrik download with md5 sidecars, source merge, prepared-input cache key (source hashes + buffer hash + Valhalla version) | Copy and adapt for source prep |
| `tools/validate_routing_pack.py` | 103 lines | listed | Extract checks, smoke routes, size and hash checks | Starting point for the validator |
| `tools/prepare_routing_publication.py`, `publish_routing_pack_r2.py`, `r2_publication_backend.py`, `docs/r2-publication.md` | - | listed | Immutable versioned objects plus `latest.json` pointer written last | Pattern for tile-level publication (Phase 7) |
| `config/regions/*.json`, `schemas/` | - | listed | Region configuration format (polygon, buffer km, Valhalla version) | Reuse the format for region definitions |
| `tools/audit_f8_graph_edge_parity.py`, `docs/F8_NATIVE_PARITY_AUDIT.md` | - | listed | Parity comparison against the F8 reference | Final acceptance evidence (Phase 8) |
| `graph-studio/` (Tauri app) | 5.7k TS + 4.5k Rust | read (structure, commands) | Desktop UI shell | **Imported** (F8 / border features removed, renamed). Later: ledger and dirty-set views |
| `AGENTS.md`, `docs/ARCHITECTURE_MEMORY.md`, `docs/FAILED_APPROACHES.md` | - | read | Continuity and failure-history process | Already adapted in this repository |

Not reused: the retired OSM-based crossing discovery and connector-matrix code (retired there by decision D-014).

## From `valhalla-graph-lab` (private)

| Path | Reviewed | What to reuse |
|---|---|---|
| `docs/EXPERIMENTS.md` | read | Experiment record template and the one-variable-at-a-time rule |
| `experiments/001-identical-builds/` | read | Determinism experiment, becomes Phase 1 |
| `docs/DECISIONS.md` (D-001 to D-006) | read | Andorra as first dataset, no planet builds first, full build as oracle, determinism first |
| `docs/ARCHITECTURE.md` sections 1-8 | read | Module, stable identity and recomposer ideas; this project narrows them to a ledger-based rewriter |
| `docs/ARCHITECTURE.md` section 9 (virtual portal layer) | read | Alternative path the lab recorded; not used here |

## From `RoadPilot-Seam-Compiler` (public)
Only a README, license (Apache-2.0) and process documents exist. Nothing to reuse except possibly the license choice.

## Not yet reviewed
`SmartRide-Next` (Android app, private): the consumer side of the update protocol. Review when Phase 7 starts.

## Other candidates found during the UI import (not imported)
- `tools/audit_f8_graph_edge_parity.py`, `tools/probe_native_f8_*.py`, `docs/F8_*.md`: F8 parity and journey probes. Not needed by the new design; revisit only for the final cross-region acceptance comparison.
- `docs/ENGINEERING_DECISIONS.md`, `docs/CONTINUITY_PLAYBOOK.md`: process documents; the relevant parts were adapted into `AGENTS.md`.
- `.github/workflows` for AppImage / deb packaging of the app: not reviewed yet; needed when this repository should build releases.
