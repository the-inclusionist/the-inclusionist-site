# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read this first

**[docs/superpowers/specs/2026-08-25-inclusionist-demos-design.md](docs/superpowers/specs/2026-08-25-inclusionist-demos-design.md) is the authority.** It holds the decision table (D1–D12), the
architecture, what is lifted from the tracer, the phasing and the non-goals. This file is the entry
rules and the pointers; do not duplicate the spec here — a duplicated fact rots.

## What this is

A collection of JS minigames in 320×180 pixel art: one playable game for each of the 383 subgenres
listed in the catalog page, each in its own folder, linked from its catalog entry. The stack and
conventions come from `SP-the-inclusionist-tracer` (sibling directory, GitLab `jrocha-dev/the-inclusionist`).

**State as of 2026-08-25: Phase 0 is done, Phase 1 has not started.** The repository holds the
catalog page and the spec. There is no `package.json`, no engine, no game, and therefore **no build,
test or lint command yet** — do not invent one. Phase 1 creates the toolchain (Node 24, TypeScript,
Vite, Vitest, PixiJS 7.4.2, `vite-plugin-pwa`), mirroring the tracer's versions.

## Conventions

- **English for every artifact** — docs, code, comments, commit messages. The conversation with the
  Dev is pt-BR. Exception: the catalog page's own content (category names, subgenre names and hints)
  stays pt-BR, because it is the product.
- **GPL-3.0-or-later**, with an SPDX header on every source file, as in the tracer.
- **Atomic, frequent commits** in English. Never one giant "initial". No absolute paths in versioned files.
- Decisions worth keeping go into the spec in the same turn they are made, not left as chat prose.

## Two inherited conventions that are easy to break

- **`dt` is counted in frames, not seconds.** Physics copied from a seconds-based tutorial runs wrong.
- **The keyboard is listened to on `#game-region`, not on `window`.** On `window` the game steals the
  page's keys.

## The catalog page

[minigames-catalog-v2.html](minigames-catalog-v2.html) is currently the only content: a self-contained
pt-BR page, 35 categories and 383 subgenres, dark neon terminal aesthetic, no JavaScript. Its only
network dependency is the Google Fonts stylesheet — the "0 dependências" claim in the hero is about
runtime and build deps, not that link.

In Phase 1 this file is split (prose and CSS into `src/catalog.template.html`, data into
`data/catalog.json`) and the root page becomes generated output. Until then, **the markup semantics
below are what `import-catalog.mts` must preserve**, so they are worth knowing before touching it:

- `article.card.cat-N` — `cat-N` is the only thing that sets `--accent`, which drives the left rail,
  the title color, the `▸` bullets and the `NEW` badge. A card without a matching `.card.cat-N` rule
  silently falls back to `--c1`.
- `.card-num` holds `NN / 35`; the denominator is hardcoded in all 35 cards.
- `.density-dots[data-level="leve|medio|denso"]` lights 1, 2 or 3 dots via `:nth-child`. All three
  `<span class="dot">` children must always be present. The adjacent `Densidade: …` label is a
  separate hardcoded string that must agree with `data-level`.
- `li.fresh` renders a yellow `+` bullet (subgenre added in v2); `.new-badge` marks a whole category
  added in v2 (cards 31–35).
- Subgenre entries read `Nome <small>qualificador em minúsculas</small>`, no trailing period.

Editing it by hand means sweeping every invariant at once: the `--cN` token and its `.card.cat-N`
rule, the `NN / 35` denominator in all 35 cards, `.section-head .count`, the five hand-written hero
statistics, the version string in five places, and the changelog prose. That fragility is the reason
for D8 — the generator retires all of it.

## Style of the page

Stylesheet sections are marked `/* ────────── NAME ────────── */`, markup sections `<!-- NAME ─── -->`,
and card blocks are preceded by `<!-- NN -->` (or `<!-- NN NEW -->`). Dark tokens only, no light
theme branch, monospace body text, uppercase letter-spaced micro-labels, `//` prefixes on eyebrow text.
