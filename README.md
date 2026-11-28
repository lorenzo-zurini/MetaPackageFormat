# MetaPackage Format (MPF)

**A platform-agnostic, content-addressed format for packaging, composing, sharing and running software: a node is
an ordered list of layers, and a node's parents are layers too.**

> ### One primitive, for all software, on any host.
>
> Every existing format hard-codes an assumption about what the payload is and where it runs, and then spends its
> life apologising for it. MPF makes one structural commitment, **a node is an ordered list of typed layers**, and
> derives the rest: what a thing is, what it runs on, what it is built from, what is optional, what state is the
> user's, and how to obtain it. A SNES ROM and a Wine runtime are the same kind of object. "Runs on" is a path
> through a graph, not a field. Nothing is installed, so nothing is lost.

MPF describes software, from any platform on any host, as small JSON files called **nodes**. A SNES ROM, a Windows
game, a native Linux binary, an emulator, a Wine/Proton runtime, a mod, an expansion, a configuration option and a
registry write are all expressed in nodes. The format is the nodes and the rules for resolving, composing and running
them.

This repository is the **specification**. It is implementation-independent: the format is defined here, not in any
one program. The reference implementation is [**VidyaGod**](https://github.com/lorenzo-zurini/VidyaGod), a Qt/C++
launcher, and the spec often notes *how* VidyaGod realises a rule. Those notes are illustrative. **The spec is
normative; the implementation follows it.** A determined reader should be able to build a working MPF runtime from
this document alone: node identity, resolution, the overlay runtime, runner chaining, ownership of user state and
content distribution.

---

## Build

Nothing to build: this repository is the specification, in plain Markdown, on every platform. Read
[`docs/00`–`24`](docs/) in order, or start with [the model in one page](#the-model-in-one-page) below.

To build the reference implementation instead, see
[VidyaGod](https://github.com/lorenzo-zurini/VidyaGod#build) (Linux and Windows).

---

## Why another format?

Existing formats each bake in an assumption the others cannot shed:

- **App images and containers** (AppImage, Flatpak, Docker) assume the payload is native to the host. They cannot say
  "this Windows game runs under Proton on Linux", let alone "this SNES ROM runs under a Windows emulator under Proton".
- **Game manifests** (Steam, GOG, Lutris and Heroic scripts) tie the description to a storefront, an installer or a
  scripting language. They are recipes, not a portable, declarative data format.
- **Archives** (zip, tar) carry bytes and no meaning: nothing about what runs, what it runs on, what it is built
  from, what is optional or what state must survive.

MPF's bet is that these collapse into one model:

> A piece of runnable software is a **node**: an ordered list of layers, bottom to top. A layer is files, another
> node (selected and placed), an edit to a file, a registry tree, variables, environment, DLL policy, a way to run,
> a statement of what the user owns, or a check. Later layers win. "What it runs on" is a path through a **platform
> graph** of runners, found by capability. "How to obtain it" is a content ID on every file.

From that, a lot falls out with no special case in the format: mod load order, optional expansions, multi-edition
games, cross-platform execution, peer-to-peer distribution, portable saves and reproducible runtimes.

---

## The model in one page

- **A node is content.** A JSON file, `{"LABEL": …, "VARIANT"?: …, "SECTION"?: …, "LAYERS": [ … ]}`. Its
  identity is the CID of its canonical bytes. Every reference to another node is that CID, as a string.
- **Parents are layers.** There is no parent edge. A node's ancestors (the previous version, the game a patch is for,
  the libraries it uses) are `NODE` layers in its own list, at the position it chooses, selected with `TAKE` and
  placed with `TARGET`.
- **A layer is one typed entry.** Its type is its one payload key: bytes (`ZIP` `FILE` `DELTA` `DIR`), another node
  (`NODE`), a part's base (`INTO`), an edit (`EDIT`), facts (`REG` `VARS` `ENV` `DLL` `EXEC`), ownership (`KEEP`), or
  a stream coded back from its content (`CODED`). Any layer may carry a `WHEN` condition.
- **Everything a layer writes has an address**: `FILES/…`, `FILES/<file>#<key>`, `REG/<key>`, `REG/<key>#<value>`,
  `VARS/…`, `ENV/…`, `DLL/…`, `EXEC/<label>/…`.
- **Composition by history.** Walk the list; a `NODE` layer brings the contained node's layers in place. Per address,
  the later writer wins, except that a value a layer only *carries* never overrides one already composed; an edit
  applies once, to what is beneath it. A node never rises above the node that contains it.
- **Nothing folds.** Every address is written whole: a later write of `EXEC/Play` replaces the entry, a later
  `REG/<key>#<value>` the value. Part of something is changed by writing (or taking) its deeper address.
- **Paths are anchored, not lettered.** A package writes `%GameDir%/Data`, `%Documents%/My Games`, `%SysDir32%`. The
  runner that runs it says where each anchor is in its guest (`C:\766`, `C:\users\steamuser\Documents`) and where
  each drive lives on disk. No drive letter appears in a package.
- **A tile rides its entry.** `TILE {UID, PARENTUID, TITLE, COVER, META}` sits on an `EXEC` entry. A **variant** is a
  node marked `VARIANT`; a shelf row is a variant × a tiled entry. One row per card says `"RECOMMENDED": true` on its
  entry: the card's default, and the row its facts come from; the others keep only `{"UID"}`.
- **Grafts are found, not referenced.** A node whose list begins with `INTO` layers is a part: offered to every
  resolution that contains one of the nodes it names, applied in the instance's order; `"RECOMMENDED": true` on its
  `INTO` layer ticks it by default. A part with an entry is a row (its own card, or under the tile it takes).
- **Ownership, not override.** Everything is the package's unless `KEEP` gives it to the user; the most specific
  address wins. The user's state is captured after a run and restored before the next.
- **One home per node.** A node lives in one package; others contain it by CID. Shared things are packages.
- **Runners are never referenced.** The runner chain is found by capability: the shortest path of runners from the
  entry's platform to the machine's. An engine for a family of games is a runner on a canonical platform token; an
  engine for one game is `custom`, in that game's package.

A console whose only emulator is a Windows program, on Linux:

```
   Vortex Quest   EXEC {HOST "vortex"}              the ROM, and how to run it
        │
        ▼ the runner chain, found by capability (shortest path, installed runners first):
   vortex ──[vortexemu.exe]──▶ win32 ──[Proton]──▶ linux64 ──[native]──▶ executed
```

The packager declares facts only: "this is Vortex content", "vortexemu is a win32 program that runs Vortex content",
"Proton is a linux64 program that runs win32 programs". The runtime derives the route. Add an ARM runner and ARM
hosts work with no package changes.

---

## How to read this document

Read the chapters in order the first time.

| # | Chapter | What it defines |
|---|---------|-----------------|
| 00 | [Glossary](docs/00-glossary.md) | Every term used normatively. |
| 01 | [Overview & design model](docs/01-overview.md) | The model, its goals and its invariants. |
| 02 | [Nodes](docs/02-nodes.md) | The node file: fields, identity, the working tree. |
| 03 | [Layers & addresses](docs/03-layers-and-addresses.md) | The layer types, the fields every layer may carry, addresses, anchors. |
| 04 | [Content layers](docs/04-content-layers.md) | `ZIP` `FILE` `DELTA` `DIR`: bytes, `SOURCE`, `TARGET`, stacking. |
| 05 | [`NODE` layers](docs/05-node-layers.md) | Containment, `TAKE` and `TARGET`, composition. |
| 06 | [`EDIT` layers](docs/06-edit-layers.md) | Text and binary edits to one file. |
| 07 | [`REG`, `DLL`, `ENV`](docs/07-registry-dll-env.md) | The registry tree, DLL load policy, the process environment. |
| 08 | [Variables & `WHEN`](docs/08-variables.md) | `VARS`, the launch window, render formats, conditions. |
| 09 | [`EXEC`, tiles & variants](docs/09-exec-tiles-variants.md) | How to run, what the shelf shows, what can be picked. |
| 10 | [Platforms & runners](docs/10-platforms-and-runners.md) | Platform tokens, runners, anchors and drives, `custom`. |
| 11 | [Runner chaining](docs/11-runner-chaining.md) | The chain, the native terminal, nesting across namespaces. |
| 12 | [Resolution](docs/12-resolution.md) | Phases, membership, composition, grafts, the instance. |
| 13 | [Ownership: `KEEP`](docs/13-ownership.md) | What the user owns, how it is captured and restored. |
| 14 | [The runtime model](docs/14-runtime-model.md) | The overlay mount, the session, the sandbox, teardown. |
| 15 | [Packages, CIDs & sharing](docs/15-packages-and-sharing.md) | Package folders, publishing, receiving, pinning. |
| 16 | [Validation](docs/16-validation.md) | What a validator refuses and what it warns about. |
| 17 | [Run modes, instances & the CLI](docs/17-cli-and-run-modes.md) | Data roots, instances, the headless surface. |
| 18 | [Worked examples](docs/18-examples.md) | Complete packages, from a native game to a daisy chain. |
| 19 | [Reference implementation map](docs/19-reference-implementation.md) | Where each rule lives in VidyaGod. |
| 20 | [Conformance](docs/20-conformance.md) | The checklist a from-scratch implementation must satisfy. |
| 21 | [Appendix: platform tokens](docs/21-platform-tokens.md) | The canonical tokens, engine kinds, naming rules. |
| 22 | [Decoding deltas (DRAFT)](docs/22-decoding-deltas.md) | Not normative yet: pieces, decoded forms, tree deltas, declarative two-way descriptions, the no-cache runtime, measurements. |
| 23 | [Mods](docs/23-mods.md) | Parts, conditions on node CIDs, conflicts and resolutions, graft cards. |
| 24 | [Format atlas](docs/24-format-atlas.md) | Every codec, archive, installer and disc filesystem met, how MPF reads it, and what each primitive unblocks. |

---

## Status & versioning

This document describes **MPF generation 9**: a node is an ordered list of typed layers, its parents are layers, and
a resolution is a composition by history in which nothing folds. Earlier generations are obsolete and appear here only
as history:

| gen | Shape | Why it went |
|-----|-------|-------------|
| 1 | one `MANIFEST.json` with `SUBGAMES`/`COMPONENTS` arrays | the graph was implicit and could not be addressed |
| 2 | a node with a `ROLE` and an ordered `LAYERS[]` array | identity collapsed into `Declare*` layers; nothing could reference one layer |
| 3 | a node is one layer: `TYPE` on the node, order carried by `PARENTS` | one node per registry write; the types multiplied |
| 4 | one untyped node kind, one edge `OVER`, tiles on launchables, grafts | the edge carried requirements and composition at once |
| 5 | facts fold, choices don't; the tile at the base; `VARIANT` declared | the parent edge still could not select or place |
| 6 | parents are layers (`NODE` + `TAKE` + `TARGET`); one fold over addresses; `KEEP` ownership; tiles on entries; anchors instead of drive letters | grafts by `ANY`/`NOT` could not say what a part goes into |
| 7 | the library as a git history (2026-09-24) | shelved: a design, never the format |
| 8 | generation 6 with `INTO` parts, conditions naming node CIDs, `CODED` layers (2026-09-25 → 11-27) | entries folded per field, re-mentions were held or moved by position, `RECOMMENDED` named tiles from the node |
| **9** | **composition by history (provided vs carried values); nothing folds (every address written whole, `EXEC` a tree keyed by label); `RECOMMENDED` on the entry and on a part's `INTO`; `REG` as backslash key paths; conditions decided on the final composition** | current |

What generation 9 changed:

- **Composition** ([05 §5.4](docs/05-node-layers.md)): an occurrence's own layers are *provided*, what it brings of
  other nodes is *carried*. A carried copy of something already composed never comes back; a provided re-mention moves
  that node's own layers to the later place (a validator warns when that overrides a newer value). An edit applies
  once. Gen 8's held/moved positional rule is gone.
- **Nothing folds** ([09 §9.2](docs/09-exec-tiles-variants.md)): `EXEC` is `{label: entry}`; a write of
  `EXEC/<label>` replaces the entry whole, and `TAKE` reaches any part of it (`EXEC/Play/TILE/COVER`).
- **Recommended** ([09 §9.3](docs/09-exec-tiles-variants.md)): `"RECOMMENDED": true` on an entry (one per card) or on
  a part's `INTO` layer (ticked by default). Node-level `RECOMMENDED` is gone.
- **Registry** ([07](docs/07-registry-dll-env.md)): `{"HKCU\\Software\\X": {value: data}}`, with an optional layer
  `KEY`; addresses `REG/<key>` and `REG/<key>#<value>`.
- **Conditions** ([08 §8.6](docs/08-variables.md), [12](docs/12-resolution.md)): a condition naming a node CID is
  decided on the final composition, found as a fixpoint; one that defeats itself is refused.

Where this spec and older material disagree, **this spec wins**. The format is pre-1.0. The keywords **MUST**,
**MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** have their RFC 2119 meanings. Invariants **I1**–**I10** are
stated in [chapter 1](docs/01-overview.md) and referenced throughout.

---

## License & contributions

The format is open. This repository is documentation; contributions that clarify, correct against the reference
implementation, or add worked examples are welcome. When the spec and the reference implementation diverge, file it:
one of them is wrong, and saying which is the point of a written spec.
