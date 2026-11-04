# Test Arena — Content Model (Must-Have / MVP)

*Companion to the Anritsu AI-RAN Test Arena PRD v1.2 · 2 Nov 2026*

> **Change log**
> - **4 Nov 2026 (v1.5):** Workflow library v3 (schema 3.0.0): NAS messages are library steps with their own decode (NAS library: TS 24.501 clause, IE table, absent IEs, security header, protected PDU, TS 24.301 equivalent); RRC steps name the NAS they carry; Settings lists the 5GS↔EPS timer and interworking tables; branch sets grow to L1 19, L2 14, L3 12, L5 15.
> - **3 Nov 2026 (v1.4):** Workflow library v2 (schema 2.0.0): message flows include PHY/MAC steps with a Signalling / All layers toggle; trace rows carry SFN/slot; the inspector decodes DCI, PDSCH/PUSCH/PUCCH/PRACH, MAC PDUs, RRC ASN.1 (UPER hex, IE tree, absent IEs), NAS PDUs and SIP; a PHY · MAC filter; IMSI masked in hex; 10 verification areas.
> - **2 Nov 2026 (v1.3):** Added the signalling trace (Arena tab and Game Over section): ladder-diagram rows, IE inspector and DUT response verdicts.
> - **2 Nov 2026 (v1.2):** Ladder content now comes from the signalling workflow library (M11): workflow IDs on every branch, a message-flow view on the Briefing, message narration in the Arena, workflow steps in failure reports, Level 5 (IMS Voice & Video), a Workflow library panel in Settings, three emulated cells, and gaps G17–G20.
> - **2 Nov 2026:** Added Procedure Ladder content (M10): Lobby mode and level cards, game model and branch set on the Briefing, per-game KPI sets, branch coverage in the Arena, `LEVEL CLEAR` outcome, and gaps G13–G16.
> - **29 Oct 2026:** First release.

## Page order

The MVP needs six screens plus a persistent shell. Each maps to Must-Haves M1–M11 and to nothing else.

0. **App shell**: persistent top bar on all screens
1. **Game Lobby**: choose mode, target, game or ladder level, and difficulty (M1, M10)
2. **Testcase Briefing**: review the generated testcase and its message flows before Play (M2, M10, M11)
3. **Live Arena**: cohort, DUT, agent loop and diagnostics strip (M3–M7, M10)
4. **Game Over**: failure, stop or level-clear report and log capture (M3, M7, M10)
5. **Run History**: saved games and replay (M9)
6. **Instrument Settings**: adapter connection and workflow library (M8, M11)

### Shared rules for every field

- **Tone has two registers.**
  - *Arcade* (Press Start 2P, uppercase, ≤ 3 words) is used only for titles, the score and game-state banners.
  - *Instrument* (Inter / JetBrains Mono, sentence case, plain and exact) is used for everything else.
- **Numbers** always carry a unit. KPIs use JetBrains Mono with fixed decimals.
- **Timestamps** are stored as ISO 8601. They display as `HH:MM:SS` in the arena and `DD MMM YYYY HH:MM` in history, in local time.
- **Identifiers** (IMSI/SUPI) are masked to the last 4 digits wherever they are shown.
- **Game names.** Stress games use their name ("Signalling Storm"). Ladder levels use `L<n> <Procedure>` ("L1 Registration", "L5 IMS Call") wherever space is tight, and "Level 1 · Registration" / "Level 5 · IMS Voice & Video" elsewhere.
- **Workflow references.** Written as the workflow ID plus case, variant or cause, separated by " · ": `WF-07 · #22`, `WF-08a`, `WF-12c`. Shown in JetBrains Mono. Never invent an ID; tester-defined branches show `—`.
- **Message names** are copied exactly from the library (`REGISTRATION REQUEST`, `RRCSetupRequest`, `486 Busy Here`), never reworded. Direction is shown as UL ↑, DL ↓, SS ⚑ or UE ●.

---

## 0. App shell

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Product name | Text | "Test Arena" wordmark + Anritsu logo | Fixed; brand green only here |
| Nav items | Text ×4 | Lobby · Arena · History · Settings | 1 word each; Arena disabled when no game runs |
| Adapter status | Badge | Connection to instrument | Enum: `LIVE` / `SIMULATED` / `OFFLINE`; always visible |
| Active game indicator | Badge | Game ID + elapsed time when a game runs | `PLAY 042 · 01:23:45`; links to Arena |

## 1. Game Lobby (M1, M10)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Screen title | Arcade | "SELECT GAME" | ≤ 3 words, uppercase |
| Mode | Toggle | Which kind of game | Enum: `Stress games`, `Procedure Ladder`; default Stress games |
| Target type | Toggle | What is being stressed | Enum: `gNB / AI-RAN`, `Modem`; required |
| DUT label | Text input | Name + build of the device, e.g. "gNB-X sw 4.2.1" | Required, ≤ 40 chars; saved with every run; ladder unlocks are tracked per DUT label |
| Feature under test (Stress mode) | Card picker | One card per stress game | Enum (4): Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze; single select |
| Ladder level (Ladder mode) | Card picker, ordered 1→5 | One card per level, joined by a connector line | Enum (5): L1 Registration, L2 Data Call, L3 Handover & Mobility, L4 Throughput, L5 IMS Voice & Video; single select; locked levels not selectable |
| Card · description | Text | What the game does to the DUT | ≤ 90 chars, instrument tone, starts with a verb ("Floods RACH with…", "Breaks the registration tree at…") |
| Card · KPIs attacked | Tags | KPIs this game pressures | 2–3 tags from that game's own KPI set |
| Level card · game model | Text | The level's game-theory model | ≤ 30 chars: "Sequential game", "Coordination game", "Stackelberg game", "Congestion game", "Bargaining game" |
| Level card · workflows | Mono tags | Library workflows the level plays | 1–4 IDs, e.g. `WF-05` `WF-06` `WF-11` `WF-12`; L4 shows `WF-01` |
| Level card · lock state | Badge + icon | Whether the level can be played | Enum: `LOCKED` / `OPEN` / `CLEARED`; locked cards show "Clear L<n−1> to unlock" (≤ 30 chars) |
| Difficulty | Segmented control | Stress intensity | Enum: Easy / Normal / Hard / Brutal; default Normal |
| Difficulty hint | Text | What the level means in numbers | ≤ 60 chars, e.g. "10K UEs · 2× burst rate" or "Baseline + 3 branches · 100 UEs" |
| Primary action | Button | Generates the testcase | Label "GENERATE"; disabled until target, DUT label and a game or open level are set |

## 2. Testcase Briefing (M2, M10)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Testcase ID | Mono text | Unique ID for replay | Format `TC-YYYYMMDD-NNNN`; read-only |
| Summary | Text | One-line description of the generated sequence | ≤ 120 chars, instrument tone |
| Game model | Text | The game-theory model the agent uses | ≤ 40 chars, e.g. "Stackelberg · leader–follower" (stress games) or "Sequential game · subgame perfection" (L1) |
| Model explainer | Text | What the model means for this test | ≤ 140 chars, instrument tone; one per game, from the PRD catalogue |
| Branch set (Ladder only) | Ordered list | Branches the agent must cover to clear the level | 4–18 items; name ≤ 40 chars; each row shows its workflow reference (`WF-07 · #22`, or `—` if tester-defined); count depends on difficulty; one row is selected (baseline by default) |
| Message flow (Ladder only) | Step list + toggle | The selected branch's workflow steps | 5–155 steps (initial registration 129, VoNR 141); toggle "Signalling (n)" (default: RRC, NAS, SIP, UE events, test-system actions) / "All layers (n)" (adds DCI, PDSCH/PUSCH, HARQ-ACK, SR, PRACH, gNB decode); per step: seq, direction, layer, message (≤ 56 chars), container for NAS steps (≤ 48 chars, e.g. "RRCSetupComplete.dedicatedNAS-Message", "DL NAS TRANSPORT Payload container in RRCReconfiguration"), note, timers (`T3510 ▶` start, `■` stop, `✕` expired, `…` running); the injection step is marked "Injection"; tags "Adapted" (value differs from the library message), "Response matrix", "Composed" (flow joined from library flows, with a "from WF-xx" rule at each join; only L5 "RLF during active call") |
| Expected behaviour (Ladder only) | Text | What the DUT must do for the selected branch to pass | ≤ 200 chars, from the library in this order: matrix row `expectedUE`; the injection step's `expect`; the next UE-event step (timer expiry first); the next UL NAS summary; `passCriteria` |
| Spec baseline | Text | Specs the workflow library is baselined on | Every TS number in `specBaseline`, e.g. "TS 38.331 · 38.508-1 · 38.212 · 38.321 · 38.213 · 38.214 · 38.211 · 24.501 · 24.229 · 34.229-1" |
| Defaults to verify | Badge + count | Configuration areas in the library's `defaults.verify` | Integer; warning badge when > 0, e.g. "10 defaults to verify"; tooltip lists the first 3 |
| Clear rule (Ladder only) | Text | What clears the level | ≤ 80 chars, e.g. "Pass all 7 branches with no DUT failure, min 05:00" |
| Cohort mix | 3 numbers + bar | UE counts per slice | URLLC / eMBB / mMTC; integers; total ≤ 60,000 |
| Source captures | List | Datalake captures used to seed the testcase | 1–5 items: capture ID, location, date; no subscriber IDs |
| Stop conditions | List | What ends the game | Always "User Stop" + the game's failure rules (see G1); ladder levels add "Level clear" |
| Generation time | Mono number | Proof of the < 2 s target | Seconds, 2 decimals |
| Actions | Buttons | Start or go back | "PLAY" (primary), "Regenerate", "Back" |

## 3. Live Arena (M3–M7, M10)

The screen has four zones. All live values update at 10–20 Hz.

### 3a. Arena header

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Game ID | Arcade | e.g. "PLAY 042" | Zero-padded, 3+ digits |
| Game name + difficulty | Text | e.g. "Signalling Storm · Hard" or "Level 1 · Registration · Hard" | From Lobby; read-only |
| DUT label | Text | Device being tested | From Lobby; ≤ 40 chars |
| Elapsed time | Mono | Time since Play | `HH:MM:SS`, counts up, no upper limit |
| Branch coverage (Ladder only) | Mono + bar | Branches passed so far | `n / N` and %, integer; bar fills as branches pass |
| Workflow step (Ladder only) | Mono | Position in the current branch's workflow | `WF-11 · step 4 / 10` |
| Game state | Badge | Current state | Enum: `RUNNING` / `ESCALATING` / `STOPPED` / `DUT DOWN` / `CLEARED` |
| Stop control | Button | Ends the game | Label "STOP"; one confirm: "Stop game and save run?" |

### 3b. Cohort arena (canvas)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Cohort tiles | Sprite grid | Aggregated UE groups, not single UEs | ≤ 48 tiles; each = slice + count; shape differs per slice |
| Slice counts | Mono numbers | Active UEs per slice | URLLC / eMBB / mMTC; integers |
| Current move | Text | Stress move or ladder branch the leader is playing | Enum from game; ≤ 30 chars, e.g. "RACH burst ×4" or "REJECT #22 + T3346", with its workflow reference beneath in mono |
| DUT status | Icon + text | gNB tower or modem sprite with state | Enum: `Serving` / `Barring` / `Rejecting` / `Down` |
| Cells (L3 only) | Sprite ×3 | The library's three NR cells, with the UE cohort moving between them | Labels "Cell 1", "Cell 2" (TAC 000001) and "Cell 3" (TAC 000002); Cell 3 is used by WF-05c and WF-06b |
| Legend | Fixed | URLLC · eMBB · mMTC · NETWORK | Static; colour + shape |

### 3c. Agent loop panel

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| State | 3 mono values | The game's three state inputs | Stress games: RACH attempts, PRB utilisation, rejects (/s, %, /s). Ladder: defined per level in `DATA.ladder` (see table below) |
| Reward | Arcade number | Current reward from QoS | Signed, 2 decimals, e.g. "+12.75"; yellow only |
| Reward breakdown | Text | Components driving reward | 3 components per game (stress: Reject · Latency · Drop), each signed, 2 decimals |
| Action | Text | Agent's last chosen move or branch | ≤ 30 chars; same vocabulary as "Current move" |
| Agent | Text | Algorithm + game model | e.g. "DQN v0.3 · Stackelberg" or "DQN v0.3 · Sequential"; read-only |
| Total score | Arcade number | Cumulative score this game | Integer, counts up |

**Agent state inputs per ladder level**

| Level | State values (units) | Reward components |
| --- | --- | --- |
| L1 Registration | Attempts (/s), Rejects (/s), Timeouts (/s) | Wrong recovery · Latency · Hang |
| L2 Data Call | Session requests (/s), Fallbacks (/s), Session rejects (/s) | Miscoordination · Latency · Drop |
| L3 Handover | HO attempts (/min), Ping-pongs (/min), RLF (/min) | HO failure · Interruption · Drop |
| L4 Throughput | Active UEs, PRB utilisation (%), BLER (%) | Throughput gap · Unfairness · BLER |
| L5 IMS Voice & Video | Call attempts (/s), SIP errors (/s), QoS flow failures (/min) | Failed call · Setup time · Drop |

### 3c-bis. Signalling trace (tab beside the agent loop)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Tabs | Tab pair | "Agent loop" / "Signalling trace" | One selected |
| Summary | Text | Counts for this game | "n messages (n PHY/MAC) · n DUT responses · n as expected · n late · n wrong or missing" |
| Filters | Toggles | "PHY · MAC" (on by default; off hides DCI, PDSCH/PUSCH, HARQ-ACK, SR, PRACH and gNB decode rows), "DUT responses only", "Live" (follow newest) | Selecting a row turns Live off; in Live the inspector updates at most about twice a second |
| Lane header | Labels | UE · gNB · 5GC · IMS, DUT lane emphasised | Caption names the DUT, e.g. "gNB is the DUT · UE simulated · 5GC and IMS emulated" |
| Trace row | Row | One message | Time `+MM:SS.s`; arrow between lanes; message name (exact library spelling); meta "layer · channel · in container · SFN s.slot · workflow #step"; PHY/MAC rows in caption size with a dotted arrow; gNB decode shown as a square on the gNB lane; sender ("UE sim", "5GC emu", "gNB sim", "Test system") or DUT verdict ("DUT 12.4 ms", "DUT late", "DUT sent RRCReject", "DUT no response") |
| Inspector · details | Key–value | Direction (DUT tagged), layer and channel, container, time (SFN · slot · ms), workflow, step and library source (e.g. `WF-07a#24`), move, timers, representative UE | 5G-S-TMSI masked to last 4 |
| Inspector · DUT response | Key–value | Verdict, expected, actual, latency (library SFN/slot gap × load; late = 3×, with the expected value), "must do" from the workflow | When the DUT sent RRCReject instead of RRCSetup, the library's RRCReject ASN.1 is shown |
| Inspector · DCI | Key–value + table | PDCCH: RNTI type/value, search space, CORESET, aggregation level, CCE, payload bits + CRC, payload hex; one row per field: name · bits · value · meaning | 0-bit fields listed underneath |
| Inspector · PHY | Key–value | PDSCH / PUSCH (PRBs, MCS, TBS, RV, HARQ process, NDI, symbols, K2, DMRS), PUCCH (format, resource, K1, HARQ process, codebook, counter DAI, HARQ-ACK bits, or SR fields), PRACH, SS/PBCH | Values as in the library |
| Inspector · MAC PDU | Hex + tree | MAC PDU hex and its subPDUs (subheaders, RAR with 27-bit UL grant, CCCH SDU, BSR) | Hex grouped in bytes with Copy |
| Inspector · RRC ASN.1 | Hex + tree | ASN.1 type, UPER length, `messageLibrary` reference, UPER hex, decoded IE tree; single-child envelopes folded into one path (message › c1 › rrcSetup › …); "Absent (OPTIONAL)" and "Absent (extensions)" lines at every level; contained IEs (CellGroupConfig, NAS) decoded below | Large sub-trees start collapsed |
| Inspector · NAS PDU | Key–value + hex + table | Message with TS 24.501 clause, procedure, decoded summary, TS 24.301 equivalent, carried-in container, `nasLibrary` reference and length; plain PDU hex; IE table (IE · IEI · length · value tree); "Absent (OPTIONAL)" list; contained messages; NAS security block (security header type, MAC placeholder, sequence number, NAS COUNT, ciphering) with the protected PDU hex as carried | A played cause-matrix row shows a "Cause matrix" tag, its hex, and a note that the IE table is the library's own message; adapted PDUs (#21) show an amber "Adapted" tag; IMSI/SUPI digits masked except the last 4 in hex, IE values and summaries |
| Inspector · Carries NAS | Text | On an RRC step that carries NAS: the NAS message names and where their row is | — |
| Inspector · SIP | Key–value + block | Headers and SDP lines, size in bytes | IMSI-based IMPU masked |
| Inspector · Annotation | Text | Step note, library note, "Adapted: …" (amber) | — |
| Buffer | — | Live trace length | Last 600 messages kept; last 260 rows rendered; 220 saved per run (trace kept for the 10 newest saved runs) |

### 3d. Diagnostics strip (always visible)

The strip always shows five KPIs. Stress games use the standard five below; each ladder level uses its own five.

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| RACH success | Mono + sparkline | Share of RACH attempts succeeding | %, 1 decimal, last 60 s |
| PRB utilisation | Mono + sparkline | Downlink + uplink load | %, 1 decimal |
| Rejects | Mono + sparkline | RRC / admission rejects | Per second, integer |
| Latency | Mono + sparkline | User-plane latency, p95 | ms, 1 decimal |
| Drops | Mono + sparkline | Dropped sessions | Per second, integer |
| Threshold marker | Line | Failure threshold per KPI | Turns red when crossed; values TBD (see G1) |
| Event log | Scrolling list | Last 20 events | `HH:MM:SS` + ≤ 80-char message; IMSI masked; ladder adds "Branch passed: …" events and narrates workflow steps, e.g. "↑ UL 5GMM REGISTRATION REQUEST · T3510 ▶" |

**Ladder KPI sets** (same format rules: mono + 60 s sparkline + threshold marker)

| Level | KPI 1 | KPI 2 | KPI 3 | KPI 4 | KPI 5 |
| --- | --- | --- | --- | --- | --- |
| L1 Registration | Registration success (%, 1 dp) | Registration latency (ms, 1 dp) | Retries (/s, int) | Rejects (/s, int) | Timeouts (/s, int) |
| L2 Data Call | Session setup success (%, 1 dp) | Setup latency (ms, 1 dp) | Fallbacks (/s, int) | Session rejects (/s, int) | Session drops (/s, int) |
| L3 Handover | HO success (%, 1 dp) | Interruption (ms, 1 dp) | Ping-pong rate (%, 1 dp) | RLF (/min, int) | Session drops (/s, int) |
| L4 Throughput | Cell throughput (Mbps, 1 dp) | Per-UE p5 (Mbps, 1 dp) | Fairness (Jain, 0–1, 2 dp) | BLER (%, 1 dp) | PRB utilisation (%, 1 dp) |
| L5 IMS Voice & Video | Call setup success (%, 1 dp) | Call setup time (ms, 1 dp) | QoS flow failures (/min, int) | SIP errors (/s, int) | Call drops (/min, int) |

## 4. Game Over (M3, M7, M10)

This screen appears when the DUT fails, the user stops the game, or a ladder level is cleared. Only the banner and actions differ between the cases; the rest of the report is the same.

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Banner | Arcade | Outcome | Enum: "DUT DOWN" (red) / "GAME STOPPED" (green) / "LEVEL CLEAR" (green) |
| Failure reason | Text | What broke | ≤ 100 chars, instrument tone, names the KPI or event, e.g. "gNB crash after RRC setup storm". On a ladder level it names the workflow step, e.g. "WF-11 step 4: no RRCReestablishmentRequest before T311". For a clear: "All 7 branches passed". |
| Time to failure | Mono | Elapsed time at end | `HH:MM:SS` |
| Final score | Arcade number | Total score | Integer |
| Branch results (Ladder only) | Table | Each branch with pass / fail / not reached | Branch name + workflow reference + result + time first passed; a failed row adds the step reached and the expected behaviour (≤ 160 chars) |
| Unlock notice (Ladder clear only) | Text | What is now open | ≤ 60 chars, e.g. "Level 2 · Data Call unlocked for gNB-X sw 4.2.1" |
| Triggering moves | Ordered list | Last 5 moves before the end | Move + timestamp, oldest first |
| KPI snapshot | Table | All 5 KPIs at the end vs threshold | Same KPI set and units as that game's Diagnostics strip |
| Signalling trace | Section | Last 160 messages of the run, same rows and inspector as the Arena | Auto-selects the last wrong or missing DUT response, else the last DUT response; "DUT responses only" filter |
| Log capture | File link | Logs + game state for the last N seconds | Default N = 30 s; download as .zip; IMSI masked |
| Replay seed | Mono | Seed + agent version for exact replay | Copyable |
| Actions | Buttons | Next step | Default: "REPLAY", "New game", "View in History". After a level clear: "NEXT LEVEL" (primary), "Replay", "View in History"; after clearing L5: "New game" replaces "NEXT LEVEL". |

## 5. Run History (M9)

| Field | Format | Description | Constraints |
| --- | --- | --- | --- |
| Run list · columns | Table | One row per game | Date, Game ID, DUT label, Game, Difficulty, Duration, Outcome, Score. Ladder runs show Game as "L1 Registration". |
| Outcome | Badge | How the game ended | Enum: `DUT DOWN` / `STOPPED` / `LEVEL CLEAR` / `PLATFORM ERROR` |
| Filters | Controls | Narrow the list | DUT label, Game (stress games + five ladder levels), Outcome, date range |
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
| Emulated cells | Mono (read-only) | Cells available for Level 3 | Integer; 3 required for L3 (two in one TA, one in a second TA) |
| Workflow library | Panel | The loaded `5g_signalling_workflows.json` | Schema name + version (`test-arena.signalling-workflows` 3.0.0), technology, generated date, contents ("17 workflows · 29 cases · 1927 steps · 107 RRC messages (UPER) · 92 NAS PDUs"), carrier (band · duplex · SCS · PRBs · TDD pattern), spec baseline, "10 defaults to verify", workflow list (ID · name · case count · category), expandable "NAS timers · 5GS ↔ EPS (14)" table (5GS timer, default, start, stop, TS 24.301 timer), "TS 24.301 interworking (10)" list with the 4G-GUTI → 5G-GUTI mapping, and "Not covered yet (6)" |
| Datalake source | Text input | Endpoint for field captures | URL; read-only access only |
| Ladder unlocks | Toggle | How ladder levels open | Enum: `Progressive` (default) / `All unlocked (demo)` |

---

## Gaps between the PRD and the content model

Four gaps (G1–G4) and the new G13 block MVP content. The rest can be settled during the build.

| # | Gap | Where it bites | Fix in PRD | Blocks MVP? |
| --- | --- | --- | --- | --- |
| G1 | **Failure rules are undefined.** The PRD says "crash, KPI breach, assert" but gives no per-game thresholds. This now includes the five ladder KPI sets. | Stop conditions, threshold markers, Failure reason | Add a per-game failure table (KPI, threshold, duration) | Yes |
| G2 | **The score and reward formula is undefined.** No weights are set for reject, latency or drop, or for the ladder reward components. | Reward, Reward breakdown, Total score | Define the reward function and score scale per game | Yes |
| G3 | **Difficulty levels have no parameters.** For ladder levels this includes the branch-set size per difficulty. | Difficulty hint, Cohort mix, Branch set | Map each level to UE count, burst rate, move set and branch count | Yes |
| G4 | **The datalake schema is unknown.** This is already an open question in the PRD. | Source captures, Datalake source | Confirm capture ID, metadata and access method | Yes |
| G5 | **The PRD has no DUT identity**, but history and replay need one. | DUT label (added to Lobby) | Add a DUT label/build field to M1 | No (added here) |
| G6 | **Mock vs real mode isn't visible to users.** The PRD ships a mock adapter but has no rule for showing it. | Adapter status badge | Require a `SIMULATED` badge whenever mock is active | No (added here) |
| G7 | **Copy tone isn't specified.** The Design Standard covers visuals only. | All text fields | Adopt the two-register rule (Arcade / Instrument) above | No |
| G8 | **KPI units and definitions are not stated** (e.g. latency p50 vs p95). | Diagnostics strip, KPI snapshot | Add a KPI glossary with units, covering the ladder KPIs (e.g. Jain's fairness index) | No |
| G9 | **There are no users or auth**, so Run History can't show who ran a game. | Run list | Decide: single-user lab tool for MVP, or add a user field | No |
| G10 | **The log capture window isn't set.** | Log capture | Confirm N (30 s assumed) | No |
| G11 | **Hot-add (S1) is core to the vibe but out of MVP scope**, so there is no "CONTINUE?" content here. | Live Arena | Keep as Should-Have; reserve space in the arena layout | No |
| G12 | **Platform errors aren't covered.** The PRD only defines user Stop and DUT failure. | Outcome enum (`PLATFORM ERROR` added) | Add a third stop condition so tool faults are never blamed on the DUT | No (added here) |
| G13 | **The level-clear rule needs numbers.** The PRD sets 100% branch coverage but leaves the minimum run time open. | Clear rule, Branch coverage, `CLEARED` state | Set the minimum run time per level and difficulty | Yes |
| G14 | **Unlock persistence is undefined.** Should a clear carry over to a new DUT build, and does it expire? | Level card lock state, Unlock notice | Decide unlock scope (assumed: per exact DUT label, no expiry) | No (assumption used) |
| G15 | **Level 3 needs multi-cell emulation**, which the PRD lists as an open question. | Cells (L3), Emulated cells | Confirm simulator support for ≥ 2 cells | No (mock emulates 2) |
| G16 | **Tournament mode (S8) is out of MVP scope**, so there is no chained-run content here. | Lobby, Game Over | Keep as Should-Have; "NEXT LEVEL" is the manual equivalent for MVP | No |
| G17 | **Workflow coverage gaps.** The library has no #21 synch-failure encoding, no full flows for SIP 480/503/408 (only a matrix row), no video response matrix, no UE-requested PDU session modification and no throughput flows (L4); `notCoveredYet` lists 6 more areas, and the LTE RRC/L1 side of EPS fallback waits for the LTE stage. | L1 branch 13, L5 branches 4–6, L2, L4 | Add the flows; until then branch 13 adapts WF-07b and branches 4–6 swap the 486 ("Adapted"), three video branches are dropped and L4 shows `—` | No (marked in UI) |
| G18 | **10 configuration areas are listed in `defaults.verify`** (band/ARFCN, TDD pattern, CORESET#0, PRACH, SR/HARQ timing, TDRA, RLC/PDCP timers, LCH priorities, SIB1 timers, A3/cell power). They depend on the TS 38.508-1 release and PIXIT/ICS. | Defaults to verify badge | Confirm the 38.508-1 baseline per customer | No (warning shown) |
| G19 | **Library ownership and versioning are undefined.** | Settings · Workflow library | Name an owner, a release process and a compatibility rule for `schemaVersion` | No |
| G20 | **Cohort fan-out of single-UE workflows is unspecified** (stagger, identities, per-UE judging, log volume). | Arena KPIs, Event log | Define the fan-out rule; the static build narrates one representative UE | No (assumption used) |
