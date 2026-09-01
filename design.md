# Design — rajnandan.com

A locked design system for this site. Every page redesign reads this file before
emitting code. Do not regenerate per page — extend or amend this file when the
system needs to grow.

Implementation: this is a Zola site using the Serene theme. The design layer
lives in `templates/_custom_css.html` (inline `<style>` override) and
`templates/_custom_font.html`. Tokens are declared on `body` / `body.dark` and
mapped onto Serene's theme API variables (`--primary-color`, `--bg-color`, …).
`tokens.css` at the project root is the portable copy of the token block.

## System

**Kami (紙)**, the design language from the `kami` Claude Code plugin.
Upstream spec: `~/.claude/plugins/cache/kami/kami/<version>/skills/kami/references/design.md`.

Its one-sentence compression: *warm parchment canvas, ink-blue accent, serif
carries hierarchy, avoid cool grays and hard shadows.*

## The invariants this site holds

1. Page ground is parchment `#f5f4ed`, never white.
2. One chromatic colour: ink blue `#1b365d`, kept under ~5% of any viewport.
3. Every gray is warm (R ≥ G > B). No cool blue-grays anywhere.
4. One serif family carries headings **and** body. A system sans (`--font-ui`)
   appears only as genuine chrome: overlines, nav labels, dates, tags, footer.
5. Serif runs at two weights only — 400 body, 500 emphasis and headings. No
   700, no synthetic bold (`font-synthesis-weight: none`).
6. Line-heights: headlines 1.1–1.3, reading body 1.55.
7. Letter-spacing is 0 on body. Tracking is for small uppercase overlines only
   (+0.4px).
8. Surfaces are flat. No shadows, no gradients, no lift on hover.
9. A line ships only when it separates content regions, encodes state, or
   carries a data relationship. Decorative rules, ticks, side bars and fleurons
   do not.
10. Sizes land on the ladder, never between its steps.

## Voice

Editorial, quiet, printed. The page reads like a well-set report: the type does
the work, the paper does the rest. Dates, tags and nav labels sit in a small
uppercase system sans, the way a running head sits above a page of prose. Code
is the one place the page goes dark, and it stays dark in both editions.

## Macrostructure

- **Home** — Kami hero, stacked. Eyebrow row (section labels left, social
  icons right), then the mark, the display name, and the title in olive, each
  on its own line so the display line always gets the full measure. Then the
  prose brief, a closing hairline, and the writing index.
- **Blog / projects / books / contact / support** — section head (serif title,
  olive lede, closing hairline) over the page's own content.
- **Post pages** — long document. Serif headline, metadata line (date left,
  tags right) closed by a hairline, prose body, marginal TOC rail.
- **Index rows** — title left, date right, one hairline per row. No leaders.

## Colour

Light ("day edition"):

| Token | Value | Role |
|---|---|---|
| `--color-bg` | `#f5f4ed` | parchment · page ground |
| `--color-surface` | `#faf9f5` | ivory · quiet filled container |
| `--color-code-bg` | `#f0eee6` | inline annotation fill |
| `--color-control` | `#e8e6dc` | warm sand · interactive surface |
| `--color-text` | `#141413` | near black · primary |
| `--color-text-2` | `#3d3d3a` | dark warm · secondary |
| `--color-text-3` | `#504e49` | olive · subtext, ledes, quotes |
| `--color-text-4` | `#6b6a64` | stone · metadata, labels |
| `--color-border` | `#e8e6dc` | region separators |
| `--color-border-soft` | `#e5e3d8` | row separators |
| `--color-brand` | `#1b365d` | ink blue · the only chromatic colour |
| `--color-brand-hover` | `#2d5a8a` | link hover |
| `--color-brand-tint` | `#eef2f7` | lightest fill |
| `--color-tag-bg` | `#e4ecf5` | tag swatch, selection, mark |
| `--color-warn` | `#8b4513` | the one sanctioned warm accent |
| `--color-warn-bg` | `#f0e0d8` | its fill |

Dark ("night edition") — same roles, warm charcoals:

`--color-bg #141413` · `--color-surface #1c1b19` · `--color-code-bg #23221f` ·
`--color-control #30302e` · `--color-text #faf9f5` · `--color-text-2 #d4d1c8` ·
`--color-text-3 #b0aea5` · `--color-text-4 #8f8d85` · `--color-border #302f2b` ·
`--color-border-soft #262521` · `--color-brand #7fa6ce` ·
`--color-brand-hover #a3c1e0` · `--color-brand-tint`/`--color-tag-bg #1e2a38` ·
`--color-warn #d9a273` · `--color-warn-bg #2c231c`.

In the night edition headings stay ivory and running text steps down to
`--color-text-2`, so a long article does not glare.

### Code surface

One palette, both editions. Day: `#141318` (Kami's screen code/gallery frame) —
a dark block on parchment. Night: `#23221f`, one warm step *up* from the page,
because `#141318` against a `#141413` ground stops reading as a container.

Six tokens, per Kami: comment `#948e80` · keyword `#84aad6` · string `#8cbb91` ·
number `#cbab86` · function `#d6c78c` · builtin `#b59ccd`. Base text `#d4d1c8`,
deleted-diff `#c08a7d`. Everything without a `language-*` class stays
monochrome.

## Typography

- **Serif** — `"Source Serif 4", Charter, Georgia, Palatino, "Times New Roman"`.
  Weights 400/500, italic 400 for body emphasis only.
- **UI sans** — `system-ui` stack. Chrome only: overlines, nav, dates, tags,
  footer, TOC, figure captions.
- **Mono** — `"JetBrains Mono", "SF Mono", "Fira Code", Consolas, Monaco`. Code
  only.

Ladder (screen px), and nothing between the steps:

| Token | Size | Use |
|---|---|---|
| `--text-label` | 12 | uppercase overlines, nav, tags |
| `--text-meta` | 13 | dates, footer, back-to-top |
| `--text-small` | 14 | code blocks, TOC, captions |
| `--text-ui` | 15 | index rows, tables, section lede |
| `--text-body` | 18 | reading body (16 below 768px) |
| `--text-lede` | 21 | tagline, opening paragraph |
| `--text-h3` | 22 | prose h3 |
| `--text-h2` | 30 | prose h2 |
| `--text-h1` | clamp(30, 5vw, 36) | post and section titles |
| `--text-display` | clamp(30, 6vw, 56) | the name on the home page |

Measures: `--main-max-width: 720px` (Kami's docs prose measure),
`--homepage-max-width: 760px`.

## Spacing

4px base, named tokens only: `--space-xs` 4 · `--space-sm` 8 · `--space-md` 16 ·
`--space-lg` 24 · `--space-xl` 40 · `--space-2xl` 64 · `--space-3xl` 96.

Proximity law: the gap under a section head is at least 2× smaller than the gap
above it. A region's first block carries no top margin of its own.

Radii: `--radius-sm` 2px (inline code) · `--radius-md` 4px (tags) ·
`--radius-lg` 8px (code blocks, images, cards, controls) · `--radius-pill` 999px
(reserved; no pill ships yet).

## Components

- **Tag** — `--color-tag-bg` fill, brand text, 12px uppercase 600, +0.4px
  tracking, 4px radius. Used for post tags and the `Featured` state chip.
- **Card** — ivory fill, one `--color-border` hairline, 8px radius, flat on
  hover. `details` blocks only.
- **Table** — no frame, no vertical rules, no tinted header. Hairline row rules
  on `--color-border-soft`, header rule on `--color-border`, header set in the
  uppercase UI sans. Scrolls inside its own box.
- **Quote** — indentation plus olive text. No side rule, no italic.
- **Rule** — full-width `--color-border` hairline, aligned to the 15px content
  gutter so every rule on the page shares one left and right edge.
- **Ghost control** — theme toggle, RSS, back-to-top: transparent, one border
  hairline, 8px radius, brand on hover.

## Motion

`--dur: 150ms` on colour and border transitions. Nothing else animates. There is
no page-load entrance and no scroll animation: the name should be readable the
instant it paints. Every `:hover` rule is wrapped in `@media (hover: hover)` so
a tap never flashes a hover state.

## Deliberate deviations from upstream Kami

Each one is a Kami rule bent for a reason, not an oversight.

1. **Serif is Source Serif 4 first, Charter/Georgia after.** Invariant 5 needs a
   real 500. Charter and Georgia ship 400 and 700 only, so `font-weight: 500`
   would resolve to 400 and headings would lose their one allowed step. Source
   Serif 4 is a variable face with a true 400–500 range; the Kami stack stays
   behind it as fallback.
2. **Body links keep an underline**, and clear Serene's `border-bottom`. Kami's site-wide contract is brand colour
   with no underline. Ink blue against near-black is 1.5:1 — under the 3:1 a
   colour-only affordance requires — so prose links carry a 40%-brand underline.
   Chrome links, where position identifies them, follow Kami exactly.
3. **Dark-mode ink blue is `#7fa6ce`, not `#2d5a8a`.** Kami's "links on dark"
   value reaches 2.6:1 on `#141413`. `#7fa6ce` reaches 7.3:1.
4. **Code comments are `#948e80`, not `#79756a`.** Kami's value lands under 4:1
   on its own code surface.
5. **Italic stays live in prose.** Kami bans italic in print templates and
   allows it on screen; this is a screen surface and article copy uses `<em>`
   for real emphasis. Chrome carries no italic.

## Standalone pages

`static/demos/lttb/index.html` is a self-contained interactive demo linked from
the LTTB article. It carries its own copy of the token block under the same
names, because it ships as one file with no build step. When the palette here
changes, change it there too. Its chart series follow Kami's data palette:
primary `#1b365d`, secondary `#504e49`, context in warm gray; the canvas reads
those through the `--chart-*` variables.

## Known gaps

- The five nav labels hold one line down to 375px and wrap at 320px. Five items
  cannot fit 320px without going under the 12px web floor.
- `config.toml`'s `description` still reads "Director of Engineering at Cashfree
  Payments". The home prose puts Cashfree in the past and Crustdata in the
  present, so the two disagree. That string is the site-wide meta and og
  description on every page.
