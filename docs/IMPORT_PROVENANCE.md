# Import provenance

Date: 2026-11-04. Source: `kandyman991/RoadPilot-Offline-Data`, commit `f29efad`. The source repository was not modified.

## Copied as-is (layout kept so Tauri resource paths still work)
`config/`, `schemas/`, `visual/`, `search/`, `requirements-*.txt`, and `tools/*.py` except the groups below.

## Copied and modified
- `graph-studio/` (Tauri app). Removed the Native F8 connectivity panel, the Border inspector and the Graph A/B overlays: about 1,300 lines of `src/main.ts`, 1,600 lines of `src-tauri/src/lib.rs`, 180 lines of CSS, 26 Tauri commands. Renamed the app (package, crate, bundle identifier, product name) and the workspace directory to `~/ValhallaGraphStudio` so it can coexist with the RoadPilot app.
- `tests/`: kept the 8 generic tests, dropped the three that assert the old UI source text (they require the F8 and border panels).

## Not imported (old architecture)
`tools/` files matching `*f8*`, `*connectiv*`, `*connector*`, `*frontier*`, `*crossing*`, `*handoff*` (29 files), `tools/legacy_archive/`, the F8 and retired-architecture documents, and the F8 / historical-fixture tests and fixtures.

## Seed only
`tools/native-f8-edge-scan/*.cpp` and `CMakeLists.txt` are in `native/edge-scan-seed/`, unwired.

## Verification done on import
- `npx tsc --noEmit`: clean. `npx vite build`: succeeds.
- `cargo check` (src-tauri): clean, no warnings, after installing the Tauri Linux libraries.
- `python3 -m py_compile tools/*.py`: ok. `python3 -m pytest tests`: 27 passed, 1 skipped.

## Not verified
- The app was not launched (no display here); no build, route or publication was run.
- No Valhalla binaries were available, so the build pipeline and the C++ seed were not executed.
- Packaging (AppImage / deb) was not attempted.

## Re-syncing from the source
There is no automatic sync. Compare against `RoadPilot-Offline-Data` by hand for fixes to the shared build tooling (`tools/build_routing_region.py`, `source_cache.py`, R2 scripts) and port them deliberately.
