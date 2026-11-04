# Anritsu AI-RAN Test Arena — Product Requirements Document

*v1.7 · 4 Nov 2026 · Marsh Dsouza*

> **Change log**
> - **v1.7 (4 Nov 2026):** Added two **play modes** for every game and both targets (gNB / AI-RAN and modem). **Manual:** the engineer picks by click, chat command or voice dictation. **Autopilot:** the application runs a campaign, choosing each next game from what it has learned breaks the DUT. New features M12 and M13, a new §3 section, new §4.7 (Autopilot as a multi-armed bandit), constraints, success criteria and open questions. Also fixed two stale items: workflow coverage is now 17 of 17, and the L1 success-workflow question is closed.
> - **v1.6 (4 Nov 2026):** New **§4 Game Theory Guide** for readers new to game theory: core terms, the learning engine shared by all games, the Stackelberg model behind the stress games, the five ladder models (sequential, coordination, Stackelberg with a bad leader, congestion + Nash bargaining + Nature, Rubinstein bargaining), and how a game becomes a verdict, with nine diagrams. Later sections renumbered (Technical Constraints §5, Design Standard §6, Success Criteria §7). Ladder catalogue workflow column updated for library v3.
> - **v1.5 (4 Nov 2026):** Moved to **workflow library v3** (schema 3.0.0): every NAS message is its own step, decoded IE by IE from a new NAS library (92 PDUs, TS 24.501 clause, security header, protected PDU, TS 24.301 equivalent); five new workflows (WF-13 initial registration + PDU sessions, WF-14 de-registration, WF-15 periodic registration, WF-16 PDU session release and 5GSM/5GMM errors, WF-17 5GS–EPS interworking); 5GS↔EPS timer and interworking tables. Branch sets updated: L1 19, L2 14, L3 12, L5 15. L1's baseline is now a real library flow.
> - **v1.4 (3 Nov 2026):** Moved to **workflow library v2** (schema 2.0.0): every flow now carries PHY and MAC steps (DCI grants, PRACH/RAR, Msg3/Msg4, SR, HARQ-ACK/NACK and retransmissions, MAC PDUs) with SFN/slot timing; RRC messages are full TS 38.331 Rel-18 ASN.1 (UPER-encoded, every absent IE listed) in a shared message library; NAS PDUs are encoded and ride inside their RRC containers. Failure workflows are now organised as cases. Branch mapping, stress-move slices, the trace and the verification list updated to match.
> - **v1.3 (2 Nov 2026):** Added the **signalling trace** to diagnostics (M7): every message the cohort sends and every DUT response, decoded to IE level from the workflow library, with a verdict on each DUT response. Stress moves now replay workflow slices, with random access (Msg1/Msg2) ahead of RACH moves.
> - **v1.2 (2 Nov 2026):** Games are now driven by the **5G signalling workflow library** (`5g_signalling_workflows.json`, 12 workflows, 3GPP-baselined). Every ladder branch and stress move traces to a workflow ID. Added Level 5 (IMS Voice & Video), new feature M11, workflow mapping (§3), three-cell emulation, and new success criteria and open questions.
> - **v1.1 (2 Nov 2026):** Added the **Procedure Ladder**, four game-theory levels for core mobile procedures (Registration → Data Call → Handover → Throughput). New features M10, S8 and N8; new game catalogue (§3); new success criteria and open questions (§6).
> - **v1.0 (29 Oct 2026):** First release.

## 1. Vision

Test Arena turns AI-RAN and modem testing into an endless, adversarial game. An AI-trained UE cohort plays against the device under test (DUT) until the DUT breaks or the engineer calls time.

**The problem.** Today's protocol and RF test suites are scripted, static and written by hand. They prove conformance but rarely find the edge cases a live 6G network will throw at a gNB or modem: signalling storms, retry loops, and mixed URLLC/eMBB/mMTC contention. Field data that could reveal those cases sits unused in the datalake.

**The product.** Test Arena is a web application that sits on top of Anritsu's Multi-UE simulator (up to 60K UEs). An engineer picks a feature and a difficulty level, and the platform generates testcases from live field captures and launches them as a "game". A Game Theory agent (DQN, Stackelberg leader–follower) learns which sequences push the DUT hardest, scores the DUT's QoS response, and escalates.

**Two kinds of game.**

| Mode | What it is | Ends when |
| --- | --- | --- |
| **Stress games** | Four endless load games (Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze). The agent escalates until something breaks. | User Stop or DUT failure |
| **Procedure Ladder** *(v1.1, extended in v1.2)* | Five levels that test the core mobile procedures in the order a UE uses them: **Registration → Data Call → Handover & Mobility → Throughput → IMS Voice & Video**. Each level is modelled as a different kind of game, and a level unlocks only when the one before it is cleared. | Level clear, User Stop or DUT failure |

**Two ways to choose.** In **Manual** mode the engineer picks the game by clicking, typing a chat command or speaking. In **Autopilot** mode the application picks, running a campaign of games chosen from what it has learned breaks this DUT. Both modes cover gNB / AI-RAN and modem targets (§3, Play modes).

The ladder is the "basics first" path. A DUT that can't register correctly has no business being stress-tested, so engineers (and new users learning the tool) start at Level 1 and climb.

**Built on real signalling.** Moves are not invented. Each one replays a 3GPP signalling workflow from the **workflow library** (`5g_signalling_workflows.json`): the message sequence, the information elements, the timers and the expected UE behaviour, baselined on TS 38.508-1, 38.331, 24.501, 24.229 and 38.523-1. The game decides *which* workflow, failure case or reject cause to play next; the workflow decides *exactly what goes on the air* and what "correct" looks like. See §3, Signalling workflow library.

**The game loop.**

1. **Leader (UE cohort)** chooses an attack move: RACH burst, access-retry loop, small-packet flood, or GBR throughput demand. On the ladder, moves are the level's branch set (§3, Procedure Ladder catalogue). Every move is a workflow from the library (or a failure case, variant or reject cause within one), replayed across the cohort.
2. **Follower (DUT)** responds with barring, admission, scheduling or rejects.
3. **State** (RACH, PRB use, rejects) and **Reward** (reject, latency, drop) feed the agent.
4. **Agent** computes its best response and escalates. Stress games never end on their own; ladder levels end when the level is cleared.

New to game theory? §4 explains each game in plain terms, with diagrams.

```mermaid
flowchart LR
    DL[Field datalake<br/>live captures] -- seeds --> L[UE cohort · leader<br/>Multi-UE simulator, up to 60K]
    WF[Signalling workflow library<br/>5g_signalling_workflows.json] -- message sequences + pass criteria --> L
    L -- stress move --> D[DUT · follower<br/>gNB / AI-RAN or modem]
    D -- QoS response --> SR[State + Reward<br/>RACH, PRB, rejects, latency]
    SR -- state + reward --> A[Game Theory agent<br/>DQN · Stackelberg]
    A -- best response --> L
```

Field captures seed the cohort and the workflow library supplies the moves. Every DUT response becomes the agent's next state and reward, so a stress game only ends on Stop or failure.

**Why it matters.** Anritsu moves from selling test equipment to selling *discovered failure modes*. R&D teams find breaking points in days rather than in field trials. Every run makes the agent and the testcase library smarter.

**One-line pitch:** *Don't write tests. Play the network until it breaks.*

## 2. User Personas

Three people use Test Arena daily. Priya is the primary design target.

### Priya — gNB / AI-RAN Protocol Test Engineer (primary)

- **Works at:** a Tier-1 RAN vendor's 6G lab, where she owns L2/L3 and RRM verification for an AI-RAN scheduler.
- **Goal:** find where the AI scheduler breaks under mixed URLLC/eMBB/mMTC load, before the operator does.
- **Pain today:** she spends 60% of her week writing and maintaining TTCN/Python scripts that only test what she already thought of.
- **Needs from Test Arena:** to pick "RACH storm, Hard", hit Play and leave it running overnight, then come back to a ranked list of failures with full logs. On a new software drop she first runs the Procedure Ladder to confirm the basics still hold. When she knows what she wants she types or says it ("Signalling Storm, Hard"); before leaving for the night she starts an Autopilot campaign.
- **Success for her:** "It found a bug I would never have scripted."

### Kenji — Modem Firmware & RF Engineer

- **Works at:** a chipset/modem maker, validating 5G-Advanced/6G modem firmware on the RF and protocol side.
- **Goal:** stress the modem's protocol stack and RF front-end against realistic, hostile network behaviour.
- **Pain today:** network emulator scenarios are idealised, and real-network quirks only appear in expensive drive tests.
- **Needs from Test Arena:** to flip the game so the *network* becomes the adversary, replaying field-captured operator behaviour (e.g. a specific MNO's configuration) against his modem. The ladder's off-path branches (reject causes, timer expiry, RRC reject, handover failure, IMS SIP errors) are exactly the cases his scripts miss, and each one traces to a 3GPP workflow and clause he already knows.
- **Play modes:** at the bench with both hands on the RF set-up, he dictates "L1, only REJECT #22, modem M3 fw 2.0.1" instead of clicking. Autopilot keeps a separate memory of what breaks each modem firmware.
- **Success for him:** reproducing a field-reported crash in the lab within an hour.

### Sofia — Test Automation Lead / Programme Manager

- **Works at:** a mobile operator's or vendor's integration lab, where she runs a team of 8 engineers and the CI pipeline.
- **Goal:** hyper-automated, lights-out regression with clear evidence for release go/no-go decisions.
- **Pain today:** test results live in scattered logs, and no single score shows whether a software drop is stronger or weaker than the last.
- **Needs from Test Arena:** API/CI triggers, a leaderboard of DUT builds by resilience score, and exportable reports for management. "Highest ladder level cleared" is a simple first gate for every build.
- **Play modes:** Autopilot campaigns with a fixed budget run every nightly build, retest last night's failures first, and hand her a Campaign Report.
- **Success for her:** every nightly build gets a resilience score with no human in the loop.

## 3. Feature List

The scope assumes one developer working with AI coding agents. The Must-Haves form a demoable MVP in roughly 18–20 weeks (v1.0 estimate plus ~2 weeks for the Procedure Ladder, ~2 weeks for the workflow library and Level 5, and ~4 weeks for the Manual chat/voice and Autopilot play modes). Hardware integration starts as a simulated adapter, so the UI and game engine never wait on lab access.

### Must-Have (MVP)

| # | Feature | What it does | Maps to |
| --- | --- | --- | --- |
| M1 | Game Lobby | Pick play mode (Manual or Autopilot), target (gNB or modem) and DUT; in Manual, the game type (Stress games or Procedure Ladder), feature or ladder level, and difficulty (Easy / Normal / Hard / Brutal). | Vibe 1 |
| M2 | Testcase generator | Builds a testcase sequence from datalake field captures filtered by feature and difficulty, and shows the source captures used. For ladder levels it also lists the branch set and game model. | Vibe 1, live data |
| M3 | Endless game loop | Runs continuously and stops only on user Stop or a defined DUT failure (crash, KPI breach, assert). Ladder levels also end on Level Clear. | Vibe 2 |
| M4 | Game Theory engine v1 | Stackelberg leader–follower loop with a DQN agent choosing the next stress move from state + reward. On the ladder the agent chooses which branch to play next. | Vibe 6 |
| M5 | Built-in stress games (4) | Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze, each a preset game with tunable intensity. Each move scales up a library workflow (see the stress-game mapping in §3). | Vibe 6 |
| M6 | Live arena view | Cohort grid (URLLC / eMBB / mMTC), DUT status, State → Reward → Action panel, live QoS score. | Vibe 1 |
| M7 | Built-in diagnostics | Live KPI strip (five KPIs per game; see catalogue) and a **signalling trace**: a protocol-analyser view of every message the cohort sends and every DUT response, drawn as a UE ↔ gNB ↔ 5GC ↔ IMS ladder diagram, from PHY up: DCI grants, HARQ-ACK and SR, MAC PDUs, the full TS 38.331 ASN.1 of every RRC message (UPER hex and decoded IE tree, absent IEs listed), every NAS PDU decoded IE by IE (TS 24.501, with its security header and TS 24.301 equivalent) and SIP headers/SDP and a verdict on each DUT response (as expected, late, wrong message, no response). On failure, auto-captures the last N seconds of logs, game state and the trace, focused on the failing exchange. | Vibe 3 |
| M8 | Instrument adapter layer | One interface to the Multi-UE simulator; ships with a mock adapter for development and demos. | Platform |
| M9 | Run history | Every game saved with config, score timeline, failure point and replay of the move sequence. | Diagnostics |
| M10 | **Procedure Ladder (5 levels)** | Registration → Data Call → Handover & Mobility → Throughput → IMS Voice & Video. Each level has its own game model, branch set, five KPIs and clear rule. Levels unlock in order per DUT. | Vibe 1, Vibe 6 |
| M11 | **Signalling workflow library** | Loads `5g_signalling_workflows.json` (schema `test-arena.signalling-workflows` v3.0.0). Every ladder branch and stress move references a workflow ID (plus failure case, variant or cause). The Briefing shows each branch's message flow; the Arena narrates messages and timers as they play; reports name the workflow step where the DUT went wrong. | Vibe 1, Vibe 3 |
| M12 | **Manual play: click, chat or voice** | The engineer chooses the game, ladder level, branch subset, difficulty and target by clicking, typing a chat command or dictating by voice (push-to-talk). Commands are shown back as editable chips and never start a game without confirmation. Works for gNB / AI-RAN and modem targets. | Vibe 1, Vibe 4 |
| M13 | **Autopilot campaigns** | The application chooses each next game itself (stress game, ladder level or branch subset, and difficulty) from a per-target, per-DUT-family knowledge base of past runs, failures, move values and coverage gaps. It explains every pick, respects ladder order, runs within a time or failure budget, recovers the DUT after a failure and produces a campaign report. The engineer can skip, pin, take control or stop at any time. | Vibe 2, Vibe 4, Vibe 6 |

### Should-Have (v1.1)

| # | Feature | What it does | Maps to |
| --- | --- | --- | --- |
| S1 | Hot-add games | While a game runs, prompts the user to stack another feature ("DUT survived 30 min — add Mobility Storm?"). Games run concurrently. | Vibe 2 |
| S2 | Vendor & operator profiles | Import RAN vendor parameter sets and MNO network models (JSON/YAML). The agent adapts its strategy per profile. | Vibe 5 |
| S3 | Automation API + CI hooks | REST/WebSocket API to start, stop and query games; webhooks on failure; Jenkins/GitLab examples. | Vibe 4 |
| S4 | Failure root-cause hints | AI summary of the failure: the move sequence that triggered it, the KPIs that drifted first, and the likely layer. | Vibe 3 |
| S5 | Reverse mode (modem) | The network becomes the adversary for modem testing, replaying operator behaviour against the modem. | Persona Kenji |
| S6 | Resilience leaderboard | Ranks DUT builds by resilience score and highest ladder level cleared, and shows the trend across software drops. | Persona Sofia |
| S7 | Scheduled campaigns | Queue games overnight with a stop budget (time, failures). | Vibe 4 |
| S8 | **Tournament mode** | Chains all five ladder levels in one run with state carried across (registered UE → active PDU session → handover with data running → throughput after handover → VoNR call over the same bearers). Cross-workflow cases such as RLF during an active call (WF-03 + WF-12) are played here. Finds interaction bugs that pass in isolated levels. | Vibe 2 |

### Nice-to-Have (later)

| # | Feature | What it does |
| --- | --- | --- |
| N1 | Game designer | No-code editor to compose new stress games from move primitives. |
| N2 | Agent library | Swap algorithms (PPO, multi-agent, Nash equilibrium search) and compare which breaks the DUT fastest. |
| N3 | Testcase export | Export winning attack sequences as reusable conformance-style scripts. |
| N4 | Achievements & streaks | Badges such as "First Blood" (first DUT failure), "Iron DUT" (24 h with no failure) and "Ladder Master" (all five levels cleared on Brutal). |
| N5 | Digital Twin hook | Run games against a network digital twin before touching real hardware. |
| N6 | Multi-user arena | Shared live view, comments on failure moments, team spectator mode. |
| N7 | NTN / RF impairment games | Doppler, delay and fading stress for non-terrestrial and RF front-end testing. |
| N8 | **Mobility under load** (Ladder Level 6) | A repeated game where handovers happen during congestion, so the Level 3 and Level 4 rules must hold at the same time. |
| N9 | **Workflow authoring and import** | Add or edit workflows in the library schema, and import them from existing TTCN-3 / TS 38.523-1 test cases, so customers can bring their own procedures into the game. |

### Play modes: Manual and Autopilot *(new in v1.7)*

Every game, stress or ladder, starts in one of two **play modes**. The play mode decides **who chooses what to play**; the game models in §4 decide how each game is played once chosen. Both modes work the same way for **gNB / AI-RAN** and **modem** targets.

| | **Manual** — the engineer chooses | **Autopilot** — the application chooses |
| --- | --- | --- |
| Who picks the game | The engineer: stress game or ladder level, branches, difficulty, target | The agent, from what it has learned breaks this DUT |
| How the choice is made | Click the cards, type a **chat command**, or **dictate by voice** | A campaign plan built from run history, failure records and coverage gaps |
| What runs | One game (or an explicit queue the engineer dictates) | A continuous campaign: one game after another until the budget is spent or the engineer stops it |
| Engineer's role | Driver | Supervisor: approves the plan, can skip, pin or take control at any time |
| Best for | Reproducing a bug, checking a fix, a demo, learning the tool | Overnight and CI runs, a new software drop, hunting unknown weaknesses |
| Persona | Priya reproducing a defect; Kenji replaying a field crash | Sofia's nightly regression; Priya's "leave it running overnight" |

```mermaid
flowchart TD
    S([Engineer opens the Lobby]) --> T[Pick target: gNB / AI-RAN or Modem<br/>and DUT label]
    T --> M{Play mode}
    M -- Manual --> I{How?}
    I -- click --> C[Select stress game or ladder level,<br/>branches, difficulty]
    I -- chat --> K[Type: Play L1 registration on Hard,<br/>only the reject branches]
    I -- voice --> V[Hold the mic and say it] --> K
    K --> P[App shows its interpretation<br/>as editable chips]
    C --> G[GENERATE → Briefing → PLAY]
    P -- engineer confirms --> G
    M -- Autopilot --> B[Set scope and budget]
    B --> PL[Agent proposes a campaign plan:<br/>next games, each with a reason]
    PL -- engineer approves, edits or<br/>takes control --> R[Campaign runs game after game]
    R --> L[Every result updates what<br/>the agent knows about this DUT]
    L --> PL
```

#### Manual mode

The engineer decides what is played. There are three ways to say it, and all three produce the same Lobby selection.

1. **Click or tap.** The cards, segmented controls and branch picker. This is today's Lobby.
2. **Chat command.** A command bar accepts plain-language instructions. Examples:
   - "Play L1 registration on Hard for gNB-X sw 4.2.1"
   - "Run only REJECT #22 and T3510" (a subset of a level's branches)
   - "Signalling Storm, brutal, modem M3 fw 2.0.1"
   - "L2 then L3 on Normal" (a short queue, played in order)
   - "Replay run 044" or "Replay the last failure"
3. **Voice dictation.** A push-to-talk microphone in the same command bar. Speech is transcribed into the bar and handled exactly like typed chat.

**Rules for chat and voice**
- **Nothing starts on a command alone.** The app shows how it understood the command as editable chips (target · DUT · game or level · branches · difficulty), fills the Lobby to match, and waits for GENERATE / PLAY. A misheard word can never launch a 60K-UE storm.
- **Ambiguity asks, never guesses.** If a command could mean two things ("registration" in Stress mode, "reject" without a level), the app offers one clarification with tappable choices.
- **Vocabulary is the library's.** Commands understand level numbers and names, stress game names, branch names, 5GMM causes (#3 … #92), SIP codes (480, 486 …), workflow and case IDs (WF-07a), difficulties, targets and DUT labels.
- **During a game** the same bar accepts "stop" (opens the usual Stop confirmation), "switch to L2 next", "add Retry Loop" (S1 hot-add, when built) and "show the trace".
- **Ladder rules still apply.** Manual mode can choose any *open* level. A locked level is refused with the reason ("Clear L2 to unlock") unless the demo unlock mode is on.
- **Branch subsets.** Manual mode may play a chosen subset of a level's branches; the baseline branch is always included. A subset run is a *practice run*: it is reported normally but cannot clear the level or unlock the next one.
- **Accessibility.** Voice is an alternative, never the only route. Every voice action can also be typed or clicked.

#### Autopilot mode

The application decides what to play, using what it has learned about breaking this DUT. It runs a **campaign**: a sequence of games chosen one at a time, each picked because it is the most promising way to find a new failure.

**What Autopilot learns from** (its knowledge base, kept per target type, DUT family and DUT build):
- **Run history:** every game played against this DUT label and earlier builds of the same family; outcome, time to failure, KPIs breached, failed branches and the workflow step reached.
- **Move values:** the Q value of every move and branch at the end of each run (§4.3), carried over instead of starting from zero.
- **Coverage:** ladder levels not yet cleared, branches never played or not played since the last software drop, workflows added to the library since the last campaign.
- **Field captures** (production): datalake profiles that resemble where this DUT will be deployed.

**How Autopilot chooses the next game.**
1. **Basics first.** If an earlier ladder level is not cleared for this DUT build, that level comes first. Autopilot does not stress-test a DUT that can't register.
2. **Retest known failures.** Branches and stress moves that broke the *previous* build of this DUT family are replayed early, to confirm a fix or catch a regression.
3. **Exploit and explore.** Every remaining candidate game (stress game × difficulty, ladder level, or a focused branch subset) gets a **break score**: how strongly it has pressured this DUT family before, plus an exploration bonus for games tried rarely (§4.7). The highest score plays next.
4. **Escalate what survives.** A game the DUT passes comfortably is retried later at the next difficulty.
5. **Explain every pick.** Each planned game carries a one-line reason, for example: "REJECT #22 failed on sw 4.2.1: retest" or "Signalling Storm broke this family at 35K UEs: try Hard".

**Campaign controls**
- **Scope:** everything, ladder only, stress only, or chosen levels and games.
- **Budget:** time (e.g. 8 h), number of DUT failures to collect (e.g. 3), or both. The campaign stops when either is reached.
- **Ceiling:** the highest difficulty and cohort size Autopilot may use.
- **Plan preview:** before starting, the engineer sees the next five games with their reasons and can remove, reorder or pin games, or switch to Manual.
- **During the campaign:** Skip this game, Pin it (play again), Take control (switch to Manual with the current game kept) or Stop campaign.
- **After a DUT failure:** the run is saved with full diagnostics. Autopilot recovers the DUT through the adapter (reboot/reset where the instrument supports it, otherwise it waits for the engineer), then continues with the next game.
- **Campaign report:** failures found ranked by severity, ladder levels cleared, coverage before and after, and the evidence trail for every run.

**Same modes, two targets.** The knowledge base is kept separately for gNB / AI-RAN and modem targets, because what breaks them differs:

| | gNB / AI-RAN target | Modem target |
| --- | --- | --- |
| Who plays the tester | UE cohort (Multi-UE simulator, up to 60K) against a real gNB; 5GC and IMS emulated | Emulated network (gNB, 5GC, IMS) against the real modem; S5 reverse mode replays operator behaviour |
| What the DUT must get right | Admission, barring, scheduling, RRC/NGAP handling, handover decisions | Reactions to causes and timers, NAS state handling, measurement and re-establishment |
| Typical early Autopilot picks | Signalling Storm and GBR Squeeze at rising load; L3 bad-leader branches | L1 reject causes and timers; L2 STATUS and release cases; L3 RLF and T311 |
| Learned per | gNB family and software build | Modem family, firmware build and band |

These typical picks are the cold-start priors; after the first few runs the picks come from what has actually been learned about that DUT.

### Signalling workflow library

The library file `5g_signalling_workflows.json` is the single source of truth for what every move sends and what counts as correct.

- **Schema:** `test-arena.signalling-workflows` v3.0.0 · technology 5G NR SA (5GS), FR1 TDD · generated 3 Nov 2026 · 17 workflows, 29 cases, 1,927 steps.
- **Spec baseline:** RRC ASN.1 TS 38.331 Rel-18 (UPER-encoded and decoded back); default message contents TS 38.508-1; DCI TS 38.212 7.3.1; MAC TS 38.321; PHY procedures TS 38.213/38.214/38.211; NAS TS 24.501 (every PDU hand-encoded and round-tripped); EPS NAS TS 24.301 for interworking; IMS TS 24.229 with TS 34.229-1/-5.
- **Test environment defaults:** band n78, TDD, 30 kHz SCS, 40 MHz BWP, pattern DDDDDDDSUU; PLMN 001/01; Cells 1 and 2 in TAC 1, Cell 3 in TAC 2; reference UE identities (IMSI, 5G-S-TMSI, C-RNTI); DCI sizes, HARQ timing (K1, K2), SR config, radio bearers (SRB1–2, DRB1–4) and RRC procedure delays.
- **Step model:** each step has `seq`, `time` (SFN, slot, ms), `dir` (UL, DL, UE-internal, SS test-system action), `layer` (PHY, MAC, RRC, 5GMM, 5GSM, EMM, SIP), `channel` and `message`, plus whichever applies: `pdcch` (DCI format, RNTI, search space, CORESET, aggregation level and every field with bit width and value), `pdsch`/`pusch`, `pucch` (HARQ-ACK or SR), `prach`, `macPdu`, `messageRef` (RRC, into the message library), `carriesNAS` (on RRC steps), SIP `headers`/`sdp`, `timers` (start, stop, expired, running), `note` and `expect`.
- **NAS steps:** every NAS message is its own step (5GMM; 5GSM inside UL/DL NAS TRANSPORT; EMM for TS 24.301) with `nasMessageRef` into the NAS library, `carriedIn` (RRC message, field, and the NAS TRANSPORT payload container for 5GSM), `security` (security header type, MAC placeholder, sequence number, NAS COUNT, protected hex as carried), `procedure`, `summary` and `eps24301Equivalent`. UL NAS appears just before its RRC container; DL NAS just after.
- **Message libraries:** 107 RRC messages (ASN.1 type, UPER hex, decoded IE tree with `_absentOptional`/`_absentExtensions` at every level, contained CellGroupConfig/NAS) and 92 NAS PDUs (TS 24.501 clause, procedure, TS 24.301 equivalent, plain hex, IEs with IEI/length/value, absent optional IEs, contained messages).
- **Interworking references:** `nasInterworking24301` (10 places where EPS IEs ride in 5GS NAS, e.g. S1 UE network capability, selected EPS NAS algorithms, mapped EPS bearer contexts), `nasTimers` (14 timers with defaults, start/stop and their TS 24.301 equivalents, e.g. T3510 ↔ T3410/T3430) and `guti4G5GMapping` (TS 23.003 2.10.2).
- **Outcome model:** success workflows have `passCriteria`; failure workflows have `cases` (each a full step list), some with a `causeMatrix` (cause, encoded NAS PDU, expected UE behaviour) or `sipResponseMatrix`; `variants` describe changes to a base flow.
- **Verification list:** `defaults.verify` names 10 configuration areas (band/ARFCN, TDD pattern, CORESET#0, PRACH, SR and HARQ timing, TDRA tables, RLC/PDCP timers, logical-channel priorities, SIB1 UE timers, A3 and cell power) to check against the customer's TS 38.508-1 baseline.
- **Not covered yet** (`notCoveredYet`): UE-requested PDU session modification, EAP-AKA′, NSSAA, SOR/UE parameters update; MT voice/video, IMS registration and emergency, SMS; RRC_INACTIVE, release with redirect, UAC barring; CHO, DAPS, inter-frequency HO with gaps; CA, BWP switching, standalone BFR, on-demand SI; ETWS/CMAS, positioning, NR-DC, RedCap, NTN. The LTE RRC and L1 side of EPS fallback arrives with the LTE stage.

#### The 17 workflows and where they play

| ID | Workflow | Category | Plays in |
| --- | --- | --- | --- |
| WF-01 | MO Data Call | data · success | L2 baseline; Signalling Storm, Small-Packet Flood, GBR Squeeze |
| WF-02 | MT Data Call (paging) | data · success | L2; Signalling Storm (paging flood), Small-Packet Flood |
| WF-03 | IMS Voice Call (VoNR, MO) | ims-voice · success | L5 baseline; GBR Squeeze |
| WF-04 | IMS Video Call (ViNR, MO) | ims-video · success | L5 |
| WF-05 | Intra-frequency handover (+ variant b, new TA) | mobility-connected · success | L3 baseline |
| WF-06 | Cell reselection in RRC_IDLE (cases a same TA, b new TA + mobility registration) | mobility-idle · success | L3 |
| WF-07 | Registration failure (cases a REJECT with 9-cause matrix, b auth failure #20 + AUTHENTICATION REJECT, c NAS SMC reject #23, d no response/T3510) | failure-registration | L1; Retry Loop |
| WF-08 | MO data call failure (cases a–e, SERVICE REJECT cause matrix) | failure-data | L2; Retry Loop |
| WF-09 | IMS voice call failure (cases a 486 with response matrix [480/503/408, no final response, IMS-VoPS=0], b no 5QI 1 flow, c QoS rule rejected #83) | failure-ims-voice | L5 |
| WF-10 | IMS video call failure (cases a 488, b video port 0, c video DRB configuration failure) | failure-ims-video | L5 |
| WF-11 | Handover failure, T304 expiry (+ variant b, re-establishment fails on T311) | failure-mobility | L3 |
| WF-12 | Radio link failure (+ variant e, no recovery) | failure-radio | L3; L5 "RLF during an active call" |
| WF-13 | Initial registration + PDU session establishment (cases a internet + IMS sessions and configuration update, b with identification, c IMS PDU session reject #27) | registration · success | L1 baseline and identification; L2 PDU session reject; Retry Loop |
| WF-14 | De-registration (cases a UE switch-off, b UE normal, c network-initiated with re-registration) | deregistration · success | L1 |
| WF-15 | Periodic registration update (T3512) | registration · success | L1 |
| WF-16 | PDU session release and 5GSM/5GMM error handling (cases a UE release, b network release #39 + re-establishment, c 5GSM STATUS, d 5GMM STATUS) | session-management · success | L2 |
| WF-17 | 5GS–EPS interworking, TS 24.301 linkage (cases a EPS → 5GS idle mobility, b EPS fallback for IMS voice with 24.301 TAU) | interworking · success | L3 (a), L5 (b) |

#### How a workflow becomes a move

1. **Pick.** The agent picks a branch, which names a workflow and optionally a case, a matrix row or a variant (for example `WF-07a · #22`).
2. **Replay.** The adapter replays the steps in order, PHY to SIP; each NAS PDU is its own step next to the RRC message that carries it. The branch's injection step is where the game deviates from the happy path: an `SS` action (cell power change, withheld response), a cause-matrix row (the REJECT's NAS PDU is replaced by the matrix encoding), a SIP response-matrix row, or a variant note.
3. **Fan out.** Workflows describe one UE. The cohort plays the same branch across many UEs with staggered start times, so per-UE outcomes become the level's KPIs (success %, latency, rejects, timeouts).
4. **Judge.** The branch passes only if the DUT meets the workflow's `passCriteria` or the matching `expectedUE` (correct message, cause, timer start/stop, state). A wrong message, a missed timer or a hang is a branch failure; a crash or KPI breach is `DUT DOWN`.
5. **Record.** The run log keeps the workflow ID, case and the step reached, so a failure report reads "Failed at WF-11 step 4: no RRCReestablishmentRequest before T311 expiry".

#### Stress games and their workflow basis

| Stress game | Workflow basis | What is scaled up |
| --- | --- | --- |
| Signalling Storm | WF-01 steps 8–16 (Msg1 → Msg4 HARQ-ACK), 8–22 (to RRCSetupComplete carrying SERVICE REQUEST); WF-02 steps 1–18 (paging → RRCSetupComplete) | Random-access and RRC setup bursts, paging floods |
| Retry Loop | WF-08a, WF-08b, WF-07a (#22 + T3346), WF-13a (re-attach) | UEs that ignore RRCReject wait time, missing Msg4/T300 and T3346 back-off |
| Small-Packet Flood | WF-01 steps 42–58 (SR → grant → PUSCH, CRC fail and retransmission), 42–45, 8–45; WF-02 | SR storms and short UL bursts from mMTC UEs |
| GBR Squeeze | WF-03 steps 43–61 (PDU SESSION MODIFICATION for 5QI 1 to COMPLETE), 1–61; WF-01 steps 46–51 (DL data with NACK and retransmission) | Dedicated 5QI 1 QoS flows until admission and PRBs run out |

Random access, scheduling requests and HARQ are now in the library itself, so stress moves replay them exactly as listed.

#### Signalling trace and DUT response

- **Who is the DUT.** For a gNB target, the gNB is the DUT: its DL RRC/MAC/PHY messages are DUT responses; the UE is simulated and the 5GC and IMS are emulated. For a modem target, the UE is the DUT and everything it sends is a DUT response.
- **What each entry shows.** Time, sender and receiver lanes, layer, channel, container, SFN/slot, workflow and step, timers, the representative UE (index, slice, C-RNTI, masked 5G-S-TMSI, cell) and the full decode: DCI fields, PDSCH/PUSCH/PUCCH/PRACH parameters, MAC PDU, RRC ASN.1 (UPER hex, IE tree, absent IEs, contained CellGroupConfig/NAS, and which NAS messages it carries), NAS PDU (TS 24.501 clause, procedure, IE table with IEI/length/value, absent optional IEs, contained messages, security header with the protected PDU as carried, TS 24.301 equivalent), SIP headers and SDP. A PHY · MAC filter hides transport steps to leave the signalling.
- **gNB-internal steps.** "gNB decodes PUSCH: CRC OK/fail" is the gNB's own HARQ decision, so for a gNB target it is a DUT response. NR has no PHICH: UL ACK/NACK shows as NDI toggling in the next grant.
- **Identity masking.** The reference UE's IMSI/SUPI is masked to its last 4 digits everywhere it appears, including SIP URIs and the BCD digits inside NAS hex.
- **Latency.** A DUT response's latency is the library's own SFN/slot gap to the previous step, stretched by the current load; under barring or rejecting it is 3× that.
- **A verdict on every DUT response.** *As expected* (the message the workflow expects, with its latency), *late* (correct but slower than unloaded, e.g. while barring access), *wrong message* (e.g. RRCReject instead of RRCSetup) or *no response* (timer expiry, or the DUT going down).
- **Effect on the game.** On the ladder, a wrong or missing DUT response means the branch must replay cleanly before it can pass. In stress games that UE's flow is abandoned and the next UE starts.

### Procedure Ladder — game catalogue

Every level is a two-player game: the **Tester** (agent + UE cohort) tries to make the DUT misbehave, and the **DUT** wins a branch only if it behaves as the workflow says it should. Each level uses a different game-theory model (explained for newcomers in §4), which decides how the agent explores.

| Level | Procedure | Game model | Workflows | What the model tells the tester |
| --- | --- | --- | --- | --- |
| 1 | Registration ("The Handshake") | **Sequential (extensive-form) game**, solved by backward induction | WF-07, WF-13, WF-14, WF-15 | The DUT must be correct at *every* node of the procedure tree, including off-path branches (subgame perfection). The happy path alone proves little. |
| 2 | Data Call ("The Agreement") | **Coordination game** | WF-01, WF-02, WF-08, WF-13, WF-16 | UE and network must agree to bring up service. Test the miscoordination cells (reject, no response, partial accept), not just the agreed one. |
| 3 | Handover & Mobility ("Leader and Follower") | **Stackelberg game** | WF-05, WF-06, WF-11, WF-12, WF-17 | The network leads by setting measurement and trigger parameters; the UE follows. The tester plays a "bad leader" and breaks the handover to test recovery. |
| 4 | Throughput ("Sharing the Pie") | **Congestion game + Nash bargaining**; link adaptation as a game against Nature | WF-01 (bearer set-up only) | Users compete for PRBs. Proportional-fair scheduling corresponds to the Nash Bargaining Solution, so fairness and efficiency are both measured. |
| 5 | IMS Voice & Video ("The Negotiation") | **Alternating-offers bargaining** (Rubinstein) | WF-03, WF-04, WF-09, WF-10, WF-17 | SIP/SDP offer–answer and QoS preconditions are a sequence of offers and counter-offers. The tester rejects, counters or stalls to see whether the DUT settles, falls back or releases resources cleanly. |

Branch tables list the order the branches are played in; the first row is always the baseline. Easy plays the first four rows.

#### Level 1 — Registration

- **Procedure tree:** RRC setup → REGISTRATION REQUEST (T3510 started) → [IDENTITY REQUEST/RESPONSE] → AUTHENTICATION → NAS SECURITY MODE → AS security, UE capability → REGISTRATION ACCEPT (T3510 stopped) / REJECT → REGISTRATION COMPLETE → PDU sessions (internet, IMS) → CONFIGURATION UPDATE; plus periodic registration and de-registration. Source: WF-07, WF-13, WF-14, WF-15; TS 24.501 5.5.1, 5.5.2, 5.4; TS 38.523-1 9.1.5.

| # | Branch | Workflow | Correct DUT behaviour (from the library) |
| --- | --- | --- | --- |
| 1 | Initial registration + PDU sessions (baseline) | WF-13a | Cleartext REGISTRATION REQUEST, full message in the SECURITY MODE COMPLETE container; SHT 3/4 then 2; internet and IMS PDU sessions with mapped EPS bearer contexts; configuration update completed |
| 2 | REJECT #3 Illegal UE | WF-07a cause matrix | USIM invalid for 5GS; delete 5G-GUTI, TAI list, ngKSI; 5GMM-DEREGISTERED |
| 3 | REJECT #7 5GS services not allowed | WF-07a cause matrix | USIM invalid for 5GS services |
| 4 | REJECT #11 PLMN not allowed | WF-07a cause matrix | Forbidden PLMN list; PLMN selection |
| 5 | REJECT #12 Tracking area not allowed | WF-07a cause matrix | Forbidden TAs for regional provision of service; LIMITED-SERVICE |
| 6 | REJECT #13 Roaming not allowed in this TA | WF-07a cause matrix | Forbidden TAs for roaming; PLMN selection |
| 7 | REJECT #15 No suitable cells in TA | WF-07a cause matrix | Forbidden TAs for roaming; search another TA |
| 8 | REJECT #22 Congestion + T3346 | WF-07a cause matrix | Start T3346; no new attempt until expiry |
| 9 | REJECT #27 N1 mode not allowed | WF-07a cause matrix | Disable N1 mode; select E-UTRA/EPS (24.301 attach) |
| 10 | REJECT #62 No network slices available | WF-07a cause matrix | Store rejected NSSAI |
| 11 | No response (T3510) | WF-07d | Retry after T3511; after 5 attempts start T3502 |
| 12 | Authentication MAC failure (#20) | WF-07b | AUTHENTICATION FAILURE #20 (T3520); accepts AUTHENTICATION REJECT |
| 13 | Authentication synch failure (#21 + AUTS) | WF-07b, adapted | AUTHENTICATION FAILURE #21 with AUTS *(library shows #20; the cause octet is changed, AUTS is a placeholder, and the flow stops after the failure)* |
| 14 | NAS SMC rejected (#23) | WF-07c | SECURITY MODE REJECT #23 on mismatched replayed capabilities |
| 15 | Identification procedure | WF-13b | IDENTITY RESPONSE with SUCI, then registration completes |
| 16 | Periodic registration (T3512 expiry) | WF-15 | REGISTRATION REQUEST type periodic; REGISTRATION ACCEPT stops T3510 |
| 17 | De-registration, switch off | WF-14a | DEREGISTRATION REQUEST (switch off); no accept expected |
| 18 | De-registration, normal | WF-14b | T3521 started; stopped on DEREGISTRATION ACCEPT |
| 19 | Network de-registration, re-registration required | WF-14c | DEREGISTRATION ACCEPT, then a new initial registration |

- **KPIs (5):** Registration success (%), Registration latency (ms), Retries (/s), Rejects (/s), Procedure timeouts (/s).

#### Level 2 — Data Call

- **Procedure:** service request from 5GMM-REGISTERED/RRC_IDLE to an active user plane on PDU session 1 (QFI 1). Source: WF-01, WF-02, WF-08; TS 24.501 5.6.1, TS 38.331 5.3.3.
- **Prerequisite:** Level 1 cleared for this DUT.

| # | Branch | Workflow | Correct DUT behaviour |
| --- | --- | --- | --- |
| 1 | MO data call (baseline) | WF-01 | RRC cause mo-Data; service accepted, T3517 stopped; DRB1 carries data on QFI 1 |
| 2 | MT data call via paging | WF-02 | Responds to paging with mt-Access; service type "mobile terminated services"; DL data received |
| 3 | RRC connection rejected (wait time + T302) | WF-08a | No new mo-Data RRCSetupRequest until T302 expires |
| 4 | No Msg4, T300 expiry | WF-08b | Contention resolution fails, RA re-attempted; on T300 expiry NAS is informed and the service request aborted |
| 5 | SERVICE REJECT #9 / #10 | WF-08c cause matrix | Deregister, delete 5G-GUTI as required, initial registration |
| 6 | SERVICE REJECT #22 + T3346 | WF-08c cause matrix | Start T3346; no service request until expiry |
| 7 | SERVICE REJECT #28 / #7 | WF-08c cause matrix | Restricted area: NON-ALLOWED-SERVICE and mobility registration; #7: USIM invalid for 5GS |
| 8 | Service request no response (T3517) | WF-08d | Abort after T3517 (15 s); release NAS connection locally; back to 5GMM-REGISTERED |
| 9 | PDU session reactivation failure (#92) | WF-08e | Stop T3517; no user data on PSI 1; retry later |
| 10 | IMS PDU session establishment reject (#27) | WF-13c | Stop T3580; keep the internet session; no immediate retry with the same DNN |
| 11 | UE-requested PDU session release | WF-16a | T3582 started and stopped; RELEASE COMPLETE; mapped EPS bearer removed |
| 12 | Network release (#39) and re-establishment | WF-16b | RELEASE COMPLETE, then a new PDU SESSION ESTABLISHMENT REQUEST |
| 13 | 5GSM STATUS on PTI mismatch | WF-16c | Answers an invalid PTI with 5GSM STATUS #47 |
| 14 | 5GMM STATUS received | WF-16d | No state change |

- **Still parked:** UE-requested PDU session modification and the other PDU-session negotiation branches from v1.1 (IPv4v6 → IPv4 only, non-subscribed S-NSSAI, SSC mode, 5QI downgrade) until the library adds them.
- **KPIs (5):** Session setup success (%), Setup latency (ms), Fallbacks (/s), Session rejects (/s), Session drops (/s).

#### Level 3 — Handover & Mobility

- **Leader parameters:** the A3 measurement config from the library defaults (`MeasConfig_A3`: a3-Offset 3 dB, hysteresis 0, TTT 0 ms, all `verify`). The tester re-tunes them to play a "bad leader".
- **Prerequisite:** Level 2 cleared. Needs three emulated cells: Cells 1 and 2 in TAC 000001, Cell 3 in TAC 000002.

| # | Branch | Workflow | Correct DUT behaviour |
| --- | --- | --- | --- |
| 1 | Intra-frequency handover (baseline) | WF-05 | MeasurementReport on A3; CFRA and RRCReconfigurationComplete before T304 |
| 2 | Too-early trigger (bad leader: low offset) | WF-05 + A3 re-tune | Hands over without a drop or immediate return |
| 3 | Too-late trigger (bad leader: high offset, long TTT) | WF-05 + A3 re-tune | Recovers before RLF, or re-establishes per WF-12 |
| 4 | Ping-pong (bad leader: zero hysteresis, short TTT) | WF-05 + A3 re-tune | Avoids bouncing between Cell 1 and Cell 2 |
| 5 | Handover into a new TA | WF-05 variant b | Hands over to Cell 3, then mobility registration updating |
| 6 | Idle-mode reselection | WF-06a | Reselects to Cell 2 and answers paging there |
| 7 | Reselection into a new TA | WF-06b | Mobility registration updating with new TAI list |
| 8 | Handover failure (T304 expiry) | WF-11 | Target withholds the RAR; T304 expiry; T311; RRCReestablishmentRequest with cause handoverFailure |
| 9 | Re-establishment fails (T311 expiry) | WF-11 variant b | RRC_IDLE; 5GMM recovers with registration or SERVICE REQUEST |
| 10 | Radio link failure (T310 expiry) | WF-12 | Declares RLF at T310 expiry, re-establishes with otherFailure, RLF report correct |
| 11 | No recovery after RLF | WF-12 variant e | RRC_IDLE; mobility registration or SERVICE REQUEST on return |
| 12 | EPS → 5GS idle mobility | WF-17a | REGISTRATION REQUEST with mapped 5G-GUTI, UE status and EPS NAS message container; PDU sessions mapped from PDN connections |

- **Dropped with library v3** (no flow in the library any more): inter-frequency handover, priority-based reselection, re-establishment answered with RRCSetup, invalid reconfigurationWithSync, RLF from RLC max retransmissions or random-access problem, beam failure recovery.
- **KPIs (5):** HO success (%), Interruption time (ms), Ping-pong rate (%), RLF (/min), Session drops (/s).

#### Level 4 — Throughput

- **Bearer set-up:** WF-01 brings up DRB1 on QFI 1 before each branch. The library has no throughput workflows yet, so the branches below remain tester-defined.
- **Prerequisite:** Level 3 cleared.

| # | Branch | Correct DUT behaviour |
| --- | --- | --- |
| 1 | Single-UE peak (baseline) | Reaches peak throughput for one UE |
| 2 | Load ramp | Holds cell throughput as UEs are added |
| 3 | Mixed GBR priority | Serves GBR users without starving the rest |
| 4 | High-CQI hog | Keeps fairness when one strong UE pushes |
| 5 | Channel fade (Nature) | Picks the right MCS as SINR drops |

- **KPIs (5):** Cell throughput (Mbps), Per-UE throughput p5 (Mbps), Fairness (Jain's index, 0–1), BLER (%), PRB utilisation (%).

#### Level 5 — IMS Voice & Video *(new in v1.2)*

- **Procedure:** VoNR/ViNR MO call over IMS PDU session 2 (DNN ims, 5QI 5): service request, SIP INVITE, network-requested PDU SESSION MODIFICATION for dedicated QoS flows (5QI 1 voice, 5QI 2 video), 183/PRACK/UPDATE preconditions, 180/200/ACK, media, BYE and release. Source: WF-03, WF-04, WF-09, WF-10; TS 24.229, TS 34.229-1/-5, TS 26.114, TS 24.501 6.3.2.
- **Preconditions:** REGISTRATION ACCEPT indicates IMS VoPS = 1; IMS PDU session established; UE IMS-registered.
- **Prerequisite:** Level 4 cleared.

| # | Branch | Workflow | Correct DUT behaviour |
| --- | --- | --- | --- |
| 1 | VoNR MO call (baseline) | WF-03 | RRC cause mo-VoiceCall; PDU SESSION MODIFICATION adds 5QI 1 before 180/200; dialog set up and released cleanly |
| 2 | ViNR MO call | WF-04 | RRC cause mo-VideoCall; 5QI 1 and 5QI 2 flows with mapped EPS bearers; separate DRBs |
| 3 | Remote busy (486) | WF-09a | ACK; call cleared |
| 4 | Temporarily unavailable (480) | WF-09a response matrix, adapted | ACK; call cleared |
| 5 | Service unavailable (503) | WF-09a response matrix, adapted | ACK; call cleared |
| 6 | Request timeout (408) | WF-09a response matrix, adapted | ACK; call cleared |
| 7 | No final response (timer B) | WF-09a response matrix | P-CSCF failure handling; re-register |
| 8 | Dedicated QoS flow never set up | WF-09b | No UPDATE with curr:qos sendrecv; CANCEL, 487, ACK |
| 9 | UE rejects bad QoS rule | WF-09c | PDU SESSION MODIFICATION COMMAND REJECT #83; QoS flow not created |
| 10 | RLF during an active call | WF-03 steps 1–96 + WF-12 (composed) | Re-establishes and continues, or drops the call cleanly |
| 11 | IMS VoPS not supported | WF-09a response matrix | No VoNR origination; EPS fallback or CS per domain selection |
| 12 | EPS fallback for voice | WF-17b | Moves to E-UTRA; TS 24.301 TRACKING AREA UPDATE with active flag; PDU sessions map to PDN connections |
| 13 | Video codec not acceptable (488) | WF-10a | ACK; may retry voice-only |
| 14 | Video m-line rejected, audio accepted | WF-10b | Continues voice-only on QFI 2 |
| 15 | Video DRB cannot be configured | WF-10c | RRC re-establishment (reconfigurationFailure); video call fails |

- **Dropped with library v3** (WF-10a no longer has a response matrix): video QoS flow not authorised, video call declined (603), mid-call downgrade glare (491).
- **KPIs (5):** Call setup success (%), Call setup time (ms, INVITE to 200 OK), QoS flow failures (/min), SIP error responses (/s), Call drops (/min).

#### Clearing a level

- A level runs in rounds. In each round the agent picks the next branch, favouring branches not yet covered and branches that scored highest reward.
- A level is **cleared** when every branch in the set has been passed at least once with no DUT failure (100% branch coverage), and the minimum run time has elapsed.
- The branch set grows with difficulty: Easy plays the baseline + 3 branches; Brutal plays the full set, at full cohort size, with branches overlapping.
- A branch passes when the DUT meets the workflow's pass criteria or expected UE behaviour for that case. The judge checks messages, causes, timer starts/stops and end state, not just KPIs.
- Clearing a level unlocks the next one for the same DUT label.
- A DUT failure ends the level as `DUT DOWN`, exactly as in a stress game.

## 4. Game Theory Guide

*For engineers who are new to game theory. It explains every game Test Arena plays, why each model fits the procedure under test, and how the agent turns a game into pass or fail. Diagrams are Mermaid.*

### 4.1 Why test with game theory?

A normal test script is a monologue: send A, expect B, send C, expect D. A real network is a conversation with an opponent. Thousands of UEs, a congested core and a fading channel all push back. Game theory is the mathematics of situations where **each side's best move depends on what the other side does**. That is exactly the situation a gNB or modem faces in the field.

Test Arena treats every test as a game between two players:

- **Tester:** the AI agent plus the simulated UE cohort (up to 60,000 UEs). It wins points when the DUT misbehaves.
- **DUT:** the gNB, AI-RAN node or modem. It "wins" when it behaves exactly as the 3GPP workflow says it should, under any pressure.

The tester is not trying to be fair. It is trying to find the move the DUT handles worst. Game theory tells it **where to look**.

### 4.2 Ten words you need

| Term | Plain meaning | In Test Arena |
| --- | --- | --- |
| **Player** | Someone who makes decisions | The tester (agent + cohort) and the DUT |
| **Move (action)** | One choice a player can make | A stress move (RACH burst ×4) or a ladder branch (REJECT #22); for the DUT, serve, bar, reject or a protocol reply |
| **Strategy** | A complete plan: which move to play in every situation | The agent's policy for picking the next move; the DUT's implementation |
| **Payoff** | What a player gets from an outcome | Tester: the **reward** (KPI pressure); DUT: staying within thresholds and following the workflow |
| **Best response** | The move that gives you the highest payoff, given what the other player does | The agent's chosen move after it sees the DUT's last reaction |
| **Equilibrium** | A state where no player can do better by changing only their own move | A DUT that is correct at every node: the tester has no move that "wins" |
| **Leader / follower** | The leader moves first and commits; the follower sees it and reacts | Stress games and L3: the cohort (or the network's parameters) leads; the DUT follows |
| **Game tree** | A diagram of moves taken in turn, with branches for each choice | L1 Registration: REQUEST → AUTH → SMC → ACCEPT/REJECT |
| **Repeated game** | The same game played many rounds, so players can learn | Every game: the agent plays a move every 2 s and learns from the result |
| **Nature** | A "player" with no goals that injects randomness | L4: the radio channel (fading, SINR drops) |

A simple way to read every model below: **who moves first, what can each player do, what does each player want, and what does "correct" look like at the end?**

### 4.3 The engine shared by every game

All nine games (four stress, five ladder) run on the same loop. Only the moves, the payoff and the end condition change.

```mermaid
flowchart LR
    A[Agent picks a move<br/>every 2 s] --> C[UE cohort plays it<br/>workflow steps on the air]
    C --> D[DUT reacts<br/>serve · bar · reject · protocol reply]
    D --> K[KPIs measured<br/>every 1 s]
    K --> R[Reward computed<br/>pressure on the DUT]
    R --> Q[Agent updates its score<br/>for that move: Q value]
    Q --> A
    K -- KPI over limit for 3 s --> X[DUT DOWN<br/>game ends]
```

**How the agent learns, step by step**

1. **Score each move.** The agent keeps a number per move, called its **Q value**: "how much pressure this move usually puts on the DUT". All start at 0.
2. **Choose.** Every 2 s it plays the move with the highest Q value (**exploit**). 15% of the time it plays a random move instead (**explore**), so it never stops looking for a weakness it hasn't found.
3. **Measure.** Every second the five KPIs are compared with their thresholds. A KPI at its limit scores a ratio of 1; idle scores 0.
4. **Reward.** Reward = Σ weight × (ratio − 0.5) × 10, over the game's three reward components (for example reject, latency and drop). Pressure above half the limit is positive reward; a comfortable DUT gives negative reward.
5. **Update.** After each 2 s window the move's Q value moves 30% of the way towards the average reward it just earned: Q ← Q + 0.3 × (reward − Q).
6. **Escalate.** On a fixed clock (every 90 s on Easy down to every 30 s on Brutal) the load level rises, so the same move hits harder. Stress games keep escalating until the DUT fails or the engineer presses Stop.

**Worked example.** In Signalling Storm, "RACH burst ×4" has Q = 0.8 and "Paging flood" has Q = 0.3. The agent plays RACH burst ×4. RACH success falls towards its limit and rejects climb, giving reward 2.1. The new Q value is 0.8 + 0.3 × (2.1 − 0.8) = 1.19, so the agent becomes even more likely to repeat it. When the DUT adapts (it starts barring access classes and rejects fall), rewards drop. The Q value sinks, and another move overtakes it. That back-and-forth is the game.

*Implementation note: the build uses a single-state Q-learning bandit, labelled "DQN v0.3". A full deep Q-network that takes the KPI vector as input is the M4 target (§3).*

### 4.4 Stress games: Stackelberg leader–follower

**The idea.** In a **Stackelberg game** one player (the **leader**) moves first and commits. The other (the **follower**) sees that move and picks its best reaction. A smart leader chooses its move *knowing how the follower will react*, and picks the move whose reaction hurts the follower most.

**In Test Arena.** The UE cohort is the leader: it commits to a load pattern (a RACH burst, a retry loop, a packet flood or a GBR demand). The DUT is the follower: its admission control, barring and scheduler react. The agent learns which leader move produces the worst follower reaction, then escalates it.

```mermaid
sequenceDiagram
    participant L as Leader · UE cohort + agent
    participant F as Follower · DUT
    L->>F: Commit to a move (e.g. RACH burst ×4 at load 0.6)
    F-->>L: Best reaction (serve, bar classes 2–9, or reject RRC setup)
    Note over L: Measure KPIs, compute reward, update Q
    L->>F: Next move: repeat if it hurt, switch if it didn't
    F-->>L: Reaction under higher load
    Note over L,F: Escalation every 30–90 s raises the stakes<br/>until a KPI breaches for 3 s (DUT DOWN) or Stop
```

**What "winning" looks like in a payoff table.** Each cell shows (tester payoff, DUT payoff) for one move against one DUT reaction. The numbers are illustrative.

| Cohort move ↓ / DUT reaction → | Serve everyone | Bar access classes | Reject RRC setup |
| --- | --- | --- | --- |
| Light RACH burst | (0, +2) DUT copes | (+1, +1) unnecessary barring | (+2, −1) over-reaction |
| Heavy RACH burst ×4 | (+3, −3) RACH collapses | (+1, +1) **correct defence** | (+2, 0) rejects spike |
| Retry loop ignoring back-off | (+3, −3) storm grows | (+2, 0) partly works | (+1, +1) **correct with wait time** |

A robust DUT picks the bold cell in every row: the best reaction to each attack. A weak DUT has at least one row where its reaction gives the tester a big payoff, and the agent will find that row.

| Stress game | Leader moves (workflow basis) | Follower reactions tested | What the agent hunts for |
| --- | --- | --- | --- |
| **Signalling Storm** | RACH burst ×2/×4, RRC setup storm, paging flood (WF-01, WF-02) | RACH capacity, RRCSetup vs RRCReject, access barring | The burst size where RACH success or setup latency collapses |
| **Retry Loop** | Access retry, back-off ignore, retry loop ×8, re-attach (WF-08a/b, WF-07a #22, WF-13a) | Wait-time enforcement, T300/T302/T3346 handling | A retry pattern that turns the DUT's own rejects into a storm |
| **Small-Packet Flood** | Tiny UL packets, keep-alive flood, SR storm, mMTC wake burst (WF-01, WF-02) | Scheduling-request handling, PUCCH capacity, PRB fragmentation | The mMTC mix where latency or drops break while PRBs are "free" |
| **GBR Squeeze** | GBR 50 Mbps demand, GBR ramp, 5QI 1 surge, PRB hog (WF-03, WF-01) | Admission control for guaranteed bit rate, pre-emption | The point where admitted GBR flows start dropping instead of new ones being refused |

### 4.5 The Procedure Ladder: five games, five lessons

Each ladder level uses the game model that best describes its procedure. The model decides **which branches the tester must cover** before the level can be cleared.

```mermaid
flowchart LR
    L1[L1 Registration<br/>Sequential game<br/>right at every node] --> L2[L2 Data Call<br/>Coordination game<br/>agree or fail cleanly]
    L2 --> L3[L3 Handover<br/>Stackelberg game<br/>follow a bad leader safely]
    L3 --> L4[L4 Throughput<br/>Congestion + bargaining<br/>share fairly]
    L4 --> L5[L5 IMS Voice & Video<br/>Alternating offers<br/>settle or release]
```

#### L1 Registration: a sequential game (game tree)

**The idea.** In a **sequential game** players move in turn, so the game is drawn as a **tree**. Each fork is a decision. **Backward induction** solves it: start at the leaves and ask "what is the right move here?", then work back to the root. A strategy is **subgame perfect** if it is right at *every* fork, including forks the normal path never reaches.

**Why it fits.** Registration is a chain of request/response turns. A UE that only ever sees REGISTRATION ACCEPT has proved one path through the tree. The tester's job is to steer it down every other branch and check it is still correct there.

```mermaid
flowchart TD
    S([UE: REGISTRATION REQUEST · T3510 starts]) --> N1{Network}
    N1 -- no answer --> T1[UE must retry after T3511;<br/>T3502 after 5 attempts]
    N1 -- IDENTITY REQUEST --> I[UE: IDENTITY RESPONSE with SUCI] -- then AUTHENTICATION REQUEST --> N2
    N1 -- AUTHENTICATION REQUEST --> N2{UE checks AUTN}
    N2 -- MAC wrong --> F20[UE: AUTH FAILURE #20]
    N2 -- SQN stale --> F21[UE: AUTH FAILURE #21 + AUTS]
    N2 -- valid --> R[UE: AUTH RESPONSE] --> N3{Network: SECURITY MODE COMMAND}
    N3 -- capabilities mismatch --> SMR[UE: SECURITY MODE REJECT #23]
    N3 -- valid --> SMC[UE: SECURITY MODE COMPLETE] --> N4{Network}
    N4 -- REGISTRATION REJECT #3 #7 #11 #12 #13 #15 #22 #27 #62 --> RJ[UE: act on the cause<br/>forbid PLMN/TA, start T3346, disable N1…]
    N4 -- REGISTRATION ACCEPT --> OK([UE: REGISTRATION COMPLETE · PDU sessions])
```

Every leaf is an L1 branch. The level clears only when the DUT is correct at all of them: that is subgame perfection. Periodic registration (T3512) and the three de-registration cases are further sub-trees of the same game.

**Payoff.** The tester scores when the DUT leaves a leaf wrongly: it retries when it must back off, keeps a forbidden PLMN, or misses a timer. Reward components: wrong recovery, latency, hang.

#### L2 Data Call: a coordination game

**The idea.** In a **coordination game** both players do best when they *agree*, and both lose when they don't. The classic example is two drivers choosing which side of the road to drive on. There are good outcomes (both agree) and bad ones (**miscoordination**).

**Why it fits.** Bringing up a data session needs the UE and the network to agree: the UE asks with SERVICE REQUEST, and the network accepts and sets up the radio bearer. The happy path is the "both agree" cell. The bugs live in the miscoordination cells: rejects, silence and half-accepts.

| UE ↓ / Network → | Accept and set up DRB | Reject (cause) | No answer |
| --- | --- | --- | --- |
| **Request service** | ✅ Agreed: data flows (WF-01/02) | ⚠ UE must act on the cause and back off (WF-08a, WF-08c #9/#10/#22/#28/#7) | ⚠ UE must abort on T300/T3517 (WF-08b/d) |
| **Request, but a session is broken** | ⚠ Partial accept: no data on PSI 1 (WF-08e #92) | ⚠ PDU session reject #27 (WF-13c) | — |
| **Session in use, someone releases it** | ⚠ Release and re-establish (WF-16a/b) | ⚠ STATUS on a protocol error (WF-16c/d) | — |

```mermaid
flowchart LR
    U[UE wants data] -->|SERVICE REQUEST| N{Network}
    N -->|ACCEPT + RRCReconfiguration| A[Coordinated:<br/>data on QFI 1]
    N -->|REJECT with cause| B[Miscoordination 1:<br/>UE follows the cause]
    N -->|silence| C[Miscoordination 2:<br/>UE times out cleanly]
    N -->|ACCEPT but PSI failed| D[Miscoordination 3:<br/>UE sends nothing on PSI 1]
```

**Payoff.** The tester scores when a miscoordination cell leaves the DUT stuck: retry loops, stale sessions, or data on a session that wasn't granted.

#### L3 Handover & Mobility: a Stackelberg game with a bad leader

**The idea.** As in §4.4, a leader commits and a follower reacts. Here the **network is the leader**: it sets the measurement and trigger parameters (A3 offset, hysteresis, time-to-trigger). The **UE is the follower**: it measures and reports when the trigger fires. Normally the leader tunes these to make handover smooth.

**The tester plays a bad leader.** It deliberately sets parameters that are too eager, too lazy or too twitchy, then breaks the handover itself, to check that the follower still ends up connected.

```mermaid
flowchart TD
    L[Leader: network sets A3<br/>offset · hysteresis · TTT] --> F[Follower: UE measures<br/>Cell 1 vs Cell 2]
    F -->|well tuned| G[Clean handover<br/>before T304]
    L -.->|bad leader: offset too low| E[Too early:<br/>hand over, bounce back]
    L -.->|bad leader: offset high, TTT long| T[Too late:<br/>link fails before HO → RLF]
    L -.->|bad leader: zero hysteresis| P[Ping-pong:<br/>Cell 1 ⇄ Cell 2]
    G --> B{Tester breaks it}
    B -->|target withholds RAR| H[T304 expiry → re-establishment]
    B -->|T311 runs out| I[RRC_IDLE → NAS recovers]
    B -->|serving cell fades| R[T310 → RLF → re-establishment]
    B -->|UE arrives from LTE| X[EPS → 5GS mobility registration]
```

**Payoff.** The tester scores on handover failures, long interruptions, ping-pong and radio link failures. A good DUT keeps the session alive whatever the leader does.

#### L4 Throughput: a congestion game, Nash bargaining and a game against Nature

Three ideas meet here.

1. **Congestion game.** Many players share one resource (the cell's PRBs), and each player's payoff falls as more players use it. Left alone, greedy players crowd each other out.
2. **Nash bargaining.** When players must share a pie, the **Nash Bargaining Solution** splits it to maximise the *product* of everyone's gains. Grabbing everything isn't allowed, and neither is an equal split that wastes capacity. A proportional-fair scheduler gives exactly this solution, so it is the "correct" answer the DUT should approximate.
3. **Game against Nature.** The radio channel is a player with no goal that keeps changing SINR. The DUT's link adaptation (MCS choice) must respond well to whatever Nature does.

```mermaid
flowchart LR
    subgraph Players
      U1[URLLC UEs]
      U2[eMBB UEs]
      U3[mMTC UEs]
    end
    N((Nature:<br/>fading, SINR)) --> S
    U1 & U2 & U3 -->|demand PRBs| S[DUT scheduler]
    S -->|proportional fair = Nash bargaining| OK[Efficient and fair:<br/>high cell throughput,<br/>healthy 5th-percentile UE]
    S -->|greedy or rigid| BAD[Unfair or wasteful:<br/>one UE hogs, edge UEs starve,<br/>BLER climbs]
```

**Payoff.** The tester scores on throughput gaps, unfairness (Jain's index falling) and BLER. The five branches (single-UE peak, load ramp, mixed GBR priority, high-CQI hog, channel fade) each stress one corner of this triangle. L4 has no library workflows yet; WF-01 only brings up the bearer.

#### L5 IMS Voice & Video: alternating-offers bargaining (Rubinstein)

**The idea.** In **Rubinstein bargaining** two players take turns making offers until one accepts. Waiting has a cost (the "discount"), so sensible players settle quickly rather than haggle for ever. A player who neither accepts nor walks away cleanly is playing badly.

**Why it fits.** A VoNR/ViNR call set-up *is* an offer–counter-offer exchange. The INVITE carries an SDP offer, 183 Session Progress answers it, PRACK and UPDATE confirm that QoS is in place, and 200 OK accepts. The QoS precondition (a dedicated 5QI 1 flow) is part of the deal. Timers (SIP timer B, the UE's own set-up timer) are the cost of waiting.

```mermaid
sequenceDiagram
    participant UE as UE (offerer)
    participant N as IMS + 5GC (responder)
    UE->>N: INVITE · SDP offer (EVS/AMR-WB, QoS precondition)
    N-->>UE: 183 Session Progress · counter-offer (chosen codec)
    N-->>UE: PDU SESSION MODIFICATION · 5QI 1 flow (the "price")
    UE->>N: PRACK, UPDATE · precondition met
    N-->>UE: 180 Ringing, 200 OK · deal accepted
    Note over UE,N: Tester's moves: reject (486/480/503/488), stall (no answer: timer B),<br/>withhold the QoS flow, send a bad QoS rule, cut the radio (RLF),<br/>or move the call to LTE (EPS fallback)
```

**What correct looks like.** For every tester move the DUT must either settle (continue voice-only, fall back to EPS, re-establish) or walk away cleanly (ACK, CANCEL, release the QoS flow). It must never haggle for ever or leave stale QoS flows behind.

**Payoff.** The tester scores on failed calls, slow set-up and drops.

### 4.6 From game to verdict

The game decides *what to play*. The workflow library decides *what is correct*. A branch passes only when both agree.

```mermaid
flowchart TD
    P[Agent picks a branch<br/>favouring uncovered, high-reward ones] --> W[Workflow replayed<br/>PHY → RRC → NAS → SIP]
    W --> J{DUT reply matches the<br/>workflow's expected behaviour?}
    J -- yes, for 4 s with no failure --> OKB[Branch passed]
    J -- wrong message or no reply --> RP[Replay the branch]
    J -- KPI over limit for 3 s --> DD[DUT DOWN]
    OKB --> C{All branches covered<br/>and minimum time met?}
    C -- yes --> CL[LEVEL CLEAR → next level unlocks]
    C -- no --> P
```

Stress games have no clear condition. They follow the left half of this loop (pick, replay, react) and end only on DUT DOWN or Stop.

### 4.7 Autopilot: the game of choosing games

Inside every game the agent plays a Stackelberg, sequential, coordination, congestion or bargaining game (§4.4–§4.5). **Autopilot adds an outer game on top:** deciding *which game to play next*.

**The idea: the multi-armed bandit.** Picture a row of slot machines ("one-armed bandits"), each paying out at an unknown rate. Every pull teaches you a little about one machine. Pull only the machine that has paid best so far and you might miss a better one you tried once and got unlucky with. Pull at random and you waste pulls. The good strategy balances **exploiting** what has worked with **exploring** what is uncertain.

For Autopilot, each **arm** is a candidate game: a stress game at a difficulty, a ladder level, or a focused branch subset. A **pull** is playing that game against the DUT. The **payout** is its **break value**: the pressure it put on the DUT (the run's average reward), with a large bonus if it caused a DUT failure or a failed branch.

**The rule Autopilot uses (UCB1, "upper confidence bound").** For each candidate game *g*:

> **score(g) = average break value of g + c × √( ln N ÷ n(g) )**

- *N* is the total number of games played in this knowledge base and *n(g)* is how often *g* has been played.
- The first term rewards games that have broken this DUT family before (**exploit**).
- The second term grows for games tried rarely (**explore**), so nothing is written off after one unlucky run. The constant *c* sets the balance.
- Games never played have an infinite score, so each candidate is tried at least once.
- The rules in §3 (basics first, retest known failures) are applied *before* the scores, as hard ordering constraints.

**Worked example.** A gNB family has three candidates after 20 games:

| Candidate | Plays n(g) | Average break value | Exploration bonus (c = 1) | Score |
| --- | --- | --- | --- | --- |
| Signalling Storm · Hard | 10 | 0.62 | √(ln 20 ÷ 10) = 0.55 | **1.17** |
| GBR Squeeze · Hard | 8 | 0.40 | √(ln 20 ÷ 8) = 0.61 | 1.01 |
| L3 bad-leader subset | 2 | 0.35 | √(ln 20 ÷ 2) = 1.22 | **1.57** ← next |

Signalling Storm has the best record, but the L3 subset has only been tried twice, so its uncertainty earns it the next slot. If it breaks nothing, its average falls and its bonus shrinks, and Storm takes over again.

```mermaid
flowchart LR
    KB[(Knowledge base<br/>per target · DUT family · build)] --> F{Hard rules}
    F -- earlier level not cleared --> LV[Play that ladder level]
    F -- previous build failed here --> RT[Retest that branch or move]
    F -- otherwise --> U[Score every candidate:<br/>average break value + exploration bonus]
    U --> PICK[Play the highest score]
    LV & RT & PICK --> RUN[Inner game runs<br/>§4.3 engine]
    RUN --> OUT[Break value, failures,<br/>coverage, Q values]
    OUT --> KB
```

**Why this matters for both targets.** The bandit doesn't need to know *why* a gNB is weak to RACH bursts or a modem to cause #22. It only needs the payouts. That is why the same Autopilot works for gNB / AI-RAN and modem DUTs: the knowledge base is separate, the method is the same.

### 4.8 At a glance

| Game | Model | Who leads | Tester's moves | DUT wins when… | Tester scores on |
| --- | --- | --- | --- | --- | --- |
| Autopilot (outer game) | Multi-armed bandit (UCB1) over games | The agent | Which game, level, branch subset and difficulty to play next | Every candidate game stops paying out: nothing left breaks it | Break value across the campaign |
| Stress games (4) | Stackelberg, repeated with escalation | UE cohort | Load patterns from WF-01/02/03/07/08/13 | It picks the right defence (serve, bar, reject with wait time) at every load | Rejects, latency, drops |
| L1 Registration | Sequential game, backward induction | Turns alternate | Rejects, auth/SMC failures, silence, identity, periodic, de-registration | It is correct at every node of the tree (subgame perfect) | Wrong recovery, latency, hang |
| L2 Data Call | Coordination game | UE requests | Reject, silence, partial accept, session reject/release, STATUS | Every miscoordination ends cleanly | Miscoordination, latency, drop |
| L3 Handover | Stackelberg with a bad leader | Network parameters | Bad A3 tuning, withheld RAR, T311/T310 expiry, EPS → 5GS | The session survives any leader | HO failure, interruption, drop |
| L4 Throughput | Congestion + Nash bargaining + Nature | Simultaneous | Load ramp, GBR mix, CQI hog, fading | Proportional-fair sharing and good MCS choice | Throughput gap, unfairness, BLER |
| L5 IMS Voice & Video | Rubinstein alternating offers | UE offers | SIP rejects, stalls, missing or bad QoS, RLF, EPS fallback | It settles or walks away cleanly, never stalls | Failed call, set-up time, drop |

**Further reading.** M. J. Osborne, *An Introduction to Game Theory* (sequential, Stackelberg and bargaining games); F. Kelly, "Charging and rate control for elastic traffic" (1997), on proportional fairness and its link to Nash bargaining; A. Rubinstein, "Perfect Equilibrium in a Bargaining Model", *Econometrica* (1982).

## 5. Technical Constraints

The browser shows and steers the game. The real-time work happens server-side, next to the instruments.

### Hard limits

- **Real-time split.** Slot-level decisions (sub-ms) run on the simulator/agent host, never in the browser. The UI receives state at 10–20 Hz over WebSocket.
- **Scale.** The UI must represent cohorts of up to 60K UEs. It renders aggregated cohort tiles, not one element per UE.
- **Instrument interfaces.** All hardware access goes through the adapter layer over Anritsu's standard interfaces (Ethernet/SCPI, GPIB, USB). There are no vendor-specific calls outside the adapter.
- **Datalake access is read-only.** Testcase generation queries field captures but never writes back to or modifies them.
- **Data confidentiality.** Field captures and vendor/MNO profiles are customer-confidential. This requires on-premise or private-cloud deployment, no third-party telemetry, and masking of subscriber identifiers (IMSI/SUPI) before display.
- **Deterministic replay.** Every game logs its seed, agent version and move sequence, so any failure can be replayed exactly.
- **Chat and voice input.** Chat commands are parsed on the Test Arena host by a constrained grammar over the library's vocabulary (an on-premise language model may be added later), so no command text leaves the lab. Voice uses an on-premise speech-to-text engine in lab deployments. Audio is processed in memory and never stored, and voice is off by default. The static demo may use the browser's built-in speech recognition only as an opt-in (see the site contract).
- **Autopilot knowledge base.** It is stored on the Test Arena host next to run history, keyed by target type, DUT family and DUT build. It never leaves the deployment and never mixes customers. Every campaign decision is logged with its inputs, so a campaign can be replayed like a single game.
- **Ladder emulation.** For gNB targets, the ladder exercises the gNB's RRC and NGAP handling of each procedure with the core network emulated. For modem targets, the network side is emulated. Level 3 needs three emulated cells: two in one tracking area and one in a second (the library's NR Cells 1–3).
- **Workflow library.** `5g_signalling_workflows.json` is read-only input, validated against its schema name and version before use. The adapter translates each step (UL/DL message, SS action, timer) into simulator commands. Fields marked `verify: true` are shown as warnings until confirmed for the customer's 38.508-1 baseline.
- **Solo-dev stack.**
  - Backend and game engine: Python (FastAPI).
  - Agent: PyTorch.
  - Front-end: TypeScript/React.
  - Storage: PostgreSQL + a time-series store for KPIs.
  - Deployment: Docker Compose.
  - Choose boring, well-documented libraries that AI agents generate reliably.
- **Standards.** Testcase vocabulary aligns with 3GPP terms (RACH, RRC, PRB, 5QI/GBR). Ladder levels use 5GMM/5GSM terms and cause values from TS 24.501, RRC procedures and measurement events from TS 38.331, idle-mode reselection from TS 38.304, IMS from TS 24.229 / 34.229-1/-5 / 26.114, and default message contents from TS 38.508-1, as recorded in the workflow library. O-RAN interfaces (E2/A1) are a target for AI-RAN DUTs.

### Explicit non-goals (v1)

- Not a 3GPP conformance or certification tool. Test Arena finds breaking points; it does not issue pass/fail certificates. Clearing a ladder level is not a conformance verdict.
- No in-browser signal processing or RF analysis.
- No public SaaS / multi-tenant cloud.
- No mobile app. Desktop browsers on lab workstations only (1920×1080 minimum).
- No training of large foundation models. The agent is a compact RL model trained per feature.
- No support for non-Anritsu simulators in v1.

## 6. Design Standard

**Direction: "Arcade Mission Control"**, a retro arcade cabinet bolted onto a professional test bench. The game layer makes engineers want to keep playing; the instrument layer makes them trust every number.

> This section is superseded by `design-standard.md` (v2.1, "Neo-Tokyo test dojo"). It is kept for history only.

Rule of thumb: *arcade on the surface, oscilloscope underneath.* Pixel styling is for moments (score, game start, DUT failure). Data is always crisp and readable.

### Colour

Dark only. Every colour carries one meaning and keeps it everywhere.

| Token | Hex | Meaning |
| --- | --- | --- |
| `bg-void` | #05070A | App background (near-black CRT) |
| `bg-panel` | #0B1015 | Panels and cards |
| `line-arena` | #39FF6A | Arena frame, Network/DUT, "running" state |
| `ue-urllc` | #FF4FD8 | URLLC UEs |
| `ue-embb` | #3FD4FF | eMBB UEs |
| `ue-mmtc` | #FFB23F | mMTC UEs |
| `agent` | #8B7BFF | DQN / Game Theory agent, AI actions |
| `score` | #FFE14D | Reward and score only |
| `fail` | #FF3B3B | DUT failure, critical alerts only |
| `text-hi` / `text-lo` | #E6EDF3 / #7D8A99 | Body text / labels |

- Neon colours are for lines, glyphs and text, never large filled areas.
- Red appears only when the DUT fails, so it feels like an event.
- Anritsu brand green sits in the top bar and reports. The arena uses the neon palette.

### Typography

| Use | Font | Rules |
| --- | --- | --- |
| Arcade display (score, game titles, "PLAY 042", "DUT DOWN") | Press Start 2P | Uppercase, ≤ 3 words, never for body text or data |
| UI and body | Inter | 14 px base, 1.5 line height |
| Data, KPIs, logs, IDs | JetBrains Mono | Tabular numerals, right-aligned in tables |

### Texture and atmosphere

- 1 px panel borders with a soft outer glow (`box-shadow 0 0 8px` in the border colour at 40%).
- A faint scanline overlay (3% opacity) on the arena canvas only, never over logs or tables.
- Pixel-sprite icons for UEs and the gNB tower, as in the concept image; standard line icons elsewhere.
- Labelled flow arrows between the STATE → REWARD → ACTION → AGENT panels, always visible while a game runs.

### Layout

- Three zones: **Arena** (left, ~60%) · **Agent loop** (right) · **Diagnostics strip** (bottom, always visible).
- A legend bar pinned under the arena: URLLC · eMBB · mMTC · NETWORK.
- Engineering density: no empty hero areas. Every pixel shows either game state or a KPI.

### Motion and feedback

- The arena is always alive while running. UE sprites pulse with traffic, RACH attempts animate upward, and barring shows as a dashed wall.
- The score counts up with a short tick on every reward update. Difficulty escalation flashes the frame once.
- DUT failure is the "game over" moment. The arena freezes, the frame turns red, a "DUT DOWN" banner appears and the diagnostics snapshot slides up.
- The hot-add prompt appears as an arcade "CONTINUE?" card. It is dismissable and never blocking.
- Respect `prefers-reduced-motion`: keep state changes, drop pulses and scanlines.

### Accessibility and trust

- Never rely on colour alone. UE types also differ by sprite shape and label.
- Text contrast is ≥ 4.5:1 against `bg-panel`.
- Every number on screen has a unit and can be exported.

## 7. Success Criteria

The MVP is done when an engineer can go from lobby to a running stress game in under 60 seconds and the agent measurably out-breaks a scripted test.

### MVP definition of done

| Area | Measure | Target |
| --- | --- | --- |
| Speed to play | Lobby → game running | < 60 s, ≤ 4 clicks |
| Generation | Testcase generated from datalake captures | < 2 s per testcase |
| Endless loop | Unattended run without platform crash | ≥ 24 h on mock adapter, ≥ 8 h on real simulator |
| Stop conditions | Game halts on user Stop, DUT failure or Level Clear | 100% of cases, within 1 s |
| Diagnostics | Failures with auto-captured logs + game state | 100% of failures |
| Replay | Failure reproduced from saved seed and move log | ≥ 95% of failures |
| UI responsiveness | Arena update rate at 60K-UE cohort | ≥ 10 fps, no dropped state |
| Agent value | Time to first DUT failure vs. scripted stress baseline | ≥ 30% faster on a reference DUT |
| Stress games | Built-in games playable end to end | 4 of 4 |
| Procedure Ladder | Ladder levels playable end to end, each showing its game model, branch set and five KPIs | 5 of 5 |
| Workflow coverage | Library workflows used by at least one game | 17 of 17 |
| Traceability | Ladder branches that name a workflow ID (or are listed as tester-defined) | 100% |
| Message trace | Ladder failure reports that name the workflow step where the DUT went wrong | 100% |
| Signalling trace | Run reports that include the decoded trace, focused on the failing exchange when there is one | 100% |
| Ladder unlocks | Level N is playable only after Level N−1 is cleared for the same DUT | 100% of cases |
| Branch coverage | Agent covers the full branch set before declaring a level clear | 100% coverage on every clear |
| Manual · chat | Commands from a 200-command reference set mapped to the intended Lobby selection | ≥ 95% correct; 100% of ambiguous commands ask rather than guess |
| Manual · voice | Same reference set spoken in a lab at normal noise | ≥ 90% correct after one confirmation; zero games started without confirmation |
| Manual · speed | Command spoken or typed → Briefing shown | < 10 s |
| Autopilot · value | Time to first DUT failure vs Manual play by an engineer new to the DUT, on a reference DUT with known seeded defects | ≥ 30% faster; finds ≥ 80% of the seeded defects within the budget |
| Autopilot · regression | Defects found on build N that are retested first on build N+1 | 100% |
| Autopilot · explainability | Planned games that show a reason and a knowledge-base source | 100% |
| Both targets | Manual and Autopilot run end to end for gNB / AI-RAN and for modem | 2 of 2 targets |

### v1.1 and adoption

- **Hot-add:** a second game is added to a running session with no restart.
- **Tournament:** all five ladder levels run back to back in one session with state carried across, and at least one interaction defect is found that isolated levels missed.
- **Automation:** a full game run is triggered and reported via API from a CI pipeline, with zero manual steps.
- **Profiles:** at least 2 RAN vendor and 2 MNO profiles are imported and change agent behaviour.
- **Pilot:** 3 internal or customer engineers each run ≥ 5 games in a week, and at least one reports a defect not found by their existing suite.
- **Usability:** pilot users rate "I wanted to keep playing" ≥ 4/5.

### Open questions

- **Play modes:** which on-premise speech-to-text engine (and which languages and accents) does voice need? Is a local language model wanted for freer chat, or is the constrained grammar enough?
- **Play modes:** can Autopilot knowledge be shared across DUT families or customers (anonymised priors), or must every deployment start cold? What default budget should a nightly campaign use?
- **Play modes:** which instruments and DUTs support automated recovery (reboot or reset) after a failure, so an Autopilot campaign can continue unattended?

- Which reference DUT sets the agent-value baseline: a commercial gNB, an O-RAN stack such as OAI/srsRAN, or a modem?
- What counts as "DUT failure" per feature: crash only, or KPI breach thresholds? This needs a table per game, including each ladder level.
- What are the datalake schema and access path for field captures? To confirm with the data team.
- **Ladder:** what minimum run time should a level need before it can be cleared, and should a clear expire when a new DUT build is loaded?
- **Ladder:** does the Multi-UE simulator emulate the two or more cells Level 3 needs, or does that require a second instrument?
- **Workflows:** who owns `5g_signalling_workflows.json`, how new versions are released, and which TS 38.508-1 version is the baseline for the 10 areas in `defaults.verify`? The library has no #21 synch-failure encoding (adapted from #20), no full flows for SIP 480/503/408 (played on the 486 flow) and no video response matrix (three L5 branches dropped). Real NAS integrity codes need keys from the test USIM; the library uses placeholders.
- **Workflows:** the library still has no throughput workflows (L4) and no UE-requested PDU session modification (parked L2 branches). Who writes them?
- **Workflows:** how are single-UE conformance workflows fanned out across a 60K cohort (stagger, identities, per-UE judging) without overloading the simulator's message logging?
