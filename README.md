# The Inclusionist — the site

This repository changed roles, and the record that changed it explains it better than a summary: under
**ADR-0068** it **stopped holding the games** and started holding the **manifest** that decides which
games, at which versions, go into a delivery.

⚠️ **And the reason is not tidiness — it is the byte budget.** Pillar 1 is public-school hardware and
pillar 8 is an offline PWA: the precache budget **always** forbade shipping the whole catalogue to the
device. That is to say, this repository was never the unit of delivery; it only looked like one. Each
game is its own repository (**ADR-0068**, `game-<slug>` under **ADR-0082**), and what happens here is the
**choosing** of what ships in a version.

Alongside the manifest it now carries the **public surface** — the definitive demonstration address,
which will hold everything the official site will hold.

## What is here today

| Piece | What it is |
|---|---|
| `index.html` | The journey in three steps: BNCC skills → game → play. No build, no external dependency, no third-party font. |
| `data/games.json` | The catalogue of the own games, with description, genre, and the BNCC tags that filter them. ⚠️ **The roster lives here and nowhere else** — no page, heading or document states how many there are. |
| `identidade.html` | The visual identity as a case study: symbol, colour, typography and geometry, with the contrast ratios **computed in the page itself** from the tokens that paint it. |
| `404.html` | The 404, which is also the mechanism: Cloudflare Pages serves this file with status 404 for any path that matches no asset. |
| `img/` | The assets the pages serve: the logo SVG set in `img/logo/`, and the 404 image. |
| `docs/` | Hosting, architecture, the cartridge contract, the journey and the BNCC tags — documents that **cite** the records rather than containing them. |

### The prototype canvases are gone, and that is deliberate

`research/` held the brand, design-system, journey and postmortem canvases. It has been **deleted and
gitignored**. They were the brand authority and the source these pages were written *from*, but they are
*design fiction*: no page ever loaded them, their curriculum data is **not verified**, and shipping them
at a public demonstration address would publish unchecked BNCC text next to a real product.

So the versioned account of the identity is now `identidade.html` alone — which is why it **computes**
what it claims from the tokens that paint it, instead of transcribing numbers from a canvas that is no
longer here to be checked against.

They are still readable from the history when one is needed:
`git show fa4ec2e:research/design-system/Inclusionista-Marca.dc.html`.

⚠️ **What is still missing: the manifest.** It is the centrepiece of the new role — game, version, and why
it is in this delivery — and it does not exist. Until it does, this repository describes a role it does
not yet perform, and that is written down here so it does not read as finished. For the same reason,
step 3 of `index.html` states outright that no cartridge is published, instead of showing an empty frame.

## Two tag classes, and the difference is the ZPD

`data/games.json` separates **`works`** from **`material`**, and the separation is the project's rule, not
a filtering convenience. A game only **works** a BNCC item if the skill holds the game up *and* the game
supplies mediation for it — if you can play and win without exercising the skill, the game does not teach
it. Chess has coordinate reading and still does not work it, because not reading coordinates stops nobody
from winning. BNCC items are **tags for filtering**, and that is all they are.

## Licence

Code: **AGPL-3.0-or-later** (**ADR-0064**) — the `LICENSE` at this root, added on 2026-09-06 during the
migration, because this repository arrived from GitLab without one.
⚠️ **The art is NOT AGPL.** A program is what Law 9.609 defines; art follows Law 9.610 and belongs to
whoever made it. What governs what is in `docs/LICENSES.md`, in the engine.

⚠️ **Economic ownership belongs to the MUNICIPALITY** (Law 9.609/1998, art. 4), not to the developer.
Publication is the subject of a **request** in the application — an act of the executive branch — and that
is why this repository is **private** (ADR-0066 §3).

---

The records live in **`the-inclusionist-docs`**, under `docs/2-Architecture/adr/` — there is **no** `adr/`
folder here, and that is a decision (**ADR-0068 §5**, and **ADR-0123**, which moved them out of the engine
into a repository of their own). Three hundred places to decide architecture are three hundred places
where one decision silently contradicts another.
