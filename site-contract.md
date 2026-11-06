# Test Arena — Site Contract

*v9.2 · 6 Nov 2026 · Derived from `PRD.md` (v1.9) and `content-model.md`. Visuals follow `design-standard.md`; any visual value not stated here comes from that document's tokens.*

> **Change log**
> - **v9.2 (6 Nov 2026):** Connects the build to the field-data lake (PRD M15). The lake's bundle is embedded as `FIELD_DATA` by `field_lake/embed_bundle.py`. Lake sessions become source captures on the Briefing (LAKE tag and matched workflow). Settings gains a Field-data lake panel listing the bundle and its sessions, with an Import bundle button (local file, never uploaded) and a way back to the embedded bundle. New behaviour §3.8 and acceptance criteria AC-F1–AC-F7. G29 is answered for the Briefing and Settings; field replay (S9) stays open.
> - **v9.1 (6 Nov 2026):** Scope note for the field-data lake (PRD M15). The lake and its bundle live outside the static build, in `field_lake/`. The build still reads `DATA.captures`, and `DATA.fieldData` is reserved for the bundle until its first in-app use is decided (G29). One departure row was added, the G4 note was updated, and G29–G32 were listed. No screen, behaviour or acceptance criterion changes; the v9 build stays conformant.
> - **v9 (4 Nov 2026):** Network model (PRD M14): `#/network` route and a fifth nav item; JSON/XML import with mapping report, parameter editor, impact preview, apply / reset / export; the workflow library moves to schema 3.1.0 (`networkParameters`); model values applied to RRC, NAS and PHY IEs with "Model" marks; timer re-timing, the VoPS and N26 procedure switches and bounded simulation effects; runs record their model. Adds `DATA.sampleNetworkModels`, behaviour §3.7, open items G25–G28 and acceptance criteria AC-M1–AC-M12; AC-S7 and AC-C26 updated. v8 play modes are now built.
> - **v8 (4 Nov 2026):** Play modes (PRD M12, M13) for gNB / AI-RAN and modem targets. **Manual:** command bar with chat and push-to-talk voice, interpretation chips, clarification, queue, branch picker and practice runs. **Autopilot:** campaign scope, budget and ceiling, campaign plan, Arena banner with Skip, Pin and Take control, next-game countdown, Campaign Report route, knowledge base and Settings controls. Adds `DATA.commands`, `CONFIG.autopilot` (G23), placeholders G21–G24, behaviour §3.6 and acceptance criteria AC-P1–AC-P14.
> - **v7 (4 Nov 2026):** Workflow library v3 (schema 3.x with `nasLibrary`): NAS messages read as library steps (no derived rows), NAS decode in the inspector and the download, "Carries NAS" on RRC steps, Settings timer and interworking tables, saved traces tied to the library version, new branches from WF-13 to WF-17. AC-S7, AC-C22, AC-C23, AC-C26, AC-C27, AC-C29 and AC-C35 updated; AC-C36 added.
> - **v6 (3 Nov 2026):** Workflow library v2 (schema 2.x): branch `wf` references to cases, matrix rows, variants and composed flows; NAS rows split from their RRC containers; PHY/MAC steps in flows and the trace with a Signalling / All layers toggle and a PHY · MAC filter; inspector decodes DCI, PHY channels, MAC PDUs, RRC ASN.1, NAS and SIP; IMSI masking in hex; latency from library slot timing; compact saved traces. AC-S7, AC-C22–AC-C32 updated; AC-C33–AC-C35 added.
> - **v5 (2 Nov 2026):** Signalling trace (PRD M7): Arena tab with ladder-diagram rows and IE inspector, Game Over trace section, stress moves replaying workflow slices with random access, trace in the downloaded log, and AC-C28–AC-C32.
> - **v4 (2 Nov 2026):** Games driven by the signalling workflow library (PRD M11): `DATA.workflows`, workflow references on every branch, Briefing message flow, Arena step narration, workflow steps in reports, Level 5 (IMS Voice & Video), three cells for L3, Settings workflow panel, and acceptance criteria AC-S7 and AC-C22–AC-C27.
> - **v3 (2 Nov 2026):** Added the Procedure Ladder (PRD M10): Lobby mode toggle and level cards with unlocks, ladder fields on Briefing, Arena and Game Over, `LEVEL CLEAR` outcome, `DATA.ladder`, placeholder G13, and acceptance criteria AC-N8 and AC-C16–AC-C21.
> - **v2 (29 Oct 2026):** Previous release.

## 0. Scope of this contract

This contract defines a **static, client-side build** of Test Arena. It is a playable demo of the MVP experience, delivered as one `index.html` with embedded CSS and JavaScript, as set out in `CLAUDE.md`.

### What "static" means here

- **No backend calls.** The game loop is a JavaScript simulation (the mock adapter from M8). It uses a seeded pseudo-random engine plus data embedded in the file as JS objects:
  - `DATA.captures`
  - `DATA.games` (the four stress games)
  - `DATA.ladder` (the five ladder levels: game model, explainer, branch set, state inputs, reward components and KPI set, as listed in `content-model.md` §3c–§3d and the PRD catalogue). Each branch carries a `wf` reference: `{ id: 'WF-07', case?: 'WF-07a', at?: 'REGISTRATION REJECT' | { seq }, cause? | causes?: ['#9'], matrix?: '480', replace?, append?, matrixFrom?, variant?: 'WF-05c', cut?, retune?: 'early', compose?: [{ id, steps: [a, b] }], steps?: [a, b], injectNote?, label?, expect? }`, or `null` for tester-defined branches. `at` names the injection step; `steps` slices a flow by `seq` (stress moves).
  - `DATA.workflows` (an embedded copy of `5g_signalling_workflows.json`, unchanged apart from whitespace). At boot the script checks `schema === 'test-arena.signalling-workflows'`, a `schemaVersion` with major version 3, a `messageLibrary` and a `nasLibrary` object. From 3.1.0 the library also carries `networkParameters` (the network-model catalogue: parameters, value sets, bindings into the libraries, timers, aliases and `libraryDefault`); without it the network model screen shows "This workflow library has no networkParameters section" and everything else works. Message flows, PHY/MAC detail, ASN.1, NAS encodings, expected behaviour, spec baseline and the verification list are read from it, never retyped. Steps are indexed by `<flow id>#<seq>` so saved traces store only that key and decode again on display; each saved run records `traceLib` (the library version), and a trace recorded against another version is dropped on load rather than decoded against the wrong steps.
  - `DATA.difficulty`
  - `FIELD_DATA` (the field-data lake bundle, schema `test-arena.field-data` 0.x, see content model §8). It is written between the `/* FIELD_DATA:BEGIN */` and `/* FIELD_DATA:END */` markers by `field_lake/embed_bundle.py`, never by hand. At boot it is checked (schema name, major version 0, a `sessions` array); an invalid or missing bundle leaves the build on `DATA.captures` alone, with the reason shown in Settings. The active bundle (embedded or imported) is held as the field-data state; each session becomes a capture `{ id, location: area, date: start (date part), profile: notes or band · DUT, wf, wfScore, features, lake: true }`. `features` lists every stress game and ladder level with a move whose `wf` (or `compose`) names the session's workflow.
  - `DATA.sampleNetworkModels` (two fictional files: "Operator A · dense urban n78" in the canonical JSON format and "Vendor B · site 0471 export" as NRM-style XML with vendor-style names, one invalid enumeration and one unencodable timer)
  - `DATA.sampleRuns` (includes at least one ladder run of each outcome, plus one sample Autopilot campaign of 4–6 runs with its decisions)
  - `DATA.commands` (the chat/voice vocabulary, **built at boot** from `DATA.games`, `DATA.ladder` and `DATA.workflows`, never retyped): game and level names and numbers, branch names, 5GMM causes, SIP codes, workflow and case IDs, difficulties, targets, and verbs (play, run, start, only, then, replay, stop, skip, pin, take control, show).
  - `CONFIG.autopilot` (G23): `{ c: 1.0, failBonus: 1.0, branchFailBonus: 0.5, planLength: 5, countdownSec: 10, defaultBudget: { time: '08:00', failures: 3 }, defaultCeiling: 'hard', priors: { gnb: [...], modem: [...] } }`. Each value is marked `// PLACEHOLDER (G23)`.
- **No form submission.** Every control changes client-side state only.
- **No auth, no database.**
  - Run History shows `DATA.sampleRuns` plus runs from the current session.
  - Session runs and ladder unlocks (keyed by exact DUT label) are held in memory.
  - `localStorage` is optional, try/catch guarded, and used only so a refresh keeps session runs, unlocks, the Voice input setting, the Autopilot knowledge base and the active network model (its changed parameters and name; never the imported file). An imported field-data bundle is kept under its own key (`test-arena.fieldData`) when it fits; if storage refuses it, it lasts until refresh and Settings says so.
  - The **Autopilot knowledge base** is held in memory under the key `<target>|<DUT family>`. The DUT family is the DUT label without its trailing build/version token; for example, "gNB-X sw 4.2.1" → family "gNB-X". Each candidate game (stress game × difficulty, ladder level, or named branch subset) stores `{ plays, meanBreak, lastBuild, failedOn: [builds], qByMove: {...} }`. The knowledge base is seeded from `DATA.sampleRuns` at boot, then updated after every run in either play mode.
- **Deterministic.** The same seed + game (or level) + difficulty always produces the same move or branch sequence, failure or clear point, and final score. An Autopilot campaign records a campaign seed and every decision, so the same knowledge base + campaign seed produces the same plan.
- **Chat and voice in the static build.** Chat commands are parsed in the page by a deterministic grammar over `DATA.commands`; no text leaves the page. Voice uses the browser's `SpeechRecognition` (or `webkitSpeechRecognition`) **only when the engineer turns on Voice input in Settings** (off by default). The mic is hidden when the API is missing, and Settings notes that the browser may process audio in its vendor's cloud (G21). This is the one allowed exception to "no external requests", and it is opt-in.
- **Placeholders.** Every undefined value (gaps G1–G4 and G13) lives in one `CONFIG` object at the top of the script, with each entry commented `// PLACEHOLDER (Gn)` (G1–G4, G13, and G23 for Autopilot).
- **Debug hook.** Loading with `?debug=1` shows a small overlay with fps, tick count and elapsed time, and exposes `window.__arena` (read-only state snapshot, including ladder unlocks and branch coverage) so acceptance tests can be automated. It is hidden without the query string.

### Departures from the PRD (by design)

| PRD item | In this static build |
| --- | --- |
| Live Multi-UE simulator, WebSocket at 10–20 Hz | JS simulation ticking at 10 Hz (100 ms) |
| Datalake queries | `DATA.captures`; "generation" picks and seeds from it |
| Field-data lake (M15) queried by the testcase generator | The lake runs outside the page (`field_lake/`, Python + DuckDB). Its exported bundle is embedded as `FIELD_DATA` or imported from Settings; there is no live query. Field replay (S9) is not built (G29) |
| Instrument Settings (host, interface, test connection) | Read-only; adapter locked to `SIMULATED` |
| Log capture download (.zip) | Client-generated `.json` via `Blob` |
| 24 h unattended run, ≥ 10 fps at 60K UEs | 60 min unattended run, ≥ 30 fps (see §5.7) |
| Desktop only (1920×1080 min) | Desktop primary; tablet and phone layouts defined in §3.4 |
| Ladder clear minimum run time (G13) | `CONFIG` placeholder; short defaults so a level can be cleared in a demo |
| Workflow library loaded from a file and replayed through the adapter | Embedded as `DATA.workflows`; the simulation narrates the steps and judges branches by its own model |
| Cohort fan-out of workflows (G20) | One representative UE's steps are narrated; KPIs come from the cohort model |
| Three emulated cells for Level 3 | Mock emulates Cell 1, Cell 2 (TAC 000001) and Cell 3 (TAC 000002) |
| On-premise speech-to-text (G21) | Browser `SpeechRecognition`, opt-in, off by default; hidden when unsupported |
| Chat understanding (G22) | Deterministic in-page grammar over `DATA.commands`; no language model |
| Autopilot knowledge on the Test Arena host | In memory + optional `localStorage`, seeded from sample runs |
| DUT recovery after a failure (G24) | Mock recovers instantly; the countdown shows "Recovering DUT…" for 2 s |
| Network model applied through the adapter, hex re-encoded (G27) | Model values shown in the decoded IE trees and logs; UPER and NAS hex stay the library encoding and are flagged |
| Parameter effect on the DUT (G28) | Bounded load factors per KPI (±20%) from A3, RACH, RRC and NAS timer values |

---

## 1. Page Structure

The site is a single-page app with hash routes. The shell is always present.

| # | Route | Section | Purpose |
| --- | --- | --- | --- |
| 0 | all | **App shell** | Keeps the product, current game and simulator state visible from every screen. |
| 1 | `#/lobby` (default) | **Game Lobby** | Lets the engineer choose the play mode, target and DUT label, then either the game (by click, chat or voice) or an Autopilot campaign's scope and budget. |
| 2b | `#/plan` | **Campaign plan** (Autopilot) | Shows the next five games Autopilot will play, each with its reason, source and break score, so the engineer can edit or approve before the campaign starts. |
| 2 | `#/briefing` | **Testcase Briefing** | Shows the generated testcase, its game model and seed captures (and, for a ladder level, its branch set, each branch's workflow message flow and the clear rule) so the engineer knows exactly what will go on the air before pressing Play. |
| 3 | `#/arena` | **Live Arena** | Plays the leader–follower game in real time, with the cohort, agent loop and diagnostics on one screen. |
| 3a | | Arena header | Identifies the game and DUT, shows elapsed time, state and (ladder) branch coverage, and offers Stop. |
| 3b | | Cohort arena | Visualises the UE cohort attacking the DUT and the DUT's current response. |
| 3c | | Agent loop panel | Shows the State → Reward → Action cycle and the running score. |
| 3d | | Diagnostics strip | Shows the game's five live KPIs against their failure thresholds, plus the event log. |
| 4 | `#/gameover/:runId` | **Game Over** | Reports how and why the game ended (failure, stop or level clear), with the evidence needed to replay it. |
| 4b | `#/campaign/:id` | **Campaign Report** | Summarises an Autopilot campaign: failures found, coverage before and after, every decision with its reason, and what the knowledge base learned. |
| 5 | `#/history` | **Run History** | Lists every saved game and opens any run's report or score timeline. |
| 6 | `#/settings` | **Instrument Settings** | Shows the simulated adapter configuration, simulator limits, the ladder unlock mode and the loaded workflow library. |
| 7 | `#/network` | **Network model** | Imports an operator's or RAN vendor's parameters (JSON or XML), shows how each entry was mapped and checked, lets the engineer edit values, previews the signalling impact and applies the model to every flow. |

---

## 2. Content Inventory

### Global rules (apply to every field)

- **Type roles** follow `design-standard.md` §2:
  - **Display** (Chakra Petch, uppercase, tracked, ≤ 4 words) for screen titles, panel titles, scores, game IDs and state banners only.
  - **UI** (Inter) for all other interface text, in sentence case. Tone is plain and exact, with no exclamation marks.
  - **Mono** (JetBrains Mono, tabular numerals) for every live number, ID, seed, timestamp and log line.
- **Japanese sub-labels** come only from the fixed vocabulary in `design-standard.md` §2.4. There is at most one per component, always paired with English, and always `aria-hidden="true"`.
- **Numbers** always carry a unit and use the fixed decimals stated below.
- **Time** is shown as `HH:MM:SS` in the arena and `DD MMM YYYY HH:MM` in history, in local time.
- **Subscriber IDs** in sample data are masked to the last 4 digits.
- **Truncation.** Text over its limit is cut with an ellipsis, and the full value goes in a `title` attribute.
- **Game names.** Ladder levels show as "Level n · Procedure" in headers and "Ln Procedure" in tables and badges (L5 is "L5 IMS Call" in tables).
- **Workflow references and message names** follow `content-model.md` shared rules: mono, exact library spelling, direction glyph UL ↑ / DL ↓ / SS ⚑ / UE ●.

### 0. App shell

| Field | Description | Constraints |
| --- | --- | --- |
| Wordmark | "Test Arena" + Anritsu logo | Fixed text |
| Nav items | Lobby · Arena · History · Network · Settings, each with a JP sub-label | 1 word each. Arena is `aria-disabled` when no game is running. Briefing and Plan mark Lobby as current; Campaign Report marks History. |
| Adapter badge | Simulator state | Always `SIMULATED`; `--accent-warn` |
| Active game indicator | Game ID + elapsed time + running status dot | Format `PLAY 042 · 01:23:45`. Hidden when no game runs; links to `#/arena`. |

### 1. Game Lobby

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "SELECT GAME" + JP sub-label ロビー + blade line | Display `h1` |
| Play mode | Manual or Autopilot | Two-option segmented control above everything else; default Manual; remembered for the session. Autopilot hides the game type, cards, difficulty and command bar and shows the campaign fields. Target and DUT label are kept. |
| Game type (Manual) | Stress games or Procedure Ladder | Two-option segmented control (formerly "Mode"); default Stress games. Switching clears the game/level selection but keeps target, DUT label and difficulty. |
| Command bar (Manual) | Type or say what to play | Full-width input under the play mode, `aria-label="Command"`, ≤ 200 chars. Enter submits. Mic icon button at the right end (push-to-talk: hold Space while focused, or press once to start and once to stop); hidden when Voice input is off or unsupported. Behaviour in §3.6. |
| Interpretation chips (Manual) | How the last command was understood | Chip row under the bar: Target · DUT · Game or level · Branches · Difficulty. Each chip is a button that focuses the matching control. Slots the command didn't set read "(unchanged)". The row is announced politely: "Understood: …". |
| Clarification (Manual) | Ambiguous command | Question (≤ 80 chars) + 2–4 choice chips in place of the interpretation row; nothing changes until a choice is made; `Esc` dismisses it |
| Queue (Manual) | Games dictated in order | Up to 5 chips, e.g. "L2 · Normal → L3 · Normal"; each removable; GENERATE starts the first; the next is offered on Game Over |
| Target type | gNB / AI-RAN or Modem | Two-option segmented control; required; default gNB / AI-RAN |
| DUT label | Device name + build | Text input, required, 1–40 chars, trimmed. Changing it re-evaluates ladder locks immediately. |
| Game cards (×4, Stress mode) | Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze | Single select; content from `DATA.games` |
| Level cards (×5, Ladder mode) | L1 Registration, L2 Data Call, L3 Handover & Mobility, L4 Throughput, L5 IMS Voice & Video | Ordered left to right (snake on tablet, top to bottom on mobile), joined by a connector line; single select; content from `DATA.ladder` |
| Card · title | Game or level name | ≤ 20 chars, `h3`. Level cards prefix a level number badge (1–4). |
| Card · description | What the game does to the DUT | ≤ 90 chars; starts with a verb |
| Card · KPI tags | KPIs this game pressures | 2–3 tags from that game's own KPI set |
| Level card · game model | e.g. "Sequential game" | ≤ 30 chars, `caption` style, `--accent-fuji` |
| Level card · workflows | Workflow IDs the level plays | Mono tags, `--text-2`; 1–4 IDs |
| Level card · lock state | `LOCKED` / `OPEN` / `CLEARED` | Badge + icon (lock / none / check). Locked cards are `aria-disabled`, not selectable, and show "Clear L<n−1> to unlock". L1 is always `OPEN` or `CLEARED`. |
| Difficulty | Easy / Normal / Hard / Brutal | Segmented control; default Normal |
| Difficulty hint | What the level means in numbers | ≤ 60 chars, from `DATA.difficulty` (ladder hints include branch count) |
| Validation message | Shown under DUT label if empty on Generate | Exactly: "Enter a DUT label to continue." |
| Primary button | "GENERATE" (Manual) / "PLAN CAMPAIGN" (Autopilot) | Hero button. Manual: disabled until target, DUT label and an open game or level are set. Autopilot: disabled until target, DUT label and a budget are set. |
| Campaign scope (Autopilot) | What Autopilot may choose | Segmented: Everything (default) / Ladder only / Stress only / Custom; Custom reveals multi-select chips for the 5 levels and 4 stress games |
| Campaign budget (Autopilot) | Time and/or failures | Time input `HH:MM` (00:05–72:00, default from `CONFIG.autopilot`) and failures stepper (1–20, default 3). At least one must be set. |
| Difficulty ceiling (Autopilot) | Highest difficulty allowed | Segmented Easy–Brutal; default Hard |
| Knowledge summary (Autopilot) | What is known about this DUT | ≤ 100 chars from the knowledge base, updates as the DUT label changes; cold start reads "No history for this DUT: starting from the <target> defaults" |

### 2. Testcase Briefing

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "BRIEFING" + JP 概要 | Display `h1` |
| Testcase ID | Unique ID for replay | Format `TC-YYYYMMDD-NNNN`; copyable |
| Summary | One-line description of the sequence | ≤ 120 chars |
| Game model | Model name | ≤ 40 chars; `--accent-fuji` label; e.g. "Sequential game · subgame perfection" |
| Model explainer | What the model means for this test | ≤ 140 chars, `body`, `--text-2` |
| Branch set (Ladder) | Branches to cover | Ordered list, 4–18 items, count per difficulty from `CONFIG`; each ≤ 40 chars plus its workflow reference; rows are toggle buttons (`aria-pressed`) that select which message flow is shown; baseline selected by default |
| Message flow (Ladder) | Selected branch's workflow steps | Panel titled "MESSAGE FLOW" + JP 手順, with the flow label, case/variant name and "Composed"/"Adapted" tags. Segmented toggle "Signalling (n)" (default) / "All layers (n)" (remembered for the session), with a count caption. Table: seq · dir glyph · layer · message (+ container, note) · timers. Each NAS message is its own library step: an uplink NAS step comes just before its RRC container, a downlink one just after; 5GSM steps follow their NAS TRANSPORT step. Composed flows show a "from WF-xx" rule at each join. The injection step is tagged "Injection" (`--accent-warn`). Steps whose IE values or time come from the network model are tagged "Model" with a note naming the parameters or the re-timing; the panel header shows "Model" when any step is. Tester-defined branches show "No library workflow (tester-defined)". |
| Expected behaviour (Ladder) | What passes the selected branch | ≤ 200 chars, derived from the library as in `content-model.md` §3 |
| Spec baseline | Library spec references | ≤ 100 chars, `caption` |
| Defaults to verify | `defaults.verify` areas | `--accent-warn` badge "n defaults to verify"; tooltip lists up to 3 areas |
| Clear rule (Ladder) | What clears the level | ≤ 80 chars; built from branch count + `CONFIG` minimum time (G13) |
| Cohort mix | UE counts per slice + stacked bar | URLLC / eMBB / mMTC integers; total ≤ 60,000 |
| Source captures | Captures that seeded the testcase | 1–5 items: capture ID · location · date. Lake sessions carry a `LAKE` tag and their matched workflow (`WF-01`, mono). The caption counts both sources: "n from the field-data lake · m from the embedded sample" (a zero part is omitted) |
| Stop conditions | What ends the game | "User Stop" + the game's failure rules from `CONFIG`; ladder adds "Level clear" |
| Generation time | Simulated generation duration | Seconds, 2 decimals, always < 2.00 |
| Seed | Deterministic seed | 8-char hex; copyable |
| Network model | Active model | Name + "n parameters" + fingerprint and a "Model" tag, or "Library defaults"; "Edit" link to `#/network` |
| Branch picker (Manual, Ladder) | Choose a subset of branches | Checkbox per branch row; baseline checked and disabled; all checked by default. Unchecking any shows the caption "Practice run: won't clear the level", and the clear rule reads "Practice run". |
| Buttons | "PLAY" (hero), "Regenerate", "Back" | Regenerate draws a new seed |

### 2b. Campaign plan (Autopilot)

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "CAMPAIGN PLAN" + JP 作戦 | Display `h1` |
| Campaign ID | Unique reference | `CMP-YYYYMMDD-NNN`; copyable |
| Settings line | Target · DUT · scope · budget · ceiling | ≤ 100 chars |
| Planned games | Next `CONFIG.autopilot.planLength` picks | Ordered list. Per row: game or level (+ subset), difficulty, reason (≤ 80 chars), source chip (`Ladder order` / `Retest` / `Best record` / `Explore` / `Escalate`), break score (2 dp or "new"). Row buttons: Remove, Up, Down, Pin. |
| Method note | How picks are made | Exactly: "Ladder order and retests first, then best break score with an exploration bonus (UCB)." |
| Buttons | "START CAMPAIGN" (hero), "Switch to Manual", "Back" | Switch to Manual opens the Lobby in Manual with the first planned game pre-selected |

### 3a. Arena header

| Field | Description | Constraints |
| --- | --- | --- |
| Game ID | e.g. "PLAY 042" | Display. Zero-padded 3+ digits, increments per session. |
| Game + difficulty | e.g. "Signalling Storm · Hard" or "Level 1 · Registration · Hard" | Read-only |
| DUT label | From Lobby | ≤ 40 chars |
| Elapsed time | Time since Play | `HH:MM:SS`, counts up |
| Branch coverage (Ladder) | Branches passed | `n / N` + %, mono; thin bar in `--accent-hisui`; hidden for stress games |
| Workflow step (Ladder) | Position in the current workflow | `WF-11 · step 4 / 10`, mono, `--text-2`; hidden on mobile |
| Game state badge | `RUNNING` / `ESCALATING` / `STOPPED` / `DUT DOWN` / `CLEARED` | Colours: hisui / primary / text-2 / shu / hisui (with check icon) |
| Stop button | "STOP" | Opens a confirm dialog: "Stop game and save run?" with "Stop" / "Keep playing". In Autopilot the dialog offers "Stop this game", "Stop campaign" and "Keep playing". |
| Play mode badge | Who is choosing | `MANUAL` (neutral tag) or `AUTOPILOT · n / plan` (`--accent-fuji`) |
| Why this game (Autopilot) | Reason for the current pick | ≤ 80 chars, caption, under the game name |
| Campaign controls (Autopilot) | Skip · Pin · Take control | Secondary buttons. Skip ends this game as `STOPPED` (reason "Skipped by engineer") and moves on. Pin re-queues it after this one. Take control switches the campaign to Manual and keeps the current game running. |
| Budget left (Autopilot) | Remaining budget | Mono, e.g. "05:12:40 · 2 of 3 failures left" |
| Command bar (Arena) | In-game commands | Collapsed to an icon button in the header; opens a one-line bar. Accepts stop, skip, pin, take control, show the trace, switch to X next. |

### 3b. Cohort arena

| Field | Description | Constraints |
| --- | --- | --- |
| Cohort tiles | Aggregated UE groups | ≤ 48 tiles. Shapes per `design-standard.md` §1.5: URLLC diamond, eMBB rounded rectangle, mMTC square. |
| Slice counts | Active UEs per slice | Integers, mono |
| Current move | Leader's current stress move or ladder branch | ≤ 30 chars; ladder adds the workflow reference beneath in mono |
| DUT sprite + status | gNB tower or modem, by target type | `Serving` / `Barring` / `Rejecting` / `Down` |
| Cells (L3 only) | Three cell sprites, "Cell 1", "Cell 2" and "Cell 3" | Cell 3 is drawn with a dashed TA boundary between it and Cells 1–2; cohort tiles drift to the serving cell on handover or reselection; serving cell outlined in `--line-2` |
| Legend | URLLC · eMBB · mMTC · NETWORK | Colour + shape + label |

### 3c. Agent loop panel

| Field | Description | Constraints |
| --- | --- | --- |
| Panel title | "AGENT LOOP" + JP 戦略 | Display `h2`; `--accent-fuji` accents |
| State values | The game's three state inputs | Stress: RACH attempts, PRB utilisation, rejects (`/s`, `%`, `/s`). Ladder: per level from `DATA.ladder` |
| Reward | Current reward | `display-l`, signed, 2 decimals, `--accent-kin` |
| Reward breakdown | The game's three reward components | Each signed, 2 decimals |
| Action | Agent's last move or branch | ≤ 30 chars |
| Signalling trace tab | Protocol-analyser view of the run so far | Tab beside "Agent loop" (`role="tablist"`, ←/→ switch). Contents per `content-model.md` §3c-bis. Rows are buttons (`aria-pressed`); ↑/↓ moves the selection; Live follows the newest row and stops when a row is picked. The event log has an "Open signalling trace" link. |
| Agent label | Algorithm + game model | "DQN v0.3 · " + model short name (Stackelberg, Sequential, Coordination, Congestion, Bargaining) |
| Total score | Cumulative score | `display-l`, integer, `--accent-kin` |

### 3d. Diagnostics strip

| Field | Description | Constraints |
| --- | --- | --- |
| Panel title | "DIAGNOSTICS" + JP 診断 | Display `h2` |
| KPI cells (×5) | The game's five KPIs | Stress games: RACH success (`%`, 1 dp), PRB utilisation (`%`, 1 dp), Rejects (`/s`, int), Latency p95 (`ms`, 1 dp), Drops (`/s`, int). Ladder levels: the five per level in `content-model.md` §3d. Each with a 60 s sparkline. |
| Threshold line | Failure threshold per KPI | From `CONFIG` (G1); turns `--accent-shu` when crossed |
| Event log | Last 20 events | `HH:MM:SS` + ≤ 80-char message, newest on top. Ladder logs "Branch passed: <name>" when a branch is covered, and narrates the representative UE's workflow steps: "↑ UL 5GMM REGISTRATION REQUEST · T3510 ▶". |

### 4. Game Over

| Field | Description | Constraints |
| --- | --- | --- |
| Banner | Outcome | "DUT DOWN" (shu, with hanko seal 故障), "GAME STOPPED" (hisui, no seal) or "LEVEL CLEAR" (hisui, with clear mark 突破); `display-xl` |
| Failure reason | What broke | ≤ 100 chars and names the KPI or event; on a ladder level it leads with the workflow step ("WF-11 step 4: …"). Shows "Stopped by user" when stopped and "All n branches passed" on a clear. |
| Time to failure | Elapsed at end | `HH:MM:SS` |
| Final score | Total score | `display-xl`, `--accent-kin` |
| Branch results (Ladder) | Per-branch result | Table: branch · workflow · Passed / Failed / Not reached · time first passed. Failed rows in shu with an icon and a second line: step reached + expected behaviour. |
| Unlock notice (Ladder clear) | Newly opened level | ≤ 60 chars, e.g. "Level 2 · Data Call unlocked for gNB-X sw 4.2.1". Not shown after L5. |
| Signalling trace | Captured trace of the run | Full-width section between the hero and the KPI snapshot: rows on the left (7 columns), inspector on the right (5 columns); one column below tablet; failing exchange preselected |
| Triggering moves | Last 5 moves | Move + timestamp, oldest first |
| KPI snapshot | The game's 5 KPIs at end vs threshold | Breached rows marked in shu with an icon |
| Log download | "Download logs" | Generates `run-<id>.json` client-side (ladder runs include branch results with workflow references, the step reached, and the library `schemaVersion`; every run includes `signallingTrace` with each entry's `librarySource`, SFN/slot time and `decode` (DCI, PHY, MAC PDU, NAS, SIP), plus `messageLibrary` and `nasLibrary` objects holding every RRC and NAS message referenced). IMSI/SUPI is masked to the last 4 digits throughout the file. |
| Replay seed | Seed + agent version | Copyable (Autopilot runs add the campaign ID and decision number). Also shows the run's network model (name + fingerprint, or "Library defaults") and "Replay uses the active model" when it differs. |
| Next game (Autopilot) | What plays next | Panel above the buttons: "Next: <game> · <reason>" with a `CONFIG.autopilot.countdownSec` countdown ring. "Pause campaign" stops it, and "Play now" skips the wait. After DUT DOWN the panel first shows "Recovering DUT…" (G24). When the budget is reached, routes to `#/campaign/:id` instead. |
| Next in queue (Manual) | Next dictated game | "Next in queue: <game>" with a "Generate" button; no countdown |
| Buttons | Depends on outcome | Default: "REPLAY" (hero), "New game", "View in History". After L1–L4 clear: "NEXT LEVEL" (hero), "Replay", "View in History". After L5 clear: "NEW GAME" (hero), "Replay", "View in History". |

### 4b. Campaign Report

| Field | Description | Constraints |
| --- | --- | --- |
| Banner | Outcome | "CAMPAIGN COMPLETE" (budget reached; hisui) or "CAMPAIGN STOPPED" |
| Summary | Totals | Games played · run time · DUT failures · levels cleared · branches newly covered |
| Failures found | Ranked by severity (DUT DOWN before failed branches, then by time to failure) | Table: run ID · game · failing KPI or workflow step · time; rows link to `#/gameover/:runId` |
| Coverage | Before → after for this DUT build | Per level: `n / N → n / N` |
| Decisions | Every pick | Table: # · game · source · reason · break score at the time |
| Knowledge update | What changed | ≤ 120 chars |
| Buttons | "Run again" (hero), "Download report", "View runs in History" | Download generates `campaign-<id>.json` client-side, including every decision and run ID |

### 5. Run History

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "HISTORY" + JP 履歴 | Display `h1` |
| Table columns | Date · Game ID · DUT · Target · Mode · Game · Difficulty · Duration · Outcome · Score | Newest first. Ladder rows show Game as "L1 Registration". |
| Outcome badge | How the game ended | `DUT DOWN` / `STOPPED` / `LEVEL CLEAR` / `PLATFORM ERROR` (sample data only) |
| Filters | Game (stress games + five ladder levels), Outcome, Target, Play mode, text search on DUT | Client-side. Autopilot runs are grouped under a campaign row (CMP ID · runs · failures) that expands; the campaign row links to `#/campaign/:id`. |
| Row expand · score timeline | Score over time | Inline SVG line; x = elapsed, y = score |
| Row action | "Open report" | Goes to `#/gameover/:runId` |
| Empty state | Shown when filters match nothing | Exactly: "No runs match these filters." |

### 6. Instrument Settings

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "SETTINGS" + JP 設定 | Display `h1` |
| Adapter mode | Real or mock | Shows `Mock (simulated)`; real option disabled with note "Requires lab deployment" |
| Host address | Simulator address | Read-only `—` |
| Interface | Ethernet/SCPI, GPIB, USB | Read-only list |
| Max UEs | Simulator limit | `60,000`, read-only |
| Emulated cells | Cells for Level 3 | `3`, read-only |
| Workflow library | Loaded library | Panel titled "Workflow library": schema + version (3.1.0), the active network model with an "Open" link, technology, generated date, contents line (workflows · cases · steps · RRC messages · NAS PDUs), carrier, spec baseline, "10 defaults to verify" warning badge, 17 workflows (ID · name · case count · category), expandable "NAS timers · 5GS ↔ EPS" table, "TS 24.301 interworking" list with the GUTI mapping, and "Not covered yet" |
| Datalake source | Capture source | "Field-data lake (n sessions) + embedded sample captures (N)"; "Embedded sample captures (N)" when no bundle is active |
| Field-data lake | Panel: the active bundle | Title "Field-data lake" + JP データ. Key-values: Bundle (`test-arena.field-data 0.1.0`), Source ("Embedded in this build" or "Imported · <file name>", plus "kept in this browser" / "until refresh"), Generated (`DD MMM YYYY HH:MM`), Mapped against (library version, with a warning badge "This app runs library x.y.z" when it differs), Contents ("n sessions · n events · n RF points · n I/Q summaries · n fading profiles"). Sessions table: Session (mono) · Area · Start · Workflow (wftag + score %) · Events (n, "n% of RRC/NAS/SIP mapped": the share of RRC, 5GMM, 5GSM, EMM and SIP events with a library name; PHY/MAC are not counted) · RF points · Seeds (game and level short names, ≤ 4 then "+n"). Actions: "Import bundle…" (secondary; `.json`, ≤ 5 MB), "Use embedded bundle" (ghost; only when an import is active). Error line ≤ 140 chars. No bundle: "No field-data bundle in this build. Run field_lake/embed_bundle.py or import one." |
| Ladder unlocks | Progressive or All unlocked (demo) | Segmented control; default Progressive. "All unlocked (demo)" opens every level for every DUT label without recording clears. |
| Voice input | Show the mic in command bars | Toggle, off by default. Note: "In this demo, speech is recognised by your browser and may be processed by its vendor's cloud. Lab deployments use an on-premise engine." When unsupported, disabled with "Not supported in this browser". |
| Autopilot knowledge | What Autopilot has learned | Table: target · DUT family · runs · failures · top 3 games (break score) · updated. "Reset" per row with confirm "Forget everything Autopilot learned about <family>?" |
| Autopilot defaults | Scope, budget, ceiling | Same controls as the Lobby; used to pre-fill new campaigns |
| Reset demo | "Clear session runs" | Clears in-memory and `localStorage` runs and ladder unlocks after a confirm |

### 7. Network model

Fields, formats and limits are in `content-model.md` §7. This contract adds:

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "NETWORK MODEL" + JP ネットワーク + blade line | Display `h1` |
| Layout | Active model and Import (left, 5 columns) · Signalling impact (right, 7 columns) · Import report (full width, when staged) · actions row · Parameters table (full width) | One column below 1280 px |
| Drop zone | `<input type="file">` (visually hidden, focusable) followed by its `<label>` drop zone | Enter/Space opens the file dialog; drag and drop onto the zone; the focus ring draws on the zone |
| Hero button | "APPLY TO SIGNALLING FLOW" | `aria-disabled` until a model is staged; the only solid primary button on the screen |
| Parameters table | One row per catalogue parameter, grouped | Select for enumerations and booleans, text input otherwise; validated on `change`; an invalid value keeps its previous staged value and shows the error under the control (`aria-invalid`) |
| Report table | One row per entry read | First 60 rows, then "Show all n" |
| Reset confirm | "Reset to library defaults?" | "Cancel" / "Reset"; saved runs keep the model they used |

---

## 3. Behaviour Spec

### 3.1 Navigation

- **Routing.** Hash-based (`#/route`), handled by one `hashchange` listener, with no page reloads.
  - An unknown hash redirects to `#/lobby`.
  - Back and forward follow history normally.
- **Flow.**
  - **Manual:** Lobby → (click, chat or voice) → GENERATE → Briefing → PLAY → Arena → (Stop, DUT failure or Level clear) → Game Over → (next in queue).
  - **Autopilot:** Lobby → PLAN CAMPAIGN → Campaign plan → START CAMPAIGN → Arena → Game Over (countdown) → Arena → … → Campaign Report when the budget is reached or the campaign is stopped.
- **Briefing branch selection.** Clicking a branch row (or Enter/Space on it) selects it and swaps the message-flow panel without a route change; focus stays on the row.
- **Ladder flow.** On a level clear, Game Over offers NEXT LEVEL. It generates the next level's testcase with the same target, DUT label and difficulty (new seed) and routes straight to Briefing.
- **Unlocks.** A clear in Progressive mode marks the level `CLEARED` and the next level `OPEN` for that exact DUT label. A DUT failure or Stop changes no lock state.
- **Guards.**
  - `#/briefing` without a generated testcase redirects to `#/lobby`.
  - A testcase for a level that is locked for its DUT label can't be generated; if one is reached through history state, `#/briefing` redirects to `#/lobby`.
  - `#/arena` with no running game shows "No game running" and a "Go to Lobby" link.
  - `#/plan` without a planned campaign redirects to `#/lobby`; an unknown campaign `:id` redirects to `#/history`.
  - An unknown `:runId` redirects to `#/history`.
- **Leaving a running game.** Navigating away does **not** stop the game; it keeps ticking. Only STOP, a DUT failure or a level clear ends it.
- **Active nav item.** Gets `aria-current="page"` and the active state from `design-standard.md` §5.4.
- **Keyboard.**
  - Every control is reachable by Tab in visual order.
  - `Enter`/`Space` activate.
  - `Esc` closes dialogs and the mobile nav.
  - `S` opens the Stop dialog in the Arena.
  - `/` focuses the command bar (Lobby and Arena).
  - With the command bar focused and Voice input on, holding `Space` on an empty bar records; releasing it stops recording.
- **Focus.**
  - On a route change, focus moves to the screen's `h1` (`tabindex="-1"`).
  - Dialogs trap focus and return it to the trigger on close.
  - The focus ring follows `design-standard.md` §5.4.
  - Locked level cards stay focusable (so their tooltip can be read) but cannot be selected.

### 3.2 Scroll

- **Shell.** `position: sticky; top: 0`, 56 px high. `scroll-padding-top: 72px` keeps focused elements clear of it.
- **Lobby, Briefing, Game Over, History, Settings.** These scroll normally. Each route change scrolls to the top instantly.
- **Live Arena (desktop).**
  - Fixed height of `100vh − shell`, with no page scroll. Only the event log scrolls.
  - The log auto-sticks to the newest entry unless the user scrolls up. A "Jump to latest" chip then restores auto-stick.
- **History table.** The header row is sticky within its container.
- **Smooth scroll.** Used only for "Jump to latest", and disabled under reduced motion.

### 3.3 Hover, focus and active states by element type

- Hover, press, selected, disabled and focus visuals for buttons, STOP, game cards, level cards, nav items, table rows, segmented controls, inputs and icon buttons are defined in **`design-standard.md` §5.4**. Implement them exactly.
- This contract adds:

| Element | Behaviour |
| --- | --- |
| Cohort tile | Tooltip: slice · UE count · current move |
| KPI cell | Tooltip: current value · threshold · 60 s min/max |
| Copy control (IDs, seeds) | Tooltip "Copy"; after click the label reads "Copied" for 1.5 s; copies to clipboard |
| Disabled Arena nav | Tooltip "No game running" |
| Locked level card | Tooltip "Clear L<n−1> to unlock"; no lift on hover |
| Branch list item (Briefing) | Tooltip: what the DUT must do to pass the branch (≤ 60 chars) |
| All tooltips | 300 ms hover delay, immediate on focus, ≤ 60 chars, flip if clipped |

- Hover effects apply only under `@media (hover: hover)`. Touch devices get press states only.

### 3.4 Responsive layout

Breakpoints: **≥ 1280 px** desktop (primary), **768–1279 px** tablet, **< 768 px** mobile. The grid, gutters and type step-downs follow `design-standard.md` §2.3 and §3.4.

| Section | Tablet (768–1279) | Mobile (< 768) |
| --- | --- | --- |
| Shell | Nav inline; active game indicator shows Game ID only | Nav collapses to a hamburger (`aria-expanded`) opening a glass drawer; adapter badge becomes a dot + `SIM` |
| Lobby | Game cards 2 × 2. Level cards in a two-column snake (L1 L2 / L4 L3 / L5) with the connector running L1→L2, L2→L3 (down), L3→L4, L4→L5 (down). Desktop shows the five level cards in one row. | Cards in 1 column; level connector runs vertically; GENERATE sticky at bottom |
| Briefing | Single column; message flow full width under the branch set | Single column; captures collapse to 2 + "Show all"; branch set collapses to 4 + "Show all"; message flow becomes a list (dir glyph + message, timers beneath) and collapses to 8 steps + "Show all" |
| Arena | Cohort full width on top; Agent panel and Diagnostics side by side below; page scroll allowed | Stack: header (branch coverage shows % only) → score/reward strip → cohort (max 24 tiles) → agent panel (collapsible, closed) → KPIs in a 2-column grid (sparklines hidden) → event log (last 5) |
| Game Over | One column | One column; KPI snapshot and branch results as lists; hanko seal and clear mark 72 px |
| History | Table scrolls horizontally inside its container | Rows become cards: Game ID + outcome, DUT, game · difficulty, duration · score |
| Settings | Unchanged | Single column |
| Network model | Single column; Parameters table scrolls horizontally inside its container | Single column; drop zone full width; parameter inputs full width; the hero button wraps |

- No horizontal page scroll at any width from 320 px up.
- Touch targets are ≥ 44 × 44 px.
- Grain overlay is off below 768 px.

### 3.5 Transitions and animations

All timing and easing comes from the tokens in `design-standard.md` §5.2–§5.3. Choreography for these moments is defined in **`design-standard.md` §5.5** and is implemented exactly:

- route change (including the blade line drawing under `h1`)
- list entrance
- PLAY splash
- score update
- escalation
- KPI breach
- branch passed
- DUT failure (hanko stamp)
- level clear
- level unlock
- dialog
- tooltip
- running status dot

Contract-specific rules:

| Trigger | Behaviour |
| --- | --- |
| GENERATE pressed | Button shows a loading state for the simulated generation time (< 2 s), then routes to Briefing |
| Mode switched | Card grid cross-fades (`--dur-base`); no layout jump |
| Game tick | State updates every 100 ms. Only the score, reward and branch coverage bar animate; every other number swaps instantly. |
| UE traffic | Tiles pulse in opacity, staggered, on a linear loop of 1.2 s while `RUNNING` |
| RACH attempt | Small particles rise from tiles towards the DUT, 600 ms each, at most 30 on screen |
| Barring | A dashed wall fades in above the DUT over `--dur-base` |
| Handover / reselection (L3) | Tiles drift to the new serving cell (Cell 1, 2 or 3) over `--dur-slow`, `--ease-inout` |
| Message-flow selection (Briefing) | Panel content cross-fades over `--dur-fast`; no height animation |
| DUT failure → Game Over | Auto-routes 1.2 s after the hanko lands |
| Level clear → Game Over | Auto-routes 1.2 s after the clear mark lands |
| Returning to Lobby after a clear | The newly opened level card plays the level-unlock moment once |
| Tab hidden | The tick loop pauses on `visibilitychange`; elapsed time is reconciled on return |

**Reduced motion** follows `design-standard.md` §5.6.

| Trigger | Behaviour |
| --- | --- |
| Play mode switched | Lobby body cross-fades between the Manual and Autopilot field sets (`--dur-base`) |
| Command understood | Interpretation chips slide in left to right with a 40 ms stagger; the matching Lobby controls flash their selected state once |
| Listening | Mic ring pulses (reduced motion: static ring + "Listening…") |
| Campaign plan updated | Changed rows cross-fade; reordering animates row position over `--dur-base` |
| Autopilot countdown | Ring empties linearly over the countdown; reduced motion: number only |

### 3.6 Play modes

**Manual: one grammar, three inputs.** Clicks, chat and voice all write to the same Lobby state.
- **Parsing.** A command is lower-cased and tokenised, then matched against `DATA.commands`. The deterministic rules, in order:
  1. the target (gnb, ai-ran, modem);
  2. the DUT label (text after "for" or "on" that isn't a known keyword);
  3. the game type plus a game or level (number or name);
  4. branches ("only …", or causes, codes or case IDs that match branch names or `wf` references in that level);
  5. the difficulty;
  6. a queue ("then", "and then", ",").
- **Ambiguity.** If a token matches more than one slot value (e.g. "reject" in L1 and L2) or no level is given for a branch-only command, show the clarification instead of guessing.
- **No match.** Leave the Lobby unchanged and show "Didn't understand. Try e.g. 'L1 on Hard'." (polite live region).
- **Never auto-start.** A command only fills the Lobby (and the Briefing branch picker). GENERATE, PLAY and STOP still need their own confirmation. "stop" during a game opens the Stop dialog.
- **Locked levels.** A command naming a locked level selects nothing and says "Level 3 is locked for <DUT>. Clear L2 first."
- **Voice states.** `idle` → (press) `listening` → (release or 8 s silence) `transcribing` → transcript placed in the bar and parsed as chat → `idle`. An error ("Didn't catch that…") returns to `idle`. The recogniser runs only while the button is held or toggled on, never continuously.
- **Practice runs.** A ladder run with any branch unchecked is saved with `practice: true`: branch results are reported, but the level is never marked cleared and nothing unlocks.

**Autopilot: the campaign loop.**
1. **Plan.** Build the candidate set for the scope (stress games × difficulties up to the ceiling, open ladder levels, and per-level subsets of branches that failed before). Apply the hard rules in order:
   - the lowest uncleared open ladder level for this DUT build;
   - retests of games and branches whose `failedOn` contains an earlier build of the same family;
   - then UCB1 on the rest: `meanBreak + c·√(ln N / plays)`, with unplayed candidates scored ∞ and ties broken by the campaign seed.
   Take the top `planLength` as the plan, each with its reason and source.
2. **Play.** Generate and play the first planned game exactly as in Manual mode (same Briefing data, seed, engine and judging), with the Autopilot header fields.
3. **Learn.** At the end of the game, compute the break value:
   - the run's mean reward, normalised to 0–1;
   - plus `failBonus` if the outcome is DUT DOWN;
   - plus `branchFailBonus` per failed branch.
   Update `plays`, `meanBreak`, `failedOn` and `qByMove` for the candidate, update ladder unlocks as in Manual, and append the decision to the campaign record.
4. **Continue.** If the budget remains, recover the DUT (G24), re-plan (the plan can change after every game) and count down to the next game. Otherwise route to the Campaign Report.
5. **Override.** Skip, Pin and Take control act immediately. Edits to the plan screen persist for the campaign. Take control ends the campaign record with reason "Taken over by engineer" and produces a Campaign Report of what ran.

**Targets.** Both play modes work identically for gNB / AI-RAN and modem. The knowledge base key includes the target, so a gNB family and a modem family never share learning. Cold-start priors come from `CONFIG.autopilot.priors[target]`:
- **gNB:** stress games first, then L3.
- **modem:** L1/L2 failure branches first, then L3 RLF branches.

### 3.7 Network model

**Pipeline: read → map → convert and check → report → apply.** Nothing changes until the engineer applies.
- **Read.** A file starting with `{` or `[` is JSON; one starting with `<` is XML (parsed with the browser's `DOMParser`, no library). Other content shows "Use a JSON or XML file." JSON in the canonical format (`schema: "test-arena.network-model"`) is read from its `parameters` object. Any other JSON is flattened to leaf key/value pairs with their paths (an object `{name, value[, unit]}` counts as one entry). XML yields one entry per `name`/`key`/`param`/`parameter` attribute with a `value` attribute or text, per leaf element with text and per non-identifying attribute; `id` and `name` of container elements become part of the path (`NRCellDU=Cell2`). Files over 5 MB are refused.
- **Map.** Keys are normalised (lower case, letters and digits only) and matched against each catalogue parameter's key, label and `aliases`. A key that matches nothing is *Not mapped* ("No binding in the workflow library"); metadata keys (`name`, `vendor`, `operator`, `schema`, `version`, …) are skipped. Per-cell parameters (PCI, TAC) take their cell from the path (`Cell1`–`Cell3`) or, failing that, from the order of appearance; Cells 1–2 share one TAC and Cell 3 has its own, and a third distinct TAC is *Not mapped*. `plmn`/`plmnId` values such as `001-01` split into MCC and MNC. A second value for a parameter already set is *Duplicate* (the first one wins).
- **Convert and check.** Explicit units are converted (ms, s, min, h, dB, dBm). A unitless number keyed by the exact TS 38.331 IE spelling is read in IE units; any other unitless number is read in the catalogue unit. Enumerations must be in the TS 38.331 set (numbers map to `msN`, `nN`, `slN`, `sfN`, `rfN`, `dBN`, `neaN`, `niaN`). Integers must be in range and on the step. NAS timers must be encodable as a GPRS timer. Booleans accept 1/0, true/false, yes/no, on/off, supported/not supported, enabled/disabled. Failing values are *Rejected* with the reason; conversions are *Converted* with the rule ("600 ms → ms600").
- **Report and stage.** The import report and the impact preview render, the "APPLY TO SIGNALLING FLOW" button enables and focus moves to the report heading. A file with no mappable entry shows the error and stages nothing. Editing a value in the Parameters table stages the change the same way (source "edited in Test Arena").
- **Apply.** The staged changed values become the active model (values equal to the library default are dropped). Its fingerprint is an 8-hex hash of the sorted changed values (`library` when none). Every ladder branch and stress move is re-resolved through the model; a running game keeps the flows and load factors it started with.

**Effects of the active model.**
1. **IE values.** For every binding of a changed parameter, matching IEs in `messageLibrary`, `nasLibrary` and step PHY detail (`prach`, `phy`) show the converted value (codecs: digit arrays, PLMN BCD, 24-bit TAC hex, ×2 dBm, half-dB, GPRS timer 2/3, SST name, NAS algorithm names in 5G and EPS form). Per-cell bindings only replace IEs holding that cell's library value. Message names, step order and hex are unchanged; the inspector shows "Network model: n values below differ from this library encoding (…). The hex is the library's."
2. **Timing.** For each changed parameter with a `timer`, the flow's expiry step (`timers.expired`, or a message naming the timer and "expir…") moves by (model − library) milliseconds when the library gap from the timer's start (or from the previous step) equals the library value within 10%. Every later step moves with it; SFN and slot are recomputed (10 ms frames, 0.5 ms slots). Otherwise the step is annotated and nothing moves. DUT latency, which comes from library slot gaps, follows the new timing.
3. **Procedures.** With IMS voice over PS = 0 (library 1), every Level 5 branch whose flow contains an INVITE (except WF-17b EPS fallback and the VoPS branch itself) plays the `IMS-VoPS` row of the WF-09a response matrix at the INVITE, with its `expectedUE`. A changed N26 indication appends the N26 expectation to WF-17 branches.
4. **Simulation.** Bounded, deterministic load factors per KPI (0.85–1.2) from hysteresis, time-to-trigger, A3 offset, T310, T311, T304, preambleTransMax, ra-ResponseWindow, T300, waitTime, T3346 and q-RxLevMin multiply the cohort's pressure on the matching KPIs. The same seed + game + difficulty + model gives the same run.
5. **Records.** Each trace entry stores the model fingerprint, and the inspector decodes it with that model (sample runs and runs before a model decode with library values). Each run stores `networkModel` (name, fingerprint, source, file, changed parameters). Downloads include it, plus per trace entry the IE values it changed (`decode.networkModel`).

**Reset and export.** "Reset to library defaults" (after a confirm) clears the active model. "Export model" downloads the active model in the canonical format (`network-model-<fingerprint>.json`). The JSON and XML templates contain every parameter at its library default with its reference and allowed values; importing a template maps all 37 parameters and changes nothing.

### 3.8 Field-data lake

- **Embedding.** `python field_lake/embed_bundle.py` rebuilds the bundle from the lake and writes it between the `FIELD_DATA` markers in `index.html`. It changes nothing else in the file and warns when the bundle's `libraryVersion` differs from the embedded library.
- **Captures.** The testcase generator draws from lake sessions and the sample captures together:
  - First come the captures whose `features` (or, for sample captures, `feature`) include the chosen game or level: lake sessions, then samples.
  - Then come the rest: samples, then lake sessions.
  - The count rule is unchanged (1–5 captures). The same seed, game, difficulty and bundle give the same captures.
  - Runs record capture IDs only.
- **Import.** "Import bundle…" opens the file picker for one `.json` file. The file is read with `File.text()` and parsed natively; it never leaves the page. It is checked like the embedded bundle (schema, major version 0, sessions array; files over 5 MB are refused). The error line reads, for example, "Not a field-data bundle (schema test-arena.network-model)." or "Not valid JSON: …".
  - **Accepted:** the bundle replaces the active one at once. The panel re-renders, focus moves to its heading, and "Field-data bundle imported: n sessions" is announced. Testcases generated afterwards use it; a running game and an open Briefing keep their captures.
  - **Rejected:** the active bundle is unchanged.
- **Back to embedded.** "Use embedded bundle" removes the import from memory and storage without a confirm, because nothing is lost: the file is still on disk.
- **Library mismatch.** When `libraryVersion` differs from the embedded library, the bundle is still used and the warning badge says so. Workflow IDs are stable across 3.x libraries.
- **Privacy.** Bundle values are shown as they are. The lake masks identifiers before export, and the build never adds or reveals any.

---

## 4. Open items carried from the content model

These values use placeholders in `CONFIG`, each marked `// PLACEHOLDER (Gn)`:

- **G1:** per-game failure thresholds, including the five ladder KPI sets.
- **G2:** reward weights and score scale, per game.
- **G3:** difficulty parameters, including branch count per ladder difficulty.
- **G4:** capture schema. The assumed shape is `id`, `location`, `date`, `feature`, `profile`. The field-data lake now defines sessions (`id`, `area`, `start`, `wf`, see content model §8). When the bundle is adopted, `id` ← session `id`, `location` ← `area`, `date` ← `start`; `feature` and `profile` still need a mapping rule.
- **G13:** level-clear minimum run time per level and difficulty.
- **G23:** Autopilot exploration constant, break-value bonuses, plan length, countdown, default budget and ceiling, and cold-start priors per target.

Field-data lake items (no `CONFIG` placeholder): G29 field replay (S9) and RF-driven cohort pressure, both not built (the bundle seeds Briefing captures and is listed in Settings), G30 UE log tool formats, G31 PCAP decoding (Wireshark, NAS keys, direction heuristic), G32 synthetic fading and channel samples.

Network model items (no `CONFIG` placeholder; documented in the code): G25 vendor alias tables (unmapped keys are reported, never guessed), G26 parameters beyond the 37 bound today, G27 hex re-encoding by the adapter, G28 the bounded simulation effect (`nmLoadFactors`).

Assumed without a placeholder: ladder unlocks are scoped to the exact DUT label and never expire (G14); the mock emulates three cells (G15); one representative UE is narrated per branch (G20); the `defaults.verify` areas are shown as warnings, not blocked (G18); voice uses the browser recogniser, opt-in (G21); chat uses an in-page grammar (G22); the mock recovers the DUT instantly after a failure (G24); L1 branch 1 is composed from WF-07c steps 1–30 and WF-06b steps 23–44, L1 branch 13 adapts WF-07b to cause #21, and L4 branches are tester-defined (G17).

Replacing any placeholder must need no code changes outside `CONFIG`.

---

## 5. Acceptance Criteria

The project is **complete** when every criterion below passes on every target in §5.1.

Each criterion can be checked two ways:
- **Human:** follow the check in a browser and tick the box.
- **Agent:** run the same check with browser automation (navigate, click, read the DOM or `window.__arena` with `?debug=1`, read console and network logs) and report pass or fail per ID.

A criterion fails if it fails on any target.

### 5.1 Test targets

| Browser (latest stable) | Viewports |
| --- | --- |
| Chrome, Edge, Firefox, Safari | Desktop 1920 × 1080 · Laptop 1280 × 800 · Tablet 768 × 1024 · Phone 390 × 844 |

Performance criteria (§5.7) are measured in Chrome on a mid-range laptop (4-core CPU, integrated graphics) at 1920 × 1080.

### 5.2 Stack and build

| ID | Criterion | Check |
| --- | --- | --- |
| AC-S1 | Deliverable is one `index.html`, with all CSS and JS embedded | Repo contains no other `.css`/`.js` files; file opens from disk and works |
| AC-S2 | No frameworks, libraries, npm or build artefacts | No `package.json`, no `node_modules`, no `<script src>` |
| AC-S3 | Only Google Fonts requests leave the page | Network log over a full game shows requests only to `fonts.googleapis.com` / `fonts.gstatic.com` |
| AC-S4 | No console errors or warnings | Console is clean across the full flow on every browser |
| AC-S5 | Works with `localStorage` blocked | Private window or storage disabled: full flow works, including ladder unlocks within the session; session runs and unlocks simply don't persist |
| AC-S7 | Workflow library embedded and checked | `DATA.workflows` deep-equals `5g_signalling_workflows.json` (3.1.0). Changing `schemaVersion` to `4.0.0` (or removing `messageLibrary` or `nasLibrary`) makes ladder mode show "Workflow library version not supported" in the Lobby, with no console errors and stress games still playable. Removing `networkParameters` makes `#/network` show the "no networkParameters section" message and nothing else changes. |
| AC-S6 | Placeholders isolated | All G1–G4, G13 and G23 values sit in `CONFIG` with `// PLACEHOLDER (Gn)` comments. Changing one threshold changes behaviour with no other edits. |

### 5.3 Navigation and flow

| ID | Criterion | Check |
| --- | --- | --- |
| AC-N1 | Default route is the Lobby | Open with no hash → `#/lobby`, "SELECT GAME" visible |
| AC-N2 | Full happy path works | Lobby → GENERATE → Briefing → PLAY → Arena → STOP → confirm → Game Over "GAME STOPPED" |
| AC-N3 | Guards redirect correctly | `#/briefing` cold → `#/lobby`; `#/gameover/xyz` → `#/history`; `#/nonsense` → `#/lobby` |
| AC-N4 | Arena without a game | `#/arena` cold shows "No game running" + "Go to Lobby" |
| AC-N5 | Game survives navigation | During a game, go to History and back: elapsed time continued; shell indicator showed `PLAY nnn · time` throughout |
| AC-N6 | Back/forward behave | Browser back from Briefing returns to Lobby with selections kept |
| AC-N7 | Focus moves on route change | After each navigation, `document.activeElement` is the screen's `h1` |
| AC-N8 | Ladder path works | Ladder mode → L1 → GENERATE → PLAY → level clear → Game Over "LEVEL CLEAR" → NEXT LEVEL → Briefing for L2 with the same DUT label, target and difficulty |

### 5.4 Screens and content

| ID | Criterion | Check |
| --- | --- | --- |
| AC-C1 | Lobby validation | GENERATE disabled until target, DUT label (1–40 chars) and an open game or level are set. Empty label shows the exact validation message. |
| AC-C2 | Games, levels and difficulties | Stress mode shows exactly 4 game cards (names per §2.1); Ladder mode shows exactly 5 level cards in order L1–L5; Easy/Normal/Hard/Brutal with Normal preselected |
| AC-C3 | Briefing fields complete | Testcase ID matches `TC-YYYYMMDD-NNNN`; cohort total ≤ 60,000; 1–5 captures; generation time < 2.00 s; 8-char hex seed; game model and explainer present |
| AC-C4 | Arena shows all zones | Header, cohort arena, agent loop panel and diagnostics strip are all visible at 1920 × 1080 with no page scroll |
| AC-C5 | Live values update | Over 10 s, elapsed time advances 10 s (± 1) and at least the KPIs, reward and score change |
| AC-C6 | Endless until stop or failure | On Easy, a stress game is still `RUNNING` after 10 min. It ends only on STOP or a DUT failure. |
| AC-C7 | DUT failure path | On Brutal, a DUT failure occurs within 10 min. Hanko seal appears, then auto-route to Game Over with "DUT DOWN", failure reason, 5 triggering moves and KPI snapshot with breached rows marked. |
| AC-C8 | Deterministic replay | REPLAY on a failed run gives the same time to failure (± 1 tick) and the same final score. Replaying a cleared level gives the same clear time (± 1 tick) and score. |
| AC-C9 | Log download | "Download logs" saves `run-<id>.json` containing config, seed, moves and KPI snapshot (plus branch results for ladder runs) |
| AC-C10 | History records runs | Each completed game appears at the top of History with all 8 columns; "Open report" opens its Game Over view |
| AC-C11 | History filters | Filtering by game (including ladder levels), outcome and DUT text narrows the list; no match shows the exact empty-state text |
| AC-C12 | Settings read-only | Adapter shows `Mock (simulated)`; real mode disabled; "Clear session runs" removes session runs and unlocks after a confirm |
| AC-C13 | Content limits respected | No field exceeds its length limit in §2. Over-limit sample values truncate with an ellipsis + `title`. |
| AC-C14 | Subscriber IDs masked | No unmasked IMSI/SUPI anywhere in the DOM or downloaded logs |
| AC-C15 | Japanese sub-labels correct | Only fixed-vocabulary terms appear, each paired with English and `aria-hidden="true"` |
| AC-C16 | Progressive unlocks | With a fresh DUT label, only L1 is selectable; L2–L5 show `LOCKED` and the unlock hint. Clearing L1 opens L2 for that label only; a different DUT label still sees L2 locked. |
| AC-C17 | Unlock mode | Settings "All unlocked (demo)" makes all five levels selectable for any DUT label; switching back to Progressive restores real lock states |
| AC-C18 | Level-specific content | Each level's Briefing shows its own game model, explainer, branch set and clear rule; its Arena shows its own 3 state values, 3 reward components and 5 KPIs (per `content-model.md` §3c–§3d); L3 shows three cells |
| AC-C19 | Branch coverage | Branch coverage in the header increases only when a branch passes, matches the Briefing count `N`, and a "Branch passed" event is logged each time |
| AC-C20 | Level clear path | On Easy, L1 reaches 100% coverage and clears within 10 min with no DUT failure: clear mark appears, auto-route to Game Over "LEVEL CLEAR" with branch results all Passed and the unlock notice |
| AC-C22 | Workflow traceability | Every ladder branch row shows a workflow reference whose ID exists in `DATA.workflows` (case, variant and cause also exist), or `—` only for the branches listed as tester-defined in the PRD. All 17 library workflows are referenced by at least one branch or stress game. |
| AC-C23 | Message flow matches the library | For any selected branch, the Briefing's steps (All layers) equal the library steps of that flow in order and exact message name, including the library's NAS steps (e.g. L1 branch 2 = WF-07a with REGISTRATION REQUEST just before RRCSetupComplete); the injection step is tagged |
| AC-C24 | Step narration | During a ladder game, every step narrated in the event log (signalling steps only; PHY/MAC go to the trace) uses an exact library message or NAS name, and the header's workflow step advances in order |
| AC-C25 | Workflow step in reports | A ladder DUT failure's reason and failed branch row name the workflow and step; the downloaded log contains `wf`, `stepReached` and `schemaVersion` |
| AC-C26 | Settings shows the library | Workflow library panel shows `test-arena.signalling-workflows` 3.1.0, 17 workflows, "107 RRC messages", "92 NAS PDUs" and "10 defaults to verify" |
| AC-C27 | Level 5 plays | With All unlocked (demo), L5 runs on Easy with branches WF-03, WF-04, WF-09a, WF-09a · 480, shows the IMS KPI set and "DQN v0.3 · Bargaining", and can clear |
| AC-C28 | Trace shows the full flow | During any game the trace lists every narrated message with exact library names; stress RACH moves show Msg1: PRACH preamble, Msg2 DCI and RAR, Msg3, then RRCSetupRequest; ladder flows match the Briefing's message flow |
| AC-C29 | Messages decoded | RRCSetupRequest shows `ue-Identity` and `establishmentCause` in its ASN.1 tree; RRCSetup shows UPER hex, the contained CellGroupConfig and "Absent (OPTIONAL)" lines; REGISTRATION REJECT shows the played cause and the cause-matrix NAS hex (e.g. `7E 00 44 03` for #3) tagged "Cause matrix"; DLInformationTransfer shows "Carries NAS" |
| AC-C30 | DUT verdicts | gNB target: DL PHY/MAC/RRC rows and "gNB decodes PUSCH" carry a DUT verdict, UL rows read "UE sim", NAS rows "5GC emu". Latency is the library SFN/slot gap × load. Modem target: the UE lane is the DUT. Under barring, DUT responses read "late"; when rejecting, RRCSetup becomes "DUT sent RRCReject" or "DUT no response" |
| AC-C31 | Trace in reports | Every Game Over (session and sample runs) shows the trace, with the failing exchange preselected on DUT DOWN; the downloaded log has `signallingTrace` with sender, verdict, latency and `decode`, plus the referenced `messageLibrary` entries |
| AC-C32 | Trace usability | "PHY · MAC" off hides every transport row; "DUT responses only" hides tester rows; ↑/↓ moves the selection; no horizontal scroll at 320 px; message names wrap on phones |
| AC-C33 | PHY and MAC decoded | Selecting "Msg2: DCI 1_0 (RA-RNTI)" shows RA-RNTI, CORESET#0 and a field table including "Modulation and coding scheme"; "Msg2: Random Access Response" shows the MAC PDU with `timingAdvanceCommand` and `tempC-RNTI`; "HARQ-ACK" shows PUCCH with `harqProcess` and `counterDAI` |
| AC-C34 | Identity masking | No view and no downloaded log contains the reference IMSI in plain digits or in NAS BCD hex; the last 4 digits remain |
| AC-C35 | Library v3 coverage | Every L1–L3 and L5 branch resolves without `?`; L1 baseline is WF-13a; L1 branch 13 and L5 branches 4–6 are tagged "Adapted"; L5 "RLF during active call" plays WF-03 then WF-12 ("Composed"); every workflow WF-01 to WF-17 is referenced by a branch or stress move |
| AC-C36 | NAS decoded | Selecting REGISTRATION REQUEST shows "TS 24.501 8.2.6", a TS 24.301 equivalent, an IE table including "5GS registration type" and an "Absent (OPTIONAL)" list; a protected NAS message (e.g. SECURITY MODE COMMAND in WF-13a) shows the NAS security block and the protected hex |
| AC-C21 | Failure on a ladder level | A DUT failure during a level routes to "DUT DOWN" with branch results showing Passed / Failed / Not reached, and does not change any lock state |

### 5.4b Play modes

| ID | Criterion | Check |
| --- | --- | --- |
| AC-P1 | Play mode switch | Lobby shows Manual / Autopilot above everything else; Manual is default; switching keeps target and DUT label and swaps the field sets with no layout jump |
| AC-P2 | Chat fills the Lobby | Typing "L1 registration on Hard for gNB-X sw 4.2.1" + Enter sets Procedure Ladder, L1, Hard and the DUT label, shows the interpretation chips, and does **not** navigate or start anything |
| AC-P3 | Branch subset by command | "L1, only REJECT #22 and T3510" + GENERATE opens the Briefing with only the baseline, REJECT #22 and No response · T3510 checked, and the clear rule reads "Practice run" |
| AC-P4 | Ambiguity asks | "play reject" with no level shows a clarification with L1 and L2 choices and changes nothing until one is picked |
| AC-P5 | Locked level refused | With Progressive unlocks and a fresh DUT label, "play L3" selects nothing and shows the locked message |
| AC-P6 | Queue | "L1 then L2 on Easy" shows two queue chips; after L1 ends, Game Over offers "Next in queue: L2 · Easy" |
| AC-P7 | Voice (where supported) | With Voice input on in Chrome, holding the mic shows "Listening…"; the transcript lands in the bar and is parsed exactly like typed text. With Voice input off (default) or in an unsupported browser, no mic is shown and no recogniser is created |
| AC-P8 | Stop by command | During a game, "stop" in the Arena command bar opens the Stop dialog; the game keeps running until it is confirmed |
| AC-P9 | Campaign plan | Autopilot + PLAN CAMPAIGN on a fresh DUT label shows 5 planned games, the first being L1 (Ladder order), each with a reason and source; Remove, Up/Down and Pin change the plan |
| AC-P10 | Campaign runs unattended | START CAMPAIGN with a 00:05 budget plays games back to back with the Autopilot badge, reason and budget in the header; after each game the countdown starts the next one; at budget end, the Campaign Report appears |
| AC-P11 | Retest first | With a sample run where "SERVICE REJECT #22" failed on sw 4.2.1, a campaign for the same family on sw 4.2.2 lists that branch as a Retest within the first two picks |
| AC-P12 | Overrides | Skip saves the current run as STOPPED "Skipped by engineer" and plays the next pick; Take control switches to Manual with the game still running; Stop campaign routes to the Campaign Report |
| AC-P13 | Both targets | Manual (chat) and Autopilot each run end to end with target gNB / AI-RAN and with target Modem; knowledge rows are separate per target in Settings |
| AC-P14 | Deterministic campaign | The same knowledge base + campaign seed produces the same plan and decisions; the campaign download lists every decision with its inputs |

### 5.4c Network model

| ID | Criterion | Check |
| --- | --- | --- |
| AC-M1 | Route and nav | Nav shows Lobby · Arena · History · Network · Settings; `#/network` shows "NETWORK MODEL", 37 parameter rows in 9 groups and "LIBRARY DEFAULTS" on a fresh load |
| AC-M2 | Canonical JSON import | Sample "Operator A" reports 21 Mapped and nothing else; Apply makes 20 parameters differ, fingerprint ≠ `library` |
| AC-M3 | Vendor XML import | Sample "Vendor B" reports Mapped, Converted (e.g. "600 ms → ms600", "30 min → 1800 s", "false" → Not supported), Rejected (t311 7000; periodicRegistrationTimer 54 min), Not mapped (gNBId, maxNumHarqTx) and Duplicate (repeated ssbFrequency and Cells 1–2 TAC) |
| AC-M4 | Bad files | Invalid JSON, invalid XML, a JSON with no known key and a non-JSON/XML file each show their error and stage nothing; no console errors |
| AC-M5 | Editor validation | q-RxLevMin −141 shows "outside −140…−44" and is not staged; −120 is staged and enables Apply |
| AC-M6 | Values in messages | With Operator A applied, the SIB1 step decodes MCC 999, MNC 70, TAC 001234, t300 ms2000, t310 ms2000, n310 n4, t311 ms10000, preambleTransMax n8 and ra-ResponseWindow sl10 with the library values struck through and "Model" tags; REGISTRATION ACCEPT shows T3512 2 h and the TAI list with TAC 4660 |
| AC-M7 | Timer re-timing | With T310 = ms2000, L3 "Radio link failure · T310" has its T310 expiry at 21030.5 ms (library 20030.5 ms) and every later step 1000 ms later, with a "retimed +1000 ms" note; T311 (compressed in the library) is annotated, not moved |
| AC-M8 | Procedure switch | With IMS voice over PS = Not supported, L5 "VoNR MO call" resolves to `WF-03 · IMS-VoPS`, ends at the matrix row before INVITE and expects "No VoNR origination…"; resetting restores WF-03 |
| AC-M9 | Briefing and Game Over | Briefing shows the model name, parameter count and fingerprint; changed message-flow rows carry "Model"; Game Over shows the run's model |
| AC-M10 | Records and replay | A run's download has `networkModel.hash` equal to the active fingerprint and `decode.networkModel` on changed entries; REPLAY with the same model gives the same duration and score |
| AC-M11 | Running game isolation | Applying a model while a game runs leaves that game's flows, trace decodes and KPIs unchanged; the next game uses the new model |
| AC-M12 | Templates, export, reset, persistence | Importing the JSON or XML template maps 37 and changes nothing; Export downloads the canonical file, and re-importing it reproduces the fingerprint; Reset (after confirm) returns to "LIBRARY DEFAULTS"; a refresh keeps the active model |

### 5.4d Field-data lake

| ID | Criterion | Check |
| --- | --- | --- |
| AC-F1 | Bundle embedded and checked | `FIELD_DATA` sits between the markers; with `?debug=1`, `__arena.fieldData` reports `source: 'embedded'`, the schema version and the session count; replacing the block with `null` loads the app with no console error and Settings shows the "No field-data bundle" line |
| AC-F2 | Lake sessions seed testcases | For a game or level whose workflows a lake session matches (e.g. Signalling Storm or L2 with a WF-01 session), the Briefing's source captures start with that session, with a `LAKE` tag and its workflow; the caption counts lake and sample captures |
| AC-F3 | Deterministic captures | Regenerating with the same seed, game and difficulty gives the same capture IDs in the same order; replaying a run gives its recorded capture IDs |
| AC-F4 | Settings panel | The Field-data lake panel shows bundle, source, generated, mapped-against, contents and one table row per session with workflow, events, mapped %, RF points and seeded games; a library version mismatch shows the warning badge |
| AC-F5 | Import | Importing `export/fieldData.min.json` (or a copy with one session removed) replaces the bundle: the session count changes, source reads "Imported · <file>", and new testcases use it. No network request is made (network log empty) |
| AC-F6 | Import errors | A network-model JSON, invalid JSON and a file over 5 MB are each refused with their message, and the active bundle is unchanged |
| AC-F7 | Persistence and reset | After an import, a refresh keeps it (when storage allows); "Use embedded bundle" restores the embedded sessions and clears the stored copy |

### 5.5 Behaviour and visual conformance

| ID | Criterion | Check |
| --- | --- | --- |
| AC-V1 | Tokens only | Every colour, font, spacing, radius, duration and easing in the CSS references a `design-standard.md` token; no stray hex values outside `:root` |
| AC-V2 | Vermilion discipline | `--accent-shu` visible only on STOP, the `DUT DOWN` state, breached KPIs, failed branch rows, Rejected rows of the network-model import report and the hanko seal |
| AC-V3 | One hero action per view | At most one solid `--accent-primary` button visible on any screen |
| AC-V4 | Hover/press states | Each element type in `design-standard.md` §5.4 (including locked and cleared level cards) shows the specified hover, press, selected and disabled states |
| AC-V5 | Signature moments | PLAY splash, escalation, KPI breach, branch passed, DUT failure (hanko stamp), level clear and level unlock play as choreographed in `design-standard.md` §5.5 |
| AC-V6 | No bounce or overshoot | No easing outside the four tokens; no transition over 400 ms except the four epic moments |
| AC-V7 | Live numbers don't jitter | All live numbers use mono with tabular numerals; text width is stable while values change |
| AC-V8 | Reviewer sign-off | Side-by-side review of each screen against `design-standard.md` passes the §7 checklist |

### 5.6 Responsive

| ID | Criterion | Check |
| --- | --- | --- |
| AC-R1 | No horizontal page scroll | At 320, 390, 768, 1280 and 1920 px, `document.documentElement.scrollWidth ≤ innerWidth` on every route |
| AC-R2 | Layout changes per §3.4 | Each screen matches the tablet and mobile layouts in the §3.4 table, including the five-card snake, the level connector and the collapsed message flow |
| AC-R3 | Mobile nav | Below 768 px the hamburger opens and closes the drawer, toggles `aria-expanded`, and closes on `Esc` and on navigation |
| AC-R4 | Touch targets | Every interactive element is ≥ 44 × 44 px at 390 px width |
| AC-R5 | Playable on phone | Full happy path (AC-N2), failure path (AC-C7) and ladder path (AC-N8) complete at 390 × 844 using touch only |

### 5.7 Performance

| ID | Criterion | Check |
| --- | --- | --- |
| AC-P1 | Fast first load | Lighthouse (desktop) First Contentful Paint < 2 s and Performance score ≥ 90 |
| AC-P2 | Smooth arena | During a Hard game, the `?debug=1` fps reading averages ≥ 30 over 60 s, with no frame over 100 ms |
| AC-P3 | 60-minute unattended run | On Easy, a game runs 60 min unattended with no console errors, no freeze, and elapsed time accurate to ± 2 s |
| AC-P4 | No memory growth | Chrome JS heap at 60 min is within 10% of the heap at 5 min (DevTools or `performance.memory`) |
| AC-P5 | Background tab | Hiding the tab for 5 min pauses ticking; on return, elapsed time is reconciled and no tick burst occurs |

### 5.8 Accessibility (WCAG 2.2 AA)

| ID | Criterion | Check |
| --- | --- | --- |
| AC-A1 | Automated audit clean | axe-core, injected by the tester, reports zero serious or critical violations on every route and in each dialog state |
| AC-A2 | Keyboard only | AC-N2, AC-N8 and AC-C7 complete using only the keyboard; tab order matches visual order; no keyboard traps |
| AC-A3 | Focus visible and not obscured | Focus ring visible on every interactive element. The sticky shell never covers the focused element (WCAG 2.4.11). |
| AC-A4 | Contrast | Text meets 4.5:1 (3:1 for large text and UI components). `--text-3` is never used for essential text. |
| AC-A5 | Not colour alone | Game state, UE slices, outcomes, KPI breaches, level lock states and branch results are each identifiable by text, icon or shape as well as colour |
| AC-A6 | Reduced motion | With `prefers-reduced-motion: reduce`, no transform animations run, and all state changes are still visible |
| AC-A7 | Screen reader | Routes announce their `h1`. Live game state, branch passed, level clear and DUT failure are announced via one polite `aria-live` region, at most once per state change (not per tick). Locked level cards announce "locked" via `aria-disabled` and their label. Dialogs use `role="dialog"` + `aria-modal`. |
| AC-A8 | Target size | Interactive targets are ≥ 24 × 24 px on all viewports (WCAG 2.5.8), and ≥ 44 px on touch |

### 5.9 Definition of done

- [ ] Every AC above passes on every target in §5.1. Agent report attached, plus human checklist ticked.
- [ ] `CONFIG` placeholders (G1–G4, G13) are listed in the hand-off note with their current values.
- [ ] No open issue labelled blocker.
