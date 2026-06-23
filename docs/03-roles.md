# 03 · Roles

A node's `ROLE` decides how the runtime treats it. There are exactly three roles; the default is `content`.

| `ROLE` | Started? | Carries content? | Has `PLATFORM`? | Purpose |
|--------|----------|------------------|-----------------|---------|
| `content` | never directly | yes (`LAYERS`) | no | The building blocks: files, edits, persistence, knobs; and grouping via `PARENTS`. |
| `launchable` | yes (entry point) | usually via parents | `HOST` only | A thing the user starts: a game, tool, or edition. |
| `runner` | only as an executor | via parents (its *build*) | `HOST` + `GUEST[]` | Executes content of guest platforms while running on a host platform. |

## 3.1 `content` — the building block

The default role. A content node contributes `LAYERS` and/or groups other content through `PARENTS`. It is never an entry
point and is never executed on its own; it enters a runtime only when some launchable's or runner's closure pulls it in.

Content nodes are where almost everything *lives*: a game's files, a mod's overlay, an expansion's data, a registry tweak,
a persistence rule, a runtime's binaries. A content node may be pure grouping (only `PARENTS`, no `LAYERS`) — a useful
way to bundle several optional sub-pieces behind one toggle.

```json
{ "NODE_ID": "morrowind_data", "ROLE": "content",
  "PARENTS": ["morrowind_textures_hd", "morrowind_bloodmoon", "morrowind_tribunal"] }
```

Selection attributes that make a content node behave as a toggle (`OPTIONAL`, `DEFAULT`) or a choice
(`EXCLUDE`) are defined in [chapter 12](12-resolution.md).

## 3.2 `launchable` — the entry point

A launchable is what a user picks and starts. It declares:

- **`PLATFORM.HOST`** — the platform of *its content* (what the chain must run): `"win32"`, `"snes"`, `"linux64"`, …
- **`EXEC`** — how to invoke the content once mounted: `CONTENTPATH` (the path, relative to the content mount, of the
  thing to run — an exe, a ROM, a data root, or nothing for self-contained content), plus optional `WORKDIR` and
  `EXEARGS`. See [chapter 9](09-exec.md).
- **`PARENTS`** — the content closure that supplies the files.
- Usually **`META`** (so it appears in a library) and grouping fields (`GAME`, `LABEL`, `RECOMMENDED`, `UID`).

A launchable does **not** specify *which* runner runs it. It states the platform of its content; the runtime *derives*
the runner chain (chapter 11). This is the crucial inversion of control: launchables describe *what they are*, not *how
to run them*. The same launchable runs via Wine on Linux today and via a native Windows runner on Windows tomorrow with
no change.

A launchable MAY carry its own `LAYERS` (in addition to its parents'). Those layers sit at the **top** of the overlay
(highest priority), which makes a launchable a natural place for variant-specific overrides — e.g. a GOG edition that
replaces one executable.

## 3.3 `runner` — the executor

A runner runs *guest*-platform content while being a *host*-platform program. It is a directed edge `GUEST → HOST` in the
platform graph. Examples:

- **Proton/Wine** — `HOST: "linux64"`, `GUEST: ["win32","win64"]`. Runs Windows programs on Linux; generates a prefix.
- **An emulator** — e.g. snes9x — `GUEST: ["snes"]`, `HOST` = whatever the emulator binary is (`"linux64"` for a native
  build, `"win32"` for a Windows build).
- **A native pass-through** — `HOST: "linux64"`, `GUEST: ["linux64"]`, empty/`%Content%` executable. The universal
  terminal that simply runs the inner command (chapter 11).

A runner declares its invocation as data in `EXEC` (executable, args, env, content placement, prefix needs); see
[chapter 9 §9.3](09-exec.md). A runner's **build** — the binaries it needs (the Proton tree, the emulator exe) — is **not**
its own `LAYERS`; it comes from the runner's **`PARENTS`** content nodes. (A runner's own VFS layers are deliberately
ignored — see [ch. 10 §10.5](10-platforms-and-runners.md).) This keeps the runner node a thin description and the heavy
bytes on ordinary content nodes that can be shared and content-addressed like anything else.

Runners almost always live in a dedicated runner library (a separate bundle/repo), shared across every game. A runner MAY
also be **embedded** in a game's bundle (shipped alongside the launchable) — useful for a game-specific or experimental
executor. There is no "embedded" flag: a runner is a runner wherever its file sits; the only difference is bundle
locality, which the resolver uses as a tie-break (chapters 11–12).

## 3.4 Variants and the `GAME` grouping

Several launchable nodes that represent *the same game in different editions/versions* share a `GAME` value. The library
collapses them into **one tile**; the user opens the tile and picks a **variant** from a dropdown.

```
GAME = "aoe2_aok"
 ├── aoe2_aok            LABEL "Original (Forgotten Empires patch)"   RECOMMENDED
 └── aoe2_aok_gog        LABEL "GOG edition"
```

Rules:

- The grouping key is the value of `GAME`. If `GAME` is empty/absent, the node forms its own one-variant group keyed by
  its `NODE_ID` (so every launchable belongs to exactly one group, possibly of size one).
- Within a group, `RECOMMENDED: true` marks the default variant (the one pre-selected, shown first). If none is marked,
  the implementation chooses deterministically (e.g. by id).
- Each variant is a full launchable with its own `PLATFORM`, `EXEC`, `PARENTS` and `LAYERS`. Variants need not share a
  platform or even a runner — a "native Linux" variant and a "Windows" variant of the same game can coexist under one
  tile, each resolving its own chain.

> *Historical note.* Generation-1 manifests modelled this with a `SUBGAMES`→`VARIANTS` nesting and a `GROUP` field; the
> field was renamed `GAME` and variants became first-class launchable nodes. Old material may say `GROUP`/`VARIANT_ID`.

## 3.5 Presentable nodes & `META`

A node is **presentable** — eligible to appear in a library/browser UI — iff it carries a non-empty `META` object.
`content` and `runner` nodes normally have no `META` and are not presentable; `launchable` nodes normally do.

`META` is display metadata. No field is structurally required, but these are conventional:

| `META` field | Meaning |
|--------------|---------|
| `TITLE` | Human title shown in the UI. Falls back to the `NODE_ID` if absent. |
| `COVER` | Cover art: `{ "PATH": "<file>", "SOURCE": { "TYPE": "ipfs", "CID": "…" } }` — a content-addressed image resolved against the node's bundle dir (see [ch. 14](14-content-addressing.md)). |
| `RELEASEDATE` / `EDITIONDATE` | ISO dates for sorting/display. |
| `EDITION` | Edition label (legacy display; prefer `LABEL` for the variant picker). |
| `DEVELOPER`, `PUBLISHER`, `SERIES`, `SERIESSORTNUMBER` | Catalog/sort metadata. |
| `TGDBID` / `GAMEUID` | External metadata-database ids (e.g. TheGamesDB). |
| `UMUID` | A runner knob surfaced as the `%UMUID%` token (a Steam/UMU app id used by some Wine runtimes). See [ch. 8](08-variables.md). |

`META` is free-form: implementations MUST ignore unknown keys and MAY surface any they understand. Cover CIDs participate
in publishing/seeding exactly like layer content (chapter 14).

Next: [Bundles, the library & indexing](04-bundles-and-library.md).
