# 09 · `EXEC`, tiles & variants

An `EXEC` layer says how to run something. An entry without `GUEST` is a **way to run** the node's content; an entry
with `GUEST` makes the node a **runner** (chapter 10). A `TILE` on an entry is what the shelf shows; `VARIANT` on a
node is what the user can pick.

## 9.1 Entry fields

```jsonc
{ "EXEC": [
    { "LABEL": "Play", "HOST": "win32", "EXE": "%GameDir%/Wipeout2.exe", "ARGS": ["-nocd"],
      "TILE": { "UID": "17260", "TITLE": "Wipeout XL",
                "COVER": { "FILE": "WOXL_Cover.png", "SOURCE": "Qm…", "SIZE": 412339 },
                "META": { "YEAR": "1996", "DEVELOPER": "Psygnosis" } } },
    { "LABEL": "Setup", "HOST": "win32", "EXE": "%GameDir%/SETUP.EXE" } ] }
```

| Field | Type | Applies to | Meaning |
|-------|------|------------|---------|
| `LABEL` | string, required | both | The entry's name, unique within the layer. Its address: `EXEC/<label>`. |
| `HOST` | string | both | The platform this entry needs: for a game, its content's platform; for a runner, the platform the runner program is. |
| `GUEST` | list of strings | runner | The platforms the runner provides. Non-empty ⇒ a runner entry. |
| `EXE` | string | both | What to run. A game's: a path in the runtime, from an anchor (`%GameDir%/game.exe`), or empty for content a runner finds itself (a classpath in `ARGS`). A runner's: its program (`%RunnerMount%/proton`), or empty / `%Content%` for a pass-through. |
| `ARGS` | list of strings | both | One element per argv entry, verbatim: spaces are safe and nothing is re-split. An element that substitutes to `""` is dropped. |
| `WORKDIR` | string | game | The working directory. Default: the directory of `EXE`. |
| `TILE` | object | game | What the shelf shows for this entry (§9.3). |
| `CONTENT_ROOT`, `PREFIX_GENERATE`, `UNIFIED_RUNTIME`, `GUEST_ROOTS`, `DRIVES` | | runner | The runner's runtime layout (chapter 10). |

`%ContentPath%` and `%Content%` expose a game's `EXE` to its runner (relative to the content root, and as a host
path). `EXE` is matched against the real content case-exactly (chapter 04 §4.4).

An entry carries no environment and no condition of its own: the environment is an `ENV` layer, and a conditional
entry is an `EXEC` layer with a `WHEN`.

## 9.2 The `EXEC` fold

Entries fold **per label, per field**. A later layer's entry with the same `LABEL` changes only the fields it gives:
objects merge recursively, any other value replaces, `null` deletes a field. The folded entries keep the order in
which each label was **first** declared.

This is how one entry serves a whole version history: the pristine declares `Play` with its `EXE` and tile, a later
version that renamed the executable declares `{"LABEL": "Play", "EXE": "%GameDir%/new.exe"}`, and everything else
of the entry (its tile, its args) stays. Games conventionally label their main entry `Play`, so that every version's
entry is the same entry.

A node's **default entry** is its first folded entry without `GUEST` (for a runner, its first entry with `GUEST`).
A node with an effective entry without `GUEST` is **runnable**: it can be started (by the CLI, by an author
testing), whether or not it is on the shelf.

## 9.3 Tiles

A **tile** is one presentable game: a `TILE` object on an entry.

| Field | Meaning |
|-------|---------|
| `UID` | the tile's identity, a string. Faces with the same `UID` anywhere (other nodes, other packages) are one tile. |
| `PARENTUID` | the family it belongs to: an expansion's base game (`"803"` → `"802"`). Absent: the tile is a family root. |
| `TITLE` | the title shown |
| `COVER` | `{FILE, SOURCE, SIZE}`: an image in the node's package, by file name and CID |
| `META` | free-form presentation fields (year, developer, series, …) |

An effective entry carrying a tile is a **face** of its node. Because the entry folds, every version shows the tile
its pristine declared until one redefines a field (`EXEC/Play/TILE/TITLE`).

- **Families.** Tiles nest by `PARENTUID`. The root of the chain is the family's **`%PackageUID%`**: the install
  folder (`%GameDir%` is normally `C:\<PackageUID>`), and the owner of the instance and of everything the user keeps.
  An expansion therefore shares the base game's install and saves. A family's child tiles sort by title.
- **Presentation.** What the shelf shows for a tile (title, cover, meta) comes from its recommended variant's folded
  tile, else its first variant's (by variant name), else any node presenting it.
- **Across packages.** Anyone may extend a tile: publish a node that contains an existing variant (a new row), or one
  whose entry carries a tile with a new `UID` and `PARENTUID` set (a new face, such as an editor). Only that node is
  new.
- Tiles never name runners.

## 9.4 Variants and rows

A **variant** is a node with `VARIANT: "<name>"`. It is what the user picks. A variant is not a child of a tile: one
node can present several tiles (a Warcraft III version is a row under both Reign of Chaos and The Frozen Throne).

**Rows = variant nodes × the tiled entries in their fold.** Under each tile, the variants that present it are its
rows; picking a row selects that node and that entry.

- `RECOMMENDED` on a variant lists the tile UIDs under which it is the default row. Recommended rows come first,
  then the rest by name.
- A validator refuses a `VARIANT` node with no tiled effective entry, and a `RECOMMENDED` UID the node does not
  present.
- A variant whose requirements fail (an `ANY` unmet, a `NOT` hit, chapter 12 §12.5) is listed but blocked, with the
  reason.
- An implementation SHOULD hide a tile none of whose rows has content present locally (it is in the catalog, not the
  library).

**Grafts with a face.** A graft (chapter 12 §12.5) whose entry carries a tile gets its own card: on that tile, the
graft is applied first on every row it is offered to, and its entry is what runs. A mod that is effectively its own
game (Horn of the Abyss over Heroes III) is published this way.

## 9.5 Runner entries

```jsonc
{ "LABEL": "geproton_runner", "HOST": "linux64", "GUEST": ["win32", "win64"],
  "EXE": "%RunnerMount%/proton", "ARGS": ["waitforexitandrun", "%GameDir%\\%ContentPath%"],
  "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true,
  "GUEST_ROOTS": { "%GameDir%": "C:\\%PackageUID%", "%Documents%": "C:\\users\\steamuser\\Documents" },
  "DRIVES": { "C:": "%PrefixRoot%/drive_c" } }
```

A runner entry is a launcher definition; its fields are chapter 10's. A runner needs no tile and no variant: it is
found by capability.

A runner whose `EXE` is empty or `%Content%` is a **pass-through**: innermost, it runs the content's own executable;
as an outer wrapper it forwards the inner command unchanged. This is how the native terminal works.

## 9.6 How a command is built (one runner)

```
program = the runner's EXE (substituted)                       # /…/RUNNER/proton
args    = the runner's ARGS (substituted, empty ones dropped)   # waitforexitandrun  C:\7804\aom.exe
        + the game entry's ARGS (substituted, empty ones dropped)
env     = chapter 07 §7.3
workdir = the game entry's WORKDIR, else the directory of its EXE
```

Exactly one process is started. Chains of runners nest (chapter 11).

An implementation MAY run a runner with an override target instead of the content, for tooling (generating a prefix
with `wineboot`). That is a runtime mechanism, not a package field.

Next: [Platforms & runners](10-platforms-and-runners.md).
