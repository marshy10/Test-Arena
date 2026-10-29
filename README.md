# Test Arena

*Don't write tests. Play the network until it breaks.*

Test Arena turns AI-RAN and modem testing into an endless, adversarial game. An AI-trained UE cohort plays against the device under test (DUT) until the DUT breaks or the engineer calls time.

This repository holds a static, playable demo. The game loop is a seeded simulation that runs in the browser; there is no backend.

## Run it

Open `index.html` in any modern browser. There is nothing to install or build.

- `?debug=1` shows an fps and tick overlay and exposes `window.__arena` for automated acceptance tests.
- `?debug=1&speed=40` fast-forwards game time, which makes the DUT-failure path quick to demo.

## Files

| File | What it is |
| --- | --- |
| `index.html` | The whole app: HTML, CSS and JavaScript in one file |
| `CLAUDE.md` | Project brief for AI coding agents |
| `site-contract.md` | Screens, fields, behaviour and acceptance criteria (source of truth) |
| `design-standard.md` | "Neo-Tokyo test dojo" visual specification and tokens |
| `content-model.md` | Field-level content constraints and open gaps G1–G12 |
| `PRD.md` | Product requirements |

## Placeholders

Failure thresholds, reward weights, difficulty parameters and the capture schema (gaps G1–G4) are placeholder values in the `CONFIG` block at the top of the script in `index.html`, each marked `// PLACEHOLDER (Gn)`.
