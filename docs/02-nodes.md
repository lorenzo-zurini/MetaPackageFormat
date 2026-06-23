# 02 · The Node object

A node is a single JSON object stored in its own `<node_id>.json` file. This chapter is the exhaustive field reference.
Layers (`LAYERS`), the exec block (`EXEC`), platforms (`PLATFORM`) and metadata (`META`) are large enough to have their
own chapters; this one defines the *envelope* and links out.

## 2.1 Identity & the minimal node

The only structurally-required field is `NODE_ID`. A file that does not parse as an object with a non-empty string
`NODE_ID` is **not a node** and MUST be ignored by the indexer (it might be unrelated JSON sitting in a bundle).

The minimal valid node:

```json
{ "NODE_ID": "some_unique_id" }
```

It defaults to `ROLE: "content"` with no layers and no parents — an empty content node. Useless alone, but valid.

## 2.2 Field reference

Every field is optional except `NODE_ID`. Unknown fields MUST be ignored (forward-compatibility). Defaults are applied
exactly as listed.

| Field | JSON type | Default | Applies to | Meaning |
|-------|-----------|---------|-----------|---------|
| `NODE_ID` | string | — (required) | all | Globally-unique identity. The currency of every edge. MUST be non-empty and unique across the whole graph (invariant **I1**). Convention: lowercase snake-case slug (`aoe2_aok_base`, `ge-proton10-30`, `super_metroid`). |
| `ROLE` | string | `"content"` | all | `"content"`, `"launchable"`, or `"runner"`. See [ch. 3](03-roles.md). Any other value is treated as `"content"` by a tolerant reader but SHOULD be rejected by a validator. |
| `UID` | string | `""` | launchable (presentable) | A stable numeric-ish identifier for a presentable node — used as the package/save key and (for games) a metadata-DB id. Held as a string ("299", "7804"). Empty for non-presentable nodes. |
| `GAME` | string | `""` | launchable | The grouping key that collapses multiple launchable variants into one library tile (e.g. all editions of "Age of Empires II"). When empty, the node is its own group (the effective group key is the `NODE_ID`). See [ch. 3 §3.4](03-roles.md). |
| `LABEL` | string | `""` | launchable | Human-readable label distinguishing this variant within its `GAME` (e.g. "Original Release", "GOG v2.0"). Shown in the variant picker. |
| `RECOMMENDED` | bool | `false` | launchable, runner | Marks the default/preferred choice within a set: the default variant within a `GAME`, or a preferred runner. The picker presents recommended entries first. |
| `META` | object | absent | launchable (presentable) | Presence of a non-empty `META` object marks a node **presentable** (it appears as a library tile). Holds display metadata: `TITLE`, `COVER`, ids, dates, etc. See [ch. 3 §3.5](03-roles.md). |
| `PLATFORM` | object | absent | launchable, runner | `{ "HOST": <token>, "GUEST": [<token>, …] }`. A launchable sets `HOST` = the platform of its content. A runner sets `HOST` = the platform it runs on, and `GUEST` = the platforms it can run. See [ch. 10](10-platforms-and-runners.md). |
| `EXEC` | object | absent | launchable, runner | The invocation/content block. For a launchable: `CONTENTPATH`/`WORKDIR`/`EXEARGS`. For a runner: `EXECUTABLE`/`ARGS`/`ENV`/`REMOVE_ENV`/`CONTENT_ROOT`/`PREFIX_GENERATE`/`UNIFIED_RUNTIME`/`GUEST_PATH`. See [ch. 9](09-exec.md). |
| `LAYERS` | array | `[]` | content (mainly) | The typed payloads this node contributes: VFS file layers, edit layers, persistence layers, CustomVar knobs. See [ch. 5](05-layers.md)–[ch. 8](08-variables.md). A runner node's *own* `LAYERS` are **ignored** for its build (see [ch. 10 §10.5](10-platforms-and-runners.md)). |
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

## 2.3 A representative launchable

A complete launchable node for a Windows game (from the reference library), annotated:

```json
{
    "NODE_ID": "aom_base_game",          // global id
    "ROLE": "launchable",                 // an entry point
    "UID": "7804",                        // save/package key + metadata-db id
    "GAME": "aom_base",                   // tile grouping key (variants share this)
    "LABEL": "Original Release",          // this variant's label
    "RECOMMENDED": true,                  // the default variant of the tile
    "META": {                             // presence ⇒ shows as a library tile
        "TITLE": "Age of Mythology",
        "COVER": { "PATH": "AoM_Cover.jpg",
                   "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
        "RELEASEDATE": "2002-10-30",
        "DEVELOPER": "Ensemble Studios",
        "UMUID": "266840"                 // a runner knob (see ch. 8/9)
    },
    "PLATFORM": { "HOST": "win32" },      // its content is a win32 program
    "EXEC": {                             // how to invoke it (ch. 9)
        "CONTENTPATH": "aom.exe",
        "EXEARGS": "xres=%ScreenWidth% yres=%ScreenHeight%"
    },
    "PARENTS": [ "aom_base_content" ]      // pulls in the game's files
}
```

## 2.4 A representative content node

```json
{
    "NODE_ID": "aom_base_content",
    "ROLE": "content",
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
    "ROLE": "runner",
    "PLATFORM": { "HOST": "linux64", "GUEST": ["win32", "win64"] },
    "EXEC": {
        "EXECUTABLE": "%RunnerMount%/proton",
        "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
        "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%" },
        "REMOVE_ENV": ["LD_LIBRARY_PATH"],
        "CONTENT_ROOT": "pfx/drive_c/%PackageUID%",
        "PREFIX_GENERATE": true
    },
    "PARENTS": [ "geproton_build" ]        // its runtime build lives on a content parent
}
```

## 2.6 Authoring conventions (non-normative)

- One node per file; name the file after the `NODE_ID` (`super_metroid.json` for `NODE_ID: "super_metroid"`). The indexer
  does not require this, but it keeps a bundle navigable.
- Prefix content nodes that belong to a launchable with the launchable's slug (`aom_base_content`, `aom_titans_expansion`)
  so the graph reads top-down.
- Keep `NODE_ID`s descriptive and stable. They are the public contract: other packages, runner libraries and saved user
  settings reference them by id. Renaming a node id is a breaking change.

Next: [Roles](03-roles.md).
