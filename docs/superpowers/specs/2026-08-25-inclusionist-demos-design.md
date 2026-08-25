# Design — inclusionist-demos: um jogo por item do catálogo

**Data:** 2026-08-25 · **Status:** aprovado pelo Dev · **Escopo:** arquitetural

> [!note] Idioma dos artefatos
> Documentação em **pt-BR** (o produto e o público desta página são pt-BR, e o repo é privado).
> **Código, comentários e mensagens de commit em inglês**, como no tracer — higiene de projeto GPL.

## 1. Objetivo

`minigames-catalog-v2.html` lista 35 categorias e **383 subgêneros** de minigame JS. O objetivo é
construir um jogo jogável para cada item e ligar o item ao jogo, reaproveitando a stack e as
convenções de `SP-the-inclusionist-tracer`.

Pelo menos 4 nomes aparecem em mais de um card (`Snake roguelite`, `Auto-runner platformer`,
`Esgrima de timing`, `Spot the difference`), então o número de jogos **únicos** é menor que 383 —
o manifesto resolve isso com `aliasOf` em vez de duplicar implementação.

## 2. Decisões

| # | Decisão | Alternativa descartada |
|---|---|---|
| D1 | 383 jogos como alvo, entrega fatiada por card | 35 jogos (um por categoria); subconjunto curado |
| D2 | Herda só o **núcleo técnico** do tracer | Herdar a11y+i18n; herdar os 10 pilares do EdSP |
| D3 | Monorepo com engine compartilhada | Jogos autocontidos; engine como pacote consumido pelos dois repos |
| D4 | `jrocha-dev/inclusionist-demos`, **privado** | Público; nome `js-minigames` |
| D5 | Pixel art em canvas **320×180** (igual ao tracer) | 360×180 — descartado: não é 16:9 e não fecha escala inteira |
| D6 | SVG ou 3D só quando a natureza do jogo exigir, com justificativa registrada | Deixar a escolha de renderer livre por jogo |
| D7 | Casca única com import dinâmico | 383 entradas em `rollupOptions.input` |
| D8 | Catálogo **gerado** a partir de `data/catalog.json` | Editar os 383 `<li>` à mão |

**D6 na prática:** `pixel` é o padrão. Um jogo que declare outro renderer preenche `rendererWhy`
no **próprio `meta`** (o código é a fonte, não o manifesto), e o build falha se um renderer
não-`pixel` vier sem justificativa — o desvio fica auditável em vez de virar hábito.

## 3. Arquitetura

### 3.1 Estrutura

```
inclusionist-demos/
├─ index.html                  # GERADO por scripts/build-catalog.mts — nunca editar à mão
├─ play.html                   # casca única que hospeda qualquer jogo
├─ data/catalog.json           # fonte da verdade do catálogo
├─ src/catalog.template.html   # hero, intro, changelog, CTA e CSS atuais, preservados
├─ scripts/
│  ├─ import-catalog.mts       # uso único: HTML atual → catalog.json
│  └─ build-catalog.mts        # catalog.json + template → index.html
├─ engine/
│  ├─ core/      constants, loop, rng, collision, state
│  ├─ input/     keyboard, gamepad, state, latch
│  ├─ render/    canvas, palette, sprites, fx
│  ├─ platform/  audio, storage
│  └─ shell/     boot, router, hud, pause
└─ games/<categoria-slug>/<jogo-slug>/
   ├─ main.ts
   └─ main.test.ts
```

Slugs derivam do **nome**, não do número (`games/arcade-classico/snake/`): reordenar o catálogo
não renomeia pastas nem quebra links.

### 3.2 Casca única e roteamento

`play.html` é a única entrada de build além de `index.html`. Ela descobre os jogos por
`import.meta.glob('./games/*/*/main.ts')` e carrega sob demanda, roteando por hash:

```
play.html#arcade-classico/snake
```

Ganhos: uma entrada de build em vez de 383; PixiJS num chunk compartilhado; HUD, pausa e
"voltar ao catálogo" nascem idênticos em todos os jogos; adicionar jogo é criar pasta, sem
tocar em configuração.

### 3.3 Contrato do jogo

```ts
export const meta: GameMeta;
export function setup(ctx: GameContext): void;
export function update(dt: number): void;   // dt em QUADROS, não em segundos
export function teardown(): void;
```

```ts
interface GameMeta {
  slug: string; title: string; category: string;
  density: 'leve' | 'medio' | 'denso';
  players: 1 | 2 | 3 | 4;
  renderer?: 'pixel' | 'svg' | '3d';        // default 'pixel'
  rendererWhy?: string;                     // obrigatório se renderer !== 'pixel'
}
interface GameContext {
  stage: Container;                          // raiz PixiJS, espaço lógico 320×180
  input: InputApi;                           // held(pl, act), pressed(pl, act), players
  audio: AudioApi; rng: Rng; storage: StorageApi;
  onGameOver(score: number): void;
}
```

> [!warning] Duas convenções herdadas do tracer que é fácil quebrar
> **`dt` é contado em quadros, não em segundos** — física copiada de tutorial em segundos anda errado.
> **O teclado é escutado em `#game-region`, não em `window`** — em `window` o jogo rouba as teclas da página.

### 3.4 Engine

Espelha a divisão do tracer, com os módulos reescritos para o escopo daqui:

- **core** — `constants` (`LOGICAL_W=320, LOGICAL_H=180, TILE=16`), `loop` (RAF com `dt` em quadros e
  clamp contra travadas), `rng` (PRNG semeável, para jogo procedural ser testável), `collision` (AABB e varrida).
- **input** — as **8 ações** do tracer (`up, left, down, right, run, jump, swap, especial`), esquemas
  `solo/p2/p3/p4` persistidos, gamepad com `PAD_DEAD = 0.5`, e a consulta unificada
  `held(pl, act) = teclado OU gamepad`. `latch` fornece as bordas (`pressed`/`released`).
- **render** — `canvas` monta o PixiJS com `NEAREST` e **escala inteira** (nunca fracionária, que borra
  pixel art); `palette`, `sprites` procedurais e `fx` (partículas, screen shake).
- **platform** — `audio` (Web Audio, sem arquivo) e `storage` (recordes, controles remapeados).
- **shell** — `boot`, `router`, `hud`, `pause`.

**Arte é dado**, como no tracer: nenhum PNG embutido. Sprites nascem de descrição procedural.

## 4. Catálogo dirigido por dados

`data/catalog.json` é a fonte da verdade:

```json
{ "version": "2.0.0",
  "categories": [{
    "id": 1, "slug": "arcade-classico", "title": "Arcade Clássico",
    "desc": "tela única · loop curto · score", "density": "leve", "accent": "c1", "new": false,
    "items": [
      { "slug": "snake", "name": "Snake / Cobrinha", "hint": "grid · auto-move · cresce",
        "fresh": false, "status": "todo" },
      { "slug": "snake-roguelite", "name": "Snake roguelite", "hint": "upgrades por morte",
        "fresh": true, "status": "todo", "aliasOf": "hibridos-mashups/snake-roguelite" }
    ]
  }]
}
```

`status` é `todo | wip | done`. A geração:

1. item `done` vira `<a href="play.html#<categoria>/<slug>">`; `todo` continua texto puro — nada de
   383 links quebrados, e a página vira medidor de progresso honesto;
2. item com `aliasOf` (formato `"<categoria>/<slug>"`) aponta para o jogo já existente, marcado
   como variação, e não gera pasta própria em `games/`;
3. as estatísticas do hero passam a ser **calculadas**, o que conserta o "280+" hoje defasado (são 383);
4. hero, intro, changelog, CTA, rodapé e todo o CSS vêm do template, intocados.

`import-catalog.mts` roda uma vez para extrair o JSON do HTML atual, preservando nome, hint, marca
`fresh`, densidade, cor de acento e badge `NEW`.

**Destino do arquivo atual:** na Fase 1, `minigames-catalog-v2.html` é dividido — a prosa e o CSS
viram `src/catalog.template.html`, os dados viram `data/catalog.json`, e o arquivo original sai da
raiz. O `index.html` gerado continua **autocontido e abrível offline**, que é a única propriedade
do arquivo original que precisa sobreviver. O histórico do git preserva o original, e o commit que
o remove cita esta seção.

## 5. Testes

Vitest com os dois *projects* do tracer: **node** (regras puras — lógica de jogo, colisão, rng) e
**browser**/Playwright (render, input, casca). Cada jogo nasce com teste de regra; TDD.

## 6. Fatiamento

| Fase | Entrega | Por quê |
|---|---|---|
| 0 | Repo, remote, esta especificação | — |
| 1 | Engine + `play.html` + **Snake, Pong, Breakout** + gerador do catálogo + links vivos | Cobrem lógica de grade, input de 2 jogadores e colisão contínua com FX — se a engine estiver errada, esses três revelam |
| 2 | Fechar o card 01 inteiro (17 itens, todos leves) | Primeiro card 100% ligado, valida o ritmo por categoria |
| 3+ | Card a card, densidade crescente | Categorias `denso` entram como **fatia vertical**, não jogo completo |

## 7. Infraestrutura

- Git em `main`; commits **atômicos e frequentes**, em inglês, sem "initial" gigante.
- Remote: `jrocha-dev/inclusionist-demos`, privado.
- Licença **GPL-3.0-or-later** com cabeçalho SPDX por arquivo, como no tracer.
- Node 24, TypeScript, Vite, Vitest, PixiJS 7.4.2 — as versões do tracer.
- Sem caminho absoluto em arquivo versionado.

## 8. Não-objetivos (YAGNI explícito)

Fora de escopo por D2: i18n, conformidade WCAG 2.2 vendida como tal, PWA/offline, telemetria
xAPI, LGPD/COPPA, multiplayer em rede, editor de níveis, backend.

Higiene de acessibilidade **continua valendo** (foco visível, contraste razoável, nada de flash
rápido, jogo controlável só pelo teclado) — o que não fazemos é auditar e **vender** conformidade.
A distinção importa: o tracer marca honestamente onde só alcança AA, e o pior resultado aqui seria
alegar um nível que ninguém verificou.

## 9. Riscos

| Risco | Mitigação |
|---|---|
| 383 jogos é trabalho de meses e pode morrer no meio | O medidor de progresso no catálogo torna o avanço visível; entrega por card fecha unidades úteis |
| Fork silencioso: a engine daqui divergir da do tracer | Se as duas convergirem na prática, D3 é revisitado e a engine vira pacote consumido pelos dois |
| Tempo de build crescer com centenas de chunks | Casca única já evita 383 entradas; se doer, agrupar chunks por categoria |
| Categorias `denso` inflarem escopo | Contrato explícito: fatia vertical (uma sala, um chefão, um loop de 60s) |
