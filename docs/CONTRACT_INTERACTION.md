# Interaction with the RoadPilot architecture contract

**Status: conflict flagged. This repository has no effect on production until the steps below are done in the RoadPilot repositories.**

## Conflicts

`RoadPilot-Offline-Data` has a binding contract (`docs/ARCHITECTURE_CONTRACT.md`) and decision log. This project proposes behaviour that departs from:

| Clause | Text (summary) | How this project differs |
|---|---|---|
| Contract A3 | Each region is an immutable, independently built graph. No direct cross-graph sharing or comparison of GraphIds; never modify raw `.gph` files. | Graph IDs are deliberately made consistent across builds, and final tiles are produced by a rewriter, not taken raw from a stock build. |
| `ENGINEERING_DECISIONS.md` D-002 | Never mutate raw `.gph` to repair seams. | Seams are made native by pinned IDs in rewritten tiles. |
| `routing-packs-v1.md` "Independence rule" | Cross-region compatibility is not expressed by requiring graphs from one synchronized build. | Partly preserved: regions still build and ship independently. Changed: seam tiles carry cross-region dependencies tracked in a manifest. |
| `FAILED_APPROACHES.md` (native F8 entries) | No direct cross-build GraphId equivalence. | This project's premise is exactly such an equivalence, enforced by the ledger. |

What is preserved: independent region versions and downloads, exact version pinning, fingerprint-bound artifacts, and F8 as the runtime fallback.

## Why it is acceptable to work here

The contract governs RoadPilot Android and the RoadPilot Graph Studio production pipeline. This repository is a separate project and changes none of their code. Research and tooling here do not violate the contract.

## What is needed before any production use

Per contract section E, a PR in `RoadPilot-Offline-Data` must:
1. cite the exact superseded clauses (table above);
2. give the technical justification and evidence from Phases 1-6 of this roadmap;
3. include a real comparison against F8 on the same graph fingerprints (contract B4/D3);
4. reference a tracked issue;
5. record explicit approval by the owner as a superseding decision.

Approval to work on this project is not approval to change the contract.
