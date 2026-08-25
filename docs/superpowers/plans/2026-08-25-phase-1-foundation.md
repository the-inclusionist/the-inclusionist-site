# Phase 1 Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the engine, accessible shell and catalog generator that all 383 games inherit, proven by three reference games (Snake, Pong, Breakout).

**Architecture:** A single Vite app with one shell page (`play.html`) that lazy-loads any game through `import.meta.glob`, and a generated catalog page (`index.html`) that links to it. Games are thin: they declare sprites with semantic role tags, a three-language string dictionary, and a frame-stepped `update(dt)`. Everything else — input, scaling, contrast, screen-reader announcements, pause, offline caching — lives in the engine and is written exactly once.

**Tech Stack:** Node 24, TypeScript 5.7, Vite 8, Vitest 4 (node + browser/Playwright projects), PixiJS 7.4.2, `vite-plugin-pwa` 1.3, `@axe-core/playwright`, `node-html-parser` (one-shot import only).

**Spec:** [docs/superpowers/specs/2026-08-25-inclusionist-demos-design.md](../specs/2026-08-25-inclusionist-demos-design.md)

## Global Constraints

Every task's requirements implicitly include this section.

- **Logical canvas is exactly `320×180`, `TILE = 16`.** Never fractional scaling — it blurs pixel art.
- **`dt` is counted in FRAMES, not seconds.** `1.0` means one 60 fps frame. Clamp at `2`.
- **Keyboard listeners attach to `#game-region`, never to `window`.**
- **English for every artifact**: code, comments, commit messages, docs. Exception: the catalog page's visible content (category names, subgenre names and hints) stays pt-BR.
- **Every source file starts with `// SPDX-License-Identifier: GPL-3.0-or-later`.**
- **No PNG ships inside a game.** All art is procedural, painted through `pixelCanvas`.
- **No absolute paths in versioned files.**
- **Three-language floor**: `pt`, `en`, `es`. `pt` is the fallback for every key.
- **Commits are atomic and frequent**, in English, with the trailer `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- **Node commands run on the Dev's machine.** Every task states the exact command and its expected output.
- Tracer source referenced as `<TRACER>` = the sibling checkout of `SP-the-inclusionist-tracer`.

## File Structure

| File | Responsibility |
|---|---|
| `engine/core/constants.ts` | `LOGICAL_W`, `LOGICAL_H`, `TILE`, `MAX_DT`. Leaf, zero deps. |
| `engine/platform/storage.ts` | Exception-proof `localStorage` wrapper + `KEYS` registry. Leaf. |
| `engine/ui/dom.ts` | `$`, `$$`, `toggleBtn`. Leaf, pure (touches no DOM at import). |
| `engine/core/rng.ts` | Seeded LCG: `reseed`, `rnd`, `randInt`, `shuffle`. Leaf, pure. |
| `engine/core/loop.ts` | `startLoop(ticker, frame, maxDt)` — clamped frame driver. Leaf. |
| `engine/core/collision.ts` | `aabb`, `sweptAabb`. Leaf, pure. |
| `engine/core/i18n.ts` | `t`, `scopedT`, `registerDict`, `setLocale`, `applyDom`, `bcp47`. Depends on `storage`. |
| `engine/core/a11y-sr.ts` | `srSay`, `srAlert` against the live regions. Depends on `dom`. |
| `engine/input/keyboard.ts` | Key schemes for `solo/p2/p3/p4`, persisted. Depends on `storage`. |
| `engine/input/state.ts` | `keys` set, pad state, `held`. Leaf. |
| `engine/input/latch.ts` | Edge detection: `pressed`, `released`. Leaf. |
| `engine/input/attach.ts` | Binds listeners to `#game-region`, polls gamepads, exposes `InputApi`. |
| `engine/render/canvas.ts` | `makeCanvas`, `tex`, `pixelCanvas`, `pixelTexture`, `pixDisc`. Depends on PixiJS. |
| `engine/render/viz-modes.ts` | The visual accessibility mode table. Leaf, pure data. |
| `engine/render/cvd-matrices.ts` | Colour-vision-deficiency matrices. Leaf, pure data. |
| `engine/render/high-contrast.ts` | Role → colour resolution at 3:1 / 4.5:1 / 7:1. Leaf, pure. |
| `engine/render/sprites.ts` | Role-tagged sprite factory + repaint registry. Depends on `canvas`, `high-contrast`. |
| `engine/shell/a11y-regions.ts` | Mounts and drives `#sr-status` / `#sr-alert`. |
| `engine/shell/router.ts` | Hash → `{category, slug}`, and the game registry from `import.meta.glob`. |
| `engine/shell/hud.ts` | Score/lives strip, `aria-live="off"`. |
| `engine/shell/pause.ts` | `role="dialog" aria-modal` pause menu, focus trap. |
| `engine/shell/boot.ts` | Composition root: builds `GameContext`, loads a game, runs the loop. |
| `play.html` | The accessible shell markup. One page for all games. |
| `games/<cat>/<slug>/main.ts` | One game: `meta`, `setup`, `update`, `teardown`. |
| `games/<cat>/<slug>/strings.ts` | One game's `pt`/`en`/`es` dictionary. |
| `scripts/import-catalog.mts` | One-shot: `minigames-catalog-v2.html` → `data/catalog.json`. |
| `scripts/build-catalog.mts` | `catalog.json` + template → `index.html`. |
| `scripts/axe-check.mjs` | WCAG A/AA gate over the built shell and catalog. |

### Shared interfaces

These names are used across tasks. Define them exactly as written.

```ts
// engine/core/collision.ts
export interface Box { x: number; y: number; w: number; h: number }
export interface SweptHit { t: number; nx: number; ny: number }

// engine/core/i18n.ts
export type LocaleDict = Record<string, string>;
export type GameStrings = { pt: LocaleDict; en: LocaleDict; es: LocaleDict };
export type Translate = (key: string, params?: Record<string, string | number>) => string;

// engine/input/state.ts
export type Action = 'up' | 'left' | 'down' | 'right' | 'run' | 'jump' | 'swap' | 'especial';

// engine/input/attach.ts
export interface InputApi {
  held(pl: number, act: Action): boolean;
  pressed(pl: number, act: Action): boolean;
  readonly players: number;
}

// engine/render/high-contrast.ts
export type SpriteRole = 'player' | 'ally' | 'hazard' | 'goal' | 'pickup' | 'bg' | 'ui' | 'neutral';
export type ContrastLevel = 0 | 3 | 4.5 | 7;   // 0 = off

// engine/render/sprites.ts
export interface SpriteSpec {
  role: SpriteRole;
  w: number; h: number;
  paint: (px: (x: number, y: number, w: number, h: number, col: string) => void) => void;
}

// engine/shell/boot.ts — what every game receives
export interface GameContext {
  stage: import('pixi.js').Container;
  input: InputApi;
  sprites: { make(spec: SpriteSpec): import('pixi.js').Sprite };
  audio: { beep(freq: number, ms: number): void };
  rng: { rnd(): number; randInt(lo: number, hi: number): number; reseed(s: number): void };
  storage: { get(k: string, f?: string | null): string | null; set(k: string, v: string | number | boolean): boolean };
  t: Translate;
  srSay(text: string): void;
  srAlert(text: string): void;
  onGameOver(score: number): void;
}

// The game contract
export interface GameMeta {
  slug: string; title: string; category: string;
  density: 'leve' | 'medio' | 'denso';
  players: 1 | 2 | 3 | 4;
  renderer?: 'pixel' | 'svg' | '3d';
  rendererWhy?: string;
}
export interface GameModule {
  meta: GameMeta;
  strings: GameStrings;
  setup(ctx: GameContext): void;
  update(dt: number): void;
  teardown(): void;
}
```

---

### Task 1: Toolchain and engine constants

**Files:**
- Create: `package.json`, `tsconfig.json`, `vite.config.ts`, `.node-version`
- Create: `engine/core/constants.ts`
- Test: `engine/core/constants.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `LOGICAL_W: 320`, `LOGICAL_H: 180`, `TILE: 16`, `MAX_DT: 2` from `engine/core/constants.ts`. The npm scripts `dev`, `build`, `preview`, `typecheck`, `test`, `test:node`, `test:browser`.

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
    "validate": "npm run typecheck && vitest run && vite build"
  },
  "devDependencies": {
    "@axe-core/playwright": "^4.10.0",
    "@types/node": "^22.0.0",
    "@vitest/browser": "^4.0.0",
    "@vitest/browser-playwright": "^4.0.0",
    "node-html-parser": "^6.1.13",
    "pixi.js": "7.4.2",
    "playwright": "^1.49.0",
    "typescript": "^5.7.0",
    "vite": "^8.0.0",
    "vite-plugin-pwa": "^1.3.0",
    "vitest": "^4.0.0"
  },
  "overrides": { "vite-plugin-pwa": { "vite": "$vite" } }
}
```

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

`vite.config.ts` (PWA is added in Task 16 — do not add it now):
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

> The `input` map names `index.html` and `play.html`, which do not exist yet. Create both as
> one-line placeholders now so `vite build` resolves — Task 9 replaces `play.html` and Task 15
> generates `index.html`.
>
> `index.html`: `<!doctype html><title>catalog placeholder</title>`
> `play.html`: `<!doctype html><title>play placeholder</title>`

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
// 320x180 is exactly 16:9, which is why it integer-scales onto 1280x720 (x4) and 1920x1080 (x6)
// with no letterboxing and no fractional pixels. Any other logical size reintroduces blur on the
// most common displays, so this pair is not a preference.
export const LOGICAL_W = 320;
export const LOGICAL_H = 180;
export const TILE = 16;

// Frame-count clamp for the game loop. dt is measured in FRAMES (1.0 = one 60 fps frame), so a
// backgrounded tab returning after two seconds must not deliver dt = 120 and teleport everything
// through walls. Two frames is the tracer's value.
export const MAX_DT = 2;
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `npx vitest run --project node engine/core/constants.test.ts`
Expected: PASS, 4 tests.

- [ ] **Step 7: Run typecheck**

Run: `npx tsc --noEmit`
Expected: no output, exit 0.

- [ ] **Step 8: Commit**

```bash
git add package.json package-lock.json tsconfig.json vite.config.ts .node-version index.html play.html engine/core/constants.ts engine/core/constants.test.ts
git commit -m "feat: set up the toolchain and lock the logical canvas at 320x180"
```

---

### Task 2: Leaf modules lifted from the tracer

**Files:**
- Create: `engine/platform/storage.ts`, `engine/ui/dom.ts`, `engine/core/rng.ts`, `engine/core/loop.ts`
- Test: `engine/core/rng.test.ts`, `engine/core/loop.test.ts`, `engine/platform/storage.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `storage`: `get(key, fallback?)`, `set(key, value)`, `remove(key)`, `getBool`, `setBool`, `getNum`, `getJSON<T>`, `setJSON`, `KEYS`.
  - `dom`: `$<T>(sel)`, `$$<T>(sel)`, `toggleBtn(el, on)`.
  - `rng`: `reseed(s)`, `rnd()`, `randInt(lo, hi)`, `shuffle<T>(arr)`.
  - `loop`: `startLoop(ticker, frame, maxDt?)` where `ticker` is `{ add(fn): void; deltaTime: number }`.

- [ ] **Step 1: Copy the four files from the tracer**

Copy verbatim, then apply exactly these changes:

| From `<TRACER>` | To | Changes |
|---|---|---|
| `app/js/platform/storage.ts` | `engine/platform/storage.ts` | Translate the header comment to English. Replace the whole `KEYS` object with the one below — the tracer's keys are its own game's. |
| `app/js/ui/dom.ts` | `engine/ui/dom.ts` | Translate the header comment to English. Keep `$`, `$$`, `toggleBtn` unchanged. |
| `app/js/core/rng.ts` | `engine/core/rng.ts` | Translate the header comment to English. Keep the LCG constants unchanged. |
| `app/js/core/loop.ts` | `engine/core/loop.ts` | Translate the header comment to English. Import `MAX_DT` from `constants.js` and use it as the default instead of the literal `2`. |

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
import { beforeEach, describe, expect, it } from 'vitest';
import { randInt, reseed, rnd, shuffle } from './rng.js';

describe('rng', () => {
  beforeEach(() => reseed(1));

  it('is deterministic: the same seed replays the same sequence', () => {
    const first = [rnd(), rnd(), rnd()];
    reseed(1);
    expect([rnd(), rnd(), rnd()]).toEqual(first);
  });

  it('diverges on a different seed', () => {
    const a = [rnd(), rnd(), rnd()];
    reseed(2);
    expect([rnd(), rnd(), rnd()]).not.toEqual(a);
  });

  it('stays inside [0, 1)', () => {
    for (let i = 0; i < 1000; i++) {
      const v = rnd();
      expect(v).toBeGreaterThanOrEqual(0);
      expect(v).toBeLessThan(1);
    }
  });

  it('randInt covers both endpoints and never exceeds them', () => {
    const seen = new Set<number>();
    for (let i = 0; i < 1000; i++) {
      const v = randInt(3, 6);
      expect(Number.isInteger(v)).toBe(true);
      expect(v).toBeGreaterThanOrEqual(3);
      expect(v).toBeLessThanOrEqual(6);
      seen.add(v);
    }
    expect([...seen].sort()).toEqual([3, 4, 5, 6]);
  });

  it('randInt with lo === hi returns that value', () => {
    expect(randInt(7, 7)).toBe(7);
  });

  it('shuffle returns a permutation and leaves the input alone', () => {
    const input = Object.freeze([1, 2, 3, 4, 5]);
    const out = shuffle(input);
    expect(out).not.toBe(input);
    expect([...out].sort((a, b) => a - b)).toEqual([1, 2, 3, 4, 5]);
  });

  it('shuffle of an empty array is an empty array', () => {
    expect(shuffle([])).toEqual([]);
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
function installStorage(): Map<string, string> {
  const map = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => (map.has(k) ? map.get(k)! : null),
    setItem: (k: string, v: string) => { map.set(k, v); },
    removeItem: (k: string) => { map.delete(k); },
  });
  return map;
}

describe('storage', () => {
  beforeEach(() => { installStorage(); });

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

- [ ] **Step 4: Create the four files as described in Step 1**

- [ ] **Step 5: Run the tests to verify they pass**

Run: `npx vitest run --project node engine/`
Expected: PASS. `constants` 4, `rng` 7, `loop` 4, `storage` 6.

- [ ] **Step 6: Run typecheck**

Run: `npx tsc --noEmit`
Expected: no output, exit 0.

- [ ] **Step 7: Commit**

```bash
git add engine/platform/storage.ts engine/platform/storage.test.ts engine/ui/dom.ts engine/core/rng.ts engine/core/rng.test.ts engine/core/loop.ts engine/core/loop.test.ts
git commit -m "feat: lift the leaf modules from the tracer

storage, dom, rng and loop cross over unchanged apart from English headers.
The KEYS registry is replaced with this project's own, namespaced demos.,
and startLoop now defaults to MAX_DT instead of a literal."
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
- Create: `engine/core/i18n.ts`
- Create: `engine/i18n/pt.ts`, `engine/i18n/en.ts`, `engine/i18n/es.ts` (shell strings only)
- Test: `engine/core/i18n.test.ts`

**Interfaces:**
- Consumes: `storage` (`KEYS.lang`, `get`, `set`) from Task 2.
- Produces:
  - `t(key, params?): string` — global/shell keys.
  - `scopedT(ns: string): Translate` — a translate function pre-namespaced to one game.
  - `registerDict(ns: string, s: GameStrings): void` — merges a game's three dictionaries under `ns`.
  - `setLocale(code: string): Promise<void>`, `getLocale(): string`, `bcp47(code?): string`, `availableLocales(): string[]`, `applyDom(root?): void`, `initI18n(): void`.
  - The `i18n:change` `CustomEvent` on `window`, carrying `{ locale }`.

> The tracer's version is the starting point (`<TRACER>/app/js/core/i18n.ts`, 90 lines). Two things change,
> both forced by having 383 games instead of one: dictionaries arrive at runtime with a game's chunk rather
> than being three static files, and lookups are namespaced so `snake.gameOver` cannot collide with
> `pong.gameOver`. Everything else — the chained fallback, `{param}` interpolation, `applyDom`, `bcp47` —
> is copied.

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
};
```

- [ ] **Step 2: Write the failing test**

`engine/core/i18n.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { availableLocales, bcp47, getLocale, registerDict, scopedT, setLocale, t } from './i18n.js';

function installStorage(): void {
  const map = new Map<string, string>();
  vi.stubGlobal('localStorage', {
    getItem: (k: string) => (map.has(k) ? map.get(k)! : null),
    setItem: (k: string, v: string) => { map.set(k, v); },
    removeItem: (k: string) => { map.delete(k); },
  });
}

const snakeStrings = {
  pt: { gameOver: 'Fim de jogo', ate: 'Comeu {n} maçãs' },
  en: { gameOver: 'Game over', ate: 'Ate {n} apples' },
  es: { gameOver: 'Fin del juego', ate: 'Comió {n} manzanas' },
};

describe('i18n', () => {
  beforeEach(async () => {
    installStorage();
    vi.stubGlobal('document', { documentElement: { lang: '' } });
    vi.stubGlobal('window', { dispatchEvent: () => true });
    await setLocale('pt');
  });

  it('offers exactly the three floor languages', () => {
    expect(availableLocales().sort()).toEqual(['en', 'es', 'pt']);
  });

  it('translates a shell key', () => {
    expect(t('shell.resume')).toBe('Continuar');
  });

  it('returns the key itself when nothing matches, so a miss is visible rather than blank', () => {
    expect(t('shell.doesNotExist')).toBe('shell.doesNotExist');
  });

  it('interpolates {param}, including repeats', () => {
    registerDict('demo', { pt: { hi: '{a} e {a} e {b}' }, en: { hi: '' }, es: { hi: '' } });
    expect(scopedT('demo')('hi', { a: 'x', b: 2 })).toBe('x e x e 2');
  });

  it('namespaces a game dictionary so two games can share a key name', () => {
    registerDict('snake', snakeStrings);
    registerDict('pong', { pt: { gameOver: 'Acabou' }, en: { gameOver: 'Done' }, es: { gameOver: 'Listo' } });
    expect(scopedT('snake')('gameOver')).toBe('Fim de jogo');
    expect(scopedT('pong')('gameOver')).toBe('Acabou');
  });

  it('switches every registered namespace when the locale changes', async () => {
    registerDict('snake', snakeStrings);
    await setLocale('en');
    expect(getLocale()).toBe('en');
    expect(t('shell.resume')).toBe('Resume');
    expect(scopedT('snake')('ate', { n: 3 })).toBe('Ate 3 apples');
  });

  it('falls back to pt for a key the active locale is missing', async () => {
    registerDict('half', { pt: { only: 'só em pt' }, en: {}, es: {} });
    await setLocale('en');
    expect(scopedT('half')('only')).toBe('só em pt');
  });

  it('rejects an unknown locale by falling back to pt', async () => {
    await setLocale('de');
    expect(getLocale()).toBe('pt');
  });

  it('tags pt with a region and leaves en and es without one', () => {
    expect(bcp47('pt')).toBe('pt-BR');
    expect(bcp47('en')).toBe('en');
    expect(bcp47('es')).toBe('es');
  });

  it('registering the same namespace twice replaces rather than merges', () => {
    registerDict('twice', { pt: { k: 'first' }, en: {}, es: {} });
    registerDict('twice', { pt: { k: 'second' }, en: {}, es: {} });
    expect(scopedT('twice')('k')).toBe('second');
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
// Adapted from the tracer's core/i18n.ts. Two deliberate differences, both forced by 383 games:
//
//   1. A game's strings are NOT in the locale files. Each game exports its own `strings` and calls
//      registerDict(slug, strings) when its chunk loads, so a visitor downloads only the text of the
//      games they actually open.
//   2. Lookups are namespaced. `scopedT('snake')('gameOver')` reads `snake.gameOver`, so two games
//      may use the same key name without one silently winning.
//
// pt is imported statically so the shell has text before any await resolves; en and es load on demand.
import pt from '../i18n/pt.js';
import * as store from '../platform/storage.js';

export type LocaleDict = Record<string, string>;
export type GameStrings = { pt: LocaleDict; en: LocaleDict; es: LocaleDict };
export type Translate = (key: string, params?: Record<string, string | number>) => string;

const AVAILABLE = ['pt', 'en', 'es'] as const;
type LocaleCode = (typeof AVAILABLE)[number];

const shellLoaders = import.meta.glob<{ default: LocaleDict }>('../i18n/*.ts');

// Shell dictionaries already loaded. pt is present from the start.
const shell: Record<string, LocaleDict> = { pt };
// Game dictionaries by namespace, all three locales kept so a locale switch needs no reload.
const games = new Map<string, GameStrings>();

let locale: LocaleCode = 'pt';

/** Look a key up in one dictionary set, chained locale → pt → null. */
function lookup(dicts: { pt: LocaleDict; [k: string]: LocaleDict | undefined }, key: string): string | null {
  const active = dicts[locale];
  if (active && key in active) return active[key]!;
  if (key in dicts.pt) return dicts.pt[key]!;
  return null;
}

function interpolate(s: string, params?: Record<string, string | number>): string {
  if (!params) return s;
  let out = s;
  for (const k in params) out = out.replaceAll('{' + k + '}', String(params[k]));
  return out;
}

/** Translate a shell key. Unknown keys return the key itself, so a miss is visible, not blank. */
export function t(key: string, params?: Record<string, string | number>): string {
  const hit = lookup({ pt, ...shell }, key);
  return interpolate(hit ?? key, params);
}

/** Register one game's three dictionaries under a namespace. Re-registering replaces. */
export function registerDict(ns: string, s: GameStrings): void {
  games.set(ns, s);
}

/** A translate function bound to one namespace, so the game never writes its own prefix. */
export function scopedT(ns: string): Translate {
  return (key, params) => {
    const s = games.get(ns);
    if (!s) return interpolate(key, params);
    return interpolate(lookup(s, key) ?? key, params);
  };
}

export function getLocale(): string { return locale; }
export function availableLocales(): string[] { return [...AVAILABLE]; }

/**
 * The BCP-47 tag for `<html lang>` and for browser APIs that speak or compare language.
 * Only Portuguese needs a region: bare 'pt' lets the browser pick between pt-PT and pt-BR, and the
 * prosody difference is audible. English and Spanish stay region-free on purpose — pinning 'en-US'
 * would impose an American accent on a reader in India or Nigeria.
 */
export function bcp47(code: string = locale): string { return code === 'pt' ? 'pt-BR' : code; }

/** Apply declarative translations: [data-i18n] → textContent, [data-i18n-aria] → aria-label. */
export function applyDom(root: ParentNode = document): void {
  root.querySelectorAll('[data-i18n]').forEach((el) => {
    const k = el.getAttribute('data-i18n');
    if (k) el.textContent = t(k);
  });
  root.querySelectorAll('[data-i18n-aria]').forEach((el) => {
    const k = el.getAttribute('data-i18n-aria');
    if (k) el.setAttribute('aria-label', t(k));
  });
}

async function ensureShell(code: LocaleCode): Promise<void> {
  if (shell[code]) return;
  const load = shellLoaders[`../i18n/${code}.ts`];
  if (!load) return;
  shell[code] = (await load()).default;
}

/**
 * Switch language: load the shell dictionary if needed, persist, update <html lang>, re-apply the DOM,
 * and announce. Game dictionaries need no loading — all three locales shipped with the game's chunk.
 */
export async function setLocale(code: string): Promise<void> {
  const next = (AVAILABLE as readonly string[]).includes(code) ? (code as LocaleCode) : 'pt';
  await ensureShell(next);
  locale = next;
  store.set(store.KEYS.lang, next);
  document.documentElement.lang = bcp47(next);
  if (typeof document.querySelectorAll === 'function') applyDom(document);
  window.dispatchEvent(new CustomEvent('i18n:change', { detail: { locale } }));
}

/** Boot: pt is already applied synchronously; switch asynchronously if another language is preferred. */
export function initI18n(): void {
  const saved = store.get(store.KEYS.lang, null);
  const nav = (globalThis.navigator?.language ?? 'pt').slice(0, 2).toLowerCase();
  const want = saved && (AVAILABLE as readonly string[]).includes(saved) ? saved : nav;
  if ((AVAILABLE as readonly string[]).includes(want) && want !== 'pt') void setLocale(want);
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `npx vitest run --project node engine/core/i18n.test.ts`
Expected: PASS, 10 tests.

- [ ] **Step 6: Run typecheck**

Run: `npx tsc --noEmit`
Expected: no output, exit 0.

- [ ] **Step 7: Commit**

```bash
git add engine/core/i18n.ts engine/core/i18n.test.ts engine/i18n/
git commit -m "feat: add i18n with per-namespace game dictionaries

Adapted from the tracer. A game ships its own three locales with its chunk
and registers them under its slug, so no visitor downloads 383 games' text
and two games may reuse a key name."
```

---

### Task 5: Screen-reader announcements

**Files:**
- Create: `engine/core/a11y-sr.ts`
- Test: `engine/core/a11y-sr.browser.test.ts`

**Interfaces:**
- Consumes: `$` from `engine/ui/dom.ts` (Task 2).
- Produces: `srSay(text: string): void` (polite) and `srAlert(text: string): void` (assertive), both writing into `#sr-status` / `#sr-alert`, which Task 9 puts in `play.html`.

> This is a browser test, not a node one: the clear → `requestAnimationFrame` → write pattern is the
> whole point of the module, and a fake timer would test the fake rather than the behaviour.

- [ ] **Step 1: Write the failing test**

`engine/core/a11y-sr.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it } from 'vitest';
import { srAlert, srSay } from './a11y-sr.js';

const nextFrame = (): Promise<void> => new Promise((r) => requestAnimationFrame(() => r()));

describe('screen-reader regions', () => {
  beforeEach(() => {
    document.body.innerHTML =
      '<div id="sr-status" role="status" aria-live="polite" aria-atomic="true"></div>' +
      '<div id="sr-alert" role="alert" aria-live="assertive" aria-atomic="true"></div>';
  });

  it('writes into the polite region', async () => {
    srSay('ten points');
    await nextFrame();
    expect(document.querySelector('#sr-status')!.textContent).toBe('ten points');
  });

  it('writes into the assertive region', async () => {
    srAlert('game over');
    await nextFrame();
    expect(document.querySelector('#sr-alert')!.textContent).toBe('game over');
  });

  it('clears before writing, so the same text is announced twice', async () => {
    srSay('same');
    await nextFrame();
    srSay('same');
    expect(document.querySelector('#sr-status')!.textContent).toBe('');
    await nextFrame();
    expect(document.querySelector('#sr-status')!.textContent).toBe('same');
  });

  it('keeps the two regions independent', async () => {
    srSay('polite');
    srAlert('urgent');
    await nextFrame();
    expect(document.querySelector('#sr-status')!.textContent).toBe('polite');
    expect(document.querySelector('#sr-alert')!.textContent).toBe('urgent');
  });

  it('does not throw when the regions are absent', () => {
    document.body.innerHTML = '';
    expect(() => srSay('nobody listening')).not.toThrow();
    expect(() => srAlert('nobody listening')).not.toThrow();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run --project browser engine/core/a11y-sr.browser.test.ts`
Expected: FAIL — `Failed to resolve import "./a11y-sr.js"`.

- [ ] **Step 3: Write the implementation**

`engine/core/a11y-sr.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// core/a11y-sr — announcements for screen readers. Adapted from the tracer's core/a11y-sr.ts, minus
// its Libras injection hook, which belongs to that project.
//
// srSay writes into the aria-live="polite" region (status: waits its turn); srAlert writes into the
// aria-live="assertive" one (interrupts). The clear → requestAnimationFrame → write dance is not
// ceremony: writing the same string twice in a row is a no-op to a screen reader, so scoring ten
// points twice would announce once. Clearing first forces the re-announcement.
//
// The regions themselves live in play.html, so a game never has to create them.
import { $ } from '../ui/dom.js';

function announce(sel: string, text: string): void {
  const el = $(sel);
  if (!el) return;
  el.textContent = '';
  requestAnimationFrame(() => { el.textContent = text; });
}

/** Polite announcement: does not interrupt what the reader is currently saying. */
export const srSay = (text: string): void => announce('#sr-status', text);

/** Assertive announcement: interrupts. Reserve it for what the player must hear now. */
export const srAlert = (text: string): void => announce('#sr-alert', text);
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project browser engine/core/a11y-sr.browser.test.ts`
Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

```bash
git add engine/core/a11y-sr.ts engine/core/a11y-sr.browser.test.ts
git commit -m "feat: add screen-reader announcement helpers

Clear-then-write on the next frame, so repeating the same announcement is
actually announced twice instead of silently swallowed."
```

---

### Task 6: Input

**Files:**
- Create: `engine/input/state.ts`, `engine/input/keyboard.ts`, `engine/input/latch.ts`, `engine/input/attach.ts`
- Test: `engine/input/state.test.ts`, `engine/input/latch.test.ts`, `engine/input/attach.browser.test.ts`

**Interfaces:**
- Consumes: `storage` from Task 2.
- Produces:
  - `ACTIONS: readonly Action[]` and `type Action` from `state.ts`.
  - `keys: Set<string>`, `held(scheme, act, padIndex)` from `state.ts`.
  - `KB_DEFAULTS`, `loadKB()`, `saveKB(kb)`, `resetKB()`, `kb`, `initKB()` from `keyboard.ts`.
  - `attachInput(el: HTMLElement, players: number): InputApi & { poll(): void; detach(): void }` from `attach.ts`.

> The eight actions and the four schemes come from the tracer verbatim, because a player who knows one
> game in the constellation should not have to relearn the keys for the next. The gamepad dead zone of
> `0.5` is also the tracer's, chosen for ergonomics rather than sensitivity.

- [ ] **Step 1: Write the failing tests**

`engine/input/state.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it } from 'vitest';
import { ACTIONS, held, keys, padCur, PAD_DEAD, type KeyScheme } from './state.js';

const scheme: KeyScheme = {
  left: ['KeyA'], right: ['KeyD'], up: ['KeyW'], down: ['KeyS'],
  run: ['KeyU'], jump: ['KeyJ', 'Space'], swap: ['KeyI'], especial: ['KeyK'],
};

describe('input state', () => {
  beforeEach(() => { keys.clear(); for (const k of Object.keys(padCur)) delete padCur[Number(k)]; });

  it('names exactly the eight actions of the constellation', () => {
    expect([...ACTIONS]).toEqual(['up', 'left', 'down', 'right', 'run', 'jump', 'swap', 'especial']);
  });

  it('reports a held key', () => {
    keys.add('KeyA');
    expect(held(scheme, 'left', -1)).toBe(true);
    expect(held(scheme, 'right', -1)).toBe(false);
  });

  it('accepts any of the alternate keys bound to one action', () => {
    keys.add('Space');
    expect(held(scheme, 'jump', -1)).toBe(true);
    keys.clear();
    keys.add('KeyJ');
    expect(held(scheme, 'jump', -1)).toBe(true);
  });

  it('reports a gamepad action when a pad is associated', () => {
    padCur[0] = { jump: true };
    expect(held(scheme, 'jump', 0)).toBe(true);
  });

  it('ignores gamepad state when no pad is associated', () => {
    padCur[0] = { jump: true };
    expect(held(scheme, 'jump', -1)).toBe(false);
  });

  it('keeps the dead zone at half the stick travel', () => {
    expect(PAD_DEAD).toBe(0.5);
  });
});
```

`engine/input/latch.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import { makeLatch } from './latch.js';

describe('latch', () => {
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
});
```

`engine/input/attach.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { afterEach, beforeEach, describe, expect, it } from 'vitest';
import { attachInput } from './attach.js';

let region: HTMLElement;
let api: ReturnType<typeof attachInput>;

const key = (type: 'keydown' | 'keyup', code: string): KeyboardEvent =>
  new KeyboardEvent(type, { code, bubbles: true, cancelable: true });

describe('attachInput', () => {
  beforeEach(() => {
    document.body.innerHTML = '<div id="game-region" tabindex="0"></div><input id="outside" />';
    region = document.querySelector('#game-region')!;
    api = attachInput(region, 1);
  });
  afterEach(() => api.detach());

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

  it('prevents the default action for keys it owns, so Space does not scroll the page', () => {
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
    api = attachInput(region, 2);
    region.dispatchEvent(key('keydown', 'KeyA'));
    region.dispatchEvent(key('keydown', 'ArrowLeft'));
    expect(api.held(0, 'left')).toBe(true);
    expect(api.held(1, 'left')).toBe(true);
    region.dispatchEvent(key('keyup', 'KeyA'));
    expect(api.held(0, 'left')).toBe(false);
    expect(api.held(1, 'left')).toBe(true);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run engine/input/`
Expected: FAIL — unresolved imports for `state.js`, `latch.js`, `attach.js`.

- [ ] **Step 3: Write `engine/input/state.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/state — runtime input state plus the generic query. Leaf, zero deps.
// The eight actions are the tracer's, unchanged: someone who learned one game in the constellation
// should not have to relearn the keys for the next.
export type Action = 'up' | 'left' | 'down' | 'right' | 'run' | 'jump' | 'swap' | 'especial';
export const ACTIONS: readonly Action[] = ['up', 'left', 'down', 'right', 'run', 'jump', 'swap', 'especial'];

/** One player's binding: action → the physical KeyboardEvent.code values that trigger it. */
export type KeyScheme = Record<Action, string[]>;

/** Physical keys held right now, by KeyboardEvent.code. Mutated by the keydown/keyup handlers. */
export const keys = new Set<string>();

/** Gamepad actions held this frame, per pad index. Mutated in place by the poll. */
export type PadState = Partial<Record<Action, boolean>>;
export const padCur: Record<number, PadState> = {};

/** Dead zone = the first HALF of the stick's travel. Ergonomics, not sensitivity (tracer's value). */
export const PAD_DEAD = 0.5;

/** Is this action held, by keyboard or by the associated pad? `pad = -1` means no pad. */
export function held(scheme: KeyScheme, act: Action, pad: number): boolean {
  if (scheme[act].some((k) => keys.has(k))) return true;
  return pad >= 0 && padCur[pad]?.[act] === true;
}
```

- [ ] **Step 4: Write `engine/input/latch.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/latch — edge detection over a polled boolean. Leaf, zero deps.
//
// A held key is true on every frame. A menu that advances on "jump" would advance sixty times a
// second without this. `pressed` fires on the false → true edge, `released` on true → false.
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

- [ ] **Step 5: Write `engine/input/keyboard.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/keyboard — key schemes per player count, plus persistence. Depends only on storage.
// Copied from the tracer's defaults so muscle memory carries across the constellation. No Alt,
// AltGr, Ctrl or Shift anywhere: those collide with the browser and with assistive technology.
import * as store from '../platform/storage.js';
import type { Action, KeyScheme } from './state.js';

export type KBDefaults = { solo: KeyScheme; p2: KeyScheme[]; p3: KeyScheme[]; p4: KeyScheme[] };

const SCHEMES4: KeyScheme[] = [
  { left: ['KeyA'], right: ['KeyD'], up: ['KeyW'], down: ['KeyS'], run: ['KeyZ'], jump: ['KeyX'], swap: ['KeyC'], especial: ['KeyV'] },
  { left: ['KeyJ'], right: ['KeyL'], up: ['KeyI'], down: ['KeyK'], run: ['KeyM'], jump: ['Comma'], swap: ['Period'], especial: ['Semicolon', 'Slash'] },
  { left: ['ArrowLeft'], right: ['ArrowRight'], up: ['ArrowUp'], down: ['ArrowDown'], run: ['Home'], jump: ['End'], swap: ['PageUp'], especial: ['PageDown'] },
  { left: ['Numpad4'], right: ['Numpad6'], up: ['Numpad8'], down: ['Numpad5'], run: ['Numpad2'], jump: ['Numpad0'], swap: ['Numpad3'], especial: ['NumpadDecimal'] },
];

export const KB_DEFAULTS: KBDefaults = {
  solo: {
    left: ['KeyA', 'ArrowLeft'], right: ['KeyD', 'ArrowRight'], up: ['KeyW', 'ArrowUp'], down: ['KeyS', 'ArrowDown'],
    run: ['KeyU'], jump: ['KeyJ', 'Space'], swap: ['KeyI'], especial: ['KeyK'],
  },
  p2: [
    { left: ['KeyA'], right: ['KeyD'], up: ['KeyW'], down: ['KeyS'], run: ['KeyU'], jump: ['KeyJ'], swap: ['KeyI'], especial: ['KeyK'] },
    { left: ['ArrowLeft'], right: ['ArrowRight'], up: ['ArrowUp'], down: ['ArrowDown'], run: ['Numpad8'], jump: ['Numpad5'], swap: ['Numpad9'], especial: ['Numpad6'] },
  ],
  p3: JSON.parse(JSON.stringify(SCHEMES4.slice(0, 3))) as KeyScheme[],
  p4: JSON.parse(JSON.stringify(SCHEMES4)) as KeyScheme[],
};

type SavedKB = Partial<KBDefaults>;

/** Load the saved schemes ON TOP of the defaults, so a new action added later is never missing. */
export function loadKB(): KBDefaults {
  const d = JSON.parse(JSON.stringify(KB_DEFAULTS)) as KBDefaults;
  const s = store.getJSON<SavedKB>(store.KEYS.kbcontrols, null);
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
export function resetKB(): KBDefaults { store.remove(store.KEYS.kbcontrols); return JSON.parse(JSON.stringify(KB_DEFAULTS)) as KBDefaults; }

// The live map. It is born with the defaults and does NOT read storage at import time: a test that
// imports anything from this file would otherwise inherit the environment's localStorage, and a key
// map leaking in from another case fails far from its cause.
export let kb: KBDefaults = JSON.parse(JSON.stringify(KB_DEFAULTS)) as KBDefaults;
export function initKB(): KBDefaults { kb = loadKB(); return kb; }
export function setKB(next: KBDefaults): void { kb = next; }

/** The schemes for a given player count. */
export function schemesFor(players: number): KeyScheme[] {
  if (players <= 1) return [kb.solo];
  if (players === 2) return kb.p2;
  if (players === 3) return kb.p3;
  return kb.p4;
}

/** Every physical code currently bound for this player count — what the handler may swallow. */
export function ownedCodes(players: number): Set<string> {
  const out = new Set<string>();
  for (const s of schemesFor(players)) for (const act of Object.keys(s) as Action[]) for (const c of s[act]) out.add(c);
  return out;
}
```

- [ ] **Step 6: Write `engine/input/attach.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// input/attach — binds the input state to a DOM element and to the gamepads.
//
// Listeners go on #game-region, NEVER on window. On window the game swallows keys belonging to the
// page: the skip link stops working, Space scrolls nothing, and a screen-reader user navigating the
// surrounding document finds their keys eaten by a canvas they are not focused on.
import { makeLatch } from './latch.js';
import { ownedCodes, schemesFor } from './keyboard.js';
import { ACTIONS, held, keys, padCur, PAD_DEAD, type Action } from './state.js';

export interface InputApi {
  held(pl: number, act: Action): boolean;
  pressed(pl: number, act: Action): boolean;
  readonly players: number;
}

export function attachInput(el: HTMLElement, players: number): InputApi & { poll(): void; detach(): void } {
  const schemes = schemesFor(players);
  const owned = ownedCodes(players);
  const latch = makeLatch();

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

  /** Read the gamepads into padCur. Call once per frame, before the game's update. */
  function poll(): void {
    const pads = navigator.getGamepads?.() ?? [];
    for (let i = 0; i < pads.length; i++) {
      const p = pads[i];
      if (!p) { delete padCur[i]; continue; }
      const ax = p.axes[0] ?? 0, ay = p.axes[1] ?? 0;
      const b = (n: number): boolean => p.buttons[n]?.pressed === true;
      padCur[i] = {
        left: ax < -PAD_DEAD || b(14), right: ax > PAD_DEAD || b(15),
        up: ay < -PAD_DEAD || b(12), down: ay > PAD_DEAD || b(13),
        jump: b(0), run: b(2), swap: b(1), especial: b(3),
      };
    }
  }

  return {
    players,
    held: (pl, act) => held(schemes[Math.min(pl, schemes.length - 1)]!, act, pl),
    pressed: (pl, act) => latch.pressed(`${pl}:${act}`, held(schemes[Math.min(pl, schemes.length - 1)]!, act, pl)),
    poll,
    detach() {
      el.removeEventListener('keydown', onKeyDown);
      el.removeEventListener('keyup', onKeyUp);
      el.removeEventListener('blur', onBlur);
      keys.clear();
      latch.clear();
      for (const k of Object.keys(padCur)) delete padCur[Number(k)];
    },
  };
}

export { ACTIONS, type Action };
```

- [ ] **Step 7: Run the tests to verify they pass**

Run: `npx vitest run engine/input/`
Expected: PASS. `state` 6, `latch` 4, `attach` 8.

- [ ] **Step 8: Run typecheck**

Run: `npx tsc --noEmit`
Expected: no output, exit 0.

- [ ] **Step 9: Commit**

```bash
git add engine/input/
git commit -m "feat: add the eight-action input layer

Schemes and the 0.5 dead zone come from the tracer so muscle memory carries
across the constellation. Listeners bind to #game-region rather than window,
and only codes the current scheme owns get preventDefault."
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

### Task 8: High contrast by sprite role

**Files:**
- Create: `engine/render/high-contrast.ts`, `engine/render/sprites.ts`
- Test: `engine/render/high-contrast.test.ts`, `engine/render/sprites.browser.test.ts`

**Interfaces:**
- Consumes: `pixelCanvas`, `tex` from Task 7.
- Produces:
  - From `high-contrast.ts`: `type SpriteRole`, `type ContrastLevel`, `relativeLuminance(hex)`, `contrastRatio(a, b)`, `roleColor(role, level)`, `outlineColor(level)`, `HC_BG`, `getContrastLevel()`, `setContrastLevel(level)`, `onContrastChange(fn)`.
  - From `sprites.ts`: `roleCanvas(spec, level)`, `makeSpriteApi(stage)` returning `{ make(spec), repaintAll(), clear() }`.

> This is the task the whole per-game economy rests on. A game says what a thing *means*
> (`role: 'hazard'`) and never writes contrast code. When the player raises the contrast level, the
> engine repaints every registered sprite: the game's own `paint` still draws the silhouette, but every
> colour it asks for is replaced by the role's colour, and a one-pixel outline is added so adjacent
> roles never merge into one blob.
>
> The colours are not hand-picked hex values hoping to be contrasty. Each role has a hue, and the
> lightness is searched until the colour hits the exact WCAG luminance the level demands. The test then
> measures the real ratio — if a colour fails 4.5:1, the suite fails.

- [ ] **Step 1: Write the failing test**

`engine/render/high-contrast.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { describe, expect, it } from 'vitest';
import {
  contrastRatio, getContrastLevel, HC_BG, onContrastChange, outlineColor,
  relativeLuminance, roleColor, setContrastLevel, type ContrastLevel, type SpriteRole,
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
    const bg = roleColor('bg', 7)!;
    expect(contrastRatio(bg, HC_BG)).toBeLessThan(3);
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

  it('gives an outline that contrasts with the background at every level', () => {
    for (const level of LEVELS) {
      expect(contrastRatio(outlineColor(level), HC_BG)).toBeGreaterThanOrEqual(3);
    }
  });
});

describe('contrast level state', () => {
  it('starts off', () => {
    setContrastLevel(0);
    expect(getContrastLevel()).toBe(0);
  });

  it('notifies subscribers on change and not on a no-op set', () => {
    setContrastLevel(0);
    let calls = 0;
    const off = onContrastChange(() => calls++);
    setContrastLevel(4.5);
    expect(calls).toBe(1);
    setContrastLevel(4.5);
    expect(calls).toBe(1);
    off();
    setContrastLevel(7);
    expect(calls).toBe(1);
    setContrastLevel(0);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run --project node engine/render/high-contrast.test.ts`
Expected: FAIL — `Failed to resolve import "./high-contrast.js"`.

- [ ] **Step 3: Write `engine/render/high-contrast.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/high-contrast — semantic roles resolved to colours that MEET a stated WCAG ratio.
//
// This inverts what the tracer does. There, contrast was retrofitted onto a finished platformer, so
// the colour-blocking is welded to that game's four entities. Here every sprite is born through one
// factory, so it can carry a role tag from the start and no game ever writes contrast code.
//
// The colours are computed, not chosen. Each role owns a hue; the lightness is searched until the
// colour's relative luminance is exactly what the target ratio requires against the backdrop. That
// makes the promise checkable, and the test checks it.

export type SpriteRole = 'player' | 'ally' | 'hazard' | 'goal' | 'pickup' | 'bg' | 'ui' | 'neutral';
/** 0 = off (the game's own art). Otherwise the WCAG contrast ratio the palette must meet. */
export type ContrastLevel = 0 | 3 | 4.5 | 7;

/** The backdrop every ratio is measured against. High contrast forces the field to this colour. */
export const HC_BG = '#000000';

/** Hue and saturation per role. `ui` and `neutral` are greys, distinguished by lightness alone. */
const ROLE_HS: Record<Exclude<SpriteRole, 'bg'>, { h: number; s: number }> = {
  player: { h: 210, s: 1 },     // blue — the thing you are
  ally: { h: 150, s: 1 },       // green — safe
  hazard: { h: 0, s: 1 },       // red — kills you
  goal: { h: 45, s: 1 },        // amber — where you are going
  pickup: { h: 300, s: 1 },     // magenta — take it
  ui: { h: 0, s: 0 },           // white-ish — chrome, never gameplay
  neutral: { h: 180, s: 0.35 }, // desaturated cyan — scenery that still needs to be seen
};

/** The background role is RECESSED: it must not compete with anything the player must react to. */
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
 * The lightness at which this hue reaches the luminance the ratio demands.
 * Luminance rises monotonically with HSL lightness, so a bisection always converges — and at l = 1
 * every hue is white, whose luminance is 1, so no target below 1 can be out of reach.
 */
function solveLightness(h: number, s: number, targetLum: number): number {
  let lo = 0, hi = 1;
  for (let i = 0; i < 40; i++) {
    const mid = (lo + hi) / 2;
    if (relativeLuminance(hslToHex(h, s, mid)) < targetLum) lo = mid; else hi = mid;
  }
  return (lo + hi) / 2;
}

const cache = new Map<string, string>();

/**
 * The colour this role must be painted at this contrast level, or null when contrast is off.
 * Against a black backdrop the ratio R needs luminance (0.05R - 0.05), which is what is solved for.
 */
export function roleColor(role: SpriteRole, level: ContrastLevel): string | null {
  if (level === 0) return null;
  if (role === 'bg') return BG_RECESSED;
  const key = `${role}:${level}`;
  const hit = cache.get(key);
  if (hit) return hit;
  const { h, s } = ROLE_HS[role];
  const target = 0.05 * level - 0.05 + relativeLuminance(HC_BG);
  const out = hslToHex(h, s, solveLightness(h, s, target));
  cache.set(key, out);
  return out;
}

/** The one-pixel border drawn around every shape so two adjacent roles never read as one blob. */
export function outlineColor(_level: ContrastLevel): string { return '#ffffff'; }

let level: ContrastLevel = 0;
const listeners = new Set<() => void>();

export function getContrastLevel(): ContrastLevel { return level; }

/** Set the level and notify. A no-op set does not notify — repainting every sprite is not free. */
export function setContrastLevel(next: ContrastLevel): void {
  if (next === level) return;
  level = next;
  for (const fn of listeners) fn();
}

/** Subscribe to level changes. Returns the unsubscribe function. */
export function onContrastChange(fn: () => void): () => void {
  listeners.add(fn);
  return () => { listeners.delete(fn); };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run --project node engine/render/high-contrast.test.ts`
Expected: PASS, 12 tests. Every role at every level provably meets its ratio.

- [ ] **Step 5: Write the failing sprite test**

`engine/render/sprites.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { afterEach, describe, expect, it } from 'vitest';
import { Container } from 'pixi.js';
import { makeSpriteApi, roleCanvas } from './sprites.js';
import { roleColor, setContrastLevel } from './high-contrast.js';

afterEach(() => setContrastLevel(0));

const hexAt = (cv: HTMLCanvasElement, x: number, y: number): string => {
  const d = cv.getContext('2d')!.getImageData(x, y, 1, 1).data;
  return d[3] === 0 ? 'transparent' : `#${[d[0], d[1], d[2]].map((v) => v!.toString(16).padStart(2, '0')).join('')}`;
};

const square = { role: 'hazard' as const, w: 4, h: 4, paint: (px: (x: number, y: number, w: number, h: number, c: string) => void) => px(1, 1, 2, 2, '#00ff00') };

describe('roleCanvas', () => {
  it('keeps the game colours when contrast is off', () => {
    const cv = roleCanvas(square, 0);
    expect(hexAt(cv, 1, 1)).toBe('#00ff00');
  });

  it('is the size the spec asked for when contrast is off', () => {
    const cv = roleCanvas(square, 0);
    expect([cv.width, cv.height]).toEqual([4, 4]);
  });

  it('replaces every game colour with the role colour when contrast is on', () => {
    const cv = roleCanvas(square, 4.5);
    const expected = roleColor('hazard', 4.5)!;
    // The shape is offset by one pixel because the outline grows the canvas by a border.
    expect(hexAt(cv, 2, 2)).toBe(expected);
  });

  it('grows by one pixel of border on each side so the outline has somewhere to live', () => {
    const cv = roleCanvas(square, 4.5);
    expect([cv.width, cv.height]).toEqual([6, 6]);
  });

  it('draws an outline around the silhouette', () => {
    const cv = roleCanvas(square, 4.5);
    expect(hexAt(cv, 1, 2)).toBe('#ffffff');   // immediately left of the shape
  });

  it('leaves the area outside the outline transparent', () => {
    const cv = roleCanvas(square, 4.5);
    expect(hexAt(cv, 0, 0)).toBe('transparent');
  });

  it('recesses a bg-role sprite instead of brightening it', () => {
    const cv = roleCanvas({ ...square, role: 'bg' }, 7);
    expect(hexAt(cv, 2, 2)).toBe(roleColor('bg', 7));
  });
});

describe('makeSpriteApi', () => {
  it('adds nothing to the stage by itself — the game positions what it makes', () => {
    const stage = new Container();
    makeSpriteApi(stage);
    expect(stage.children.length).toBe(0);
  });

  it('produces a sprite sized to the spec', () => {
    const api = makeSpriteApi(new Container());
    const s = api.make(square);
    expect([s.width, s.height]).toEqual([4, 4]);
    api.clear();
  });

  it('repaints every registered sprite when the contrast level changes', () => {
    const api = makeSpriteApi(new Container());
    const s = api.make(square);
    const before = s.texture;
    setContrastLevel(7);
    expect(s.texture).not.toBe(before);
    api.clear();
  });

  it('stops repainting sprites released by clear()', () => {
    const api = makeSpriteApi(new Container());
    const s = api.make(square);
    api.clear();
    const after = s.texture;
    setContrastLevel(3);
    expect(s.texture).toBe(after);
  });
});
```

- [ ] **Step 6: Run the test to verify it fails**

Run: `npx vitest run --project browser engine/render/sprites.browser.test.ts`
Expected: FAIL — `Failed to resolve import "./sprites.js"`.

- [ ] **Step 7: Write `engine/render/sprites.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/sprites — the one factory every game's art goes through.
//
// A game describes a shape and says what it MEANS. It never picks contrast colours, never subscribes
// to the accessibility settings, and never repaints. Because every sprite is registered here, raising
// the contrast level repaints all of them at once — which is the whole reason a game costs 30 lines
// of accessibility instead of 300.
import { Sprite, type Container, type Texture } from 'pixi.js';
import { pixelCanvas, tex } from './canvas.js';
import { getContrastLevel, onContrastChange, outlineColor, roleColor, type ContrastLevel, type SpriteRole } from './high-contrast.js';

export type PixelBrush = (x: number, y: number, w: number, h: number, col: string) => void;

export interface SpriteSpec {
  role: SpriteRole;
  w: number;
  h: number;
  paint: (px: PixelBrush) => void;
}

/**
 * Render one spec at one contrast level.
 *
 * Off: the game's own painter runs untouched.
 *
 * On: the painter runs three times over a canvas grown by a one-pixel border. First eight offset
 * passes in the outline colour, which together form a sticker outline around whatever silhouette the
 * game drew; then one centred pass in the role colour. The game's requested colours are discarded —
 * that is the point, since two roles that happen to share a hue must not read as the same thing.
 */
export function roleCanvas(spec: SpriteSpec, level: ContrastLevel): HTMLCanvasElement {
  if (level === 0) return pixelCanvas(spec.w, spec.h, spec.paint);

  const fill = roleColor(spec.role, level)!;
  const line = outlineColor(level);
  const OFFSETS: ReadonlyArray<readonly [number, number]> = [
    [0, 0], [2, 0], [0, 2], [2, 2], [1, 0], [0, 1], [2, 1], [1, 2],
  ];

  return pixelCanvas(spec.w + 2, spec.h + 2, (px) => {
    for (const [ox, oy] of OFFSETS) spec.paint((x, y, w, h) => px(x + ox, y + oy, w, h, line));
    spec.paint((x, y, w, h) => px(x + 1, y + 1, w, h, fill));
  });
}

export interface SpriteApi {
  make(spec: SpriteSpec): Sprite;
  repaintAll(): void;
  clear(): void;
}

/**
 * Build the sprite API for one game session. `clear()` releases everything and unsubscribes, so a
 * game that is torn down cannot leave sprites behind that repaint forever.
 */
export function makeSpriteApi(_stage: Container): SpriteApi {
  const registry = new Map<Sprite, SpriteSpec>();

  const paint = (spec: SpriteSpec): Texture => tex(roleCanvas(spec, getContrastLevel()));

  function repaintAll(): void {
    for (const [sprite, spec] of registry) {
      const old = sprite.texture;
      sprite.texture = paint(spec);
      old.destroy(true);
    }
  }

  const unsubscribe = onContrastChange(repaintAll);

  return {
    make(spec) {
      const s = new Sprite(paint(spec));
      // The border added in high-contrast mode must not shift the game's hitbox or layout, so the
      // sprite is anchored on the shape's own top-left rather than the texture's.
      s.anchor.set(0, 0);
      s.width = spec.w;
      s.height = spec.h;
      registry.set(s, spec);
      return s;
    },
    repaintAll,
    clear() {
      unsubscribe();
      registry.clear();
    },
  };
}
```

- [ ] **Step 8: Run the tests to verify they pass**

Run: `npx vitest run engine/render/`
Expected: PASS. `canvas` 4, `mount` 10, `high-contrast` 12, `sprites` 11.

- [ ] **Step 9: Run typecheck and the full suite**

Run: `npx tsc --noEmit && npx vitest run`
Expected: no typecheck output; every test passes.

- [ ] **Step 10: Commit**

```bash
git add engine/render/high-contrast.ts engine/render/high-contrast.test.ts engine/render/sprites.ts engine/render/sprites.browser.test.ts
git commit -m "feat: high contrast resolved from sprite role tags

Each role owns a hue and the lightness is solved until the colour hits the
luminance the target ratio demands, so the test can measure the real WCAG
ratio rather than trust hand-picked hex. Games declare meaning; the engine
repaints every registered sprite when the level changes."
```

---

### Task 9: Colour-vision and low-vision filters

**Files:**
- Create: `engine/render/cvd-matrices.ts`, `engine/render/viz.ts`
- Test: `engine/render/viz.test.ts`, `engine/render/viz.browser.test.ts`

**Interfaces:**
- Consumes: `storage` (`KEYS.viz`) from Task 2.
- Produces:
  - From `cvd-matrices.ts` (lifted): `type CvdKey`, `CVD_KEYS`, `CVD_MATRIX`, `CVD_SVG_ID`, `cvdMatrixValues(k)`, `installCvdFilters(host): number`.
  - From `viz.ts`: `type VizKey`, `VIZ_MODES`, `vizFilter(key)`, `vizZoom(key)`, `getViz()`, `setViz(key, target)`, `initViz(host, target)`.

> `cvd-matrices.ts` is copied verbatim from `<TRACER>/app/js/render/cvd-matrices.ts` — 120 numbers from
> Machado 2009 (simulation) and Fidaner et al. (correction), which nobody should retype. Only the header
> comment is translated.
>
> `viz.ts` is **not** the tracer's `viz-modes.ts`. That file carries pt-BR `nome`/`desc` strings inline,
> which contradicts D10: every visible string in this project resolves through `t()`. So the mode table
> is rewritten with i18n keys, and its sixteen tracer modes are cut to the eight that apply without a
> game-specific renderer.

- [ ] **Step 1: Copy `cvd-matrices.ts` from the tracer**

Copy `<TRACER>/app/js/render/cvd-matrices.ts` to `engine/render/cvd-matrices.ts`. Translate the header
comment to English. Change nothing else — the matrices, `CVD_SVG_ID` and `installCvdFilters` are used
exactly as they are.

- [ ] **Step 2: Add the shell i18n keys for the modes**

Append to `engine/i18n/pt.ts`:
```ts
  'viz.none': 'Cores normais',
  'viz.simProtan': 'Simular protanopia',
  'viz.simDeuter': 'Simular deuteranopia',
  'viz.simTritan': 'Simular tritanopia',
  'viz.fixProtan': 'Corrigir para protanopia',
  'viz.fixDeuter': 'Corrigir para deuteranopia',
  'viz.fixTritan': 'Corrigir para tritanopia',
  'viz.lowVision': 'Baixa visão (ampliar)',
```

Append to `engine/i18n/en.ts`:
```ts
  'viz.none': 'Normal colours',
  'viz.simProtan': 'Simulate protanopia',
  'viz.simDeuter': 'Simulate deuteranopia',
  'viz.simTritan': 'Simulate tritanopia',
  'viz.fixProtan': 'Correct for protanopia',
  'viz.fixDeuter': 'Correct for deuteranopia',
  'viz.fixTritan': 'Correct for tritanopia',
  'viz.lowVision': 'Low vision (magnify)',
```

Append to `engine/i18n/es.ts`:
```ts
  'viz.none': 'Colores normales',
  'viz.simProtan': 'Simular protanopía',
  'viz.simDeuter': 'Simular deuteranopía',
  'viz.simTritan': 'Simular tritanopía',
  'viz.fixProtan': 'Corregir para protanopía',
  'viz.fixDeuter': 'Corregir para deuteranopía',
  'viz.fixTritan': 'Corregir para tritanopía',
  'viz.lowVision': 'Baja visión (ampliar)',
```

- [ ] **Step 3: Write the failing node test**

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

- [ ] **Step 4: Write the failing browser test**

`engine/render/viz.browser.test.ts`:
```ts
// SPDX-License-Identifier: GPL-3.0-or-later
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { installCvdFilters } from './cvd-matrices.js';
import { getViz, initViz, setViz } from './viz.js';

let host: HTMLElement;
let target: HTMLElement;

beforeEach(() => {
  vi.stubGlobal('localStorage', (() => {
    const m = new Map<string, string>();
    return { getItem: (k: string) => m.get(k) ?? null, setItem: (k: string, v: string) => { m.set(k, v); }, removeItem: (k: string) => { m.delete(k); } };
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

describe('viz application', () => {
  it('applies no filter in the default mode', () => {
    initViz(host, target);
    expect(getViz()).toBe('none');
    expect(target.style.filter).toBe('');
  });

  it('applies the SVG filter reference for a cvd mode', () => {
    initViz(host, target);
    setViz('fix-deuter', target);
    expect(target.style.filter).toContain('url(#cvd-');
  });

  it('applies a zoom transform in low vision and removes it on return', () => {
    initViz(host, target);
    setViz('low-vision', target);
    expect(target.style.transform).toContain('scale(');
    setViz('none', target);
    expect(target.style.transform).toBe('');
  });

  it('persists the choice and restores it on the next init', () => {
    initViz(host, target);
    setViz('sim-tritan', target);
    const fresh = document.createElement('div');
    initViz(host, fresh);
    expect(getViz()).toBe('sim-tritan');
    expect(fresh.style.filter).toContain('url(#cvd-');
  });
});
```

- [ ] **Step 5: Run both tests to verify they fail**

Run: `npx vitest run engine/render/viz`
Expected: FAIL — `Failed to resolve import "./viz.js"`.

- [ ] **Step 6: Write `engine/render/viz.ts`**

```ts
// SPDX-License-Identifier: GPL-3.0-or-later
// render/viz — the visual accessibility modes that cost a game nothing.
//
// These apply to the CANVAS ELEMENT, not to anything a game draws: a CSS filter over the whole
// surface, plus a magnification transform. So every one of the 383 games gets colour-blindness
// simulation, colour-blindness correction and low-vision magnification without a line of its own.
//
// Not the tracer's viz-modes.ts. That table carries pt-BR labels inline; here every visible string is
// an i18n key, and the sixteen tracer modes are cut to the eight that need no game-specific renderer.
// High contrast is NOT here — it is a repaint, not a filter, and lives in high-contrast.ts.
import * as store from '../platform/storage.js';
import { CVD_SVG_ID, type CvdKey } from './cvd-matrices.js';

export type VizKey = 'none' | CvdKey | 'low-vision';

export interface VizMode {
  key: VizKey;
  kind: 'none' | 'cvd' | 'zoom';
  /** i18n key for the label. Never a literal string — see D10. */
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
  if (!m || m.kind !== 'cvd') return '';
  return `url(#${CVD_SVG_ID[key as CvdKey]})`;
}

/** The magnification for a mode. Unknown keys degrade to 1. */
export function vizZoom(key: VizKey): number {
  return BY_KEY.get(key)?.kind === 'zoom' ? LOW_VISION_ZOOM : 1;
}

let current: VizKey = 'none';
export function getViz(): VizKey { return current; }

/** Apply a mode to the target element and persist the choice. */
export function setViz(key: VizKey, target: HTMLElement): void {
  current = BY_KEY.has(key) ? key : 'none';
  store.set(store.KEYS.viz, current);
  const filter = vizFilter(current);
  const zoom = vizZoom(current);
  target.style.filter = filter;
  // Empty string rather than `scale(1)`: a lingering transform creates a containing block and a
  // stacking context, which silently changes how the pause dialog above the canvas is positioned.
  target.style.transform = zoom === 1 ? '' : `scale(${zoom})`;
  target.style.transformOrigin = zoom === 1 ? '' : 'center center';
}

/** Boot: inject the filter definitions into `host` and restore the saved mode onto `target`. */
export function initViz(host: Element | null, target: HTMLElement): void {
  void host; // filters are installed by the caller via installCvdFilters; kept for call-site symmetry
  const saved = store.get(store.KEYS.viz, null);
  setViz(saved && BY_KEY.has(saved) ? (saved as VizKey) : 'none', target);
}
```

- [ ] **Step 7: Run both tests to verify they pass**

Run: `npx vitest run engine/render/viz`
Expected: PASS. `viz` node 9, `viz` browser 7.

- [ ] **Step 8: Commit**

```bash
git add engine/render/cvd-matrices.ts engine/render/viz.ts engine/render/viz.test.ts engine/render/viz.browser.test.ts engine/i18n/
git commit -m "feat: add colour-vision and low-vision filters

The 120 Machado/Fidaner matrices come from the tracer untouched. The mode
table is rewritten with i18n keys instead of the tracer's inline pt-BR
labels, and trimmed to the modes that need no game-specific renderer."
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

<script type="module" src="./engine/shell/boot.ts"></script>
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
