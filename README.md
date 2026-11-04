# Valhalla Graph Studio

An **ID-stable Valhalla graph builder**: tooling to build Valhalla routing graphs region by region, in a way that keeps graph IDs compatible across builds, so that

- regions built at different times still connect natively at their borders, and
- an OSM update changes only the tiles that really changed, so users download a small delta instead of whole regions.

> **Status: design phase for the ID-stable builder (no ledger or rewriter code yet). The imported desktop app and build tooling are present but unused for this purpose so far.**
> Nothing here is approved for production use. See [`docs/CONTRACT_INTERACTION.md`](docs/CONTRACT_INTERACTION.md).

Not to be confused with the *RoadPilot Graph Studio* desktop app in `RoadPilot-Offline-Data`; this project is a graph-builder research and tooling effort that may later feed it.

## The problem

Valhalla tiles reference each other by binary IDs (hierarchy level, tile, index inside the tile) and by positional indices (end node, opposing edge, hierarchy transition). Two independent builds assign these IDs independently, so:

- tiles from different builds cannot simply be placed in the same tile directory; roads that cross a region border do not connect;
- rebuilding anything can renumber large parts of a tile set, so a naive update means re-downloading everything.

## The approach

1. Build with **stock Valhalla** (pinned version) as the source of all routing logic.
2. Keep an **identity ledger** on the build server that maps stable keys (OSM-derived identity of every node and directed edge) to fixed graph IDs.
3. After each stock build, run a **tile rewriter** that applies the ledger: retained entities keep their ID, new entities are appended, deleted entities become dead slots, and only the affected ("dirty") tiles are rewritten.
4. Build regions **one after another**: the region built first fixes the IDs at the seam; later regions adopt them.
5. Publish **tile-level manifests** so a device downloads only the tiles whose content changed.

Details: [`docs/DESIGN.md`](docs/DESIGN.md). Plan: [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Desktop app (imported)

`graph-studio/` is a Tauri + TypeScript workstation for building, inspecting, comparing and publishing Valhalla graphs, imported from RoadPilot Graph Studio without its F8 cross-region and border/handoff features. It compiles and type-checks; it has not been launched or used for a real build here. See [`graph-studio/README.md`](graph-studio/README.md) and [`docs/IMPORT_PROVENANCE.md`](docs/IMPORT_PROVENANCE.md). Its build pipeline lives in `tools/`, `config/`, `schemas/`; `native/edge-scan-seed/` holds unwired C++ seed code for the tile reader.

## Documentation

| Document | Purpose |
|---|---|
| [`docs/DESIGN.md`](docs/DESIGN.md) | Architecture of the ledger, rewriter, seams and update protocol |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | Phased proof-of-concept plan with pass criteria |
| [`docs/OPEN_QUESTIONS.md`](docs/OPEN_QUESTIONS.md) | Assumptions about Valhalla internals that must be verified in its source before relying on them |
| [`docs/DECISIONS.md`](docs/DECISIONS.md) | Decision log |
| [`docs/FAILED_APPROACHES.md`](docs/FAILED_APPROACHES.md) | Rejected and inconclusive approaches, with status definitions |
| [`docs/REUSE.md`](docs/REUSE.md) | What can be reused from the other repositories |
| [`docs/IMPORT_PROVENANCE.md`](docs/IMPORT_PROVENANCE.md) | What was imported from RoadPilot-Offline-Data, what was removed, what was verified |
| [`docs/CONTRACT_INTERACTION.md`](docs/CONTRACT_INTERACTION.md) | How this project conflicts with the RoadPilot architecture contract, and what approval is needed |
| [`AGENTS.md`](AGENTS.md), [`CHAT_HANDOFF.md`](CHAT_HANDOFF.md) | Continuity rules and current state for resuming work |

## Related repositories

- `kandyman991/RoadPilot-Offline-Data`: production regional pack builder and RoadPilot Graph Studio. Source of reusable build tooling (see `docs/REUSE.md`).
- `kandyman991/valhalla-graph-lab` (private): earlier research on modular graph composition (Andorra test area, determinism experiment 001).
- `kandyman991/SmartRide-Next`: the RoadPilot Android app, which consumes the packs.

## License

Not chosen yet.
