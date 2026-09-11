# BNCC tags — what earns one, and what the six games actually earn

## What a tag is here

A BNCC code on a game card is a **filter facet**, not a coverage claim. A teacher ticks codes on
screen 1 and the catalogue narrows. That is the whole job.

But a filter that returns a game which does not work the skill is a broken filter, so the bar is not
zero. The bar is the **zone of proximal development**, and this project already has an operational
definition of it — `ADR-0048` §5, which replaced an earlier version of its own clause for getting
this exactly wrong:

> REPLACES the first version of this clause, which set "20% of error costs nothing" and claimed the
> zone of proximal development as its target. The requerimento (Anexo XIII §73.c) is right and that
> version was wrong: **measuring only unaided performance yields DIFFICULTY, not the zone.**

and, in the band that names it:

> **7 or more of the last 10 resolved, using the second and third attempts → THE ZONE.** She cannot
> do it alone and she can do it with mediation, **which is the definition, not an approximation of it.**

## The gate

A code earns a tag only if it passes both questions.

**1 · Does the skill hold the game up?** Can the child win around it? If yes, the game does not work
that skill — it merely displays it.

**2 · Does the game supply the mediation?** Is there scaffolding inside the game that makes the skill
reachable *with help* — a hint, a teacher that turns itself on, narration that names the parts, a
difficulty dial that is curricular rather than motor?

Question 1 alone yields difficulty. Question 2 is what makes it the zone.

### The worked case: chess and coordinates

Chess looks like it teaches coordinates. Its accessible board is not a canvas but
**64 real buttons in a real grid, each labelled in algebraic notation** —
`app/js/ui/grid-mirror.ts:7`. Every square announces its own name.

And it fails question 1 outright: **you can play chess and win without reading a single one of those
labels.** The notation is a reading surface laid over the game, not a load-bearing part of it. So
`EF05MA14` and `EF05MA15` (coordenadas cartesianas) do not become tags — however tempting the
64-square grid makes them look.

This is the test every row below had to survive.

## Two classes of tag

Applied honestly, the gate rejects most of what "looks right", and some games end with very few
tags. That is the result, not a failure. But it leaves out something legitimate: a game can be
**material for a lesson** about a skill without working the skill. That is exactly the shape of the
Educação Física habilidades, whose verb is *experimentar e fruir* — playing already is doing, and the
second half ("valorizando… os sentidos e significados atribuídos a eles por diferentes grupos sociais
e etários") is the teacher's work, not the game's.

| Class | Means | Enters if |
|---|---|---|
| **Works** | the game works the zone | passes both questions |
| **Material** | the game is valid material for the lesson; the work belongs to the teacher | the habilidade's verb is *experimentar e fruir*, or the game's object is the habilidade's object without being a win condition |

> ⚠️ **Decision pending.** Adopting both classes, or only "Works", is the Dev's call. It changes the
> card's wording — see *The card says "Cobre"* below.

---

## What the six games earn

### whackwhack — the cleanest case of the six

**`EF06MA05`** · «Classificar números naturais em primos e compostos, estabelecer relações entre
números, expressas pelos termos "é múltiplo de", "é divisor de", "é fator de", e estabelecer, por
meio de investigações, critérios de divisibilidade por **2, 3, 4, 5, 6, 8, 9**, 10, 100 e 1000.»
**`EF06MA06`** · «Resolver e elaborar problemas que envolvam as ideias de múltiplo e de divisor.»
— class **Works**.

- *Holds it up?* Every hammer stroke is a multiple judgement, and a lit tile carrying a wrong value
  is a `hazard`, not a miss (`app/js/declaration/whack-declaration.ts:168`). There is no way to score
  around the skill.
- *Mediates?* `LIT_AT_ONCE = { easy: 2, medium: 3, hard: 4 }` (`app/js/rules/difficulty.ts:50`) is
  declared a **curricular** dial, deliberately kept orthogonal to the engine's motor EASY mode
  (`difficulty.ts:22`) — folding them together would offer accessibility as if it were a baby mode.
  And the level rises by tiles judged rather than by a clock, so a child who works slowly is not
  hurried by a timer she was never shown.
- *Corroboration worth recording:* the game's categories are
  `FACTORS = [2, 3, 4, 5, 6, 7, 8, 9]` (`app/js/rules/category.ts:145`) and the habilidade's
  divisibility list is 2, 3, 4, 5, 6, 8, 9. They agree on **seven of eight**, and the one
  disagreement is **7** — precisely the divisor with no simple divisibility criterion. The overlap
  was not designed against the text; it fell out of the mathematics.

### 2048

**`EF02MA08`** · «Resolver e elaborar problemas envolvendo **dobro**, metade, triplo e terça parte,
com o suporte de imagens ou material manipulável, utilizando estratégias pessoais.»
— class **Works**.

- *Holds it up?* Every merge is a doubling. Nothing scores without one.
- *Mediates?* The narration names both parts and the result rather than announcing a new number:
  `'move.pair': '{a} e {a} viraram {b}'` (`app/js/i18n/pt.ts:51`), filled at
  `app/js/narration.ts:44` from the exponents on either side of the merge. The tiles on the board
  are the "material manipulável" the habilidade asks for.

### 15-puzzle — and Hanoi, when it exists

**No Works tag today.** This reverses an earlier draft of this document, which proposed `EF04MA01`
(«Ler, escrever e **ordenar** números naturais até a ordem de dezenas de milhar»), and the Dev's
objection is the reason:

> Ordenar é o de menos no jogo do 15: ou se cria uma estratégia para movimentar a peça desejada ou
> não se chega ao fim deste jogo.

He is right, and the gate agrees with him once it is applied properly. Ordering survives question 1
only in a trivial sense — the goal state happens to be ordered — but it is not where the difficulty
lives. A child who can count to fifteen is no closer to solving the puzzle. What she cannot do alone
is **plan a sequence of moves that reaches a target without destroying what is already placed**, and
that is the zone. `EF04MA01` would have tagged the goal state instead of the work, which is the same
error as tagging chess with coordinates.

The mediation, for the record, is real and stays relevant to whatever tag eventually lands here: the
hint is free, unlimited, uncounted and unpenalised, and it reveals the next moves without ever
playing them (`app/js/puzzle/run.ts:15-27`). The reasoning written there is the ZPD argument in all
but the name — a lock would have to invent a penalty in order to have something to protect, and the
child who most needs help on move one is exactly the child the game exists for.

#### Does `EF67EF05` cover it, if the sport theme is set aside?

Asked directly: **no.** The sport list in `EF67EF05` is not theming, it is the scope. The habilidade
lives inside the *Esportes* unidade temática of Educação Física, a component built around the cultura
corporal de movimento, and «os desafios técnicos e táticos» is bound to the sports named beside it
rather than free-standing. Read as generic strategy it would match every game with a challenge in it,
which as a filter facet is the same uselessness as `EF06MA03` — a facet that matches everything
filters nothing.

#### Where the strategy skill actually lives in the BNCC

It has a home, and the home has a condition attached. Matemática carries algorithm habilidades:

- **`EF06MA04`** · «Construir algoritmo em linguagem natural e representá-lo por fluxograma que
  indique a resolução de um problema simples (por exemplo, se um número natural qualquer é par).»
- **`EF07MA07`** · «Representar por meio de um fluxograma os passos utilizados para resolver um grupo
  de problemas.»
- **`EF08MA10`** · «Identificar a regularidade de uma sequência numérica ou figural não recursiva e
  construir um algoritmo por meio de um fluxograma que permita indicar os números ou as figuras
  seguintes.»

Every one of them binds the strategy to an **externalised representation** — *construir*,
*representar*, *descrever* — and the 15-puzzle externalises nothing. The child plans in her head,
moves, and the plan is never uttered. So these fail question 1 today for the same reason
`EF04MA16`'s *descrever deslocamentos* failed it.

**But they are one design decision away from passing.** If the puzzle gains a step where the child
states or assembles her plan before executing it — even a three-step "which row, which direction,
why" — `EF06MA04` and `EF07MA07` become honest Works tags, and they would be the first tags in the
collection to sit on the skill the puzzle is actually about. That is a design call, not a tagging one,
and it is recorded here so the choice is visible.

**Hanoi**, when implemented, is the stronger case of the two: its solution *is* a recursive algorithm,
and `EF08MA10` is written for exactly that shape. The same condition applies — the algorithm has to
leave the child's head.

Meanwhile the BNCC's own prose is on this side of the argument even where its habilidades are not.
The Matemática introduction names **pensamento computacional** eight times and defines it as «a
identificação de padrões para se estabelecer generalizações, propriedades e algoritmos». These
puzzles sit squarely inside the document's stated intent while matching none of its habilidades — a
gap worth recording rather than papering over with a code that does not fit.

### All six — class **Material**

**`EF67EF01`** · «Experimentar e fruir, na escola e fora dela, **jogos eletrônicos** diversos,
valorizando e respeitando os sentidos e significados atribuídos a eles por diferentes grupos sociais
e etários.»

This is the only code that truthfully covers all six — and it covers them **only in the 6º–7º ano**.
*Jogos eletrônicos* exists as a unidade temática in exactly two habilidades, `EF67EF01` and
`EF67EF02`, and in no other year band. There is no "electronic game" anchor anywhere in the
fundamental I.

**`EF67EF02`** · «Identificar as **transformações** nas características dos jogos eletrônicos em
função dos **avanços das tecnologias** e nas respectivas **exigências corporais** colocadas por esses
diferentes tipos de jogos.»

The proposed rule — *tag it when the game has more than one graphic version* — has the right
instinct and is too short. Three skins are not the habilidade. The habilidade has two halves, and
different games serve each:

- *Technological transformation:* **chess**, whose 2D, 2.5D and 3D are three separate build entries
  (`vite.config.ts`, inputs `main` / `flat` / `solid`) — the same game rendered in three eras, side by
  side, reachable by a link rather than hidden in a settings menu.
- *Bodily demands:* **pinball**, whose control model is a cabinet — three keys per flipper, the
  plunger held rather than tapped, and the arrow keys deliberately not wired to the flippers; and
  **the platformer**, with gamepad remapping per player, a wheelchair mode, split screen for two to
  four players, and audio routed to each player's own device.

**`EF35EF01`** · «Experimentar e fruir brincadeiras e jogos populares do Brasil e do mundo, incluindo
aqueles de matriz indígena e africana, e recriá-los, valorizando a importância desse patrimônio
histórico cultural.»

Applies to **chess** and **15-puzzle** squarely; defensible for **whackwhack** (a fairground game)
and **pinball** (an arcade one); false for **2048** and the **platformer**, which are not popular
games in the sense the text means.

---

## What was proposed and does not survive

| Proposed | Verdict |
|---|---|
| `EF01EF01`, `EF03EF01` | **These codes do not exist.** See below. |
| `EF67EF05`, `EF89EF03` (chess, and the puzzles) | Both are scoped to bodily sport categories — «esportes de marca, precisão, invasão e técnico-combinatórios» and «esportes de campo e taco, rede/parede, invasão e combate». Chess is in neither list, and neither is a sliding puzzle. The Educação Física component is built around the cultura corporal de movimento. The "estratégias técnicas e táticas" wording is the trap: the strategy clause is bound to the sports named beside it, not free-standing. |
| `EF04MA02` (15-puzzle) | Asks for decomposition and composition **por potências de dez**. The puzzle decomposes nothing. |
| `EF04MA01` (15-puzzle) | **Proposed by an earlier draft of this document and withdrawn.** Ordering is the goal state, not the work — see the 15-puzzle section. |
| `EF06MA03` (2048) | «Resolver e elaborar problemas que envolvam cálculos… com números naturais, por meio de estratégias variadas». Passes question 1 and fails at being useful: nearly every arithmetic game satisfies it. A facet that matches everything filters nothing. |
| `EF08MA01` (2048) | **Fails question 1**, and it was the most tempting code in the whole set. The exponent is real but internal — `OBJETIVO = 11` (`app/js/board.ts:31`) is an exponent, and the child never sees one, never writes one, and never touches notação científica, which is the habilidade's second half. She sees tiles that double. |
| `EF05MA14`, `EF05MA15` (chess) | Fails question 1 — the worked case above. |
| `EF04MA16`, `EF03MA12` (15-puzzle) | Both verbs are **descrever** deslocamentos. The game describes the movement *for* the child, through the screen reader; the child never describes anything. |
| `EF04MA11` (whackwhack, 2048) | «Identificar regularidades em **sequências** numéricas compostas por múltiplos de um número natural». Neither game presents a sequence — whackwhack presents isolated values to judge, and 2048 presents pairs to merge. |
| `EF07MA01` (whackwhack) | MDC and MMC are never exercised. |

### The codes that do not exist

`EF01EF01` and `EF03EF01` cannot exist, and the reason is structural rather than accidental:
**Educação Física groups its years.** The component's codes run `EF12EF__` (1º–2º), `EF35EF__`
(3º–5º), `EF67EF__` (6º–7º) and `EF89EF__` (8º–9º). A search of the official document returns zero
occurrences of `EF01EF`, `EF02EF`, `EF03EF`, `EF04EF` or `EF05EF`.

The intended equivalents are **`EF12EF01`** and **`EF35EF01`**:

- `EF12EF01` · «Experimentar, fruir e recriar diferentes brincadeiras e jogos da cultura popular
  presentes **no contexto comunitário e regional**…» — the locality clause excludes most of these six.
- `EF35EF01` · «…jogos populares **do Brasil e do mundo**…» — the broader one, and the one used above.

---

## Three structural caveats

**A tag matches a verb, rarely a whole habilidade.** A BNCC habilidade binds a verb to a content
range *and* a school year. Our games routinely hit the verb and miss the range — the 15-puzzle orders
numbers but only up to 24. This is the strongest argument for the Dev's own framing: a code is a
facet, not a claim.

**The key is `(habilidade, ano)`, never the habilidade alone.** The project already recorded this and
already rejected the naive shape:
`the-inclusionist-gamified-learning/docs/campos-inferidos-candidatos.md:109` — «Não é função.
`quadro-numerico` está no 1º e no 3º ano… A chave mínima é `(skill, ano)`».

**The card no longer says "Cobre".** The design adopted as the home page asserted coverage: the
screen-2 card showed `Cobre: EF05MA03` / `Não cobre: …`, and the design system states that «o cartão
sempre mostra quais habilidades ele cobre. Sem isso, um jogo não entra no catálogo». With this gate
rejecting most candidates, *Cobre* was not sustainable.

**Decided by the Dev on 2026-09-11:** the card reads **"Trabalha"** for the Works class and
**"Serve de material para"** for the Material class. The design system's rule survives intact — a
game with neither still does not enter the catalogue — but it now claims what it can defend.

---

## Not tagged, and why

**pinball** earns **no Works tag today.** A full sweep of the repository finds no BNCC reference and
no curricular content of any kind — it is a faithful port of a pinball table, and that is all it
claims to be. It keeps `EF67EF01` and `EF67EF02` as Material, where it is in fact the best instrument
in the collection for the second.

**But the Dev decided on 2026-09-11 that it becomes a game about building tables and sharing them**,
and that changes its ceiling more than any other decision in this document. See below.

**chess** earns no Works tag either, and this is a fact about the norm rather than a defect of the
game. It is by far the richest teaching apparatus of the six — thirteen lessons, two hundred curated
tactics, geometry-generated endgames — and its scaffold is the most deliberate in the collection:
`STUMBLES_BEFORE_HELP = 3` (`app/js/chess/protection.ts:15`), counted **per position rather than per
game**, with the reasoning written out beside it («three blunders spread over forty moves is a child
learning at a perfectly normal rate; three in a row from one position is a child stuck»), and the
game says out loud that it made the decision (`app/js/boot/game-shell.ts:1263-1266`). What it teaches
is chess, and chess is not a BNCC habilidade.

---

---

## The pinball as a table builder — conditional tags

**Decision, 2026-09-11:** to be educational, the pinball becomes a game about **creating tables and
sharing them with other players**.

Half of this already exists in that repository: `docs/plans/2026-09-08-a-table-editor.md` sets the
three steps (size → background → parts), records the Dev's own answer to who holds the mouse — «a
teacher or a child making a table of their own» — and its step 1, the part library, landed the same
day (`app/js/table/parts.ts`). What today's decision changes is the second half: §8 of that plan
lists «sharing a table with anybody else» among the things v1 deliberately does **not** do. That is
now a requirement rather than a deferral.

### Why this clears the gate when nothing else in the pinball did

The BNCC's geometry habilidades keep naming the instrument, and the instrument is an editor:
«com o uso de malhas quadriculadas e de **softwares de geometria**», «usando instrumentos de desenho
ou **softwares de geometria dinâmica**», «utilizando material de desenho ou **tecnologias digitais**».
The document anticipated the tool; the tool simply has not existed here until now.

And the mediation is already the plan's headline feature. §4 is titled «Live checking, **which is the
feature**», and the Dev's own answer to what stops the editor producing broken tables was «THE EDITOR
HAS TO ANSWER ALL FIVE WHILE THE AUTHOR IS STILL LOOKING AT THE TABLE». A tool that tells a child her
construction does not work *while she is still building it* is scaffolding in the strict sense — she
cannot do it alone and she can do it with that answer in front of her.

### The tags, and what each depends on

| Code | Class | Condition |
|---|---|---|
| **`EF07MA21`** · «Reconhecer e construir figuras obtidas por simetrias de **translação, rotação e reflexão**, usando instrumentos de desenho ou softwares de geometria dinâmica» | **Works** | The editor's placement operations *are* translate, rotate and mirror. This is the closest fit in the whole document between a habilidade's verbs and a game's actual controls — but only once placing, turning and mirroring are the operations the child performs by name. |
| **`EF04MA19`** · «Reconhecer **simetria de reflexão** … e utilizá-la na construção de figuras congruentes, com o uso de malhas quadriculadas e de softwares de geometria» | **Works** | Requires a mirror tool. A table is usually symmetric and does not have to be; without an operation that mirrors, the child never uses symmetry, she merely ends up near it. |
| **`EF04MA18`** · ângulos retos e não retos com softwares de geometria; **`EF05MA17`** · polígonos, lados, vértices e ângulos, desenhados com tecnologias digitais | **Material** | Recognition tasks. The editor displays angles and polygons; it does not require the child to classify them. Promote only if the live check asks her to. |
| **`EF35EF01`** · «…jogos populares do Brasil e do mundo … **e recriá-los**» | **Works** *(was Material)* | The verb *recriar* is in the habilidade and is precisely what building a table and handing it to another child is. Sharing is what moves this from Material to Works: without it, she recreates nothing for anybody. |

> ⚠️ **All four are conditional and none is earned yet.** Nothing is tagged before it is built. This
> table is the target for the editor, not a description of the game that exists.

> ⚠️ **And do not conflate two reflections.** A ball bouncing off a wall is reflection in the physics
> sense; `EF04MA19` and `EF07MA21` are about reflection of *figures*. The trajectory is not the
> habilidade, however tempting the shared word is. What earns the tag is the child mirroring a shape,
> not the ball mirroring off it.

### What sharing raises, and what has to be answered before it is built

**It can be done with no server and no account**, which is the decisive fact. The plan's step 9
already has «Export: JSON to a file», and `ADR-0048`'s path (a) promises that nothing is saved and
nothing leaves the network. A table that serialises to a file — or to a short code, the same trick
the teacher's BNCC filter uses — keeps that promise intact. Sharing here does not need infrastructure;
it needs a format.

**But a table is child-authored content, and the records do not cover that.** A table carries a name,
and a name is free text written by one child and read by another. The brandbook's antipromessas
forbid behavioural profiling and predatory engagement; they say nothing about moderation, because
until now nothing in this product let one child send anything to another. A municipal product for
children cannot ship that gap unexamined. It is an open question, named here so it is answered before
it is built rather than after.

**The platformer has not been mapped.** It is the only one of the six carrying school curriculum
directly: eighteen activities in the engine's registry — `alf1`–`alf5` (Ferreiro's psychogenesis,
including Braille), `mat1`–`mat6` (quantity, addition, subtraction, times tables, division), six on
fractions — plus recycling to the CONAMA 275/2001 colour standard. It passes question 1 with room to
spare: the quiz gates the coin, at **two attempts before revealing and three wins per coin**
(`app/js/game/quiz.ts:18` and `:721`, «3 VITÓRIAS = 1 MOEDA em TODOS os minigames, sem exceção»).
Mapping it requires a pass against the Língua Portuguesa list that this review did not make. It is
recorded as debt rather than filled with guesses.

**No game carries a code today.** A regex sweep of all six repositories, excluding `node_modules`,
returns zero `EF__XX__` codes. Wherever these tags come to live, they do not live in the games yet.

## Source of the habilidade texts

Every enunciado quoted here was taken verbatim from a markdown conversion of the official document,
`BNCC_EI_EF_110518_versaofinal_site.pdf`, held on this machine at
`ROCK-Alfred/Research/test-samples/` as part of a PDF-extraction benchmark corpus — not as an
authoritative copy. Before any of these codes reaches a published card, the enunciados should be
re-checked against basenacionalcomum.mec.gov.br.
