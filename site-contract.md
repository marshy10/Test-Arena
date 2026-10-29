# Test Arena — Site Contract

*v2 · 29 Oct 2026 · Derived from `PRD.md` and `content-model.md`. Visuals follow `design-standard.md`; any visual value not stated here comes from that document's tokens.*

## 0. Scope of this contract

This contract defines a **static, client-side build** of Test Arena. It is a playable demo of the MVP experience, delivered as one `index.html` with embedded CSS and JavaScript, as set out in `CLAUDE.md`.

### What "static" means here

- **No backend calls.** The game loop is a JavaScript simulation (the mock adapter from M8). It uses a seeded pseudo-random engine plus data embedded in the file as JS objects:
  - `DATA.captures`
  - `DATA.games`
  - `DATA.difficulty`
  - `DATA.sampleRuns`
- **No form submission.** Every control changes client-side state only.
- **No auth, no database.**
  - Run History shows `DATA.sampleRuns` plus runs from the current session.
  - Session runs are held in memory.
  - `localStorage` is optional, try/catch guarded, and used only so a refresh keeps session runs.
- **Deterministic.** The same seed + game + difficulty always produces the same move sequence, failure point and final score.
- **Placeholders.** Every undefined value (gaps G1–G4) lives in one `CONFIG` object at the top of the script, with each entry commented `// PLACEHOLDER (Gn)`.
- **Debug hook.** Loading with `?debug=1` shows a small overlay with fps, tick count and elapsed time, and exposes `window.__arena` (read-only state snapshot) so acceptance tests can be automated. It is hidden without the query string.

### Departures from the PRD (by design)

| PRD item | In this static build |
| --- | --- |
| Live Multi-UE simulator, WebSocket at 10–20 Hz | JS simulation ticking at 10 Hz (100 ms) |
| Datalake queries | `DATA.captures`; "generation" picks and seeds from it |
| Instrument Settings (host, interface, test connection) | Read-only; adapter locked to `SIMULATED` |
| Log capture download (.zip) | Client-generated `.json` via `Blob` |
| 24 h unattended run, ≥ 10 fps at 60K UEs | 60 min unattended run, ≥ 30 fps (see §5.7) |
| Desktop only (1920×1080 min) | Desktop primary; tablet and phone layouts defined in §3.4 |

---

## 1. Page Structure

The site is a single-page app with hash routes. The shell is always present.

| # | Route | Section | Purpose |
| --- | --- | --- | --- |
| 0 | all | **App shell** | Keeps the product, current game and simulator state visible from every screen. |
| 1 | `#/lobby` (default) | **Game Lobby** | Lets the engineer choose a target, DUT label, stress game and difficulty to start a game. |
| 2 | `#/briefing` | **Testcase Briefing** | Shows the generated testcase and its seed captures so the engineer knows what will run before pressing Play. |
| 3 | `#/arena` | **Live Arena** | Plays the endless leader–follower game in real time, with the cohort, agent loop and diagnostics on one screen. |
| 3a | | Arena header | Identifies the game and DUT, shows elapsed time and state, and offers Stop. |
| 3b | | Cohort arena | Visualises the UE cohort attacking the DUT and the DUT's current response. |
| 3c | | Agent loop panel | Shows the State → Reward → Action cycle and the running score. |
| 3d | | Diagnostics strip | Shows the five live KPIs against their failure thresholds, plus the event log. |
| 4 | `#/gameover/:runId` | **Game Over** | Reports how and why the game ended, with the evidence needed to replay it. |
| 5 | `#/history` | **Run History** | Lists every saved game and opens any run's report or score timeline. |
| 6 | `#/settings` | **Instrument Settings** | Shows the simulated adapter configuration and simulator limits. |

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
| Target type | gNB / AI-RAN or Modem | Two-option segmented control; required; default gNB / AI-RAN |
| DUT label | Device name + build | Text input, required, 1–40 chars, trimmed |
| Game cards (×4) | Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze | Single select; content from `DATA.games` |
| Card · title | Game name | ≤ 20 chars, `h3` |
| Card · description | What the game does to the DUT | ≤ 90 chars; starts with a verb |
| Card · KPI tags | KPIs this game pressures | 2–3 tags from the five Diagnostics KPIs |
| Difficulty | Easy / Normal / Hard / Brutal | Segmented control; default Normal |
| Difficulty hint | What the level means in numbers | ≤ 60 chars, from `DATA.difficulty` |
| Validation message | Shown under DUT label if empty on Generate | Exactly: "Enter a DUT label to continue." |
| Primary button | "GENERATE" | Hero button. Disabled until target, DUT label and game are set. |

### 2. Testcase Briefing

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "BRIEFING" + JP 概要 | Display `h1` |
| Testcase ID | Unique ID for replay | Format `TC-YYYYMMDD-NNNN`; copyable |
| Summary | One-line description of the sequence | ≤ 120 chars |
| Cohort mix | UE counts per slice + stacked bar | URLLC / eMBB / mMTC integers; total ≤ 60,000 |
| Source captures | Captures that seeded the testcase | 1–5 items: capture ID · location · date |
| Stop conditions | What ends the game | "User Stop" + the game's failure rules from `CONFIG` |
| Generation time | Simulated generation duration | Seconds, 2 decimals, always < 2.00 |
| Seed | Deterministic seed | 8-char hex; copyable |
| Buttons | "PLAY" (hero), "Regenerate", "Back" | Regenerate draws a new seed |

### 3a. Arena header

| Field | Description | Constraints |
| --- | --- | --- |
| Game ID | e.g. "PLAY 042" | Display. Zero-padded 3+ digits, increments per session. |
| Game + difficulty | e.g. "Signalling Storm · Hard" | Read-only |
| DUT label | From Lobby | ≤ 40 chars |
| Elapsed time | Time since Play | `HH:MM:SS`, counts up |
| Game state badge | `RUNNING` / `ESCALATING` / `STOPPED` / `DUT DOWN` | Colours: hisui / primary / text-2 / shu |
| Stop button | "STOP" | Opens a confirm dialog: "Stop game and save run?" with "Stop" / "Keep playing" |

### 3b. Cohort arena

| Field | Description | Constraints |
| --- | --- | --- |
| Cohort tiles | Aggregated UE groups | ≤ 48 tiles. Shapes per `design-standard.md` §1.5: URLLC diamond, eMBB rounded rectangle, mMTC square. |
| Slice counts | Active UEs per slice | Integers, mono |
| Current move | Leader's current stress move | ≤ 30 chars |
| DUT sprite + status | gNB tower or modem, by target type | `Serving` / `Barring` / `Rejecting` / `Down` |
| Legend | URLLC · eMBB · mMTC · NETWORK | Colour + shape + label |

### 3c. Agent loop panel

| Field | Description | Constraints |
| --- | --- | --- |
| Panel title | "AGENT LOOP" + JP 戦略 | Display `h2`; `--accent-fuji` accents |
| State values | RACH attempts, PRB utilisation, rejects | `/s`, `%`, `/s` |
| Reward | Current reward | `display-l`, signed, 2 decimals, `--accent-kin` |
| Reward breakdown | Reject · Latency · Drop | Each signed, 2 decimals |
| Action | Agent's last move | ≤ 30 chars |
| Agent label | Algorithm + version | Fixed: "DQN v0.3 · Stackelberg" |
| Total score | Cumulative score | `display-l`, integer, `--accent-kin` |

### 3d. Diagnostics strip

| Field | Description | Constraints |
| --- | --- | --- |
| Panel title | "DIAGNOSTICS" + JP 診断 | Display `h2` |
| RACH success | Share of RACH attempts succeeding | `%`, 1 decimal; 60 s sparkline |
| PRB utilisation | DL + UL load | `%`, 1 decimal |
| Rejects | RRC / admission rejects | `/s`, integer |
| Latency p95 | User-plane latency | `ms`, 1 decimal |
| Drops | Dropped sessions | `/s`, integer |
| Threshold line | Failure threshold per KPI | From `CONFIG` (G1); turns `--accent-shu` when crossed |
| Event log | Last 20 events | `HH:MM:SS` + ≤ 80-char message, newest on top |

### 4. Game Over

| Field | Description | Constraints |
| --- | --- | --- |
| Banner | Outcome | "DUT DOWN" (shu, with hanko seal 故障) or "GAME STOPPED" (hisui, no seal); `display-xl` |
| Failure reason | What broke | ≤ 100 chars and names the KPI or event. Shows "Stopped by user" when stopped. |
| Time to failure | Elapsed at end | `HH:MM:SS` |
| Final score | Total score | `display-xl`, `--accent-kin` |
| Triggering moves | Last 5 moves | Move + timestamp, oldest first |
| KPI snapshot | 5 KPIs at end vs threshold | Breached rows marked in shu with an icon |
| Log download | "Download logs" | Generates `run-<id>.json` client-side |
| Replay seed | Seed + agent version | Copyable |
| Buttons | "REPLAY" (hero), "New game", "View in History" | REPLAY reruns with the same seed |

### 5. Run History

| Field | Description | Constraints |
| --- | --- | --- |
| Screen title | "HISTORY" + JP 履歴 | Display `h1` |
| Table columns | Date · Game ID · DUT · Game · Difficulty · Duration · Outcome · Score | Newest first |
| Outcome badge | How the game ended | `DUT DOWN` / `STOPPED` / `PLATFORM ERROR` (sample data only) |
| Filters | Game, Outcome, text search on DUT | Client-side |
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
| Datalake source | Capture source | "Embedded sample captures (N)" |
| Reset demo | "Clear session runs" | Clears in-memory and `localStorage` runs after a confirm |

---

## 3. Behaviour Spec

### 3.1 Navigation

- **Routing.** Hash-based (`#/route`), handled by one `hashchange` listener, with no page reloads.
  - An unknown hash redirects to `#/lobby`.
  - Back and forward follow history normally.
- **Flow.** Lobby → GENERATE → Briefing → PLAY → Arena → (Stop or DUT failure) → Game Over.
- **Guards.**
  - `#/briefing` without a generated testcase redirects to `#/lobby`.
  - `#/arena` with no running game shows "No game running" and a "Go to Lobby" link.
  - An unknown `:runId` redirects to `#/history`.
- **Leaving a running game.** Navigating away does **not** stop the game; it keeps ticking. Only STOP or a DUT failure ends it.
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

### 3.2 Scroll

- **Shell.** `position: sticky; top: 0`, 56 px high. `scroll-padding-top: 72px` keeps focused elements clear of it.
- **Lobby, Briefing, Game Over, History, Settings.** These scroll normally. Each route change scrolls to the top instantly.
- **Live Arena (desktop).**
  - Fixed height of `100vh − shell`, with no page scroll. Only the event log scrolls.
  - The log auto-sticks to the newest entry unless the user scrolls up. A "Jump to latest" chip then restores auto-stick.
- **History table.** The header row is sticky within its container.
- **Smooth scroll.** Used only for "Jump to latest", and disabled under reduced motion.

### 3.3 Hover, focus and active states by element type

- Hover, press, selected, disabled and focus visuals for buttons, STOP, game cards, nav items, table rows, segmented controls, inputs and icon buttons are defined in **`design-standard.md` §5.4**. Implement them exactly.
- This contract adds:

| Element | Behaviour |
| --- | --- |
| Cohort tile | Tooltip: slice · UE count · current move |
| KPI cell | Tooltip: current value · threshold · 60 s min/max |
| Copy control (IDs, seeds) | Tooltip "Copy"; after click the label reads "Copied" for 1.5 s; copies to clipboard |
| Disabled Arena nav | Tooltip "No game running" |
| All tooltips | 300 ms hover delay, immediate on focus, ≤ 60 chars, flip if clipped |

- Hover effects apply only under `@media (hover: hover)`. Touch devices get press states only.

### 3.4 Responsive layout

Breakpoints: **≥ 1280 px** desktop (primary), **768–1279 px** tablet, **< 768 px** mobile. The grid, gutters and type step-downs follow `design-standard.md` §2.3 and §3.4.

| Section | Tablet (768–1279) | Mobile (< 768) |
| --- | --- | --- |
| Shell | Nav inline; active game indicator shows Game ID only | Nav collapses to a hamburger (`aria-expanded`) opening a glass drawer; adapter badge becomes a dot + `SIM` |
| Lobby | Game cards 2 × 2 | Cards in 1 column; GENERATE sticky at bottom |
| Briefing | Single column | Single column; captures collapse to 2 + "Show all" |
| Arena | Cohort full width on top; Agent panel and Diagnostics side by side below; page scroll allowed | Stack: header → score/reward strip → cohort (max 24 tiles) → agent panel (collapsible, closed) → KPIs in a 2-column grid (sparklines hidden) → event log (last 5) |
| Game Over | One column | One column; KPI snapshot as a list; hanko seal 72 px |
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
- DUT failure (hanko stamp)
- dialog
- tooltip
- running status dot

Contract-specific rules:

| Trigger | Behaviour |
| --- | --- |
| GENERATE pressed | Button shows a loading state for the simulated generation time (< 2 s), then routes to Briefing |
| Game tick | State updates every 100 ms. Only the score and reward animate; every other number swaps instantly. |
| UE traffic | Tiles pulse in opacity, staggered, on a linear loop of 1.2 s while `RUNNING` |
| RACH attempt | Small particles rise from tiles towards the DUT, 600 ms each, at most 30 on screen |
| Barring | A dashed wall fades in above the DUT over `--dur-base` |
| DUT failure → Game Over | Auto-routes 1.2 s after the hanko lands |
| Tab hidden | The tick loop pauses on `visibilitychange`; elapsed time is reconciled on return |

**Reduced motion** follows `design-standard.md` §5.6.

---

## 4. Open items carried from the content model

These values use placeholders in `CONFIG`, each marked `// PLACEHOLDER (Gn)`:

- **G1:** per-game failure thresholds.
- **G2:** reward weights and score scale.
- **G3:** difficulty parameters.
- **G4:** capture schema. The assumed shape is `id`, `location`, `date`, `feature`, `profile`.

Replacing them must need no code changes outside `CONFIG`.

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
| AC-S5 | Works with `localStorage` blocked | Private window or storage disabled: full flow works; session runs simply don't persist |
| AC-S6 | Placeholders isolated | All G1–G4 values sit in `CONFIG` with `// PLACEHOLDER (Gn)` comments. Changing one threshold changes behaviour with no other edits. |

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

### 5.4 Screens and content

| ID | Criterion | Check |
| --- | --- | --- |
| AC-C1 | Lobby validation | GENERATE disabled until target, DUT label (1–40 chars) and game are set. Empty label shows the exact validation message. |
| AC-C2 | Four games and four difficulties | Exactly 4 game cards (names per §2.1) and Easy/Normal/Hard/Brutal with Normal preselected |
| AC-C3 | Briefing fields complete | Testcase ID matches `TC-YYYYMMDD-NNNN`; cohort total ≤ 60,000; 1–5 captures; generation time < 2.00 s; 8-char hex seed |
| AC-C4 | Arena shows all zones | Header, cohort arena, agent loop panel and diagnostics strip are all visible at 1920 × 1080 with no page scroll |
| AC-C5 | Live values update | Over 10 s, elapsed time advances 10 s (± 1) and at least the KPIs, reward and score change |
| AC-C6 | Endless until stop or failure | On Easy, the game is still `RUNNING` after 10 min. It ends only on STOP or a DUT failure. |
| AC-C7 | DUT failure path | On Brutal, a DUT failure occurs within 10 min. Hanko seal appears, then auto-route to Game Over with "DUT DOWN", failure reason, 5 triggering moves and KPI snapshot with breached rows marked. |
| AC-C8 | Deterministic replay | REPLAY on a failed run gives the same time to failure (± 1 tick) and the same final score |
| AC-C9 | Log download | "Download logs" saves `run-<id>.json` containing config, seed, moves and KPI snapshot |
| AC-C10 | History records runs | Each completed game appears at the top of History with all 8 columns; "Open report" opens its Game Over view |
| AC-C11 | History filters | Filtering by game, outcome and DUT text narrows the list; no match shows the exact empty-state text |
| AC-C12 | Settings read-only | Adapter shows `Mock (simulated)`; real mode disabled; "Clear session runs" removes session runs after a confirm |
| AC-C13 | Content limits respected | No field exceeds its length limit in §2. Over-limit sample values truncate with an ellipsis + `title`. |
| AC-C14 | Subscriber IDs masked | No unmasked IMSI/SUPI anywhere in the DOM or downloaded logs |
| AC-C15 | Japanese sub-labels correct | Only fixed-vocabulary terms appear, each paired with English and `aria-hidden="true"` |

### 5.5 Behaviour and visual conformance

| ID | Criterion | Check |
| --- | --- | --- |
| AC-V1 | Tokens only | Every colour, font, spacing, radius, duration and easing in the CSS references a `design-standard.md` token; no stray hex values outside `:root` |
| AC-V2 | Vermilion discipline | `--accent-shu` visible only on STOP, the `DUT DOWN` state, breached KPIs and the hanko seal |
| AC-V3 | One hero action per view | At most one solid `--accent-primary` button visible on any screen |
| AC-V4 | Hover/press states | Each element type in `design-standard.md` §5.4 shows the specified hover, press, selected and disabled states |
| AC-V5 | Signature moments | PLAY splash, escalation, KPI breach and DUT failure (hanko stamp) play as choreographed in `design-standard.md` §5.5 |
| AC-V6 | No bounce or overshoot | No easing outside the four tokens; no transition over 400 ms except the three epic moments |
| AC-V7 | Live numbers don't jitter | All live numbers use mono with tabular numerals; text width is stable while values change |
| AC-V8 | Reviewer sign-off | Side-by-side review of each screen against `design-standard.md` passes the §7 checklist |

### 5.6 Responsive

| ID | Criterion | Check |
| --- | --- | --- |
| AC-R1 | No horizontal page scroll | At 320, 390, 768, 1280 and 1920 px, `document.documentElement.scrollWidth ≤ innerWidth` on every route |
| AC-R2 | Layout changes per §3.4 | Each screen matches the tablet and mobile layouts in the §3.4 table |
| AC-R3 | Mobile nav | Below 768 px the hamburger opens and closes the drawer, toggles `aria-expanded`, and closes on `Esc` and on navigation |
| AC-R4 | Touch targets | Every interactive element is ≥ 44 × 44 px at 390 px width |
| AC-R5 | Playable on phone | Full happy path (AC-N2) and failure path (AC-C7) complete at 390 × 844 using touch only |

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
| AC-A2 | Keyboard only | AC-N2 and AC-C7 complete using only the keyboard; tab order matches visual order; no keyboard traps |
| AC-A3 | Focus visible and not obscured | Focus ring visible on every interactive element. The sticky shell never covers the focused element (WCAG 2.4.11). |
| AC-A4 | Contrast | Text meets 4.5:1 (3:1 for large text and UI components). `--text-3` is never used for essential text. |
| AC-A5 | Not colour alone | Game state, UE slices, outcomes and KPI breaches are each identifiable by text, icon or shape as well as colour |
| AC-A6 | Reduced motion | With `prefers-reduced-motion: reduce`, no transform animations run, and all state changes are still visible |
| AC-A7 | Screen reader | Routes announce their `h1`. Live game state and DUT failure are announced via one polite `aria-live` region, at most once per state change (not per tick). Dialogs use `role="dialog"` + `aria-modal`. |
| AC-A8 | Target size | Interactive targets are ≥ 24 × 24 px on all viewports (WCAG 2.5.8), and ≥ 44 px on touch |

### 5.9 Definition of done

- [ ] Every AC above passes on every target in §5.1. Agent report attached, plus human checklist ticked.
- [ ] `CONFIG` placeholders (G1–G4) are listed in the hand-off note with their current values.
- [ ] No open issue labelled blocker.
