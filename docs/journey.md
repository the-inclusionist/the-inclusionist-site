# The journey — from the door to the first mistake

The platform-layer user story. It starts when somebody opens the address and ends at the moment a
player gets something wrong, because that is where this product's pedagogy actually lives: the zone
of proximal development is measured in what happens **after** the error, not before it
(`ADR-0048` §5).

The engine's own stories are a different layer and live at
`the-inclusionist-engine/docs/1-Discovery/User-Stories.md`. Per-game stories belong in each game's
repository, by that document's own rule. This one is the connective tissue nobody had written.

## Who arrives

Four people open this address, and the records name three of them.

| Who | Role in the records | What they came for |
|---|---|---|
| **The child** | *player (child)* | To play. She never authenticates — `ADR-0048` §2 |
| **The teacher** | *teacher* | To put the right activity in front of thirty children in the ten minutes before the bell |
| **The parent** | *responsible adult* (with the professional — psychologist, physio, neuropsych) | To let a child play at home, and sometimes to see how it went |
| **The older player** | **not in the records** | To play. Eleven to fourteen years old, or older |

> ⚠️ **The fourth role has no home yet.** The engine's five roles are player (child), teacher,
> responsible adult, professional author and game developer. An adolescent playing for himself is
> none of those, and he is not a rounding error: the BNCC tags that cover these games at all —
> `EF67EF01`, `EF67EF02` — are 6º and 7º ano, which is eleven and twelve years old, and the brandbook
> commits to «brinca sem infantilizar». Somebody should decide whether he is a variant of *player*
> or a role of his own.

## The journey

### 1 · The door

All four arrive at **one address**, and what loads is one PWA — one service worker, one install, one
set of accessibility preferences that will follow the player from game to game (`ADR-0117`,
`ADR-0118`). On a second visit with no network it opens anyway; on the first visit it does not,
and that is the promise as written: «A PWA of this project is guaranteed to work offline **after the
first day**» (`ADR-0116`).

**Before anything else is on screen, the accessibility bar is.** Blind mode, sonar and palette are
mounted by the platform from the first frame, not by any game — that is `ADR-0106` §4 and the Dev's
instruction of 2026-09-07, and the reason it is the platform's job is that when it was each game's
job, five of six forgot.

### 2 · Screen 1 — what this session is for

Three ways in, and no child uses the second or third (`ADR-0048` §2):

- **(a) Choose an activity and play.** Nothing is saved, nothing leaves the network. «The promise
  that student data does not leave the school holds trivially, because no data exists.»
- **(b) Log in** — progress recorded in Bússola Escolar, end-to-end encrypted.
- **(c) Log in and point at a local IP** — Bússola Escolar on the school's own LAN.

**Today only (a) is being built.** That is the Dev's decision of 2026-09-11, and it decides the shape
of everything downstream: no login means no roster, which means the avatar screen of `ADR-0048` §1
does not appear, which means this journey has three screens and not four — exactly as that record
prescribes for this path.

On screen 1 the visitor picks BNCC skills. The teacher picks them for a class; the child or the older
player picks them for himself, or skips and browses.

### 3 · The code

The teacher's filter becomes a **short code**. A child types it and inherits the same filter.

This is the piece that makes a class work without a class existing. The code carries a **query, not a
person** — nothing about any student is encoded in it — so it can be written on the board and read
aloud without touching the data promise of (a). It is also the teacher's answer to the ten-minute
problem the Postmortem recorded: «O professor monta em casa e recebe um código de 6 caracteres. Em
sala, uma tela só: digitar o código e projetar. **As três telas viram o caminho de preparo, não o de
execução.**»

### 4 · Screen 2 — which game

The catalogue, narrowed by the filter. Each card says what it does with the skills that were ticked:
**Trabalha** when the game works the zone, **Serve de material para** when the game is valid material
and the work belongs to the teacher (see [curriculum/bncc-tags.md](curriculum/bncc-tags.md)).

A card also shows what it does **not** cover, so that four ticked skills and nine returned games is
not a puzzle the teacher has to solve by opening all nine. And a game with no accessibility audit is
not offered — it is shown, disabled, labelled as outside the catalogue.

The player picks whatever he wants. The filter narrows; it does not choose.

### 5 · Screen 3 — the game

Same origin, same document. **No new tab and no iframe** — the cartridge was installed into the
platform at build time, selected by the manifest (`ADR-0068` §2). What the player notices is that the
contrast setting, the typeface, the colour-vision correction and the remapped controls he set two
games ago are still the ones in force. That continuity is the entire argument for one origin.

### 6 · The mistake

And here the journey stops being one journey, because **the six games hold three different theories
of error.**

**Error as judgement** — the child asserted something and it was wrong.
- *whackwhack*: a lit tile carrying the wrong value is a `hazard`, not a miss. Hitting it is a
  multiple judged wrongly. The difficulty dial is declared **curricular** and kept orthogonal to the
  motor easy mode, and the level rises by tiles judged rather than by a clock — «A slow player is no
  longer hurried by a clock they cannot see» (`app/js/rules/round.ts:26`).
- *the platformer's quiz*: **two attempts before revealing, three wins per coin**, and the reward is
  set by the level of the last one, so a child who climbs mid-minigame is paid at the level she
  reached (`ADR-0048` §4).
- *chess*: a blunder shows what it cost. Three blunders **from the same position** — not three in a
  game — and the teacher turns itself on by itself, and says out loud that it did. «Three blunders
  spread over forty moves is a child learning at a perfectly normal rate; three in a row from one
  position is a child stuck, and the answer to being stuck is not a fourth chance to guess.»

**Error that does not exist.**
- *15-puzzle*: «THERE IS NO WRONG MOVE IN A SLIDING PUZZLE.» Every move is reversible in one keypress,
  nothing is spent, and the hint is therefore free, unlimited, uncounted and unpenalised — a lock
  would have to invent a penalty in order to have something to protect.
- *2048*: a bad move fills the board and the round ends. The round's score dies with the round and no
  record is kept, deliberately.

**Error as a lost turn.** *pinball*: the ball drains. Nothing curricular is being judged.

> ⚠️ **This is the finding the journey forces into the open.** `ADR-0048` §5 computes the zone from
> questions with alternatives and attempts — «8 or more of the last 10 right on the first try →
> PROFICIENT; 7 or more resolved using the second and third attempts → THE ZONE; 6 or fewer, or four
> failed questions in a row → FRUSTRATION». That arithmetic can be run on the platformer's quiz and,
> with work, on chess. **It cannot be run on a 15-puzzle session at all**, because the puzzle has no
> event that counts as a failed question.
>
> So «difficulty follows the child» is not a property the platform can compute for every game from a
> single signal. Either each cartridge declares what counts as an attempt and an outcome, or the
> adaptation applies only to the games that have questions — and the catalogue has to be honest about
> which those are. Nothing in the records decides this yet.

## Where this journey breaks today

Every line below is a story the engine already tracks as **not built** (⬜), and each is load-bearing
for the journey above:

- «As a **teacher**, I want a **room code** children can enter, so that a class shares a session» —
  this is step 3, and it does not exist.
- «As a **player**, I want the game to **notice when I am struggling and adjust**» — this is step 6.
  ⚠️ **The engine half of it is built**, and is more general than `ADR-0048` §5: `educational/adaptive-engine`
  implements the three bands (`proficiente` / `zona` / `frustracao`), says *why* through a `Motivo`, and
  takes the guess floor as a **parameter** rather than embedding the 0,60 of the five-alternative case —
  «uma atividade de duas alternativas tem piso 0,50 com uma tentativa; uma de dez com uma tentativa tem
  0,10». What is missing is the other end: its input is `ResultadoDaQuestao` — `'primeira' | 'mediada' |
  'falhou'`, «o que sobra de uma questão depois de as tentativas dela colapsarem» — and **nothing in the
  contract says which cartridges produce one.** A game without questions has nothing to hand it.
- «As a **player without a keyboard**, I want to **type a room code with the pad**» — step 3 on a
  device with no keyboard.
- «As a **responsible adult**, I want **one place** to set time, content and resources» — the parent
  of step 1 has nowhere to go.

And two that are not stories but holes: the **manifest** that step 5 depends on does not exist, and
**no game is installable** as a cartridge yet (see [hosting.md](hosting.md)).

## What this document does not decide

- Whether the older player is a role or a variant of *player*.
- What a cartridge declares as an "attempt" and an "outcome", which is what step 6 needs before
  difficulty can follow anybody.
- Whether this is promoted to an ADR, or lives here as the platform's story layer while the engine
  keeps its own.
