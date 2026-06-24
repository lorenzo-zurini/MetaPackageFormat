# 03 · Identity (the `Declare*` layers)

A node has **no `ROLE` field**. What a node *is* — content, launchable, runner, a library tile — **emerges from which
layers it carries**. Three identity layers, the `Declare*` family, sit alongside the ordinary content/edit/CustomVar/
Persist layers in `LAYERS`:

| Layer | Presence ⇒ the node is | Carries |
|-------|------------------------|---------|
| *(none of the below)* | **content** | just `LAYERS` payloads and/or `PARENTS` grouping |
| **`DeclareExec`** | **launchable** (an entry point) | `PLATFORM` (its content's platform) + `CONTENTPATH`/`EXEARGS`/`WORKDIR` + `LABEL`/`RECOMMENDED` |
| **`DeclareLibraryItem`** | a **library tile** (presentable) | `TITLE`/`COVER`/`UID` + descriptive metadata |
| **`DeclareRunner`** | a **runner** (a `GUEST→HOST` executor edge) | `HOST` + `GUEST[]` + `EXECUTABLE`/`ARGS`/`ENV`/`CONTENT_ROOT`/`PREFIX_GENERATE`/… |

A node may carry several: a typical single-edition game is `DeclareExec` **+** `DeclareLibraryItem`. The identity is the
*union* of what its layers declare. This is the conclusion of "everything is a node": there is one primitive — the layer
— and a node is a bag of layers plus graph edges.

## 3.1 `content` — the building block

A node with no `Declare*` layer. It contributes `LAYERS` (files, edits, persistence, knobs) and/or groups other nodes
through `PARENTS`. Never an entry point, never executed alone; it enters a runtime only when some launchable's or
runner's closure pulls it in. A content node may be pure grouping (only `PARENTS`).

```json
{ "NODE_ID": "morrowind_data",
  "PARENTS": ["morrowind_textures_hd", "morrowind_bloodmoon", "morrowind_tribunal"] }
```

## 3.2 `DeclareExec` — launchable

A `DeclareExec` layer makes a node an **entry point**. It declares the platform of *its content* and how to invoke it
once mounted:

```jsonc
{ "TYPE": "DeclareExec", "PLATFORM": "win32",
  "CONTENTPATH": "age2_x1/age2_x1.exe",     // the thing to run, relative to the content mount (or absent = self-contained)
  "EXEARGS": "", "WORKDIR": "",
  "LABEL": "The Conquerors", "RECOMMENDED": true }   // the variant's name/default in its game's picker
```

A launchable does **not** name a runner — it states `PLATFORM`, and the runtime *derives* the runner chain (chapter 11).
A node's own `LAYERS` sit at the **top** of the overlay, so a launchable variant is the natural home for overrides. See
[chapter 9](09-exec.md) (incl. how `DeclareExec` **composes** along the closure — a base supplies `CONTENTPATH`, a
variant overrides `EXEARGS`).

## 3.3 `DeclareLibraryItem` — the library tile (a game)

A `DeclareLibraryItem` layer makes a node a **presentable library tile** — a *game*:

```jsonc
{ "TYPE": "DeclareLibraryItem", "UID": "749", "TITLE": "Age of Empires II",
  "COVER": { "PATH": "cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
  "DEVELOPER": "Ensemble Studios", "SERIES": "Age of Empires" }   // free-form descriptive metadata
```

It is display metadata; no field is structurally required (`TITLE` falls back to the `NODE_ID`). Cover CIDs participate
in publishing/seeding like layer content (chapter 14). A tile is **not launchable on its own** unless it also carries a
`DeclareExec`. Implementations MUST ignore unknown metadata keys.

## 3.4 Variants & games — grouping is a graph edge

Several launchables that are *the same game in different editions/versions* are grouped into **one tile**; the user opens
the tile and picks a **variant**. Grouping is a **`PARENTS` edge to the game's `DeclareLibraryItem` node** — there is no
`GAME` string:

- A **game** is a node with a `DeclareLibraryItem` (the tile). It carries no content.
- A **variant** is a node with a `DeclareExec` that has that game node as a `PARENTS` ancestor. So the game node is a
  parent of *only the exec nodes*; each variant's content chain hangs off the variant separately. Variants **inherit**
  the tile's metadata (it composes down the closure) and contribute their own `LABEL`/`RECOMMENDED`.
- A **single-variant game collapses to one node** carrying both `DeclareLibraryItem` + `DeclareExec` (no separate game
  node).
- Within a tile, `RECOMMENDED: true` marks the default variant; else the implementation picks deterministically.

```
        aoe2  (DeclareLibraryItem — the tile)
       ╱  │  ╲                ← parent of only the exec nodes
 aoe2_aok aoe2_tc aoe2_fe     (DeclareExec — the variants)
    │       │       │         ← + their own content tips
  …content DAG, unchanged, hangs off the variants…
```

(VidyaGod links variants to their tile via the nearest `DeclareLibraryItem` ancestor — `ManifestModel::LinkGames`.)

> *Historical note.* Generation-1 manifests used a `ROLE` field with `SUBGAMES`→`VARIANTS` nesting and a `GROUP`/`GAME`
> string. Roles collapsed into the `Declare*` layers; grouping became a graph edge.

## 3.5 `DeclareRunner` — the executor

A `DeclareRunner` layer makes a node a **runner**: it runs *guest*-platform content while being a *host*-platform
program — a directed edge `GUEST → HOST`. It declares its invocation as data; its **build** (the Proton tree, the
emulator exe) comes from the runner's **`PARENTS`** content nodes, not its own layers. See
[chapter 10](10-platforms-and-runners.md) and [chapter 9 §9.3](09-exec.md).

```jsonc
{ "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["win32", "win64"],
  "EXECUTABLE": "%RunnerMount%/proton", "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
  "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%" }, "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true }
```

Next: [Bundles, the library & indexing](04-bundles-and-library.md).
