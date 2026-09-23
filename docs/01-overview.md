# 01 · Overview & design model

## 1.1 The single idea: everything is a node

MPF has exactly one structural primitive. Every concept that other formats model with distinct constructs — a game, an
edition, a dependency, a mod, an optional add-on, an emulator, a runtime, a config option — is, in MPF, **a node**.

A node is a small JSON object whose identity is **exclusively its CID** (the content hash of its frozen block); it
carries a stored `CID` **handle** that references point to, and an optional, purely-cosmetic `LABEL`. There is
**one node kind** and it does exactly two things:

1. **It is one meaningful change.** Any subset of the payload sections — files to overlay (`LAYERS`), byte patches
   (`PATCHES`), config edits (`FILEEDITS`), registry writes (`REGEDITS`), a DLL policy (`DLLOVERRIDES`), user knobs
   (`VARS`), durable state (`PERSISTS`) — plus how to run (`ENTRYPOINTS`) and two declared facets: a **face**
   (`TILE`: which title begins here) and a **variant** (`VARIANT`: list me on the shelf). A change may span kinds.
   A node with no payload is just a node. There is no `TYPE`, no `ROLE`.
2. **It is `OVER` other nodes.** The one edge — "I am made of you; you are under me" — a list in which a bare ref
   *composes* and a group or a `NOT` *requires*. This is how composition, dependencies, compatibility, exclusion,
   identity *and order* are expressed. The edge points newer → older: the older side never enumerates what builds
   on it, so nothing that grows is ever listed.

Nothing else is a first-class concept. There is no package object, no installer, no variant table, no runner
registry, no tile node, no second edge. There is the graph, and the rules for walking it.

> **Facts fold, choices don't.** Along the chain, bytes overlay, the entrypoint is inherited unless replaced, and
> identity ascends from the face. What never travels is a *choice*: picking a variant selects exactly that node, a
> graft is ticked, a mod is offered against what is selected — never against what it is made of. This is what lets
> a version chain carry bytes and a command without carrying mods, and lets a mod attach to any of several versions
> without either side listing the other. See [ch. 12](12-resolution.md).

> **Design consequence.** Because there is one primitive, every feature is expressed by *composing* nodes rather than by
> *adding* format constructs. The format stays small while the expressible space stays large. New capabilities tend to
> *fall out* of the existing model (see §1.4) rather than requiring new fields.

## 1.2 What a complete "package" is

There is no monolithic package object. A *package*, informally, is **a bundle directory** (chapter 4) containing the node
files and loose content for one logical product, e.g. a game and its base content:

```
[9001] Vortex Quest/
├── Vortex Quest.json           # face + VARIANT + ENTRYPOINTS + LAYERS (the ROM), in one node
├── VortexQuest.vtx             # the ROM bytes
└── VortexQuest_Cover.png       # cover art referenced by TILE.COVER
```

But a "package" has fuzzy edges *by design*: its variants' closures can reference content nodes in *other* bundles
(shared runtimes, common dependencies), and the runners that execute it almost always live in a *separate* bundle (a
runner library). The unit of distribution is a bundle; the unit of *meaning* is a variant plus its resolved closure,
which may span bundles.

## 1.3 The lifecycle of a launch

Resolving and running a variant proceeds in well-defined phases. Each is specified in detail later; this is the map.

1. **Index** the graph: scan every library root's bundles, parse each `.json` node, key by its CID handle
   ([ch. 4](04-bundles-and-library.md)).
2. **Resolve the mount**: the closure of the picked variant — the transitive union of bare `OVER` refs,
   topologically ordered — then the ticked, applicable grafts above it in instance precedence
   ([ch. 12](12-resolution.md)).
3. **Resolve the runner chain**: BFS the platform graph from the chosen entry's `HOST` to the machine platform,
   appending the native terminal ([ch. 11](11-runner-chaining.md)).
4. **Resolve variables & persistence**: expand `%TOKEN%`s, resolve `CustomVar` knobs, decide what state persists
   ([ch. 8](08-variables.md), [ch. 7](07-persistence.md)).
5. **Materialize content**: ensure every layer's bytes are present locally, fetching content-addressed sources as needed
   ([ch. 14](14-content-addressing.md)).
6. **Build the runtime**: provision the runner prefix (if any), build the default-data layer from base edits, assemble
   the overlay mount in the correct stacking order ([ch. 13](13-runtime-model.md)).
7. **Compose & execute**: build the nested runner command, translating paths across each namespace boundary, and execute
   one process ([ch. 11](11-runner-chaining.md)).
8. **Capture & tear down**: persist declared state back to `USERDATA`, unmount save-safely, wipe ephemeral state
   ([ch. 7](07-persistence.md), [ch. 13](13-runtime-model.md)).

## 1.4 Things that fall out of the model (for free)

These are *not* features the format special-cases. They are shapes of the one graph:

- **Mods, at any scale** — a mod is a node `OVER` the version(s) it works on: a **graft**. Nobody lists it; it is
  offered to whoever selects a version it names, mounts above the game when ticked, and a graft can be `OVER` a
  graft (an HD pack over a texture mod, a compat patch `OVER [modA, modB, game]`). A thousand-mod Skyrim is a
  thousand grafts and one instance; a hundred configurations are a hundred instances over one pool.
- **Optional pieces and expansions** — an optional piece (a soundtrack, a fix) is a graft the author pre-ticks,
  `OVER` the variants it applies to; an expansion is its own face `OVER` the base's pristine, nested under the
  base by containment; `NOT` makes a set of grafts mutually exclusive (pick-one).
- **Multi-edition and multi-version games** — one tile on the pristine, several `VARIANT`s over it; the library
  groups them under one card, the user picks a variant, each resolves its own closure. Minecraft is one tile, 903
  variants each `OVER` the previous version's bytes, and an entrypoint declared only where the command changes.
- **Mod loaders** — a graft that carries an entry (SKSE `OVER [["640", "659"]]`, Forge `OVER ["1.16.5"]`): ticked
  on the version you picked, it adds a way to run; the version stays selected, so its mods stay offered.
- **Authoring by capture** — run an installer on a live runtime built from any point of a chain, and the files and
  registry it wrote become *new nodes `OVER` that point*. Nothing has to be spliced into an existing node.
- **Cross-platform execution & ARM** — a runner is an edge in a platform graph; running anything anywhere is shortest-path
  over runner edges. New platforms are new runners, not new format.
- **P2P distribution & portable installs** — every payload carries a content-addressed `SOURCE`; an install is "fetch the
  closure's CIDs," sharing is "seed them." The graph is the manifest of what to fetch.
- **Portable saves** — persistence writes to the package's own `USERDATA`, so saves travel with the bundle and survive a
  full wipe of the app's data root.
- **Reproducible runtimes** — the runner ships its exact runtime as content layers; the prefix is generated once and
  reused read-only; the overlay is rebuilt deterministically each launch.

When evaluating a proposed feature, the first question is always: *what shape of the existing graph already expresses
this?* Only if the answer is genuinely "none" does the format grow.

## 1.5 Core invariants

A conforming implementation MUST preserve these properties. They are referenced by later chapters.

- **I1 — CID identity.** A node's identity is **exclusively its CID**. Nodes are keyed by their `CID` handle (unique by
  construction — a content hash; a placeholder before first mint). `LABEL` is cosmetic and MAY repeat. On a duplicate
  *handle* (a stale or forged stored CID), first-seen wins; the loser is re-keyed, never dropped, so no node vanishes
  ([ch. 4](04-bundles-and-library.md)).
- **I2 — Acyclic composition.** Positive `OVER` refs MUST form a DAG. Cycles are reported; resolution still completes by
  breaking the back-edge ([ch. 12](12-resolution.md)).
- **I3 — Single overlay.** The entire runtime is ONE overlay mount at the runtime path. Resolved closure order =
  overlay priority, lowest first; the variant — the terminal node of its chain — is highest
  ([ch. 13](13-runtime-model.md)).
- **I4 — One process.** A launch executes exactly one host process — the outermost (native-terminal) runner — which
  nests every inner runner and the content as arguments. Inner runners are not separately spawned by the implementation
  ([ch. 11](11-runner-chaining.md)).
- **I5 — Always-terminated chains.** Every runner chain ends at the machine platform via a native terminal; content with
  no path to the machine platform is unrunnable and the launch is refused, not silently "succeeded"
  ([ch. 11](11-runner-chaining.md)).
- **I6 — Pristine sources, ephemeral runtime, durable saves.** Source content (downloaded layers, generated prefixes) is
  never mutated in place; the assembled runtime is ephemeral and wiped after the session; only explicitly-declared
  persistence survives, written to `USERDATA` ([ch. 7](07-persistence.md), [ch. 13](13-runtime-model.md)).
- **I7 — Save-safety.** The runtime MUST NOT destroy durable user data. Teardown unmounts durable-backed mounts
  non-lazily and verifies them gone before deleting any ephemeral tree ([ch. 13](13-runtime-model.md)).
- **I8 — Content addressing is advisory-to-present, authoritative-to-fetch.** A locally-present file at a node's path is
  authoritative; the `SOURCE` `CID` is consulted only to obtain a missing file ([ch. 14](14-content-addressing.md)).
- **I9 — Unrelated order is unspecified.** Two nodes with no `OVER` relation are not ordered with respect to each
  other. An implementation MUST be deterministic, but a package MUST NOT depend on which of two unrelated nodes
  writes last; if the order matters, it must be an edge — or, for grafts, the instance's precedence.
- **I10 — Facts fold, choices don't.** A bare `OVER` ref composes and is never a choice; a group or a `NOT` is a
  requirement evaluated only when a node is offered. Mounts are built from the closure; the entrypoint and the face
  are inherited along the chain; grafts are judged against the selected set; picking a variant selects exactly that
  node ([ch. 12](12-resolution.md)).
- **I12 — A node describes only its own contribution.** Never anything about a node above it. Declared is exactly:
  the payload sections, `ENTRYPOINTS`, `TILE`, `VARIANT`, and the edge; everything else is derived.
- **I11 — Nothing enumerates what grows.** Every edge points newer → older; a library never lists the games that use
  it, a version never lists its mods, a tile never lists its variants. Grouping and offering are derived.

## 1.6 Relationship to the reference implementation

VidyaGod implements all of the above. Throughout, callouts like *“(VidyaGod: `ResolveNodeOrder` in `manifestmodel.cpp`)”*
point at the concrete code so a reader can cross-check behavior against a working system. These pointers are a
convenience, **not** part of the normative format. If VidyaGod and this document disagree, that is a bug in one of them;
the spec is the arbiter of *what the format is*, and the implementation is the arbiter of *what currently runs*.

Next: [the Node object](02-nodes.md).
