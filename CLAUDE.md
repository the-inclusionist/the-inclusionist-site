# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read this first

**The records are the authority, and they are not here.** They live in `the-inclusionist-docs`, under
`docs/2-Architecture/adr/` — 143 of them today. There is **no `adr/` folder in this repository**, and that
is itself a decision (**ADR-0068 §5**, **ADR-0123**): three hundred places to decide architecture are three
hundred places where one decision silently contradicts another. Before deciding anything structural here,
read the record; before writing a decision down, write it there.

⚠️ **`docs/superpowers/specs/…-inclusionist-demos-design.md` is history, not the authority.** It designs
the superseded role — one game per catalog entry, 383 subgenres, this repository as the unit of delivery.
**ADR-0068** ended that. It is kept for one reason: its decision table **D1–D16** is cited by name from ten
records, so deleting it would strand those citations. Read it to learn *why* a rule exists, never what to
build; where it and a record disagree, the record wins. Its companion phase-1 plan was **deleted** — no
record cited it, and 7210 lines of build steps for a cancelled phase are something a future session runs by
mistake.

## What this is

Two things, both under **ADR-0068**:

1. **The manifest** — which games, at which versions, go into a delivery. ⚠️ **It does not exist yet.** Until
   it does, this repository describes a role it does not yet perform.
2. **The public surface** — the definitive demonstration address, which will hold everything the official
   site will hold. That part exists: `index.html`, `data/games.json`, `identidade.html`, `404.html`.

The games are **not** here. Each is its own repository, `game-<slug>` under **ADR-0082**. The reason is the
byte budget, not tidiness: pillar 1 is public-school hardware and pillar 8 is an offline PWA, so the
precache budget always forbade shipping the whole catalogue to the device.

## State, and what not to invent

There is **no `package.json`** — therefore **no build, no test and no lint command**. Do not invent one, and
do not add a toolchain to make a change convenient. Every page is hand-written, self-contained, and opens
by double-clicking it. That is a property worth defending: a site that needs a build step is a site that
will be broken on the day someone needs to fix it in a hurry.

## Conventions

- **English for every artifact** — docs, code, comments, commit messages, file names. The conversation with
  the Dev is pt-BR. ⚠️ **The exception is product surface only**: the text a child, a teacher or a parent
  reads on the page stays pt-BR, because it is the product. The comments inside that same file are English.
  A commit message is an artifact, not conversation — it goes in English even when everything around it is
  pt-BR.
- **AGPL-3.0-or-later** (**ADR-0064**), with an SPDX header on every source file. The `LICENSE` at the root
  is the AGPL; anything claiming GPL-3.0 here is stale and wrong.
  ⚠️ **The art is not AGPL.** A program is what Law 9.609 defines; art follows Law 9.610.
- **Atomic commits.** One commit carries **one separable decision**. A data artifact, the page that consumes
  it, the removal of what it replaces and an asset move are four commits, not one. Order them by dependency
  — assets, then data, then consumer, then documentation — so each commit leaves the tree coherent on its
  own. Announce what goes in before committing.
- **The Dev pushes.** Commit, say how many are waiting, and stop. Never `git push`, never `gh pr create`,
  never anything that deploys or spends.

## The rules the pages carry

These are not style preferences; each came from a measurement, and undoing one silently costs accessibility.

- **No third-party fonts, no CDN, no external anything.** Postmortem finding 04: package the fonts with the
  product, with per-language subsetting, served locally. A page that promises to work offline while asking a
  third party for its letterforms has already broken the promise — invisibly, because on the developer's
  machine the font is cached. The named debt is Jersey 15 and Atkinson Hyperlegible; until there is a build
  that packages them, the pages use the system stack.
- **`--yellow` FILLS, `--yellowInk` draws LINES.** In the light theme the fill yellow measures 1.3–1.5:1
  against every surface, so a border or an outline made of it draws nothing; `--yellowInk` measures 5.4–6.1:1.
  In the dark theme they are the same colour, which is exactly what hides the mistake. Every border, boundary
  and focus ring uses `--yellowInk`.
- **The focus ring needs `outline-offset`.** Without it, focus on the primary button is yellow on yellow,
  1.0:1, in both themes. The offset lands the ring on the page instead of on the button.
- **Never put brand green `#0B7A46` next to brand blue `#2E5BFF`.** They measure 1.0:1 — the same luminance.
  Inside the symbol the yellow cross separates them on all four sides; nowhere else will it be there to.
- **44 px touch targets, zero `border-radius`, the 8 px spacing scale, colour never the only signal.**
- **`404.html` uses root-absolute paths for everything.** It is the one page whose address is not its own:
  Pages serves that file at whatever URL was requested, so a relative path breaks at depth.
- **`identidade.html` computes its contrast ratios** from the tokens that paint it. Do not transcribe a
  ratio into it. The brandbook stated seven and five were wrong, which is the whole reason.

## `research/` is gone, and stays gone

It held the brand, design-system, journey and postmortem prototype canvases. They are **design fiction**:
written to make a screen work, not audited. Measured against the BNCC, several of the Jornada's skill rows
describe a different skill than the code names. The folder is deleted and in `.gitignore`.

⚠️ **Nothing downstream may treat those canvases as a source.** They are readable from the history
(`git show fa4ec2e:research/design-system/<file>`) when a brand question needs the original wording, and
that is all. The checkable account of the identity is `identidade.html`.

## The one rule that is easy to get wrong in `data/games.json`

`works` versus `material` is not a filtering convenience. A game only **works** a BNCC item if the skill
holds the game up **and** the game supplies mediation for it. If you can play and win without exercising the
skill, the game does not teach it — chess has coordinate reading and still does not work it, because not
reading coordinates stops nobody from winning. BNCC items are **tags for filtering**, and that is all they
are. Some games work no skill at all, and saying so is the point.

⚠️ **The roster lives in `data/games.json` and nowhere else.** Never write the count into prose — not
into a page, a heading, a button label or a README. `index.html` derives every "N games" string from
`dados.games.length` at runtime, and `404.html`, which must work with scripting off, simply does not count.
A number in prose is wrong the day a game is added, and wrong silently.

## Where the engine rules live

Anything about the engine, the cartridge contract, frame timing or input belongs in the engine repository
and its records — not here. This repository ships no game code.
