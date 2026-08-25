# Phase 1 Foundation Implementation Plan

> # 🛑 DO NOT EXECUTE THIS PLAN AS WRITTEN — 2026-08-25
>
> **This plan builds an engine. As of 2026-08-25 that is the wrong instruction**, and executing it would
> create a SECOND engine on the same day the two repositories swapped roles.
>
> `SP-the-inclusionist-tracer` is now `inclusionist-engine`: the engine repository. The game leaves it and
> comes here as one cartridge; the shell comes with it. The decisions are ADR-0035 (the engine is ours,
> Phaser is read and never imported) and **ADR-0036** (this repository consumes the engine as a module —
> "link locally, pin remotely"), both in `../inclusionist-engine/docs/2-Architecture/adr/`.
>
> **What changes, task by task.** Roughly half of Phase 1 already exists in the engine repository, tested:
>
> | Tasks | What they are | What to do instead |
> |---|---|---|
> | 2, 3, 4, 5, 6, 7, 9, 11, 19, 20 | leaf modules, collision, i18n, screen reader, input, canvas + PixiJS mount, CVD/low-vision filters, HUD/pause/audio, PWA + a11y gate + CI, settings | **Do not rewrite.** They exist in `@pm-monte/inclusionist-engine` and are covered by 2 099 passing tests. Consume them. |
> | 8 | high contrast by sprite ROLE, and the `Scene` | **Keep as a rewrite.** Spec D9 is a deliberate redesign: the engine's 534 lines are welded to The Inclusionist's own entity taxonomy. |
> | 10, 12, 13 | accessible shell + router, the public `game-api`, session + composition root | **Keep.** This is the shell of ADR-0036, and it genuinely does not exist anywhere. |
> | 14, 15, 16 | Snake, Pong, Breakout | **Keep.** They are the only honest test that the engine works outside its first genre. |
> | 17, 18 | catalog into data, generated catalog page | **Keep.** |
>
> **One conflict must be resolved before any of this compiles.** Spec D13 forbids module-level mutable state
> in the engine. The engine's `core/state.ts` is exactly that: 26 `export let` read through live bindings by
> twelve modules. It also mixes two kinds of state that must not share a home — PAGE-scoped (accessibility
> settings, language, audio: correctly global, because a blind child must not reconfigure per game) and
> RUN-scoped (`phase`, `players`, `gateTiles`, `ended`: never global in a shell that loads game after game).
> Cutting page × run in the engine is a prerequisite of "consume the engine", and it is the next ADR.
>
> Rewrite this plan against the table above before executing it. This banner comes out with the rewrite.


> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the engine, accessible shell and catalog generator that all 383 games inherit, proven by three reference games (Snake, Pong, Breakout).

**Architecture:** A single Vite app with one shell page (`play.html`) that lazy-loads any game through `import.meta.glob`, and a generated catalog page (`index.html`) that links to it. Games are thin and closed: a game is a factory that receives a `GameContext` and returns `{ update, teardown }`, draws through a renderer-agnostic `Scene`, and imports exactly one engine module. Input, scaling, contrast, screen-reader announcements, pause, language and offline caching live in the engine and are written once.

**Tech Stack:** Node 24, TypeScript 5.7, Vite 8, Vitest 4 (node + browser/Playwright projects), PixiJS 7.4.2, `vite-plugin-pwa` 1.3, `@axe-core/playwright`, `dependency-cruiser`, Prettier, `node-html-parser` (one-shot import only).

**Spec:** [docs/superpowers/specs/2026-08-25-inclusionist-demos-design.md](../specs/2026-08-25-inclusionist-demos-design.md)

## Global Constraints

Every task's requirements implicitly include this section.

- **Logical canvas is exactly `320×180`, `TILE = 16`.** Never fractional scaling — it blurs pixel art.
- **`dt` is counted in FRAMES, not seconds.** `1.0` means one 60 fps frame. Clamp at `2`.
- **Keyboard listeners attach to `#game-region`, never to `window`.**
- **No module-level mutable state in the engine (spec D13).** A module that holds state exports a
  `createX()` factory; the composition root owns the instance. Pure functions and frozen data may be
  module-level. If a test needs a cleanup ritual in `beforeEach` to undo the previous test, the module
  is wrong, not the test.
- **A game is a factory (spec D14).** `create(ctx)` returns `{ update, teardown }`. No `let` at module
  scope in `games/**`.
- **`games/**` may import only `engine/game-api.ts` and files inside its own folder (spec D15).**
  Enforced by `npm run lint:deps`, not by good intentions.
- **English for every artifact**: code, comments, commit messages, docs. Exception: the catalog page's
  visible content (category names, subgenre names and hints) stays pt-BR.
- **Every source file starts with `// SPDX-License-Identifier: GPL-3.0-or-later`.**
- **No PNG ships inside a game.** All art is procedural, painted through `pixelCanvas`.
- **No absolute paths in versioned files.**
- **Three-language floor**: `pt`, `en`, `es`. `pt` is the fallback for every key.
- **Commits are atomic and frequent**, in English, with the trailer `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- **Node commands run on the Dev's machine.** Every task states the exact command and its expected output.
- Tracer source referenced as `<TRACER>` = the sibling checkout of `SP-the-inclusionist-tracer`.

## File Structure

| File | Responsibility | Shape |
|---|---|---|
| `engine/core/constants.ts` | `LOGICAL_W`, `LOGICAL_H`, `TILE`, `MAX_DT` | pure |
| `engine/platform/storage.ts` | Exception-proof `localStorage` wrapper + `KEYS` | pure fns |
| `engine/ui/dom.ts` | `$`, `$$`, `toggleBtn` | pure fns |
| `engine/core/rng.ts` | Seeded LCG | **factory** `createRng(seed)` |
| `engine/core/loop.ts` | Clamped frame driver | pure fn |
| `engine/core/collision.ts` | `aabb`, `sweptAabb` | pure |
| `engine/core/i18n.ts` | Translation, per-game dictionaries | **factory** `createI18n()` |
| `engine/core/a11y-sr.ts` | Screen-reader announcements | **factory** `createAnnouncer(root)` |
| `engine/input/actions.ts` | The eight actions, `KeyScheme`, pure `heldIn` | pure |
| `engine/input/keyboard.ts` | Key schemes, load/save, `schemesFor`, `ownedCodes` | pure fns |
| `engine/input/latch.ts` | Edge detection | **factory** `makeLatch()` |
| `engine/input/attach.ts` | Owns key/pad state, binds to an element | **factory** `attachInput(el, players, kb)` |
| `engine/render/canvas.ts` | `makeCanvas`, `tex`, `pixelCanvas`, `pixelTexture`, `pixDisc` | pure fns |
| `engine/render/mount.ts` | `integerScale`, PixiJS mount | **factory** `mountPixi(host)` |
| `engine/render/high-contrast.ts` | Role → colour at a measured WCAG ratio | pure fns + **factory** `createVisualState()` |
| `engine/render/cvd-matrices.ts` | Machado/Fidaner matrices (lifted) | pure data |
| `engine/render/viz.ts` | Colour-vision and low-vision modes | **factory** `createViz(target)` |
| `engine/render/scene-pixi.ts` | The `Scene` implementation over PixiJS | **factory** `createPixiScene(stage, visual)` |
| `engine/game-api.ts` | **The only module `games/**` may import** | types + pure helpers |
| `engine/shell/router.ts` | `parseHash`, the lazy game registry | pure |
| `engine/shell/hud.ts` | Title and score strip | **factory** `createHud(root)` |
| `engine/shell/pause.ts` | Modal pause menu with a focus trap | **factory** `createPause(panel)` |
| `engine/shell/settings.ts` | Language, contrast and colour-vision controls | **factory** `mountSettings(deps)` |
| `engine/shell/session.ts` | One game's lifecycle: build ctx, run, tear down | **factory** `startSession(deps)` |
| `engine/shell/boot.ts` | Composition root. Wires everything, owns every instance | entry |
| `play.html` | The accessible shell markup | — |
| `games/<cat>/<slug>/rules.ts` | Pure game logic, tested in the node project | pure |
| `games/<cat>/<slug>/main.ts` | `meta`, `strings`, `create(ctx)` | **factory** |
| `scripts/import-catalog.mts` | One-shot: HTML → `data/catalog.json` | — |
| `scripts/build-catalog.mts` | `catalog.json` + template → `index.html` | — |
| `scripts/axe-check.mjs` | WCAG A/AA gate over the built pages | — |
| `.dependency-cruiser.cjs` | The D15 rule, as an executable check | — |

### Shared interfaces

Define these exactly as written. They are the contract every task composes against.

```ts
// engine/core/collision.ts
export interface Box { x: number; y: number; w: number; h: number }
export interface SweptHit { t: number; nx: number; ny: number }

// engine/core/i18n.ts
export type LocaleDict = Record<string, string>;
export type GameStrings = { pt: LocaleDict; en: LocaleDict; es: LocaleDict };
export type Translate = (key: string, params?: Record<string, string | number>) => string;

// engine/input/actions.ts
export type Action = 'up' | 'left' | 'down' | 'right' | 'run' | 'jump' | 'swap' | 'especial';
export type KeyScheme = Record<Action, string[]>;

// engine/input/attach.ts
export interface InputApi {
  held(pl: number, act: Action): boolean;
  pressed(pl: number, act: Action): boolean;
  readonly players: number;
}

// engine/render/high-contrast.ts
export type SpriteRole = 'player' | 'ally' | 'hazard' | 'goal' | 'pickup' | 'bg' | 'ui' | 'neutral';
export type ContrastLevel = 0 | 3 | 4.5 | 7;   // 0 = off

// engine/render/scene-pixi.ts — the drawing contract, renderer-agnostic BY DESIGN
export interface SpriteSpec {
  role: SpriteRole;
  w: number; h: number;
  paint: (px: (x: number, y: number, w: number, h: number, col: string) => void) => void;
}
/** An opaque drawable. A game moves and hides it; it cannot reach the renderer through it. */
export interface Handle { x: number; y: number; visible: boolean }
export interface Scene {
  add(spec: SpriteSpec): Handle;
  remove(h: Handle): void;
  clear(): void;
}

// engine/game-api.ts — the whole surface a game sees
export interface GameContext {
  scene: Scene;
  view: { w: number; h: number; tile: number };
  input: InputApi;
  audio: { beep(freq: number, ms: number): void };
  rng: { rnd(): number; randInt(lo: number, hi: number): number; reseed(s: number): void };
  storage: { get(k: string, f?: string | null): string | null; set(k: string, v: string | number | boolean): boolean };
  t: Translate;
  srSay(text: string): void;
  srAlert(text: string): void;
  onGameOver(score: number): void;
}
export interface GameMeta {
  slug: string; title: string; category: string;
  density: 'leve' | 'medio' | 'denso';
  players: 1 | 2 | 3 | 4;
  renderer?: 'pixel' | 'svg' | '3d';
  rendererWhy?: string;
}
export interface GameInstance { update(dt: number): void; teardown(): void }
export interface GameModule {
  meta: GameMeta;
  strings: GameStrings;
  create(ctx: GameContext): GameInstance;
}
```

> [!warning] Three conventions that are easy to break
> **`dt` is counted in frames, not seconds** — physics copied from a seconds-based tutorial runs wrong.
> **The keyboard is listened to on `#game-region`, not on `window`** — on `window` the game steals the page's keys.
> **`PIXI` must not appear in any type a game can see** — the moment it does, 383 games are pinned to one renderer.

---

### Task 1: Toolchain, formatting and engine constants

**Files:**
- Create: `package.json`, `tsconfig.json`, `vite.config.ts`, `.node-version`, `.editorconfig`, `.prettierrc.json`, `.prettierignore`
- Create: `engine/core/constants.ts`
- Test: `engine/core/constants.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `LOGICAL_W: 320`, `LOGICAL_H: 180`, `TILE: 16`, `MAX_DT: 2`. The npm scripts `dev`, `build`, `preview`, `typecheck`, `test`, `test:node`, `test:browser`, `format`, `format:check`.

> Formatting is automated from the first commit rather than added later. This repository will receive
> hundreds of contributions across separate sessions; without a formatter, half of every future diff is
> whitespace, and reviewers spend attention on nothing.

- [ ] **Step 1: Create the toolchain files**

`.node-version`:
```
24
```

`package.json`:
```json
{
  "name": "inclusionist-demos",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "description": "JS minigame collection in 320x180 pixel art, one game per catalog item.",
  "license": "GPL-3.0-or-later",
  "engines": { "node": ">=24" },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:node": "vitest run --project node",
    "test:browser": "vitest run --project browser",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "validate": "npm run format:check && npm run typecheck && vitest run && npm run build"
  },
  "devDependencies": {
    "@axe-core/playwright": "^4.10.0",
    "@types/node": "^22.0.0",
    "@vitest/browser": "^4.0.0",
    "@vitest/browser-playwright": "^4.0.0",
    "dependency-cruiser": "^16.0.0",
    "node-html-parser": "^6.1.13",
    "pixi.js": "7.4.2",
    "playwright": "^1.49.0",
    "prettier": "^3.4.0",
    "typescript": "^5.7.0",
    "vite": "^8.0.0",
    "vite-plugin-pwa": "^1.3.0",
    "vitest": "^4.0.0",
    "wait-on": "^8.0.0"
  },
  "overrides": { "vite-plugin-pwa": { "vite": "$vite" } }
}
```

`.editorconfig`:
```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

`.prettierrc.json`:
```json
{
  "printWidth": 110,
  "singleQuote": true,
  "trailingComma": "all",
  "arrowParens": "always"
}
```

`.prettierignore`:
```
node_modules/
dist/
index.html
data/catalog.json
```

> `index.html` and `data/catalog.json` are generated. Formatting generated files means the formatter
> and the generator fight, and the diff is never clean.

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "skipLibCheck": true,
    "types": ["vite/client", "node"]
  },
  "include": ["engine/**/*.ts", "games/**/*.ts", "scripts/**/*.mts", "*.config.ts"]
}
```

> `noUnusedParameters` is on deliberately. It is what would have caught the unused `host` parameter
> that the plan audit found — a parameter kept "for symmetry" is a lie in an interface.

`vite.config.ts` (PWA arrives in Task 19 — do not add it now):
```ts
import { defineConfig } from 'vite';
import { playwright } from '@vitest/browser-playwright';
import { resolve } from 'node:path';

export default defineConfig({
  build: {
    rollupOptions: {
      input: {
        catalog: resolve(__dirname, 'index.html'),
        play: resolve(__dirname, 'play.html'),
      },
    },
  },
  test: {
    projects: [
      {
        test: {
          name: 'node',
          environment: 'node',
          include: ['engine/**/*.test.ts', 'games/**/*.test.ts', 'scripts/**/*.test.mts'],
          exclude: ['**/*.browser.test.ts'],
        },
      },
      {
        test: {
          name: 'browser',
          include: ['**/*.browser.test.ts'],
          browser: {
            enabled: true,
            provider: playwright(),
            headless: true,
            instances: [{ browser: 'chromium' }],
          },
        },
      },
    ],
  },
});
```

Create both entry pages as placeholders so `vite build` resolves. Task 10 replaces `play.html`;
Task 17 generates `index.html`.

`index.html`: `<!doctype html><title>catalog placeholder</title>`
`play.html`: `<!doctype html><title>play placeholder</title>`

- [ ] **Step 2: Install dependencies**

Run: `npm install`
Expected: completes without error, `node_modules/` appears.

> If it appears to hang with `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, the local antivirus is re-signing TLS.
> Fix: `NODE_OPTIONS=--use-system-ca npm install`.

- [ ] **Step 3: Write the failing test**

`engine/core/constants.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { LOGICAL_W, LOGICAL_H, TILE, MAX_DT } from './constants.js';

describe('engine constants', () => {
  it('locks the logical canvas at 320x180', () => {
    expect(LOGICAL_W).toBe(320);
    expect(LOGICAL_H).toBe(180);
  });

  it('is exactly 16:9, so integer scaling lands on 1280x720 and 1920x1080', () => {
    expect(LOGICAL_W / LOGICAL_H).toBeCloseTo(16 / 9, 10);
    expect(1280 % LOGICAL_W).toBe(0);
    expect(720 % LOGICAL_H).toBe(0);
    expect(1920 % LOGICAL_W).toBe(0);
    expect(1080 % LOGICAL_H).toBe(0);
  });

  it('divides evenly into whole tiles on both axes', () => {
    expect(LOGICAL_W % TILE).toBe(0);
    expect(LOGICAL_H % TILE).toBe(0);
  });

  it('clamps dt at two frames', () => {
    expect(MAX_DT).toBe(2);
  });
});
```

- [ ] **Step 4: Run the test to verify it fails**

Run: `npx vitest run --project node engine/core/constants.test.ts`
Expected: FAIL — `Failed to resolve import "./constants.js"`.

- [ ] **Step 5: Write the implementation**

`engine/core/constants.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// core/constants — the numbers every other module derives from. Leaf, zero deps.
//
// 320x180 is exactly 16:9, which is why it integer-scales onto 1280x720 (x4) and 1920x1080 (x6) with
// no letterboxing and no fractional pixels. Any other logical size reintroduces blur on the most
// common displays, so this pair is not a preference.
export const LOGICAL_W = 320;
export const LOGICAL_H = 180;
export const TILE = 16;

// Frame-count clamp for the game loop. dt is measured in FRAMES (1.0 = one 60 fps frame), so a
// backgrounded tab returning after two seconds must not deliver dt = 120 and teleport everything
// through walls. Two frames is the tracer's value.
export const MAX_DT = 2;
```

- [ ] **Step 6: Run the test, the typecheck and the formatter**

Run: `npx vitest run --project node engine/core/constants.test.ts`
Expected: PASS, 4 tests.

Run: `npx tsc --noEmit && npm run format`
Expected: no typecheck output; Prettier rewrites what it needs to.

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json tsconfig.json vite.config.ts .node-version .editorconfig .prettierrc.json .prettierignore index.html play.html engine/core/constants.ts engine/core/constants.test.ts
git commit -m "chore: set up the toolchain and lock the logical canvas at 320x180

Prettier and EditorConfig from the first commit rather than later: this repo
will take hundreds of contributions across separate sessions, and retrofitting
a formatter makes one diff that touches everything."
```

---

### Task 2: Leaf modules lifted from the tracer

**Files:**
- Create: `engine/platform/storage.ts`, `engine/ui/dom.ts`, `engine/core/rng.ts`, `engine/core/loop.ts`
- Test: `engine/core/rng.test.ts`, `engine/core/loop.test.ts`, `engine/platform/storage.test.ts`

**Interfaces:**
- Consumes: `MAX_DT` from Task 1.
- Produces:
  - `storage`: `get`, `set`, `remove`, `getBool`, `setBool`, `getNum`, `getJSON<T>`, `setJSON`, `KEYS`.
  - `dom`: `$<T>(sel)`, `$$<T>(sel)`, `toggleBtn(el, on)`.
  - `rng`: `interface Rng` and **`createRng(seed?): Rng`** with `rnd`, `randInt`, `shuffle`, `reseed`.
  - `loop`: `startLoop(ticker, frame, maxDt?)` where `ticker` is `{ add(fn): void; deltaTime: number }`.

> Three of these cross over from the tracer unchanged. `rng` does not: the tracer keeps `_seed` at
> module scope, which means one game reseeding for a level would silently reshuffle another, and two
> tests in the same file would share a sequence. It becomes a factory (spec D13). The LCG constants and
> the call order are preserved, so a given seed still produces the tracer's sequence.

- [ ] **Step 1: Copy three files from the tracer**

| From `<TRACER>` | To | Changes |
|---|---|---|
| `app/js/platform/storage.ts` | `engine/platform/storage.ts` | Header comment to English. Replace the whole `KEYS` object with the one below. |
| `app/js/ui/dom.ts` | `engine/ui/dom.ts` | Header comment to English. `$`, `$$`, `toggleBtn` unchanged. |
| `app/js/core/loop.ts` | `engine/core/loop.ts` | Header comment to English. Import `MAX_DT` and use it as the default instead of the literal `2`. |

The `KEYS` registry for this project:
```ts
// Known storage keys, in one place. Namespaced `demos.` so this project can never collide with a
// tracer build served from the same origin.
export const KEYS = {
  lang: 'demos.lang',
  contrast: 'demos.contrast',
  viz: 'demos.viz',
  kbcontrols: 'demos.kbcontrols.v1',
  highScore: (slug: string): string => `demos.hi.${slug}`,
};
```

The changed default in `loop.ts`:
```ts
import { MAX_DT } from './constants.js';
type Ticker = { add: (fn: () => void) => void; deltaTime: number };
export function startLoop(ticker: Ticker, frame: (dt: number) => void, maxDt = MAX_DT): void {
  ticker.add(() => frame(Math.min(ticker.deltaTime, maxDt)));
}
```

- [ ] **Step 2: Write the failing tests**

`engine/core/rng.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { createRng } from './rng.js';

describe('createRng', () => {
  it('is deterministic: the same seed replays the same sequence', () => {
    const a = createRng(1);
    const b = createRng(1);
    expect([a.rnd(), a.rnd(), a.rnd()]).toEqual([b.rnd(), b.rnd(), b.rnd()]);
  });

  it('diverges on a different seed', () => {
    const a = createRng(1);
    const b = createRng(2);
    expect([a.rnd(), a.rnd()]).not.toEqual([b.rnd(), b.rnd()]);
  });

  it('gives each instance its OWN sequence, so one game cannot disturb another', () => {
    const a = createRng(1);
    const b = createRng(1);
    a.rnd();
    a.reseed(999);
    // b is untouched: it must still be on the second value of seed 1.
    const fresh = createRng(1);
    fresh.rnd();
    expect(b.rnd()).not.toBe(fresh.rnd());
  });

  it('reseeding restarts the sequence', () => {
    const r = createRng(1);
    const first = [r.rnd(), r.rnd()];
    r.reseed(1);
    expect([r.rnd(), r.rnd()]).toEqual(first);
  });

  it('stays inside [0, 1)', () => {
    const r = createRng(7);
    for (let i = 0; i < 1000; i++) {
      const v = r.rnd();
      expect(v).toBeGreaterThanOrEqual(0);
      expect(v).toBeLessThan(1);
    }
  });

  it('randInt covers both endpoints and never exceeds them', () => {
    const r = createRng(3);
    const seen = new Set<number>();
    for (let i = 0; i < 1000; i++) {
      const v = r.randInt(3, 6);
      expect(Number.isInteger(v)).toBe(true);
      expect(v).toBeGreaterThanOrEqual(3);
      expect(v).toBeLessThanOrEqual(6);
      seen.add(v);
    }
    expect([...seen].sort()).toEqual([3, 4, 5, 6]);
  });

  it('randInt with lo === hi returns that value', () => {
    expect(createRng(1).randInt(7, 7)).toBe(7);
  });

  it('shuffle returns a permutation and leaves the input alone', () => {
    const input = Object.freeze([1, 2, 3, 4, 5]);
    const out = createRng(1).shuffle(input);
    expect(out).not.toBe(input);
    expect([...out].sort((a, b) => a - b)).toEqual([1, 2, 3, 4, 5]);
  });

  it('shuffle of an empty array is an empty array', () => {
    expect(createRng(1).shuffle([])).toEqual([]);
  });
});
```

`engine/core/loop.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { startLoop } from './loop.js';

/** Minimal stand-in for PIXI.Ticker: `deltaTime` is set by the test, then `tick()` fires the frame. */
function fakeTicker() {
  let fn: (() => void) | null = null;
  return {
    deltaTime: 1,
    add(f: () => void) { fn = f; },
    tick(delta: number) { this.deltaTime = delta; fn?.(); },
  };
}

describe('startLoop', () => {
  it('passes the ticker delta straight through when it is small', () => {
    const t = fakeTicker();
    const seen: number[] = [];
    startLoop(t, (dt) => seen.push(dt));
    t.tick(1);
    t.tick(0.5);
    expect(seen).toEqual([1, 0.5]);
  });

  it('clamps a backgrounded-tab spike to MAX_DT', () => {
    const t = fakeTicker();
    const seen: number[] = [];
    startLoop(t, (dt) => seen.push(dt));
    t.tick(120);
    expect(seen).toEqual([2]);
  });

  it('honours an explicit maxDt override', () => {
    const t = fakeTicker();
    const seen: number[] = [];
    startLoop(t, (dt) => seen.push(dt), 0.5);
    t.tick(10);
    expect(seen).toEqual([0.5]);
  });

  it('does not call the frame before the ticker fires', () => {
    const t = fakeTicker();
    let calls = 0;
    startLoop(t, () => calls++);
    expect(calls).toBe(0);
  });
});
```

`engine/platform/storage.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import * as store from './storage.js';

/** In-memory localStorage, installed on globalThis — the node project has no DOM. */
function installStorage(): void {
  const map = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => (map.has(k) ? map.get(k)! : null),
    setItem: (k: string, v: string) => { map.set(k, v); },
    removeItem: (k: string) => { map.delete(k); },
  });
}

describe('storage', () => {
  beforeEach(() => installStorage());

  it('round-trips a string', () => {
    expect(store.set('k', 'v')).toBe(true);
    expect(store.get('k')).toBe('v');
  });

  it('returns the fallback for a missing key', () => {
    expect(store.get('nope', 'fb')).toBe('fb');
    expect(store.get('nope')).toBeNull();
  });

  it('round-trips booleans and numbers', () => {
    store.setBool('b', true);
    expect(store.getBool('b')).toBe(true);
    expect(store.getBool('missing', true)).toBe(true);
    store.set('n', 42);
    expect(store.getNum('n')).toBe(42);
    expect(store.getNum('missing', 7)).toBe(7);
  });

  it('falls back rather than throwing on unparseable JSON', () => {
    store.set('j', 'not json');
    expect(store.getJSON('j', { ok: false })).toEqual({ ok: false });
  });

  it('survives a localStorage that throws, which is what file:// does', () => {
    vi.stubGlobal('localStorage', {
      getItem() { throw new Error('denied'); },
      setItem() { throw new Error('denied'); },
      removeItem() { throw new Error('denied'); },
    });
    expect(() => store.remove('k')).not.toThrow();
    expect(store.get('k', 'fb')).toBe('fb');
    expect(store.set('k', 'v')).toBe(false);
  });

  it('namespaces every key under demos. so a tracer build cannot collide', () => {
    expect(store.KEYS.lang.startsWith('demos.')).toBe(true);
    expect(store.KEYS.highScore('snake')).toBe('demos.hi.snake');
  });
});
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `npx vitest run --project node engine/`
Expected: FAIL — unresolved imports for `rng.js`, `loop.js`, `storage.js`.

- [ ] **Step 4: Write `engine/core/rng.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// core/rng — a seeded LCG, per instance.
//
// The tracer keeps its seed at module scope. That is fine for one game and wrong for a collection:
// a game reseeding for a level would silently reshuffle whatever else held a reference, and two tests
// in one file would share a sequence. The generator itself is the tracer's, constants and call order
// unchanged, so a given seed still produces the same numbers.
export interface Rng {
  rnd(): number;
  randInt(lo: number, hi: number): number;
  shuffle<T>(arr: readonly T[]): T[];
  reseed(s: number): void;
}

export function createRng(seed = 20260601): Rng {
  let s = seed >>> 0;
  const rnd = (): number => (s = (s * 1103515245 + 12345) & 0x7fffffff) / 0x7fffffff;
  return {
    rnd,
    randInt: (lo, hi) => lo + Math.floor(rnd() * (hi - lo + 1)),
    shuffle<T>(arr: readonly T[]): T[] {
      const a = [...arr];
      for (let i = a.length - 1; i > 0; i--) {
        const j = (rnd() * (i + 1)) | 0;
        [a[i], a[j]] = [a[j]!, a[i]!];
      }
      return a;
    },
    reseed(next) { s = next >>> 0; },
  };
}
```

- [ ] **Step 5: Create the other three files as described in Step 1**

- [ ] **Step 6: Run the tests to verify they pass**

Run: `npx vitest run --project node engine/`
Expected: PASS. `constants` 4, `rng` 9, `loop` 4, `storage` 6.

- [ ] **Step 7: Typecheck and format**

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

- [ ] **Step 8: Commit**

```bash
git add engine/platform/storage.ts engine/platform/storage.test.ts engine/ui/dom.ts engine/core/rng.ts engine/core/rng.test.ts engine/core/loop.ts engine/core/loop.test.ts
git commit -m "feat: lift the leaf modules from the tracer

storage, dom and loop cross over unchanged apart from English headers and this
project's KEYS. rng becomes a factory: a module-level seed would let one game's
reseed reshuffle another's sequence."
```

---

### Task 3: Collision

**Files:**
- Create: `engine/core/collision.ts`
- Test: `engine/core/collision.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `aabb(a: Box, b: Box): boolean` and `sweptAabb(mover: Box, vx: number, vy: number, target: Box): SweptHit | null`, with `Box = { x, y, w, h }` and `SweptHit = { t, nx, ny }`. `t` is the fraction of the motion (`0..1`) at which contact happens; `nx`/`ny` is the surface normal (`-1`, `0` or `1`).

> Why swept and not just overlap: Breakout's ball crosses more than its own width in a frame at speed,
> and an overlap-only test lets it pass through a brick between two frames. This is the classic tunnelling
> bug, and it is cheaper to have the primitive right once than to debug it in three games.

- [ ] **Step 1: Write the failing test**

`engine/core/collision.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { aabb, sweptAabb, type Box } from './collision.js';

const box = (x: number, y: number, w = 10, h = 10): Box => ({ x, y, w, h });

describe('aabb', () => {
  it('detects a clear overlap', () => {
    expect(aabb(box(0, 0), box(5, 5))).toBe(true);
  });

  it('rejects a clear separation on each axis', () => {
    expect(aabb(box(0, 0), box(20, 0))).toBe(false);
    expect(aabb(box(0, 0), box(0, 20))).toBe(false);
  });

  it('treats edge-touching as no overlap, so a resting body is not permanently colliding', () => {
    expect(aabb(box(0, 0), box(10, 0))).toBe(false);
  });

  it('detects containment', () => {
    expect(aabb(box(0, 0, 100, 100), box(10, 10))).toBe(true);
  });

  it('handles a zero-sized box without reporting a hit', () => {
    expect(aabb(box(0, 0, 0, 0), box(0, 0))).toBe(false);
  });
});

describe('sweptAabb', () => {
  it('returns null when the mover is not moving', () => {
    expect(sweptAabb(box(0, 0), 0, 0, box(50, 0))).toBeNull();
  });

  it('returns null when the motion falls short of the target', () => {
    expect(sweptAabb(box(0, 0), 5, 0, box(50, 0))).toBeNull();
  });

  it('finds the contact fraction on a rightward sweep', () => {
    // mover right edge at 10, target left edge at 30, so 20 of the 40 units of motion are free.
    const hit = sweptAabb(box(0, 0), 40, 0, box(30, 0));
    expect(hit).not.toBeNull();
    expect(hit!.t).toBeCloseTo(0.5, 6);
    expect(hit!.nx).toBe(-1);
    expect(hit!.ny).toBe(0);
  });

  it('finds the contact fraction on a downward sweep and reports an upward normal', () => {
    const hit = sweptAabb(box(0, 0), 0, 40, box(0, 30));
    expect(hit).not.toBeNull();
    expect(hit!.t).toBeCloseTo(0.5, 6);
    expect(hit!.nx).toBe(0);
    expect(hit!.ny).toBe(-1);
  });

  it('catches a mover that would tunnel clean through a thin target in one frame', () => {
    // A 2-wide ball moving 100 units past a 1-wide brick: overlap-only testing misses this entirely.
    const hit = sweptAabb({ x: 0, y: 0, w: 2, h: 2 }, 100, 0, { x: 50, y: 0, w: 1, h: 2 });
    expect(hit).not.toBeNull();
    expect(hit!.t).toBeCloseTo(0.48, 6);
  });

  it('reports the axis of latest entry on a diagonal sweep', () => {
    // Reaches the target's x span later than its y span, so the hit is on the vertical face.
    const hit = sweptAabb(box(0, 0), 40, 40, box(30, 20));
    expect(hit).not.toBeNull();
    expect(hit!.nx).toBe(-1);
    expect(hit!.ny).toBe(0);
  });

  it('ignores a target that is behind the direction of travel', () => {
    expect(sweptAabb(box(50, 0), 40, 0, box(0, 0))).toBeNull();
  });

  it('returns t = 0 for a mover already overlapping the target', () => {
    const hit = sweptAabb(box(0, 0), 10, 0, box(5, 0));
    expect(hit).not.toBeNull();
    expect(hit!.t).toBe(0);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run --project node engine/core/collision.test.ts`
Expected: FAIL — `Failed to resolve import "./collision.js"`.

- [ ] **Step 3: Write the implementation**

`engine/core/collision.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// core/collision — axis-aligned box tests. Leaf, pure, zero deps.
export interface Box { x: number; y: number; w: number; h: number }

/** Contact fraction along the sweep (0..1) and the surface normal of the face that was hit. */
export interface SweptHit { t: number; nx: number; ny: number }

/**
 * Do two boxes overlap? Edge-touching counts as NOT overlapping: a body resting exactly on a floor
 * would otherwise report a collision every single frame, and every caller would need the same
 * epsilon workaround.
 */
export function aabb(a: Box, b: Box): boolean {
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}

/**
 * Swept AABB: where along `(vx, vy)` does `mover` first touch `target`?
 *
 * Returns null when there is no contact within this frame's motion. Returns `t = 0` when the two
 * already overlap, so a caller can treat "stuck inside" and "just touched" through one code path.
 *
 * The method is the slab test: per axis, compute the fraction of the motion at which the mover
 * ENTERS the target's span and the fraction at which it LEAVES. A hit needs the latest entry to
 * come before the earliest exit; the axis that produced that latest entry is the face that was hit,
 * which is what gives the normal.
 */
export function sweptAabb(mover: Box, vx: number, vy: number, target: Box): SweptHit | null {
  if (vx === 0 && vy === 0) return null;
  if (aabb(mover, target)) return { t: 0, nx: 0, ny: 0 };

  // Distance to the near and far faces on each axis, in the direction of travel.
  const nearX = vx > 0 ? target.x - (mover.x + mover.w) : target.x + target.w - mover.x;
  const farX = vx > 0 ? target.x + target.w - mover.x : target.x - (mover.x + mover.w);
  const nearY = vy > 0 ? target.y - (mover.y + mover.h) : target.y + target.h - mover.y;
  const farY = vy > 0 ? target.y + target.h - mover.y : target.y - (mover.y + mover.h);

  // Convert to fractions of the motion. A zero component means the axis never bounds the sweep,
  // so it contributes an infinite span rather than a division by zero.
  const tNearX = vx === 0 ? -Infinity : nearX / vx;
  const tFarX = vx === 0 ? Infinity : farX / vx;
  const tNearY = vy === 0 ? -Infinity : nearY / vy;
  const tFarY = vy === 0 ? Infinity : farY / vy;

  // A zero-motion axis must still respect the existing overlap on that axis, otherwise a mover
  // sliding parallel to a distant wall would report a hit against it.
  if (vx === 0 && (mover.x + mover.w <= target.x || mover.x >= target.x + target.w)) return null;
  if (vy === 0 && (mover.y + mover.h <= target.y || mover.y >= target.y + target.h)) return null;

  const tEnter = Math.max(tNearX, tNearY);
  const tExit = Math.min(tFarX, tFarY);

  if (tEnter > tExit || tEnter < 0 || tEnter > 1) return null;

  // The axis with the later entry is the face that stopped the motion.
  if (tNearX > tNearY) return { t: tEnter, nx: vx > 0 ? -1 : 1, ny: 0 };
  return { t: tEnter, nx: 0, ny: vy > 0 ? -1 : 1 };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node engine/core/collision.test.ts`
Expected: PASS, 14 tests.

- [ ] **Step 5: Run typecheck and the full node suite**

Run: `npx tsc --noEmit && npx vitest run --project node`
Expected: no typecheck output; all node tests pass.

- [ ] **Step 6: Commit**

```bash
git add engine/core/collision.ts engine/core/collision.test.ts
git commit -m "feat: add AABB and swept-AABB collision

Swept because a Breakout ball crosses more than its own width per frame and
overlap-only testing lets it tunnel through bricks."
```

---


### Task 4: i18n with per-game dictionaries

**Files:**
- Create: `engine/core/i18n.ts`, `engine/i18n/pt.ts`, `engine/i18n/en.ts`, `engine/i18n/es.ts`
- Test: `engine/core/i18n.test.ts`

**Interfaces:**
- Consumes: `storage` from Task 2.
- Produces: `createI18n(): I18n`, where

```ts
export interface I18n {
  t: Translate;
  scoped(ns: string): Translate;
  register(ns: string, s: GameStrings): void;
  locale(): string;
  available(): string[];
  bcp47(code?: string): string;
  applyDom(root?: ParentNode): void;
  setLocale(code: string): Promise<void>;
  onChange(fn: (locale: string) => void): () => void;
  init(): Promise<void>;
}
```

> The tracer's version is the starting point (`<TRACER>/app/js/core/i18n.ts`, 90 lines). Three things
> change. Dictionaries arrive at runtime with a game's chunk rather than being three static files, and
> lookups are namespaced so `snake.gameOver` cannot collide with `pong.gameOver` — both forced by having
> 383 games. And the module becomes a factory with an explicit `onChange` subscription instead of a
> module-level `locale` plus a `window` CustomEvent: an ambient event is an undeclared dependency that
> every consumer has to know about, and it makes the module untestable without stubbing `window`.

- [ ] **Step 1: Write the shell dictionaries**

`engine/i18n/pt.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Shell strings only. A game's own strings live in games/<cat>/<slug>/strings.ts and are registered
// at load time — never added here, or every visitor would download all 383 games' text.
export default {
  'shell.skipToGame': 'Pular para o jogo',
  'shell.backToCatalog': 'Voltar ao catálogo',
  'shell.pause': 'Pausa',
  'shell.resume': 'Continuar',
  'shell.restart': 'Reiniciar',
  'shell.pauseMenu': 'Menu de pausa',
  'shell.gameRegion': 'Área de jogo',
  'shell.score': 'Pontos',
  'shell.language': 'Idioma',
  'shell.contrast': 'Alto contraste',
  'shell.contrastOff': 'Desligado',
  'shell.loading': 'Carregando o jogo…',
  'shell.notFound': 'Jogo não encontrado: {slug}',
  'shell.gameOver': 'Fim de jogo. {score} pontos.',
  'shell.crashed': 'O jogo falhou e foi interrompido. Volte ao catálogo ou reinicie.',
  'viz.none': 'Cores normais',
  'viz.simProtan': 'Simular protanopia',
  'viz.simDeuter': 'Simular deuteranopia',
  'viz.simTritan': 'Simular tritanopia',
  'viz.fixProtan': 'Corrigir para protanopia',
  'viz.fixDeuter': 'Corrigir para deuteranopia',
  'viz.fixTritan': 'Corrigir para tritanopia',
  'viz.lowVision': 'Baixa visão (ampliar)',
};
```

`engine/i18n/en.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
export default {
  'shell.skipToGame': 'Skip to the game',
  'shell.backToCatalog': 'Back to the catalog',
  'shell.pause': 'Pause',
  'shell.resume': 'Resume',
  'shell.restart': 'Restart',
  'shell.pauseMenu': 'Pause menu',
  'shell.gameRegion': 'Game area',
  'shell.score': 'Score',
  'shell.language': 'Language',
  'shell.contrast': 'High contrast',
  'shell.contrastOff': 'Off',
  'shell.loading': 'Loading the game…',
  'shell.notFound': 'Game not found: {slug}',
  'shell.gameOver': 'Game over. {score} points.',
  'shell.crashed': 'The game failed and was stopped. Go back to the catalog or restart.',
  'viz.none': 'Normal colours',
  'viz.simProtan': 'Simulate protanopia',
  'viz.simDeuter': 'Simulate deuteranopia',
  'viz.simTritan': 'Simulate tritanopia',
  'viz.fixProtan': 'Correct for protanopia',
  'viz.fixDeuter': 'Correct for deuteranopia',
  'viz.fixTritan': 'Correct for tritanopia',
  'viz.lowVision': 'Low vision (magnify)',
};
```

`engine/i18n/es.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
export default {
  'shell.skipToGame': 'Saltar al juego',
  'shell.backToCatalog': 'Volver al catálogo',
  'shell.pause': 'Pausa',
  'shell.resume': 'Continuar',
  'shell.restart': 'Reiniciar',
  'shell.pauseMenu': 'Menú de pausa',
  'shell.gameRegion': 'Área de juego',
  'shell.score': 'Puntos',
  'shell.language': 'Idioma',
  'shell.contrast': 'Alto contraste',
  'shell.contrastOff': 'Apagado',
  'shell.loading': 'Cargando el juego…',
  'shell.notFound': 'Juego no encontrado: {slug}',
  'shell.gameOver': 'Fin del juego. {score} puntos.',
  'shell.crashed': 'El juego falló y se detuvo. Vuelve al catálogo o reinicia.',
  'viz.none': 'Colores normales',
  'viz.simProtan': 'Simular protanopía',
  'viz.simDeuter': 'Simular deuteranopía',
  'viz.simTritan': 'Simular tritanopía',
  'viz.fixProtan': 'Corregir para protanopía',
  'viz.fixDeuter': 'Corregir para deuteranopía',
  'viz.fixTritan': 'Corregir para tritanopía',
  'viz.lowVision': 'Baja visión (ampliar)',
};
```

- [ ] **Step 2: Write the failing test**

`engine/core/i18n.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it, vi } from 'vitest';
import { createI18n, type GameStrings } from './i18n.js';

function stubEnv(): void {
  const map = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => map.get(k) ?? null,
    setItem: (k: string, v: string) => { map.set(k, v); },
    removeItem: (k: string) => { map.delete(k); },
  });
  vi.stubGlobal('document', { documentElement: { lang: '' } });
}

const snake: GameStrings = {
  pt: { gameOver: 'Fim de jogo', ate: 'Comeu {n} maçãs' },
  en: { gameOver: 'Game over', ate: 'Ate {n} apples' },
  es: { gameOver: 'Fin del juego', ate: 'Comió {n} manzanas' },
};

/**
 * No beforeEach cleanup ritual here, and that is the point of the factory: every test builds its own
 * instance, so nothing leaks between cases.
 */
describe('createI18n', () => {
  it('offers exactly the three floor languages', () => {
    stubEnv();
    expect(createI18n().available().sort()).toEqual(['en', 'es', 'pt']);
  });

  it('translates a shell key', () => {
    stubEnv();
    expect(createI18n().t('shell.resume')).toBe('Continuar');
  });

  it('returns the key itself when nothing matches, so a miss is visible rather than blank', () => {
    stubEnv();
    expect(createI18n().t('shell.doesNotExist')).toBe('shell.doesNotExist');
  });

  it('interpolates {param}, including repeats', () => {
    stubEnv();
    const i18n = createI18n();
    i18n.register('demo', { pt: { hi: '{a} e {a} e {b}' }, en: {}, es: {} });
    expect(i18n.scoped('demo')('hi', { a: 'x', b: 2 })).toBe('x e x e 2');
  });

  it('namespaces game dictionaries so two games can share a key name', () => {
    stubEnv();
    const i18n = createI18n();
    i18n.register('snake', snake);
    i18n.register('pong', { pt: { gameOver: 'Acabou' }, en: {}, es: {} });
    expect(i18n.scoped('snake')('gameOver')).toBe('Fim de jogo');
    expect(i18n.scoped('pong')('gameOver')).toBe('Acabou');
  });

  it('keeps two instances fully independent', () => {
    stubEnv();
    const a = createI18n();
    const b = createI18n();
    a.register('x', { pt: { k: 'from a' }, en: {}, es: {} });
    expect(b.scoped('x')('k')).toBe('k');
  });

  it('switches every registered namespace when the locale changes', async () => {
    stubEnv();
    const i18n = createI18n();
    i18n.register('snake', snake);
    await i18n.setLocale('en');
    expect(i18n.locale()).toBe('en');
    expect(i18n.t('shell.resume')).toBe('Resume');
    expect(i18n.scoped('snake')('ate', { n: 3 })).toBe('Ate 3 apples');
  });

  it('falls back to pt for a key the active locale is missing', async () => {
    stubEnv();
    const i18n = createI18n();
    i18n.register('half', { pt: { only: 'só em pt' }, en: {}, es: {} });
    await i18n.setLocale('en');
    expect(i18n.scoped('half')('only')).toBe('só em pt');
  });

  it('rejects an unknown locale by falling back to pt', async () => {
    stubEnv();
    const i18n = createI18n();
    await i18n.setLocale('de');
    expect(i18n.locale()).toBe('pt');
  });

  it('tags pt with a region and leaves en and es without one', () => {
    stubEnv();
    const i18n = createI18n();
    expect(i18n.bcp47('pt')).toBe('pt-BR');
    expect(i18n.bcp47('en')).toBe('en');
    expect(i18n.bcp47('es')).toBe('es');
  });

  it('notifies subscribers on change, and stops after unsubscribe', async () => {
    stubEnv();
    const i18n = createI18n();
    const seen: string[] = [];
    const off = i18n.onChange((l) => seen.push(l));
    await i18n.setLocale('en');
    off();
    await i18n.setLocale('es');
    expect(seen).toEqual(['en']);
  });

  it('registering the same namespace twice replaces rather than merges', () => {
    stubEnv();
    const i18n = createI18n();
    i18n.register('twice', { pt: { k: 'first' }, en: {}, es: {} });
    i18n.register('twice', { pt: { k: 'second' }, en: {}, es: {} });
    expect(i18n.scoped('twice')('k')).toBe('second');
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `npx vitest run --project node engine/core/i18n.test.ts`
Expected: FAIL — `Failed to resolve import "./i18n.js"`.

- [ ] **Step 4: Write the implementation**

`engine/core/i18n.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// core/i18n — translation with a chained fallback and per-namespace dictionaries.
//
// Adapted from the tracer's core/i18n.ts. Three deliberate differences:
//
//   1. A game's strings are NOT in the locale files. Each game exports its own `strings` and the
//      session registers them under the game's slug, so a visitor downloads only the text of the games
//      they open.
//   2. Lookups are namespaced. `scoped('snake')('gameOver')` reads snake's dictionary, so two games may
//      use the same key name without one silently winning.
//   3. It is a factory with an explicit onChange subscription, not a module singleton broadcasting a
//      window CustomEvent. An ambient event is a dependency nobody declares, and it forces every test
//      of this module to stub `window`.
//
// pt is imported statically so the shell has text before any await resolves; en and es load on demand.
import pt from '../i18n/pt.js';
import * as store from '../platform/storage.js';

export type LocaleDict = Record<string, string>;
export type GameStrings = { pt: LocaleDict; en: LocaleDict; es: LocaleDict };
export type Translate = (key: string, params?: Record<string, string | number>) => string;

const AVAILABLE = ['pt', 'en', 'es'] as const;
type LocaleCode = (typeof AVAILABLE)[number];
const isLocale = (c: string): c is LocaleCode => (AVAILABLE as readonly string[]).includes(c);

const shellLoaders = import.meta.glob<{ default: LocaleDict }>('../i18n/*.ts');

function interpolate(s: string, params?: Record<string, string | number>): string {
  if (!params) return s;
  let out = s;
  for (const k in params) out = out.replaceAll('{' + k + '}', String(params[k]));
  return out;
}

export interface I18n {
  t: Translate;
  scoped(ns: string): Translate;
  register(ns: string, s: GameStrings): void;
  locale(): string;
  available(): string[];
  bcp47(code?: string): string;
  applyDom(root?: ParentNode): void;
  setLocale(code: string): Promise<void>;
  onChange(fn: (locale: string) => void): () => void;
  init(): Promise<void>;
}

export function createI18n(): I18n {
  const shell: Record<string, LocaleDict> = { pt };
  const games = new Map<string, GameStrings>();
  const listeners = new Set<(l: string) => void>();
  let locale: LocaleCode = 'pt';

  /** Chained lookup: active locale, then pt, then nothing. */
  const lookup = (dicts: { pt: LocaleDict; [k: string]: LocaleDict | undefined }, key: string): string | null => {
    const active = dicts[locale];
    if (active && key in active) return active[key]!;
    return key in dicts.pt ? dicts.pt[key]! : null;
  };

  const t: Translate = (key, params) => interpolate(lookup({ ...shell, pt }, key) ?? key, params);

  const bcp47 = (code: string = locale): string => (code === 'pt' ? 'pt-BR' : code);

  const applyDom = (root: ParentNode = document): void => {
    root.querySelectorAll('[data-i18n]').forEach((el) => {
      const k = el.getAttribute('data-i18n');
      if (k) el.textContent = t(k);
    });
    root.querySelectorAll('[data-i18n-aria]').forEach((el) => {
      const k = el.getAttribute('data-i18n-aria');
      if (k) el.setAttribute('aria-label', t(k));
    });
  };

  async function ensureShell(code: LocaleCode): Promise<void> {
    if (shell[code]) return;
    const load = shellLoaders[`../i18n/${code}.ts`];
    if (load) shell[code] = (await load()).default;
  }

  return {
    t,
    scoped: (ns) => (key, params) => {
      const s = games.get(ns);
      return interpolate(s ? (lookup(s, key) ?? key) : key, params);
    },
    register: (ns, s) => { games.set(ns, s); },
    locale: () => locale,
    available: () => [...AVAILABLE],
    bcp47,
    applyDom,

    async setLocale(code) {
      const next = isLocale(code) ? code : 'pt';
      await ensureShell(next);
      locale = next;
      store.set(store.KEYS.lang, next);
      document.documentElement.lang = bcp47(next);
      if (typeof document.querySelectorAll === 'function') applyDom(document);
      for (const fn of listeners) fn(locale);
    },

    onChange(fn) {
      listeners.add(fn);
      return () => { listeners.delete(fn); };
    },

    /**
     * Boot: pt is already live synchronously. Switch only if another language is preferred, and AWAIT
     * it — the tracer fires and forgets, which makes the first frame race the dictionary.
     */
    async init() {
      const saved = store.get(store.KEYS.lang, null);
      const nav = (globalThis.navigator?.language ?? 'pt').slice(0, 2).toLowerCase();
      const want = saved && isLocale(saved) ? saved : nav;
      if (isLocale(want) && want !== 'pt') await this.setLocale(want);
    },
  };
}
```

- [ ] **Step 5: Run the test, typecheck and format**

Run: `npx vitest run --project node engine/core/i18n.test.ts`
Expected: PASS, 13 tests.

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

- [ ] **Step 6: Commit**

```bash
git add engine/core/i18n.ts engine/core/i18n.test.ts engine/i18n/
git commit -m "feat: add i18n as a factory with per-namespace game dictionaries

Adapted from the tracer. A game ships its own three locales with its chunk and
registers them under its slug, so nobody downloads 383 games' text and two
games may reuse a key name. Subscription is explicit instead of a window
CustomEvent: an ambient event is a dependency nobody declares."
```

---


### Task 5: Screen-reader announcements

**Files:**
- Create: `engine/core/a11y-sr.ts`
- Test: `engine/core/a11y-sr.browser.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `createAnnouncer(root?: ParentNode): Announcer` with `say(text)` (polite) and `alert(text)` (assertive), writing into `#sr-status` / `#sr-alert`, which Task 10 puts in `play.html`.

> A browser test, not a node one: the clear → `requestAnimationFrame` → write pattern is the whole
> module, and a fake timer would test the fake. It takes a `root` so a test can hand it a fragment,
> which also means it never reaches for a global document it did not ask for.

- [ ] **Step 1: Write the failing test**

`engine/core/a11y-sr.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it } from 'vitest';
import { createAnnouncer } from './a11y-sr.js';

const nextFrame = (): Promise<void> => new Promise((r) => requestAnimationFrame(() => r()));

beforeEach(() => {
  document.body.innerHTML =
    '<div id="sr-status" role="status" aria-live="polite" aria-atomic="true"></div>' +
    '<div id="sr-alert" role="alert" aria-live="assertive" aria-atomic="true"></div>';
});

describe('createAnnouncer', () => {
  it('writes into the polite region', async () => {
    createAnnouncer().say('ten points');
    await nextFrame();
    expect(document.querySelector('#sr-status')!.textContent).toBe('ten points');
  });

  it('writes into the assertive region', async () => {
    createAnnouncer().alert('game over');
    await nextFrame();
    expect(document.querySelector('#sr-alert')!.textContent).toBe('game over');
  });

  it('clears before writing, so the same text is announced twice', async () => {
    const a = createAnnouncer();
    a.say('same');
    await nextFrame();
    a.say('same');
    expect(document.querySelector('#sr-status')!.textContent).toBe('');
    await nextFrame();
    expect(document.querySelector('#sr-status')!.textContent).toBe('same');
  });

  it('keeps the two regions independent', async () => {
    const a = createAnnouncer();
    a.say('polite');
    a.alert('urgent');
    await nextFrame();
    expect(document.querySelector('#sr-status')!.textContent).toBe('polite');
    expect(document.querySelector('#sr-alert')!.textContent).toBe('urgent');
  });

  it('announces into the root it was given, not the document', async () => {
    const frag = document.createElement('div');
    frag.innerHTML = '<div id="sr-status"></div>';
    createAnnouncer(frag).say('scoped');
    await nextFrame();
    expect(frag.querySelector('#sr-status')!.textContent).toBe('scoped');
    expect(document.querySelector('#sr-status')!.textContent).toBe('');
  });

  it('does not throw when the regions are absent', () => {
    document.body.innerHTML = '';
    const a = createAnnouncer();
    expect(() => { a.say('nobody listening'); a.alert('nobody listening'); }).not.toThrow();
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project browser engine/core/a11y-sr.browser.test.ts`
Expected: FAIL — `Failed to resolve import "./a11y-sr.js"`.

- [ ] **Step 3: Write the implementation**

`engine/core/a11y-sr.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// core/a11y-sr — announcements for screen readers. Adapted from the tracer's core/a11y-sr.ts, minus
// its Libras injection hook, which belongs to that project.
//
// The clear → requestAnimationFrame → write dance is not ceremony: writing the same string twice in a
// row is a no-op to a screen reader, so scoring ten points twice would be announced once. Clearing
// first forces the re-announcement.
//
// The regions live in play.html, so a game never creates them.
export interface Announcer {
  /** Polite: does not interrupt what the reader is currently saying. */
  say(text: string): void;
  /** Assertive: interrupts. Reserve it for what the player must hear now. */
  alert(text: string): void;
}

export function createAnnouncer(root: ParentNode = document): Announcer {
  const announce = (sel: string, text: string): void => {
    const el = root.querySelector(sel);
    if (!el) return;
    el.textContent = '';
    requestAnimationFrame(() => { el.textContent = text; });
  };
  return {
    say: (text) => announce('#sr-status', text),
    alert: (text) => announce('#sr-alert', text),
  };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project browser engine/core/a11y-sr.browser.test.ts`
Expected: PASS, 6 tests.

- [ ] **Step 5: Commit**

```bash
git add engine/core/a11y-sr.ts engine/core/a11y-sr.browser.test.ts
git commit -m "feat: add screen-reader announcements as a factory

Clear-then-write on the next frame, so repeating an announcement is actually
announced twice. Takes its root as an argument rather than reaching for a
global document."
```

---

### Task 6: Input

**Files:**
- Create: `engine/input/actions.ts`, `engine/input/keyboard.ts`, `engine/input/latch.ts`, `engine/input/attach.ts`
- Test: `engine/input/actions.test.ts`, `engine/input/latch.test.ts`, `engine/input/keyboard.test.ts`, `engine/input/attach.browser.test.ts`

**Interfaces:**
- Consumes: `storage` from Task 2.
- Produces:
  - `actions.ts`: `type Action`, `ACTIONS`, `type KeyScheme`, `type PadState`, `PAD_DEAD`, and the **pure** `heldIn(scheme, keys, pad, act)`.
  - `keyboard.ts`: `type KBDefaults`, `KB_DEFAULTS`, `loadKB()`, `saveKB(kb)`, `resetKB()`, `schemesFor(kb, players)`, `ownedCodes(kb, players)` — all pure functions over an explicit `kb`.
  - `latch.ts`: `makeLatch(): Latch`.
  - `attach.ts`: `attachInput(el, players, kb): InputApi & { poll(): void; detach(): void }`.

> The tracer's `input/state.ts` exports a `Set` and an object for other modules to mutate. That module
> hides nothing, which is what Parnas contrasts modularity against, and it is why its tests need a
> `keys.clear()` ritual. Here the key set and the pad state are **private to the `attachInput`
> instance**, and the only thing exported at module level is a pure function that takes them as
> arguments. The eight actions, the schemes and the `0.5` dead zone are still the tracer's, so muscle
> memory carries across the constellation.

- [ ] **Step 1: Write the failing pure tests**

`engine/input/actions.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { ACTIONS, heldIn, PAD_DEAD, type KeyScheme } from './actions.js';

const scheme: KeyScheme = {
  left: ['KeyA'], right: ['KeyD'], up: ['KeyW'], down: ['KeyS'],
  run: ['KeyU'], jump: ['KeyJ', 'Space'], swap: ['KeyI'], especial: ['KeyK'],
};

describe('actions', () => {
  it('names exactly the eight actions of the constellation', () => {
    expect([...ACTIONS]).toEqual(['up', 'left', 'down', 'right', 'run', 'jump', 'swap', 'especial']);
  });

  it('keeps the dead zone at half the stick travel', () => {
    expect(PAD_DEAD).toBe(0.5);
  });
});

describe('heldIn', () => {
  it('reports a held key', () => {
    expect(heldIn(scheme, new Set(['KeyA']), null, 'left')).toBe(true);
    expect(heldIn(scheme, new Set(['KeyA']), null, 'right')).toBe(false);
  });

  it('accepts any of the alternate keys bound to one action', () => {
    expect(heldIn(scheme, new Set(['Space']), null, 'jump')).toBe(true);
    expect(heldIn(scheme, new Set(['KeyJ']), null, 'jump')).toBe(true);
  });

  it('reports a gamepad action when a pad state is supplied', () => {
    expect(heldIn(scheme, new Set(), { jump: true }, 'jump')).toBe(true);
  });

  it('ignores the pad when none is supplied', () => {
    expect(heldIn(scheme, new Set(), null, 'jump')).toBe(false);
  });

  it('is pure: it mutates neither argument', () => {
    const keys = new Set(['KeyA']);
    const pad = { jump: true };
    heldIn(scheme, keys, pad, 'left');
    expect([...keys]).toEqual(['KeyA']);
    expect(pad).toEqual({ jump: true });
  });
});
```

`engine/input/latch.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { makeLatch } from './latch.js';

describe('makeLatch', () => {
  it('reports a press only on the frame the state turns on', () => {
    const l = makeLatch();
    expect(l.pressed('a', false)).toBe(false);
    expect(l.pressed('a', true)).toBe(true);
    expect(l.pressed('a', true)).toBe(false);
  });

  it('re-arms after release', () => {
    const l = makeLatch();
    l.pressed('a', true);
    l.pressed('a', false);
    expect(l.pressed('a', true)).toBe(true);
  });

  it('reports a release only on the frame the state turns off', () => {
    const l = makeLatch();
    l.released('a', true);
    expect(l.released('a', false)).toBe(true);
    expect(l.released('a', false)).toBe(false);
  });

  it('tracks separate keys independently', () => {
    const l = makeLatch();
    expect(l.pressed('a', true)).toBe(true);
    expect(l.pressed('b', true)).toBe(true);
    expect(l.pressed('a', true)).toBe(false);
  });

  it('gives each instance its own memory', () => {
    const a = makeLatch();
    const b = makeLatch();
    a.pressed('k', true);
    expect(b.pressed('k', true)).toBe(true);
  });
});
```

`engine/input/keyboard.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { KB_DEFAULTS, loadKB, ownedCodes, resetKB, saveKB, schemesFor } from './keyboard.js';

beforeEach(() => {
  const map = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => map.get(k) ?? null,
    setItem: (k: string, v: string) => { map.set(k, v); },
    removeItem: (k: string) => { map.delete(k); },
  });
});

describe('keyboard schemes', () => {
  it('gives one scheme for solo and one per player otherwise', () => {
    expect(schemesFor(KB_DEFAULTS, 1).length).toBe(1);
    expect(schemesFor(KB_DEFAULTS, 2).length).toBe(2);
    expect(schemesFor(KB_DEFAULTS, 3).length).toBe(3);
    expect(schemesFor(KB_DEFAULTS, 4).length).toBe(4);
  });

  it('never binds a modifier, which would collide with the browser and with AT', () => {
    for (const n of [1, 2, 3, 4]) {
      for (const code of ownedCodes(KB_DEFAULTS, n)) {
        expect(code, code).not.toMatch(/^(Alt|Control|Shift|Meta)/);
      }
    }
  });

  it('gives two players disjoint bindings', () => {
    const [a, b] = schemesFor(KB_DEFAULTS, 2);
    const codesA = new Set(Object.values(a!).flat());
    for (const code of Object.values(b!).flat()) expect(codesA.has(code), code).toBe(false);
  });

  it('collects every bound code for a player count', () => {
    const owned = ownedCodes(KB_DEFAULTS, 1);
    expect(owned.has('KeyA')).toBe(true);
    expect(owned.has('Space')).toBe(true);
    expect(owned.has('Tab')).toBe(false);
  });

  it('loads defaults when nothing is saved', () => {
    expect(loadKB().solo.left).toEqual(KB_DEFAULTS.solo.left);
  });

  it('layers a saved partial ON TOP of the defaults, so a new action is never missing', () => {
    saveKB({ ...KB_DEFAULTS, solo: { ...KB_DEFAULTS.solo, left: ['KeyQ'] } });
    const kb = loadKB();
    expect(kb.solo.left).toEqual(['KeyQ']);
    expect(kb.solo.jump).toEqual(KB_DEFAULTS.solo.jump);
  });

  it('reset returns the defaults and forgets the saved map', () => {
    saveKB({ ...KB_DEFAULTS, solo: { ...KB_DEFAULTS.solo, left: ['KeyQ'] } });
    expect(resetKB().solo.left).toEqual(KB_DEFAULTS.solo.left);
    expect(loadKB().solo.left).toEqual(KB_DEFAULTS.solo.left);
  });

  it('loadKB never returns the shared defaults object', () => {
    const kb = loadKB();
    kb.solo.left.push('KeyZ');
    expect(KB_DEFAULTS.solo.left).not.toContain('KeyZ');
  });
});
```

- [ ] **Step 2: Write the failing browser test**

`engine/input/attach.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { afterEach, beforeEach, describe, expect, it } from 'vitest';
import { attachInput } from './attach.js';
import { KB_DEFAULTS } from './keyboard.js';

let region: HTMLElement;
let api: ReturnType<typeof attachInput>;

const key = (type: 'keydown' | 'keyup', code: string): KeyboardEvent =>
  new KeyboardEvent(type, { code, bubbles: true, cancelable: true });

beforeEach(() => {
  document.body.innerHTML = '<div id="game-region" tabindex="0"></div><input id="outside" />';
  region = document.querySelector('#game-region')!;
  api = attachInput(region, 1, KB_DEFAULTS);
});
afterEach(() => api.detach());

describe('attachInput', () => {
  it('sees a key pressed on the game region', () => {
    region.dispatchEvent(key('keydown', 'KeyA'));
    expect(api.held(0, 'left')).toBe(true);
    region.dispatchEvent(key('keyup', 'KeyA'));
    expect(api.held(0, 'left')).toBe(false);
  });

  it('ignores a key pressed outside the game region, so the page keeps its keys', () => {
    document.querySelector('#outside')!.dispatchEvent(key('keydown', 'KeyA'));
    expect(api.held(0, 'left')).toBe(false);
  });

  it('reports a press exactly once per physical press', () => {
    region.dispatchEvent(key('keydown', 'KeyJ'));
    expect(api.pressed(0, 'jump')).toBe(true);
    expect(api.pressed(0, 'jump')).toBe(false);
    region.dispatchEvent(key('keyup', 'KeyJ'));
    region.dispatchEvent(key('keydown', 'KeyJ'));
    expect(api.pressed(0, 'jump')).toBe(true);
  });

  it('prevents the default for keys it owns, so Space does not scroll the page', () => {
    const ev = key('keydown', 'Space');
    region.dispatchEvent(ev);
    expect(ev.defaultPrevented).toBe(true);
  });

  it('leaves keys it does not own alone, so Tab still moves focus', () => {
    const ev = key('keydown', 'Tab');
    region.dispatchEvent(ev);
    expect(ev.defaultPrevented).toBe(false);
  });

  it('drops every held key on detach, so a game teardown cannot leak input', () => {
    region.dispatchEvent(key('keydown', 'KeyA'));
    api.detach();
    expect(api.held(0, 'left')).toBe(false);
  });

  it('clears held keys when the region loses focus, so alt-tab does not stick a direction', () => {
    region.dispatchEvent(key('keydown', 'KeyD'));
    expect(api.held(0, 'right')).toBe(true);
    region.dispatchEvent(new FocusEvent('blur'));
    expect(api.held(0, 'right')).toBe(false);
  });

  it('gives two players different default keys', () => {
    api.detach();
    api = attachInput(region, 2, KB_DEFAULTS);
    region.dispatchEvent(key('keydown', 'KeyA'));
    region.dispatchEvent(key('keydown', 'ArrowLeft'));
    expect(api.held(0, 'left')).toBe(true);
    expect(api.held(1, 'left')).toBe(true);
    region.dispatchEvent(key('keyup', 'KeyA'));
    expect(api.held(0, 'left')).toBe(false);
    expect(api.held(1, 'left')).toBe(true);
  });

  it('keeps two instances independent, so a detached one cannot answer for the live one', () => {
    const other = attachInput(region, 1, KB_DEFAULTS);
    other.detach();
    region.dispatchEvent(key('keydown', 'KeyA'));
    expect(api.held(0, 'left')).toBe(true);
    expect(other.held(0, 'left')).toBe(false);
  });
});
```

- [ ] **Step 3: Run both to verify they fail**

Run: `npx vitest run engine/input/`
Expected: FAIL — unresolved imports for `actions.js`, `latch.js`, `keyboard.js`, `attach.js`.

- [ ] **Step 4: Write `engine/input/actions.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/actions — the vocabulary of input, and the one query over it. Pure, zero deps, zero state.
//
// The eight actions are the tracer's, unchanged: someone who learned one game in the constellation
// should not have to relearn the keys for the next.
export type Action = 'up' | 'left' | 'down' | 'right' | 'run' | 'jump' | 'swap' | 'especial';
export const ACTIONS: readonly Action[] = ['up', 'left', 'down', 'right', 'run', 'jump', 'swap', 'especial'];

/** One player's binding: action → the physical KeyboardEvent.code values that trigger it. */
export type KeyScheme = Record<Action, string[]>;

/** Gamepad actions held this frame. */
export type PadState = Partial<Record<Action, boolean>>;

/** Dead zone = the first HALF of the stick's travel. Ergonomics, not sensitivity (tracer's value). */
export const PAD_DEAD = 0.5;

/**
 * Is this action held, by keyboard or by the pad?
 *
 * The state arrives as arguments rather than living in this module. That is the difference between a
 * module that hides something and a module that is a bag of globals: this one can be reasoned about,
 * tested and used twice over without a cleanup ritual between uses.
 */
export function heldIn(scheme: KeyScheme, keys: ReadonlySet<string>, pad: PadState | null, act: Action): boolean {
  if (scheme[act].some((k) => keys.has(k))) return true;
  return pad?.[act] === true;
}
```

- [ ] **Step 5: Write `engine/input/latch.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/latch — edge detection over a polled boolean. Factory, zero deps.
//
// A held key is true on every frame. A menu that advances on "jump" would advance sixty times a second
// without this. `pressed` fires on the false → true edge, `released` on true → false.
export interface Latch {
  pressed(key: string, now: boolean): boolean;
  released(key: string, now: boolean): boolean;
  clear(): void;
}

export function makeLatch(): Latch {
  const prevPressed = new Map<string, boolean>();
  const prevReleased = new Map<string, boolean>();
  return {
    pressed(key, now) {
      const was = prevPressed.get(key) ?? false;
      prevPressed.set(key, now);
      return now && !was;
    },
    released(key, now) {
      const was = prevReleased.get(key) ?? false;
      prevReleased.set(key, now);
      return !now && was;
    },
    clear() { prevPressed.clear(); prevReleased.clear(); },
  };
}
```

- [ ] **Step 6: Write `engine/input/keyboard.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/keyboard — key schemes per player count, plus persistence. Pure functions over an explicit
// `kb` value; nothing here is stored at module scope.
//
// Copied from the tracer's defaults so muscle memory carries across the constellation. No Alt, AltGr,
// Control or Shift anywhere: those collide with the browser and with assistive technology.
import * as store from '../platform/storage.js';
import type { Action, KeyScheme } from './actions.js';

export type KBDefaults = { solo: KeyScheme; p2: KeyScheme[]; p3: KeyScheme[]; p4: KeyScheme[] };

const SCHEMES4: KeyScheme[] = [
  { left: ['KeyA'], right: ['KeyD'], up: ['KeyW'], down: ['KeyS'], run: ['KeyZ'], jump: ['KeyX'], swap: ['KeyC'], especial: ['KeyV'] },
  { left: ['KeyJ'], right: ['KeyL'], up: ['KeyI'], down: ['KeyK'], run: ['KeyM'], jump: ['Comma'], swap: ['Period'], especial: ['Semicolon', 'Slash'] },
  { left: ['ArrowLeft'], right: ['ArrowRight'], up: ['ArrowUp'], down: ['ArrowDown'], run: ['Home'], jump: ['End'], swap: ['PageUp'], especial: ['PageDown'] },
  { left: ['Numpad4'], right: ['Numpad6'], up: ['Numpad8'], down: ['Numpad5'], run: ['Numpad2'], jump: ['Numpad0'], swap: ['Numpad3'], especial: ['NumpadDecimal'] },
];

const clone = <T>(v: T): T => JSON.parse(JSON.stringify(v)) as T;

export const KB_DEFAULTS: KBDefaults = {
  solo: {
    left: ['KeyA', 'ArrowLeft'], right: ['KeyD', 'ArrowRight'], up: ['KeyW', 'ArrowUp'], down: ['KeyS', 'ArrowDown'],
    run: ['KeyU'], jump: ['KeyJ', 'Space'], swap: ['KeyI'], especial: ['KeyK'],
  },
  p2: [
    { left: ['KeyA'], right: ['KeyD'], up: ['KeyW'], down: ['KeyS'], run: ['KeyU'], jump: ['KeyJ'], swap: ['KeyI'], especial: ['KeyK'] },
    { left: ['ArrowLeft'], right: ['ArrowRight'], up: ['ArrowUp'], down: ['ArrowDown'], run: ['Numpad8'], jump: ['Numpad5'], swap: ['Numpad9'], especial: ['Numpad6'] },
  ],
  p3: clone(SCHEMES4.slice(0, 3)),
  p4: clone(SCHEMES4),
};

/** Load the saved schemes ON TOP of the defaults, so an action added later is never missing. */
export function loadKB(): KBDefaults {
  const d = clone(KB_DEFAULTS);
  const s = store.getJSON<Partial<KBDefaults>>(store.KEYS.kbcontrols, null);
  if (s) {
    if (s.solo) Object.assign(d.solo, s.solo);
    (['p2', 'p3', 'p4'] as const).forEach((g) => {
      const arr = s[g];
      if (Array.isArray(arr)) arr.forEach((m, i) => { const slot = d[g][i]; if (slot && m) Object.assign(slot, m); });
    });
  }
  return d;
}

export function saveKB(next: KBDefaults): void { store.setJSON(store.KEYS.kbcontrols, next); }
export function resetKB(): KBDefaults { store.remove(store.KEYS.kbcontrols); return clone(KB_DEFAULTS); }

/** The schemes for a given player count. */
export function schemesFor(kb: KBDefaults, players: number): KeyScheme[] {
  if (players <= 1) return [kb.solo];
  if (players === 2) return kb.p2;
  if (players === 3) return kb.p3;
  return kb.p4;
}

/** Every physical code bound for this player count — what the handler may swallow, and nothing more. */
export function ownedCodes(kb: KBDefaults, players: number): Set<string> {
  const out = new Set<string>();
  for (const s of schemesFor(kb, players)) {
    for (const act of Object.keys(s) as Action[]) for (const c of s[act]) out.add(c);
  }
  return out;
}
```

- [ ] **Step 7: Write `engine/input/attach.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/attach — binds input to a DOM element and to the gamepads. Owns the key set and the pad state
// PRIVATELY: no other module can reach them, which is the difference between this and the tracer's
// exported mutable `keys`.
//
// Listeners go on #game-region, NEVER on window. On window the game swallows keys belonging to the
// page: the skip link stops working, and a screen-reader user navigating the surrounding document
// finds their keys eaten by a canvas they are not focused on.
import { heldIn, PAD_DEAD, type Action, type PadState } from './actions.js';
import { makeLatch } from './latch.js';
import { ownedCodes, schemesFor, type KBDefaults } from './keyboard.js';

export interface InputApi {
  held(pl: number, act: Action): boolean;
  pressed(pl: number, act: Action): boolean;
  readonly players: number;
}

export function attachInput(el: HTMLElement, players: number, kb: KBDefaults): InputApi & { poll(): void; detach(): void } {
  const schemes = schemesFor(kb, players);
  const owned = ownedCodes(kb, players);
  const latch = makeLatch();
  const keys = new Set<string>();
  const pads = new Map<number, PadState>();

  const onKeyDown = (e: KeyboardEvent): void => {
    if (!owned.has(e.code)) return;   // Tab, F5 and everything else stay the browser's
    e.preventDefault();
    keys.add(e.code);
  };
  const onKeyUp = (e: KeyboardEvent): void => { keys.delete(e.code); };
  // Losing focus mid-press never delivers the keyup, which would stick a direction on forever.
  const onBlur = (): void => { keys.clear(); latch.clear(); };

  el.addEventListener('keydown', onKeyDown);
  el.addEventListener('keyup', onKeyUp);
  el.addEventListener('blur', onBlur);

  const schemeFor = (pl: number) => schemes[Math.min(pl, schemes.length - 1)]!;
  const isHeld = (pl: number, act: Action): boolean => heldIn(schemeFor(pl), keys, pads.get(pl) ?? null, act);

  return {
    players,
    held: isHeld,
    pressed: (pl, act) => latch.pressed(`${pl}:${act}`, isHeld(pl, act)),

    /** Read the gamepads. Call once per frame, before the game's update. */
    poll() {
      const list = navigator.getGamepads?.() ?? [];
      for (let i = 0; i < list.length; i++) {
        const p = list[i];
        if (!p) { pads.delete(i); continue; }
        const ax = p.axes[0] ?? 0, ay = p.axes[1] ?? 0;
        const b = (n: number): boolean => p.buttons[n]?.pressed === true;
        pads.set(i, {
          left: ax < -PAD_DEAD || b(14), right: ax > PAD_DEAD || b(15),
          up: ay < -PAD_DEAD || b(12), down: ay > PAD_DEAD || b(13),
          jump: b(0), run: b(2), swap: b(1), especial: b(3),
        });
      }
    },

    detach() {
      el.removeEventListener('keydown', onKeyDown);
      el.removeEventListener('keyup', onKeyUp);
      el.removeEventListener('blur', onBlur);
      keys.clear();
      latch.clear();
      pads.clear();
    },
  };
}
```

- [ ] **Step 8: Run the tests, typecheck and format**

Run: `npx vitest run engine/input/`
Expected: PASS. `actions` 7, `latch` 5, `keyboard` 8, `attach` 9.

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

- [ ] **Step 9: Commit**

```bash
git add engine/input/
git commit -m "feat: add the eight-action input layer

Schemes and the 0.5 dead zone come from the tracer so muscle memory carries
across the constellation. Unlike the tracer, the key set and pad state are
private to the attachInput instance and the module-level query is pure, so
there is no cleanup ritual between tests and two instances cannot interfere."
```

---

### Task 7: Canvas primitives and the scaled PixiJS mount

**Files:**
- Create: `engine/render/canvas.ts`, `engine/render/mount.ts`
- Test: `engine/render/canvas.browser.test.ts`, `engine/render/mount.browser.test.ts`

**Interfaces:**
- Consumes: `LOGICAL_W`, `LOGICAL_H` from Task 1.
- Produces:
  - From `canvas.ts`: `makeCanvas(w, h)`, `tex(cv)`, `pixDisc(ctx, cx, cy, r, col, edge?)`, `type PixelBrush`, `type PixelPainter`, `pixelCanvas(w, h, paint)`, `pixelTexture(w, h, paint)`.
  - From `mount.ts`: `mountPixi(host: HTMLElement): { app: PIXI.Application; stage: PIXI.Container; resize(): void; destroy(): void }` and `integerScale(hostW, hostH): number`.

> `canvas.ts` is lifted from `<TRACER>/app/js/render/canvas.ts` with the header translated. `mount.ts`
> is new: the tracer scales through CSS on `.pixi-mount`, which works for one game but leaves the scale
> factor implicit. Here the factor is a tested function, because "never fractional" is a constraint the
> plan asserts and an untested constraint is a wish.

- [ ] **Step 1: Copy `canvas.ts` from the tracer**

Copy `<TRACER>/app/js/render/canvas.ts` to `engine/render/canvas.ts` verbatim, translating the header
comment to English. Keep `makeCanvas`, `tex`, `pixDisc`, `PixelBrush`, `PixelPainter`, `pixelCanvas`
and `pixelTexture` exactly as they are — every game's art goes through `pixelCanvas`.

- [ ] **Step 2: Write the failing tests**

`engine/render/canvas.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { makeCanvas, pixelCanvas } from './canvas.js';

/** Read one pixel as [r,g,b,a]. */
function at(cv: HTMLCanvasElement, x: number, y: number): number[] {
  return [...cv.getContext('2d')!.getImageData(x, y, 1, 1).data];
}

describe('canvas primitives', () => {
  it('makes a canvas of the exact size asked for', () => {
    const cv = makeCanvas(7, 3);
    expect([cv.width, cv.height]).toEqual([7, 3]);
  });

  it('pixelCanvas paints where the brush is told to', () => {
    const cv = pixelCanvas(4, 4, (px) => px(1, 1, 2, 2, '#ff0000'));
    expect(at(cv, 0, 0)[3]).toBe(0);         // untouched stays transparent
    expect(at(cv, 1, 1).slice(0, 3)).toEqual([255, 0, 0]);
    expect(at(cv, 2, 2).slice(0, 3)).toEqual([255, 0, 0]);
    expect(at(cv, 3, 3)[3]).toBe(0);
  });

  it('pixelCanvas leaves an unpainted canvas fully transparent', () => {
    const cv = pixelCanvas(2, 2, () => { /* paints nothing */ });
    expect(at(cv, 0, 0)[3]).toBe(0);
  });

  it('later brush strokes paint over earlier ones', () => {
    const cv = pixelCanvas(2, 2, (px) => { px(0, 0, 2, 2, '#ff0000'); px(0, 0, 1, 1, '#0000ff'); });
    expect(at(cv, 0, 0).slice(0, 3)).toEqual([0, 0, 255]);
    expect(at(cv, 1, 1).slice(0, 3)).toEqual([255, 0, 0]);
  });
});
```

`engine/render/mount.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { afterEach, describe, expect, it } from 'vitest';
import { integerScale, mountPixi } from './mount.js';
import { LOGICAL_H, LOGICAL_W } from '../core/constants.js';

let teardown: (() => void) | null = null;
afterEach(() => { teardown?.(); teardown = null; document.body.innerHTML = ''; });

describe('integerScale', () => {
  it('is exactly 4 on a 1280x720 host', () => {
    expect(integerScale(1280, 720)).toBe(4);
  });

  it('is exactly 6 on a 1920x1080 host', () => {
    expect(integerScale(1920, 1080)).toBe(6);
  });

  it('rounds DOWN rather than producing a fractional factor', () => {
    expect(integerScale(1279, 719)).toBe(3);
    expect(integerScale(1000, 1000)).toBe(3);
  });

  it('is limited by the tighter axis', () => {
    expect(integerScale(4000, 400)).toBe(2);   // height allows 2, width would allow 12
  });

  it('never drops below 1, even on a host smaller than the logical canvas', () => {
    expect(integerScale(100, 50)).toBe(1);
    expect(integerScale(0, 0)).toBe(1);
  });

  it('always returns a whole number', () => {
    for (const [w, h] of [[1366, 768], [1440, 900], [800, 600], [2560, 1440]]) {
      expect(Number.isInteger(integerScale(w!, h!))).toBe(true);
    }
  });
});

describe('mountPixi', () => {
  it('creates a canvas at the logical resolution, not the host resolution', () => {
    const host = document.createElement('div');
    Object.defineProperty(host, 'clientWidth', { value: 1280 });
    Object.defineProperty(host, 'clientHeight', { value: 720 });
    document.body.appendChild(host);
    const m = mountPixi(host);
    teardown = () => m.destroy();
    const cv = host.querySelector('canvas')!;
    expect(cv.width).toBe(LOGICAL_W);
    expect(cv.height).toBe(LOGICAL_H);
  });

  it('sizes the canvas on screen by a whole multiple of the logical size', () => {
    const host = document.createElement('div');
    Object.defineProperty(host, 'clientWidth', { value: 1280 });
    Object.defineProperty(host, 'clientHeight', { value: 720 });
    document.body.appendChild(host);
    const m = mountPixi(host);
    teardown = () => m.destroy();
    const cv = host.querySelector('canvas')!;
    expect(cv.style.width).toBe(`${LOGICAL_W * 4}px`);
    expect(cv.style.height).toBe(`${LOGICAL_H * 4}px`);
  });

  it('asks the browser not to smooth the upscale', () => {
    const host = document.createElement('div');
    document.body.appendChild(host);
    const m = mountPixi(host);
    teardown = () => m.destroy();
    const cv = host.querySelector('canvas')!;
    expect(cv.style.imageRendering).toBe('pixelated');
  });

  it('removes its canvas on destroy', () => {
    const host = document.createElement('div');
    document.body.appendChild(host);
    const m = mountPixi(host);
    m.destroy();
    teardown = null;
    expect(host.querySelector('canvas')).toBeNull();
  });
});
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `npx vitest run --project browser engine/render/`
Expected: FAIL — `Failed to resolve import "./mount.js"`.

- [ ] **Step 4: Write `engine/render/mount.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/mount — puts a PixiJS application into a host element at the logical resolution, then
// scales it up by a WHOLE number.
//
// The renderer always draws 320x180. Only the CSS box grows. Any fractional factor makes the browser
// resample, and resampled pixel art is mush — one logical pixel would land on 3.7 physical ones and
// the edges would shimmer as things move. So the factor is floored, and the canvas is centred inside
// whatever space is left over.
import * as PIXI from 'pixi.js';
import { LOGICAL_H, LOGICAL_W } from '../core/constants.js';

/** The largest whole multiple of the logical canvas that fits in the host. Never below 1. */
export function integerScale(hostW: number, hostH: number): number {
  const byWidth = Math.floor(hostW / LOGICAL_W);
  const byHeight = Math.floor(hostH / LOGICAL_H);
  return Math.max(1, Math.min(byWidth, byHeight));
}

export interface PixiMount {
  app: PIXI.Application;
  stage: PIXI.Container;
  resize(): void;
  destroy(): void;
}

export function mountPixi(host: HTMLElement): PixiMount {
  const app = new PIXI.Application({
    width: LOGICAL_W,
    height: LOGICAL_H,
    antialias: false,
    resolution: 1,
    autoDensity: false,
    backgroundColor: 0x05070f,
  });
  PIXI.BaseTexture.defaultOptions.scaleMode = PIXI.SCALE_MODES.NEAREST;

  const view = app.view as unknown as HTMLCanvasElement;
  view.style.imageRendering = 'pixelated';
  view.style.display = 'block';
  view.style.margin = 'auto';
  host.appendChild(view);

  function resize(): void {
    const k = integerScale(host.clientWidth, host.clientHeight);
    view.style.width = `${LOGICAL_W * k}px`;
    view.style.height = `${LOGICAL_H * k}px`;
  }
  resize();

  const onWindowResize = (): void => resize();
  window.addEventListener('resize', onWindowResize);

  return {
    app,
    stage: app.stage,
    resize,
    destroy() {
      window.removeEventListener('resize', onWindowResize);
      app.destroy(true, { children: true, texture: true, baseTexture: true });
      view.remove();
    },
  };
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `npx vitest run --project browser engine/render/`
Expected: PASS. `canvas` 4, `mount` 10.

- [ ] **Step 6: Run typecheck**

Run: `npx tsc --noEmit`
Expected: no output, exit 0.

- [ ] **Step 7: Commit**

```bash
git add engine/render/canvas.ts engine/render/canvas.browser.test.ts engine/render/mount.ts engine/render/mount.browser.test.ts
git commit -m "feat: add canvas primitives and the integer-scaled PixiJS mount

canvas.ts is the tracer's, unchanged. mount.ts is new: the scale factor is a
tested function rather than an implicit CSS rule, because 'never fractional'
is a constraint and an untested constraint is a wish."
```

---


### Task 8: High contrast by sprite role, and the Scene

**Files:**
- Create: `engine/render/high-contrast.ts`, `engine/render/scene-pixi.ts`
- Test: `engine/render/high-contrast.test.ts`, `engine/render/scene-pixi.browser.test.ts`

**Interfaces:**
- Consumes: `pixelCanvas`, `tex` from Task 7.
- Produces:
  - Pure, from `high-contrast.ts`: `type SpriteRole`, `type ContrastLevel`, `relativeLuminance(hex)`, `contrastRatio(a, b)`, `roleColor(role, level)`, `outlineColor()`, `HC_BG`.
  - Factory, from `high-contrast.ts`: `createVisualState(initial?): VisualState` with `level()`, `setLevel(l)`, `onChange(fn): () => void`.
  - From `scene-pixi.ts`: `type SpriteSpec`, `type Handle`, `type Scene`, `roleCanvas(spec, level)`, and `createPixiScene(stage, visual): Scene & { destroy(): void }`.

> Two things happen here and they are deliberately in one task, because neither is testable without
> the other being decided.
>
> **The palette is computed, not chosen.** Each role owns a hue; the lightness is bisected until the
> colour's relative luminance is exactly what the target ratio demands. That makes the promise
> checkable, and the test measures the real WCAG ratio instead of trusting hand-picked hex.
>
> **The `Scene` is the boundary that keeps PixiJS out of every game.** `add()` takes a description and
> returns a `Handle` with three properties. A game moves and hides things; it cannot reach the renderer.
> This is what lets `renderer: 'svg'` in `GameMeta` be a real escape hatch rather than a type that
> promises what the contract cannot deliver.

- [ ] **Step 1: Write the failing palette test**

`engine/render/high-contrast.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import {
  contrastRatio, createVisualState, HC_BG, outlineColor,
  relativeLuminance, roleColor, type ContrastLevel, type SpriteRole,
} from './high-contrast.js';

const PAINTED: SpriteRole[] = ['player', 'ally', 'hazard', 'goal', 'pickup', 'ui', 'neutral'];
const LEVELS: ContrastLevel[] = [3, 4.5, 7];

describe('luminance and ratio', () => {
  it('puts black at 0 and white at 1', () => {
    expect(relativeLuminance('#000000')).toBeCloseTo(0, 6);
    expect(relativeLuminance('#ffffff')).toBeCloseTo(1, 6);
  });

  it('gives the textbook 21:1 for black on white', () => {
    expect(contrastRatio('#000000', '#ffffff')).toBeCloseTo(21, 2);
  });

  it('is symmetric', () => {
    expect(contrastRatio('#123456', '#abcdef')).toBeCloseTo(contrastRatio('#abcdef', '#123456'), 10);
  });

  it('gives 1:1 for a colour against itself', () => {
    expect(contrastRatio('#3366aa', '#3366aa')).toBeCloseTo(1, 10);
  });

  it('accepts shorthand hex', () => {
    expect(relativeLuminance('#fff')).toBeCloseTo(1, 6);
  });
});

describe('roleColor', () => {
  it('returns null when contrast is off, so the game keeps its own art', () => {
    for (const r of PAINTED) expect(roleColor(r, 0)).toBeNull();
  });

  it('MEETS the ratio its level promises, for every painted role', () => {
    for (const level of LEVELS) {
      for (const role of PAINTED) {
        const c = roleColor(role, level)!;
        expect(c, `${role} at ${level}`).toMatch(/^#[0-9a-f]{6}$/);
        expect(contrastRatio(c, HC_BG), `${role} at ${level}`).toBeGreaterThanOrEqual(level - 0.05);
      }
    }
  });

  it('recesses the background role instead of raising it', () => {
    expect(contrastRatio(roleColor('bg', 7)!, HC_BG)).toBeLessThan(3);
  });

  it('keeps every painted role distinguishable from every other at the same level', () => {
    for (const level of LEVELS) {
      const seen = new Map<string, SpriteRole>();
      for (const role of PAINTED) {
        const c = roleColor(role, level)!;
        expect(seen.has(c), `${role} duplicates ${seen.get(c)} at ${level}`).toBe(false);
        seen.set(c, role);
      }
    }
  });

  it('is deterministic', () => {
    expect(roleColor('hazard', 4.5)).toBe(roleColor('hazard', 4.5));
  });

  it('gives an outline that contrasts with the background', () => {
    expect(contrastRatio(outlineColor(), HC_BG)).toBeGreaterThanOrEqual(3);
  });
});

describe('createVisualState', () => {
  it('starts off unless told otherwise', () => {
    expect(createVisualState().level()).toBe(0);
    expect(createVisualState(7).level()).toBe(7);
  });

  it('notifies on change and not on a no-op set', () => {
    const v = createVisualState();
    let calls = 0;
    v.onChange(() => calls++);
    v.setLevel(4.5);
    expect(calls).toBe(1);
    v.setLevel(4.5);
    expect(calls).toBe(1);
  });

  it('stops notifying after unsubscribe', () => {
    const v = createVisualState();
    let calls = 0;
    const off = v.onChange(() => calls++);
    off();
    v.setLevel(7);
    expect(calls).toBe(0);
  });

  it('keeps two instances independent', () => {
    const a = createVisualState();
    const b = createVisualState();
    a.setLevel(7);
    expect(b.level()).toBe(0);
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node engine/render/high-contrast.test.ts`
Expected: FAIL — `Failed to resolve import "./high-contrast.js"`.

- [ ] **Step 3: Write `engine/render/high-contrast.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/high-contrast — semantic roles resolved to colours that MEET a stated WCAG ratio.
//
// This inverts what the tracer does. There, contrast was retrofitted onto a finished platformer, so the
// colour-blocking is welded to that game's four entities. Here every sprite is born through one factory,
// so it can carry a role tag from the start and no game ever writes contrast code.
//
// The colours are computed. Each role owns a hue; the lightness is bisected until the relative luminance
// is what the target ratio requires against the backdrop. The test then measures the ratio, so the claim
// is checked rather than asserted.

export type SpriteRole = 'player' | 'ally' | 'hazard' | 'goal' | 'pickup' | 'bg' | 'ui' | 'neutral';
/** 0 = off (the game's own art). Otherwise the WCAG contrast ratio the palette must meet. */
export type ContrastLevel = 0 | 3 | 4.5 | 7;

/** The backdrop every ratio is measured against. High contrast forces the field to this colour. */
export const HC_BG = '#000000';

/** Hue and saturation per role. `ui` is a grey; `neutral` is desaturated but not grey. */
const ROLE_HS: Record<Exclude<SpriteRole, 'bg'>, { h: number; s: number }> = {
  player: { h: 210, s: 1 },     // blue — the thing you are
  ally: { h: 150, s: 1 },       // green — safe
  hazard: { h: 0, s: 1 },       // red — kills you
  goal: { h: 45, s: 1 },        // amber — where you are going
  pickup: { h: 300, s: 1 },     // magenta — take it
  ui: { h: 0, s: 0 },           // white-ish — chrome, never gameplay
  neutral: { h: 180, s: 0.35 }, // desaturated cyan — scenery that still must be seen
};

/** The background role is RECESSED: it must not compete with anything the player reacts to. */
const BG_RECESSED = '#0b0b12';

function hexToRgb(hex: string): [number, number, number] {
  let h = hex.replace('#', '');
  if (h.length === 3) h = h[0]! + h[0]! + h[1]! + h[1]! + h[2]! + h[2]!;
  return [parseInt(h.slice(0, 2), 16), parseInt(h.slice(2, 4), 16), parseInt(h.slice(4, 6), 16)];
}

const toHex = (n: number): string => Math.round(n).toString(16).padStart(2, '0');

/** WCAG 2.x relative luminance. */
export function relativeLuminance(hex: string): number {
  const lin = hexToRgb(hex).map((v) => {
    const c = v / 255;
    return c <= 0.04045 ? c / 12.92 : ((c + 0.055) / 1.055) ** 2.4;
  }) as [number, number, number];
  return 0.2126 * lin[0] + 0.7152 * lin[1] + 0.0722 * lin[2];
}

/** WCAG 2.x contrast ratio, always >= 1 and order-independent. */
export function contrastRatio(a: string, b: string): number {
  const la = relativeLuminance(a), lb = relativeLuminance(b);
  return (Math.max(la, lb) + 0.05) / (Math.min(la, lb) + 0.05);
}

function hslToHex(h: number, s: number, l: number): string {
  const c = (1 - Math.abs(2 * l - 1)) * s;
  const x = c * (1 - Math.abs(((h / 60) % 2) - 1));
  const m = l - c / 2;
  const seg = Math.floor(h / 60) % 6;
  const [r, g, b] = ([[c, x, 0], [x, c, 0], [0, c, x], [0, x, c], [x, 0, c], [c, 0, x]] as const)[seg]!;
  return `#${toHex((r + m) * 255)}${toHex((g + m) * 255)}${toHex((b + m) * 255)}`;
}

/**
 * The lightness at which this hue reaches a target luminance.
 * Luminance rises monotonically with HSL lightness, so bisection always converges — and at l = 1 every
 * hue is white, whose luminance is 1, so no target below 1 is out of reach.
 */
function solveLightness(h: number, s: number, targetLum: number): number {
  let lo = 0, hi = 1;
  for (let i = 0; i < 40; i++) {
    const mid = (lo + hi) / 2;
    if (relativeLuminance(hslToHex(h, s, mid)) < targetLum) lo = mid; else hi = mid;
  }
  return (lo + hi) / 2;
}

// Memoised because the result is a pure function of (role, level) and repainting a wall of bricks
// would otherwise bisect forty times per brick.
const cache = new Map<string, string>();

/**
 * The colour this role must be painted at this contrast level, or null when contrast is off.
 * Against a black backdrop, ratio R needs luminance (0.05R - 0.05), which is what is solved for.
 */
export function roleColor(role: SpriteRole, level: ContrastLevel): string | null {
  if (level === 0) return null;
  if (role === 'bg') return BG_RECESSED;
  const key = `${role}:${level}`;
  const hit = cache.get(key);
  if (hit) return hit;
  const { h, s } = ROLE_HS[role];
  const out = hslToHex(h, s, solveLightness(h, s, 0.05 * level - 0.05 + relativeLuminance(HC_BG)));
  cache.set(key, out);
  return out;
}

/** The one-pixel border drawn around every shape, so two adjacent roles never read as one blob. */
export function outlineColor(): string { return '#ffffff'; }

export interface VisualState {
  level(): ContrastLevel;
  setLevel(next: ContrastLevel): void;
  onChange(fn: () => void): () => void;
}

/** The current contrast level, owned by whoever creates it — the composition root, in practice. */
export function createVisualState(initial: ContrastLevel = 0): VisualState {
  let level = initial;
  const listeners = new Set<() => void>();
  return {
    level: () => level,
    setLevel(next) {
      if (next === level) return;   // repainting every sprite is not free; a no-op set stays a no-op
      level = next;
      for (const fn of listeners) fn();
    },
    onChange(fn) {
      listeners.add(fn);
      return () => { listeners.delete(fn); };
    },
  };
}
```

- [ ] **Step 4: Run the palette test to verify it passes**

Run: `npx vitest run --project node engine/render/high-contrast.test.ts`
Expected: PASS, 13 tests. Every role at every level provably meets its ratio.

- [ ] **Step 5: Write the failing Scene test**

`engine/render/scene-pixi.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { Container } from 'pixi.js';
import { createPixiScene, roleCanvas, type SpriteSpec } from './scene-pixi.js';
import { createVisualState, roleColor } from './high-contrast.js';

const hexAt = (cv: HTMLCanvasElement, x: number, y: number): string => {
  const d = cv.getContext('2d')!.getImageData(x, y, 1, 1).data;
  return d[3] === 0 ? 'transparent' : `#${[d[0], d[1], d[2]].map((v) => v!.toString(16).padStart(2, '0')).join('')}`;
};

const square: SpriteSpec = {
  role: 'hazard', w: 4, h: 4,
  paint: (px) => px(1, 1, 2, 2, '#00ff00'),
};

describe('roleCanvas', () => {
  it('keeps the game colours when contrast is off', () => {
    expect(hexAt(roleCanvas(square, 0), 1, 1)).toBe('#00ff00');
  });

  it('is the size the spec asked for when contrast is off', () => {
    const cv = roleCanvas(square, 0);
    expect([cv.width, cv.height]).toEqual([4, 4]);
  });

  it('replaces every game colour with the role colour when contrast is on', () => {
    // Offset by one pixel: the outline grows the canvas by a border.
    expect(hexAt(roleCanvas(square, 4.5), 2, 2)).toBe(roleColor('hazard', 4.5));
  });

  it('grows by one pixel of border on each side so the outline has somewhere to live', () => {
    const cv = roleCanvas(square, 4.5);
    expect([cv.width, cv.height]).toEqual([6, 6]);
  });

  it('draws an outline around the silhouette', () => {
    expect(hexAt(roleCanvas(square, 4.5), 1, 2)).toBe('#ffffff');
  });

  it('leaves the area outside the outline transparent', () => {
    expect(hexAt(roleCanvas(square, 4.5), 0, 0)).toBe('transparent');
  });

  it('recesses a bg-role sprite instead of brightening it', () => {
    expect(hexAt(roleCanvas({ ...square, role: 'bg' }, 7), 2, 2)).toBe(roleColor('bg', 7));
  });
});

describe('createPixiScene', () => {
  it('adds a child to the stage for each handle', () => {
    const stage = new Container();
    const scene = createPixiScene(stage, createVisualState());
    scene.add(square);
    expect(stage.children.length).toBe(1);
    scene.destroy();
  });

  it('returns a handle that exposes position and visibility AND NOTHING ELSE', () => {
    const stage = new Container();
    const scene = createPixiScene(stage, createVisualState());
    const h = scene.add(square);
    expect(Object.keys(h).sort()).toEqual(['visible', 'x', 'y']);
    expect((h as Record<string, unknown>)['texture']).toBeUndefined();
    scene.destroy();
  });

  it('moves the underlying sprite when the handle moves', () => {
    const stage = new Container();
    const scene = createPixiScene(stage, createVisualState());
    const h = scene.add(square);
    h.x = 12; h.y = 34;
    expect([stage.children[0]!.x, stage.children[0]!.y]).toEqual([12, 34]);
    scene.destroy();
  });

  it('hides the underlying sprite when the handle is hidden', () => {
    const stage = new Container();
    const scene = createPixiScene(stage, createVisualState());
    const h = scene.add(square);
    h.visible = false;
    expect(stage.children[0]!.visible).toBe(false);
    scene.destroy();
  });

  it('removes a handle from the stage', () => {
    const stage = new Container();
    const scene = createPixiScene(stage, createVisualState());
    const h = scene.add(square);
    scene.remove(h);
    expect(stage.children.length).toBe(0);
    scene.destroy();
  });

  it('clear removes everything', () => {
    const stage = new Container();
    const scene = createPixiScene(stage, createVisualState());
    scene.add(square); scene.add(square);
    scene.clear();
    expect(stage.children.length).toBe(0);
    scene.destroy();
  });

  it('repaints every handle when the contrast level changes', () => {
    const stage = new Container();
    const visual = createVisualState();
    const scene = createPixiScene(stage, visual);
    scene.add(square);
    const before = (stage.children[0] as { texture: unknown }).texture;
    visual.setLevel(7);
    expect((stage.children[0] as { texture: unknown }).texture).not.toBe(before);
    scene.destroy();
  });

  it('stops repainting after destroy, so a torn-down game leaves nothing behind', () => {
    const stage = new Container();
    const visual = createVisualState();
    const scene = createPixiScene(stage, visual);
    scene.add(square);
    scene.destroy();
    expect(() => visual.setLevel(3)).not.toThrow();
    expect(stage.children.length).toBe(0);
  });
});
```

- [ ] **Step 6: Run it to verify it fails**

Run: `npx vitest run --project browser engine/render/scene-pixi.browser.test.ts`
Expected: FAIL — `Failed to resolve import "./scene-pixi.js"`.

- [ ] **Step 7: Write `engine/render/scene-pixi.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/scene-pixi — the PixiJS implementation of the drawing contract.
//
// The contract is the point. `Scene` is four methods and `Handle` is three properties, and neither
// mentions PixiJS. Handing a game a PIXI.Sprite would pin 383 games to one library's API and would
// contradict the renderer escape hatch in GameMeta: a game declaring renderer 'svg' would still be
// holding a Pixi object. A second renderer implements this same file's exported types and nothing in
// any game changes.
//
// The handle is a real wrapper, not the sprite widened by a type. A structural type would still hand
// over the live object, and `as any` would reach straight through it.
import { Sprite, type Container, type Texture } from 'pixi.js';
import { pixelCanvas, tex } from './canvas.js';
import { outlineColor, roleColor, type ContrastLevel, type SpriteRole, type VisualState } from './high-contrast.js';

export type PixelBrush = (x: number, y: number, w: number, h: number, col: string) => void;

export interface SpriteSpec {
  role: SpriteRole;
  w: number;
  h: number;
  paint: (px: PixelBrush) => void;
}

/** An opaque drawable. A game moves and hides it; it cannot reach the renderer through it. */
export interface Handle { x: number; y: number; visible: boolean }

export interface Scene {
  add(spec: SpriteSpec): Handle;
  remove(h: Handle): void;
  clear(): void;
}

/**
 * Render one spec at one contrast level.
 *
 * Off: the game's own painter runs untouched.
 *
 * On: the painter runs nine times over a canvas grown by a one-pixel border — eight offset passes in
 * the outline colour, which together form a sticker outline around whatever silhouette the game drew,
 * then one centred pass in the role colour. The game's requested colours are discarded, which is the
 * point: two roles that happen to share a hue must not read as the same thing.
 */
export function roleCanvas(spec: SpriteSpec, level: ContrastLevel): HTMLCanvasElement {
  if (level === 0) return pixelCanvas(spec.w, spec.h, spec.paint);

  const fill = roleColor(spec.role, level)!;
  const line = outlineColor();
  const OFFSETS: ReadonlyArray<readonly [number, number]> = [
    [0, 0], [2, 0], [0, 2], [2, 2], [1, 0], [0, 1], [2, 1], [1, 2],
  ];

  return pixelCanvas(spec.w + 2, spec.h + 2, (px) => {
    for (const [ox, oy] of OFFSETS) spec.paint((x, y, w, h) => px(x + ox, y + oy, w, h, line));
    spec.paint((x, y, w, h) => px(x + 1, y + 1, w, h, fill));
  });
}

/**
 * Build the scene for one game session. `destroy()` releases every sprite and unsubscribes, so a game
 * that is torn down cannot leave drawables behind that repaint forever.
 */
export function createPixiScene(stage: Container, visual: VisualState): Scene & { destroy(): void } {
  const sprites = new Map<Handle, Sprite>();
  const specs = new Map<Handle, SpriteSpec>();

  const paint = (spec: SpriteSpec): Texture => tex(roleCanvas(spec, visual.level()));

  const unsubscribe = visual.onChange(() => {
    for (const [handle, sprite] of sprites) {
      const old = sprite.texture;
      sprite.texture = paint(specs.get(handle)!);
      old.destroy(true);
    }
  });

  function detach(handle: Handle): void {
    const sprite = sprites.get(handle);
    if (!sprite) return;
    stage.removeChild(sprite);
    sprite.destroy({ texture: true, baseTexture: true });
    sprites.delete(handle);
    specs.delete(handle);
  }

  return {
    add(spec) {
      const sprite = new Sprite(paint(spec));
      sprite.anchor.set(0, 0);
      // The border added in high-contrast mode must not shift the game's layout, so the sprite keeps
      // the spec's dimensions rather than the texture's.
      sprite.width = spec.w;
      sprite.height = spec.h;
      stage.addChild(sprite);

      const handle: Handle = {
        get x() { return sprite.x; },
        set x(v) { sprite.x = v; },
        get y() { return sprite.y; },
        set y(v) { sprite.y = v; },
        get visible() { return sprite.visible; },
        set visible(v) { sprite.visible = v; },
      };
      sprites.set(handle, sprite);
      specs.set(handle, spec);
      return handle;
    },

    remove: detach,
    clear() { for (const handle of [...sprites.keys()]) detach(handle); },
    destroy() { unsubscribe(); for (const handle of [...sprites.keys()]) detach(handle); },
  };
}
```

- [ ] **Step 8: Run everything, typecheck and format**

Run: `npx vitest run engine/render/`
Expected: PASS. `canvas` 4, `mount` 10, `high-contrast` 13, `scene-pixi` 15.

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

- [ ] **Step 9: Commit**

```bash
git add engine/render/high-contrast.ts engine/render/high-contrast.test.ts engine/render/scene-pixi.ts engine/render/scene-pixi.browser.test.ts
git commit -m "feat: high contrast from role tags, behind a renderer-agnostic Scene

Each role owns a hue and the lightness is solved until the colour hits the
luminance the target ratio demands, so the test measures the real WCAG ratio
instead of trusting hand-picked hex.

Scene and Handle mention no PixiJS type. The handle is a real wrapper rather
than the sprite widened by a type, because a structural type still hands over
the live object and 'as any' reaches straight through it."
```

---

### Task 9: Colour-vision and low-vision filters

**Files:**
- Create: `engine/render/cvd-matrices.ts`, `engine/render/viz.ts`
- Test: `engine/render/viz.test.ts`, `engine/render/viz.browser.test.ts`

**Interfaces:**
- Consumes: `storage` (`KEYS.viz`) from Task 2.
- Produces:
  - Lifted, from `cvd-matrices.ts`: `type CvdKey`, `CVD_KEYS`, `CVD_MATRIX`, `CVD_SVG_ID`, `cvdMatrixValues(k)`, `installCvdFilters(host): number`.
  - Pure, from `viz.ts`: `type VizKey`, `VIZ_MODES`, `vizFilter(key)`, `vizZoom(key)`.
  - Factory, from `viz.ts`: `createViz(target: HTMLElement): Viz` with `mode()`, `set(key)`, `restore()`.

> `cvd-matrices.ts` is copied verbatim from `<TRACER>/app/js/render/cvd-matrices.ts` — 120 numbers from
> Machado 2009 (simulation) and Fidaner et al. (correction), which nobody should retype. Only the header
> comment is translated.
>
> `viz.ts` is **not** the tracer's `viz-modes.ts`. That file carries pt-BR `nome`/`desc` strings inline,
> which contradicts the rule that every visible string resolves through `t()`. The mode table is
> rewritten with i18n keys and trimmed to the eight modes that need no game-specific renderer. It is a
> factory over its target element, and it takes **only** what it uses: the first draft carried a `host`
> parameter it never touched, which the audit called out as a lie in an interface.

- [ ] **Step 1: Copy `cvd-matrices.ts` from the tracer**

Copy `<TRACER>/app/js/render/cvd-matrices.ts` to `engine/render/cvd-matrices.ts`. Translate the header
comment to English. Change nothing else.

- [ ] **Step 2: Write the failing node test**

`engine/render/viz.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { CVD_KEYS, CVD_MATRIX, cvdMatrixValues } from './cvd-matrices.js';
import { VIZ_MODES, vizFilter, vizZoom, type VizKey } from './viz.js';

describe('cvd matrices', () => {
  it('carries all six modes: three simulations and three corrections', () => {
    expect(CVD_KEYS.length).toBe(6);
    expect(CVD_KEYS.filter((k) => k.startsWith('sim-')).length).toBe(3);
    expect(CVD_KEYS.filter((k) => k.startsWith('fix-')).length).toBe(3);
  });

  it('gives every mode a 20-value feColorMatrix', () => {
    for (const k of CVD_KEYS) {
      expect(CVD_MATRIX[k].length, k).toBe(20);
      expect(cvdMatrixValues(k).split(/\s+/).length, k).toBe(20);
    }
  });

  it('emits only finite numbers', () => {
    for (const k of CVD_KEYS) for (const n of CVD_MATRIX[k]) expect(Number.isFinite(n)).toBe(true);
  });
});

describe('viz modes', () => {
  it('starts from a none mode that applies nothing', () => {
    expect(vizFilter('none')).toBe('');
    expect(vizZoom('none')).toBe(1);
  });

  it('lists every mode with an i18n key rather than a literal string', () => {
    for (const m of VIZ_MODES) {
      expect(m.i18nKey, m.key).toMatch(/^viz\./);
      expect(m).not.toHaveProperty('nome');
    }
  });

  it('has no duplicate keys', () => {
    const keys = VIZ_MODES.map((m) => m.key);
    expect(new Set(keys).size).toBe(keys.length);
  });

  it('maps every cvd mode to an SVG filter reference', () => {
    for (const m of VIZ_MODES.filter((x) => x.kind === 'cvd')) {
      expect(vizFilter(m.key), m.key).toMatch(/^url\(#cvd-/);
    }
  });

  it('magnifies only in low vision', () => {
    expect(vizZoom('low-vision')).toBeGreaterThan(1);
    for (const m of VIZ_MODES.filter((x) => x.key !== 'low-vision')) {
      expect(vizZoom(m.key), m.key).toBe(1);
    }
  });

  it('treats an unknown key as none rather than throwing', () => {
    expect(vizFilter('nonsense' as VizKey)).toBe('');
    expect(vizZoom('nonsense' as VizKey)).toBe(1);
  });
});
```

- [ ] **Step 3: Write the failing browser test**

`engine/render/viz.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { installCvdFilters } from './cvd-matrices.js';
import { createViz } from './viz.js';

let host: HTMLElement;
let target: HTMLElement;

beforeEach(() => {
  vi.stubGlobal('localStorage', (() => {
    const m = new Map<string, string>();
    return {
      getItem: (k: string) => m.get(k) ?? null,
      setItem: (k: string, v: string) => { m.set(k, v); },
      removeItem: (k: string) => { m.delete(k); },
    };
  })());
  document.body.innerHTML = '<div id="filters"></div><div id="region"></div>';
  host = document.querySelector('#filters')!;
  target = document.querySelector('#region')!;
});

describe('installCvdFilters', () => {
  it('injects six filter definitions', () => {
    expect(installCvdFilters(host)).toBe(6);
    expect(host.querySelectorAll('filter').length).toBe(6);
  });

  it('does nothing and reports zero when the host is missing', () => {
    expect(installCvdFilters(null)).toBe(0);
  });
});

describe('createViz', () => {
  it('applies no filter in the default mode', () => {
    const viz = createViz(target);
    expect(viz.mode()).toBe('none');
    expect(target.style.filter).toBe('');
  });

  it('applies the SVG filter reference for a cvd mode', () => {
    createViz(target).set('fix-deuter');
    expect(target.style.filter).toContain('url(#cvd-');
  });

  it('applies a zoom transform in low vision and removes it on return', () => {
    const viz = createViz(target);
    viz.set('low-vision');
    expect(target.style.transform).toContain('scale(');
    viz.set('none');
    expect(target.style.transform).toBe('');
  });

  it('persists the choice, and restore() brings it back on a fresh instance', () => {
    createViz(target).set('sim-tritan');
    const fresh = document.createElement('div');
    const viz = createViz(fresh);
    viz.restore();
    expect(viz.mode()).toBe('sim-tritan');
    expect(fresh.style.filter).toContain('url(#cvd-');
  });

  it('falls back to none when the stored mode is nonsense', () => {
    localStorage.setItem('demos.viz', 'not-a-mode');
    const viz = createViz(target);
    viz.restore();
    expect(viz.mode()).toBe('none');
  });

  it('keeps two instances independent', () => {
    const other = document.createElement('div');
    const a = createViz(target);
    const b = createViz(other);
    a.set('sim-protan');
    expect(b.mode()).toBe('none');
    expect(other.style.filter).toBe('');
  });
});
```

- [ ] **Step 4: Run both to verify they fail**

Run: `npx vitest run engine/render/viz`
Expected: FAIL — `Failed to resolve import "./viz.js"`.

- [ ] **Step 5: Write `engine/render/viz.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/viz — the visual accessibility modes that cost a game nothing.
//
// These apply to the CANVAS ELEMENT, not to anything a game draws: a CSS filter over the whole surface
// plus a magnification transform. So every one of the 383 games gets colour-blindness simulation,
// colour-blindness correction and low-vision magnification without a line of its own.
//
// High contrast is NOT here — it is a repaint, not a filter, and lives in high-contrast.ts.
import * as store from '../platform/storage.js';
import { CVD_SVG_ID, type CvdKey } from './cvd-matrices.js';

export type VizKey = 'none' | CvdKey | 'low-vision';

export interface VizMode {
  key: VizKey;
  kind: 'none' | 'cvd' | 'zoom';
  /** i18n key for the label. Never a literal string. */
  i18nKey: string;
}

export const VIZ_MODES: readonly VizMode[] = [
  { key: 'none', kind: 'none', i18nKey: 'viz.none' },
  { key: 'sim-protan', kind: 'cvd', i18nKey: 'viz.simProtan' },
  { key: 'sim-deuter', kind: 'cvd', i18nKey: 'viz.simDeuter' },
  { key: 'sim-tritan', kind: 'cvd', i18nKey: 'viz.simTritan' },
  { key: 'fix-protan', kind: 'cvd', i18nKey: 'viz.fixProtan' },
  { key: 'fix-deuter', kind: 'cvd', i18nKey: 'viz.fixDeuter' },
  { key: 'fix-tritan', kind: 'cvd', i18nKey: 'viz.fixTritan' },
  { key: 'low-vision', kind: 'zoom', i18nKey: 'viz.lowVision' },
];

const BY_KEY = new Map<string, VizMode>(VIZ_MODES.map((m) => [m.key, m]));

/** Magnification for low vision. 1.5 keeps the whole 320x180 field visible on a 4:3 host. */
const LOW_VISION_ZOOM = 1.5;

/** The CSS `filter` value for a mode, or '' for none. Unknown keys degrade to none. */
export function vizFilter(key: VizKey): string {
  const m = BY_KEY.get(key);
  return m?.kind === 'cvd' ? `url(#${CVD_SVG_ID[key as CvdKey]})` : '';
}

/** The magnification for a mode. Unknown keys degrade to 1. */
export function vizZoom(key: VizKey): number {
  return BY_KEY.get(key)?.kind === 'zoom' ? LOW_VISION_ZOOM : 1;
}

export interface Viz {
  mode(): VizKey;
  set(key: VizKey): void;
  /** Re-apply the persisted mode. Separate from `set` so boot does not have to know the stored value. */
  restore(): void;
}

export function createViz(target: HTMLElement): Viz {
  let current: VizKey = 'none';

  const apply = (key: VizKey): void => {
    current = BY_KEY.has(key) ? key : 'none';
    const zoom = vizZoom(current);
    target.style.filter = vizFilter(current);
    // Empty string rather than `scale(1)`: a lingering transform creates a containing block and a
    // stacking context, which silently changes how the pause dialog above the canvas is positioned.
    target.style.transform = zoom === 1 ? '' : `scale(${zoom})`;
    target.style.transformOrigin = zoom === 1 ? '' : 'center center';
  };

  return {
    mode: () => current,
    set(key) {
      apply(key);
      store.set(store.KEYS.viz, current);
    },
    restore() {
      const saved = store.get(store.KEYS.viz, null);
      apply(saved && BY_KEY.has(saved) ? (saved as VizKey) : 'none');
    },
  };
}
```

- [ ] **Step 6: Run both, typecheck and format**

Run: `npx vitest run engine/render/viz`
Expected: PASS. node 9, browser 8.

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

- [ ] **Step 7: Commit**

```bash
git add engine/render/cvd-matrices.ts engine/render/viz.ts engine/render/viz.test.ts engine/render/viz.browser.test.ts
git commit -m "feat: add colour-vision and low-vision filters

The 120 Machado/Fidaner matrices come from the tracer untouched. The mode table
is rewritten with i18n keys instead of the tracer's inline pt-BR labels, and it
is a factory that takes only the element it actually touches."
```

---


### Task 10: The accessible shell and the router

**Files:**
- Create: `play.html` (replacing the Task 1 placeholder), `engine/shell/shell.css`, `engine/shell/router.ts`
- Test: `engine/shell/router.test.ts`, `engine/shell/shell.browser.test.ts`

**Interfaces:**
- Consumes: nothing at runtime; the markup is what Tasks 5, 9 and 11 attach to.
- Produces:
  - `play.html` containing `#sr-status`, `#sr-alert`, `#game-region`, `#hud`, `#pause`, `#cvd-filters`, the skip link and the back link.
  - From `router.ts`: `parseHash(hash): { category: string; slug: string } | null`, `gameLoaders(): Record<string, () => Promise<unknown>>`, `loaderFor(category, slug)`.

> `#game-region` carries `role="application"` and `tabindex="0"`. `role="application"` tells a screen
> reader to stop intercepting arrow keys and hand them to the page — without it the game is unplayable
> for that user. It is also why the region must be a *focusable* island rather than the whole document:
> outside it, normal reading keys must keep working.

- [ ] **Step 1: Write the failing router test**

`engine/shell/router.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { parseHash } from './router.js';

describe('parseHash', () => {
  it('parses a category and slug', () => {
    expect(parseHash('#arcade-classico/snake')).toEqual({ category: 'arcade-classico', slug: 'snake' });
  });

  it('tolerates a missing leading hash', () => {
    expect(parseHash('arcade-classico/snake')).toEqual({ category: 'arcade-classico', slug: 'snake' });
  });

  it('returns null for an empty hash, which is the shell landing on no game', () => {
    expect(parseHash('')).toBeNull();
    expect(parseHash('#')).toBeNull();
  });

  it('returns null when a segment is missing', () => {
    expect(parseHash('#snake')).toBeNull();
    expect(parseHash('#arcade-classico/')).toBeNull();
    expect(parseHash('#/snake')).toBeNull();
  });

  it('rejects path traversal rather than trying to resolve it', () => {
    expect(parseHash('#../../etc/passwd')).toBeNull();
    expect(parseHash('#a/../b')).toBeNull();
  });

  it('rejects anything outside lowercase slug characters', () => {
    expect(parseHash('#Arcade/snake')).toBeNull();
    expect(parseHash('#arcade classico/snake')).toBeNull();
    expect(parseHash('#arcade/snake?x=1')).toBeNull();
  });

  it('ignores extra path segments rather than guessing', () => {
    expect(parseHash('#a/b/c')).toBeNull();
  });

  it('accepts digits and hyphens inside a slug', () => {
    expect(parseHash('#puzzle-logico/2048-merge')).toEqual({ category: 'puzzle-logico', slug: '2048-merge' });
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node engine/shell/router.test.ts`
Expected: FAIL — `Failed to resolve import "./router.js"`.

- [ ] **Step 3: Write `engine/shell/router.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// shell/router — hash to game, and the lazy registry of every game in the repository.
//
// import.meta.glob is how Vite discovers modules by pattern at build time. It produces one chunk per
// game, so the shell downloads only the game the visitor opened. Adding a game is therefore creating
// a folder: no registry to edit, no build config to touch.

/** Slugs are lowercase letters, digits and hyphens. Nothing else — see parseHash. */
const SLUG = /^[a-z0-9]+(?:-[a-z0-9]+)*$/;

export interface Route { category: string; slug: string }

/**
 * Parse `#category/slug`. Returns null for anything that is not exactly two valid slugs.
 *
 * The strictness is deliberate. This value selects a key into the glob registry, and a hash comes
 * from whatever the visitor typed or was linked. Validating the shape here means the lookup below can
 * never be asked to resolve `../` into something outside the games folder.
 */
export function parseHash(hash: string): Route | null {
  const raw = hash.startsWith('#') ? hash.slice(1) : hash;
  if (!raw) return null;
  const parts = raw.split('/');
  if (parts.length !== 2) return null;
  const [category, slug] = parts as [string, string];
  if (!SLUG.test(category) || !SLUG.test(slug)) return null;
  return { category, slug };
}

/** Every game module in the repository, keyed by its glob path, loaded on demand. */
export function gameLoaders(): Record<string, () => Promise<unknown>> {
  return import.meta.glob('../../games/*/*/main.ts');
}

/** The loader for one route, or null when no such game exists. */
export function loaderFor(category: string, slug: string): (() => Promise<unknown>) | null {
  return gameLoaders()[`../../games/${category}/${slug}/main.ts`] ?? null;
}
```

- [ ] **Step 4: Run the router test to verify it passes**

Run: `npx vitest run --project node engine/shell/router.test.ts`
Expected: PASS, 8 tests.

- [ ] **Step 5: Write `play.html`**

Replace the Task 1 placeholder entirely:
```html
<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>JS Minigames</title>
<link rel="stylesheet" href="./engine/shell/shell.css">
</head>
<body>

<a class="skip-link" href="#game-region" data-i18n="shell.skipToGame">Pular para o jogo</a>

<!-- Screen-reader live regions. Every game announces through these; none of them creates its own. -->
<div id="sr-status" class="sr-only" role="status" aria-live="polite" aria-atomic="true"></div>
<div id="sr-alert" class="sr-only" role="alert" aria-live="assertive" aria-atomic="true"></div>

<!-- The colour-vision filter definitions are injected here at boot by installCvdFilters. -->
<svg id="cvd-filters" aria-hidden="true" focusable="false" width="0" height="0"></svg>

<header class="bar">
  <a class="back" href="./index.html" data-i18n="shell.backToCatalog">Voltar ao catálogo</a>
  <h1 id="game-title" class="title"></h1>
  <p id="hud" class="hud"><span data-i18n="shell.score">Pontos</span>:
    <strong id="hud-score" aria-live="off">0</strong></p>
</header>

<main>
  <!--
    role="application" makes a screen reader pass arrow keys through to the page instead of using them
    to move its own reading cursor. Without it the game cannot be played by that user at all. It is
    scoped to this element, never the document, so ordinary reading keys keep working everywhere else.
  -->
  <div id="game-region" class="game-region" role="application" tabindex="0"
       data-i18n-aria="shell.gameRegion" aria-label="Área de jogo">

    <div id="pause" class="pause" hidden>
      <div class="pause-card" role="dialog" aria-modal="true" aria-labelledby="pause-h">
        <h2 id="pause-h" data-i18n="shell.pause">Pausa</h2>
        <div id="pause-menu" role="menu" data-i18n-aria="shell.pauseMenu" aria-label="Menu de pausa">
          <button type="button" role="menuitem" data-act="resume" data-i18n="shell.resume">Continuar</button>
          <button type="button" role="menuitem" data-act="restart" data-i18n="shell.restart">Reiniciar</button>
          <button type="button" role="menuitem" data-act="quit" data-i18n="shell.backToCatalog">Voltar ao catálogo</button>
        </div>
      </div>
    </div>

  </div>
</main>

<!-- The module tag arrives in Task 13, with boot.ts. Vite resolves script sources in an HTML entry, so
     pointing at a file that does not exist yet breaks this task's own browser test. -->
</body>
</html>
```

- [ ] **Step 6: Write `engine/shell/shell.css`**

```css
/* SPDX-License-Identifier: GPL-3.0-or-later */
:root { --bg: #0a0a12; --fg: #ececf2; --line: #2a2a40; --focus: #ffd60a; }

* { box-sizing: border-box; margin: 0; padding: 0; }
html, body { background: var(--bg); color: var(--fg); font-family: ui-monospace, monospace; min-height: 100%; }

/* Visually hidden but still read aloud. Never display:none — that removes it from the a11y tree too. */
.sr-only {
  position: absolute; width: 1px; height: 1px; overflow: hidden;
  clip-path: inset(50%); white-space: nowrap;
}

.skip-link {
  position: absolute; left: -999px; top: 0; z-index: 100;
  background: var(--focus); color: #000; padding: 8px 16px;
}
.skip-link:focus { left: 0; }

.bar { display: flex; align-items: baseline; gap: 16px; padding: 12px 16px; border-bottom: 1px solid var(--line); }
.title { font-size: 16px; }
.hud { margin-left: auto; font-size: 14px; }
.back { color: var(--fg); }

main { display: flex; justify-content: center; padding: 16px; }

.game-region { position: relative; background: #05070f; line-height: 0; }
/* A visible focus ring is not decoration: it is how a keyboard user knows the game has their keys. */
.game-region:focus-visible { outline: 4px solid var(--focus); outline-offset: 3px; }

.pause { position: absolute; inset: 0; z-index: 6; display: flex; align-items: center; justify-content: center; background: rgba(4, 7, 15, .82); line-height: 1.5; }
.pause[hidden] { display: none; }
.pause-card { padding: 16px 20px; border: 1px solid var(--line); background: var(--bg); }
.pause-card h2 { font-size: 18px; margin-bottom: 12px; }
#pause-menu { display: flex; flex-direction: column; gap: 6px; }
#pause-menu button { font: inherit; padding: 6px 12px; background: #161624; color: var(--fg); border: 1px solid var(--line); cursor: pointer; }
#pause-menu button:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }

@media (prefers-reduced-motion: reduce) { * { animation: none !important; transition: none !important; } }
```

- [ ] **Step 7: Write the failing shell markup test**

`engine/shell/shell.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeAll, describe, expect, it } from 'vitest';

/**
 * Loads the real play.html and asserts the accessibility contract every game inherits. This markup is
 * written once for 383 games, so a regression here is a regression in all of them at the same time.
 */
let doc: Document;

beforeAll(async () => {
  const html = await (await fetch('/play.html')).text();
  doc = new DOMParser().parseFromString(html, 'text/html');
});

describe('play.html accessibility contract', () => {
  it('has both live regions with the right politeness', () => {
    expect(doc.querySelector('#sr-status')!.getAttribute('aria-live')).toBe('polite');
    expect(doc.querySelector('#sr-alert')!.getAttribute('aria-live')).toBe('assertive');
  });

  it('marks both live regions atomic, so partial updates are not read', () => {
    for (const id of ['#sr-status', '#sr-alert']) {
      expect(doc.querySelector(id)!.getAttribute('aria-atomic'), id).toBe('true');
    }
  });

  it('has a skip link pointing at the game region', () => {
    expect(doc.querySelector('.skip-link')!.getAttribute('href')).toBe('#game-region');
  });

  it('makes the game region a focusable application', () => {
    const r = doc.querySelector('#game-region')!;
    expect(r.getAttribute('role')).toBe('application');
    expect(r.getAttribute('tabindex')).toBe('0');
  });

  it('gives the game region an accessible name', () => {
    const r = doc.querySelector('#game-region')!;
    expect(r.getAttribute('aria-label')).toBeTruthy();
    expect(r.getAttribute('data-i18n-aria')).toBe('shell.gameRegion');
  });

  it('makes the pause overlay a modal dialog with a label', () => {
    const d = doc.querySelector('.pause-card')!;
    expect(d.getAttribute('role')).toBe('dialog');
    expect(d.getAttribute('aria-modal')).toBe('true');
    expect(doc.querySelector(`#${d.getAttribute('aria-labelledby')}`)).not.toBeNull();
  });

  it('starts with the pause overlay hidden', () => {
    expect(doc.querySelector('#pause')!.hasAttribute('hidden')).toBe(true);
  });

  it('keeps the score out of the live region, so every point is not announced', () => {
    expect(doc.querySelector('#hud-score')!.getAttribute('aria-live')).toBe('off');
  });

  it('hides the filter svg from assistive technology', () => {
    const svg = doc.querySelector('#cvd-filters')!;
    expect(svg.getAttribute('aria-hidden')).toBe('true');
    expect(svg.getAttribute('focusable')).toBe('false');
  });

  it('gives every visible string an i18n key', () => {
    const texts = [...doc.querySelectorAll('.skip-link, .back, #pause-h, #pause-menu button')];
    expect(texts.length).toBeGreaterThan(0);
    for (const el of texts) expect(el.getAttribute('data-i18n'), el.outerHTML).toBeTruthy();
  });

  it('declares a language on the document', () => {
    expect(doc.documentElement.getAttribute('lang')).toBe('pt-BR');
  });
});
```

- [ ] **Step 8: Run the shell test**

Run: `npx vitest run --project browser engine/shell/shell.browser.test.ts`
Expected: PASS, 11 tests. (It fails first only if the markup in Step 5 was not written; write it, then run.)

- [ ] **Step 9: Run typecheck and the full suite**

Run: `npx tsc --noEmit && npx vitest run`
Expected: no typecheck output; every test passes.

- [ ] **Step 10: Commit**

```bash
git add play.html engine/shell/shell.css engine/shell/router.ts engine/shell/router.test.ts engine/shell/shell.browser.test.ts
git commit -m "feat: add the accessible shell and the hash router

The markup is written once and inherited by every game: live regions, skip
link, role=application on a focusable game region, and a modal pause dialog.
parseHash rejects anything that is not two plain slugs, so a crafted hash can
never reach outside the games folder."
```

---


### Task 11: HUD, pause dialog and audio

**Files:**
- Create: `engine/shell/hud.ts`, `engine/shell/pause.ts`, `engine/platform/audio.ts`
- Test: `engine/shell/hud.browser.test.ts`, `engine/shell/pause.browser.test.ts`, `engine/platform/audio.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `createHud(root: ParentNode): Hud` with `setTitle(text)`, `setScore(n)`, `reset()`.
  - `createPause(panel: HTMLElement): Pause` with `open(onAction)`, `close()`, `isOpen()`, and `type PauseAction = 'resume' | 'restart' | 'quit'`.
  - `createAudio(): Audio` with `beep(freq, ms)` and `setMuted(on)`.

> All three were module singletons in the first draft; all three are factories now. `pause` is the one
> where it mattered most: its module-level `open` flag meant a test had to call `closePause()` in
> `beforeEach` to undo the previous case, which is the receipt for shared state.
>
> Three properties make the dialog usable without a mouse, and all three are tested: focus enters the
> dialog on open, Tab cycles inside it and cannot reach the page behind, and focus returns where it came
> from on close. A dialog missing the third strands a keyboard user at the top of the document.

- [ ] **Step 1: Write the failing HUD test**

`engine/shell/hud.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it } from 'vitest';
import { createHud } from './hud.js';

beforeEach(() => {
  document.body.innerHTML = '<h1 id="game-title"></h1><strong id="hud-score" aria-live="off">0</strong>';
});

describe('createHud', () => {
  it('writes the title', () => {
    createHud(document).setTitle('Snake');
    expect(document.querySelector('#game-title')!.textContent).toBe('Snake');
  });

  it('also writes the document title, so the browser tab is not generic', () => {
    createHud(document).setTitle('Pong');
    expect(document.title).toContain('Pong');
  });

  it('writes the score', () => {
    createHud(document).setScore(42);
    expect(document.querySelector('#hud-score')!.textContent).toBe('42');
  });

  it('leaves the score out of the live region, so sixty points are not sixty announcements', () => {
    createHud(document).setScore(1);
    expect(document.querySelector('#hud-score')!.getAttribute('aria-live')).toBe('off');
  });

  it('resets the score to zero', () => {
    const hud = createHud(document);
    hud.setScore(9);
    hud.reset();
    expect(document.querySelector('#hud-score')!.textContent).toBe('0');
  });

  it('writes into the root it was given', () => {
    const frag = document.createElement('div');
    frag.innerHTML = '<strong id="hud-score"></strong>';
    createHud(frag).setScore(5);
    expect(frag.querySelector('#hud-score')!.textContent).toBe('5');
    expect(document.querySelector('#hud-score')!.textContent).toBe('0');
  });

  it('does not throw when the elements are missing', () => {
    const hud = createHud(document.createElement('div'));
    expect(() => { hud.setTitle('x'); hud.setScore(1); hud.reset(); }).not.toThrow();
  });
});
```

- [ ] **Step 2: Write the failing pause test**

`engine/shell/pause.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { createPause, type PauseAction } from './pause.js';

const MARKUP = `
  <button id="before">outside</button>
  <div id="game-region" tabindex="0">
    <div id="pause" hidden>
      <div class="pause-card" role="dialog" aria-modal="true" aria-labelledby="pause-h">
        <h2 id="pause-h">Pausa</h2>
        <div id="pause-menu" role="menu">
          <button type="button" role="menuitem" data-act="resume">Continuar</button>
          <button type="button" role="menuitem" data-act="restart">Reiniciar</button>
          <button type="button" role="menuitem" data-act="quit">Voltar</button>
        </div>
        <div class="settings">
          <label for="set-contrast">C</label>
          <select id="set-contrast"><option value="0">off</option></select>
        </div>
      </div>
    </div>
  </div>`;

const tab = (shift = false): KeyboardEvent =>
  new KeyboardEvent('keydown', { key: 'Tab', shiftKey: shift, bubbles: true, cancelable: true });

let panel: HTMLElement;
let pause: ReturnType<typeof createPause>;

/** No cleanup ritual: each test builds its own instance over its own markup. */
beforeEach(() => {
  document.body.innerHTML = MARKUP;
  panel = document.querySelector('#pause')!;
  pause = createPause(panel);
});

describe('createPause', () => {
  it('starts closed', () => {
    expect(pause.isOpen()).toBe(false);
    expect(panel.hasAttribute('hidden')).toBe(true);
  });

  it('shows the overlay when opened', () => {
    pause.open(() => {});
    expect(pause.isOpen()).toBe(true);
    expect(panel.hasAttribute('hidden')).toBe(false);
  });

  it('moves focus into the dialog on open', () => {
    pause.open(() => {});
    expect(document.activeElement).toBe(document.querySelector('[data-act="resume"]'));
  });

  it('returns focus to whatever had it before, on close', () => {
    const before = document.querySelector<HTMLElement>('#before')!;
    before.focus();
    pause.open(() => {});
    pause.close();
    expect(document.activeElement).toBe(before);
  });

  it('TABS INTO A SETTING, not only the menu buttons', () => {
    // The regression this pins down: a trap that cycles [data-act] alone leaves every accessibility
    // control in the dialog unreachable by keyboard, which defeats the reason they are there.
    pause.open(() => {});
    document.querySelector<HTMLElement>('[data-act="quit"]')!.focus();
    panel.dispatchEvent(tab());
    expect(document.activeElement).toBe(document.querySelector('#set-contrast'));
  });

  it('wraps Tab from the last focusable back to the first', () => {
    pause.open(() => {});
    document.querySelector<HTMLElement>('#set-contrast')!.focus();
    panel.dispatchEvent(tab());
    expect(document.activeElement).toBe(document.querySelector('[data-act="resume"]'));
  });

  it('wraps Shift+Tab from the first focusable back to the last', () => {
    pause.open(() => {});
    document.querySelector<HTMLElement>('[data-act="resume"]')!.focus();
    panel.dispatchEvent(tab(true));
    expect(document.activeElement).toBe(document.querySelector('#set-contrast'));
  });

  it('does not fire an action when a setting is clicked', () => {
    const seen: PauseAction[] = [];
    pause.open((a) => seen.push(a));
    document.querySelector<HTMLElement>('#set-contrast')!.click();
    expect(seen).toEqual([]);
  });

  it('reports the action of the button that was clicked', () => {
    const seen: PauseAction[] = [];
    pause.open((a) => seen.push(a));
    document.querySelector<HTMLElement>('[data-act="restart"]')!.click();
    expect(seen).toEqual(['restart']);
  });

  it('closes itself on resume', () => {
    pause.open(() => {});
    document.querySelector<HTMLElement>('[data-act="resume"]')!.click();
    expect(pause.isOpen()).toBe(false);
  });

  it('resumes on Escape', () => {
    const seen: PauseAction[] = [];
    pause.open((a) => seen.push(a));
    panel.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
    expect(seen).toEqual(['resume']);
    expect(pause.isOpen()).toBe(false);
  });

  it('ignores a second open while already paused', () => {
    const first = vi.fn();
    pause.open(first);
    pause.open(vi.fn());
    document.querySelector<HTMLElement>('[data-act="quit"]')!.click();
    expect(first).toHaveBeenCalledWith('quit');
  });

  it('detaches its listeners on destroy', () => {
    const seen: PauseAction[] = [];
    pause.open((a) => seen.push(a));
    pause.destroy();
    document.querySelector<HTMLElement>('[data-act="quit"]')!.click();
    expect(seen).toEqual([]);
  });
});
```

- [ ] **Step 3: Write the failing audio test**

`engine/platform/audio.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { createAudio } from './audio.js';

const started: number[] = [];

class FakeOsc {
  frequency = { value: 0 };
  type = '';
  connect(): void {}
  start(): void { started.push(this.frequency.value); }
  stop(): void {}
}

beforeEach(() => {
  started.length = 0;
  vi.stubGlobal('AudioContext', class {
    currentTime = 0;
    destination = {};
    createOscillator(): FakeOsc { return new FakeOsc(); }
    createGain() { return { gain: { setValueAtTime() {}, exponentialRampToValueAtTime() {} }, connect() {} }; }
  });
});

describe('createAudio', () => {
  it('plays a tone at the requested frequency', () => {
    createAudio().beep(440, 50);
    expect(started).toEqual([440]);
  });

  it('plays nothing while muted', () => {
    const a = createAudio();
    a.setMuted(true);
    a.beep(440, 50);
    expect(started).toEqual([]);
  });

  it('resumes playing when unmuted', () => {
    const a = createAudio();
    a.setMuted(true);
    a.beep(440, 50);
    a.setMuted(false);
    a.beep(880, 50);
    expect(started).toEqual([880]);
  });

  it('keeps two instances independent', () => {
    const a = createAudio();
    const b = createAudio();
    a.setMuted(true);
    b.beep(220, 10);
    expect(started).toEqual([220]);
  });

  it('does not throw when the browser has no AudioContext', () => {
    vi.stubGlobal('AudioContext', undefined);
    expect(() => createAudio().beep(440, 50)).not.toThrow();
  });
});
```

- [ ] **Step 4: Run all three to verify they fail**

Run: `npx vitest run engine/shell/hud engine/shell/pause engine/platform/audio`
Expected: FAIL — unresolved imports for `hud.js`, `pause.js`, `audio.js`.

- [ ] **Step 5: Write `engine/shell/hud.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// shell/hud — the score strip. Deliberately tiny: anything richer is the game's own business and
// belongs on the canvas.
//
// The score is NOT in a live region. A screen reader would read every increment aloud, and in a game
// that scores sixty times a minute that is not information, it is noise drowning out the announcements
// that matter. Games call srSay() at the moments worth interrupting for.
export interface Hud {
  setTitle(text: string): void;
  setScore(n: number): void;
  reset(): void;
}

export function createHud(root: ParentNode): Hud {
  // Resolved once. Re-querying on every score change is work per frame for an element that never moves.
  const titleEl = root.querySelector('#game-title');
  const scoreEl = root.querySelector('#hud-score');
  return {
    setTitle(text) {
      if (titleEl) titleEl.textContent = text;
      document.title = `${text} · JS Minigames`;
    },
    setScore(n) { if (scoreEl) scoreEl.textContent = String(n); },
    reset() { if (scoreEl) scoreEl.textContent = '0'; },
  };
}
```

- [ ] **Step 6: Write `engine/shell/pause.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// shell/pause — the modal pause menu, written once for every game.
export type PauseAction = 'resume' | 'restart' | 'quit';

export interface Pause {
  open(onAction: (a: PauseAction) => void): void;
  close(): void;
  isOpen(): boolean;
  destroy(): void;
}

export function createPause(panel: HTMLElement): Pause {
  let open = false;
  let restoreFocus: HTMLElement | null = null;
  let handler: ((a: PauseAction) => void) | null = null;

  // TWO lists, deliberately. Every focusable control in the dialog belongs to the Tab cycle, but only
  // [data-act] elements carry an action. Conflating them was a real bug: while the trap cycled
  // [data-act] alone, the accessibility selects added in Task 20 could never be reached by keyboard —
  // the one thing they exist for — because every Tab was preventDefault'd and redirected to a button.
  const FOCUSABLE = 'button, select, input, textarea, a[href], [tabindex]:not([tabindex="-1"])';
  const focusables = (): HTMLElement[] =>
    [...panel.querySelectorAll<HTMLElement>(FOCUSABLE)].filter((el) => !el.hasAttribute('disabled'));

  function close(): void {
    panel.hidden = true;
    if (open) restoreFocus?.focus();
    open = false;
    handler = null;
    restoreFocus = null;
  }

  function act(a: PauseAction): void {
    const fn = handler;
    if (a === 'resume') close();
    fn?.(a);
  }

  function onKeyDown(e: KeyboardEvent): void {
    if (!open) return;
    if (e.key === 'Escape') { e.preventDefault(); act('resume'); return; }
    if (e.key !== 'Tab') return;
    const list = focusables();
    if (list.length === 0) return;
    e.preventDefault();
    const i = list.indexOf(document.activeElement as HTMLElement);
    const next = e.shiftKey ? (i <= 0 ? list.length - 1 : i - 1) : (i === list.length - 1 ? 0 : i + 1);
    list[next]!.focus();
  }

  // Actions come from [data-act] only: a <select> in the dialog is a setting, not a menu command.
  function onClick(e: Event): void {
    if (!open) return;
    const btn = (e.target as HTMLElement).closest<HTMLElement>('[data-act]');
    if (btn) act(btn.dataset['act'] as PauseAction);
  }

  panel.addEventListener('keydown', onKeyDown);
  panel.addEventListener('click', onClick);

  return {
    open(onAction) {
      if (open) return;                     // a second open must not replace the first handler
      handler = onAction;
      restoreFocus = document.activeElement instanceof HTMLElement ? document.activeElement : null;
      panel.hidden = false;
      open = true;
      focusables()[0]?.focus();
    },
    close,
    isOpen: () => open,
    destroy() {
      panel.removeEventListener('keydown', onKeyDown);
      panel.removeEventListener('click', onClick);
      close();
    },
  };
}
```

- [ ] **Step 7: Write `engine/platform/audio.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// platform/audio — one square-wave beep. No asset files, matching the "art is data" rule.
//
// A game gets exactly this. Anything richer means sound files, and 383 games with sound files is a
// download problem and a licensing problem at the same time.
export interface Audio {
  beep(freq: number, ms: number): void;
  setMuted(on: boolean): void;
}

export function createAudio(): Audio {
  let ctx: AudioContext | null = null;
  let muted = false;

  return {
    setMuted(on) { muted = on; },
    beep(freq, ms) {
      if (muted) return;
      try {
        const Ctor = globalThis.AudioContext;
        if (!Ctor) return;                 // no Web Audio (older browser, node test) — stay silent
        ctx ??= new Ctor();
        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        osc.type = 'square';
        osc.frequency.value = freq;
        // Ramp down rather than cutting: an abrupt stop is an audible click on every single sound.
        gain.gain.setValueAtTime(0.06, ctx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + ms / 1000);
        osc.connect(gain);
        gain.connect(ctx.destination);
        osc.start();
        osc.stop(ctx.currentTime + ms / 1000);
      } catch { /* audio is never worth breaking a game over */ }
    },
  };
}
```

- [ ] **Step 8: Run all three, typecheck and format**

Run: `npx vitest run engine/shell/hud engine/shell/pause engine/platform/audio`
Expected: PASS. `hud` 7, `pause` 14, `audio` 5.

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

- [ ] **Step 9: Commit**

```bash
git add engine/shell/hud.ts engine/shell/hud.browser.test.ts engine/shell/pause.ts engine/shell/pause.browser.test.ts engine/platform/audio.ts engine/platform/audio.test.ts
git commit -m "feat: add the HUD, the modal pause menu and a beep, as factories

Focus enters the dialog, cycles inside it and returns where it came from, all
three tested. None of the three keeps module-level state, so no test needs a
cleanup hook to undo the previous one."
```

---

### Task 12: The public game API, and the rule that keeps it public

**Files:**
- Create: `engine/game-api.ts`, `.dependency-cruiser.cjs`
- Modify: `package.json` (add `lint:deps`, extend `validate`)
- Test: `engine/game-api.test.ts`, `games/conformance.test.ts`

**Interfaces:**
- Consumes: types from Tasks 3, 4, 6, 8.
- Produces: `GameContext`, `GameMeta`, `GameInstance`, `GameModule`, re-exported `Scene`, `Handle`, `SpriteSpec`, `SpriteRole`, `InputApi`, `Action`, `GameStrings`, `Translate`, `Box`, and the pure helpers `aabb`, `sweptAabb`. Plus `npm run lint:deps`.

> **This is the module that makes the other 382 games cheap.** One import path, one contract. Two things
> follow from having it, and neither works without the other.
>
> A game must not import `boot.ts` for its types. Depending on the composition root inverts the
> dependency direction: the thing being composed would define nothing and the composer would define
> everything, so any change to wiring would ripple into every game.
>
> And a written rule at this scale is a rule that decays. `dependency-cruiser` runs in `validate` and in
> CI, so the first game that reaches past `game-api.ts` fails the build instead of setting a precedent.

- [ ] **Step 1: Write the failing test**

`engine/game-api.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import * as api from './game-api.js';

describe('game-api', () => {
  it('exports the pure helpers a game may legitimately need', () => {
    expect(typeof api.aabb).toBe('function');
    expect(typeof api.sweptAabb).toBe('function');
  });

  it('exports the eight action names, so a game can iterate them', () => {
    expect([...api.ACTIONS].length).toBe(8);
  });

  it('exposes NO renderer object or class', () => {
    for (const [name, value] of Object.entries(api)) {
      expect(String(name), name).not.toMatch(/pixi/i);
      expect(String(value), name).not.toMatch(/PIXI/);
    }
  });

  it('carries only functions and plain data at runtime — the rest is types', () => {
    for (const [name, value] of Object.entries(api)) {
      expect(['function', 'object', 'string', 'number'], name).toContain(typeof value);
    }
  });
});
```

`games/conformance.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Every game in the repository must satisfy the contract. This passes vacuously today and gains teeth
// with each game added — which is the point: the check exists BEFORE the games it will police.
import { describe, expect, it } from 'vitest';
import type { GameModule } from '../engine/game-api.js';

const modules = import.meta.glob<GameModule>('./*/*/main.ts', { eager: true });
const entries = Object.entries(modules);

describe('game conformance', () => {
  it('every game exports meta, strings and create', () => {
    for (const [path, mod] of entries) {
      expect(mod.meta, path).toBeTruthy();
      expect(mod.strings, path).toBeTruthy();
      expect(typeof mod.create, path).toBe('function');
    }
  });

  it("every game's slug and category match its folder", () => {
    for (const [path, mod] of entries) {
      const [, category, slug] = path.split('/');
      expect(mod.meta.category, path).toBe(category);
      expect(mod.meta.slug, path).toBe(slug);
    }
  });

  it('every game declares all three locales', () => {
    for (const [path, mod] of entries) {
      for (const loc of ['pt', 'en', 'es'] as const) {
        expect(mod.strings[loc], `${path} / ${loc}`).toBeTruthy();
      }
    }
  });

  it('no locale is missing a key that pt has', () => {
    for (const [path, mod] of entries) {
      const keys = Object.keys(mod.strings.pt);
      for (const loc of ['en', 'es'] as const) {
        for (const k of keys) {
          expect(mod.strings[loc][k], `${path}: ${loc} is missing "${k}"`).toBeTruthy();
        }
      }
    }
  });

  it('a non-pixel renderer always carries its justification', () => {
    for (const [path, mod] of entries) {
      if (mod.meta.renderer && mod.meta.renderer !== 'pixel') {
        expect(mod.meta.rendererWhy, path).toBeTruthy();
      }
    }
  });

  it('declares a sane player count', () => {
    for (const [path, mod] of entries) {
      expect([1, 2, 3, 4], path).toContain(mod.meta.players);
    }
  });
});
```

- [ ] **Step 2: Run them to verify they fail**

Run: `npx vitest run --project node engine/game-api.test.ts games/conformance.test.ts`
Expected: FAIL — `Failed to resolve import "./game-api.js"`.

- [ ] **Step 3: Write `engine/game-api.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// game-api — THE ONLY MODULE A GAME MAY IMPORT.
//
// Everything a game is allowed to know lives behind this one path: the context it receives, the shapes
// it declares, and the handful of pure helpers it would otherwise reimplement badly. Enforced by
// `npm run lint:deps`, because a rule this important cannot depend on everyone remembering it.
//
// Note what is NOT here: no PixiJS, no DOM, no storage keys, no shell. A game that needs one of those
// is a game whose need belongs in the context instead — which is a change to this file, reviewed once,
// rather than 383 games each solving it their own way.
export { aabb, sweptAabb, type Box, type SweptHit } from './core/collision.js';
export { ACTIONS, type Action } from './input/actions.js';
export type { InputApi } from './input/attach.js';
export type { GameStrings, LocaleDict, Translate } from './core/i18n.js';
export type { SpriteRole } from './render/high-contrast.js';
export type { Handle, Scene, SpriteSpec } from './render/scene-pixi.js';

import type { InputApi } from './input/attach.js';
import type { Translate, GameStrings } from './core/i18n.js';
import type { Scene } from './render/scene-pixi.js';

/** Everything the shell hands a game. Adding a field here is a spec change: 383 games depend on it. */
export interface GameContext {
  scene: Scene;
  /** The logical field. A game reads its dimensions here instead of importing engine constants. */
  view: { w: number; h: number; tile: number };
  input: InputApi;
  audio: { beep(freq: number, ms: number): void };
  rng: { rnd(): number; randInt(lo: number, hi: number): number; reseed(s: number): void };
  storage: { get(k: string, f?: string | null): string | null; set(k: string, v: string | number | boolean): boolean };
  /** Already namespaced to this game: call t('gameOver'), never t('snake.gameOver'). */
  t: Translate;
  srSay(text: string): void;
  srAlert(text: string): void;
  onGameOver(score: number): void;
}

export interface GameMeta {
  slug: string;
  title: string;
  category: string;
  density: 'leve' | 'medio' | 'denso';
  players: 1 | 2 | 3 | 4;
  renderer?: 'pixel' | 'svg' | '3d';
  /** Required when renderer is not 'pixel'. The conformance test enforces it. */
  rendererWhy?: string;
}

export interface GameInstance {
  /** dt is in FRAMES, not seconds. 1.0 is one 60 fps frame. */
  update(dt: number): void;
  teardown(): void;
}

/** What every games/<cat>/<slug>/main.ts exports. */
export interface GameModule {
  meta: GameMeta;
  strings: GameStrings;
  create(ctx: GameContext): GameInstance;
}
```

- [ ] **Step 4: Write `.dependency-cruiser.cjs`**

```js
// SPDX-License-Identifier: GPL-3.0-or-later
/** @type {import('dependency-cruiser').IConfiguration} */
module.exports = {
  forbidden: [
    {
      name: 'games-only-via-game-api',
      severity: 'error',
      comment:
        'A game may import engine/game-api.ts and files inside its own folder, and nothing else. ' +
        'If a game needs something the API does not offer, add it to the context in game-api.ts — ' +
        'reviewed once — rather than reaching past the boundary here.',
      from: { path: '^games/' },
      to: { path: '^engine/', pathNot: '^engine/game-api\\.ts$' },
    },
    {
      name: 'engine-never-imports-a-game',
      severity: 'error',
      comment:
        'The engine must not depend on any game. Games are discovered at runtime through ' +
        'import.meta.glob in the router; a static import here would bundle every game into the shell.',
      from: { path: '^engine/' },
      to: { path: '^games/' },
    },
    {
      name: 'no-circular',
      severity: 'error',
      comment: 'A cycle means the two modules are one module wearing two names.',
      from: {},
      to: { circular: true },
    },
    {
      name: 'no-orphans',
      severity: 'warn',
      comment: 'A module nothing imports is either dead or miswired.',
      from: { orphan: true, pathNot: '\\.(test|config)\\.(ts|mts|cjs)$|^engine/shell/boot\\.ts$' },
      to: {},
    },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    exclude: { path: '\\.test\\.(ts|mts)$' },
    tsConfig: { fileName: 'tsconfig.json' },
    enhancedResolveOptions: { extensions: ['.ts', '.mts', '.js'] },
  },
};
```

> The `games-only-via-game-api` rule is the one that matters. `no-circular` and `no-orphans` are cheap
> to add while the config is open and would each cost an afternoon to retrofit.
>
> `boot.ts` is exempt from the orphan rule because nothing imports it — `play.html` does, and
> dependency-cruiser does not read HTML.

- [ ] **Step 5: Wire the check into the scripts**

In `package.json`:
```json
"lint:deps": "depcruise engine games scripts --config .dependency-cruiser.cjs",
"validate": "npm run format:check && npm run typecheck && npm run lint:deps && vitest run && npm run build"
```

- [ ] **Step 6: Run the tests and the dependency check**

Run: `npx vitest run --project node engine/game-api.test.ts games/conformance.test.ts`
Expected: PASS. `game-api` 4; `conformance` 6, all vacuous — there are no games yet.

Run: `npm run lint:deps`
Expected: `no dependency violations found`.

- [ ] **Step 7: Prove the rule actually bites**

This step exists because an unverified guard is not a guard. Temporarily create
`games/tmp/probe/main.ts`:
```ts
import { LOGICAL_W } from '../../../engine/core/constants.js';
export const probe = LOGICAL_W;
```

Run: `npm run lint:deps`
Expected: FAIL, naming `games-only-via-game-api`.

Then delete `games/tmp/` and run it again — expected: clean. Do not commit the probe.

- [ ] **Step 8: Typecheck, format and commit**

Run: `npx tsc --noEmit && npm run format:check`
Expected: no output from either.

```bash
git add engine/game-api.ts engine/game-api.test.ts games/conformance.test.ts .dependency-cruiser.cjs package.json
git commit -m "feat: add the public game API and enforce it

One import path for all 383 games, and a dependency-cruiser rule that fails the
build when a game reaches past it. A written boundary at this scale decays; a
checked one does not.

Games no longer take their types from the composition root, which had the
dependency arrow pointing the wrong way."
```

---

### Task 13: The session and the composition root

**Files:**
- Create: `engine/shell/session.ts`, `engine/shell/boot.ts`
- Test: `engine/shell/session.browser.test.ts`, `engine/shell/boot.browser.test.ts`

**Interfaces:**
- Consumes: everything from Tasks 1–12.
- Produces:
  - `startSession(deps: SessionDeps): Session` with `restart()`, `destroy()`, `hasCrashed()`.
  - `startShell(): Promise<void>`, plus the debug handle `window.__demos`.

> Split in two on purpose. `boot` **composes**: it creates every instance, in one place, and hands them
> down. `session` **runs**: one game's lifetime, from mounting the canvas to tearing it down. The first
> draft did both in one file and was already accumulating pause wiring, language redraw, high-score
> persistence and renderer validation — the shape a god-module has before anyone calls it that.
>
> The session owns the **error boundary**. Across 383 games written over months, some game will throw in
> `update`. Without a boundary it throws again every frame: the player sees a frozen canvas, the console
> fills, and a broken game is indistinguishable from a broken engine. The boundary stops the loop once,
> says so out loud through the assertive live region, and leaves the pause menu reachable so the player
> can get back to the catalog.

- [ ] **Step 1: Write the failing session test**

`engine/shell/session.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { startSession } from './session.js';
import { createI18n } from '../core/i18n.js';
import { createVisualState } from '../render/high-contrast.js';
import { createHud } from './hud.js';
import { createAnnouncer } from '../core/a11y-sr.js';
import { createAudio } from '../platform/audio.js';
import { KB_DEFAULTS } from '../input/keyboard.js';
import type { GameContext, GameModule } from '../game-api.js';

function stubStorage(): void {
  const m = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => m.get(k) ?? null,
    setItem: (k: string, v: string) => { m.set(k, v); },
    removeItem: (k: string) => { m.delete(k); },
  });
}

/** A module the test controls completely: no game folder, no glob, no surprises. */
function fakeModule(over: Partial<GameModule> = {}, body?: (ctx: GameContext) => void): GameModule {
  return {
    meta: { slug: 'probe', title: 'Probe', category: 'test', density: 'leve', players: 1 },
    strings: { pt: { hi: 'olá' }, en: { hi: 'hi' }, es: { hi: 'hola' } },
    create: (ctx) => {
      body?.(ctx);
      return { update() {}, teardown() {} };
    },
    ...over,
  };
}

function deps(mod: GameModule, onCrash = vi.fn()) {
  const region = document.querySelector<HTMLElement>('#game-region')!;
  const i18n = createI18n();
  i18n.register(mod.meta.slug, mod.strings);
  return {
    region, mod, i18n,
    visual: createVisualState(),
    hud: createHud(document),
    announcer: createAnnouncer(document),
    audio: createAudio(),
    kb: KB_DEFAULTS,
    isPaused: () => false,
    onCrash,
  };
}

beforeEach(() => {
  stubStorage();
  document.body.innerHTML =
    '<h1 id="game-title"></h1><strong id="hud-score" aria-live="off">0</strong>' +
    '<div id="sr-status"></div><div id="sr-alert"></div>' +
    '<div id="game-region" tabindex="0"></div>';
});

describe('startSession', () => {
  it('puts a 320x180 canvas in the region', () => {
    const s = startSession(deps(fakeModule()));
    const cv = document.querySelector<HTMLCanvasElement>('#game-region canvas')!;
    expect([cv.width, cv.height]).toEqual([320, 180]);
    s.destroy();
  });

  it('shows the title from meta', () => {
    const s = startSession(deps(fakeModule()));
    expect(document.querySelector('#game-title')!.textContent).toBe('Probe');
    s.destroy();
  });

  it('hands the game a context with no renderer object in it', () => {
    let seen: GameContext | null = null;
    const s = startSession(deps(fakeModule({}, (ctx) => { seen = ctx; })));
    expect(seen).not.toBeNull();
    expect(Object.keys(seen!)).not.toContain('stage');
    expect((seen as unknown as Record<string, unknown>)['app']).toBeUndefined();
    s.destroy();
  });

  it('hands the game the logical field size, so it imports no constants', () => {
    let seen: GameContext | null = null;
    const s = startSession(deps(fakeModule({}, (ctx) => { seen = ctx; })));
    expect(seen!.view).toEqual({ w: 320, h: 180, tile: 16 });
    s.destroy();
  });

  it('namespaces the translate function to this game', () => {
    let seen: GameContext | null = null;
    const s = startSession(deps(fakeModule({}, (ctx) => { seen = ctx; })));
    expect(seen!.t('hi')).toBe('olá');
    s.destroy();
  });

  it('rejects a non-pixel renderer with no justification', () => {
    const bad = fakeModule({ meta: { slug: 'p', title: 'P', category: 'test', density: 'leve', players: 1, renderer: 'svg' } });
    expect(() => startSession(deps(bad))).toThrow(/rendererWhy/);
  });

  it('records a high score and announces the end', async () => {
    let seen: GameContext | null = null;
    const s = startSession(deps(fakeModule({}, (ctx) => { seen = ctx; })));
    seen!.onGameOver(30);
    expect(localStorage.getItem('demos.hi.probe')).toBe('30');
    expect(document.querySelector('#hud-score')!.textContent).toBe('30');
    s.destroy();
  });

  it('keeps the better of two scores', () => {
    let seen: GameContext | null = null;
    const s = startSession(deps(fakeModule({}, (ctx) => { seen = ctx; })));
    seen!.onGameOver(30);
    seen!.onGameOver(10);
    expect(localStorage.getItem('demos.hi.probe')).toBe('30');
    s.destroy();
  });

  it('STOPS THE LOOP when a game throws, instead of throwing every frame', async () => {
    const onCrash = vi.fn();
    let ticks = 0;
    const crashing = fakeModule({
      create: () => ({ update() { ticks++; throw new Error('boom'); }, teardown() {} }),
    });
    const s = startSession(deps(crashing, onCrash));
    await new Promise((r) => setTimeout(r, 80));   // several frames
    expect(ticks).toBe(1);
    expect(s.hasCrashed()).toBe(true);
    expect(onCrash).toHaveBeenCalledOnce();
    s.destroy();
  });

  it('says out loud that the game failed, rather than freezing silently', async () => {
    const crashing = fakeModule({ create: () => ({ update() { throw new Error('boom'); }, teardown() {} }) });
    const s = startSession(deps(crashing));
    await new Promise((r) => setTimeout(r, 80));
    await new Promise((r) => requestAnimationFrame(() => r(null)));
    expect(document.querySelector('#sr-alert')!.textContent).not.toBe('');
    s.destroy();
  });

  it('restart builds a fresh instance and resets the score', () => {
    let creates = 0;
    const counting = fakeModule({ create: () => { creates++; return { update() {}, teardown() {} }; } });
    const s = startSession(deps(counting));
    expect(creates).toBe(1);
    s.restart();
    expect(creates).toBe(2);
    expect(document.querySelector('#hud-score')!.textContent).toBe('0');
    s.destroy();
  });

  it('tears the game down and removes the canvas on destroy', () => {
    const teardown = vi.fn();
    const s = startSession(deps(fakeModule({ create: () => ({ update() {}, teardown }) })));
    s.destroy();
    expect(teardown).toHaveBeenCalledOnce();
    expect(document.querySelector('#game-region canvas')).toBeNull();
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project browser engine/shell/session.browser.test.ts`
Expected: FAIL — `Failed to resolve import "./session.js"`.

- [ ] **Step 3: Write `engine/shell/session.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// shell/session — one game's lifetime: mount, build the context, run the loop, tear down.
//
// It receives every collaborator as an argument and constructs none of them, which is what lets the
// test above drive it with a fake game module and no game folder at all.
import { LOGICAL_H, LOGICAL_W, TILE } from '../core/constants.js';
import { startLoop } from '../core/loop.js';
import { createRng } from '../core/rng.js';
import * as store from '../platform/storage.js';
import { attachInput } from '../input/attach.js';
import type { KBDefaults } from '../input/keyboard.js';
import { mountPixi } from '../render/mount.js';
import { createPixiScene } from '../render/scene-pixi.js';
import type { VisualState } from '../render/high-contrast.js';
import type { I18n } from '../core/i18n.js';
import type { Announcer } from '../core/a11y-sr.js';
import type { Audio } from '../platform/audio.js';
import type { Hud } from './hud.js';
import type { GameContext, GameInstance, GameMeta, GameModule } from '../game-api.js';

export interface SessionDeps {
  region: HTMLElement;
  mod: GameModule;
  i18n: I18n;
  visual: VisualState;
  hud: Hud;
  announcer: Announcer;
  audio: Audio;
  kb: KBDefaults;
  isPaused(): boolean;
  onCrash(err: unknown): void;
}

export interface Session {
  restart(): void;
  destroy(): void;
  hasCrashed(): boolean;
}

/**
 * A game declaring a non-pixel renderer must say why (spec D6). The conformance test catches this at
 * build time; this catches a module that was loaded some other way. Throwing beats warning: a silent
 * deviation is exactly what the decision was written to stop.
 */
function assertRenderer(meta: GameMeta): void {
  if (meta.renderer && meta.renderer !== 'pixel' && !meta.rendererWhy) {
    throw new Error(`Game "${meta.slug}" declares renderer "${meta.renderer}" without rendererWhy (spec D6).`);
  }
}

export function startSession(deps: SessionDeps): Session {
  const { region, mod, i18n, visual, hud, announcer, audio, kb } = deps;
  assertRenderer(mod.meta);

  const mount = mountPixi(region);
  const scene = createPixiScene(mount.stage, visual);
  const input = attachInput(region, mod.meta.players, kb);
  const rng = createRng();
  const hiKey = store.KEYS.highScore(mod.meta.slug);

  const ctx: GameContext = {
    scene,
    view: { w: LOGICAL_W, h: LOGICAL_H, tile: TILE },
    input,
    audio: { beep: (f, ms) => audio.beep(f, ms) },
    rng: { rnd: rng.rnd, randInt: rng.randInt, reseed: rng.reseed },
    storage: { get: (k, f = null) => store.get(k, f), set: (k, v) => store.set(k, v) },
    t: i18n.scoped(mod.meta.slug),
    srSay: (text) => announcer.say(text),
    srAlert: (text) => announcer.alert(text),
    onGameOver(score) {
      hud.setScore(score);
      if (score > Number(store.get(hiKey, '0'))) store.set(hiKey, score);
      announcer.alert(i18n.t('shell.gameOver', { score }));
    },
  };

  hud.setTitle(mod.meta.title);
  hud.reset();

  let instance: GameInstance = mod.create(ctx);
  let crashed = false;

  /**
   * The error boundary. One throw stops the loop for good: a game that failed once will fail again on
   * the next frame with the same state, so retrying only fills the console while the player stares at
   * a frozen picture. The failure is announced assertively and logged with the game's slug, so a bug
   * report can name which of the 383 broke.
   */
  startLoop(mount.app.ticker, (dt) => {
    if (crashed || deps.isPaused()) return;
    try {
      input.poll();
      instance.update(dt);
    } catch (err) {
      crashed = true;
      console.error(`[${mod.meta.category}/${mod.meta.slug}] crashed:`, err);
      announcer.alert(i18n.t('shell.crashed'));
      deps.onCrash(err);
    }
  });

  return {
    hasCrashed: () => crashed,

    restart() {
      instance.teardown();
      scene.clear();
      hud.reset();
      crashed = false;
      instance = mod.create(ctx);
    },

    destroy() {
      instance.teardown();
      scene.destroy();
      input.detach();
      mount.destroy();
    },
  };
}
```

- [ ] **Step 4: Run the session test to verify it passes**

Run: `npx vitest run --project browser engine/shell/session.browser.test.ts`
Expected: PASS, 12 tests — including the two that prove a throwing game stops once and says so.

- [ ] **Step 5: Write the failing boot test**

`engine/shell/boot.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { afterEach, describe, expect, it } from 'vitest';
import { startShell } from './boot.js';

async function mountShell(hash: string): Promise<void> {
  const html = await (await fetch('/play.html')).text();
  const parsed = new DOMParser().parseFromString(html, 'text/html');
  document.body.innerHTML = parsed.body.innerHTML;
  location.hash = hash;
  await startShell();
}

afterEach(() => { document.body.innerHTML = ''; location.hash = ''; });

describe('startShell', () => {
  it('reports a missing game instead of failing silently', async () => {
    await mountShell('#no-such-category/no-such-game');
    expect(document.querySelector('#sr-alert')!.textContent).not.toBe('');
    expect(document.querySelector('#game-title')!.textContent).toContain('no-such-game');
  });

  it('survives a malformed hash without throwing', async () => {
    await expect(mountShell('#../../etc/passwd')).resolves.toBeUndefined();
  });

  it('boots a real game and puts a canvas in the game region', async () => {
    await mountShell('#arcade-classico/snake');
    const cv = document.querySelector<HTMLCanvasElement>('#game-region canvas');
    expect(cv).not.toBeNull();
    expect([cv!.width, cv!.height]).toEqual([320, 180]);
  });

  it('installs the six colour-vision filters', async () => {
    await mountShell('#arcade-classico/snake');
    expect(document.querySelectorAll('#cvd-filters filter').length).toBe(6);
  });

  it('focuses the game region, so the keyboard reaches the game without a click', async () => {
    await mountShell('#arcade-classico/snake');
    expect(document.activeElement).toBe(document.querySelector('#game-region'));
  });

  it('exposes a debug handle for the preview harness', async () => {
    await mountShell('#arcade-classico/snake');
    const h = (window as unknown as { __demos?: { game: { slug: string } } }).__demos;
    expect(h?.game.slug).toBe('snake');
  });

  it('registered the game strings, so its own keys resolve', async () => {
    await mountShell('#arcade-classico/snake');
    const h = (window as unknown as { __demos: { t(k: string): string } }).__demos;
    expect(h.t('gameOver')).not.toBe('gameOver');
  });
});
```

- [ ] **Step 6: Run it — five pass, two wait for Snake**

Run: `npx vitest run --project browser engine/shell/boot.browser.test.ts`
Expected after Step 7: the two "real game" cases still fail with "Game not found" until Task 14 lands.
Everything else passes.

- [ ] **Step 7: Write `engine/shell/boot.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// shell/boot — the composition root. The ONLY module that knows how the engine fits together, and the
// only one that creates instances of anything.
//
// Every collaborator below is built here and passed down. Nothing reaches for a module-level singleton,
// which is why every one of them can be tested on its own and why two of anything can coexist.
import { createAnnouncer } from '../core/a11y-sr.js';
import { createI18n } from '../core/i18n.js';
import { loadKB } from '../input/keyboard.js';
import { installCvdFilters } from '../render/cvd-matrices.js';
import { createVisualState, type ContrastLevel } from '../render/high-contrast.js';
import { createViz } from '../render/viz.js';
import { createAudio } from '../platform/audio.js';
import * as store from '../platform/storage.js';
import { $ } from '../ui/dom.js';
import { createHud } from './hud.js';
import { createPause } from './pause.js';
import { loaderFor, parseHash } from './router.js';
import { startSession, type Session } from './session.js';
import type { GameModule } from '../game-api.js';
// NOTE: settings.ts arrives in Task 20, which adds its import and its two call sites here. Importing it
// now would not compile.

export async function startShell(): Promise<void> {
  const region = $<HTMLElement>('#game-region');
  if (!region) return;

  const i18n = createI18n();
  await i18n.init();
  i18n.applyDom(document);

  const announcer = createAnnouncer(document);
  const hud = createHud(document);
  const audio = createAudio();
  const visual = createVisualState(Number(store.get(store.KEYS.contrast, '0')) as ContrastLevel);

  installCvdFilters($('#cvd-filters'));
  const viz = createViz(region);
  viz.restore();

  const route = parseHash(location.hash);
  const load = route ? loaderFor(route.category, route.slug) : null;
  if (!route || !load) {
    const label = i18n.t('shell.notFound', { slug: route ? `${route.category}/${route.slug}` : '—' });
    hud.setTitle(label);
    announcer.alert(label);
    return;
  }

  const mod = (await load()) as GameModule;
  i18n.register(mod.meta.slug, mod.strings);

  const pausePanel = $<HTMLElement>('#pause');
  const pause = pausePanel ? createPause(pausePanel) : null;

  let session: Session | null = null;
  session = startSession({
    region, mod, i18n, visual, hud, announcer, audio,
    kb: loadKB(),
    isPaused: () => pause?.isOpen() ?? false,
    // A crashed game must still be escapable, so the pause menu is opened FOR the player rather than
    // leaving them on a dead canvas with no visible way out.
    onCrash: () => pause?.open((a) => { if (a === 'quit') location.href = './index.html'; }),
  });

  // Pause belongs to the shell, not to any game: one implementation, one focus contract, 383 games.
  region.addEventListener('keydown', (e) => {
    if (e.key !== 'Escape' || pause === null || pause.isOpen()) return;
    e.preventDefault();
    pause.open((action) => {
      if (action === 'restart') session?.restart();
      if (action === 'quit') { session?.destroy(); location.href = './index.html'; }
    });
  });

  // Task 20 mounts the accessibility settings here.

  // Canvas text is invisible to applyDom, which only walks [data-i18n] in the DOM. Restarting the game
  // is the blunt way to redraw it, and it means the discipline lives here instead of in 383 games.
  i18n.onChange(() => {
    i18n.applyDom(document);
    session?.restart();   // Task 20 also re-mounts the settings here
  });

  // Reloading on hash change is cruder than swapping games in place, and correct: a torn-down game
  // cannot leave a stray ticker, listener or texture behind to haunt the next one.
  window.addEventListener('hashchange', () => location.reload());

  region.focus();
  announcer.say(mod.meta.title);

  (window as unknown as Record<string, unknown>)['__demos'] = {
    game: mod.meta,
    t: i18n.scoped(mod.meta.slug),
    i18n,
    visual,
    viz,
    pause,
    session,
  };
}

// Auto-boot only when the shell markup is already present. play.html loads this module at the end of
// the body, so it is; a test importing startShell has an empty document, so it is not. Guarding on the
// markup rather than on an environment flag leaves no test-only branch to drift.
if (typeof document !== 'undefined' && document.querySelector('#game-region')) void startShell();
```

- [ ] **Step 8: Add the module tag to `play.html`**

Task 10 left a comment where this goes, just before `</body>`:
```html
<script type="module" src="./engine/shell/boot.ts"></script>
```

- [ ] **Step 9: Typecheck, run everything and commit**

Run: `npx tsc --noEmit && npm run lint:deps && npm run format:check`
Expected: no output from any of the three.

```bash
git add engine/shell/session.ts engine/shell/session.browser.test.ts engine/shell/boot.ts engine/shell/boot.browser.test.ts play.html
git commit -m "feat: split the composition root from the game session

boot composes and creates every instance; session runs one game's lifetime.
The first draft did both and was already collecting pause wiring, language
redraw, score persistence and renderer validation.

The session owns an error boundary. A throwing game stops the loop once,
announces the failure and opens the pause menu, instead of throwing every
frame behind a frozen canvas that looks exactly like a broken engine."
```

---

### Task 14: Snake — the first reference game

**Files:**
- Create: `games/arcade-classico/snake/rules.ts`, `strings.ts`, `main.ts`
- Test: `games/arcade-classico/snake/rules.test.ts`

**Interfaces:**
- Consumes: `GameContext`, `GameMeta`, `GameInstance`, `GameStrings` from `engine/game-api.ts` — and nothing else from the engine.
- Produces: the shape every later game copies.

> **This task defines the pattern for the other 382 games.** Three properties, and the plan is wrong if
> any of them slips:
>
> Everything decidable without a screen lives in `rules.ts`, tested at speed in the node project.
> `main.ts` only draws and reads input — 383 browser-only suites is not a suite anyone runs.
>
> `create(ctx)` returns the instance and **closes over its own state**. No `let` at module scope, so
> restart cannot inherit anything from the run before it.
>
> The only engine import is `game-api`. `npm run lint:deps` fails the build otherwise.

- [ ] **Step 1: Write the failing rules test**

`games/arcade-classico/snake/rules.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { createSnake, step, turn, type SnakeState } from './rules.js';

/** Deterministic stand-in for ctx.rng, cycling through scripted values. */
function scripted(values: number[]): () => number {
  let i = 0;
  return () => values[i++ % values.length]!;
}

/** A snake laid out horizontally, head at (hx, hy), moving right, apple parked far away. */
function fixture(hx: number, hy: number, len = 3): SnakeState {
  const s = createSnake(10, 10, scripted([9, 9]));
  s.body = Array.from({ length: len }, (_, i) => ({ x: hx - i, y: hy }));
  s.dir = { x: 1, y: 0 };
  s.next = { x: 1, y: 0 };
  s.apple = { x: 9, y: 9 };
  return s;
}

describe('createSnake', () => {
  it('starts alive, scoreless and moving', () => {
    const s = createSnake(10, 10, scripted([5, 5]));
    expect(s.alive).toBe(true);
    expect(s.score).toBe(0);
    expect(s.body.length).toBeGreaterThan(0);
    expect(s.dir).not.toEqual({ x: 0, y: 0 });
  });

  it('never places the first apple on the snake', () => {
    const s = createSnake(10, 10, scripted([0, 0, 5, 5]));
    expect(s.body.some((c) => c.x === s.apple.x && c.y === s.apple.y)).toBe(false);
  });
});

describe('turn', () => {
  it('accepts a perpendicular turn', () => {
    const s = fixture(5, 5);
    turn(s, { x: 0, y: -1 });
    step(s, scripted([9, 9]));
    expect(s.body[0]).toEqual({ x: 5, y: 4 });
  });

  it('refuses a reversal, which would drive the head into the neck', () => {
    const s = fixture(5, 5);
    turn(s, { x: -1, y: 0 });
    step(s, scripted([9, 9]));
    expect(s.body[0]).toEqual({ x: 6, y: 5 });
    expect(s.alive).toBe(true);
  });

  it('refuses a reversal even through two turns in the same frame', () => {
    const s = fixture(5, 5);
    turn(s, { x: 0, y: -1 });
    turn(s, { x: 0, y: 1 });
    step(s, scripted([9, 9]));
    expect(s.body[0]).toEqual({ x: 5, y: 4 });
  });

  it('allows a reversal for a snake of length one, which has no neck to hit', () => {
    const s = fixture(5, 5, 1);
    turn(s, { x: -1, y: 0 });
    step(s, scripted([9, 9]));
    expect(s.body[0]).toEqual({ x: 4, y: 5 });
  });
});

describe('step', () => {
  it('moves the head and drops the tail, keeping the length', () => {
    const s = fixture(5, 5);
    step(s, scripted([9, 9]));
    expect(s.body[0]).toEqual({ x: 6, y: 5 });
    expect(s.body.length).toBe(3);
  });

  it('dies on the right wall', () => {
    const s = fixture(9, 5);
    step(s, scripted([0, 0]));
    expect(s.alive).toBe(false);
  });

  it('dies on the left wall', () => {
    const s = fixture(0, 5);
    s.dir = { x: -1, y: 0 }; s.next = { x: -1, y: 0 };
    step(s, scripted([0, 0]));
    expect(s.alive).toBe(false);
  });

  it('dies on the top and bottom walls', () => {
    const top = fixture(5, 0);
    top.dir = { x: 0, y: -1 }; top.next = { x: 0, y: -1 };
    step(top, scripted([0, 0]));
    expect(top.alive).toBe(false);

    const bottom = fixture(5, 9);
    bottom.dir = { x: 0, y: 1 }; bottom.next = { x: 0, y: 1 };
    step(bottom, scripted([0, 0]));
    expect(bottom.alive).toBe(false);
  });

  it('dies on its own body', () => {
    const s = createSnake(10, 10, scripted([9, 9]));
    s.body = [{ x: 5, y: 5 }, { x: 5, y: 4 }, { x: 4, y: 4 }, { x: 4, y: 5 }];
    s.dir = { x: -1, y: 0 }; s.next = { x: -1, y: 0 };
    s.apple = { x: 9, y: 9 };
    step(s, scripted([9, 9]));
    expect(s.alive).toBe(false);
  });

  it('survives moving into the cell its own tail is vacating', () => {
    const s = createSnake(10, 10, scripted([9, 9]));
    s.body = [{ x: 5, y: 5 }, { x: 5, y: 4 }, { x: 4, y: 4 }, { x: 4, y: 5 }];
    s.dir = { x: 0, y: 1 }; s.next = { x: 0, y: 1 };
    s.apple = { x: 9, y: 9 };
    step(s, scripted([9, 9]));
    expect(s.alive).toBe(true);
  });

  it('grows and scores on the apple', () => {
    const s = fixture(5, 5);
    s.apple = { x: 6, y: 5 };
    step(s, scripted([1, 1]));
    expect(s.body.length).toBe(4);
    expect(s.score).toBe(1);
  });

  it('moves the apple somewhere free after it is eaten', () => {
    const s = fixture(5, 5);
    s.apple = { x: 6, y: 5 };
    step(s, scripted([1, 1]));
    expect(s.apple).not.toEqual({ x: 6, y: 5 });
    expect(s.body.some((c) => c.x === s.apple.x && c.y === s.apple.y)).toBe(false);
  });

  it('does nothing once dead, so a late frame cannot resurrect the run', () => {
    const s = fixture(9, 5);
    step(s, scripted([0, 0]));
    const frozen = JSON.stringify(s);
    step(s, scripted([0, 0]));
    expect(JSON.stringify(s)).toBe(frozen);
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node games/arcade-classico/snake`
Expected: FAIL — `Failed to resolve import "./rules.js"`.

- [ ] **Step 3: Write `games/arcade-classico/snake/rules.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Snake rules. Pure: no renderer, no DOM, no engine. Everything decidable without a screen lives here,
// so the node project tests it at speed.
export interface Cell { x: number; y: number }

export interface SnakeState {
  cols: number;
  rows: number;
  /** Head first. */
  body: Cell[];
  dir: Cell;
  /** The direction accepted this frame but not yet applied — see `turn`. */
  next: Cell;
  apple: Cell;
  score: number;
  alive: boolean;
}

/** Pick a free cell. `rand(n)` must return an integer in [0, n). */
function placeApple(s: SnakeState, rand: (n: number) => number): Cell {
  // Rejection sampling: the board is never near full in a reference game, and enumerating free cells
  // would allocate every time the apple moves.
  for (let i = 0; i < 500; i++) {
    const c = { x: rand(s.cols), y: rand(s.rows) };
    if (!s.body.some((b) => b.x === c.x && b.y === c.y)) return c;
  }
  return { x: 0, y: 0 };
}

export function createSnake(cols: number, rows: number, rand: (n: number) => number): SnakeState {
  const midY = Math.floor(rows / 2);
  const s: SnakeState = {
    cols, rows,
    body: [{ x: 3, y: midY }, { x: 2, y: midY }, { x: 1, y: midY }],
    dir: { x: 1, y: 0 },
    next: { x: 1, y: 0 },
    apple: { x: 0, y: 0 },
    score: 0,
    alive: true,
  };
  s.apple = placeApple(s, rand);
  return s;
}

/**
 * Queue a direction for the next step.
 *
 * Reversal is refused: turning back drives the head into the neck and reads as an unfair instant death.
 * It is checked against the QUEUED direction, not the current one, because two turns inside one frame
 * would otherwise sneak a reversal past the guard. A snake of length one has no neck and may turn.
 */
export function turn(s: SnakeState, dir: Cell): void {
  if (dir.x === 0 && dir.y === 0) return;
  if (s.body.length > 1 && dir.x === -s.next.x && dir.y === -s.next.y) return;
  s.next = dir;
}

/** Advance exactly one cell. A dead snake is inert: a late frame must not restart the run. */
export function step(s: SnakeState, rand: (n: number) => number): void {
  if (!s.alive) return;
  s.dir = s.next;

  const head = { x: s.body[0]!.x + s.dir.x, y: s.body[0]!.y + s.dir.y };
  if (head.x < 0 || head.y < 0 || head.x >= s.cols || head.y >= s.rows) { s.alive = false; return; }

  const ate = head.x === s.apple.x && head.y === s.apple.y;

  // The tail cell is only an obstacle while growing; otherwise it moves out of the way in this very
  // step, and treating it as solid is the false death every Snake gets wrong once.
  const solid = ate ? s.body : s.body.slice(0, -1);
  if (solid.some((c) => c.x === head.x && c.y === head.y)) { s.alive = false; return; }

  s.body.unshift(head);
  if (ate) { s.score++; s.apple = placeApple(s, rand); } else { s.body.pop(); }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node games/arcade-classico/snake`
Expected: PASS, 15 tests.

- [ ] **Step 5: Write `games/arcade-classico/snake/strings.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Snake's own strings, registered under its slug when this chunk loads. No other game downloads them
// and no other game's keys collide with them.
import type { GameStrings } from '../../../engine/game-api.js';

export const strings: GameStrings = {
  pt: {
    start: 'Use as setas ou WASD para mover.',
    ate: '{score} maçãs',
    gameOver: 'Você bateu. {score} maçãs.',
  },
  en: {
    start: 'Use the arrows or WASD to move.',
    ate: '{score} apples',
    gameOver: 'You crashed. {score} apples.',
  },
  es: {
    start: 'Usa las flechas o WASD para moverte.',
    ate: '{score} manzanas',
    gameOver: 'Chocaste. {score} manzanas.',
  },
};
```

- [ ] **Step 6: Write `games/arcade-classico/snake/main.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Snake — the reference game. Thin by design: rules live in rules.ts, and everything accessible
// (contrast, colour-vision filters, live regions, pause, scaling, language) comes from the shell.
//
// The accessibility cost of this file is three things: a role tag on each of the two sprite kinds, a
// dictionary in three languages, and three announcements. That is the budget every other game in the
// collection is meant to fit into.
import type { GameContext, GameInstance, GameMeta, Handle } from '../../../engine/game-api.js';
import { createSnake, step, turn } from './rules.js';
import { strings } from './strings.js';

export { strings };

export const meta: GameMeta = {
  slug: 'snake',
  title: 'Snake',
  category: 'arcade-classico',
  density: 'leve',
  players: 1,
};

const CELL = 8;
/** Frames between cell advances. dt is in frames, so this is a count, never milliseconds. */
const FRAMES_PER_STEP = 6;

export function create(ctx: GameContext): GameInstance {
  const cols = Math.floor(ctx.view.w / CELL);
  const rows = Math.floor(ctx.view.h / CELL);
  const offsetY = Math.floor((ctx.view.h - rows * CELL) / 2);

  // All state is local to this closure. Restarting builds a new one, so nothing survives a run.
  const state = createSnake(cols, rows, (n) => ctx.rng.randInt(0, n - 1));
  const segments: Handle[] = [];
  let accum = 0;
  let over = false;

  const block = (role: 'player' | 'pickup'): Handle =>
    ctx.scene.add({
      role,
      w: CELL, h: CELL,
      // These colours are what the game looks like with contrast OFF. With it on, the engine replaces
      // them by role and keeps this silhouette.
      paint: (px) => px(1, 1, CELL - 2, CELL - 2, role === 'player' ? '#b8ff3d' : '#ff2d8e'),
    });

  const apple = block('pickup');

  const place = (h: Handle, cx: number, cy: number): void => {
    h.x = cx * CELL;
    h.y = cy * CELL + offsetY;
  };

  function sync(): void {
    while (segments.length < state.body.length) segments.push(block('player'));
    while (segments.length > state.body.length) ctx.scene.remove(segments.pop()!);
    state.body.forEach((c, i) => place(segments[i]!, c.x, c.y));
    place(apple, state.apple.x, state.apple.y);
  }

  sync();
  ctx.srSay(ctx.t('start'));

  return {
    update(dt) {
      if (over) return;

      if (ctx.input.pressed(0, 'up')) turn(state, { x: 0, y: -1 });
      if (ctx.input.pressed(0, 'down')) turn(state, { x: 0, y: 1 });
      if (ctx.input.pressed(0, 'left')) turn(state, { x: -1, y: 0 });
      if (ctx.input.pressed(0, 'right')) turn(state, { x: 1, y: 0 });

      accum += dt;
      if (accum < FRAMES_PER_STEP) return;
      accum -= FRAMES_PER_STEP;

      const before = state.score;
      step(state, (n) => ctx.rng.randInt(0, n - 1));
      sync();

      if (state.score !== before) {
        ctx.audio.beep(880, 40);
        ctx.srSay(ctx.t('ate', { score: state.score }));
      }

      if (!state.alive) {
        over = true;
        ctx.audio.beep(110, 240);
        ctx.srAlert(ctx.t('gameOver', { score: state.score }));
        ctx.onGameOver(state.score);
      }
    },

    teardown() {
      for (const s of segments) ctx.scene.remove(s);
      segments.length = 0;
      ctx.scene.remove(apple);
    },
  };
}
```

- [ ] **Step 7: Run the conformance test, the boot tests and the dependency check**

Run: `npx vitest run`
Expected: PASS everywhere. `conformance` now has a game to police; the two boot cases that waited for
Snake go green.

Run: `npm run lint:deps`
Expected: `no dependency violations found` — Snake imports only `game-api`.

- [ ] **Step 8: See it actually run**

Run: `npm run build && npm run preview`, open `http://localhost:4173/play.html#arcade-classico/snake`.

Confirm by looking, not by assuming:
- the snake moves and turns with both the arrows and WASD;
- eating an apple lengthens it and the header score rises;
- hitting a wall stops the game;
- `Escape` opens the pause dialog and `Tab` cycles inside it without escaping;
- in the console, `__demos.visual.setLevel(7)` repaints the snake and apple into the high-contrast
  palette, each block outlined in white.

- [ ] **Step 9: Commit**

```bash
git add games/arcade-classico/snake/
git commit -m "feat: add Snake, the first reference game

Establishes the shape the other 382 copy: pure rules in the node project, a
three-locale dictionary, and a create(ctx) factory whose state is closed over
rather than parked at module scope. Its only engine import is game-api."
```

---

### Task 15: Pong — two-player input

**Files:**
- Create: `games/arcade-classico/pong/rules.ts`, `strings.ts`, `main.ts`
- Test: `games/arcade-classico/pong/rules.test.ts`

**Interfaces:**
- Consumes: `engine/game-api.ts` only.
- Produces: nothing other tasks consume. Its job is to prove `players: 2` works end to end.

> Pong is here for the input layer, not the game. Snake never calls `held` and never uses a second
> scheme. If `attachInput` had the two schemes crossed, Snake would pass and every two-player game in
> the collection would be broken.

- [ ] **Step 1: Write the failing rules test**

`games/arcade-classico/pong/rules.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { advance, createPong, movePaddle, WIN_SCORE, type PongState } from './rules.js';

const fresh = (): PongState => createPong(320, 176);

describe('createPong', () => {
  it('starts level, scoreless and with the ball moving', () => {
    const s = fresh();
    expect(s.score).toEqual([0, 0]);
    expect(s.ball.vx).not.toBe(0);
    expect(s.winner).toBeNull();
  });

  it('centres both paddles', () => {
    const s = fresh();
    expect(s.paddles[0].y).toBe(s.paddles[1].y);
  });
});

describe('movePaddle', () => {
  it('moves a paddle by the step given', () => {
    const s = fresh();
    const before = s.paddles[0].y;
    movePaddle(s, 0, 1, 1);
    expect(s.paddles[0].y).toBeGreaterThan(before);
  });

  it('scales the step by dt', () => {
    const a = fresh(); const b = fresh();
    const start = a.paddles[0].y;
    movePaddle(a, 0, 1, 1);
    movePaddle(b, 0, 1, 2);
    expect(b.paddles[0].y - start).toBeCloseTo((a.paddles[0].y - start) * 2, 6);
  });

  it('clamps at the top and bottom rather than leaving the field', () => {
    const s = fresh();
    movePaddle(s, 0, -1, 1000);
    expect(s.paddles[0].y).toBe(0);
    movePaddle(s, 0, 1, 1000);
    expect(s.paddles[0].y).toBe(s.h - s.paddleH);
  });

  it('moves only the paddle asked for', () => {
    const s = fresh();
    const other = s.paddles[1].y;
    movePaddle(s, 0, 1, 5);
    expect(s.paddles[1].y).toBe(other);
  });
});

describe('advance', () => {
  it('moves the ball by its velocity, scaled by dt', () => {
    const s = fresh();
    const x0 = s.ball.x;
    advance(s, 1);
    const d1 = s.ball.x - x0;
    const t = fresh();
    advance(t, 2);
    expect(t.ball.x - x0).toBeCloseTo(d1 * 2, 6);
  });

  it('bounces off the top edge and ends up inside the field', () => {
    const s = fresh();
    s.ball.y = 1; s.ball.vy = -4;
    advance(s, 1);
    expect(s.ball.vy).toBeGreaterThan(0);
    expect(s.ball.y).toBeGreaterThanOrEqual(0);
  });

  it('bounces off the bottom edge', () => {
    const s = fresh();
    s.ball.y = s.h - 1; s.ball.vy = 4;
    advance(s, 1);
    expect(s.ball.vy).toBeLessThan(0);
    expect(s.ball.y).toBeLessThanOrEqual(s.h - s.ball.size);
  });

  it('scores for the right player when the ball leaves on the left', () => {
    const s = fresh();
    s.ball.x = 1; s.ball.vx = -6;
    expect(advance(s, 1)).toBe(1);
    expect(s.score).toEqual([0, 1]);
  });

  it('scores for the left player when the ball leaves on the right', () => {
    const s = fresh();
    s.ball.x = s.w - 1; s.ball.vx = 6;
    expect(advance(s, 1)).toBe(0);
    expect(s.score).toEqual([1, 0]);
  });

  it('re-serves towards the player who was just scored on', () => {
    const s = fresh();
    s.ball.x = 1; s.ball.vx = -6;
    advance(s, 1);
    expect(s.ball.vx).toBeLessThan(0);
  });

  it('bounces off a paddle and reverses direction', () => {
    const s = fresh();
    s.paddles[0].y = 40;
    s.ball.x = s.paddleW + 1;
    s.ball.y = 44;
    s.ball.vx = -4; s.ball.vy = 0;
    advance(s, 1);
    expect(s.ball.vx).toBeGreaterThan(0);
  });

  it('deflects up off the top of a paddle and down off the bottom', () => {
    const up = fresh();
    up.paddles[0].y = 40;
    up.ball.x = up.paddleW + 1; up.ball.y = 41; up.ball.vx = -4; up.ball.vy = 0;
    advance(up, 1);
    expect(up.ball.vy).toBeLessThan(0);

    const down = fresh();
    down.paddles[0].y = 40;
    down.ball.x = down.paddleW + 1;
    down.ball.y = 40 + down.paddleH - 1;
    down.ball.vx = -4; down.ball.vy = 0;
    advance(down, 1);
    expect(down.ball.vy).toBeGreaterThan(0);
  });

  it('declares a winner at the winning score and then freezes', () => {
    const s = fresh();
    s.score = [WIN_SCORE - 1, 0];
    s.ball.x = s.w - 1; s.ball.vx = 6;
    advance(s, 1);
    expect(s.winner).toBe(0);
    const frozen = JSON.stringify(s);
    advance(s, 1);
    expect(JSON.stringify(s)).toBe(frozen);
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node games/arcade-classico/pong`
Expected: FAIL — `Failed to resolve import "./rules.js"`.

- [ ] **Step 3: Write `games/arcade-classico/pong/rules.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Pong rules. Pure: no renderer, no DOM, no engine.
export interface Paddle { y: number }
export interface Ball { x: number; y: number; vx: number; vy: number; size: number }

export interface PongState {
  w: number; h: number;
  paddleW: number; paddleH: number; paddleSpeed: number;
  paddles: [Paddle, Paddle];
  ball: Ball;
  score: [number, number];
  winner: 0 | 1 | null;
}

export const WIN_SCORE = 5;
const BALL_SPEED = 2.2;

export function createPong(w: number, h: number): PongState {
  const paddleH = 32;
  return {
    w, h,
    paddleW: 4, paddleH, paddleSpeed: 2.4,
    paddles: [{ y: (h - paddleH) / 2 }, { y: (h - paddleH) / 2 }],
    ball: { x: w / 2, y: h / 2, vx: BALL_SPEED, vy: BALL_SPEED * 0.4, size: 4 },
    score: [0, 0],
    winner: null,
  };
}

/** Move one paddle. `dir` is -1, 0 or 1; `dt` is in FRAMES. Clamped to the field. */
export function movePaddle(s: PongState, i: 0 | 1, dir: number, dt: number): void {
  const p = s.paddles[i];
  p.y = Math.max(0, Math.min(s.h - s.paddleH, p.y + dir * s.paddleSpeed * dt));
}

function serve(s: PongState, towards: -1 | 1): void {
  s.ball.x = s.w / 2;
  s.ball.y = s.h / 2;
  s.ball.vx = BALL_SPEED * towards;
  s.ball.vy = BALL_SPEED * 0.4;
}

/**
 * Advance one frame. Returns the index of the player who just scored, or null.
 *
 * Paddle contact is an overlap test rather than a swept one: the ball is slow relative to its own size
 * and the paddle is a wall the full height of its span, so there is nothing to tunnel through.
 * Breakout, whose ball is fast and whose bricks are thin, uses the swept test instead.
 */
export function advance(s: PongState, dt: number): 0 | 1 | null {
  if (s.winner !== null) return null;

  s.ball.x += s.ball.vx * dt;
  s.ball.y += s.ball.vy * dt;

  if (s.ball.y <= 0) { s.ball.y = 0; s.ball.vy = Math.abs(s.ball.vy); }
  if (s.ball.y + s.ball.size >= s.h) { s.ball.y = s.h - s.ball.size; s.ball.vy = -Math.abs(s.ball.vy); }

  for (const i of [0, 1] as const) {
    const px = i === 0 ? 0 : s.w - s.paddleW;
    const p = s.paddles[i];
    const hitX = s.ball.x <= px + s.paddleW && s.ball.x + s.ball.size >= px;
    const hitY = s.ball.y + s.ball.size >= p.y && s.ball.y <= p.y + s.paddleH;
    const movingInto = i === 0 ? s.ball.vx < 0 : s.ball.vx > 0;
    if (hitX && hitY && movingInto) {
      s.ball.vx = -s.ball.vx;
      // Where on the paddle it landed steers the return: -1 at the top edge, +1 at the bottom. Without
      // it every rally is identical, which is the difference between a game and a demo.
      const rel = (s.ball.y + s.ball.size / 2 - p.y) / s.paddleH;
      s.ball.vy = (rel - 0.5) * 2 * BALL_SPEED;
      s.ball.x = i === 0 ? px + s.paddleW : px - s.ball.size;
    }
  }

  if (s.ball.x + s.ball.size < 0) {
    s.score[1]++;
    if (s.score[1] >= WIN_SCORE) { s.winner = 1; return 1; }
    serve(s, -1);
    return 1;
  }
  if (s.ball.x > s.w) {
    s.score[0]++;
    if (s.score[0] >= WIN_SCORE) { s.winner = 0; return 0; }
    serve(s, 1);
    return 0;
  }
  return null;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node games/arcade-classico/pong`
Expected: PASS, 15 tests.

- [ ] **Step 5: Write `games/arcade-classico/pong/strings.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import type { GameStrings } from '../../../engine/game-api.js';

export const strings: GameStrings = {
  pt: {
    start: 'Jogador 1: W e S. Jogador 2: seta para cima e seta para baixo.',
    point: 'Ponto do jogador {who}. {a} a {b}.',
    gameOver: 'Jogador {who} venceu por {a} a {b}.',
  },
  en: {
    start: 'Player 1: W and S. Player 2: up arrow and down arrow.',
    point: 'Point for player {who}. {a} to {b}.',
    gameOver: 'Player {who} won {a} to {b}.',
  },
  es: {
    start: 'Jugador 1: W y S. Jugador 2: flecha arriba y flecha abajo.',
    point: 'Punto para el jugador {who}. {a} a {b}.',
    gameOver: 'El jugador {who} ganó {a} a {b}.',
  },
};
```

- [ ] **Step 6: Write `games/arcade-classico/pong/main.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Pong. Two players from one InputApi: paddle 0 reads player 0's scheme, paddle 1 reads player 1's.
// Movement uses `held`, not `pressed` — a paddle keeps moving while the key is down.
import type { GameContext, GameInstance, GameMeta, Handle } from '../../../engine/game-api.js';
import { advance, createPong, movePaddle } from './rules.js';
import { strings } from './strings.js';

export { strings };

export const meta: GameMeta = {
  slug: 'pong',
  title: 'Pong',
  category: 'arcade-classico',
  density: 'leve',
  players: 2,
};

const FIELD_H = 176;

export function create(ctx: GameContext): GameInstance {
  const top = Math.floor((ctx.view.h - FIELD_H) / 2);
  const state = createPong(ctx.view.w, FIELD_H);
  let done = false;

  const mk = (w: number, h: number, role: 'player' | 'pickup', col: string): Handle =>
    ctx.scene.add({ role, w, h, paint: (px) => px(0, 0, w, h, col) });

  // Both paddles are 'player': to someone using high contrast, both are things a person controls.
  // Tagging one 'hazard' would be a lie about what that colour means everywhere else in the collection.
  const paddles: [Handle, Handle] = [
    mk(state.paddleW, state.paddleH, 'player', '#00d9ff'),
    mk(state.paddleW, state.paddleH, 'player', '#ff2d8e'),
  ];
  const ball = mk(state.ball.size, state.ball.size, 'pickup', '#ececf2');

  paddles[0].x = 0;
  paddles[1].x = ctx.view.w - state.paddleW;

  function render(): void {
    paddles[0].y = top + state.paddles[0].y;
    paddles[1].y = top + state.paddles[1].y;
    ball.x = state.ball.x;
    ball.y = top + state.ball.y;
  }

  render();
  ctx.srSay(ctx.t('start'));

  return {
    update(dt) {
      if (done) return;

      for (const i of [0, 1] as const) {
        const dir = (ctx.input.held(i, 'down') ? 1 : 0) - (ctx.input.held(i, 'up') ? 1 : 0);
        if (dir !== 0) movePaddle(state, i, dir, dt);
      }

      const scorer = advance(state, dt);
      render();
      if (scorer === null) return;

      ctx.audio.beep(scorer === 0 ? 660 : 440, 60);
      const params = { who: scorer + 1, a: state.score[0], b: state.score[1] };
      if (state.winner === null) {
        ctx.srSay(ctx.t('point', params));
      } else {
        done = true;
        ctx.srAlert(ctx.t('gameOver', params));
        ctx.onGameOver(Math.max(state.score[0], state.score[1]));
      }
    },

    teardown() {
      ctx.scene.remove(paddles[0]);
      ctx.scene.remove(paddles[1]);
      ctx.scene.remove(ball);
    },
  };
}
```

- [ ] **Step 7: Run everything**

Run: `npx tsc --noEmit && npx vitest run && npm run lint:deps && npm run format:check`
Expected: all clean.

- [ ] **Step 8: See it run and confirm the two schemes are not crossed**

Run: `npm run build && npm run preview`, open `http://localhost:4173/play.html#arcade-classico/pong`.

- W and S must move the LEFT paddle only; the up and down arrows the RIGHT paddle only.
- Pressing both at once must move both, independently.
- A ball hitting the top of a paddle must come off upwards, the bottom downwards.

- [ ] **Step 9: Commit**

```bash
git add games/arcade-classico/pong/
git commit -m "feat: add Pong, proving two-player input

Snake never calls held() and never uses a second scheme, so crossed player
bindings would have gone unnoticed until every two-player game was broken."
```

---


### Task 16: Breakout — swept collision and role separation

**Files:**
- Create: `games/arcade-classico/breakout/rules.ts`, `strings.ts`, `main.ts`
- Test: `games/arcade-classico/breakout/rules.test.ts`

**Interfaces:**
- Consumes: `sweptAabb`, `type Box`, and the game types — all from `engine/game-api.ts`.
- Produces: nothing other tasks consume. It is the case that proves the swept primitive is load-bearing.

> Breakout is the third reference game because it is the one that breaks if `sweptAabb` is wrong: its
> ball crosses several times a brick's thickness per frame, so an overlap-only engine lets it fly
> through the wall. It is also the game where the role tags earn their keep — a player who cannot
> separate ball from brick cannot play it at all.

- [ ] **Step 1: Write the failing rules test**

`games/arcade-classico/breakout/rules.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { advance, createBreakout, movePaddle, type BreakoutState } from './rules.js';

const fresh = (): BreakoutState => createBreakout(320, 176);

describe('createBreakout', () => {
  it('starts with a full wall, three lives and no score', () => {
    const s = fresh();
    expect(s.bricks.length).toBeGreaterThan(0);
    expect(s.bricks.every((b) => b.alive)).toBe(true);
    expect(s.lives).toBe(3);
    expect(s.score).toBe(0);
  });

  it('lays every brick inside the field', () => {
    const s = fresh();
    for (const b of s.bricks) {
      expect(b.x).toBeGreaterThanOrEqual(0);
      expect(b.x + b.w).toBeLessThanOrEqual(s.w);
      expect(b.y).toBeGreaterThanOrEqual(0);
    }
  });
});

describe('movePaddle', () => {
  it('clamps the paddle inside the field', () => {
    const s = fresh();
    movePaddle(s, -1, 1000);
    expect(s.paddle.x).toBe(0);
    movePaddle(s, 1, 1000);
    expect(s.paddle.x).toBe(s.w - s.paddle.w);
  });
});

describe('advance', () => {
  it('bounces off the left and right walls', () => {
    const left = fresh();
    left.ball.x = 0; left.ball.vx = -3;
    advance(left, 1);
    expect(left.ball.vx).toBeGreaterThan(0);

    const right = fresh();
    right.ball.x = right.w - right.ball.size; right.ball.vx = 3;
    advance(right, 1);
    expect(right.ball.vx).toBeLessThan(0);
  });

  it('bounces off the ceiling', () => {
    const s = fresh();
    s.ball.y = 0; s.ball.vy = -3;
    advance(s, 1);
    expect(s.ball.vy).toBeGreaterThan(0);
  });

  it('destroys a brick it hits and scores for it', () => {
    const s = fresh();
    const brick = s.bricks[0]!;
    s.ball.x = brick.x + brick.w / 2;
    s.ball.y = brick.y + brick.h + 1;
    s.ball.vx = 0; s.ball.vy = -3;
    advance(s, 1);
    expect(brick.alive).toBe(false);
    expect(s.score).toBe(1);
  });

  it('reverses the ball on the axis of the face it struck', () => {
    const s = fresh();
    const brick = s.bricks[0]!;
    s.ball.x = brick.x + brick.w / 2;
    s.ball.y = brick.y + brick.h + 1;
    s.ball.vx = 0; s.ball.vy = -3;
    advance(s, 1);
    expect(s.ball.vy).toBeGreaterThan(0);
  });

  it('does not tunnel through the wall at high speed', () => {
    const s = fresh();
    const brick = s.bricks[0]!;
    s.ball.x = brick.x + brick.w / 2;
    s.ball.y = brick.y + brick.h + 2;
    s.ball.vx = 0;
    s.ball.vy = -200;             // far past the brick in a single frame
    advance(s, 1);
    expect(brick.alive).toBe(false);
  });

  it('breaks at most one brick per frame, so one shot is one point', () => {
    const s = fresh();
    s.ball.x = s.bricks[0]!.x + 1;
    s.ball.y = s.h / 2;
    s.ball.vx = 0; s.ball.vy = -400;
    advance(s, 1);
    expect(s.bricks.filter((b) => !b.alive).length).toBe(1);
  });

  it('loses a life when the ball falls past the floor', () => {
    const s = fresh();
    s.ball.y = s.h + 1; s.ball.vy = 3;
    advance(s, 1);
    expect(s.lives).toBe(2);
  });

  it('ends the game when the last life is gone', () => {
    const s = fresh();
    s.lives = 1;
    s.ball.y = s.h + 1; s.ball.vy = 3;
    advance(s, 1);
    expect(s.over).toBe(true);
  });

  it('bounces off the paddle and steers by where it landed', () => {
    const s = fresh();
    s.paddle.x = 100;
    s.ball.x = 102;
    s.ball.y = s.paddle.y - s.ball.size;
    s.ball.vx = 0; s.ball.vy = 3;
    advance(s, 1);
    expect(s.ball.vy).toBeLessThan(0);
    expect(s.ball.vx).toBeLessThan(0);
  });

  it('is won when the last brick falls', () => {
    const s = fresh();
    for (const b of s.bricks) b.alive = false;
    const brick = s.bricks[0]!;
    brick.alive = true;
    s.ball.x = brick.x + brick.w / 2;
    s.ball.y = brick.y + brick.h + 1;
    s.ball.vx = 0; s.ball.vy = -3;
    advance(s, 1);
    expect(s.won).toBe(true);
  });

  it('freezes once over', () => {
    const s = fresh();
    s.over = true;
    const frozen = JSON.stringify(s);
    advance(s, 1);
    expect(JSON.stringify(s)).toBe(frozen);
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node games/arcade-classico/breakout`
Expected: FAIL — `Failed to resolve import "./rules.js"`.

- [ ] **Step 3: Write `games/arcade-classico/breakout/rules.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Breakout rules. Pure apart from the collision primitive, which is itself pure and comes through the
// public game API rather than by reaching into the engine.
import { sweptAabb, type Box } from '../../../engine/game-api.js';

export interface Brick extends Box { alive: boolean }
export interface Ball { x: number; y: number; vx: number; vy: number; size: number }

export interface BreakoutState {
  w: number; h: number;
  paddle: Box;
  ball: Ball;
  bricks: Brick[];
  score: number;
  lives: number;
  over: boolean;
  won: boolean;
}

const COLS = 10, ROWS = 4, BRICK_H = 8, GAP = 2, TOP = 16;
const PADDLE_SPEED = 3.2;

export function createBreakout(w: number, h: number): BreakoutState {
  const brickW = Math.floor((w - GAP * (COLS + 1)) / COLS);
  const bricks: Brick[] = [];
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      bricks.push({
        x: GAP + c * (brickW + GAP),
        y: TOP + r * (BRICK_H + GAP),
        w: brickW, h: BRICK_H, alive: true,
      });
    }
  }
  return {
    w, h,
    paddle: { x: w / 2 - 20, y: h - 10, w: 40, h: 4 },
    ball: { x: w / 2, y: h / 2, vx: 1.8, vy: 2.4, size: 4 },
    bricks,
    score: 0, lives: 3, over: false, won: false,
  };
}

export function movePaddle(s: BreakoutState, dir: number, dt: number): void {
  s.paddle.x = Math.max(0, Math.min(s.w - s.paddle.w, s.paddle.x + dir * PADDLE_SPEED * dt));
}

function resetBall(s: BreakoutState): void {
  s.ball.x = s.w / 2;
  s.ball.y = s.h / 2;
  s.ball.vx = 1.8;
  s.ball.vy = 2.4;
}

/**
 * Advance one frame. Returns what happened, so the caller can pick a sound without re-deriving it.
 *
 * The brick pass is SWEPT. At full speed the ball travels several times a brick's thickness in one
 * frame, and an overlap test would find it already past the wall with nothing to report. Only the
 * EARLIEST hit resolves: breaking a whole column because the path crossed four bricks would turn one
 * shot into four points.
 */
export function advance(s: BreakoutState, dt: number): 'brick' | 'paddle' | 'wall' | 'life' | null {
  if (s.over || s.won) return null;

  const vx = s.ball.vx * dt, vy = s.ball.vy * dt;
  const box: Box = { x: s.ball.x, y: s.ball.y, w: s.ball.size, h: s.ball.size };

  let first: { t: number; nx: number; ny: number; brick: Brick } | null = null;
  for (const b of s.bricks) {
    if (!b.alive) continue;
    const hit = sweptAabb(box, vx, vy, b);
    if (hit && (first === null || hit.t < first.t)) first = { ...hit, brick: b };
  }

  if (first) {
    s.ball.x += vx * first.t;
    s.ball.y += vy * first.t;
    if (first.nx !== 0) s.ball.vx = -s.ball.vx;
    if (first.ny !== 0) s.ball.vy = -s.ball.vy;
    first.brick.alive = false;
    s.score++;
    if (s.bricks.every((b) => !b.alive)) s.won = true;
    return 'brick';
  }

  s.ball.x += vx;
  s.ball.y += vy;

  let bounced = false;
  if (s.ball.x <= 0) { s.ball.x = 0; s.ball.vx = Math.abs(s.ball.vx); bounced = true; }
  if (s.ball.x + s.ball.size >= s.w) { s.ball.x = s.w - s.ball.size; s.ball.vx = -Math.abs(s.ball.vx); bounced = true; }
  if (s.ball.y <= 0) { s.ball.y = 0; s.ball.vy = Math.abs(s.ball.vy); bounced = true; }

  const p = s.paddle;
  const onPaddle = s.ball.vy > 0
    && s.ball.y + s.ball.size >= p.y && s.ball.y <= p.y + p.h
    && s.ball.x + s.ball.size >= p.x && s.ball.x <= p.x + p.w;
  if (onPaddle) {
    s.ball.y = p.y - s.ball.size;
    s.ball.vy = -Math.abs(s.ball.vy);
    // Where it landed steers the return, same idea as Pong: the paddle is an aiming tool, not a wall.
    // Two occurrences is not a pattern; this is extracted into the engine at the third, not before.
    const rel = (s.ball.x + s.ball.size / 2 - p.x) / p.w;
    s.ball.vx = (rel - 0.5) * 2 * 3;
    return 'paddle';
  }

  if (s.ball.y > s.h) {
    s.lives--;
    if (s.lives <= 0) { s.lives = 0; s.over = true; } else resetBall(s);
    return 'life';
  }

  return bounced ? 'wall' : null;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node games/arcade-classico/breakout`
Expected: PASS, 14 tests — including the tunnelling case, which is the point of the task.

- [ ] **Step 5: Write `games/arcade-classico/breakout/strings.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import type { GameStrings } from '../../../engine/game-api.js';

export const strings: GameStrings = {
  pt: {
    start: 'Use as setas ou A e D para mover a raquete.',
    life: 'Você perdeu uma bola. Restam {lives}.',
    won: 'Parede destruída. {score} tijolos.',
    gameOver: 'Fim de jogo. {score} tijolos.',
  },
  en: {
    start: 'Use the arrows or A and D to move the paddle.',
    life: 'You lost a ball. {lives} left.',
    won: 'Wall cleared. {score} bricks.',
    gameOver: 'Game over. {score} bricks.',
  },
  es: {
    start: 'Usa las flechas o A y D para mover la paleta.',
    life: 'Perdiste una bola. Quedan {lives}.',
    won: 'Muro destruido. {score} ladrillos.',
    gameOver: 'Fin del juego. {score} ladrillos.',
  },
};
```

- [ ] **Step 6: Write `games/arcade-classico/breakout/main.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Breakout. The role tags earn their keep here: bricks are 'goal' (what you are removing), the paddle
// is 'player', the ball is 'pickup'. In high contrast those become three provably distinct colours,
// which matters more here than in the other two — someone who cannot separate ball from brick cannot
// play this at all.
import type { GameContext, GameInstance, GameMeta, Handle } from '../../../engine/game-api.js';
import { advance, createBreakout, movePaddle } from './rules.js';
import { strings } from './strings.js';

export { strings };

export const meta: GameMeta = {
  slug: 'breakout',
  title: 'Breakout',
  category: 'arcade-classico',
  density: 'leve',
  players: 1,
};

const FIELD_H = 176;
const BRICK_COLOURS = ['#ff2d8e', '#ffd60a', '#00d9ff', '#b8ff3d'];

export function create(ctx: GameContext): GameInstance {
  const top = Math.floor((ctx.view.h - FIELD_H) / 2);
  const state = createBreakout(ctx.view.w, FIELD_H);
  let finished = false;

  const mk = (w: number, h: number, role: 'player' | 'pickup' | 'goal', col: string): Handle =>
    ctx.scene.add({ role, w, h, paint: (px) => px(0, 0, w, h, col) });

  const brickHandles = state.bricks.map((b, i) => {
    const h = mk(b.w, b.h, 'goal', BRICK_COLOURS[i % BRICK_COLOURS.length]!);
    h.x = b.x;
    h.y = top + b.y;
    return h;
  });
  const paddle = mk(state.paddle.w, state.paddle.h, 'player', '#ececf2');
  const ball = mk(state.ball.size, state.ball.size, 'pickup', '#ffffff');

  function render(): void {
    state.bricks.forEach((b, i) => { brickHandles[i]!.visible = b.alive; });
    paddle.x = state.paddle.x;
    paddle.y = top + state.paddle.y;
    ball.x = state.ball.x;
    ball.y = top + state.ball.y;
  }

  render();
  ctx.srSay(ctx.t('start'));

  return {
    update(dt) {
      if (finished) return;

      const dir = (ctx.input.held(0, 'right') ? 1 : 0) - (ctx.input.held(0, 'left') ? 1 : 0);
      if (dir !== 0) movePaddle(state, dir, dt);

      const event = advance(state, dt);
      render();

      if (event === 'brick') ctx.audio.beep(660, 30);
      if (event === 'paddle') ctx.audio.beep(440, 30);
      if (event === 'wall') ctx.audio.beep(330, 20);
      if (event === 'life' && !state.over) {
        ctx.audio.beep(180, 120);
        ctx.srSay(ctx.t('life', { lives: state.lives }));
      }

      if (state.won) {
        finished = true;
        ctx.srAlert(ctx.t('won', { score: state.score }));
        ctx.onGameOver(state.score);
      } else if (state.over) {
        finished = true;
        ctx.audio.beep(110, 240);
        ctx.srAlert(ctx.t('gameOver', { score: state.score }));
        ctx.onGameOver(state.score);
      }
    },

    teardown() {
      for (const h of brickHandles) ctx.scene.remove(h);
      brickHandles.length = 0;
      ctx.scene.remove(paddle);
      ctx.scene.remove(ball);
    },
  };
}
```

- [ ] **Step 7: Run everything**

Run: `npx tsc --noEmit && npx vitest run && npm run lint:deps && npm run format:check`
Expected: all clean. The conformance suite now polices three games.

- [ ] **Step 8: See it run**

Run: `npm run build && npm run preview`, open `http://localhost:4173/play.html#arcade-classico/breakout`.

- The ball must never pass through a brick, however fast it moves.
- Exactly one brick disappears per contact.
- `__demos.visual.setLevel(7)` in the console must leave the paddle, the ball and the bricks visibly
  different from one another — that is the whole promise of the role tags.

- [ ] **Step 9: Commit**

```bash
git add games/arcade-classico/breakout/
git commit -m "feat: add Breakout, exercising the swept collision

Its ball crosses several brick-thicknesses per frame, so this is the game that
would tunnel if sweptAabb were wrong. Only the earliest hit resolves, or one
shot down a column would score four."
```

---

### Task 17: Import the catalog into data

**Files:**
- Create: `scripts/catalog-parse.mts`, `scripts/import-catalog.mts`, `data/catalog.json` (generated)
- Test: `scripts/catalog-parse.test.mts`

**Interfaces:**
- Consumes: `minigames-catalog-v2.html` at the repository root.
- Produces: `slugify(name)`, `uniqueSlug(name, taken)`, `parseCatalog(html): Catalog`, the types `Catalog`, `Category`, `Item`, and `data/catalog.json`.

> The parser is separate from the script so it can be tested against a string. The script is the
> one-shot: run it once, commit the output, edit the JSON from then on. Keeping the script afterwards is
> still worth it — it documents how the data was derived, which is the first question anyone asks when a
> field looks wrong.
>
> **It is strict on purpose.** The first draft defaulted a missing density to `leve` and a missing id to
> `0`. That is being liberal in what you accept exactly where RFC 9413 says not to be: a card the parser
> half-understood would produce a plausible-looking entry that is silently wrong, and nobody would find
> it until a category rendered with the wrong colour. A malformed card now stops the import and names
> itself.

- [ ] **Step 1: Write the failing test**

`scripts/catalog-parse.test.mts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { parseCatalog, slugify, uniqueSlug } from './catalog-parse.mts';

const CARD = `
<div class="grid">
  <article class="card cat-1">
    <div class="card-num">01 / 35</div>
    <h3 class="card-title">Arcade Clássico</h3>
    <div class="card-desc">tela única · loop curto · score</div>
    <div class="card-meta">
      <span class="density-dots" data-level="leve"><span class="dot"></span></span>
      <span>Densidade: Leve</span>
    </div>
    <ul>
      <li>Snake / Cobrinha <small>grid · auto-move · cresce</small></li>
      <li class="fresh">Snake roguelite <small>upgrades por morte</small></li>
      <li>Pong</li>
    </ul>
  </article>
  <article class="card cat-31">
    <div class="card-num">31 / 35 <span class="new-badge">NEW</span></div>
    <h3 class="card-title">Cozinha &amp; Produção</h3>
    <div class="card-desc">ordem · tempo</div>
    <div class="card-meta">
      <span class="density-dots" data-level="denso"><span class="dot"></span></span>
      <span>Densidade: Denso</span>
    </div>
    <ul><li>Restaurante &amp; pedidos</li></ul>
  </article>
</div>`;

describe('slugify', () => {
  it('strips accents and lowercases', () => {
    expect(slugify('Arcade Clássico')).toBe('arcade-classico');
    expect(slugify('Cozinha & Produção')).toBe('cozinha-producao');
  });

  it('keeps only the part before a slash, which is the primary name', () => {
    expect(slugify('Snake / Cobrinha')).toBe('snake');
    expect(slugify('2048 / merge numérico')).toBe('2048');
  });

  it('keeps digits and collapses runs of punctuation into one hyphen', () => {
    expect(slugify('Espelhos & laser')).toBe('espelhos-laser');
    expect(slugify('Point-and-Click')).toBe('point-and-click');
  });

  it('never begins or ends with a hyphen', () => {
    expect(slugify('  ...Whack-a-Mole!  ')).toBe('whack-a-mole');
  });

  it('produces something even for a name with no usable characters', () => {
    expect(slugify('???')).toBe('item');
  });

  it('always produces a slug the router will accept', () => {
    const shape = /^[a-z0-9]+(?:-[a-z0-9]+)*$/;
    for (const n of ['Arcade Clássico', 'Snake / Cobrinha', '  ...Whack-a-Mole!  ', 'Espelhos & laser', '2048']) {
      expect(slugify(n), n).toMatch(shape);
    }
  });
});

describe('uniqueSlug', () => {
  it('returns the plain slug when it is free', () => {
    expect(uniqueSlug('Pong', new Set())).toBe('pong');
  });

  it('suffixes rather than overwriting a taken slug', () => {
    expect(uniqueSlug('Pong', new Set(['pong']))).toBe('pong-2');
  });

  it('keeps counting past the first collision', () => {
    expect(uniqueSlug('Pong', new Set(['pong', 'pong-2']))).toBe('pong-3');
  });
});

describe('parseCatalog', () => {
  it('finds every card', () => {
    expect(parseCatalog(CARD).categories.length).toBe(2);
  });

  it('reads the title, description, density and accent of a card', () => {
    const c = parseCatalog(CARD).categories[0]!;
    expect(c.id).toBe(1);
    expect(c.title).toBe('Arcade Clássico');
    expect(c.slug).toBe('arcade-classico');
    expect(c.desc).toBe('tela única · loop curto · score');
    expect(c.density).toBe('leve');
    expect(c.accent).toBe('c1');
  });

  it('decodes HTML entities in titles', () => {
    expect(parseCatalog(CARD).categories[1]!.title).toBe('Cozinha & Produção');
  });

  it('marks a card carrying the NEW badge', () => {
    const [a, b] = parseCatalog(CARD).categories;
    expect(a!.new).toBe(false);
    expect(b!.new).toBe(true);
  });

  it('splits an item into its name and its hint', () => {
    const i = parseCatalog(CARD).categories[0]!.items[0]!;
    expect(i.name).toBe('Snake / Cobrinha');
    expect(i.hint).toBe('grid · auto-move · cresce');
    expect(i.slug).toBe('snake');
  });

  it('leaves the hint empty when an item has no <small>', () => {
    expect(parseCatalog(CARD).categories[0]!.items[2]!.hint).toBe('');
  });

  it('carries the fresh marker', () => {
    const items = parseCatalog(CARD).categories[0]!.items;
    expect(items[0]!.fresh).toBe(false);
    expect(items[1]!.fresh).toBe(true);
  });

  it('starts every item as todo with no alias', () => {
    for (const c of parseCatalog(CARD).categories) {
      for (const i of c.items) {
        expect(i.status).toBe('todo');
        expect(i.aliasOf).toBeNull();
      }
    }
  });

  it('gives every item in a category a distinct slug', () => {
    const c = parseCatalog(CARD).categories[0]!;
    expect(new Set(c.items.map((i) => i.slug)).size).toBe(c.items.length);
  });
});

describe('parseCatalog is strict', () => {
  const withoutCat = CARD.replace('card cat-1', 'card');
  const withoutTitle = CARD.replace('<h3 class="card-title">Arcade Clássico</h3>', '');
  const withoutDensity = CARD.replace('data-level="leve"', '');
  const badDensity = CARD.replace('data-level="leve"', 'data-level="medium"');
  const emptyList = CARD.replace(/<ul>[\s\S]*?<\/ul>/, '<ul></ul>');

  it('refuses a card with no cat-N class instead of guessing zero', () => {
    expect(() => parseCatalog(withoutCat)).toThrow(/cat-N/);
  });

  it('refuses a card with no title', () => {
    expect(() => parseCatalog(withoutTitle)).toThrow(/title/i);
  });

  it('refuses a card with no density instead of assuming the easiest one', () => {
    expect(() => parseCatalog(withoutDensity)).toThrow(/density/i);
  });

  it('refuses a density outside the three known values', () => {
    expect(() => parseCatalog(badDensity)).toThrow(/medium/);
  });

  it('refuses a category with no items', () => {
    expect(() => parseCatalog(emptyList)).toThrow(/no items/i);
  });

  it('names the offending card, so the message is actionable', () => {
    expect(() => parseCatalog(badDensity)).toThrow(/Arcade Clássico/);
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node scripts/catalog-parse.test.mts`
Expected: FAIL — `Failed to resolve import "./catalog-parse.mts"`.

- [ ] **Step 3: Write `scripts/catalog-parse.mts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Parse the hand-written catalog page into data. Separated from the script that reads and writes files
// so it can be tested against a string.
import { parse } from 'node-html-parser';

export type Density = 'leve' | 'medio' | 'denso';
export type Status = 'todo' | 'wip' | 'done';
const DENSITIES: readonly string[] = ['leve', 'medio', 'denso'];

export interface Item {
  slug: string;
  name: string;
  hint: string;
  fresh: boolean;
  status: Status;
  /** "<category>/<slug>" when this entry is another game under a different name. */
  aliasOf: string | null;
}

export interface Category {
  id: number;
  slug: string;
  title: string;
  desc: string;
  density: Density;
  accent: string;
  new: boolean;
  items: Item[];
}

export interface Catalog { version: string; categories: Category[] }

/**
 * A folder-safe slug.
 *
 * Only the part before a slash survives: entries read "Snake / Cobrinha" or "2048 / merge numérico",
 * where the second half is a gloss rather than part of the name, and folding it in would produce
 * `snake-cobrinha` for a folder everyone will call snake.
 */
export function slugify(name: string): string {
  const primary = name.split('/')[0] ?? name;
  const out = primary
    .normalize('NFD').replace(/[̀-ͯ]/g, '')   // drop combining accents
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '');
  return out || 'item';
}

/** `slugify` plus a numeric suffix when the slug is already taken. Mutates nothing. */
export function uniqueSlug(name: string, taken: ReadonlySet<string>): string {
  const base = slugify(name);
  if (!taken.has(base)) return base;
  let n = 2;
  while (taken.has(`${base}-${n}`)) n++;
  return `${base}-${n}`;
}

/**
 * Read the page into data, refusing anything it only half-understands.
 *
 * Strictness is the whole design here. A tolerant parser would emit a card with density `leve` and
 * accent `c0` from markup it failed to read, and that entry would look completely ordinary in the JSON.
 * The failure would surface months later as a category rendered in the wrong colour, with nothing
 * pointing back to this function. Refusing early, naming the card, costs one clear error instead.
 */
export function parseCatalog(html: string): Catalog {
  const root = parse(html);
  const cards = root.querySelectorAll('article.card');
  if (cards.length === 0) throw new Error('parseCatalog: no article.card found — is this the right file?');

  const categories = cards.map((card): Category => {
    const title = card.querySelector('.card-title')?.textContent.trim() ?? '';
    const where = title || card.querySelector('.card-num')?.textContent.trim() || '(unnamed card)';
    if (!title) throw new Error(`parseCatalog: card has no .card-title (near "${where}")`);

    const catClass = card.classNames.split(/\s+/).find((c) => /^cat-\d+$/.test(c));
    if (!catClass) throw new Error(`parseCatalog: "${title}" has no cat-N class, so it has no accent colour`);
    const id = Number(catClass.slice(4));

    const density = card.querySelector('.density-dots')?.getAttribute('data-level');
    if (!density) throw new Error(`parseCatalog: "${title}" has no density (data-level on .density-dots)`);
    if (!DENSITIES.includes(density)) {
      throw new Error(`parseCatalog: "${title}" has density "${density}", expected one of ${DENSITIES.join(', ')}`);
    }

    const taken = new Set<string>();
    const items = card.querySelectorAll('ul > li').map((li): Item => {
      const small = li.querySelector('small');
      const hint = small?.textContent.trim() ?? '';
      // Remove the hint before reading the name, or the name would swallow it.
      if (small) small.remove();
      const name = li.textContent.replace(/\s+/g, ' ').trim();
      if (!name) throw new Error(`parseCatalog: "${title}" has an empty <li>`);
      const slug = uniqueSlug(name, taken);
      taken.add(slug);
      return { slug, name, hint, fresh: li.classNames.split(/\s+/).includes('fresh'), status: 'todo', aliasOf: null };
    });
    if (items.length === 0) throw new Error(`parseCatalog: "${title}" has no items`);

    return {
      id,
      slug: slugify(title),
      title,
      desc: card.querySelector('.card-desc')?.textContent.trim() ?? '',
      density: density as Density,
      accent: `c${id}`,
      new: card.querySelector('.new-badge') !== null,
      items,
    };
  });

  return { version: '2.0.0', categories };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node scripts/catalog-parse.test.mts`
Expected: PASS, 22 tests.

- [ ] **Step 5: Write `scripts/import-catalog.mts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// One-shot: minigames-catalog-v2.html -> data/catalog.json.
//
// Run once; commit the output; edit the JSON from then on. The script stays as the record of how the
// data was derived.
//
//   node --experimental-strip-types scripts/import-catalog.mts
import { mkdirSync, readFileSync, writeFileSync } from 'node:fs';
import { dirname, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';
import { parseCatalog } from './catalog-parse.mts';

const root = resolve(dirname(fileURLToPath(import.meta.url)), '..');

const html = readFileSync(resolve(root, 'minigames-catalog-v2.html'), 'utf8');
const catalog = parseCatalog(html);

const items = catalog.categories.reduce((n, c) => n + c.items.length, 0);
console.log(`parsed ${catalog.categories.length} categories, ${items} items`);

mkdirSync(resolve(root, 'data'), { recursive: true });
writeFileSync(resolve(root, 'data/catalog.json'), `${JSON.stringify(catalog, null, 2)}\n`, 'utf8');
console.log('wrote data/catalog.json');
```

- [ ] **Step 6: Run the import and check the numbers**

Run: `node --experimental-strip-types scripts/import-catalog.mts`
Expected: `parsed 35 categories, 383 items`, then `wrote data/catalog.json`.

If the counts differ from 35 and 383, or the script throws, stop and find out why. Those are the numbers
measured from the source page, and the parser is strict precisely so a mismatch surfaces here.

- [ ] **Step 7: Mark the three built games as done**

Edit `data/catalog.json` by hand: in category `arcade-classico`, set `"status": "done"` on the items
whose slugs are `snake`, `pong` and `breakout`.

- [ ] **Step 8: Commit**

```bash
git add scripts/catalog-parse.mts scripts/catalog-parse.test.mts scripts/import-catalog.mts data/catalog.json
git commit -m "feat: import the catalog page into data/catalog.json

The parser refuses a card it only half-understands rather than defaulting the
density and the accent: a tolerant parser would emit an ordinary-looking entry
that is silently wrong, and the failure would surface months later as a
category in the wrong colour with nothing pointing back here.

Slugs take the part before a slash, because 'Snake / Cobrinha' names one game
that everyone will call snake."
```

---

### Task 18: Generate the catalog page

**Files:**
- Create: `scripts/catalog-render.mts`, `scripts/build-catalog.mts`, `src/catalog.template.html`
- Modify: `index.html` (now generated), delete `minigames-catalog-v2.html`
- Test: `scripts/catalog-render.test.mts`

**Interfaces:**
- Consumes: `data/catalog.json` and the types from Task 17.
- Produces: `renderGrid(catalog): string`, `computeStats(catalog): Stats`, `renderPage(template, catalog): string`, and the generated `index.html`.

> This is where D8 pays off: the six invariants that had to be swept by hand — the `--cN` token, the
> `NN / 35` denominator in every card, the section count, the five hero statistics, the version string
> and the changelog — collapse into one data file plus one renderer. It also fixes the hero's stale
> "280+", because the statistic is now counted rather than typed.

- [ ] **Step 1: Write the failing test**

`scripts/catalog-render.test.mts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { computeStats, renderGrid, renderPage } from './catalog-render.mts';
import type { Catalog } from './catalog-parse.mts';

const catalog: Catalog = {
  version: '2.0.0',
  categories: [
    {
      id: 1, slug: 'arcade-classico', title: 'Arcade Clássico', desc: 'tela única',
      density: 'leve', accent: 'c1', new: false,
      items: [
        { slug: 'snake', name: 'Snake / Cobrinha', hint: 'grid', fresh: false, status: 'done', aliasOf: null },
        { slug: 'pong', name: 'Pong', hint: '', fresh: false, status: 'todo', aliasOf: null },
        { slug: 'snake-roguelite', name: 'Snake roguelite', hint: 'upgrades', fresh: true, status: 'todo', aliasOf: 'hibridos/snake-roguelite' },
      ],
    },
    {
      id: 31, slug: 'cozinha', title: 'Cozinha & Produção', desc: 'ordem',
      density: 'denso', accent: 'c31', new: true,
      items: [{ slug: 'restaurante', name: 'Restaurante', hint: '', fresh: false, status: 'todo', aliasOf: null }],
    },
  ],
};

describe('computeStats', () => {
  it('counts categories and items rather than trusting a typed figure', () => {
    const s = computeStats(catalog);
    expect(s.categories).toBe(2);
    expect(s.items).toBe(4);
  });

  it('counts the new categories', () => {
    expect(computeStats(catalog).newCategories).toBe(1);
  });

  it('counts what is playable', () => {
    expect(computeStats(catalog).done).toBe(1);
  });

  it('excludes aliases from the count of games to build', () => {
    expect(computeStats(catalog).unique).toBe(3);
  });
});

describe('renderGrid', () => {
  const html = renderGrid(catalog);

  it('emits one article per category with its accent class', () => {
    expect(html.match(/<article class="card cat-/g)!.length).toBe(2);
    expect(html).toContain('class="card cat-31"');
  });

  it('numbers each card against the real total', () => {
    expect(html).toContain('01 / 2');
    expect(html).toContain('31 / 2');
  });

  it('carries the density level and its label', () => {
    expect(html).toContain('data-level="leve"');
    expect(html).toContain('Densidade: Leve');
    expect(html).toContain('Densidade: Denso');
  });

  it('emits exactly three dots per density indicator', () => {
    const first = html.slice(html.indexOf('density-dots'));
    expect(first.slice(0, 200).match(/<span class="dot"><\/span>/g)!.length).toBe(3);
  });

  it('badges a new category and not an old one', () => {
    expect(html.match(/class="new-badge"/g)!.length).toBe(1);
  });

  it('links a done item to its game', () => {
    expect(html).toContain('href="play.html#arcade-classico/snake"');
  });

  it('leaves a todo item as plain text with no link', () => {
    const pong = html.slice(html.indexOf('>Pong'), html.indexOf('>Pong') + 40);
    expect(pong).not.toContain('href');
  });

  it('marks a fresh item with the fresh class', () => {
    expect(html).toContain('class="fresh"');
  });

  it('escapes HTML in names, so an ampersand cannot break the page', () => {
    expect(html).toContain('Cozinha &amp; Produção');
    expect(html).not.toContain('Cozinha & Produção');
  });

  it('omits the small element when there is no hint', () => {
    const pong = html.slice(html.indexOf('>Pong'), html.indexOf('>Pong') + 40);
    expect(pong).not.toContain('<small>');
  });
});

describe('renderPage', () => {
  const template = '<html><body><!--GRID--><p id="s">{{STAT_CATEGORIES}}/{{STAT_ITEMS}}/{{STAT_DONE}}</p></body></html>';

  it('substitutes the grid for its marker', () => {
    expect(renderPage(template, catalog)).toContain('<article class="card cat-1"');
    expect(renderPage(template, catalog)).not.toContain('<!--GRID-->');
  });

  it('substitutes every statistic placeholder', () => {
    const out = renderPage(template, catalog);
    expect(out).toContain('2/4/1');
    expect(out).not.toContain('{{');
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npx vitest run --project node scripts/catalog-render.test.mts`
Expected: FAIL — `Failed to resolve import "./catalog-render.mts"`.

- [ ] **Step 3: Write `scripts/catalog-render.mts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// Render data/catalog.json into the catalog grid. Pure string work, no file system, so it is testable
// and so the same function can later feed something other than a static page.
import type { Catalog, Category, Item } from './catalog-parse.mts';

export interface Stats {
  categories: number;
  items: number;
  /** Items that need a game built: everything except aliases of another entry. */
  unique: number;
  newCategories: number;
  done: number;
}

export function computeStats(catalog: Catalog): Stats {
  const all = catalog.categories.flatMap((c) => c.items);
  return {
    categories: catalog.categories.length,
    items: all.length,
    unique: all.filter((i) => i.aliasOf === null).length,
    newCategories: catalog.categories.filter((c) => c.new).length,
    done: all.filter((i) => i.status === 'done').length,
  };
}

const esc = (s: string): string =>
  s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');

const DENSITY_LABEL: Record<string, string> = { leve: 'Leve', medio: 'Médio', denso: 'Denso' };

function renderItem(cat: Category, item: Item): string {
  const cls = item.fresh ? ' class="fresh"' : '';
  const hint = item.hint ? ` <small>${esc(item.hint)}</small>` : '';
  // Only a built game becomes a link. Everything else stays plain text, so the page never offers a
  // door that opens onto nothing — and the growing number of links IS the progress report.
  const target = item.aliasOf ?? `${cat.slug}/${item.slug}`;
  const label = item.status === 'done'
    ? `<a href="play.html#${target}">${esc(item.name)}</a>`
    : esc(item.name);
  return `        <li${cls}>${label}${hint}</li>`;
}

function renderCard(cat: Category, total: number): string {
  const num = String(cat.id).padStart(2, '0');
  const badge = cat.new ? ' <span class="new-badge">NEW</span>' : '';
  const dots = '<span class="dot"></span>'.repeat(3);
  return [
    `    <article class="card cat-${cat.id}">`,
    `      <div class="card-num">${num} / ${total}${badge}</div>`,
    `      <h3 class="card-title">${esc(cat.title)}</h3>`,
    `      <div class="card-desc">${esc(cat.desc)}</div>`,
    '      <div class="card-meta">',
    `        <span class="density-dots" data-level="${cat.density}">${dots}</span>`,
    `        <span>Densidade: ${DENSITY_LABEL[cat.density] ?? cat.density}</span>`,
    '      </div>',
    '      <ul>',
    cat.items.map((i) => renderItem(cat, i)).join('\n'),
    '      </ul>',
    '    </article>',
  ].join('\n');
}

export function renderGrid(catalog: Catalog): string {
  const total = catalog.categories.length;
  return catalog.categories.map((c) => renderCard(c, total)).join('\n\n');
}

/** Fill the template: the grid marker plus every {{STAT_*}} placeholder. */
export function renderPage(template: string, catalog: Catalog): string {
  const s = computeStats(catalog);
  return template
    .replace('<!--GRID-->', renderGrid(catalog))
    .replaceAll('{{STAT_CATEGORIES}}', String(s.categories))
    .replaceAll('{{STAT_ITEMS}}', String(s.items))
    .replaceAll('{{STAT_UNIQUE}}', String(s.unique))
    .replaceAll('{{STAT_NEW}}', String(s.newCategories))
    .replaceAll('{{STAT_DONE}}', String(s.done))
    .replaceAll('{{VERSION}}', catalog.version)
    .replaceAll('{{COUNT_RANGE}}', `001 → ${String(s.categories).padStart(3, '0')}`);
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node scripts/catalog-render.test.mts`
Expected: PASS, 17 tests.

- [ ] **Step 5: Build the template from the existing page**

Create `src/catalog.template.html` from `minigames-catalog-v2.html` with exactly these edits, keeping
every byte of CSS and prose otherwise untouched:

1. Replace the entire `<div class="grid"> … </div>` block with:
   ```html
   <div class="grid">
   <!--GRID-->
   </div>
   ```
2. In the hero statistics, replace the five hardcoded numbers with `{{STAT_CATEGORIES}}`,
   `{{STAT_ITEMS}}`, `{{STAT_NEW}}`, `1` (the file count, still literal) and `0` (the dependency count,
   still literal). Add a sixth block before the others:
   ```html
   <div>
     <div class="stat-num alt-2">{{STAT_DONE}}</div>
     <div class="stat-label">Jogáveis</div>
   </div>
   ```
3. Replace the version string in all five places (`<title>`, the `:root` comment, `.hero-tag .ver`, the
   changelog tag and the footer) with `{{VERSION}}`.
4. Replace the text of `.section-head .count` with `{{COUNT_RANGE}}`.

- [ ] **Step 6: Write `scripts/build-catalog.mts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// data/catalog.json + src/catalog.template.html -> index.html
//
//   node --experimental-strip-types scripts/build-catalog.mts
import { readFileSync, writeFileSync } from 'node:fs';
import { dirname, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';
import type { Catalog } from './catalog-parse.mts';
import { computeStats, renderPage } from './catalog-render.mts';

const root = resolve(dirname(fileURLToPath(import.meta.url)), '..');

const catalog = JSON.parse(readFileSync(resolve(root, 'data/catalog.json'), 'utf8')) as Catalog;
const template = readFileSync(resolve(root, 'src/catalog.template.html'), 'utf8');
const out = renderPage(template, catalog);

if (out.includes('{{') || out.includes('<!--GRID-->')) {
  throw new Error('build-catalog: the template still has unfilled placeholders.');
}

writeFileSync(resolve(root, 'index.html'), out, 'utf8');

const s = computeStats(catalog);
console.log(`index.html: ${s.categories} categories, ${s.items} items, ${s.done} playable`);
```

- [ ] **Step 7: Wire it into the build and generate the page**

Add to `package.json` scripts:
```json
"catalog": "node --experimental-strip-types scripts/build-catalog.mts",
"prebuild": "npm run catalog"
```

Run: `npm run catalog`
Expected: `index.html: 35 categories, 383 items, 3 playable`.

- [ ] **Step 8: Check the generated page in a browser**

Run: `npm run build && npm run preview`, open `http://localhost:4173/`.

- The page must look like the original: same hero, same neon cards, same fonts.
- The hero must now read **383**, not "280+".
- Exactly three entries — Snake, Pong, Breakout — must be links; every other entry plain text.
- Clicking Snake must open it and it must be playable.

- [ ] **Step 9: Retire the hand-written page**

```bash
git rm minigames-catalog-v2.html
```

- [ ] **Step 10: Commit**

```bash
git add scripts/catalog-render.mts scripts/catalog-render.test.mts scripts/build-catalog.mts src/catalog.template.html index.html package.json
git commit -m "feat: generate index.html from the catalog data

Retires the hand-written page and with it the six invariants that had to be
swept by hand on every edit. The hero statistics are counted now, which is why
the item count reads 383 instead of the stale 280+.

See the design spec, section 5, for what happened to minigames-catalog-v2.html."
```

---


### Task 19: PWA, the accessibility gate and CI

**Files:**
- Modify: `vite.config.ts`, `package.json`
- Create: `scripts/axe-check.mjs`, `.gitlab-ci.yml`, `public/manifest-icon.svg`

**Interfaces:**
- Consumes: the built `dist/` from every earlier task.
- Produces: an offline-capable shell, a failing build on any WCAG A/AA violation in the DOM, and a CI pipeline that runs typecheck, tests, build and the gate.

> Two things differ from the tracer, both because there are 383 games rather than one. Its Workbox
> config precaches `**/*.{js,css,html,…}`, which here would make someone opening Snake download the
> entire collection — so only the shell, the catalog and the engine are precached, and a game's chunk
> is cached the first time it is played. And the axe gate runs over two pages instead of one.

- [ ] **Step 1: Add the PWA plugin to `vite.config.ts`**

Add the import and the plugin. Everything else in the file stays as it is:
```ts
import { VitePWA } from 'vite-plugin-pwa';

// inside defineConfig({ ... }):
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      injectRegister: 'auto',
      includeAssets: ['manifest-icon.svg'],
      manifest: {
        name: 'JS Minigames',
        short_name: 'Minigames',
        description: 'Coleção de minigames em pixel art 320x180.',
        lang: 'pt-BR',
        start_url: './index.html',
        display: 'standalone',
        background_color: '#0a0a12',
        theme_color: '#0a0a12',
        icons: [{ src: 'manifest-icon.svg', sizes: 'any', type: 'image/svg+xml', purpose: 'any' }],
      },
      workbox: {
        // Precache the SHELL only. Precaching every game would mean a visitor who opened one game
        // downloaded all 383 — the opposite of what the lazy chunks were for.
        globPatterns: ['index.html', 'play.html', 'assets/*.css', 'assets/index-*.js', 'assets/play-*.js', '*.svg', '*.webmanifest'],
        // Game chunks are cached the first time they are played, which is also what makes a game
        // replayable offline afterwards.
        runtimeCaching: [
          {
            urlPattern: /\/assets\/main-.*\.js$/,
            handler: 'CacheFirst',
            options: { cacheName: 'games', expiration: { maxEntries: 60 } },
          },
          {
            urlPattern: /^https:\/\/fonts\.(googleapis|gstatic)\.com\//,
            handler: 'StaleWhileRevalidate',
            options: { cacheName: 'fonts' },
          },
        ],
        cleanupOutdatedCaches: true,
      },
    }),
  ],
```

- [ ] **Step 2: Create `public/manifest-icon.svg`**

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64" role="img" aria-label="JS Minigames">
  <rect width="64" height="64" fill="#0a0a12"/>
  <rect x="8" y="24" width="8" height="8" fill="#b8ff3d"/>
  <rect x="16" y="24" width="8" height="8" fill="#b8ff3d"/>
  <rect x="24" y="24" width="8" height="8" fill="#b8ff3d"/>
  <rect x="40" y="32" width="8" height="8" fill="#ff2d8e"/>
</svg>
```

- [ ] **Step 3: Write `scripts/axe-check.mjs`**

Adapted from `<TRACER>/scripts/axe-check.mjs`. The tracer's VLibras exclusions are dropped — there is
no third-party widget here — and it walks two pages instead of one:
```js
// SPDX-License-Identifier: GPL-3.0-or-later
// a11y gate: runs axe-core against the RUNNING build (live DOM plus CSS), which is the only reliable
// way to measure it. Exits 1 on any WCAG A/AA violation.
//
//   npm run build && npm run preview &
//   AXE_URL=http://localhost:4173 node scripts/axe-check.mjs
//
// WHAT THIS DOES NOT COVER, stated so a green run is never mistaken for conformance: axe cannot see
// inside a <canvas>. Art contrast, flash cadence and the legibility of in-game text are outside it and
// remain human judgement. See the design spec, section 6.
import { chromium } from 'playwright';
import { AxeBuilder } from '@axe-core/playwright';

const BASE = process.env.AXE_URL || 'http://localhost:4173';
const PAGES = ['/index.html', '/play.html#arcade-classico/snake'];

const browser = await chromium.launch();
let failed = 0;
try {
  const context = await browser.newContext();
  for (const path of PAGES) {
    const page = await context.newPage();
    await page.goto(BASE + path, { waitUntil: 'networkidle' });
    // Let the shell settle. The canvas itself is not axe-scannable; its accessibility is the DOM shell.
    await page.waitForSelector('#sr-status, .grid', { timeout: 10_000 });

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
      .analyze();

    if (results.violations.length) {
      console.error(`\n✗ ${path}`);
      console.error(JSON.stringify(results.violations, null, 2));
      failed += results.violations.length;
    } else {
      console.log(`✓ ${path}: 0 WCAG A/AA violations in the DOM shell`);
    }
    await page.close();
  }
} finally {
  await browser.close();
}

if (failed) {
  console.error(`\n✗ axe: ${failed} WCAG A/AA violation(s).`);
  process.exit(1);
}
console.log('\n✓ axe: DOM shell clean. Canvas contents are NOT covered — see the design spec, section 6.');
```

- [ ] **Step 4: Add the scripts**

Add to `package.json` — `validate` already exists from Tasks 1 and 12, so only extend it:
```json
"test:a11y": "node scripts/axe-check.mjs"
```

- [ ] **Step 5: Run the gate locally**

Run, in two terminals:
```bash
npm run build && npm run preview
```
```bash
AXE_URL=http://localhost:4173 node scripts/axe-check.mjs
```
Expected: `✓ /index.html` and `✓ /play.html#…`, then the closing line about what is not covered.

If it reports violations, fix the markup rather than adding an exclusion. The likeliest findings are a
missing accessible name on a link, or a colour pair in the catalog CSS below 4.5:1 — both are real.

- [ ] **Step 6: Write `.gitlab-ci.yml`**

```yaml
# SPDX-License-Identifier: GPL-3.0-or-later
default:
  image: node:24
  cache:
    key:
      files: [package-lock.json]
    paths: [.npm/]

stages: [check, gate]

variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"

.install: &install
  - npm ci --prefer-offline

typecheck:
  stage: check
  script:
    - *install
    - npm run typecheck

# The D15 boundary and the formatter are gates, not suggestions: a game that reaches past game-api.ts,
# or a file nobody formatted, fails here rather than setting a precedent.
lint:
  stage: check
  script:
    - *install
    - npm run format:check
    - npm run lint:deps

test:
  stage: check
  script:
    - *install
    - npx playwright install --with-deps chromium
    - npx vitest run
  artifacts:
    when: always
    reports:
      junit: junit.xml
    expire_in: 1 week

build:
  stage: check
  script:
    - *install
    - npm run build
  artifacts:
    paths: [dist/]
    expire_in: 1 week

a11y:
  stage: gate
  needs: [build]
  script:
    - *install
    - npx playwright install --with-deps chromium
    - npx vite preview --port 4173 &
    - npx wait-on http://localhost:4173
    - AXE_URL=http://localhost:4173 node scripts/axe-check.mjs
```

`wait-on` is already a devDependency from Task 1. Configure Vitest's JUnit reporter by adding to the
`test` block in `vite.config.ts`:
```ts
    reporters: process.env['CI'] ? ['default', 'junit'] : ['default'],
    outputFile: { junit: 'junit.xml' },
```

- [ ] **Step 7: Verify offline actually works**

Run: `npm run build && npm run preview`. Open `http://localhost:4173/`, play Snake once, then in the
browser's devtools set the network to Offline and reload.

Expected: the catalog and Snake both still load. A game never opened before must NOT be available
offline — that is the runtime-caching decision working, not a bug.

- [ ] **Step 8: Commit**

```bash
git add vite.config.ts package.json package-lock.json scripts/axe-check.mjs .gitlab-ci.yml public/manifest-icon.svg
git commit -m "feat: add PWA caching, the axe gate and CI

Only the shell is precached; a game's chunk is cached the first time it is
played, so opening one game does not download 383. The gate states in its own
output that it cannot see inside the canvas."
```

---


### Task 20: Make the accessibility settings reachable

**Files:**
- Modify: `play.html` (pause dialog), `engine/shell/shell.css`
- Create: `engine/shell/settings.ts`
- Test: `engine/shell/settings.browser.test.ts`

**Interfaces:**
- Consumes: `I18n` (Task 4), `VisualState` (Task 8), `Viz` (Task 9), `storage` (Task 2).
- Produces: `mountSettings(deps: SettingsDeps): void`, called by `boot` and again on every language change.

> **Why this task exists.** Tasks 4, 8 and 9 build language switching, high contrast and colour-vision
> filters, and leave every one of them reachable only from the browser console. A person who needs high
> contrast cannot open a console. An accessibility feature nobody can turn on is not an accessibility
> feature, so the controls are part of Phase 1 rather than a later polish pass.
>
> They live in the pause dialog, which already has the focus contract from Task 11 — no second modal, no
> second focus trap to get wrong. And they take their collaborators as arguments, like everything else
> here, so the test drives them without touching a global.

- [ ] **Step 1: Add the controls to the pause dialog in `play.html`**

Insert inside `.pause-card`, after the `#pause-menu` div:
```html
        <div class="settings">
          <p><label for="set-lang" data-i18n="shell.language">Idioma</label>
            <select id="set-lang"></select></p>
          <p><label for="set-contrast" data-i18n="shell.contrast">Alto contraste</label>
            <select id="set-contrast">
              <option value="0" data-i18n="shell.contrastOff">Desligado</option>
              <option value="3">3:1</option>
              <option value="4.5">4.5:1</option>
              <option value="7">7:1</option>
            </select></p>
          <p><label for="set-viz" data-i18n="viz.none">Cores</label>
            <select id="set-viz"></select></p>
        </div>
```

Add to `engine/shell/shell.css`:
```css
.settings { margin-top: 14px; padding-top: 12px; border-top: 1px solid var(--line); font-size: 13px; }
.settings p { display: flex; gap: 8px; align-items: center; justify-content: space-between; margin-top: 6px; }
.settings select { font: inherit; background: #161624; color: var(--fg); border: 1px solid var(--line); padding: 3px 6px; }
.settings select:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }
```

- [ ] **Step 2: Write the failing test**

`engine/shell/settings.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { createI18n } from '../core/i18n.js';
import { createVisualState } from '../render/high-contrast.js';
import { createViz } from '../render/viz.js';
import { mountSettings } from './settings.js';

let region: HTMLElement;
let deps: Parameters<typeof mountSettings>[0];

beforeEach(() => {
  const m = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => m.get(k) ?? null,
    setItem: (k: string, v: string) => { m.set(k, v); },
    removeItem: (k: string) => { m.delete(k); },
  });
  document.body.innerHTML = `
    <div id="game-region">
      <label for="set-lang">L</label><select id="set-lang"></select>
      <label for="set-contrast">C</label>
      <select id="set-contrast">
        <option value="0">off</option><option value="3">3</option>
        <option value="4.5">4.5</option><option value="7">7</option>
      </select>
      <label for="set-viz">V</label><select id="set-viz"></select>
    </div>`;
  region = document.querySelector('#game-region')!;
  deps = { root: document, i18n: createI18n(), visual: createVisualState(), viz: createViz(region) };
  mountSettings(deps);
});

const pick = (id: string, value: string): void => {
  const el = document.querySelector<HTMLSelectElement>(id)!;
  el.value = value;
  el.dispatchEvent(new Event('change', { bubbles: true }));
};

describe('mountSettings', () => {
  it('fills the language select with the three floor languages', () => {
    expect(document.querySelectorAll('#set-lang option').length).toBe(3);
  });

  it('names each language IN that language, so nobody is trapped in one they cannot read', () => {
    const labels = [...document.querySelectorAll('#set-lang option')].map((o) => o.textContent);
    expect(labels).toEqual(['Português', 'English', 'Español']);
  });

  it('TAGS each language option with its own lang, so a screen reader pronounces it', () => {
    // WCAG 3.1.2. Without this the rescue option reads with the wrong phonetics and stops being a
    // rescue for the person who needs it.
    const langs = [...document.querySelectorAll('#set-lang option')].map((o) => o.getAttribute('lang'));
    expect(langs).toEqual(['pt-BR', 'en', 'es']);
  });

  it('fills the colour select from the viz mode table', () => {
    expect(document.querySelectorAll('#set-viz option').length).toBe(8);
  });

  it('labels every option with translated text rather than a raw key', () => {
    for (const o of document.querySelectorAll('#set-viz option')) {
      expect(o.textContent).not.toMatch(/^viz\./);
      expect(o.textContent!.trim().length).toBeGreaterThan(0);
    }
  });

  it('shows the current contrast level as the selected option', () => {
    deps.visual.setLevel(4.5);
    mountSettings(deps);
    expect(document.querySelector<HTMLSelectElement>('#set-contrast')!.value).toBe('4.5');
  });

  it('changes the contrast level when the control changes', () => {
    pick('#set-contrast', '7');
    expect(deps.visual.level()).toBe(7);
  });

  it('turns contrast back off', () => {
    pick('#set-contrast', '7');
    pick('#set-contrast', '0');
    expect(deps.visual.level()).toBe(0);
  });

  it('persists the contrast level so it survives a reload', () => {
    pick('#set-contrast', '3');
    expect(localStorage.getItem('demos.contrast')).toBe('3');
  });

  it('applies a colour-vision mode to the game region', () => {
    pick('#set-viz', 'fix-deuter');
    expect(deps.viz.mode()).toBe('fix-deuter');
    expect(region.style.filter).toContain('url(#cvd-');
  });

  it('switches the language', async () => {
    pick('#set-lang', 'en');
    await new Promise((r) => setTimeout(r, 0));
    expect(deps.i18n.locale()).toBe('en');
  });

  it('every control is associated with a label', () => {
    for (const id of ['set-lang', 'set-contrast', 'set-viz']) {
      expect(document.querySelector(`label[for="${id}"]`), id).not.toBeNull();
    }
  });

  it('re-mounting does not duplicate the options', () => {
    mountSettings(deps);
    mountSettings(deps);
    expect(document.querySelectorAll('#set-lang option').length).toBe(3);
  });
});
```

- [ ] **Step 3: Run it to verify it fails**

Run: `npx vitest run --project browser engine/shell/settings.browser.test.ts`
Expected: FAIL — `Failed to resolve import "./settings.js"`.

- [ ] **Step 4: Write `engine/shell/settings.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// shell/settings — the three accessibility controls, inside the pause dialog.
//
// They are plain <select> elements on purpose. A custom widget would need its own keyboard handling,
// its own ARIA and its own tests, and would end up worse than what every browser and every screen
// reader already implements correctly for a select.
import * as store from '../platform/storage.js';
import type { I18n } from '../core/i18n.js';
import type { VisualState, ContrastLevel } from '../render/high-contrast.js';
import { VIZ_MODES, type Viz, type VizKey } from '../render/viz.js';

export interface SettingsDeps {
  root: ParentNode;
  i18n: I18n;
  visual: VisualState;
  viz: Viz;
}

// Language names are written in their OWN language and never translated. Someone who cannot read the
// current interface language must still be able to find their way out of it.
//
// Which is precisely why each option needs its own `lang`: the text is deliberately in a language the
// page is not, and without the attribute a screen reader pronounces "Português" with the phonetics of
// whatever the document language happens to be — the option that exists to rescue someone becomes the
// one they cannot recognise. WCAG 2.2, 3.1.2 Language of Parts.
const LANG_LABEL: Record<string, string> = { pt: 'Português', en: 'English', es: 'Español' };

const esc = (s: string): string => s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/"/g, '&quot;');

/** Wire the controls. Safe to call again: it rebuilds the options and re-reads the current values. */
export function mountSettings({ root, i18n, visual, viz }: SettingsDeps): void {
  const lang = root.querySelector<HTMLSelectElement>('#set-lang');
  if (lang) {
    lang.innerHTML = i18n
      .available()
      .map((c) => `<option value="${c}" lang="${i18n.bcp47(c)}">${esc(LANG_LABEL[c] ?? c)}</option>`)
      .join('');
    lang.value = i18n.locale();
    lang.onchange = () => { void i18n.setLocale(lang.value); };
  }

  const contrast = root.querySelector<HTMLSelectElement>('#set-contrast');
  if (contrast) {
    contrast.value = String(visual.level());
    contrast.onchange = () => {
      const level = Number(contrast.value) as ContrastLevel;
      visual.setLevel(level);
      store.set(store.KEYS.contrast, level);
    };
  }

  const vizSel = root.querySelector<HTMLSelectElement>('#set-viz');
  if (vizSel) {
    vizSel.innerHTML = VIZ_MODES.map((m) => `<option value="${m.key}">${esc(i18n.t(m.i18nKey))}</option>`).join('');
    vizSel.value = viz.mode();
    vizSel.onchange = () => viz.set(vizSel.value as VizKey);
  }
}
```

> **No `region` parameter.** It is tempting, because the settings look like they act on the game area,
> but nothing here touches it: `setLocale` writes `document.documentElement.lang` and re-runs `applyDom`
> over the whole document, and the `viz` instance already closed over the region when `boot` built it. A
> field that callers find natural to pass and nobody reads is the same defect as the unused `host`
> parameter the audit removed from `initViz` — and `noUnusedParameters` does not catch a destructured
> field, so only reading the body catches it.

- [ ] **Step 5: Wire it into `boot.ts`**

Task 13 left two marked places for this. Add the import:
```ts
import { mountSettings } from './settings.js';
```
Call it once after `mountSettings` is available, replacing the `// Task 20 mounts...` comment:
```ts
  mountSettings({ root: document, i18n, visual, viz });
```
And re-mount on a language change, so the language select does not keep showing the previous choice —
inside the existing `i18n.onChange` handler, before `session?.restart()`:
```ts
    mountSettings({ root: document, i18n, visual, viz });
```

- [ ] **Step 6: Run the test, the full suite and the checks**

Run: `npx vitest run --project browser engine/shell/settings.browser.test.ts`
Expected: PASS, 13 tests.

Run: `npx tsc --noEmit && npx vitest run && npm run lint:deps && npm run format:check`
Expected: all clean.

- [ ] **Step 7: Verify by hand, with the keyboard only**

Run: `npm run build && npm run preview`, open `http://localhost:4173/play.html#arcade-classico/breakout`.

Using **no mouse at all**:
- `Tab` to the game region, `Escape` to pause.
- `Tab` to the contrast select, choose 7:1 — the bricks, paddle and ball must all change and stay
  distinguishable from one another.
- Choose a colour-vision correction — the whole field must shift.
- Choose English — the pause labels and the game's own announcements must both change.
- `Escape` to resume; the game continues with the new settings.

- [ ] **Step 8: Re-run the accessibility gate**

Run the gate as in Task 19. A `<select>` without an associated label is the likeliest new violation; if
it appears, fix the `for`/`id` pairing rather than excluding the element.

- [ ] **Step 9: Commit**

```bash
git add play.html engine/shell/shell.css engine/shell/settings.ts engine/shell/settings.browser.test.ts engine/shell/boot.ts
git commit -m "feat: expose language, contrast and colour-vision controls

Tasks 4, 8 and 9 built these and left them reachable only from a console, which
a person who needs high contrast cannot open. Plain selects, inside the pause
dialog that already has the focus contract. Language names are written in their
own language so nobody is trapped in a language they cannot read."
```

---

## Definition of done for Phase 1

Phase 1 is finished when all of the following are true, verified by running them rather than by reading
the checkboxes:

- [ ] `npm run validate` passes: formatting, typecheck, the D15 dependency boundary, every test, and the build.
- [ ] `npm run test:a11y` passes against the built preview.
- [ ] The catalog at `/` shows 35 categories and 383 items, with exactly three live links.
- [ ] Snake, Pong and Breakout are each playable from the keyboard alone, with no mouse.
- [ ] Contrast, language and colour-vision mode can each be changed **with the keyboard alone**, from
      the pause dialog, without a console.
- [ ] Raising the contrast to 7:1 visibly repaints all three games, and every role stays distinguishable.
- [ ] Switching the language changes the shell and the game announcements together.
- [ ] After playing a game once, it still loads with the network offline.
- [ ] A game that throws stops once, says so through the live region, and leaves the pause menu usable —
      check by temporarily making Snake's `update` throw, then reverting.
- [ ] No engine module holds mutable state at module scope, and no test needs a cleanup hook to undo the
      previous one. `grep -rnE "^(export )?let " engine/` returns nothing outside a factory body.
- [ ] `CLAUDE.md` is rewritten to describe the structure that now exists, replacing the Phase 0 text that
      says there is no build command.
