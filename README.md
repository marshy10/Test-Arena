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
| `content-model.md` | Field-level content constraints, the field-data bundle and open gaps G1–G32 |
| `PRD.md` | Product requirements |
| `5g_signalling_workflows*.json` | The 3GPP signalling workflow library the games are built on |
| `field_lake/` | Field-data lake demo (Python + DuckDB); see below |

## Field-data lake

`field_lake/` stores field data in a small local lakehouse and exports a sample for the app. The data includes signalling flows from UE logs and PCAP, RF measurements, fading profiles, channel models and raw I/Q. It is offline tooling, separate from `index.html`. Its bundle is embedded in the app: lake sessions seed the Briefing's source captures, and Settings → Field-data lake lists them and can import a newer bundle. Setup, adding real files and the scale-up path are in [field_lake/README.md](field_lake/README.md). Quick start:

```bash
python -m venv C:\data\field-lake\.venv
C:\data\field-lake\.venv\Scripts\python -m pip install -r field_lake/requirements.txt
C:\data\field-lake\.venv\Scripts\python field_lake/run_all.py --samples
C:\data\field-lake\.venv\Scripts\python field_lake/embed_bundle.py
```

## Placeholders

Failure thresholds, reward weights, difficulty parameters and the capture schema (gaps G1–G4) are placeholder values in the `CONFIG` block at the top of the script in `index.html`, each marked `// PLACEHOLDER (Gn)`.
