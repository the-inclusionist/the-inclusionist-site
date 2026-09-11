# The cartridge contract

> **Decided in `ADR-0139`** (the split and the types) and **`ADR-0141`** (the random stream), in
> `the-inclusionist-docs/docs/2-Architecture/adr/`. This file is the long reading of both — the
> derivation shown step by step, the evidence with file and line, and the open questions. Where the
> two disagree, the records win.

The exact shape of what a cartridge exports and what a shell hands it. Derived from what
`createGame` already takes and returns, not designed fresh — the derivation is shown so the next
person can check it rather than trust it.

Read [architecture.md](architecture.md) first for why there are two shells.

## The derivation

`CreateGameOptions` — what `createGame` takes — splits cleanly in two, and the split is the contract.

**The host knows these.** They describe the page and the device, not the game:
`host` (the `EngineHost`: `doc`, `win`, `cvdHost`, `a11yBarHost`, `pauseHost`), `carregarVozNeural`,
`baixarPesados`, `aoProgredirPesados`, `disponibilidade`, `declines`.

**Only the game knows these.** Every one is a callback *into* the game or a statement *about* it:
`declaration`, `isNavigable`, `comIndice`, `naBarraDe`, `navBar`, `players`, `setPhase`,
`sonarPlayers`, `isBlindMode`, `preset`.

So a cartridge does not receive `CreateGameOptions` — **it supplies half of it.** The shell owns the
other half, merges the two, and calls `createGame` once. What comes back, the `Engine`, is what the
cartridge consumes.

That is the whole contract. Everything below is the type, and the four places where the derivation
turns up something that has to be decided rather than read off.

## The types

```ts
// ── what a cartridge exports ──────────────────────────────────────────────
export interface Cartridge {
  /** Matches the repository and the package name, per ADR-0082 §1. */
  readonly slug: string;

  /** The engine's contract object, unchanged — the same value `CreateGameOptions.declaration` takes. */
  readonly declaration: GameDeclaration;

  /** Registered by whichever shell loads this cartridge; a cartridge never registers its own. */
  readonly dicts: Readonly<Record<string, Dict>>;

  /** The game-owned half of CreateGameOptions. */
  readonly hooks: CartridgeHooks;

  /** Nothing runs until this is called. No side effects at module scope — spec D14. */
  create(ctx: GameCtx): GameInstance;
}

export interface CartridgeHooks {
  isNavigable?(): boolean;
  comIndice?(): boolean;
  naBarraDe?(i: number): boolean;
  navBar?(i: number, k: NavKeys): void;
  players?: CreateGameOptions['players'];
  setPhase?(p: 'title' | 'playing' | 'paused'): void;
  sonarPlayers?(): SonarPlayer[];
  isBlindMode?(): boolean;
  preset?: ActionPreset;
}

// ── what a shell hands in ─────────────────────────────────────────────────
export interface GameCtx {
  /** Exactly what createGame returned. One instance, however many cartridges exist. */
  readonly engine: Engine;

  /** This cartridge's element. It may write inside it and nothing outside it. */
  readonly region: HTMLElement;

  /** This cartridge's own stream. See §RNG — this is not a convenience. */
  readonly rng: Rng;

  /** Translate, already scoped to the active locale. */
  readonly t: Translate;

  /** What the shell decided this cartridge may read from the address. See §params. */
  readonly params: URLSearchParams;
}

export interface GameInstance {
  /** dt is counted in FRAMES, not seconds. */
  update(dt: number): void;
  /** Release everything. After this returns, `region` is emptied by the shell. */
  teardown(): void;
}
```

## Why `ctx` carries three things the `Engine` does not

Each one exists because leaving it out reintroduces a specific defect.

### `region` — the teardown boundary

`Engine` has no notion of *this game's* part of the page. Without one, teardown is a promise rather
than a fact: a cartridge that appended a node somewhere else leaves it behind, and the next game
inherits it. With `region`, teardown is enforceable — the shell empties it and anything the cartridge
left outside is a bug with a name.

### `rng` — the one that would have bitten

This is not ergonomics. The engine's `core/rng` exports **two different things**, and today the games
use the wrong one:

```ts
export const createRng: (semente?: number) => Rng;   // an independent stream
const _padrao = createRng(SEMENTE_PADRAO);           // ⚠️ module-level, shared
export const rnd, randInt, shuffle, reseed;          // all bound to _padrao
```

`whackwhack/app/js/boot/main.ts:23` imports `rnd` from that module, and it is not alone. In
standalone mode that is harmless: one game, one stream. **In platform mode two cartridges importing
`rnd` share one stream, and a `reseed(s)` in one repositions the other's** — which is precisely the
failure the engine's own user story names: «As a game developer, I want the engine to **carry no game
state**, so that two games on one page do not collide».

So the rule is: **a cartridge takes `ctx.rng` and never imports `rnd`, `randInt`, `shuffle` or
`reseed` from `core/rng`.** The shell builds one `createRng(seed)` per cartridge. `createRng` already
exists and its own doc says «Reposiciona ESTA corrente. Não alcança nenhuma outra» — the engine
solved this before anyone needed it; the games simply reached for the shared one.

A gate for this is cheap and worth having: a lint rule forbidding those four named imports in a
cartridge.

### `params` — because six games share one address

In standalone mode a game reads its own query string: the 15-puzzle takes `?seed=`, the pinball takes
`?table=` and `?demo=original`. In platform mode there is **one** address for every cartridge, so a
cartridge reading `location.search` directly would read another game's parameters, or the platform's.
The shell decides what this cartridge may see and hands it over. Nothing else changes for the game.

## The loop belongs to the shell

`startLoop(ticker, frame, maxDt?, { aoFalhar })` takes a Pixi-shaped ticker and an error callback.
Three consequences:

- **A cartridge never calls `startLoop`.** In standalone the shell calls it; in platform mode the
  platform runs one loop and calls each mounted cartridge's `update(dt)`. Six cartridges each opening
  their own `requestAnimationFrame` is six loops fighting over one frame.
- **`dt` is in frames**, because the ticker's `deltaTime` is. Physics copied from a seconds-based
  tutorial runs wrong, and this is the inherited convention that most often breaks.
- **`aoFalhar` is where spec D16 lives** — «one broken game must stay distinguishable from a broken
  engine». The shell wires it to `srAlert` and to something visible, and a cartridge that throws stops
  itself rather than the platform.

## The one hard problem: `createGame` runs once, and declarations are per-game

`createGame` takes `declaration` as a value and calls `conformanceProblems(o.declaration)` once, at
boot (`dist-pkg/boot/create-game.js:99`). With one engine instance and N cartridges, that value has
to mean something different depending on which cartridge is mounted.

The precedent for a *dynamic* declaration is already in the record: `ADR-0084` changed `topology`
from a value to a function precisely because a static reading went stale — the 15-puzzle's
3×3/4×4/5×5 needed a getter, and «`conformanceProblems` lia uma vez só — quem memorizasse a topologia
ficava defasado em silêncio».

Three ways forward:

**(a) A delegating declaration, with the shell recomputing diagnostics.** The platform passes a
declaration whose every member forwards to the mounted cartridge, and re-runs
`conformanceProblems(current.declaration)` itself on each swap. No engine change; works today.
**Its cost has to be said out loud:** `engine.problems` then describes whatever was mounted at boot
and quietly goes stale, so it becomes a trap for anyone who reads it in platform mode.

**(b) The engine gains a mount API** — `engine.mount(declaration)` / `engine.unmount()` — re-running
conformance and re-deriving anything cached from the declaration. This is the right answer and it is
an engine change: an issue and an ADR in `the-inclusionist-docs`, not a decision this repository can
take alone.

**(c) One `createGame` per cartridge.** Rejected. It is the option that brings back N accessibility
bars, N TTS instances and N keyboard runtimes, which is the entire reason the single-call rule exists.

**Recommended:** build (a) now so the first two cartridges can run, and open (b) against the engine in
the same week, because (a)'s cost is a silently wrong diagnostic and those age badly.

## What stays a deep import

A cartridge keeps importing the engine's stateless utilities directly — `core/a11y-sr.js` (`srSay`,
`srAlert`), `core/i18n.js` (`t`), `ui/pause-icons.js` constants. They hold no state, the bundler
deduplicates them, and routing them through `ctx` would be ceremony.

The line is exactly this: **anything that holds module-level state comes through `ctx`; anything
stateless may be imported.** `core/rng` is the module that looks stateless and is not, which is why
it gets a rule of its own above.

## Open, and not to be invented

- Whether `Dict` and `Translate` are exported types of the engine today, or need to be. Read before
  assuming.
- Whether `declines` is host-owned or game-owned. It reads as a statement about what a *game* does not
  have, which would put it in `CartridgeHooks`, but `createGame` also returns a resolved `declines` —
  so the direction has to be read from the implementation rather than guessed.
- The seed policy: who chooses a cartridge's seed, and whether a run is meant to be reproducible
  across shells. The 15-puzzle's `?seed=` says somebody already cared.
