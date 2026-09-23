# 03 · What a node is

A node has **no `TYPE`**. What a node *is* is derived from what it carries and where it sits in the graph — never
stored, never declared, never in disagreement with the payload:

| A node is… | iff | See |
|------------|-----|-----|
| **launchable** | it carries `ENTRYPOINTS` with an entry that has no `GUEST` | [ch. 9](09-exec.md) |
| **a runner** | it carries an `ENTRYPOINTS` entry with a non-empty `GUEST` | [ch. 9](09-exec.md), [ch. 11](11-runner-chaining.md) |
| **a title** (a tile of its own) | it carries `TILE` | §3.2 |
| **content / a mutator** | it carries any payload section | [ch. 5](05-layers.md), [ch. 6](06-edit-layers.md) |
| **a plain node** | it carries no payload — composition only | §3.1 |
| **a graft** | nothing in the launchable's composition lists it, and it is `OVER` something of the title | [ch. 12 §12.5](12-resolution.md) |
| **substance** (a library) | it reaches no `TILE` at all — it belongs to no title | §3.3 |
| **canonical** | a lint, not a kind: pristine content + `ENTRYPOINTS` + `TILE` and nothing else | [ch. 15](15-validation.md) |

This is the conclusion of "everything is a node": one primitive, one edge, and every role a *reading* of them.

## 3.1 The plain node

A node with no payload contributes nothing of its own and exists to gather other nodes under one name:

```json
{ "LABEL": "morrowind_data",
  "OVER": ["morrowind_textures_hd", "morrowind_bloodmoon", "morrowind_tribunal"] }
```

It is not cosmetic and an implementation MUST NOT optimise it away: something points at it. Dropping a payload-less
node makes every referrer *silently lose an edge* rather than dangle.

## 3.2 `TILE` — identity lives on the variants

There is no tile node. A **launchable carries its own `TILE`**, and the launcher, the catalog and the share sheet
**group by `UID`**: one UID = one card. Everything else inherits its identity through `OVER`.

```json
{ "LABEL": "1.16.5", "OVER": ["…v1.16.4"],
  "TILE": { "UID": "minecraft", "TITLE": "Minecraft", "COVER": { "PATH": "cover.png", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } },
  "LAYERS": [ { "FORM": "delta", "PATH": "1.16.5.vgdelta", "TARGET": "…" } ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "java8", "PATH": "", "ARGS": ["-cp", "%MC_CP%", "net.minecraft.client.main.Main"] } ] }
```

| Field | Type | Meaning |
|-------|------|---------|
| `UID` | string | Stable identity for *the title*. Keys saved state, settings and the content root inside a prefix, and is the card the node appears under. **Required** on a `TILE`. |
| `TITLE` | string | The card's name. Falls back to `LABEL`. |
| `COVER` | object \| string | `{ "PATH": …, "SOURCE": { "TYPE": "ipfs", "CID": … } }` (or a bare filename while authoring). Cover CIDs are published and seeded like content. |
| `META` | object | Free-form descriptive metadata. Implementations MUST ignore keys they do not know. |

**Identity is derived** (`DeriveIdentity`): a node's identity is its own `TILE.UID` if it carries one, else the
**union** of its positive `OVER` requirements' identities, in `OVER` order — a mod `OVER [["aok", "conq"]]` belongs
to both titles; a node reaching no tile has none (it is substance). A node with its own `TILE` is its own identity
**and nothing under it leaks upward**: *Conquerors* `OVER [aok, conq-disc]` is Conquerors only, so an AoK mod does
not appear under Conquerors, and a mod `OVER [conq]` belongs to Conquerors only. A total conversion is the same
shape: its own `TILE`, the base game underneath.

**Nesting inside a card is derived from the chain, never declared.** Among the launchables of one UID, the
**main** is the one that is `OVER` no other launchable of that UID. A launchable `OVER` the main that carries a
*different* `TITLE` or `COVER` is a **child** — an expansion (*The Conquerors* `OVER` *Age of Kings*, *The Frozen
Throne* `OVER` *Reign of Chaos*), shown nested under the main with its own cover. One with the *same* tile is a
**variant** — an edition or a version (Minecraft's 903, each `OVER` the previous, all one tile), shown in the
picker. The card takes the main's `TITLE` and `COVER`; the default launch is the `RECOMMENDED` entry wherever it
sits. Everything else — content root, saves, settings — keys on the shared UID, so an expansion needs no field
of its own. A UID with two mains (two independent installs) shows both at top level; validators note it.

A launchable that reaches no tile appears under no card (validators warn; a runner legitimately has none).

**Sharing** follows: a share is a set of **root CIDs**; the receiver reads each root's `TILE` — or its game's, one
`OVER` hop away — and lands it under the same card. The same UID means the same card on every machine, so a mod
shared alone lands under its game by itself. See [ch. 4](04-bundles-and-library.md).

## 3.3 Substance — libraries the games declare

A library (dgVoodoo, a codec stack, a mod loader's runtime) carries no `TILE` and is `OVER` nothing: **it cannot
know every game in existence**, so the game side declares it. A node with no identity is never listed under a
card and never a choice; it is *substance*, pulled into a mount by whatever names it:

```json
{ "LABEL": "dgVoodoo 2.81", "LAYERS": […], "DLLOVERRIDES": { "d3d8": "n,b" }, "VARS": […] }

{ "LABEL": "GL wrapper", "OVER": ["…tonic", "…dgvoodoo"], "TOGGLE": "on",
  "VARS": [ { "KEY": "DGVOODOO_TARGET", "DEFAULT": "…" } ], "FILEEDITS": [ … ] }
```

The library is shared by content (every game names the same CID), never by enumeration; a library update is a new
CID that each game's node moves to on its own schedule.

## 3.4 Variants — (node, entrypoint)

A launchable's `ENTRYPOINTS` entries are its **variants**: *Play*, *Multiplayer*, *Editor*. Several launchables
under one UID are variants of the card too: the picker lists launchables × entrypoints. A launchable that is a
graft (SKSE `OVER [["640", "659"]]`, Forge `OVER ["1.16.5"]`) is picked from the card like any other; its any-of
group is the version choice. `RECOMMENDED: true` on an entry marks the default. Execution is **never transitive**: a
version under the one you picked contributes no entrypoint.

> *Historical note.* Generation 1 had a `ROLE`; generation 2 collapsed roles into `Declare*` layers and layers into
> typed nodes (`DeclareExec`, `DeclareLibraryItem`, `Group`, ten `TYPE`s in all) with `PARENTS`/`EXCLUDE`/`LIBRARYITEM`
> edges. Generation 3 — this one — collapsed the types into one pluripotent node and the edges into `OVER`. Each step
> removed a construct the previous one had needed only because of the step before it.

Next: [Bundles, the library & indexing](04-bundles-and-library.md).
