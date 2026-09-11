# Hosting — what is already decided, and what actually blocks delivery

## The question, and why it has less room than it looks

The question was: where is this best hosted, so that at the end of the journey a child clicks a game
and the 15-puzzle, 2048, chess, whackwhack, pinball or the platformer opens?

Most of that is settled, and the settled part is not a preference. It falls out of one browser fact:

> 📏 …because **Cache Storage is partitioned BY ORIGIN**.
> — `ADR-0117` (accepted)

One origin per game would mean one service worker per game, one precache per game, one set of
accessibility preferences per game — none of which follow the child from one game to the next. That
closes the option before taste gets a vote. `ADR-0068` had already rejected the same shape for the
same reason, in its option G4: «The child meets ONE product. Three hundred deployments is three
hundred URLs, three hundred service workers and three hundred accessibility settings that do not
follow her between games».

So the shape is fixed:

| Clause | Record |
|---|---|
| One origin, serving one platform | `ADR-0117`, `ADR-0118` |
| **A game is a cartridge, not a PWA** — it keeps its repository, version and licence; it stops being a unit of installation | `ADR-0117` §1 |
| The platform is **this repository** | `ADR-0118` — «A pasta the-inclusionist-demos é o que deveria ser the-inclusionist-site» |
| A game arrives as an npm package `@the-inclusionist/game-<slug>`, chosen by a manifest that names «game slug, version, and the reason it is in this release». «A build installs what the manifest names and nothing else» | `ADR-0068` §2, §3 |
| Zero submodules, zero git dependencies — «a manifest of git tags fetched at build time is a submodule with a different spelling» | `ADR-0068` §3, `ADR-0055`, `ADR-0083` |
| Interim public address `oinclusionista.jrocha.dev.br`, for demonstration | `ADR-0069` |
| Address of record: the Município's own servers, in national territory | `ADR-0058` §5 — «AND NO OTHER HOSTING HYPOTHESIS IS RECORDED, in this file or in any other» |
| Code is AGPL-3.0-or-later; the art is not | `ADR-0064` |

**Answer to the question as asked:** the child clicks, and the game opens **in the same origin, in
the same document — no new tab and no iframe.** No ADR mentions an iframe anywhere. The game is
already there because the platform's build installed it, selected by the manifest.

`ADR-0069` is worth reading before the address is treated as cosmetic: **in a PWA the origin is the
application's identity.** Moving from the interim subdomain to a municipal one is not a redirect; it
is a different application, with a different cache and a different installed app on the device. That
is why, while the address is interim, nothing the child produces may live only on the device.

---

## Three questions asked on 2026-09-11

### Can the engine be one PWA and each cartridge another, sharing the download?

**No.** Two cases, and both are closed.

**Across origins — impossible.** Cache Storage is partitioned by origin and there is no mechanism to
share across that boundary. This is option U2 in `ADR-0117`, rejected in exactly these words:

> 🔴 It cannot be built: Cache Storage is partitioned by origin, and modern browsers partition the
> HTTP cache by top-level site too. **There is no shared cache to share.**

**On one origin with different scopes — possible, and pointless.** Several service workers *can*
coexist on one origin under different scopes, and they do share Cache Storage, because that store is
per-origin. So the sharing would in fact work. What you would have built is one origin, one storage
quota, several manifests competing to be the thing the browser offers to install, several update
cycles to keep in step, and the child's accessibility preferences still living in exactly one place.
That is one application with extra machinery, described as several — you would pay the complexity to
buy the sharing you already had.

And the engine is not something a child installs. It is a dependency: published at
`@the-inclusionist/engine@8.0.0` and bundled by whatever consumes it.

> The follow-up question — *if every game depends on the engine, does the engine ship N times?* — has
> its own document: [architecture.md](architecture.md). Short answer: as the six repositories stand
> today, yes; three mechanisms fix it, and they are not alternatives.

### One PWA packaging all the games, then?

Almost — with one correction that matters, because it is the whole point of the manifest.

Not *all* the games. **The release's games.** `ADR-0068` §2: «A build installs what the manifest
names **and nothing else**» — and that record is emphatic that this is not a consolation prize but
«THE JOB THE BYTE BUDGET ALREADY REQUIRED».

And what the manifest names is **precached at install**, not fetched on first play. `ADR-0116` §2:
«precached at install, never fetched lazily at first use». `ADR-0117` ties the two together: «the
number of cartridges in the platform's precache must be **READ FROM THE MANIFEST** of `ADR-0068` §2».

> ⚠️ This supersedes decision **D11** of this repository's own 2026-08-25 spec, which said game chunks
> would be runtime-cached on first play and only the shell precached. D11 was written for a
> 383-game monorepo, where precaching everything meant «opening Snake would download the whole
> collection». With the manifest choosing a handful per release, the reasoning no longer applies.

### Cloudflare, a free AWS plan, or Magalu?

For a static PWA with no backend, all three are near-free. Price is therefore not the criterion, and
picking on price would be picking on noise. The criteria that bite are three:

1. **`ADR-0058` §5** — the systems run on municipal infrastructure, in national territory. Of the
   three, only Magalu is Brazilian.
2. **`ADR-0069`** — in a PWA the origin is the application's identity, so every move is a *new
   application*, with a new cache and a new installed app on the device.
3. **`ADR-0118`** — one origin is one thing a school network has to allow.

| | Cloudflare Pages | AWS (S3 + CloudFront) | Magalu Cloud (Object Storage) |
|---|---|---|---|
| Cost for this workload | free plan, no card | free tier that expires, then bills; card required | **billed from the first byte**: R$0,10/GiB stored per month + R$0,10/GiB egress, with R$100 of starting credit |
| CDN | yes, global | yes | **not in the public catalogue** (see caveat) |
| Data in national territory | not guaranteed | possible (`sa-east-1`), not inherent | **yes, by construction** |
| Billing in BRL | no | no | yes |

**The trap in the question is that none of the three is the destination.** `ADR-0058` §5 puts the
product on the Município's own servers, and says «AND NO OTHER HOSTING HYPOTHESIS IS RECORDED». So
whichever of the three is chosen today is interim by construction, and the right criterion for an
interim host is not *best* — it is **cheapest to leave**.

On that criterion Cloudflare Pages wins and it is not close: free, no card, custom domain, and a CDN
that matters more here than anywhere, because pillar 1's devices sit on school links. `ADR-0069`
already priced the eventual move and wrote down what it costs.

Choosing Magalu now buys data residency **for an address that is still interim**, and pays for it in
egress and in the CDN. The egress is affordable and worth knowing: `ADR-0124` records install day at
roughly 263 MB, which at R$0,10/GiB is about **R$0,026 per device** — around R$257 for ten thousand
installs, plus every re-install. The missing CDN is the larger cost, on exactly the hardware and
links pillar 1 names.

**Recommendation:** keep Cloudflare Pages for the interim demonstration address, and reopen the
provider question when the address stops being interim. At that point the contenders are the
Município's own servers and Magalu — and AWS is not among them, because it satisfies neither the
data-residency argument nor the cheapest-to-leave one.

> ⚠️ **Caveat on the Magalu figures.** The R$0,10/GiB storage and egress prices and the absence of a
> CDN in the public catalogue come from third-party summaries dated 2026 and from Magalu's own
> pricing page as quoted there — not from a direct reading of a contract. Before this decision is
> made on those numbers, confirm them with Magalu, particularly whether a CDN or edge-cache product
> now exists. A wrong reading here would send a real decision the wrong way.

---

## What actually blocks delivery

None of the following is a hosting-provider problem. They are the reasons a decision about the
provider would not, by itself, put a game in front of a child.

### 1 · No Cloudflare Pages project is attached to anything

> …the Dev measured otherwise on 2026-09-07 — **no Cloudflare Pages project is attached to any
> repository. While that holds, a push is not a deploy.**
> — `the-inclusionist-engine/docs/6-DevOps-SRE/README.md:18-19`

The migration checklist still carries the item: «Reapontar (ou recriar) o projeto do Cloudflare Pages
para o repositório do GitHub» (`CI-CD.md:98`). And there is a trap recorded beside it: the two
delivery paths are **mutually exclusive** — either Cloudflare is git-connected to the repository, or
CI direct-uploads with `wrangler`. Exactly one may be active; both at once deploys twice
(`CI-CD.md:71-78`).

*Unblocked by:* the Dev, in the Cloudflare panel. It cannot be verified from here.

### 2 · The manifest does not exist

`ADR-0068` §2 makes the manifest the unit of delivery and then declines to specify it: «⚠️ AND WHAT
THIS RECORD DELIBERATELY DOES NOT DECIDE: the slug convention, **the manifest's file format**…»
(line 240). Its own negative-consequences list already names the hole: «The catalogue's manifest
becomes the most important file in delivery, and **it has no gate today**» (line 118). `ADR-0118`
repeats that it is still open, and this repository's `README.md` says so in its own voice.

*Unblocked by:* a format, a schema and a validator. Proposed as the next round, if wanted.

### 3 · None of the six games is installable

This is the integration surface, so it is stated in terms of **remotes and the registry** — the
things a manifest can actually name — rather than in terms of local folders:

| Remote (`github.com/the-inclusionist/…`) | Package | On npm | `private` | `exports` / `files` |
|---|---|---|---|---|
| `the-inclusionist-engine` | `@the-inclusionist/engine` | **8.0.0 — published** | — | yes |
| `game-15puzzle` | `@the-inclusionist/game-15puzzle` | not published | `true` | — |
| `game-2048` | `@the-inclusionist/game-2048` | not published | `true` | — |
| `game-chess` | `@the-inclusionist/game-chess` | not published | `true` | — |
| `game-whackwhack` | `@the-inclusionist/game-whackwhack` | not published | `true` | — |
| `game-platformer` | `@the-inclusionist/game-platformer` | not published | `true` | — |
| **no remote at all** | `@the-inclusionist/game-space-cadet` | not published | `true` | — |

Two things fall out of that table.

**The mechanism is half built, and it is the better half that works.** `@the-inclusionist/engine` is
published at 8.0.0 and four of the games already consume it from the registry rather than from a
path. The dependency direction that `ADR-0068` §3 requires is real today. What does not exist is the
other end: no game publishes itself, so the manifest has nothing to name. Each one needs an entry
point the platform can import, a `files` list that travels, and `private` removed.

**The pinball has no remote.** 381 commits exist only on this machine, under a package name
(`game-space-cadet`) that matches neither the other five nor any repository. `ADR-0082` §1 makes the
repository the package name with the scope removed, so the slug has to be settled before either can
be created — and until a remote exists it cannot be a cartridge at all, whatever its code is worth.

The other five remotes are already correct, including `game-15puzzle` — only the local folder name
lags, which changes nothing for delivery.

### 4 · The PWA layer is inconsistent, and one README is wrong

- Only the **platformer** is a PWA. Its manifest hardcodes `"scope": "/"` and `"lang": "en"` — both
  wrong for a cartridge inside a platform served in pt-BR, and `scope: "/"` in particular would claim
  the whole origin.
- **2048's README line 5 claims «offline as a PWA»** and the build contains no service worker and no
  manifest. That is a false statement in a shipped document, not a missing feature.
- **No game sets `base` in its Vite config.** All six assume they own the domain root. Harmless while
  they are built into the platform; fatal the moment one is served from a subpath.

### 5 · The home page does NOT conflict with `ADR-0048` — resolved

An earlier draft of this document raised a conflict here. It was wrong, and the correction is worth
keeping because the mistake is easy to repeat: `ADR-0048` is titled *The journey is FOUR screens*,
and its §1 immediately qualifies the count.

> 2. WHO IS PLAYING — the avatar. ⚠️ **ONLY EXISTS WHEN AN ADULT IS LOGGED IN.** Without login there
>    is no roster to pick from, so screen 2 does not appear and **the journey goes straight to 3.**

So a three-screen journey is not a violation of `ADR-0048` — it is the shape `ADR-0048` prescribes
for the path with no login. That path is its own §2 (a): «CHOOSE AN ACTIVITY AND PLAY — nothing is
saved, nothing leaves the network. The promise that student data does not leave the school holds
trivially, because no data exists.»

**The Dev's decision of 2026-09-11:** build only the offline journey for now. No login, therefore no
roster, therefore three screens. Two additions:

**(a) The filter becomes a code.** The teacher filters by BNCC items and the filter yields a short
code; a student typing the code inherits the same filter. This is a good fit for the constraints
rather than a convenience: the code carries a *query*, not a child — nothing about a student is
encoded in it, so it can be written on a board and read aloud without touching the data promise. It
also lands beside machinery the prototype already has, which validates a room code with
`/^[A-Z]{3,6}-[A-Z0-9]{3,4}$/` and jumps straight to screen 2.

**(b) Avatars from `boring-avatars`.** Deterministic SVG generated from a seed string. This fits the
no-login path precisely — an avatar that is a pure function of a seed needs no account and stores
nothing, so it does not create the roster `ADR-0048` is guarding against.

> ⚠️ **One integration fact to decide before adopting the package.** `boring-avatars@2.0.4` is MIT —
> compatible — but it is a **React library**, declaring `react >= 18` and `react-dom >= 18` as peer
> dependencies. This project has no React anywhere: the games are PixiJS, Zdog, Three.js and vanilla
> DOM, and this repository is static HTML. Adding React to the platform for decorative SVG costs
> roughly 140 KB gzipped on hardware that pillar 1 defines as weak. The generator itself is small and
> MIT, so the practical route is to **port the generator rather than install the package**, keeping
> the attribution the licence requires. A `vue-boring-avatars` also exists and is equally not ours.

### 6 · The prototype is a design fiction, not a starting codebase

Adopting it as the home page means rebuilding most of it:

- It joins games to skills **by `topic`, never by BNCC code** — `gameData` entries carry
  `topics: ['fracoes']`, and the filter is `g.topics.some(...)`. The codes are display only.
- Its nine games are invented (`snake-fr`, `invaders`, `runner`…). None of the six real games appears.
- It draws the same Fraction Snake regardless of `gameId`; the `gameId` only changes the title.
- Its canvas is 360×180. The spec's D5 fixed **320×180** and rejected 360×180 explicitly — «not 16:9,
  no integer scaling».
- It loads five font families from Google Fonts, which its own Postmortem (finding 04) says to stop
  doing: «empacotar as fontes junto do produto, com subsetting por idioma, e servir tudo do próprio
  servidor local».

---

## Recommendation

Confirm `ADR-0069`, keep the interim address, and keep Cloudflare Pages under it. The provider
question is the least load-bearing part of this, and Cloudflare never had a record of its own — it
appears as a clause inside `ADR-0025` and `ADR-0026`, and `ADR-0026` was superseded whole by
`ADR-0066`. Writing that record can wait until the address stops being interim, which is the moment
the choice actually matters.

What cannot wait, in order:

1. **The manifest format** (§2) — nothing downstream can be built without it.
2. **Cartridge packaging** (§3) — `exports`, `files`, `private` removed, then published. The engine
   half already works; this is the other half.
3. **The pinball's remote and slug** (§3) — it cannot be named by a manifest until it exists on
   GitHub under a settled name.
4. **Attaching the Cloudflare project** (§1) — only the Dev can do this, and until it is done a push
   is not a deploy.

§5 and §6 are the home-page track and can run beside these.

## What this document does not decide

- The manifest's format — named as the next round, not settled here.
- The pinball's slug: `game-pinball` or `game-space-cadet`. Both the repository name and the package
  name follow from it (`ADR-0082` §1), so it is one decision, not two.
- Whether the `boring-avatars` generator is ported or the React package is installed.
- Whether any of this is promoted to an ADR. Records live in `the-inclusionist-docs` (`ADR-0123`),
  and writing there is the Dev's call. Two items here are ADR-shaped: the D11 supersession in
  *One PWA packaging all the games*, and the no-login journey of §5.
