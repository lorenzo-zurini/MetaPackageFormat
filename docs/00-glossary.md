# Glossary

Terms are defined here once and used with these exact meanings throughout the spec. Capitalized JSON keys (e.g.
`PARENTS`) are field names; `code font` lower-case words (e.g. `runner`) are identity/type values.

**Node** — the atomic unit of the format. One JSON object, stored in one `<node_id>.json` file, identified by a
globally-unique `NODE_ID`. A node groups/selects other nodes (via `PARENTS`) and/or contributes payloads (via `LAYERS`).
What it *is* — content, launchable, runner, library tile — emerges from which `Declare*` identity layers it carries;
there is no `ROLE` field. See [chapter 02](02-nodes.md).

**Node graph** — the union of every node discoverable by an implementation, keyed by `NODE_ID`. A single flat namespace;
edges are `NODE_ID` references in `PARENTS` (and platform edges implied by a runner's `DeclareRunner` host/guest). See
[chapter 04](04-bundles-and-library.md).

**Identity layer** — one of the `Declare*` layers (`DeclareExec`, `DeclareLibraryItem`, `DeclareRunner`) that, by its
presence in a node's `LAYERS`, declares what the node is. A node may carry several; their union is its identity. A node
with none is plain content. There is no `ROLE` field. See [chapter 03](03-roles.md).

**Launchable** — a node carrying a `DeclareExec` layer: an entry point the user can start (a game, a tool, an edition).
It declares the target platform of its content and how to invoke it. See [chapter 03](03-roles.md).

**Content node** — a node with no `Declare*` layer: contributes `LAYERS` (files, edits, persistence rules) and/or groups
other content via `PARENTS`. Never started directly; pulled in by a launchable's or runner's closure.

**Runner** — a node carrying a `DeclareRunner` layer: an executor that runs content of one or more *guest* platforms
while itself running on a *host* platform. Wine/Proton, an emulator, or a native pass-through. Its runnable payload (its
*build*) comes from its `PARENTS`. See [chapter 10](10-platforms-and-runners.md).

**Library tile** — a node carrying a `DeclareLibraryItem` layer: a presentable *game* in the library (`TITLE`/`COVER`/
descriptive metadata). Not launchable on its own unless it also carries a `DeclareExec`. See [chapter 03](03-roles.md).

**Variant** — a launchable (`DeclareExec`) node that has a library-tile (`DeclareLibraryItem`) node as a `PARENTS`
ancestor: one edition/version among several of "the same game," shown grouped under that one tile. Grouping is the
graph edge — there is no `GAME` field; `LABEL` distinguishes variants. A single-variant game collapses to one node
carrying both layers. See [chapter 03](03-roles.md).

**Bundle** — a directory holding one or more node `.json` files plus the loose content they reference (zips, ROMs,
covers). A bundle groups *files*; it has no semantic meaning beyond being a scan unit. See
[chapter 04](04-bundles-and-library.md).

**Library root** — a directory whose immediate subdirectories are bundles. The implementation scans library roots to
build the node graph. Multiple roots compose into one graph. See [chapter 04](04-bundles-and-library.md).

**Layer** — one entry in a node's `LAYERS` array: a typed payload. *VFS layers* contribute files
(`VFSZipLayer`/`VFSDirLayer`/`VFSFileLayer`); *edit layers* mutate files/registry (`RegEdit`/`DllOverride`/`FileEdit`);
the *Persist* layer declares durable state (one self-describing primitive: `MODE`/`KEEP`/`DROP`); the *CustomVar*
layer declares a user knob. See [chapters 05](05-layers.md)–[08](08-variables.md).

**VFS** — the *virtual filesystem*: the single overlay/union mount the runtime assembles from all VFS layers (plus a
runtime prefix, default-data and a writable layer) and presents at the *runtime path*. See
[chapter 13](13-runtime-model.md).

**Runtime path** — the single mount point the assembled VFS appears at; the root a runner's prefix/env points to. Exposed
as `%RuntimePath%`.

**Content root** — a runner property (`EXEC.CONTENT_ROOT`): the runtime-path-relative directory the game's content is
mounted *under*. `""` = the root; `"pfx/drive_c/<uid>"` places content inside a Proton prefix's `C:` drive. See
[chapter 09](09-exec.md), [chapter 13](13-runtime-model.md).

**Closure / load order** — the topologically-ordered set of nodes pulled in by resolving a launchable (or runner) over
`PARENTS`, after applying optional/exclude gating. Earlier = lower overlay priority; the launchable itself is last
(highest priority). See [chapter 12](12-resolution.md).

**Platform token** — an opaque string naming a software platform: `"linux64"`, `"win32"`, `"win64"`, `"snes"`, `"macos"`,
etc. Compared only for equality. The host's own token is the *machine platform*. See [chapter 10](10-platforms-and-runners.md).

**Machine platform** — the platform token of the host the runtime is executing on (e.g. `"linux64"`). Every runner chain
must terminate at the machine platform. See [chapter 10](10-platforms-and-runners.md).

**Guest / host platform** — a runner runs *guest*-platform content (`PLATFORM.GUEST[]`) while itself being a
*host*-platform program (`PLATFORM.HOST`). A runner is a directed edge `guest → host` in the platform graph.

**Runner chain (daisy chain)** — the ordered list of runners that takes content from its platform to the machine
platform, nesting innermost (runs the content) to outermost (runs on the machine), always terminated by a native runner.
See [chapter 11](11-runner-chaining.md).

**Native terminal** — the final runner in every chain: a runner whose host and guest are both the machine platform. It is
the universal execution primitive (an `execve`), and the single uniform place to wrap *all* launches. See
[chapter 11](11-runner-chaining.md).

**Namespace boundary** — a runner that introduces a guest filesystem namespace (it declares a `CONTENT_ROOT` / generates
a prefix), e.g. Wine's `C:` drive. Paths must be *translated* across it. See [chapter 11](11-runner-chaining.md).

**Prefix** — a Wine/Proton prefix: the `drive_c` + registry-hive directory tree a Wine-family runner needs. Generated
once per runner and reused read-only. See [chapter 13](13-runtime-model.md).

**Persistence** — the rules deciding which runtime state survives a session and travels with the package, written to the
package's `USERDATA` store. See [chapter 07](07-persistence.md).

**Source** — a content-addressed locator on a layer or cover (`SOURCE`), telling an implementation how to *obtain* the
payload's bytes if the local file is absent (currently an IPFS `CID`). See [chapter 14](14-content-addressing.md).

**Hydrate / dehydrate** — *hydrate* = fetch a package's content-addressed payloads to their local paths so it can run;
*dehydrate* (a.k.a. *publish*) = seed the payloads to the content network and record their `CID`s in the nodes for
sharing. See [chapter 14](14-content-addressing.md).

**Variable / token** — a `%NAME%` placeholder expanded at resolve/launch time from the runtime's variable map (paths,
content locators, screen geometry, custom knobs). See [chapter 08](08-variables.md).

**CustomVar** — a layer declaring a user-tweakable knob that becomes a `%KEY%` token. See [chapter 08](08-variables.md).

**Reference implementation** — VidyaGod. Where this spec says "the implementation does X," it means a *conforming*
implementation; VidyaGod is cited for concreteness. See [chapter 18](18-reference-implementation.md).

**Generation-1 manifest (obsolete)** — the predecessor format: a single `MANIFEST.json` per package with
`PACKAGENAME`/`PACKAGEUID`/`SUBGAMES`/`COMPONENTS`/`SUBCOMPONENTS`. Superseded by the node graph. Mentioned only so that
old field names (`SUBGAMES`, `COMPONENTS`, `EXEPATH`, `GROUP`, `VARIANT_ID`/`EDITION`) are recognizable; do **not** author
new packages this way.
