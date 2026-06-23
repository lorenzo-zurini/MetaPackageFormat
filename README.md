# MetaPackage Format (MPF)

**A universal, platform-agnostic, content-addressed software distribution format built on a single primitive: the node.**

MPF describes how to package, compose, share and *run* software — any software, from any platform, on any host — as a
graph of small, globally-referenceable JSON nodes. A SNES ROM, a Windows game, a native Linux binary, an emulator, a
Wine/Proton runtime, a mod, an optional expansion, a configuration knob: all of them are *nodes*. The format is the
graph and the rules for resolving, composing and executing it.

This repository is the **definitive specification** of that format. It is implementation-independent: the format is
defined here, not in any one program. The reference implementation is [**VidyaGod**](https://github.com/lorenzo-zurini),
a Qt/C++ launcher, and this document frequently points at *how* VidyaGod realizes a given rule — but those notes are
illustrative. **The spec is normative; the implementation follows it.** A sufficiently determined reader should be able
to reconstruct a working MPF runtime — graph indexing, dependency resolution, the overlay-filesystem runtime, runner
daisy-chaining, content fetching and persistence — from this document alone.

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

> A piece of runnable software is a *node* that either **groups/selects** other nodes (its dependencies, variants,
> options) or **contributes** concrete payloads (files, registry edits, config patches, persistence rules, runtime
> binaries) — or both. "What it runs on" is just an edge in a **platform graph** of runners. "How to obtain it" is a
> **content-addressed source** on each payload.

From that single idea, a remarkable amount *falls out for free*: mod load-order, optional DLC, multi-edition games,
cross-platform execution, P2P distribution, portable saves, and reproducible runtimes — none of them are special cases
in the format, they're all just shapes of the same graph.

---

## The thirty-second mental model

```
            ┌──────────────────────────────────────────────────────────┐
            │                    THE GLOBAL NODE GRAPH                    │
            │                                                            │
   runner ──┤   super_metroid (launchable, PLATFORM.HOST = "snes")       │
   edges    │        │ PARENTS                                           │
   GUEST→HOST│       ▼                                                   │
            │   super_metroid_base (content: the .sfc ROM as a layer)    │
            └──────────────────────────────────────────────────────────┘

   To RUN super_metroid on a linux64 machine, the runtime builds the SHORTEST
   chain of runner edges from the content's platform to the machine's:

        snes ──[snes9x.exe]──▶ win32 ──[Proton]──▶ linux64 ──[native]──▶ EXECUTED
              (a win32 emulator)      (a Wine runtime)     (the terminal)

   …mounts every layer into one overlay filesystem, translates paths across each
   namespace boundary, and execve's a single nested command. Platforms: abstracted.
```

That chain is not authored by hand. The packager declares only *facts* — "this is SNES content," "snes9x is a win32
program that runs SNES content," "Proton is a linux64 runtime that runs win32 programs" — and the runtime *derives* the
route. Add an ARM runner tomorrow and ARM hosts light up with no package changes.

---

## How to read this document

The chapters build on each other; read them in order the first time.

| # | Chapter | What it defines |
|---|---------|-----------------|
| — | [Glossary](docs/00-glossary.md) | Every term used normatively. |
| 01 | [Overview & design model](docs/01-overview.md) | The everything-is-a-node philosophy; goals; invariants. |
| 02 | [The Node object](docs/02-nodes.md) | Every field of a node, its type, default and meaning. |
| 03 | [Roles](docs/03-roles.md) | `content`, `launchable`, `runner` — what each means. |
| 04 | [Bundles, the library & indexing](docs/04-bundles-and-library.md) | On-disk layout, repos, NODE_ID uniqueness, index building. |
| 05 | [VFS content layers](docs/05-layers.md) | `VFSZipLayer` / `VFSDirLayer` / `VFSFileLayer`, TARGET, SUBMOUNTS. |
| 06 | [Edit layers](docs/06-edit-layers.md) | `RegEdit`, `DllOverride`, `FileEdit` (+ the OVERRIDE pass model). |
| 07 | [Persistence layers](docs/07-persistence.md) | `PersistDir/File`, `RegPersist`, `RegKeyPersist`, whole-runtime persist. |
| 08 | [Variables & CustomVar](docs/08-variables.md) | The `%TOKEN%` engine, the full token table, user-facing knobs. |
| 09 | [The EXEC block](docs/09-exec.md) | Launchable exec (CONTENTPATH/WORKDIR/EXEARGS) and runner exec. |
| 10 | [Platforms & runners](docs/10-platforms-and-runners.md) | Platform tokens, the GUEST→HOST graph, the runner build model. |
| 11 | [Runner daisy-chaining](docs/11-runner-chaining.md) | Shortest-chain resolution, the native terminal, cross-namespace nesting. |
| 12 | [Dependency resolution](docs/12-resolution.md) | The PARENTS closure, OPTIONAL/DEFAULT/EXCLUDE, load order. |
| 13 | [The runtime model](docs/13-runtime-model.md) | The single overlay mount, layer stacking order, prefixes, case rules. |
| 14 | [Content addressing & distribution](docs/14-content-addressing.md) | The SOURCE block, hydrate/dehydrate/publish/seed over IPFS. |
| 15 | [Validation](docs/15-validation.md) | Every rule a validator must enforce (errors vs. warnings). |
| 16 | [Run modes & the CLI surface](docs/16-cli-and-run-modes.md) | The four run modes; path overrides; resolve/list/validate. |
| 17 | [Worked examples](docs/17-examples.md) | Complete node sets, from a native game to the SNES daisy chain. |
| 18 | [Reference implementation map](docs/18-reference-implementation.md) | Where each rule lives in VidyaGod's source. |
| 19 | [Conformance](docs/19-conformance.md) | The checklist a from-scratch implementation must satisfy. |

---

## Status & versioning

This document describes **MPF schema generation 2 ("everything is a node")**. An earlier generation used a single
monolithic `MANIFEST.json` with `SUBGAMES`/`COMPONENTS` arrays; it is **obsolete** and not described here except as a
historical note in the glossary. Where this spec and any older material disagree, **this spec wins**.

The format is pre-1.0 and evolving. Breaking changes are recorded per-chapter where relevant. The normative keywords
**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** are used in their conventional (RFC 2119) sense.

---

## License & contributions

The format is open. This repository is documentation; contributions that clarify, correct against the reference
implementation, or add worked examples are welcome. When the spec and the reference implementation diverge, file it —
one of them is wrong, and saying which is the whole point of having a written spec.
