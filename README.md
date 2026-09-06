# The Inclusionist — o catálogo

Este repositório mudou de papel, e o registro que o mudou explica melhor do que um resumo: pelo
**ADR-0068**, ele **deixou de guardar os jogos** e passou a guardar o **manifesto** que decide quais
jogos, e em quais versões, entram numa entrega.

⚠️ **E a razão não é organização — é orçamento de bytes.** O pilar 1 é hardware de escola pública e o
pilar 8 é PWA offline: o orçamento de precache **sempre** proibiu mandar o catálogo inteiro ao aparelho.
Ou seja, este repositório nunca foi a unidade de entrega; ele só parecia ser. Cada jogo é um repositório
próprio (**ADR-0068**, `game-<slug>` pelo **ADR-0082**), e aqui se **escolhe** o que sai numa versão.

## O que tem aqui hoje

- **`minigames-catalog-v2.html`** — os 300+ jogos do catálogo de demonstração, que são o MVP.
- **`docs/`** — o material que acompanha o catálogo.

⚠️ **O que NÃO tem ainda: o manifesto.** É a peça central do papel novo — jogo, versão, e por que ele
está nesta entrega — e ela não existe. Enquanto não existir, este repositório descreve um papel que ainda
não desempenha, e está escrito aqui para que isso não se leia como pronto.

## Licença

Código: **AGPL-3.0-or-later** (**ADR-0064**) — o `LICENSE` desta raiz, acrescentado em 2026-09-06 na
migração, porque este repositório veio do GitLab sem nenhuma.
⚠️ **A arte NÃO é AGPL.** Programa é o que a Lei 9.609 define; arte segue a Lei 9.610 e pertence a quem a
fez. O que governa o quê está em `docs/LICENSES.md`, na engine.

⚠️ **A titularidade patrimonial é do MUNICÍPIO** (Lei nº 9.609/1998, art. 4º), não do desenvolvedor. A
publicação é objeto de **pedido** no requerimento — ato do Poder Executivo — e por isso este repositório é
**privado** (ADR-0066 §3).

---

Os registros vivem na engine, em `docs/2-Architecture/adr/` — **não** há pasta `adr/` aqui, e isso é
decisão (ADR-0068 §5).
