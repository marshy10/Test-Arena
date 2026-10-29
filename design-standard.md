# Test Arena — Design Standard

*Visual specification v2 · 29 Oct 2026 · supersedes §5 "Design Standard" in `PRD.md`*

## 0. The atmosphere in one line

**"Neo-Tokyo test dojo."** Imagine a modern competitive game client built for a precision lab in Tokyo. It is dark, calm and exact, with a single blade of vermilion when something matters.

Three ideas drive every decision:

| Idea | Japanese root | What it means on screen |
| --- | --- | --- |
| **Ma** (間): purposeful space | Negative space is part of the design | Generous, regular spacing; one dominant element per view; never fill space just because it is there |
| **Kire** (切れ): the cut | A clean, decisive edge | Sharp corners, hairline rules, chamfered (cut) corners on key surfaces, fast confident motion |
| **Jo-ha-kyū** (序破急): slow, break, fast | Rhythm of Noh and martial arts | Important sequences start quietly, build, then finish fast and stop dead; no bounce, no wobble |

The game feel comes from **state, score and stakes** (live numbers, a clear opponent, a decisive "defeat" moment), not from cartoon styling.

> **Rule for ambiguous decisions:** choose the option that is quieter, sharper and more precise. If two choices are equally quiet, choose the one with more space.

---

## 1. Colour palette

Dark theme only. All values are CSS custom properties on `:root`; never hard-code a hex in components.

### 1.1 Backgrounds (sumi ink layers)

Layers get lighter as they come forward. Never skip a layer: a panel on the page uses `bg-1`, a card inside a panel uses `bg-2`.

| Token | Hex | Name | Use |
| --- | --- | --- | --- |
| `--bg-0` | `#07080C` | Sumi (墨) | Page background, behind everything |
| `--bg-1` | `#0D1017` | Yoru (night) | Panels, shell bar, sidebars |
| `--bg-2` | `#141925` | Kon (deep navy) | Cards, inputs, table rows, dialogs |
| `--bg-3` | `#1C2333` | Ai-nezumi | Hover fill on `bg-2` elements, selected rows |
| `--bg-glass` | `rgba(20, 25, 37, 0.72)` | — | Floating layers (dialogs, tooltips, HUD overlays) with blur (§4.3) |

### 1.2 Lines

| Token | Hex | Use |
| --- | --- | --- |
| `--line-1` | `#222A3A` | Default 1 px borders, dividers, table rules |
| `--line-2` | `#34405A` | Input borders, hovered borders, chart axes |
| `--line-hair` | `rgba(237, 234, 227, 0.08)` | Faint grid lines, inner panel rules |

### 1.3 Text (washi paper tones)

Body text is warm off-white, not pure white. This is the "paper" in the ink-and-paper contrast.

| Token | Hex | Contrast on `bg-1` | Use |
| --- | --- | --- | --- |
| `--text-1` | `#EDEAE3` | ~15:1 | Primary text, headings, values |
| `--text-2` | `#A3ABBA` | ~7.5:1 | Secondary text, labels, captions |
| `--text-3` | `#6B7487` | ~3.9:1 | Placeholders, disabled text, Japanese sub-labels; **never for anything that must be read** |
| `--text-inverse` | `#07080C` | — | Text on solid accent fills |

### 1.4 Accents

Each accent has **one meaning**, and that meaning never changes. Count the accents on screen: if more than two are fighting for attention outside the Arena, remove one.

| Token | Hex | Name | Meaning and use |
| --- | --- | --- | --- |
| `--accent-primary` | `#3EE6F0` | Denki (electric cyan) | The interactive colour: primary buttons, focus rings, active nav, links, selected states. "You can act here." |
| `--accent-shu` | `#F2462E` | Shu-iro (朱, vermilion) | Stakes: STOP, DUT failure, KPI breaches, the hanko seal. "Something broke or will be destroyed." Never decorative. |
| `--accent-kin` | `#E8C25A` | Kin (金, gold) | Score and reward only, plus achievements later. "You earned this." |
| `--accent-hisui` | `#4ADE9A` | Hisui (翡翠, jade) | Healthy / running / connected. DUT serving normally; game `RUNNING`. |
| `--accent-fuji` | `#9B87F5` | Fuji (藤, wisteria) | The AI agent: agent panel, agent actions, model labels. |
| `--accent-warn` | `#F5A524` | — | Warnings and `SIMULATED` badges only |

### 1.5 Data colours (UE slices; never used elsewhere)

| Token | Hex | Slice | Sprite shape |
| --- | --- | --- | --- |
| `--ue-urllc` | `#FF5FA2` | URLLC (sakura pink) | Diamond |
| `--ue-embb` | `#3EB8F0` | eMBB (sora blue) | Rounded rectangle |
| `--ue-mmtc` | `#F5A524` | mMTC (amber) | Small square |

### 1.6 Tints (derived; for fills behind an accent)

To fill behind an accent, use the accent at a fixed opacity. Never use a new hex.

| Purpose | Opacity |
| --- | --- |
| Hover tint | 8% |
| Selected / active tint | 14% |
| Badge background | 16% |
| Glow (box-shadow colour) | 40% |

```css
/* example */
--accent-primary-tint-8:  rgba(62, 230, 240, 0.08);
--accent-primary-tint-14: rgba(62, 230, 240, 0.14);
```

### 1.7 Colour rules

1. **Area rule.**
   - Backgrounds take ~85% of any screen.
   - Text takes ~12%.
   - Accents take ≤ 3%.
   - Accents are for lines, text, icons and small fills, never large solid areas. The one exception is the hanko seal on DUT failure.
2. **Vermilion is rare.** It may appear only on the STOP button, failure states and breached KPIs. If vermilion is visible while nothing is wrong, it is a bug.
3. **One primary action per view** gets a solid `--accent-primary` fill. Everything else is outline or ghost.
4. **Never rely on colour alone.** Pair every state colour with an icon, label or shape.

---

## 2. Typography

### 2.1 Families

Load from Google Fonts with `font-display: swap`.

| Role | Family | Weights loaded | Use |
| --- | --- | --- | --- |
| **Display** | **Chakra Petch** | 500, 600 | Screen titles, scores, game names, state banners, big numbers. Its squared, cut terminals carry the futuristic "kire". |
| **UI / body** | **Inter** | 400, 500, 600 | All interface text, paragraphs, labels, buttons |
| **Japanese** | **Zen Kaku Gothic New** | 500, 700 | Japanese sub-labels and the hanko seal text only |
| **Data / mono** | **JetBrains Mono** | 400, 500 | KPI values, IDs, seeds, timestamps, logs, tables of numbers |

Fallback stacks:

```css
--font-display: "Chakra Petch", "Rajdhani", system-ui, sans-serif;
--font-ui:      "Inter", system-ui, -apple-system, "Segoe UI", sans-serif;
--font-jp:      "Zen Kaku Gothic New", "Hiragino Sans", "Noto Sans JP", sans-serif;
--font-mono:    "JetBrains Mono", ui-monospace, "SFMono-Regular", monospace;
```

### 2.2 Type scale

The base is 14 px, chosen for engineering density. The scale ratio is ~1.25, rounded so every line height sits on the 4 px grid.

| Token | Family | Size / line height | Weight | Letter spacing | Case | Use |
| --- | --- | --- | --- | --- | --- | --- |
| `display-xl` | Display | 56 / 64 | 600 | +0.02em | UPPER | PLAY splash, final score, "DUT DOWN" |
| `display-l` | Display | 40 / 48 | 600 | +0.02em | UPPER | Live total score, reward value |
| `h1` | Display | 28 / 36 | 600 | +0.04em | UPPER | Screen titles (SELECT GAME, HISTORY) |
| `h2` | Display | 20 / 28 | 500 | +0.06em | UPPER | Panel titles (AGENT LOOP, DIAGNOSTICS) |
| `h3` | UI | 16 / 24 | 600 | 0 | Sentence | Card titles, dialog titles, game names on cards |
| `body` | UI | 14 / 20 | 400 | 0 | Sentence | Paragraphs, descriptions, form values |
| `body-strong` | UI | 14 / 20 | 600 | 0 | Sentence | Emphasis inside body; button labels |
| `label` | UI | 11 / 16 | 600 | +0.12em | UPPER | Field labels, table headers, KPI names, tags |
| `caption` | UI | 12 / 16 | 400 | +0.01em | Sentence | Hints, helper text, tooltips |
| `data-l` | Mono | 20 / 28 | 500 | 0 | — | Primary KPI values |
| `data` | Mono | 13 / 20 | 400 | 0 | — | IDs, timestamps, table numbers, logs |
| `jp-sub` | JP | 11 / 16 | 500 | +0.08em | — | Japanese sub-label under a heading or nav item |
| `jp-seal` | JP | 32 / 32 | 700 | 0 | — | Kanji inside the hanko seal |

### 2.3 Typography rules

1. **Display font is for titles and numbers only.** Never use it for sentences or anything longer than 4 words.
2. **All numbers that change live use mono** with `font-variant-numeric: tabular-nums`, so digits don't jitter.
3. **Uppercase is always tracked.** Any uppercase text uses the letter spacing in the table; never set uppercase at 0 tracking.
4. **Line length** for body text is a maximum of 72 characters (`max-width: 72ch`).
5. **Paragraph spacing** equals one line height (20 px). There is no indent.
6. **Weights:** use only the weights listed. No 300, no 800 or 900, no italics except in quotations.
7. **Mobile (< 768 px):**
   - `display-xl` → 40/48
   - `display-l` → 32/40
   - `h1` → 22/28
   - `h2` → 16/24
   - Everything else is unchanged.

### 2.4 Japanese touch: how kanji and kana are used

Japanese is **atmosphere, not content**. It always sits *with* English and never replaces it.

- **Placement.** A `jp-sub` label sits directly under, or to the right of, an `h1` or nav label, in `--text-3`, with 4 px gap.
- **Maximum one Japanese label per component.** Body text is never translated.
- **`aria-hidden="true"`** on all decorative Japanese text, so screen readers don't read it.
- **Fixed vocabulary.** Use only these, so two developers never pick different words:

| English | Japanese | Reading |
| --- | --- | --- |
| Lobby | ロビー | robī |
| Briefing | 概要 | gaiyō |
| Arena | 闘技場 | tōgijō |
| History | 履歴 | rireki |
| Settings | 設定 | settei |
| Agent | 戦略 | senryaku (strategy) |
| Diagnostics | 診断 | shindan |
| Score | 得点 | tokuten |
| Start / Play | 開始 | kaishi |
| Stop | 停止 | teishi |
| Game over / ended | 終了 | shūryō |
| DUT down (failure) | 故障 | koshō |

---

## 3. Spacing

### 3.1 Philosophy

**Ma: space is structure.** Spacing is strict and rhythmic, and every gap is a multiple of the 4 px base unit. Dense data lives *inside* calm, generous containers. Tight inside, open outside.

### 3.2 Scale

| Token | Value | Typical use |
| --- | --- | --- |
| `--space-1` | 4 px | Icon-to-label gap, label-to-JP-sub gap |
| `--space-2` | 8 px | Gap between tags, inline controls, tight list items |
| `--space-3` | 12 px | Input horizontal padding, gap between label and field |
| `--space-4` | 16 px | Card padding, gap between form fields, mobile gutter |
| `--space-5` | 24 px | Panel padding, gap between cards, desktop gutter |
| `--space-6` | 32 px | Gap between panels / sections within a screen |
| `--space-7` | 48 px | Gap between a screen title block and its content |
| `--space-8` | 64 px | Top padding of a screen (below shell) on desktop |
| `--space-9` | 96 px | The one "ma" space: the empty field around a hero element (PLAY splash, Game Over banner) |

No values outside this scale. If something "needs 20 px", use 16 or 24.

### 3.3 Standard component dimensions

| Component | Height | Padding | Other |
| --- | --- | --- | --- |
| Shell bar | 56 px | 0 24 px | Sticky |
| Button (default) | 40 px | 0 20 px | Icon 16 px, gap 8 px |
| Button (small) | 32 px | 0 12 px | `caption` size text |
| Button (hero: PLAY, GENERATE) | 56 px | 0 32 px | `h2` text style |
| Input / select | 40 px | 0 12 px | Label above, 8 px gap |
| Card | — | 16 px | 24 px gap between cards |
| Panel | — | 24 px | Panel title → content 16 px |
| Table row | 44 px | 0 16 px | Header row 36 px |
| Badge | 22 px | 0 8 px | `label` style text |
| Tooltip | — | 8 px 12 px | Max width 240 px |
| Dialog | — | 32 px | Width 480 px, actions row gap 12 px |

### 3.4 Layout grid

- **Desktop (≥ 1280 px):**
  - 12 columns, 24 px gutters, 24 px outer margins.
  - Content max width 1600 px, centred.
  - The Arena is full-bleed.
- **Tablet (768–1279 px):** 8 columns, 24 px gutters, 24 px margins.
- **Mobile (< 768 px):** 4 columns, 16 px gutters, 16 px margins.
- **Arena split (desktop):**
  - Cohort arena: 7 columns.
  - Agent loop panel: 5 columns.
  - Diagnostics strip: full width, 168 px high.

### 3.5 Shape

| Token | Value | Use |
| --- | --- | --- |
| `--radius-0` | 0 | Panels, the shell bar, tables |
| `--radius-1` | 2 px | Buttons, inputs, cards, badges, tooltips |
| `--radius-full` | 999 px | Status dots and toggles only |
| `--cut-s` | 6 px | Chamfer on buttons and badges |
| `--cut-l` | 14 px | Chamfer on panels and cards that are "game objects" (game cards, agent panel, score block) |

**The chamfer is the signature shape** (kire): a 45° cut on the **top-left and bottom-right corners only**.

```css
.cut-l { clip-path: polygon(14px 0, 100% 0, 100% calc(100% - 14px), calc(100% - 14px) 100%, 0 100%, 0 14px); }
.cut-s { clip-path: polygon(6px 0, 100% 0, 100% calc(100% - 6px), calc(100% - 6px) 100%, 0 100%, 0 6px); }
```

Because `clip-path` hides box-shadow and borders, chamfered elements draw their border with an inner 1 px pseudo-element or SVG outline.

Use chamfers on:
- primary and hero buttons;
- game cards;
- the score block;
- agent and arena panels;
- dialogs.

Do **not** chamfer:
- inputs;
- tables;
- tooltips;
- badges in tables.

Nothing in the app is pill-shaped except status dots and toggles.

---

## 4. Texture and atmosphere

The world is **ink, paper, light and glass**. Each effect is subtle; if you can clearly see it at a glance, it is too strong.

### 4.1 Grain (washi paper)

- **What:** a fixed full-viewport noise overlay giving the dark surfaces a faint paper/film texture.
- **Implementation:** SVG `feTurbulence` (`type="fractalNoise" baseFrequency="0.85" numOctaves="2"`) rendered once to a 256 px tile. Apply it on `body::before`:
  - `position: fixed`
  - `opacity: 0.035`
  - `mix-blend-mode: overlay`
  - `pointer-events: none`
  - `z-index: var(--z-grain)`
- **Where:** everywhere on desktop and tablet. Remove it below 768 px.

### 4.2 Gradients (light in the dojo)

- **Ambient light.**
  - Two large, very soft radial gradients sit on `--bg-0`, fixed to the viewport:
    - Top-left: `radial-gradient(1200px 800px at 0% 0%, rgba(62,230,240,0.06), transparent 60%)` (cyan)
    - Bottom-right: `radial-gradient(1000px 700px at 100% 100%, rgba(155,135,245,0.05), transparent 60%)` (wisteria)
  - These are the only full-screen gradients.
- **Panel top highlight.** Every panel has a 1 px top border built from `linear-gradient(90deg, transparent, rgba(237,234,227,0.12), transparent)`. This suggests light catching the edge.
- **No gradients on buttons, text or cards.** Flat fills only.

### 4.3 Blur (glass)

- Only floating layers use glass: dialogs, tooltips, the mobile nav drawer and Arena HUD overlays.
- Glass is `background: var(--bg-glass); backdrop-filter: blur(16px) saturate(120%); border: 1px solid var(--line-1);`.
- Dialog backdrops are `rgba(7, 8, 12, 0.6)` with `blur(4px)`.
- Never use glass on base panels or cards. Base surfaces are solid.

### 4.4 Pattern (seigaiha 青海波)

- A seigaiha (wave) pattern, as an inline SVG tile 48 × 24 px, stroked 1 px in `--text-1` at **3% opacity**.
- **Allowed in exactly three places:**
  1. The Lobby background behind the title block (masked to fade out by 50% of the height).
  2. The inside of the hanko seal border.
  3. The Game Over background.
- Never use it behind data, tables or text blocks.

### 4.5 Depth (elevation)

Depth comes from **layer colour plus edge light**, not big drop shadows.

| Level | Surface | Shadow |
| --- | --- | --- |
| 0 | `--bg-0` page | none |
| 1 | `--bg-1` panels | `0 1px 0 rgba(237,234,227,0.04) inset` |
| 2 | `--bg-2` cards | `0 8px 24px rgba(0,0,0,0.35)` |
| 3 | Glass: dialogs, tooltips | `0 24px 64px rgba(0,0,0,0.55)` |

### 4.6 Glow (light as state)

- Glow is used **only to signal state**: focus, the active selection, the running DUT and the reward pulse.
- Glow is always `box-shadow: 0 0 0 1px <accent>, 0 0 16px <accent at 40%>`.
- At most **two glowing elements** at any moment outside the Arena canvas.

### 4.7 Signature elements

| Element | Spec |
| --- | --- |
| **Hanko seal** (failure) | 96 × 96 px square, rotated −6°, 3 px `--accent-shu` border, `jp-seal` text 故障 in `--accent-shu` on a 10% shu tint, with the seigaiha pattern inside the border. Stamps onto the Game Over screen. |
| **Blade line** | A 1 px horizontal line in `--accent-primary`, 48 px wide, placed under every `h1` with 12 px gap. It is the visual "cut" that starts each screen. |
| **Corner brackets** | On the cohort arena and score block: four 12 px L-shaped corner marks in `--line-2` (HUD framing). No full border. |
| **Status dot** | 8 px circle, solid accent, with a 2 s soft pulse ring only while `RUNNING`. |

### 4.8 Iconography

- Line icons from **Lucide**: 1.5 px stroke, 16 px in UI and 20 px in nav. Colour is inherited from the text colour.
- Never use filled icons except status dots and the hanko.
- UE and DUT sprites are custom geometric SVGs (shapes in §1.5) with no detail. They read as minimal game pieces, not cartoons.

---

## 5. Interactions

### 5.1 Emotional quality of movement

Movement is **calm, precise and decisive**. Think of a sword drawn and sheathed: almost no anticipation, a fast confident stroke, then complete stillness.

- **Avoid:** bounce, overshoot, elastic springs, wobble, rotation (except the hanko stamp), and anything "cute".
- **Interface motion** is quick and quiet. **Game motion** (score, failure, escalation) is where drama is allowed, and it follows jo-ha-kyū.

### 5.2 Timing tokens

| Token | Value | Use |
| --- | --- | --- |
| `--dur-instant` | 80 ms | Press/active feedback, colour change on small controls |
| `--dur-fast` | 160 ms | Hover, focus, toggles, tooltips in |
| `--dur-base` | 240 ms | Route changes, dialogs, panels expanding, cards selecting |
| `--dur-slow` | 400 ms | Score count-up, escalation flash, staggered lists |
| `--dur-epic` | 800 ms | PLAY splash, hanko stamp, Game Over entrance |

Rules:
- Exits are always **shorter** than entrances (~70% of the entrance duration).
- Nothing in the interface exceeds 400 ms except the three `--dur-epic` moments.

### 5.3 Easing tokens

| Token | Curve | Character | Use |
| --- | --- | --- | --- |
| `--ease-out` | `cubic-bezier(0.16, 1, 0.3, 1)` | Fast start, long glide to rest | **Default** for everything that enters or responds to the user |
| `--ease-in` | `cubic-bezier(0.7, 0, 0.84, 0)` | Accelerates away | Exits only |
| `--ease-inout` | `cubic-bezier(0.65, 0, 0.35, 1)` | Even | Moving an element between two positions (tabs, sliders) |
| `--ease-jhk` | `cubic-bezier(0.83, 0, 0.17, 1)` | Slow, then sudden, then stop (jo-ha-kyū) | `--dur-epic` moments only |
| `linear` | — | — | Only for continuous loops (pulse rings, progress, ticking) |

### 5.4 Hover and press behaviour by element

Hover applies only under `@media (hover: hover)`. `:focus-visible` always shows the hover state **plus** the focus ring.

| Element | Hover (`--dur-fast`, `--ease-out`) | Press (`--dur-instant`) | Selected / active |
| --- | --- | --- | --- |
| Primary button (solid cyan) | Glow appears; fill lightens 6% | `translateY(1px)`; glow removed | — |
| Secondary button (outline) | Border `--line-2` → `--accent-primary`; 8% cyan tint fill | `translateY(1px)` | — |
| Ghost button / text link | **Blade sweep:** a 1 px `--accent-primary` underline grows from left to right (`scaleX 0 → 1`, origin left) | Text to `--accent-primary` | — |
| STOP button | Border and text stay shu; 8% shu tint fill appears | `translateY(1px)` | — |
| Game card | Border → `--accent-primary` at 60%; card lifts `translateY(-2px)`; JP sub-label fades from `--text-3` to `--text-2` | `translateY(0)` | 1 px cyan border + glow; 14% cyan tint; a 6 px cyan chamfer marker fills the top-left cut |
| Nav item | Text `--text-2` → `--text-1`; blade sweep underline at 40% opacity | — | Text `--text-1`; full blade underline in cyan, 2 px |
| Table row | Fill → `--bg-3` | — | 2 px left bar in cyan |
| Segmented control | Text → `--text-1` | — | Sliding cyan 14% tint indicator moves under the option (`--dur-base`, `--ease-inout`) |
| Input | Border → `--line-2` | — | Focus: border cyan + glow |
| Cohort tile / KPI | Tooltip after 300 ms delay; tile brightness +15% | — | — |
| Icon button | Icon colour → `--text-1`; 8% white tint square behind | Scale 0.96 | — |

- **Cursor:** `pointer` on all interactive elements; `not-allowed` on disabled.
- **Disabled:** 40% opacity, no hover, no glow.
- **Focus ring:** `outline: 2px solid var(--accent-primary); outline-offset: 2px;`. It follows the chamfer using an SVG outline on chamfered elements. It is never removed.

### 5.5 Signature motion moments

| Moment | Choreography |
| --- | --- |
| **Route change** | Outgoing screen fades out (160 ms, `--ease-in`). Incoming fades in and rises 8 px (240 ms, `--ease-out`). The blade line under `h1` draws left to right (240 ms, 80 ms delay). |
| **List / grid entrance** | Items fade in and rise 8 px, staggered 40 ms each, max 8 items staggered; the rest appear together. |
| **PLAY splash** (`--dur-epic`, `--ease-jhk`) | Screen dims → game ID ("PLAY 042") appears small and faint (jo), holds 200 ms, then snaps to full size and brightness (ha) → a cyan blade line cuts across the full width in 120 ms (kyū) → splash exits and the Arena is live. |
| **Score update** | Number counts up over 400 ms (`--ease-out`). The reward value flashes from gold to `--text-1` over 400 ms. No scale bump. |
| **Escalation** | Arena corner brackets snap 4 px outward and back (240 ms). State badge switches to `ESCALATING` for 2 s. A single cyan scan line passes top-to-bottom across the arena (400 ms, linear). |
| **KPI breach** | Value and sparkline turn shu over 160 ms. One horizontal shake of 2 px, 2 cycles, 160 ms total. |
| **DUT failure** (`--dur-epic`, `--ease-jhk`) | Ticking stops. All sprites desaturate to 30% over 400 ms. The arena frame turns shu. The **hanko seal stamps in**: it starts 1.3× size at 0 opacity, then lands at 1× with −6° rotation in 160 ms, with a 1-frame 2 px screen offset (the "impact"). After 1.2 s it routes to Game Over. |
| **Dialog** | Backdrop fades in 160 ms. Panel fades and scales 0.97 → 1 over 240 ms, `--ease-out`. Closing is 160 ms, `--ease-in`. |
| **Tooltip** | 300 ms delay, then fades in and moves 4 px toward its anchor over 160 ms. Hides instantly. |
| **Running status dot** | A pulse ring scales 1 → 2.2 and fades to 0, 2 s loop, linear. |

### 5.6 Reduced motion

Under `@media (prefers-reduced-motion: reduce)`:
- Every transform-based motion becomes an opacity fade of ≤ 160 ms, or is removed. This covers lifts, rises, shakes, sweeps, the splash, the scan line, the stamp and the pulse rings.
- The score jumps to its value.
- The hanko simply appears.
- All colour and state changes stay.

### 5.7 Performance rules

- Animate only `transform`, `opacity`, `filter` (blur on enter only) and colour properties.
- Never animate `width`, `height`, `top`, `left`, `box-shadow` size or `clip-path`. A glow appears by fading in a pseudo-element.
- Keep at most 3 concurrent CSS animations outside the Arena canvas.

---

## 6. Token summary (copy into `tokens.css`)

```css
:root {
  /* backgrounds */
  --bg-0:#07080C; --bg-1:#0D1017; --bg-2:#141925; --bg-3:#1C2333; --bg-glass:rgba(20,25,37,.72);
  /* lines */
  --line-1:#222A3A; --line-2:#34405A; --line-hair:rgba(237,234,227,.08);
  /* text */
  --text-1:#EDEAE3; --text-2:#A3ABBA; --text-3:#6B7487; --text-inverse:#07080C;
  /* accents */
  --accent-primary:#3EE6F0; --accent-shu:#F2462E; --accent-kin:#E8C25A;
  --accent-hisui:#4ADE9A; --accent-fuji:#9B87F5; --accent-warn:#F5A524;
  /* data */
  --ue-urllc:#FF5FA2; --ue-embb:#3EB8F0; --ue-mmtc:#F5A524;
  /* type */
  --font-display:"Chakra Petch","Rajdhani",system-ui,sans-serif;
  --font-ui:"Inter",system-ui,-apple-system,"Segoe UI",sans-serif;
  --font-jp:"Zen Kaku Gothic New","Hiragino Sans","Noto Sans JP",sans-serif;
  --font-mono:"JetBrains Mono",ui-monospace,monospace;
  /* spacing */
  --space-1:4px; --space-2:8px; --space-3:12px; --space-4:16px; --space-5:24px;
  --space-6:32px; --space-7:48px; --space-8:64px; --space-9:96px;
  /* shape */
  --radius-0:0; --radius-1:2px; --radius-full:999px; --cut-s:6px; --cut-l:14px;
  /* motion */
  --dur-instant:80ms; --dur-fast:160ms; --dur-base:240ms; --dur-slow:400ms; --dur-epic:800ms;
  --ease-out:cubic-bezier(.16,1,.3,1); --ease-in:cubic-bezier(.7,0,.84,0);
  --ease-inout:cubic-bezier(.65,0,.35,1); --ease-jhk:cubic-bezier(.83,0,.17,1);
  /* layers */
  --z-base:0; --z-sticky:100; --z-overlay:200; --z-dialog:300; --z-tooltip:400; --z-grain:900;
}
```

---

## 7. Decision checklist

Run this before merging any visual change:

- [ ] Every colour, size, gap, radius, duration and easing comes from a token.
- [ ] Vermilion appears only for stop, failure or breach.
- [ ] Only one solid primary button is visible in the view.
- [ ] Live numbers use mono with tabular numerals.
- [ ] Uppercase text is tracked as specified.
- [ ] Any Japanese text comes from the fixed vocabulary, is paired with English and is `aria-hidden`.
- [ ] No bounce or overshoot. Exits are faster than entrances.
- [ ] Reduced-motion behaviour is defined.
- [ ] Text contrast is ≥ 4.5:1, and `--text-3` isn't used for anything essential.
- [ ] The change still looks right on mobile (grain off, type steps down, 44 px tap targets).
