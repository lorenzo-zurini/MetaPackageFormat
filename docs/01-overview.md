# 01 · Overview & design model

## 1.1 The single idea: everything is a node

MPF has exactly one structural primitive. Every concept that other formats model with distinct constructs — a game, an
edition, a dependency, a mod, an optional add-on, an emulator, a runtime, a config option — is, in MPF, **a node**.

A node is a small JSON object with a globally-unique identity (`NODE_ID`). It does at most three things:

1. **Selects** other nodes, by listing their ids in `PARENTS`. This is how composition, dependencies, variants and
   options are expressed. Edges carry no attributes; selection attributes (optional? default-on? mutually exclusive?)
   live on the *selected* node.
2. **Contributes** payloads, by listing them in `LAYERS`: files to overlay, registry edits, config patches, persistence
   rules, user knobs.
3. **Declares execution**, via `ROLE`, `PLATFORM` and `EXEC`: whether it's an entry point (`launchable`), an executor
   (`runner`), or neither (`content`), and on/for what platform, and how to invoke it.

Nothing else is a first-class concept. There is no separate "package object," no "installer," no "variant table," no
"runner registry schema." There is the graph, and the rules for walking it.

> **Design consequence.** Because there is one primitive, every feature is expressed by *composing* nodes rather than by
> *adding* format constructs. The format stays small while the expressible space stays large. New capabilities tend to
> *fall out* of the existing model (see §1.4) rather than requiring new fields.

## 1.2 What a complete "package" is

There is no monolithic package object. A *package*, informally, is **a bundle directory** (chapter 4) containing the node
files and loose content for one logical product, e.g. a game and its base content:

```
[299] Super Metroid/
├── super_metroid.json          # launchable node (the entry point / tile)
├── super_metroid_base.json     # content node (carries the ROM as a layer)
├── Super Metroid (...).sfc     # the ROM bytes referenced by the layer
└── Super_Metroid.png           # cover art referenced by META
```

But a "package" has fuzzy edges *by design*: its launchable's closure can reference content nodes in *other* bundles
(shared runtimes, common dependencies), and the runners that execute it almost always live in a *separate* bundle (a
runner library). The unit of distribution is a bundle; the unit of *meaning* is a launchable plus its resolved closure,
which may span bundles.

## 1.3 The lifecycle of a launch

Resolving and running a launchable proceeds in well-defined phases. Each is specified in detail later; this is the map.

1. **Index** the graph: scan every library root's bundles, parse each `.json` with a `NODE_ID`, key by id
   ([ch. 4](04-bundles-and-library.md)).
2. **Resolve the content closure** of the chosen launchable: walk `PARENTS`, apply `OPTIONAL`/`DEFAULT`/`EXCLUDE` gating
   and the hierarchy gate, topologically order the survivors ([ch. 12](12-resolution.md)).
3. **Resolve the runner chain**: BFS the platform graph from the launchable's `PLATFORM.HOST` to the machine platform,
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

- **Mods with automatic load order** — a mod is a content node that `PARENTS` the base. Overlay priority follows the
  resolved order, so "later in the closure wins" *is* the load order. Marking a mod `OPTIONAL` makes it a toggle.
- **Optional DLC / expansions** — an `OPTIONAL` content node; `DEFAULT` decides if it's on out of the box; `EXCLUDE`
  makes a set of them mutually exclusive (pick-one).
- **Multi-edition games** — several `launchable` nodes sharing a `GAME` value; the library groups them, the user picks a
  variant, each variant resolves its own closure.
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

- **I1 — Global identity.** `NODE_ID` is unique across the whole graph. On a duplicate, first-seen wins and the duplicate
  is dropped with a diagnostic ([ch. 4](04-bundles-and-library.md)).
- **I2 — Acyclic selection.** `PARENTS` edges MUST form a DAG. Cycles are reported; resolution still completes by
  breaking the back-edge ([ch. 12](12-resolution.md)).
- **I3 — Single overlay.** The entire runtime is ONE overlay mount at the runtime path. Layer order = overlay priority,
  lowest first; the launchable's own layers are highest ([ch. 13](13-runtime-model.md)).
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
- **I8 — Content addressing is advisory-to-present, authoritative-to-fetch.** A locally-present file at a layer's path is
  authoritative; the `SOURCE` `CID` is consulted only to obtain a missing file ([ch. 14](14-content-addressing.md)).

## 1.6 Relationship to the reference implementation

VidyaGod implements all of the above. Throughout, callouts like *“(VidyaGod: `ResolveNodeOrder` in `manifestmodel.cpp`)”*
point at the concrete code so a reader can cross-check behavior against a working system. These pointers are a
convenience, **not** part of the normative format. If VidyaGod and this document disagree, that is a bug in one of them;
the spec is the arbiter of *what the format is*, and the implementation is the arbiter of *what currently runs*.

Next: [the Node object](02-nodes.md).
