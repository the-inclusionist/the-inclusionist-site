# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML page: [minigames-catalog-v2.html](minigames-catalog-v2.html) — a pt-BR catalog of 35 JavaScript minigame genres (~380 subgenres) used to help someone pick a genre before writing a "build me this game" prompt. Content is in Brazilian Portuguese; keep it that way.

There is no build step, no JavaScript, no package manager, no test suite, and no git repo here. The whole deliverable is that one file: markup plus an inline `<style>` block. The only network dependency is the Google Fonts stylesheet (`Bungee`, `Bungee Shade`, `JetBrains Mono`, `Major Mono Display`) — the "0 dependências" claim in the hero refers to runtime/build deps, not that link.

## Working on it

Open the file directly in a browser to view; there is nothing to serve. To verify a change with the Browser pane, use `preview_start` with a `file:///C:/Users/candi/Claude/SP-the-inclusionist-demos/minigames-catalog-v2.html` URL rather than starting a dev server.

Edit the file in place. Do not introduce a bundler, framework, or external asset — single-file, offline-openable is the point of the artifact.

## Card contract

Every genre is one `<article class="card cat-N">` in `.grid`, and the CSS depends on that shape:

- `cat-N` (N = 1..35) is the only thing that sets `--accent` (see the `.card.cat-N` block near the end of the stylesheet). `--accent` drives the left rail, title color, `▸` bullets, and the `NEW` badge. A card without a matching `cat-N` rule silently falls back to `--c1`.
- `.card-num` holds `NN / 35` — the denominator is hardcoded in all 35 cards.
- `.density-dots[data-level="leve|medio|denso"]` colors 1, 2, or 3 dots via `:nth-child`. The three literal `<span class="dot">` children must always be present; the level string alone controls how many light up. The adjacent text label (`Densidade: Leve`) is a separate hardcoded string — keep it in sync with `data-level`.
- `<li class="fresh">` swaps the bullet to a yellow `+`, marking a subgenre added in v2. `.new-badge` inside `.card-num` marks a whole category added in v2 (currently cards 31–35).

## Invariants to update together

Adding or removing a category touches more than the new `<article>`. Sweep all of these:

1. A new `--cN` token in `:root` and a matching `.card.cat-N { --accent: var(--cN); }` rule.
2. The `NN / 35` denominator in **every** card, plus the sequential numbering.
3. `.section-head .count` (`001 → 035`).
4. `.hero-stats` — the category count, subgenre count, and "novas em vN" figures are hand-written and drift easily.
5. The version string, which appears in five places: `<title>`, the `:root` comment, `.hero-tag .ver`, the `.changelog` block, and the footer.
6. The `.changelog` body prose, which names the new categories explicitly.

## Style conventions

- Section boundaries in the stylesheet use `/* ────────── NAME ────────── */`; markup sections use `<!-- NAME ─────── -->`. Card blocks are preceded by `<!-- NN -->` (or `<!-- NN NEW -->`).
- Terminal/neon aesthetic: dark tokens only (no light-theme branch), monospace body text, uppercase letter-spaced micro-labels, `//` prefixes on tag/eyebrow text.
- Subgenre entries are `Nome <small>qualificador em minúsculas</small>` — the `<small>` is an optional one-line hint, lowercase, no trailing period.
