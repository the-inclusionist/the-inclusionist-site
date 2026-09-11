# Architecture — one PWA, many cartridges, one engine

> **The decisions here live in the records, not in this file.** Architecture is decided in one place
> (`ADR-0068` §5), and an earlier version of this document held decisions that bind all six game
> repositories from inside one of them.
>
> - **`ADR-0139`** — a cartridge supplies half of `CreateGameOptions` and never calls `createGame`.
> - **`ADR-0140`** — a game is a standalone PWA *and* a cartridge, from one source; supersedes
>   `ADR-0117` in part.
> - **`ADR-0141`** — a cartridge owns its random stream.
>
> They are in `the-inclusionist-docs/docs/2-Architecture/adr/`. This file is the long reading: the
> measurements, the mechanisms and the per-game consequences.

## One PWA: yes

That part is settled and the reason is not taste. `ADR-0117` and `ADR-0118` fix one origin serving
one platform, because Cache Storage is partitioned by origin and a game per origin would mean a
service worker, a precache and a set of accessibility preferences per game — none of which follow the
child from one game to the next. See [hosting.md](hosting.md) for the records and for what is still
open on the delivery side.

This document answers the next question, which is the harder one: **with every game depending on the
engine, how do we avoid loading the engine dozens of times instead of once?**

## The fear is correct, and this is the measurement

As the six repositories stand today, that is exactly what would happen. Each game is an
**application**, not a library: `vite build` produces its own `index.html` and its own `dist/` with
the engine compiled inside it. Copy six `dist/` folders into a platform and you have shipped six
engines.

Measured on 2026-09-11:

| | 15-puzzle | 2048 | chess | whackwhack | pinball | platformer |
|---|---|---|---|---|---|---|
| engine declared under | `dependencies` | `dependencies` | `dependencies` | `dependencies` | `dependencies` | `dependencies` |
| engine version asked for | `8.0.0` | `8.0.0` | `^8.0.0` | `8.0.0` | `^8.0.0` | **`^7.0.1`** |
| builds as a library (`build.lib`) | no | no | no | no | no | no |
| marks the engine `external` | no | no | no | no | no | no |

And the duplication repeats one level down, in the render libraries:

- **PixiJS 7.4.2** is declared by **the engine and by three games** (15-puzzle, 2048, platformer).
- **Zdog ^1.1.3** is declared by **two** (chess, whackwhack).
- **Three ^0.185.1** by chess alone — no sharing to win, but 743.9 KB of it is loaded eagerly on
  `3d.html` today.

> The platformer's `vite.config.ts` does contain the word `external`, which looks like a
> counter-example and is not: it is a Vitest comment insisting the engine must **not** be
> externalised *for tests*. Nothing in any of the six externalises it for the build.

## Three mechanisms, each closing a different duplication

They are not alternatives. Each one closes a leak the others leave open.

### 1 · `peerDependencies`, not `dependencies` — closes it at install

Under `dependencies`, npm is free to install a nested
`node_modules/@the-inclusionist/engine` beneath each cartridge, and it **will** the moment two
cartridges ask for versions that do not unify. Under

```json
"peerDependencies": { "@the-inclusionist/engine": "^8.0.0" }
```

the cartridge says *I need an engine; the host supplies it*, and exactly one is installed, at the top
of the tree.

This is also the mechanism that makes version skew **loud instead of silent**. Today five games ask
for engine 8 and the platformer asks for `^7.0.1`. As dependencies, that quietly ships two engines.
As peer dependencies, the platform's install fails and names the conflict — which is what you want,
because a game a major version behind is a decision, not an accident to discover in production.

Apply the same treatment to the shared render libraries: `pixi.js` and `zdog` are peers of a
cartridge, not dependencies of it.

### 2 · Build as a library, with the engine `external` — closes it in the bundle

One installed engine still travels N times if N bundles inline it. The cartridge has to stop
producing an application and start producing a module:

```js
build: {
  lib: { entry: 'src/index.ts', formats: ['es'] },
  rollupOptions: { external: [/^@the-inclusionist\/engine/, 'pixi.js', 'zdog'] },
}
```

The cartridge then emits a literal `import { … } from '@the-inclusionist/engine'` and leaves
resolution to whoever consumes it. Its `index.html` disappears: the platform owns the only one.

### 3 · One platform build, with `import()` per cartridge — closes it in the chunks

The platform runs a **single** Vite build that pulls each cartridge in through a dynamic import.
Rollup then does the work by itself:

- any module reachable from more than one dynamic chunk — the engine, Pixi, Zdog — is emitted **once**
  into a shared chunk that every game chunk imports;
- each game's own code lands in its own chunk, fetched and parsed only when that game is opened.

That is what actually produces *one engine, many games*. The manifest of `ADR-0068` §2 decides which
cartridges are in the build; the precache list is read from that same manifest (`ADR-0117`).

> ⚠️ **Lazy chunk ≠ lazy download.** `ADR-0116` §2 is explicit — «precached at install, never fetched
> lazily at first use». So every chunk is *downloaded* at install time. What splitting buys is that a
> game's code is only **parsed and executed** when the child opens it, which is the part that costs on
> the hardware pillar 1 describes. The two rules do not conflict; they answer different questions.

## The duplication that matters more than bytes: `createGame`

The engine's default export is a composition root:

```ts
export declare function createGame(o: CreateGameOptions): Engine;   // dist-pkg/boot/create-game.js
```

It mounts the accessibility bar, the pause card, the six colour-vision filters, the TTS, the sonar,
the settings panel, the menu navigation and the keyboard runtime. Today **each game calls it.**

If each cartridge kept calling it inside one platform, the bytes would be deduplicated and the
runtime would not: N accessibility bars, N TTS instances, N keyboard runtimes competing for the same
document. That is a worse failure than shipping the engine twice, and it would appear as bugs rather
than as weight.

**So the platform calls `createGame` once, and a cartridge never calls it.** The cartridge hands over
its `GameDeclaration` and its loop; the platform owns the composition root.

This is not an imposition on the games — it repairs a defect the engine already recorded. From
`create-game.d.ts`, on the Dev's request of 2026-09-07 that every game carry the same pause menu and
accessibility icons from the first screen:

> ⚠️ E A MEDIÇÃO DE 2026-09-08 MOSTROU QUE É UM ACHADO … dos seis jogos do catálogo local, **CINCO
> não têm barra de acessibilidade nenhuma** — nem menu de pausa … ou seja, cada jogo tinha de se
> lembrar, e cinco não se lembraram.

A composition root owned by the platform makes that impossible to forget, by construction.

## The cartridge contract already exists

Nothing new has to be invented. The engine publishes `createGame` and the `GameDeclaration` type
(`core/contract.js`), and every one of the six games already has an
`app/js/declaration/<game>-declaration.ts`.

And this repository's own spec wrote the cartridge shape before the topology changed — decisions
D14–D16, still correct for a reason that survived the move:

- **D14** — «A game is a **factory**: `create(ctx)` returns `{ update, teardown }`, with no `let` at
  module scope.»
- **D15** — «`games/**` may import only `engine/game-api.ts`», enforced in CI.
- **D16** — «The frame loop has an **error boundary**. One broken game must stay distinguishable from
  a broken engine.»

So a cartridge is:

```ts
export const declaration: GameDeclaration = { /* … */ };
export default function create(ctx: GameCtx): { update(dt: number): void; teardown(): void };
```

What blocks that today is small and specific: each game's `app/js/boot/main.ts` is a **side-effecting
top-level module** that boots the game on import. It has to become an exported factory that boots
nothing until called.

## Two artifacts from one source

**Requirement, 2026-09-11:** a game must still work as **its own PWA**, loading the whole engine by
itself, for anyone who wants to work with that repository alone — *and* as a cartridge inside the
platform, which is this repository's main job.

That is not a compromise between the two shapes. It is the standard library-that-is-also-an-app
pattern, and the rule from the previous section is exactly what makes it cheap: **because the
cartridge never calls `createGame`, the thing that calls it is free to be different in each mode.**

```
src/index.ts        the cartridge    — exports `declaration` and `create(ctx)`. No side effects.
src/standalone.ts   the app shell    — calls createGame, builds ctx, calls create(ctx), runs the loop.
app/index.html      loads standalone.ts
```

The shell is roughly thirty lines, and **the platform is simply a different shell around the same
factory.** Nothing in the game knows which one it got.

Two build targets, from one config switched by mode:

| | app (default) | lib (published) |
|---|---|---|
| entry | `app/index.html` → `standalone.ts` | `src/index.ts` |
| engine | **bundled** | **external** |
| `vite-plugin-pwa` | on | off |
| output | a standalone PWA | an ESM module, no HTML, no service worker |

And in `package.json` the engine and the shared render libraries are declared **twice**: as
`peerDependencies`, so a consumer installs exactly one copy, and as `devDependencies`, so the
repository can still build and test itself alone. That pairing is the canonical way a package is
both a library and an application, and it is what keeps `npm install` working in an empty clone.

Consequences worth stating plainly:

- **The standalone build carries its own engine, and that is correct.** Nobody ships six standalone
  PWAs to a child; the byte budget belongs to the platform build, which is the one that deduplicates.
- **All six become PWAs in standalone mode.** Only the platformer is one today, so that is new work
  in five repositories — and it removes the oddity of a README that claims offline and a build with
  no service worker.
- **The standalone app is also the test harness.** A game repository can be developed, demonstrated
  and audited with no platform in existence, which is most of the reason for the requirement.

> ⚠️ **This touches `ADR-0117` and the line has to be drawn deliberately.** That record says «A GAME
> IS A CARTRIDGE, NOT A PWA … what it stops being is a **unit of installation**». A standalone build
> that exists for the developer is not a unit of installation for anybody. A standalone build that is
> *deployed for children* is, and then `ADR-0117`'s whole argument bites exactly as written — the
> cache partitions, and the accessibility settings stop following the child between games. The
> compatible reading is: **the standalone artifact is a development and demonstration route, never a
> delivery route.** That distinction is not in the record today and should be.

## What each game has to change

| Change | Why |
|---|---|
| `main.ts` → exported factory, no top-level boot | D14; a module that boots on import cannot be one of six |
| stop calling `createGame`; export the declaration instead | the platform owns the composition root |
| add `src/standalone.ts`; `index.html` loads it instead of `main.ts` | keeps the repository a working PWA on its own |
| engine, `pixi.js`, `zdog` → `peerDependencies` **and** `devDependencies` | one copy for the consumer, a working install for the repository |
| two build targets: app (bundled, PWA) and lib (`build.lib` + `external`) | stop inlining what the platform will supply, without losing the standalone build |
| `exports`, `files`, `private: false`, then publish | today none of the six is installable |
| add `vite-plugin-pwa` to the app build | five of six are not PWAs today |
| export the i18n dictionaries | the shell registers them — either shell |

Three special cases:

- **chess** ships **three HTML entries** (`index.html` 2.5D, `2d.html`, `3d.html`) and switches
  between them with links. **Decided 2026-09-11: they become three board views inside the game**, not
  three entries — chosen at runtime from within the cartridge, the way any other setting is. The Vite
  config's three Rollup inputs collapse to one, and the reasoning recorded beside them survives: the
  flat board was made a separate entry so it would not carry a renderer it never draws. As runtime
  views the same saving is had with dynamic `import()` — the Three.js view in particular becomes a
  chunk nobody parses unless they pick it, which is better than today, where `3d.html` loads 743.9 KB
  eagerly.
- **the platformer** is the only one that is a PWA today. It stays a PWA in its **app** build and is
  not one in its **lib** build — which is the general rule, arrived at early. Its manifest still needs
  fixing on the way through: `"scope": "/"` claims the whole origin and `"lang": "en"` is wrong for a
  product delivered in pt-BR. It is also the one repository still on engine 7.
- **the pinball**, as of 2026-09-11, has its remote — `the-inclusionist/game-pinball` — and its folder
  renamed to match, so the slug is settled: **`game-pinball`**. One line is still out of step with it,
  the `name` field in `package.json`, which reads `@the-inclusionist/game-space-cadet`; `ADR-0082` §1
  makes the repository and the package the same word. It also acquires
  chess's problem, from a decision taken after its plan was written: the table editor of
  `docs/plans/2026-09-08-a-table-editor.md` §6 is «a second Vite entry — `app/editor.html` — so the
  game's bundle does not carry it». Inside a cartridge that is a second URL, not a second bundle. The
  reasoning behind §6 survives intact and the mechanism changes: the editor becomes a **lazily
  imported chunk** of the one cartridge, which keeps it out of the play path exactly as intended
  while leaving the platform with one address.

## Order of work

The conversion is six applications becoming six libraries, and the largest of them (the platformer:
16 MB `dist/`, engine 7, the only PWA, the only one carrying school curriculum) is also the hardest.
Doing them in parallel would mean discovering the contract's gaps six times.

`ADR-0068` §6 already requires the opposite, and for this reason:

> it does not run until **ONE game has gone end to end** from the template — created, built against
> the engine package, published, selected by the manifest, delivered inside the precache budget, and
> gated by the reusable workflow.

**whackwhack is the candidate.** It has the smallest build of the six (344 KB, against the
platformer's 16 MB and chess's 8.4 MB), it is already on
engine 8, it has CI and twenty-four test files, its only heavy render dependency is Zdog, and it
carries the cleanest curricular tag in the collection
([bncc-tags.md](curriculum/bncc-tags.md)). If the cartridge contract has a hole, it will show up
there for the least money.

Then the platform build, the manifest, and a second cartridge to prove the shared chunk is actually
shared — two is the smallest number that can demonstrate deduplication at all.

## What this document does not decide

- The cartridge's exact `GameCtx` — it should be derived from what `CreateGameOptions` already takes,
  not designed fresh, and that is a reading of the engine rather than a decision here.
- Whether chess's three views become three lazy chunks or one.
- Whether any of this is promoted to an ADR in `the-inclusionist-docs` (`ADR-0123`). The
  `createGame`-called-once clause is the one most clearly ADR-shaped.
