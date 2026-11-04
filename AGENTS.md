# Valhalla Graph Studio - repository agent instructions

## Resumption
1. Read `CHAT_HANDOFF.md`, this file, `docs/DESIGN.md`, `docs/ROADMAP.md` and `docs/DECISIONS.md`.
2. Read `docs/FAILED_APPROACHES.md` and `docs/OPEN_QUESTIONS.md` before proposing or reviving a design.
3. Check the live GitHub state (main SHA, open PRs, issues, CI). The handoff is a curated note and can be stale; GitHub is authoritative.

## Rules
- Statements about Valhalla internals are assumptions until answered in `docs/OPEN_QUESTIONS.md` with a source reference or an experiment. Do not present them as fact.
- Distinguish planned, implemented, tested and proven. Documentation is not evidence.
- One variable per experiment; record inputs, versions and SHA-256 hashes. Reproducible scripts, no hidden manual steps.
- Stock Valhalla is the routing reference. Validate rewritten graphs against a stock full build.
- This project has no production effect. Do not edit `RoadPilot-Offline-Data` or the Android app from here. Production use needs the approval in `docs/CONTRACT_INTERACTION.md`.
- Heavy builds run locally, not in cloud CI. Use `[skip ci]` for documentation-only commits.
- Never commit credentials, private paths, signing keys, OSM extracts, graph tiles or the identity ledger (see `.gitignore`).

## After a milestone
Update `CHAT_HANDOFF.md` (what changed, verified SHA/PR/CI, evidence, next exact action). Record material decisions in `docs/DECISIONS.md` and rejected or inconclusive approaches in `docs/FAILED_APPROACHES.md` in the same PR.
