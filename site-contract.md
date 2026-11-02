# Test Arena — Site Contract

*v4 · 2 Nov 2026 · Derived from `PRD.md` (v1.2) and `content-model.md`. Visuals follow `design-standard.md`; any visual value not stated here comes from that document's tokens.*

> **Change log**
> - **v4 (2 Nov 2026):** Games driven by the signalling workflow library (PRD M11): `DATA.workflows`, workflow references on every branch, Briefing message flow, Arena step narration, workflow steps in reports, Level 5 (IMS Voice & Video), three cells for L3, Settings workflow panel, and acceptance criteria AC-S7 and AC-C22–AC-C27.
> - **v3 (2 Nov 2026):** Added the Procedure Ladder (PRD M10): Lobby mode toggle and level cards with unlocks, ladder fields on Briefing, Arena and Game Over, `LEVEL CLEAR` outcome, `DATA.ladder`, placeholder G13, and acceptance criteria AC-N8 and AC-C16–AC-C21.
> - **v2 (29 Oct 2026):** Previous release.

## 0. Scope of this contract

This contract defines a **static, client-side build** of Test Arena. It is a playable demo of the MVP experience, delivered as one `index.html` with embedded CSS and JavaScript, as set out in `CLAUDE.md`.

### What "static" means here

- **No backend calls.** The game loop is a JavaScript simulation (the mock adapter from M8). It uses a seeded pseudo-random engine plus data embedded in the file as JS objects:
  - `DATA.captures`
  - `DATA.games` (the four stress games)
  - `DATA.ladder` (the five ladder levels: game model, explainer, branch set, state inputs, reward components and KPI set, as listed in `content-model.md` §3c–§3d and the PRD catalogue). Each branch carries a `wf` reference: `{ id: 'WF-07', case?: 'WF-07b', cause?: '#22', variant?: 'WF-05c', retune?: 'early' }`, or `null` for tester-defined branches.
  - `DATA.workflows` (an embedded copy of `5g_signalling_workflows.json`, unchanged apart from whitespace). At boot the script checks `schema === 'test-arena.signalling-workflows'` and a `schemaVersion` with major version 1. Message flows, expected behaviour, spec baseline and verify counts are read from it, never retyped.
  - `DATA.difficulty`
  - `DATA.sampleRuns` (includes at least one ladder run of each outcome)
- **No form submission.** Every control changes client-side state only.
- **No auth, no database.**
  - Run History shows `DATA.sampleRuns` plus runs from the current session.
  - Session runs and ladder unlocks (keyed by exact DUT label) are held in memory.
  - `localStorage` is optional, try/catch guarded, and used only so a refresh keeps session runs and unlocks.
- **Deterministic.** The same seed + game (or level) + difficulty always produces the same move or branch sequence, failure or clear point, and final score.
- **Placeholders.** Every undefined value (gaps G1–G4 and G13) lives in one `CONFIG` object at the top of the script, with each entry commented `// PLACEHOLDER (Gn)`.
- **Debug hook.** Loading with `?debug=1` shows a small overlay with fps, tick count and elapsed time, and exposes `window.__arena` (read-only state snapshot, including ladder unlocks and branch coverage) so acceptance tests can be automated. It is hidden without the query string.

### Departures from the PRD (by design)

| PRD item | In this static build |
| --- | --- |
| Live Multi-UE simulator, WebSocket at 10–20 Hz | JS simulation ticking at 10 Hz (100 ms) |
| Datalake queries | `DATA.captures`; "generation" picks and seeds from it |
| Instrument Settings (host, interface, test connection) | Read-only; adapter locked to `SIMULATED` |
| Log capture download (.zip) | Client-generated `.json` via `Blob` |
| 24 h unattended run, ≥ 10 fps at 60K UEs | 60 min unattended run, ≥ 30 fps (see §5.7) |
| Desktop only (1920×1080 min) | Desktop primary; tablet and phone layouts defined in §3.4 |
| Ladder clear minimum run time (G13) | `CONFIG` placeholder; short defaults so a level can be cleared in a demo |
| Workflow library loaded from a file and replayed through the adapter | Embedded as `DATA.workflows`; the simulation narrates the steps and judges branches by its own model |
| Cohort fan-out of workflows (G20) | One representative UE's steps are narrated; KPIs come from the cohort model |
| Three emulated cells for Level 3 | Mock emulates Cell 1, Cell 2 (TAC 000001) and Cell 3 (TAC 000002) |

---

## 1. Page Structure

The site is a single-page app with hash routes. The shell is always present.

| # | Route | Section | Purpose |
| --- | --- | --- | --- |
| 0 | all | **App shell** | Keeps the product, current game and simulator state visible from every screen. |
| 1 | `#/lobby` (default) | **Game Lobby** | Lets the engineer choose a mode, target, DUT label, stress game or ladder level, and difficulty to start a game. |
| 2 | `#/briefing` | **Testcase Briefing** | Shows the generated testcase, its game model and seed captures (and, for a ladder level, its branch set, each branch's workflow message flow and the clear rule) so the engineer knows exactly what will go on the air before pressing Play. |
| 3 | `#/arena` | **Live Arena** | Plays the leader–follower game in real time, with the cohort, agent loop and diagnostics on one screen. |
| 3a | | Arena header | Identifies the game and DUT, shows elapsed time, state and (ladder) branch coverage, and offers Stop. |
| 3b | | Cohort arena | Visualises the UE cohort attacking the DUT and the DUT's current response. |
| 3c | | Agent loop panel | Shows the State → Reward → Action cycle and the running score. |
| 3d | | Diagnostics strip | Shows the game's five live KPIs against their failure thresholds, plus the event log. |
| 4 | `#/gameover/:runId` | **Game Over** | Reports how and why the game ended (failure, stop or level clear), with the evidence needed to replay it. |
| 5 | `#/history` | **Run History** | Lists every saved game and opens any run's report or score timeline. |
| 6 | `#/settings` | **Instrument Settings** | Shows the simulated adapter configuration, simulator limits, the ladder unlock mode and the loaded workflow library. |

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
| Nav items | Lobby · Arena · History · Settings, each with a JP sub-label | 1 word each. Arena is `aria-disabled` when no game is running. |
| Adapter badge | Simulator state | Always `SIMULATED`; `--accent-warn` |
| Active game indicator | Game ID + elapsed time + running status dot | Format `PLAY 042 · 01:23:45`. Hidden when no game runs; links to `#/arena`. |

### 1. Game Lobby

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "SELECT GAME" + JP sub-label ロビー + blade line | Display `h1` |
| Mode | Stress games or Procedure Ladder | Two-option segmented control; default Stress games. Switching mode clears the game/level selection but keeps target, DUT label and difficulty. |
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
| Primary button | "GENERATE" | Hero button. Disabled until target, DUT label and an open game or level are set. |

### 2. Testcase Briefing

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "BRIEFING" + JP 概要 | Display `h1` |
| Testcase ID | Unique ID for replay | Format `TC-YYYYMMDD-NNNN`; copyable |
| Summary | One-line description of the sequence | ≤ 120 chars |
| Game model | Model name | ≤ 40 chars; `--accent-fuji` label; e.g. "Sequential game · subgame perfection" |
| Model explainer | What the model means for this test | ≤ 140 chars, `body`, `--text-2` |
| Branch set (Ladder) | Branches to cover | Ordered list, 4–18 items, count per difficulty from `CONFIG`; each ≤ 40 chars plus its workflow reference; rows are toggle buttons (`aria-pressed`) that select which message flow is shown; baseline selected by default |
| Message flow (Ladder) | Selected branch's workflow steps | Panel titled "MESSAGE FLOW" + JP 手順. Table: seq · dir glyph · layer · message (+ container) · timers. Steps from `extends`/`ref` workflows are expanded inline. The injection step is tagged "Injection" (`--accent-warn` tag). Tester-defined branches show "No library workflow (tester-defined)". |
| Expected behaviour (Ladder) | What passes the selected branch | ≤ 160 chars from `passCriteria` / `expectedUE` |
| Spec baseline | Library spec references | ≤ 100 chars, `caption` |
| Defaults to verify | `verify: true` fields used by this testcase | `--accent-warn` badge "n defaults to verify"; tooltip lists up to 3 field paths |
| Clear rule (Ladder) | What clears the level | ≤ 80 chars; built from branch count + `CONFIG` minimum time (G13) |
| Cohort mix | UE counts per slice + stacked bar | URLLC / eMBB / mMTC integers; total ≤ 60,000 |
| Source captures | Captures that seeded the testcase | 1–5 items: capture ID · location · date |
| Stop conditions | What ends the game | "User Stop" + the game's failure rules from `CONFIG`; ladder adds "Level clear" |
| Generation time | Simulated generation duration | Seconds, 2 decimals, always < 2.00 |
| Seed | Deterministic seed | 8-char hex; copyable |
| Buttons | "PLAY" (hero), "Regenerate", "Back" | Regenerate draws a new seed |

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
| Stop button | "STOP" | Opens a confirm dialog: "Stop game and save run?" with "Stop" / "Keep playing" |

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
| Triggering moves | Last 5 moves | Move + timestamp, oldest first |
| KPI snapshot | The game's 5 KPIs at end vs threshold | Breached rows marked in shu with an icon |
| Log download | "Download logs" | Generates `run-<id>.json` client-side (ladder runs include branch results with workflow references, the step reached, and the library `schemaVersion`) |
| Replay seed | Seed + agent version | Copyable |
| Buttons | Depends on outcome | Default: "REPLAY" (hero), "New game", "View in History". After L1–L4 clear: "NEXT LEVEL" (hero), "Replay", "View in History". After L5 clear: "NEW GAME" (hero), "Replay", "View in History". |

### 5. Run History

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "HISTORY" + JP 履歴 | Display `h1` |
| Table columns | Date · Game ID · DUT · Game · Difficulty · Duration · Outcome · Score | Newest first. Ladder rows show Game as "L1 Registration". |
| Outcome badge | How the game ended | `DUT DOWN` / `STOPPED` / `LEVEL CLEAR` / `PLATFORM ERROR` (sample data only) |
| Filters | Game (stress games + five ladder levels), Outcome, text search on DUT | Client-side |
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
| Workflow library | Loaded library | Panel titled "Workflow library": schema + version, technology, generated date, 12 workflows (listed as ID · name · category), spec baseline, "29 fields to verify" warning badge |
| Datalake source | Capture source | "Embedded sample captures (N)" |
| Ladder unlocks | Progressive or All unlocked (demo) | Segmented control; default Progressive. "All unlocked (demo)" opens every level for every DUT label without recording clears. |
| Reset demo | "Clear session runs" | Clears in-memory and `localStorage` runs and ladder unlocks after a confirm |

---

## 3. Behaviour Spec

### 3.1 Navigation

- **Routing.** Hash-based (`#/route`), handled by one `hashchange` listener, with no page reloads.
  - An unknown hash redirects to `#/lobby`.
  - Back and forward follow history normally.
- **Flow.** Lobby → GENERATE → Briefing → PLAY → Arena → (Stop, DUT failure or Level clear) → Game Over.
- **Briefing branch selection.** Clicking a branch row (or Enter/Space on it) selects it and swaps the message-flow panel without a route change; focus stays on the row.
- **Ladder flow.** On a level clear, Game Over offers NEXT LEVEL. It generates the next level's testcase with the same target, DUT label and difficulty (new seed) and routes straight to Briefing.
- **Unlocks.** A clear in Progressive mode marks the level `CLEARED` and the next level `OPEN` for that exact DUT label. A DUT failure or Stop changes no lock state.
- **Guards.**
  - `#/briefing` without a generated testcase redirects to `#/lobby`.
  - A testcase for a level that is locked for its DUT label can't be generated; if one is reached through history state, `#/briefing` redirects to `#/lobby`.
  - `#/arena` with no running game shows "No game running" and a "Go to Lobby" link.
  - An unknown `:runId` redirects to `#/history`.
- **Leaving a running game.** Navigating away does **not** stop the game; it keeps ticking. Only STOP, a DUT failure or a level clear ends it.
- **Active nav item.** Gets `aria-current="page"` and the active state from `design-standard.md` §5.4.
- **Keyboard.**
  - Every control is reachable by Tab in visual order.
  - `Enter`/`Space` activate.
  - `Esc` closes dialogs and the mobile nav.
  - `S` opens the Stop dialog in the Arena.
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

---

## 4. Open items carried from the content model

These values use placeholders in `CONFIG`, each marked `// PLACEHOLDER (Gn)`:

- **G1:** per-game failure thresholds, including the five ladder KPI sets.
- **G2:** reward weights and score scale, per game.
- **G3:** difficulty parameters, including branch count per ladder difficulty.
- **G4:** capture schema. The assumed shape is `id`, `location`, `date`, `feature`, `profile`.
- **G13:** level-clear minimum run time per level and difficulty.

Assumed without a placeholder: ladder unlocks are scoped to the exact DUT label and never expire (G14); the mock emulates three cells (G15); one representative UE is narrated per branch (G20); `verify: true` fields are shown as warnings, not blocked (G18); L1 branch 1 is built from WF-07 steps 1–4 + ACCEPT and L4 branches are tester-defined (G17).

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
| AC-S7 | Workflow library embedded and checked | `DATA.workflows` deep-equals `5g_signalling_workflows.json`. Changing `schemaVersion` to `2.0.0` makes ladder mode show "Workflow library version not supported" in the Lobby, with no console errors and stress games still playable. |
| AC-S6 | Placeholders isolated | All G1–G4 and G13 values sit in `CONFIG` with `// PLACEHOLDER (Gn)` comments. Changing one threshold changes behaviour with no other edits. |

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
| AC-C22 | Workflow traceability | Every ladder branch row shows a workflow reference whose ID exists in `DATA.workflows` (case, variant and cause also exist), or `—` only for the branches listed as tester-defined in the PRD. All 12 library workflows are referenced by at least one branch or stress game. |
| AC-C23 | Message flow matches the library | For any selected branch, the Briefing's steps equal the library steps in count, order, direction, layer and exact message name (with `extends`/`ref` expanded); the injection step is tagged |
| AC-C24 | Step narration | During a ladder game, every step narrated in the event log uses an exact library message name, and the header's workflow step advances in order |
| AC-C25 | Workflow step in reports | A ladder DUT failure's reason and failed branch row name the workflow and step; the downloaded log contains `wf`, `stepReached` and `schemaVersion` |
| AC-C26 | Settings shows the library | Workflow library panel shows `test-arena.signalling-workflows` 1.0.0, 12 workflows and "29 fields to verify" |
| AC-C27 | Level 5 plays | With All unlocked (demo), L5 runs on Easy with branches WF-03, WF-04, WF-09a, WF-09b, shows the IMS KPI set and "DQN v0.3 · Bargaining", and can clear |
| AC-C21 | Failure on a ladder level | A DUT failure during a level routes to "DUT DOWN" with branch results showing Passed / Failed / Not reached, and does not change any lock state |

### 5.5 Behaviour and visual conformance

| ID | Criterion | Check |
| --- | --- | --- |
| AC-V1 | Tokens only | Every colour, font, spacing, radius, duration and easing in the CSS references a `design-standard.md` token; no stray hex values outside `:root` |
| AC-V2 | Vermilion discipline | `--accent-shu` visible only on STOP, the `DUT DOWN` state, breached KPIs, failed branch rows and the hanko seal |
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
