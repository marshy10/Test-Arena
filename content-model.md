# Test Arena — Content Model (Must-Have / MVP)

*Companion to the Anritsu AI-RAN Test Arena PRD · 29 Oct 2026*

## Page order

The MVP needs six screens plus a persistent shell. Each maps to Must-Haves M1–M9 and to nothing else.

0. **App shell**: persistent top bar on all screens
1. **Game Lobby**: choose target, feature and difficulty (M1)
2. **Testcase Briefing**: review the generated testcase before Play (M2)
3. **Live Arena**: cohort, DUT, agent loop and diagnostics strip (M3–M7)
4. **Game Over**: failure report and log capture (M3, M7)
5. **Run History**: saved games and replay (M9)
6. **Instrument Settings**: adapter connection (M8)

### Shared rules for every field

- **Tone has two registers.**
  - *Arcade* (Press Start 2P, uppercase, ≤ 3 words) is used only for titles, the score and game-state banners.
  - *Instrument* (Inter / JetBrains Mono, sentence case, plain and exact) is used for everything else.
- **Numbers** always carry a unit. KPIs use JetBrains Mono with fixed decimals.
- **Timestamps** are stored as ISO 8601. They display as `HH:MM:SS` in the arena and `DD MMM YYYY HH:MM` in history, in local time.
- **Identifiers** (IMSI/SUPI) are masked to the last 4 digits wherever they are shown.

---

## 0. App shell

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Product name | Text | "Test Arena" wordmark + Anritsu logo | Fixed; brand green only here |
| Nav items | Text ×4 | Lobby · Arena · History · Settings | 1 word each; Arena disabled when no game runs |
| Adapter status | Badge | Connection to instrument | Enum: `LIVE` / `SIMULATED` / `OFFLINE`; always visible |
| Active game indicator | Badge | Game ID + elapsed time when a game runs | `PLAY 042 · 01:23:45`; links to Arena |

## 1. Game Lobby (M1)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Screen title | Arcade | "SELECT GAME" | ≤ 3 words, uppercase |
| Target type | Toggle | What is being stressed | Enum: `gNB / AI-RAN`, `Modem`; required |
| DUT label | Text input | Name + build of the device, e.g. "gNB-X sw 4.2.1" | Required, ≤ 40 chars; saved with every run |
| Feature under test | Card picker | One card per stress game | Enum (4): Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze; single select |
| Game card · description | Text | What the game does to the DUT | ≤ 90 chars, instrument tone, starts with a verb ("Floods RACH with…") |
| Game card · KPIs attacked | Tags | KPIs this game pressures | 2–3 tags from the Diagnostics strip KPI list |
| Difficulty | Segmented control | Stress intensity | Enum: Easy / Normal / Hard / Brutal; default Normal |
| Difficulty hint | Text | What the level means in numbers | ≤ 60 chars, e.g. "10K UEs · 2× burst rate" |
| Primary action | Button | Generates the testcase | Label "GENERATE"; disabled until target, DUT label and feature are set |

## 2. Testcase Briefing (M2)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Testcase ID | Mono text | Unique ID for replay | Format `TC-YYYYMMDD-NNNN`; read-only |
| Summary | Text | One-line description of the generated sequence | ≤ 120 chars, instrument tone |
| Cohort mix | 3 numbers + bar | UE counts per slice | URLLC / eMBB / mMTC; integers; total ≤ 60,000 |
| Source captures | List | Datalake captures used to seed the testcase | 1–5 items: capture ID, location, date; no subscriber IDs |
| Stop conditions | List | What ends the game | Always "User Stop" + the game's failure rules (see G1) |
| Generation time | Mono number | Proof of the < 2 s target | Seconds, 2 decimals |
| Actions | Buttons | Start or go back | "PLAY" (primary), "Regenerate", "Back" |

## 3. Live Arena (M3–M7)

The screen has four zones. All live values update at 10–20 Hz.

### 3a. Arena header

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Game ID | Arcade | e.g. "PLAY 042" | Zero-padded, 3+ digits |
| Game name + difficulty | Text | e.g. "Signalling Storm · Hard" | From Lobby; read-only |
| DUT label | Text | Device being tested | From Lobby; ≤ 40 chars |
| Elapsed time | Mono | Time since Play | `HH:MM:SS`, counts up, no upper limit |
| Game state | Badge | Current state | Enum: `RUNNING` / `ESCALATING` / `STOPPED` / `DUT DOWN` |
| Stop control | Button | Ends the game | Label "STOP"; one confirm: "Stop game and save run?" |

### 3b. Cohort arena (canvas)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Cohort tiles | Sprite grid | Aggregated UE groups, not single UEs | ≤ 48 tiles; each = slice + count; shape differs per slice |
| Slice counts | Mono numbers | Active UEs per slice | URLLC / eMBB / mMTC; integers |
| Current move | Text | Stress move the leader is playing | Enum from game; ≤ 30 chars, e.g. "RACH burst ×4" |
| DUT status | Icon + text | gNB tower or modem sprite with state | Enum: `Serving` / `Barring` / `Rejecting` / `Down` |
| Legend | Fixed | URLLC · eMBB · mMTC · NETWORK | Static; colour + shape |

### 3c. Agent loop panel

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| State | 3 mono values | RACH attempts, PRB utilisation, rejects | Units: /s, %, /s |
| Reward | Arcade number | Current reward from QoS | Signed, 2 decimals, e.g. "+12.75"; yellow only |
| Reward breakdown | Text | Components driving reward | Reject · Latency · Drop, each signed, 2 decimals |
| Action | Text | Agent's last chosen move | ≤ 30 chars; same vocabulary as "Current move" |
| Agent | Text | Algorithm + version | e.g. "DQN v0.3 · Stackelberg"; read-only |
| Total score | Arcade number | Cumulative score this game | Integer, counts up |

### 3d. Diagnostics strip (always visible)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| RACH success | Mono + sparkline | Share of RACH attempts succeeding | %, 1 decimal, last 60 s |
| PRB utilisation | Mono + sparkline | Downlink + uplink load | %, 1 decimal |
| Rejects | Mono + sparkline | RRC / admission rejects | Per second, integer |
| Latency | Mono + sparkline | User-plane latency, p95 | ms, 1 decimal |
| Drops | Mono + sparkline | Dropped sessions | Per second, integer |
| Threshold marker | Line | Failure threshold per KPI | Turns red when crossed; values TBD (see G1) |
| Event log | Scrolling list | Last 20 events | `HH:MM:SS` + ≤ 80-char message; IMSI masked |

## 4. Game Over (M3, M7)

This screen appears when the DUT fails or the user stops the game. Only the banner differs between the two cases; the rest of the report is the same.

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Banner | Arcade | Outcome | Enum: "DUT DOWN" (red) / "GAME STOPPED" (green) |
| Failure reason | Text | What broke | ≤ 100 chars, instrument tone, names the KPI or event, e.g. "gNB crash after RRC setup storm" |
| Time to failure | Mono | Elapsed time at failure | `HH:MM:SS` |
| Final score | Arcade number | Total score | Integer |
| Triggering moves | Ordered list | Last 5 moves before failure | Move + timestamp, oldest first |
| KPI snapshot | Table | All 5 KPIs at failure vs threshold | Same units as Diagnostics strip |
| Log capture | File link | Logs + game state for the last N seconds | Default N = 30 s; download as .zip; IMSI masked |
| Replay seed | Mono | Seed + agent version for exact replay | Copyable |
| Actions | Buttons | Next step | "REPLAY", "New game", "View in History" |

## 5. Run History (M9)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Run list · columns | Table | One row per game | Date, Game ID, DUT label, Game, Difficulty, Duration, Outcome, Score |
| Outcome | Badge | How the game ended | Enum: `DUT DOWN` / `STOPPED` / `PLATFORM ERROR` |
| Filters | Controls | Narrow the list | DUT label, Game, Outcome, date range |
| Run detail · score timeline | Line chart | Score over the game | x = elapsed time, y = score |
| Run detail · report | Link | Opens the Game Over report | Same fields as screen 4 |
| Empty state | Text | Shown before first run | ≤ 60 chars, e.g. "No games yet. Pick one in the Lobby." |

## 6. Instrument Settings (M8)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Adapter mode | Toggle | Real or mock instrument | Enum: `Multi-UE simulator` / `Mock (simulated)` |
| Host address | Text input | Simulator address | IPv4/hostname + port; required in real mode |
| Interface | Select | Connection type | Enum: Ethernet/SCPI, GPIB, USB |
| Test connection | Button + result | Checks reachability | Result: OK / error text ≤ 80 chars |
| Max UEs available | Mono (read-only) | Licence/hardware limit | Integer ≤ 60,000; caps the Lobby |
| Datalake source | Text input | Endpoint for field captures | URL; read-only access only |

---

## Gaps between the PRD and the content model

Four gaps (G1–G4) block MVP content. The rest can be settled during the build.

| # | Gap | Where it bites | Fix in PRD | Blocks MVP? |
| --- | --- | --- | --- | --- |
| G1 | **Failure rules are undefined.** The PRD says "crash, KPI breach, assert" but gives no per-game thresholds. | Stop conditions, threshold markers, Failure reason | Add a per-game failure table (KPI, threshold, duration) | Yes |
| G2 | **The score and reward formula is undefined.** No weights are set for reject, latency or drop. | Reward, Reward breakdown, Total score | Define the reward function and score scale | Yes |
| G3 | **Difficulty levels have no parameters.** | Difficulty hint, Cohort mix | Map each level to UE count, burst rate and move set | Yes |
| G4 | **The datalake schema is unknown.** This is already an open question in the PRD. | Source captures, Datalake source | Confirm capture ID, metadata and access method | Yes |
| G5 | **The PRD has no DUT identity**, but history and replay need one. | DUT label (added to Lobby) | Add a DUT label/build field to M1 | No (added here) |
| G6 | **Mock vs real mode isn't visible to users.** The PRD ships a mock adapter but has no rule for showing it. | Adapter status badge | Require a `SIMULATED` badge whenever mock is active | No (added here) |
| G7 | **Copy tone isn't specified.** The Design Standard covers visuals only. | All text fields | Adopt the two-register rule (Arcade / Instrument) above | No |
| G8 | **KPI units and definitions are not stated** (e.g. latency p50 vs p95). | Diagnostics strip, KPI snapshot | Add a KPI glossary with units | No |
| G9 | **There are no users or auth**, so Run History can't show who ran a game. | Run list | Decide: single-user lab tool for MVP, or add a user field | No |
| G10 | **The log capture window isn't set.** | Log capture | Confirm N (30 s assumed) | No |
| G11 | **Hot-add (S1) is core to the vibe but out of MVP scope**, so there is no "CONTINUE?" content here. | Live Arena | Keep as Should-Have; reserve space in the arena layout | No |
| G12 | **Platform errors aren't covered.** The PRD only defines user Stop and DUT failure. | Outcome enum (`PLATFORM ERROR` added) | Add a third stop condition so tool faults are never blamed on the DUT | No (added here) |
