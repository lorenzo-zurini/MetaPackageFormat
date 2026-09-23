# Glossary

Terms are defined here once and used with these exact meanings throughout the spec. Capitalized JSON keys (e.g.
`OVER`) are field names; `code font` lower-case words (e.g. `runner`) are derived readings of a node.

**Node** — the atomic unit of the format. One JSON object: facets (`CID` handle, `LABEL`, `WHEN`, `TOGGLE`,
`PUBLISH`), a `TILE`, `ENTRYPOINTS`, any subset of the payload sections, and the one edge `OVER`. Its **identity is
EXCLUSIVELY its CID** — the content hash of its canonical dag-json block, computed recursively over the CIDs it
links. Its **authoring handle** is a stored `CID` field (the CID it last minted to, or a placeholder before first
publish) that references point to; it is stripped at freeze. `LABEL` is **purely cosmetic** — never a key, may
repeat. There is no `TYPE`: what a node *is* is derived from what it carries. See [chapter 02](02-nodes.md).

**One meaningful change** — the granularity of a node: a widescreen fix that is a byte patch, an ini edit and a
knob is ONE node carrying three sections. A node may span kinds; the engine applies kinds in phases.

**Section** — one of the payload arrays a node may carry: `LAYERS` (files), `PATCHES` (byte patches), `FILEEDITS`
(text edits), `REGEDITS` (registry), `DLLOVERRIDES` (DLL policy), `VARS` (knobs), `PERSISTS` (durable state). And the
two facets: `ENTRYPOINTS` (what to run) and `TILE` (which title). See [chapters 05](05-layers.md)–[09](09-exec.md).

**`OVER`** — the one edge: "I am made of you; you are under me." A conjunction of requirements — a ref (must be
present), an any-of group `[…]` (one must be present), an exclusion `{"NOT": …}` (must not be selected). Direction
newer → older; the older side never enumerates what builds on it. See [chapter 02 §2.3](02-nodes.md).

**Node graph** — the union of every node discoverable by an implementation, keyed by CID; the edges are the CID
refs in `OVER` (plus platform edges implied by a runner's `HOST`/`GUEST`). In an authoring tree refs are the
targets' stored `CID` handles and resolve to derived CIDs at freeze. See [chapter 04](04-bundles-and-library.md).

**Launchable** — a node carrying `ENTRYPOINTS` with an entry that has no `GUEST`: something the user can start.
Each entry is a **variant**. See [chapter 09](09-exec.md).

**Runner** — a node carrying an `ENTRYPOINTS` entry with a non-empty `GUEST`: an executor that runs content of
*guest* platforms while itself running on a *host* platform. Its build is its own `LAYERS` or what it is `OVER`. A
launchable is a runner that provides nothing. See [chapter 10](10-platforms-and-runners.md).

**Tile / title** — the `TILE` facet `{UID, TITLE, COVER, META}`, carried by a launchable. One `UID` = one card in
the library; there is no tile node. Nesting inside a card (an expansion under its main game) is derived from the
chain: a different tile `OVER` the main is a child. See [chapter 03 §3.2](03-roles.md).

**Identity** — the set of UIDs a node belongs to: its own `TILE.UID`, else the union of its `OVER` requirements'
identities. Grouping, offering and sharing all follow it. A node with none is **substance**.

**Substance** — a node with no identity (a library: dgVoodoo, a codec stack): never listed under a card, never a
choice; pulled into a mount by whatever names it. The game side declares the library, never the reverse.

**Canonical** — a lint, not a kind: the pristine game — content, entrypoint, tile and nothing else. One per
title-version, eternal, reconstructible to the same CID from a user's own known files or fetched by CID.

**Graft** — a node of a title that is not part of a launchable's own composition and is `OVER` something of it:
a mod, an enhancement, a compat patch, a mod loader. It enters a mount only when **selected**. See
[chapter 12 §12.4](12-resolution.md).

**Selected set / closure** — the user's *choices* (the launchable, one member per any-of group, ticked grafts)
versus what those are *made of* (everything reachable through plain `OVER` entries). Offering is judged against the
selected set, mounting uses the closure, execution runs a selected node's entry. See [chapter 12](12-resolution.md).

**Instance** — the loadout: local configuration, not a node — the selected set, the entrypoint, per-graft
precedence, per-file winners, variable overrides, saves. Permutations are instances.

**Toggle** — a node's `TOGGLE`, `"on"` or `"off"`: present ⇒ user-toggleable, value = the author's default. On a
closure node it is an optional module; on a graft it is whether it is pre-selected.

**Chain** — a run of nodes each `OVER` the next: what used to be one node's ordered list is now a chain, and the
order is the edges.

**Bundle** — a directory holding node `.json` files plus the loose content they reference. A scan unit with no
semantics of its own. See [chapter 04](04-bundles-and-library.md).

**Library root** — a directory whose immediate subdirectories are bundles. See [chapter 04](04-bundles-and-library.md).

**Layer** — a node's payload seen from the runtime's side: the lowered, engine-facing form of a section entry.

**VFS** — the single overlay/union mount the runtime assembles from every `LAYERS` entry in the mount (plus a
runtime prefix, default data and a writable layer). See [chapter 13](13-runtime-model.md).

**Runtime path** — the mount point the assembled VFS appears at. `%RuntimePath%`.

**Content root** — a runner entry's `CONTENT_ROOT`: where the game's content mounts under the runtime path.

**Closure / load order** — the topologically-ordered mount: earlier = lower overlay priority; the launchable is
above its own closure; grafts above that, in instance precedence. See [chapter 12](12-resolution.md).

**Conflict** — two grafts providing the same target path with different content, overlapping patch ranges on one
file, or the same key with different values. Detected, never declared by pairs; resolved by precedence and
per-file winners. See [chapter 12 §12.6](12-resolution.md).

**Platform token**, **Machine platform**, **Guest / host platform**, **Runner chain**, **Native terminal**,
**Namespace boundary**, **Prefix** — as in [chapters 10](10-platforms-and-runners.md), [11](11-runner-chaining.md)
and [13](13-runtime-model.md): a runner is a directed edge `guest → host` in the platform graph; every chain ends
at the machine platform through the native terminal; a runner with a `CONTENT_ROOT` is a namespace boundary.

**Persistence** — which runtime state survives a session, written to the instance's store. See [chapter 07](07-persistence.md).

**Source** — a content-addressed locator on a layer or cover (`SOURCE`): how to obtain the bytes if the local file
is absent. See [chapter 14](14-content-addressing.md).

**Hydrate / dehydrate / publish** — fetch a mount's content-addressed payloads locally / seed them and mint the
node blocks. A **share** is a set of root CIDs; the receiver groups them by `UID` from the blocks themselves.

**Variable / token**, **CustomVar** — a `%NAME%` placeholder expanded at resolve time; a `VARS` entry declares one.
See [chapter 08](08-variables.md).

**Reference implementation** — VidyaGod. See [chapter 18](18-reference-implementation.md).

**Earlier generations (obsolete)** — generation 1: a `MANIFEST.json` with `SUBGAMES`/`COMPONENTS`; generation 2:
typed nodes (`TYPE` ∈ `Content`, `RegEdit`, …, `DeclareExec`, `DeclareLibraryItem`, `Group`) with
`PARENTS`/`EXCLUDE`/`LIBRARYITEM` edges. Superseded; migrated in one pass; never read two ways.
