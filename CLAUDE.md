# CLAUDE.md — Test Arena

## Project identity

Test Arena turns AI-RAN and modem testing into an endless, adversarial game. An AI-trained UE cohort plays against the device under test (DUT) until the DUT breaks or the engineer calls time. This repo is a static, playable demo of that experience; the game loop is simulated in the browser.

There are two ways to play:

- **Stress games:** four endless load games (Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze).
- **Procedure Ladder:** five levels played in order (L1 Registration → L2 Data Call → L3 Handover & Mobility → L4 Throughput → L5 IMS Voice & Video). Each level uses a different game-theory model (sequential, coordination, Stackelberg, congestion/Nash bargaining, alternating-offers bargaining), has its own branch set and KPIs, and unlocks the next level when cleared.

There are two play modes, for both gNB / AI-RAN and modem targets:

- **Manual:** the engineer chooses the game, level, branches and difficulty by clicking, typing a chat command or dictating by voice. Commands only fill the Lobby; nothing starts without confirmation.
- **Autopilot:** the application runs a campaign, choosing each next game from a per-target, per-DUT-family knowledge base (ladder order, then retests, then a UCB bandit over games). The engineer can skip, pin, take control or stop.

Every ladder branch and stress move traces to a 3GPP signalling workflow in `5g_signalling_workflows.json` (schema 3.1.0: 17 workflows WF-01 to WF-17, 29 cases, 1,927 PHY-to-SIP steps with SFN/slot timing, 107 RRC messages in full TS 38.331 ASN.1, 92 NAS PDUs decoded per TS 24.501 with TS 24.301 links, 5GS↔EPS timer and interworking tables, and the `networkParameters` catalogue of 37 configurable parameters with their bindings into those messages). Treat that file as the source of truth for message names, sequences, timers, expected UE behaviour and which IE each network parameter controls: embed it as `DATA.workflows` and read from it, never retype it.

**Network model.** The library's values are defaults. An engineer can import an operator's or RAN vendor's parameters (JSON or XML: the canonical Test Arena format, a TS 28.541 NRM-style export or a vendor list) or edit them on `#/network`. Entries are mapped onto `networkParameters` by alias, converted and checked against TS 38.331 / TS 24.501, then applied. Applying writes the values into every bound RRC, NAS and PHY IE (marked "Model"; the hex stays the library's), re-times timer expiries, switches L5 calls to the IMS-VoPS path when VoPS is off, changes the N26 expectation and shifts cohort pressure within ±20%. A running game keeps the model it started with, and every run records its model.

**Field-data lake.** Field data (signalling flows from UE logs and PCAP, RF measurements, fading profiles, channel models and raw I/Q) is stored in a small lakehouse in `field_lake/`, separate from the app. Raw files are listed in `raw/manifest.csv` and never modified. Ingest scripts curate them into typed Parquet tables (`schema.sql`) queried with DuckDB, masking IMSI/SUPI/IMEI on the way in and mapping every RRC/NAS/SIP message onto `DATA.workflows`. `build_bundle.py` exports a `test-arena.field-data` bundle. `embed_bundle.py` writes it into `index.html` as `FIELD_DATA` between the `FIELD_DATA:BEGIN/END` markers. Never edit that block by hand. In the app (`FIELD-DATA LAKE` block of the script), lake sessions become source captures for the games that play their matched workflow, and Settings lists the bundle and can import a newer one. Field replay (S9) is not built (G29).

## Technical stack and constraints

- **One file:** a single `index.html` with all CSS in `<style>` and all JavaScript in `<script>`.
- **No frameworks or libraries:** no React, Vue, jQuery, Tailwind or chart libraries. Use vanilla JS, the DOM, SVG and Canvas.
- **No npm, no build tools:** no `package.json`, bundlers, transpilers or preprocessors. The file opens directly in a browser.
- **No backend:** no fetch to a server, no form submission, no auth, no database. All data is embedded in the file as JS objects (`DATA.games` for stress games, `DATA.ladder` for ladder levels, `DATA.workflows` for the workflow library).
- **External requests:** Google Fonts only, as listed in the design standard.
- **Browser storage:** optional, try/catch-guarded, and used only for convenience (session runs, ladder unlocks, the Voice input setting, the Autopilot knowledge base, the active network model and an imported field-data bundle).
- **The field-data lake is the exception to "one file, no build tools".** It is offline Python tooling (DuckDB, NumPy, SciPy, PyArrow; `requirements.txt`) that runs before the app, never in it. Its data lives in `C:\data\field-lake` (`FIELD_LAKE_ROOT`), outside OneDrive and the repo. Only its exported bundle may ever be embedded in `index.html`.
- **Imported files:** read with `File.text()`, JSON parsed natively, XML with `DOMParser`. They never leave the page and the library is never modified.
- **Voice:** the browser's `SpeechRecognition` only, opt-in from Settings (off by default), push-to-talk only; it is the single allowed exception to the external-request rule. Chat is parsed in the page by a deterministic grammar; no language model and no network call.

## Supporting documents

| Document | What it holds | Read it when |
| --- | --- | --- |
| `site-contract.md` | Screens and routes, every content field, navigation/scroll/hover/responsive behaviour, animations, acceptance criteria | Before building or changing any screen or behaviour. **It is the source of truth for what to build.** |
| `design-standard.md` | Colour, typography, spacing, texture and motion tokens, plus a merge checklist | Before any visual decision. **It overrides the PRD's Design Standard section.** |
| `content-model.md` | Field-level content constraints, per-game KPI and state sets, the field-data bundle fields (§8) and the open gaps (G1–G32) | When writing copy or sample data, or when you hit an undefined value |
| `PRD.md` | Vision, personas, prioritised features, the workflow library mapping, the Procedure Ladder game catalogue, the Game Theory Guide (§4: every game model explained with diagrams), constraints, success criteria | When judging scope or priority, or when you need the branch sets, workflow references and game models for ladder levels. Must-Haves only, unless told otherwise. |
| `5g_signalling_workflows.json` | The signalling workflow library (schema 3.1.0): 17 workflows and their cases, each step with DCI/PHY/MAC detail, RRC `messageRef` into the ASN.1 message library, NAS steps with `nasMessageRef` into the NAS library (IEs, security, TS 24.301 equivalent), SIP headers/SDP, timers, cause and SIP response matrices, pass criteria, the `defaults.verify` list and `networkParameters` (37 parameters: value sets, codecs, timers, aliases, library defaults and IE bindings) | Whenever you build or change a ladder branch, message flow, narration, report or network-model parameter. |
| `field_lake/` | The field-data lake demo: `schema.sql` (curated tables), `lake.py` (paths, masking, library mapping), one `ingest_*.py` per source kind, `run_all.py`, `build_bundle.py`, `make_samples.py` (synthetic data), `queries.sql`, `README.md` | Whenever you add a field-data source, change a curated table or the bundle format, or wire field data into the app |
| `5g_workflow_generator/` | Python generator for the library (`build.py`, needs pycrate). `netparams.py` defines `networkParameters`; `add_netparams.py` adds that section to an existing 3.0 library without a full rebuild | When a parameter or binding must change. Edit `netparams.py`, never the JSON by hand |

**If documents conflict**, use this order: site contract → design standard → content model → PRD.

## Current state

- `index.html` implements **site contract v9**, on top of the v7 and v8 builds described below:
  - **v8 play modes:** Manual chat/voice with interpretation chips, clarification, queue and practice runs; Autopilot campaigns with plan, overrides, countdown and Campaign Report; `#/plan`, `#/campaign/:id`, `DATA.commands`, `CONFIG.autopilot`.
  - **v9 network model:** `#/network` with JSON/XML import, mapping report, parameter editor, impact preview, apply/reset/export and templates; `DATA.sampleNetworkModels`; library 3.1.0 embedded.
  - **Automated Chrome checks pass:** AC-P1–AC-P14, AC-M1–AC-M12 and the v7 regression set (AC-N1/N2/N8, AC-C4/C7/C8/C20, AC-S3/S5/S7, AC-R1 at 320–1920 px).
  - The full cross-browser sweep (§5.1) has still to run.
- The network-model engine lives in the `NETWORK MODEL` block of the script:
  - `nmReadFile` → `nmMap` → `nmCoerce` → `nmStage`, then `nmApplyActive`.
  - `resolveFlow` wraps the library resolver: `nmRewriteRef` (VoPS) → `resolveFlowLib` → `nmApplyFlow` (value tags, timer re-timing, N26).
  - The inspector uses `nmOverlay` with the model recorded on each trace entry (`e.nmh`).
- **Field-data lake (PRD v1.9, M15)** is built in `field_lake/` and runs end to end on the synthetic dataset:
  - `run_all.py --samples`, then `build_bundle.py`.
  - 100% of RRC/NAS/SIP messages are mapped, each session is matched to the right workflow, and the bundle is 59 KB.
  - Not yet tested: the `tshark` PCAP path (needs Wireshark and a real capture).
  - Built and browser-checked (contract v9.2, AC-F1–AC-F7): the embedded bundle, lake sessions as Briefing captures, and the Settings panel with import. Import was tested by passing `File` objects to `fdImport`; the native picker was not driven.
  - Not yet built: field replay (S9) and RF-driven pressure (G29).
  - **Build mismatch.** The `index.html` in this checkout is the **v8** build: play modes, library 3.0.0, no `#/network`. The v9 network-model build described above is not in this folder. The field-data block is self-contained (`FIELD-DATA LAKE` section, the `FIELD_DATA` markers, three small hooks in `generateTestcase`, the Briefing captures list and `viewSettings`), so it ports to the v9 build. The lake maps against the newest library in the repo root (3.1.0), and Settings flags the difference.
- To bind a new parameter: add it to `netparams.py`, run `add_netparams.py` (or `build.py`), re-embed the JSON and check that `occurrences` > 0 for every binding.

- (v7 baseline) `index.html` implemented site contract v7: the embedded library v3 (`DATA.workflows`, checked against schema 3.x with `messageLibrary` and `nasLibrary`), NAS messages as library steps with a full NAS decode, branch sets drawn from WF-01 to WF-17 (L1 19, L2 14, L3 12, L5 15), PHY/MAC steps in the Briefing and trace, an inspector that decodes DCI, PHY channels, MAC PDUs, RRC ASN.1, NAS and SIP, IMSI masking, and saved traces tied to the library version. Automated Chrome checks pass (56 regression, 29 trace); the full cross-browser AC sweep (§5.1) is still to run.
- If `5g_signalling_workflows.json` changes, re-embed it in `index.html` as `WORKFLOW_LIBRARY` (minified, whitespace-only changes) and check that every branch's `wf` reference still resolves: `seq` numbers in `at: { seq }` and stress `steps: [a, b]` slices are tied to this version. Saved traces carry the library version, so moving to 3.1.0 drops traces saved under 3.0.0 (by design).
