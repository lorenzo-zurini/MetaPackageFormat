# 03 · What a node is

A node has **no `TYPE`**. What a node *is* is derived from what it carries and where it sits in the graph — never
stored, never declared, never in disagreement with the payload. Two facets are declared and everything else is a
reading of the chain:

| A node is… | iff | See |
|------------|-----|-----|
| **runnable** | it has *effective* entrypoints without `GUEST` — its own `ENTRYPOINTS`, or those of the nearest node beneath it | [ch. 9](09-exec.md) |
| **a variant** | it is runnable and carries `VARIANT` — it is listed on its card and can be picked | §3.4 |
| **a runner** | its effective entrypoints have a non-empty `GUEST` | [ch. 9](09-exec.md), [ch. 11](11-runner-chaining.md) |
| **a face** (a title, or part of one) | it carries `TILE` | §3.2 |
| **content / a mutator** | it carries any payload section | [ch. 5](05-layers.md), [ch. 6](06-edit-layers.md) |
| **a plain node** | it carries no payload — composition only | §3.1 |
| **a graft** (relative to a selection) | it has the selected face's identity, is not a variant, and is not in the selected variant's closure | [ch. 12 §12.3](12-resolution.md) |
| **substance** | no face and no runner is beneath it — it belongs to nothing | §3.3 |
| **canonical** | a lint, not a kind: a face over pristine content and nothing else | [ch. 15](15-validation.md) |

This is the conclusion of "everything is a node": one primitive, one edge, two declared facets (`TILE`, `VARIANT`),
and every role a *reading* of them.

## 3.1 The plain node

A node with no payload contributes nothing of its own and exists to gather other nodes under one name:

```json
{ "LABEL": "morrowind_data",
  "OVER": ["morrowind_textures_hd", "morrowind_bloodmoon", "morrowind_tribunal"] }
```

It is not cosmetic and an implementation MUST NOT optimise it away: something points at it. Dropping a payload-less
node makes every referrer *silently lose an edge* rather than dangle.

## 3.2 `TILE` — a face, at the base of what it names

There is no tile node. A `TILE` is a **face**: the UID, title and cover of a thing on the shelf. It sits where the
thing *begins* — on the pristine content — and **identity ascends from it**: every node built on it (transitively
`OVER` it) belongs to that title. The launcher, the catalog and the share sheet group by `UID`: one UID = one card.

```jsonc
// the pristine: content and the face, nothing else — the canonical node of the title
{ "LABEL": "minecraft", "TILE": { "UID": "320", "TITLE": "Minecraft", "COVER": { "PATH": "cover.png", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } },
  "LAYERS": [ { "FORM": "file", "PATH": "rd-132211.jar", "TARGET": "client" } ] }

// a version: a delta over the one before, listed as a variant; its face, its identity and (unless it declares
// its own) its entrypoints all come from beneath
{ "LABEL": "1.16.5", "VARIANT": "1.16.5", "OVER": ["…v1.16.4"],
  "LAYERS": [ { "FORM": "delta", "PATH": "1.16.5.vgdelta", "TARGET": "client" } ] }
```

| Field | Type | Meaning |
|-------|------|---------|
| `UID` | string | Stable identity for *the title*. Keys saved state, settings and the content root inside a prefix, and is the card every node built on this tile appears under. **Required** on a `TILE`. |
| `TITLE` | string | The face's name. Falls back to `LABEL`. |
| `COVER` | object \| string | `{ "PATH": …, "SOURCE": { "TYPE": "ipfs", "CID": … } }` (or a bare filename while authoring). Cover CIDs are published and seeded like content. |
| `META` | object | Free-form descriptive metadata. Implementations MUST ignore keys they do not know. |

Two tiles whose fields are equal are **the same face**, however many nodes carry them.

**Identity is derived** (`DeriveIdentity`): a node's **face** is the nearest tile beneath it (its own counts as
distance zero; nearest by `OVER` distance, ties by `OVER` order — a validator warns when the tied candidates
differ); its identity is the union, across branches, of the faces' UIDs. A mod `OVER [["aok", "conq"]]` belongs to
both titles; a node with no tile beneath it belongs to nothing and is substance. For a node whose nearest
pickable thing beneath it is a **runner** rather than a tile, the runner plays the part of the face: that is how a
graft on a runner is offered ([ch. 12](12-resolution.md)).

**Nesting inside a card is containment.** Within one UID, a tile with **no same-UID tile beneath it** is the
card's **main** face; a tile with one beneath it is a **child** face, nested under whichever face is nearest
beneath it, as deep as the chain goes. *The Conquerors* carries its own tile on its base zip, which is `OVER` the
*Age of Kings* pristine that carries the Age of Kings tile: a child. *Forgotten Empires* over that: a grandchild.
The card takes the main face's `TITLE` and `COVER`; each child shows its own. A UID with two main faces (two
independent installs sharing a UID) shows both at top level; validators note it.

Everything else — content root, saves, settings — keys on the shared UID, so an expansion declares nothing beyond
its own tile. Nothing under a tile leaks upward *across faces*: an Age of Kings mod is not offered under The
Conquerors, because offers are judged against the selected variant, not the card ([ch. 12](12-resolution.md)).

**Sharing** follows: the unit shared is the **package** (a bundle dir). Publishing mints one **package manifest**
block per bundle — a link to every node block in it — and a share is the set of package manifests of a library;
there is no per-node flag and no library-level block (a library is a name that groups packages; a CID over the
whole library would change whenever any package did, and the package is what changes). The receiver lands each
manifest and every node block it names — blocks only, kilobytes per game; content stays lazy until install — so it
holds the same graph the sharer holds and derives the same things from it: the card and its faces, which game a
mod belongs to, what a graft needs. Nothing relational is declared in the share; the graph says it. A package
manifest is also the unit a pinning service pins: its closure is the package once, and only a changed package
re-pins. See [ch. 4](04-bundles-and-library.md).

## 3.3 Substance — libraries the games declare

A library (dgVoodoo, a codec stack, a mod loader's runtime) carries no `TILE` and is `OVER` nothing: **it cannot
know every game in existence**, so the game side declares it. A node with no identity is never listed under a
card and never a choice; it is *substance*, pulled into a mount by whatever names it:

```json
{ "LABEL": "dgVoodoo 2.81", "LAYERS": […], "DLLOVERRIDES": { "d3d8": "n,b" }, "VARS": […] }

{ "LABEL": "GL wrapper", "OVER": [["…tonic-sp", "…tonic-mp"], "…dgvoodoo"], "TOGGLE": "on",
  "VARS": [ { "KEY": "DGVOODOO_TARGET", "DEFAULT": "…" } ], "FILEEDITS": [ … ] }
```

The library is shared by content (every game names the same CID), never by enumeration; a library update is a new
CID that each game's node moves to on its own schedule. Note the second node: a graft over either Tonic Trouble
variant that pulls the library in as substance — the library never learns about it.

## 3.4 Variants — what can be picked

A **variant** is a runnable node that declares `VARIANT: "name"`. Variants are the only nodes a card lists, and
picking one **selects exactly that node** — nothing beneath it, however much it inherits ([ch. 12](12-resolution.md)).
A variant's *entries* (its effective `ENTRYPOINTS`: *Play*, *Multiplayer*, *Editor*) are the ways to run it; the
picker lists variants × entries, grouped by face.

Runnable is wider than variant on purpose: **any node with an effective entry can be started** — from the CLI,
by an author testing a chain at the no-CD patch level, by a tool — without being on the shelf. Declaring the
variant is what puts it on the shelf.

`RECOMMENDED: true` on a node means *prefer me among my siblings*: among the variants of one face, the default the
card opens on; among the runners serving a platform, the default runner. Among a node's entries the first is the
default. Faces have no recommendation among themselves: a card opens on its main face.

> *Historical note.* Generation 1 had a `ROLE`; generation 2 collapsed roles into `Declare*` layers and layers into
> typed nodes with `PARENTS`/`EXCLUDE`/`LIBRARYITEM` edges; generation 3 collapsed the types into one pluripotent
> node and the edges into `OVER`; generation 4 put the tile on the launchables. Generation 5 — this one — put the
> tile at the base where identity begins, declared the variant, and made the closure a pure conjunction so that
> every choice is a variant or a graft. Each step removed a construct the previous one had needed only because of
> the step before it.

Next: [Bundles, the library & indexing](04-bundles-and-library.md).
