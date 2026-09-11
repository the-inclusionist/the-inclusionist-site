# Cartridge brief

For a session working inside **one game repository**, converting it from a standalone application
into a cartridge. Read this first; the reasoning is in [architecture.md](architecture.md).

**The rules below are not this file's.** They come from three records in
`the-inclusionist-docs/docs/2-Architecture/adr/` — **`ADR-0139`** (a cartridge supplies half of
`CreateGameOptions` and never calls `createGame`), **`ADR-0140`** (standalone PWA *and* cartridge,
from one source) and **`ADR-0141`** (a cartridge owns its random stream). Read them when this brief
is not enough, and follow them when this brief is wrong.

## Where the records are

| What | Where |
|---|---|
| ADRs | `the-inclusionist-docs/docs/2-Architecture/adr/` — `ADR-0123` moved **only** this tree |
| Engine user stories | `the-inclusionist-engine/docs/1-Discovery/User-Stories.md` |
| Non-functional requirements | `the-inclusionist-engine/docs/1-Discovery/NFR.md` |
| Game-layer user stories | **your own repository** — by design, not by omission |
| Platform architecture | `the-inclusionist-site/docs/architecture.md` |

## The flow, end to end

1. The child opens **one origin**. One PWA, one service worker, one install (`ADR-0117`, `ADR-0118`).
2. **Screen 1** — skills are picked from the BNCC. No login. The filter becomes a short code a
   student can type to inherit it.
3. **Screen 2** — the catalogue, filtered. Each card says *Trabalha* or *Serve de material para*.
4. **Screen 3** — the game. Same origin, same document. No new tab, no iframe.
5. Before your code runs, the **platform** has already mounted the accessibility bar, the pause card,
   the colour-vision filters, the TTS, the sonar, the settings panel, the menu navigation and the
   keyboard runtime.

**Your game is step 4 and nothing else.**

## What your repository has to change

**Your repository ships TWO artifacts from one source**: it stays a standalone PWA *and* becomes a
cartridge. Neither replaces the other.

1. **`main.ts` stops booting on import.** Export a factory instead:
   `export default function create(ctx): { update(dt), teardown() }`. No `let` at module scope.
2. **The factory never calls `createGame`.** Export your `GameDeclaration` instead. Whoever hosts you
   calls `createGame` — once — and that is what lets one source serve both modes.
3. **Add a standalone shell**, `src/standalone.ts`: it calls `createGame`, builds a `ctx`, calls your
   factory and runs the loop. About thirty lines. **`app/index.html` loads this, not `main.ts`.**
   The platform is simply a different shell around the same factory.
4. **`package.json`** — `@the-inclusionist/engine`, `pixi.js` and `zdog` become **both**
   `peerDependencies` (so a consumer installs one copy) **and** `devDependencies` (so your own
   standalone build still works). Add `exports` and `files`; remove `private: true`.
5. **`vite.config.ts`** — two build targets from one config, switched by mode:
   - **app** (default): today's build, engine bundled, `vite-plugin-pwa` on. This is your standalone
     PWA and your test harness.
   - **lib**: `build.lib` with `formats: ['es']`, `rollupOptions.external` for the engine and the
     shared render libraries, no HTML, no service worker. This is the cartridge, and this is what
     gets published.
6. **Export your i18n dictionaries.** Your standalone shell registers them; so does the platform.
7. **Be a PWA in standalone mode.** Only the platformer is one today. Adding `vite-plugin-pwa` to the
   app build is part of this work for the other five.

## What does not change

- Your rules, your tests, your art, the content of your declaration.
- Your licence and `CREDITS.md` (`ADR-0068` §1).
- No `adr/` folder in your repository — ever (`ADR-0068` §5).
- English for artifacts; pt-BR only where it is product content.
- `dt` is counted in **frames**, not seconds.
- The keyboard is listened to on **`#game-region`**, never on `window`.

## Two things that break silently

- **Module-level `let`.** A cartridge is instantiated by a factory; state at module scope survives
  `teardown()` and leaks into the next game on the same page. This is spec decision D14, and it is
  also the engine's own user story: «I want the engine to carry no game state, so that two games on
  one page do not collide».
- **Calling `createGame` from a cartridge.** It mounts a whole accessibility stack. N calls means N
  accessibility bars, N TTS instances and N keyboard runtimes competing for one document — a defect
  that shows up as broken behaviour, not as weight.

## Your engine version

Five repositories are on engine 8; **the platformer is on `^7.0.1`**. Under `peerDependencies` a
mismatch fails the platform's install loudly instead of quietly shipping two engines. If you are on
7, that upgrade is part of this work.

## Order

`ADR-0068` §6: one game goes end to end before the others start — «created, built against the engine
package, published, selected by the manifest, delivered inside the precache budget, and gated by the
reusable workflow».

That game is **whackwhack**: smallest build (344 KB), already on engine 8, CI and 24 test files, and
only Zdog as a heavy render dependency. If you are not whackwhack, wait for the contract to come back
with its holes filled.

## Not decided yet — do not invent an answer

- The exact shape of `ctx`. Derive it from what `CreateGameOptions` already takes; do not design it
  fresh.
- **chess**: its three HTML entries (2.5D, 2D, 3D) become three runtime views of one cartridge, or
  the platform inherits three URLs. Unresolved.
- **pinball**: it has no git remote at all, and its slug is undecided — the package says
  `game-space-cadet`, the folder says `game-pinball`, and `ADR-0082` §1 makes them the same word.
