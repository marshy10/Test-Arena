# CLAUDE.md — Test Arena

## Project identity

Test Arena turns AI-RAN and modem testing into an endless, adversarial game. An AI-trained UE cohort plays against the device under test (DUT) until the DUT breaks or the engineer calls time. This repo is a static, playable demo of that experience; the game loop is simulated in the browser.

## Technical stack and constraints

- **One file:** a single `index.html` with all CSS in `<style>` and all JavaScript in `<script>`.
- **No frameworks or libraries:** no React, Vue, jQuery, Tailwind or chart libraries. Use vanilla JS, the DOM, SVG and Canvas.
- **No npm, no build tools:** no `package.json`, bundlers, transpilers or preprocessors. The file opens directly in a browser.
- **No backend:** no fetch to a server, no form submission, no auth, no database. All data is embedded in the file as JS objects.
- **External requests:** Google Fonts only, as listed in the design standard.
- **Browser storage:** optional, try/catch-guarded, and used only for convenience.

## Supporting documents

| Document | What it holds | Read it when |
| --- | --- | --- |
| `site-contract.md` | Screens and routes, every content field, navigation/scroll/hover/responsive behaviour, animations | Before building or changing any screen or behaviour. **It is the source of truth for what to build.** |
| `design-standard.md` | Colour, typography, spacing, texture and motion tokens, plus a merge checklist | Before any visual decision. **It overrides the PRD's Design Standard section.** |
| `content-model.md` | Field-level content constraints and the open gaps (G1–G12) | When writing copy or sample data, or when you hit an undefined value |
| `PRD.md` | Vision, personas, prioritised features, constraints, success criteria | When judging scope or priority. Must-Haves only, unless told otherwise. |

**If documents conflict**, use this order: site contract → design standard → content model → PRD.
