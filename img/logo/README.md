# Inclusionista logo — SVG set

Extracted from the brandbook canvas (`Inclusionista-Marca.dc.html`), section **02 · Logo**.

⚠️ **That canvas no longer exists here.** `research/` was deleted and gitignored — the canvases are
design fiction with unverified curriculum data, and must not ship to a public address. They survive in
the pushed history, in `fa4ec2e`.

So **these SVGs are now the artifact**, not a copy of one. There is no upstream left to re-derive them
from without going into the history, which means a change here is a change to the mark itself.

## Files

| File | Application | Canvas size |
|---|---|---|
| `symbol.svg` | The symbol alone — app icon, favicon, avatar, game corner | 64 × 64 |
| `lockup-horizontal-on-dark.svg` | Default signature, symbol left of the wordmark | 64 px symbol |
| `lockup-horizontal-on-light.svg` | Same, for light surfaces | 64 px symbol |
| `lockup-vertical-on-dark.svg` | Square and narrow formats — title screen, poster, sticker, shirt | 96 px symbol |
| `lockup-vertical-on-light.svg` | Same, for light surfaces | 96 px symbol |

The symbol has **one** file, not two. Its trio — green `#0B7A46`, yellow `#FFDD1C`, blue `#2E5BFF` —
is exclusive to the symbol and does not change with the theme, because the contrast that matters
there is between the three colours, not against the page.

Only the wordmark changes with the surface: `#F2F4F8` with the `ista` suffix in `#FFDD1C` on dark,
`#11131A` with the suffix in `#8A6A00` on light.

## No font dependency

The wordmark is Jersey 15 converted to outlines, so these files render identically wherever they
are opened — no webfont, no `@font-face`, no fallback that silently substitutes another face. The
cost is the usual one: **the text is no longer text.** To change the wording, regenerate rather than
edit the path data.

## Construction rules, carried over from the brandbook

- **Grid** — everything in multiples of 8 px. No diagonals, no corner radius on the symbol.
- **Clear space** — at least 1 module (⅛ of the symbol height) on every side. It is *not* baked
  into these files; the artwork is tight-bounded, so reserve the space in the layout.
- **Gaps** — horizontal lockup: ¼ of the symbol height between symbol and wordmark. Vertical
  lockup: ⅙ of the symbol height. Both are already applied here, measured from the ink rather
  than from the glyph side bearing.
- **Minimum sizes** — symbol 16 px; horizontal lockup 120 px wide; vertical lockup 72 px.
  Below 20 px of wordmark body the wordmark comes off and only the symbol stays.
- **Never** — rotate, apply a gradient or shadow, split the cross from the circle, recolour the
  blue core, use the three colours outside the symbol, add the directional to the wordmark, or
  place the mark over a photo without a contrast box.

## Licence

These files are **art, not program.** They are not covered by the repository's `LICENSE`
(AGPL-3.0-or-later, which governs code): a visual identity falls under Lei 9.610 and, as a mark,
under trademark rules. What governs what is recorded in `docs/LICENSES.md` on the engine.
Do not add an SPDX code-licence header to these files.
