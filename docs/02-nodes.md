# 02 · The Node object

A node is a single JSON object stored in its own `<node_id>.json` file. This chapter is the exhaustive field reference.
The envelope is tiny — `NODE_ID`, `PARENTS`, `LAYERS`, and a few selection flags. A node has **no `ROLE`**: its identity
(content / launchable / library tile / runner) emerges from which `Declare*` layers it carries ([ch. 3](03-roles.md)),
and those — like the content/edit/persistence/CustomVar layers — live in `LAYERS` and have their own chapters.

## 2.1 Identity & the minimal node

The only structurally-required field is `NODE_ID`. A file that does not parse as an object with a non-empty string
`NODE_ID` is **not a node** and MUST be ignored by the indexer (it might be unrelated JSON sitting in a bundle).

The minimal valid node:

```json
{ "NODE_ID": "some_unique_id" }
```

With no `Declare*` layer, no other layers and no parents, it is an empty **content** node. Useless alone, but valid.

## 2.2 Field reference

Every field is optional except `NODE_ID`. Unknown fields MUST be ignored (forward-compatibility). Defaults are applied
exactly as listed.

| Field | JSON type | Default | Applies to | Meaning |
|-------|-----------|---------|-----------|---------|
| `NODE_ID` | string | — (required) | all | Globally-unique identity. The currency of every edge. MUST be non-empty and unique across the whole graph (invariant **I1**). Convention: lowercase snake-case slug (`aoe2_aok_base`, `ge-proton10-30`, `vortex_quest`). |
| `LAYERS` | array | `[]` | all | The typed payloads this node contributes: content/edit/persistence/CustomVar layers **and** the `Declare*` identity layers (`DeclareExec`/`DeclareLibraryItem`/`DeclareRunner`) that make it launchable / a library tile / a runner — there is **no `ROLE`/`EXEC`/`META`/`PLATFORM`/`UID`/`GAME`/`LABEL`/`RECOMMENDED` field**; identity emerges from these layers (see [ch. 3](03-roles.md)). A runner node's *own* VFS layers are ignored for its build (see [ch. 10 §10.5](10-platforms-and-runners.md)). |
| `PARENTS` | array of string | `[]` | all | The `NODE_ID`s this node selects/depends on. **List order is significant**: a parent listed *later* has *higher* overlay priority (its layers win on conflict). See [ch. 12](12-resolution.md). |
| `OPTIONAL` | bool | `false` | content | When this node is referenced as a parent, it is a *toggleable* add-on rather than a hard dependency. Off-by-default unless `DEFAULT` says otherwise. See [ch. 12 §12.3](12-resolution.md). |
| `DEFAULT` | bool | `true` | content | The initial on/off state of an `OPTIONAL` node when the user hasn't chosen. Ignored when `OPTIONAL` is false (the node is always pulled). |
| `EXCLUDE` | array of string | `[]` | content | `NODE_ID`s this node is mutually exclusive with. **Symmetry is expected** (both nodes should list each other); validators warn if not. When two excluded nodes would both be enabled, first-kept wins. See [ch. 12 §12.4](12-resolution.md). |

Two fields are **derived, not authored** (an implementation computes them at index time, they never appear in JSON):

- **source file path** — the `.json` file the node was read from.
- **bundle directory** — the directory containing that file. **All relative paths inside this node's `LAYERS` and
  `META.COVER` resolve against this node's own bundle directory** — which makes cross-bundle references correct (a
  runner's build layers resolve against the runner's bundle even when pulled into a game's closure). See
  [ch. 5 §5.4](05-layers.md).

## 2.3 A representative launchable (single-variant game)

A complete entry-point node for a Windows game — a `DeclareLibraryItem` (the tile) + a `DeclareExec` (how to run) on one
node, since it's a single-variant game:

```json
{
    "NODE_ID": "aom_base_game",
    "PARENTS": [ "aom_base_content" ],
    "LAYERS": [
        { "TYPE": "DeclareLibraryItem", "UID": "7804",
          "TITLE": "Age of Mythology",
          "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
          "RELEASEDATE": "2002-10-30", "DEVELOPER": "Ensemble Studios", "UMUID": "266840" },
        { "TYPE": "DeclareExec", "PLATFORM": "win32",
          "CONTENTPATH": "aom.exe", "EXEARGS": "xres=%ScreenWidth% yres=%ScreenHeight%" }
    ]
}
```

## 2.4 A representative content node

```json
{
    "NODE_ID": "aom_base_content",
    "LAYERS": [
        { "TYPE": "VFSZipLayer", "PATH": "aom.zip",
          "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } }
    ]
}
```

## 2.5 A representative runner node

```json
{
    "NODE_ID": "ge-proton10-30",
    "PARENTS": [ "geproton_build" ],
    "LAYERS": [
        { "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["win32", "win64"],
          "EXECUTABLE": "%RunnerMount%/proton",
          "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
          "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%" },
          "REMOVE_ENV": ["LD_LIBRARY_PATH"],
          "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true }
    ]
}
```

## 2.6 Authoring conventions (non-normative)

- One node per file; name the file after the `NODE_ID` (`vortex_quest.json` for `NODE_ID: "vortex_quest"`). The indexer
  does not require this, but it keeps a bundle navigable.
- Prefix content nodes that belong to a launchable with the launchable's slug (`aom_base_content`, `aom_titans_expansion`)
  so the graph reads top-down.
- Keep `NODE_ID`s descriptive and stable. They are the public contract: other packages, runner libraries and saved user
  settings reference them by id. Renaming a node id is a breaking change.

Next: [Roles](03-roles.md).
