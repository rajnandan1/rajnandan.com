# Design — rajnandan.com

A locked design system for this site. Every page redesign reads this file before
emitting code. Do not regenerate per page — extend or amend this file when the
system needs to grow.

Implementation note: this is a Zola site using the Serene theme. The design
layer lives in `templates/_custom_css.html` (inline `<style>` override) and
`templates/_custom_font.html`. Tokens are declared on `body` / `body.dark` and
mapped onto Serene's theme API variables (`--primary-color`, `--bg-color`, …).
`tokens.css` at the project root is the portable copy of the token block.

## Genre

editorial

## Voice

Ink & paper broadsheet. Near-white cool newsprint stock, near-black warm ink,
one oxblood accent used at ≤ 3 % of any viewport. Hairline rules and classic
`double` rules carry the structure. Dates, tags, and the @handle are set in
mono like archive metadata. No gradients, no glass, no shadows-as-depth.

## Macrostructure family

- Home:          Index-First with an N6 newspaper-masthead head (centred name in
                 display serif over a double rule, dateline bio, link row) and a
                 dotted-leader recent-writing index.
- Blog index:    Index-First — section title over a double rule, dotted-leader
                 rows, mono dates right-aligned.
- Post pages:    Long Document — display-serif headline, mono dateline, single
                 hairline between meta and body, fleuron (❦) as `hr`.
- Content pages: Long Document (projects, books, contact, support) — typography
                 only.

## Theme

Light ("day edition"):

- `--color-paper`        oklch(97.5% 0.005 80)
- `--color-paper-2`      oklch(94.6% 0.007 80)
- `--color-paper-raised` oklch(98.8% 0.004 80)
- `--color-ink`          oklch(21% 0.012 60)
- `--color-ink-2`        oklch(44% 0.014 70)
- `--color-rule`         oklch(85.5% 0.009 80)
- `--color-rule-strong`  oklch(38% 0.012 70)
- `--color-accent`       oklch(45% 0.14 25)   ← oxblood
- `--color-focus`        oklch(50% 0.15 25)

Dark ("night edition") — same hues, lightness/chroma only:

- `--color-paper`        oklch(17% 0.01 70)
- `--color-paper-2`      oklch(21% 0.012 72)
- `--color-paper-raised` oklch(23% 0.012 72)
- `--color-ink`          oklch(92% 0.008 80)
- `--color-ink-2`        oklch(71% 0.012 78)
- `--color-rule`         oklch(33% 0.012 75)
- `--color-rule-strong`  oklch(76% 0.01 78)
- `--color-accent`       oklch(64% 0.12 27)
- `--color-focus`        oklch(68% 0.13 27)

## Typography

- Display: DM Serif Display, weight 400, style normal (roman only — italic
  headers are banned). Headlines, masthead name, section titles, prose h2.
- Body:    Source Serif 4 (opsz 8–60, wght 300–700), weight 400; italic for
  body-copy emphasis only.
- Mono:    IBM Plex Mono, weights 400/500. Outlier — exactly two roles:
  (1) code, (2) metadata (dates, tags, @handle). Never a third.
- Base size 18px · line-height 1.72 · display tracking uppercase +0.015em,
  labels +0.10–0.14em.
- Type scale anchor: `--text-display: clamp(2.3rem, 6vw, 3.9rem)`.
- Labels/eyebrows are uppercase letter-spaced Source Serif — used only for
  genuine labels (category names, the "Recent writing" index head), never as
  decorative section eyebrows. Tag-left/heading-right is banned.

## Spacing

4-point named scale, declared in the token block (`--space-2xs` … `--space-2xl`).
Pages use named tokens, never raw values. Measures: `--main-max-width: 700px`
(≈ 66ch at 18px serif), `--homepage-max-width: 720px`.

## Motion

- Easings: `--ease-out: cubic-bezier(0.16, 1, 0.3, 1)`,
  `--ease-in: cubic-bezier(0.7, 0, 0.84, 0)`.
- Durations: `--dur-micro: 120ms`, `--dur-short: 220ms`, `--dur-long: 420ms`.
- Reveal pattern: one orchestrated entrance on page load (rise-in, staggered
  ≤ 500 ms total), gated behind `prefers-reduced-motion: no-preference`.
  Nothing animates on scroll.
- Reduced-motion fallback: entrance does not run; transitions are colour-only
  ≤ 150 ms.

## Microinteractions stance

- Silent success; no toasts anywhere on this site.
- Hover signals are single: colour shift or underline thickening, never
  transform + shadow + colour together.
- Focus rings: `2px solid var(--color-focus)`, offset 3px, instant (outline is
  never in a transition list).

## CTA voice

This site has no buttons-as-CTAs. Actions are typographic links:
underline with `text-decoration-color: var(--color-accent)`, offset 3px,
thickness 1px → 2px + accent text on hover. The read-more link ("more posts »")
is uppercase letter-spaced small text.

## Per-page allowances

- All pages: typography only. No enrichment tiers, no illustrations, no
  hero imagery. The avatar renders as a small print-style portrait
  (square, hairline border, grayscale).
- Post pages MAY use inline images sized to the text measure with a hairline
  border and 6px radius.

## What pages MUST share

- The masthead voice (display serif name/wordmark, double rules).
- The oxblood accent and its placement (links, focus, marks — ≤ 3 % viewport).
- The three faces and their roles (display/body/mono-metadata).
- The dotted-leader index row voice for any list of dated items.
- The footer: Ft2 inline rule — hairline-strong top rule, single line,
  uppercase small copyright, square hairline theme-toggle.

## What pages MAY differ on

- Index-First pages may group rows under category labels; Long Document pages
  never show category labels.
- Post pages carry the mono dateline + tags block; content pages carry only a
  section title + subtitle.

## Exports

### tokens.css

See `tokens.css` at the project root — the canonical portable token block
(light + dark), kept in sync with `templates/_custom_css.html`.
