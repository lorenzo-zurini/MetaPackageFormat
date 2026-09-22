# MetaPackage Format (MPF)

**A universal, platform-agnostic, content-addressed software distribution format built on a single primitive: the node.**

> ### One primitive, for all software, on any host, forever.
>
> Every existing format hard-codes an assumption about what the payload is and where it runs, and then spends its
> life apologising for it. MPF makes exactly one structural commitment — **a node is one layer** — and derives the
> rest: what a thing is, what it runs on, what it depends on, what is optional, what state survives, and how to
> obtain it. A SNES ROM and a Wine runtime are the same kind of object. "Runs on" is a path through a graph, not a
> field. Nothing is installed, so nothing is lost.
>
> The bet: a format small enough to hold in your head, and general enough that new capabilities *fall out of it*
> rather than being bolted on, will outlive every format that shipped a special case for each of them.

MPF describes how to package, compose, share and *run* software — any software, from any platform, on any host — as a
graph of small, globally-referenceable JSON nodes. A SNES ROM, a Windows game, a native Linux binary, an emulator, a
Wine/Proton runtime, a mod, an optional expansion, a configuration knob, a single registry write: all of them are
*nodes*. The format is the graph and the rules for resolving, composing and executing it.

There is exactly one structural primitive and it is deliberately small: **a node is one meaningful change**. It has a CID,
any subset of the payload sections, and the one edge `OVER`. Nothing nests, nothing has a type. Order is not an array index — it is an edge.

This repository is the **definitive specification** of that format. It is implementation-independent: the format is
defined here, not in any one program. The reference implementation is [**VidyaGod**](https://github.com/lorenzo-zurini),
a Qt/C++ launcher, and this document frequently points at *how* VidyaGod realizes a given rule — but those notes are
illustrative. **The spec is normative; the implementation follows it.** A sufficiently determined reader should be able
to reconstruct a working MPF runtime — graph indexing, dependency resolution, the overlay-filesystem runtime, runner
daisy-chaining, content fetching and persistence — from this document alone.

---

## Build

Nothing to build — this repository is the specification, and it is plain Markdown. Read
[`docs/00`–`19`](docs/) in order, or start with the [thirty-second mental model](#the-thirty-second-mental-model) below.
Every platform, no toolchain.

To build the reference implementation instead, see
[VidyaGod](https://github.com/lorenzo-zurini/VidyaGod#build) (Linux and Windows).

---

## Why another format?

Existing formats each bake in assumptions the others can't shed:

- **App images / containers** (AppImage, Flatpak, Docker) assume the payload is *native* to the host. They can't express
  "this Windows game runs under Proton runs on Linux," let alone "this SNES ROM runs under a Windows emulator under
  Proton." The *platform abstraction* is hard-coded and shallow.
- **Game manifests** (Steam, GOG, Lutris/Heroic scripts) tie the description to a storefront, an installer, or a host
  scripting language. They are recipes, not a portable, declarative data format.
- **Archive formats** (zip, tar) carry bytes but no semantics: no notion of *what runs*, *what it runs on*, *what it
  depends on*, *what is optional*, or *what state must survive*.

MPF's bet is that all of these collapse into **one graph model**:

> A piece of runnable software is a **graph of nodes**, where each node is exactly **one layer** — files, a registry
> edit, a config patch, a persistence rule, a knob, a declaration of what to run — and each node's `OVER` says both
> what it depends on and what is applied before it. "What it runs on" is just an edge in a **platform graph** of
> runners. "How to obtain it" is a **content-addressed source** on each node.

From that single idea, a remarkable amount *falls out for free*: mod load-order, optional DLC, multi-edition games,
cross-platform execution, P2P distribution, portable saves, and reproducible runtimes — none of them are special cases
in the format, they're all just shapes of the same graph.

Making order an *edge* rather than an array index is what buys the last of that. A mod can depend on **one** layer of a
chain. A capture can be parented at **one** point in it. And "these two writes are unordered" becomes a statement the
format can make — and a validator can check — rather than an accident of who happened to be listed second.

---

## The thirty-second mental model

Take a (hypothetical) console, the *Vortex*, whose only emulator `vortexemu` is a **Windows** program — there is no
native-Linux build:

```
            ┌──────────────────────────────────────────────────────────┐
            │                    THE GLOBAL NODE GRAPH                    │
            │                                                            │
   runner ──┤   Vortex Quest  (ENTRYPOINTS, HOST = "vortex")           │
   edges    │        │ OVER                                              │
   GUEST→HOST│       ▼                                                   │
            │   vortex_quest_rom   (Content, FORM "file": the ROM)       │
            └──────────────────────────────────────────────────────────┘

   To RUN vortex_quest on a linux64 machine, the runtime builds the SHORTEST
   chain of runner edges from the content's platform to the machine's:

      vortex ──[vortexemu.exe]──▶ win32 ──[Proton]──▶ linux64 ──[native]──▶ EXECUTED
              (a win32 emulator)        (a Wine runtime)     (the terminal)

   …mounts every layer into one overlay filesystem, translates paths across each
   namespace boundary, and execve's a single nested command. Platforms: abstracted.
```

That chain is not authored by hand. The packager declares only *facts* — "this is Vortex content," "vortexemu is a win32
program that runs Vortex content," "Proton is a linux64 runtime that runs win32 programs" — and the runtime *derives* the
route. (When a platform *does* have a native runner, the chain is just one hop; daisy-chaining across a foreign platform
happens only when there's no shorter route.) Add an ARM runner tomorrow and ARM hosts light up with no package changes.

---

## The node: sections, not types

| section | The node carries… |
|---------|-------------------|
| `LAYERS` | files mounted into the runtime — a zip, a directory, a single file, or a binary delta over one |
| `REGEDITS` | registry keys and values written into the prefix, per architecture |
| `FILEEDITS` | text edits applied to a file in the runtime |
| `PATCHES` | byte patches over the **pristine** executable, each guarded by an `EXPECT` check |
| `DLLOVERRIDES` | which DLLs resolve native vs builtin |
| `PERSISTS` | what survives the run |
| `VARS` | variables the player sets before launch, substituted as `%KEY%` wherever they are used |
| `ENTRYPOINTS` | **what to run** — each entry a variant. No `GUEST` ⇒ a launchable; with `GUEST` ⇒ a runner providing those platforms |
| `TILE` | the identity of a title: `UID` (one card), `PARENTUID` (nesting), `TITLE`, `COVER`. Carried by launchables; inherited through `OVER` |
| *(none)* | a plain node: pure composition, exists only to be `OVER` other nodes under one name |

A node carries any subset of these — *one meaningful change*: a widescreen fix that is a byte patch, an ini edit and
a knob is ONE node with three sections. Plural payloads batch *within* a section — `REGEDITS` carries a whole hive
tree, `PATCHES` every patch to one executable — so granularity is a graph question, not a payload question. Split a
node when something needs to depend on part of it.

---

## How to read this document

The chapters build on each other; read them in order the first time.

| # | Chapter | What it defines |
|---|---------|-----------------|
| — | [Glossary](docs/00-glossary.md) | Every term used normatively. |
| 01 | [Overview & design model](docs/01-overview.md) | The everything-is-a-node philosophy; goals; invariants. |
| 02 | [The Node object](docs/02-nodes.md) | Every field of a node, its type, default and meaning. |
| 03 | [Roles](docs/03-roles.md) | Derived, never declared: launchable, runner, identity, substance, canonical, graft; tiles and variants. |
| 04 | [Bundles, the library & indexing](docs/04-bundles-and-library.md) | On-disk layout, repos, LABEL uniqueness (within a tree), index building. |
| 05 | [`LAYERS`](docs/05-layers.md) | `FORM` zip / dir / file / delta, PATH+SOURCE, TARGET, SUBMOUNTS. |
| 06 | [Edit sections](docs/06-edit-layers.md) | `REGEDITS`, `FILEEDITS`, `PATCHES`, `DLLOVERRIDES` (+ the OVERRIDE pass model). |
| 07 | [Persistence](docs/07-persistence.md) | `PERSISTS`: one entry per durable file or registry key, runner keep-sets. |
| 08 | [Variables & `VARS`](docs/08-variables.md) | The `%TOKEN%` engine, the full token table, user-facing knobs. |
| 09 | [Invocation](docs/09-exec.md) | `ENTRYPOINTS`: a launchable entry (no GUEST) and a runner entry (GUEST) are one shape; execution is not transitive. |
| 10 | [Platforms & runners](docs/10-platforms-and-runners.md) | Platform tokens, the GUEST→HOST graph, the runner build model. |
| 11 | [Runner daisy-chaining](docs/11-runner-chaining.md) | Shortest-chain resolution, the native terminal, cross-namespace nesting. |
| 12 | [Resolution](docs/12-resolution.md) | Selection ≠ closure: the OVER closure, any-of groups, NOT, grafts and the offered set, precedence, conflicts, the instance. |
| 13 | [The runtime model](docs/13-runtime-model.md) | The single overlay mount, layer stacking order, prefixes, case rules. |
| 14 | [Content addressing & distribution](docs/14-content-addressing.md) | The SOURCE block, hydrate/dehydrate/publish/seed over IPFS. |
| 15 | [Validation](docs/15-validation.md) | Every rule a validator must enforce (errors vs. warnings). |
| 16 | [Run modes & the CLI surface](docs/16-cli-and-run-modes.md) | The four run modes; path overrides; resolve/list/validate. |
| 17 | [Worked examples](docs/17-examples.md) | Complete node sets, from a native game to the SNES daisy chain. |
| 18 | [Reference implementation map](docs/18-reference-implementation.md) | Where each rule lives in VidyaGod's source. |
| 19 | [Conformance](docs/19-conformance.md) | The checklist a from-scratch implementation must satisfy. |

---

## Status & versioning

This document describes **MPF schema generation 3 — the flat node DAG**, in which a node *is* one layer. Two earlier
generations are **obsolete** and are not described here except as historical notes:

| gen | Shape | Why it went |
|-----|-------|-------------|
| 1 | one monolithic `MANIFEST.json` with `SUBGAMES`/`COMPONENTS` arrays | the graph was implicit and un-addressable |
| 2 | a node with a `ROLE` and an ordered `LAYERS[]` array | identity collapsed into `Declare*` layers, and then the array itself became the problem: nothing could reference, reorder or depend on a single layer |
| **3** | a node is one layer; `TYPE` on the node, payload hoisted onto it, order carried by `PARENTS` | superseded |
| **4** | **one pluripotent node kind, one edge `OVER` (CNF), tiles on the variants, selection ≠ closure, grafts** | current |

What generation 3 changed, beyond flattening: `DeclareExec` and `DeclareRunner` unified (a launchable is a runner that
provides nothing, so the chain needs no special case for its ends); `TOGGLE` replaced the never-independent
`OPTIONAL`+`DEFAULT` pair; the registry became a tree instead of a flat path plus a value map; and
`DeclareLibraryItem` became a node in its own right and the **parent** of the launchables it groups.

What generation 4 changed: the types went away — a node is *pluripotent*, carrying any subset of the payload sections,
and what it *is* (launchable, runner, substance, graft) is derived from what it carries and what it is `OVER`. The
edges collapsed into one, `OVER`, read as a conjunction of requirements (a ref, an any-of group, a `NOT`). The tile
moved onto the launchables themselves (same `UID` = one card; the variants are the nodes). And the rule that makes
mods work: **selection ≠ closure** — what the user chose is judged for offers and executed; what it is made of only
mounts. Nothing travels along the chain.

Where this spec and any older material disagree, **this spec wins**.

The format is pre-1.0 and evolving. Breaking changes are recorded per-chapter where relevant. The normative keywords
**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** are used in their conventional (RFC 2119) sense.

Nine numbered invariants (**I1**–**I9**) are stated in [chapter 1](docs/01-overview.md) and referenced throughout; a
conforming implementation must preserve all of them. **I9** — *sibling order is unspecified* — is the newest and the
easiest to violate by accident.

---

## License & contributions

The format is open. This repository is documentation; contributions that clarify, correct against the reference
implementation, or add worked examples are welcome. When the spec and the reference implementation diverge, file it —
one of them is wrong, and saying which is the whole point of having a written spec.
