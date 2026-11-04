# Valhalla Graph Studio (desktop app)

Linux workstation (Tauri 2 + TypeScript + MapLibre) for building, validating, inspecting, comparing and publishing Valhalla routing graphs. Imported from RoadPilot Graph Studio (`RoadPilot-Offline-Data`, commit `f29efad`) and stripped of its F8 cross-region and border/handoff features. See [`../docs/IMPORT_PROVENANCE.md`](../docs/IMPORT_PROVENANCE.md).

It is the future home of the ID-stable builder tooling (identity ledger, dirty-set and tile-diff views). Those parts do not exist yet; see [`../docs/ROADMAP.md`](../docs/ROADMAP.md).

## What it does today

- Region catalog from `config/regions`, region editor, Geofabrik discovery with buffer preview.
- Multi-region build queue with live logs, cancellation, CPU / RAM / disk / thermal monitoring and build history.
- Graph inspector: Valhalla graph tiles rendered as vector layers, click-to-inspect, deep `locate` on the exact local graph.
- Standard Valhalla route planner (costing options, geometry, maneuvers, search expansion).
- Compare retained builds (graph and boundary fingerprints, road-level diff).
- Cloudflare R2 publication manager (immutable releases, latest pointer, history, rollback).
- Visual-map (PMTiles) and Search/POI (SQLite) inspectors and builds, kept from the original app.
- Read-only headless inspection CLI (`--ai-capabilities`, `--ai-inspect`, `--ai-diagnose`, `--ai-verify-route`), see [`docs/AI_CONTROL.md`](docs/AI_CONTROL.md).

Removed: Native F8 connectivity panel, Border inspector (manual handoffs, handoff artifacts, connector matrix, border diagnostics export) and the Graph A/B border overlays.

## Local workspace

```text
~/ValhallaGraphStudio/
  work/            cached Geofabrik sources + unpacked build workspace
  builds/          immutable routing TARs, manifests and checksums
  visual-builds/   PMTiles visual versions
  search-builds/   Search/POI SQLite versions
```

The workspace directory is deliberately different from the RoadPilot app's (`~/RoadPilotGraphStudio`) so both can be installed side by side. Existing builds are not shared; copy them manually if needed.

## Host requirements

- Python 3 + venv (packages from `requirements-routing.txt`, installed automatically on first build)
- `osmium`
- Valhalla 3.6.3 on `PATH`: `valhalla_build_config`, `valhalla_build_timezones`, `valhalla_build_admins`, `valhalla_build_tiles`, `valhalla_build_extract`, `valhalla_service`
- For building the app: Node 22, Rust stable, and on Linux the Tauri system libraries (`libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `libsoup-3.0-dev`, `libssl-dev`, `patchelf`).

## Development

```bash
cd graph-studio
npm install
npm run tauri dev      # runs `tauri icon` first to generate the icons from app-icon.svg
```

Checks used when importing: `npx tsc --noEmit`, `npx vite build`, `cargo check` in `src-tauri/` (all clean), and `python3 -m pytest tests` from the repository root.

## Known leftovers to review

- The `visual/` and `search/` pipelines are RoadPilot data products, not graph tooling; decide later whether to keep them here.
- `src/main.ts` (about 4,400 lines) and `src-tauri/src/lib.rs` (about 2,900 lines) are monoliths; split them before adding the ledger views.
