# CLAUDE.md — Test Arena

## Project identity

Test Arena turns AI-RAN and modem testing into an endless, adversarial game. An AI-trained UE cohort plays against the device under test (DUT) until the DUT breaks or the engineer calls time. This repo is a static, playable demo of that experience; the game loop is simulated in the browser.

There are two ways to play:

- **Stress games:** four endless load games (Signalling Storm, Retry Loop, Small-Packet Flood, GBR Squeeze).
- **Procedure Ladder:** five levels played in order (L1 Registration → L2 Data Call → L3 Handover & Mobility → L4 Throughput → L5 IMS Voice & Video). Each level uses a different game-theory model (sequential, coordination, Stackelberg, congestion/Nash bargaining, alternating-offers bargaining), has its own branch set and KPIs, and unlocks the next level when cleared.

There are two play modes, for both gNB / AI-RAN and modem targets:

- **Manual:** the engineer chooses the game, level, branches and difficulty by clicking, typing a chat command or dictating by voice. Commands only fill the Lobby; nothing starts without confirmation.
- **Autopilot:** the application runs a campaign, choosing each next game from a per-target, per-DUT-family knowledge base (ladder order, then retests, then a UCB bandit over games). The engineer can skip, pin, take control or stop.

Every ladder branch and stress move traces to a 3GPP signalling workflow in `5g_signalling_workflows.json` (schema 3.0.0: 17 workflows WF-01 to WF-17, 29 cases, 1,927 PHY-to-SIP steps with SFN/slot timing, 107 RRC messages in full TS 38.331 ASN.1, 92 NAS PDUs decoded per TS 24.501 with TS 24.301 links, plus 5GS↔EPS timer and interworking tables). Treat that file as the source of truth for message names, sequences, timers and expected UE behaviour: embed it as `DATA.workflows` and read from it, never retype it.

## Technical stack and constraints

- **One file:** a single `index.html` with all CSS in `<style>` and all JavaScript in `<script>`.
- **No frameworks or libraries:** no React, Vue, jQuery, Tailwind or chart libraries. Use vanilla JS, the DOM, SVG and Canvas.
- **No npm, no build tools:** no `package.json`, bundlers, transpilers or preprocessors. The file opens directly in a browser.
- **No backend:** no fetch to a server, no form submission, no auth, no database. All data is embedded in the file as JS objects (`DATA.games` for stress games, `DATA.ladder` for ladder levels, `DATA.workflows` for the workflow library).
- **External requests:** Google Fonts only, as listed in the design standard.
- **Browser storage:** optional, try/catch-guarded, and used only for convenience (session runs, ladder unlocks, the Voice input setting and the Autopilot knowledge base).
- **Voice:** the browser's `SpeechRecognition` only, opt-in from Settings (off by default), push-to-talk only; it is the single allowed exception to the external-request rule. Chat is parsed in the page by a deterministic grammar; no language model and no network call.

## Supporting documents

| Document | What it holds | Read it when |
| --- | --- | --- |
| `site-contract.md` | Screens and routes, every content field, navigation/scroll/hover/responsive behaviour, animations, acceptance criteria | Before building or changing any screen or behaviour. **It is the source of truth for what to build.** |
| `design-standard.md` | Colour, typography, spacing, texture and motion tokens, plus a merge checklist | Before any visual decision. **It overrides the PRD's Design Standard section.** |
| `content-model.md` | Field-level content constraints, per-game KPI and state sets, and the open gaps (G1–G24) | When writing copy or sample data, or when you hit an undefined value |
| `PRD.md` | Vision, personas, prioritised features, the workflow library mapping, the Procedure Ladder game catalogue, the Game Theory Guide (§4: every game model explained with diagrams), constraints, success criteria | When judging scope or priority, or when you need the branch sets, workflow references and game models for ladder levels. Must-Haves only, unless told otherwise. |
| `5g_signalling_workflows.json` | The signalling workflow library (schema 3.0.0): 17 workflows and their cases, each step with DCI/PHY/MAC detail, RRC `messageRef` into the ASN.1 message library, NAS steps with `nasMessageRef` into the NAS library (IEs, security, TS 24.301 equivalent), SIP headers/SDP, timers, cause and SIP response matrices, pass criteria and the `defaults.verify` list | Whenever you build or change a ladder branch, message flow, narration or report. |

**If documents conflict**, use this order: site contract → design standard → content model → PRD.

## Current state

- `index.html` implements site contract v7: the embedded library v3 (`DATA.workflows`, checked against schema 3.x with `messageLibrary` and `nasLibrary`), NAS messages as library steps with a full NAS decode, branch sets drawn from WF-01 to WF-17 (L1 19, L2 14, L3 12, L5 15), PHY/MAC steps in the Briefing and trace, an inspector that decodes DCI, PHY channels, MAC PDUs, RRC ASN.1, NAS and SIP, IMSI masking, and saved traces tied to the library version. Automated Chrome checks pass (56 regression, 29 trace); the full cross-browser AC sweep (§5.1) is still to run.
- **Not yet built:** site contract v8 play modes (Manual chat/voice, Autopilot campaigns, the `#/plan` and `#/campaign/:id` routes, `DATA.commands`, `CONFIG.autopilot`; AC-P1–AC-P14). Build them next, against site contract §2 and §3.6 and the design standard's play-mode components.
- If `5g_signalling_workflows.json` changes, re-embed it in `index.html` as `WORKFLOW_LIBRARY` (minified, whitespace-only changes) and check that every branch's `wf` reference still resolves: `seq` numbers in `at: { seq }` and stress `steps: [a, b]` slices are tied to this version.
