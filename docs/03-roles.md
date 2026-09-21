# 03 · Node types

A node has **no `ROLE` field**. What a node *is* — content, a registry edit, a launchable, a library tile — **is its
`TYPE`**. Ten types, and the payload of each sits directly on the node:

| `TYPE` | The node is… | Payload chapter |
|--------|--------------|-----------------|
| **`Content`** | files mounted into the runtime — a zip, a directory, a single file, or a binary delta over one | [ch. 5](05-layers.md) |
| **`RegEdit`** | registry keys and values written into the prefix, per architecture | [ch. 6 §6.1](06-edit-layers.md) |
| **`FileEdit`** | text edits applied to a file in the runtime | [ch. 6 §6.3](06-edit-layers.md) |
| **`BinaryPatch`** | byte patches over the **pristine** executable, each guarded by an `EXPECT` check | [ch. 6 §6.4](06-edit-layers.md) |
| **`DllOverride`** | which DLLs resolve native vs builtin | [ch. 6 §6.2](06-edit-layers.md) |
| **`Persist`** | what survives the run: `KEEP` promotes paths/registry keys, `DROP` makes them ephemeral | [ch. 7](07-persistence.md) |
| **`CustomVar`** | a variable the player sets before launch, substituted as `%KEY%` wherever it is used | [ch. 8](08-variables.md) |
| **`DeclareExec`** | **what to run.** No `GUEST` ⇒ a launchable; with `GUEST` ⇒ a runner providing those platforms | [ch. 9](09-exec.md) |
| **`DeclareLibraryItem`** | the library tile: title, UID, cover — and the **parent** of the launchables it groups | §3.3 below |
| **`Group`** | pure composition: no payload, exists only to gather `PARENTS` under one name | §3.1 below |

This is the conclusion of "everything is a node": there is one primitive — the layer — and a node **is** one.

## 3.1 `Group` — the building block that carries nothing

A node with no payload. It contributes nothing of its own and exists to gather other nodes under one name, so that a
referrer can depend on the whole set with one edge.

```json
{ "LABEL": "morrowind_data", "TYPE": "Group",
  "PARENTS": ["morrowind_textures_hd", "morrowind_bloodmoon", "morrowind_tribunal"] }
```

`Group` is not cosmetic, and an implementation MUST NOT optimise it away: something points at it by name. Dropping a
payload-less node makes every referrer *silently lose an edge* rather than dangle — the failure has no diagnostic, and
the package just quietly does less.

## 3.2 `DeclareExec` — launchable **and** runner

One type, two readings, decided by a single field:

- **No `GUEST`** (absent or empty) ⇒ the node is a **launchable**: an entry point. `HOST` is the platform its content
  *needs*, and the runtime derives the runner chain to reach it ([ch. 11](11-runner-chaining.md)).
- **`GUEST` non-empty** ⇒ the node is a **runner**: a program that runs `GUEST`-platform content while itself being a
  `HOST`-platform program — a directed edge `GUEST → HOST`.

```jsonc
// launchable: needs win32, provides nothing
{ "LABEL": "aoe2_tc", "TYPE": "DeclareExec", "PARENTS": ["aoe2_tc_content", "aoe2"],
  "HOST": "win32", "PATH": "age2_x1/age2_x1.exe",
  "LABEL": "The Conquerors", "RECOMMENDED": true }

// runner: needs linux64, provides win32+win64
{ "LABEL": "ge-proton10-30", "TYPE": "DeclareExec", "PARENTS": ["geproton_build"],
  "HOST": "linux64", "GUEST": ["win32", "win64"],
  "PATH": "%RunnerMount%/proton", "ARGS": ["waitforexitandrun", "%Content%"],
  "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%" }, "ENV_REMOVE": ["LD_LIBRARY_PATH"],
  "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true }
```

The two used to be separate types (`DeclareExec` with a `PLATFORM`, `DeclareRunner` with `HOST`+`GUEST`), which said
the same thing twice: *a launchable is a runner that provides nothing*. Unifying them is why chaining needs no special
case for the ends of the chain. Full field reference in [chapter 9](09-exec.md).

A launchable does **not** name a runner — it states `HOST`, and the runtime *derives* the chain. Its own payload sits
at the **top** of the overlay, so a launchable variant is the natural home for overrides.

## 3.3 `DeclareLibraryItem` — the library tile (a game)

A pure-metadata node: no content, no edits. It makes a **presentable library tile** — a *game*.

```jsonc
{ "LABEL": "aoe2", "TYPE": "DeclareLibraryItem",
  "UID": "749", "TITLE": "Age of Empires II",
  "COVER": { "PATH": "cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
  "META": { "DEVELOPER": "Ensemble Studios", "SERIES": "Age of Empires" } }
```

| Field | Type | Meaning |
|-------|------|---------|
| `UID` | string | Stable identity for *the game*. Keys saved state, settings and the content root inside a prefix. **Required**: a tile with no `UID` MUST fail validation — see [ch. 15](15-validation.md). |
| `TITLE` | string | The tile's name. Falls back to the `LABEL` if absent. |
| `COVER` | object | `{ "PATH": …, "SOURCE": { "TYPE": "ipfs", "CID": … } }`. Cover CIDs participate in publishing/seeding like content ([ch. 14](14-content-addressing.md)). |
| `META` | object | Free-form descriptive metadata (release date, developer, series, external ids…). Implementations MUST ignore keys they do not know. |

A tile is **never launchable on its own**. To launch, a `DeclareExec` node must reach it through `PARENTS`.

## 3.4 Variants & games — grouping is a graph edge

Several launchables that are *the same game in different editions/versions* are grouped into **one tile**: the user
opens the tile and picks a **variant**. Grouping is a **`PARENTS` edge to the game's `DeclareLibraryItem` node** — there
is no `GAME` string:

- A **game** is a `DeclareLibraryItem` node. It carries no content.
- A **variant** is a `DeclareExec` node that has that tile as a `PARENTS` ancestor. So the tile is a parent of *only the
  exec nodes*; each variant's content chain hangs off the variant separately. Variants **inherit** the tile's metadata
  (it composes down the closure) and contribute their own `LABEL`/`RECOMMENDED`.
- A single-variant game is the same shape with one exec — the tile stays its own node.
- Within a tile, `RECOMMENDED: true` marks the default variant; else the implementation picks deterministically.

```
        aoe2  (DeclareLibraryItem — the tile)
       ╱  │  ╲                ← a PARENT of the exec nodes
 aoe2_aok aoe2_tc aoe2_fe     (DeclareExec, no GUEST — the variants)
    │       │       │         ← + their own content chains
  …content chain, unchanged, hangs off each variant…
```

**The direction is deliberate and final.** The tile is upstream so that a variant *inherits* it, and so that the
launchable — the thing you actually run — stays the terminal node of its chain. A launchable that reaches two different
tiles is an **ambiguous tile** and MUST fail validation: nothing can decide which game it belongs to.

(VidyaGod links variants to their tile via the nearest `DeclareLibraryItem` ancestor — `ManifestModel::LinkGames`.)

> *Historical note.* Generation-1 manifests used a `ROLE` field with `SUBGAMES`→`VARIANTS` nesting and a `GROUP`/`GAME`
> string. Roles collapsed into `Declare*` layers; then layers collapsed into nodes and the layer's `TYPE` became the
> node's. Grouping was a graph edge throughout.

Next: [Bundles, the library & indexing](04-bundles-and-library.md).
