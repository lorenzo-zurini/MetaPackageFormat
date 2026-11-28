# 09 · `EXEC`, tiles & variants

An `EXEC` layer says how to run something. An entry without `GUEST` is a **way to run** the node's content; an entry
with `GUEST` makes the node a **runner** (chapter 10). A `TILE` on an entry is what the shelf shows; `VARIANT` on a
node is what the user can pick.

## 9.1 Entry fields

```jsonc
{ "EXEC": {
    "Wipeout XL v1.0": { "HOST": "win32", "EXE": "%GameDir%/Wipeout2.exe", "ARGS": ["-nocd"], "RECOMMENDED": true,
      "TILE": { "UID": "17260", "TITLE": "Wipeout XL",
                "COVER": { "FILE": "WOXL_Cover.png", "SOURCE": "Qm…", "SIZE": 412339 },
                "META": { "YEAR": "1996", "DEVELOPER": "Psygnosis" } } },
    "Setup": { "HOST": "win32", "EXE": "%GameDir%/SETUP.EXE" } } }
```

| Field | Type | Applies to | Meaning |
|-------|------|------------|---------|
| *(the key)* | string, required | both | The entry's **label**: its name and its address, `EXEC/<label>`. Non-empty, without `/` (it separates the address's parts). A `LABEL` field inside an entry is refused. |
| `HOST` | string | both | The platform this entry needs: for a game, its content's platform; for a runner, the platform the runner program is. |
| `GUEST` | list of strings | runner | The platforms the runner provides. Non-empty ⇒ a runner entry. |
| `EXE` | string | both | What to run. A game's: a path in the runtime, from an anchor (`%GameDir%/game.exe`), or empty for content a runner finds itself (a classpath in `ARGS`). A runner's: its program (`%RunnerMount%/proton`), or empty / `%Content%` for a pass-through. |
| `ARGS` | list of strings | both | One element per argv entry, verbatim: spaces are safe and nothing is re-split. An element that substitutes to `""` is dropped. |
| `WORKDIR` | string | both | The working directory. A game's: where it starts (default: the directory of its `EXE`). A runner's: where a game it runs starts when the game names none — for a program that must start in its own folder (an emulator with a portable layout: `%RunnerMount%/…`). A game's own `WORKDIR` wins. |
| `RECOMMENDEDRUNNER` | string | game | A runner entry's label this entry runs best on: a version the game needs, such as a Proton build whose GameInput works. A soft pin (chapter 11 §11.4): the default chain starts with it when it is installed and carries the launch here. Otherwise the default chain is used as if the field were absent. An instance's own choice of runner wins over it. A validator warns when no runner in the library has the label. |
| `TILE` | object | game | What the shelf shows for this entry (§9.3). |
| `RECOMMENDED` | `true` | game | This entry is its card's recommended row (§9.4): one per tile `UID`. Absent means not; `false` is never written. |
| `CONTENT_ROOT`, `PREFIX_GENERATE`, `UNIFIED_RUNTIME`, `GUEST_ROOTS`, `DRIVES` | | runner | The runner's runtime layout (chapter 10). |

`%ContentPath%` and `%Content%` expose a game's `EXE` to its runner (relative to the content root, and as a host
path). An `EXE` outside the content root (a program installed beside the game, in `%ProgramFiles32%`) is still
relative to it, climbing out with `..`, so a runner that names programs as "<guest content root>\%ContentPath:winpath%"
names it too. `%ContentPath%` joins folders with `/`; a Windows runner turns it into one backslashed path with `:winpath`,
as a program that takes its own folder from its command line (everything before the last `\`) reads a mixed path
wrong. `EXE` is matched against the real content case-exactly (chapter 04 §4.4).

An entry carries no environment and no condition of its own: the environment is an `ENV` layer, and a conditional
entry is an `EXEC` layer with a `WHEN`.

## 9.2 Entries are written whole

`EXEC` is `{label: entry}`, and like every address an entry is **written whole** (invariant I10): a later layer's
entry with the same label **replaces** the entry beneath it, every field. Nothing is merged field by field. Entries
keep the order in which each label was **first** written.

An entry is also a **tree of addresses**: `EXEC/Play`, `EXEC/Play/TILE`, `EXEC/Play/TILE/COVER`, `EXEC/Play/ARGS`. A
`TAKE` of a deeper address (chapter 05 §5.2) brings just that part, placed at the same address over the entry the
composition holds there, in list order. So a version that changes only its program writes its whole entry, and a node
that wants only a contained node's cover takes `EXEC/Play/TILE/COVER`:

```jsonc
{ "LABEL": "Wipeout XL v1.1", "VARIANT": "1.1", "LAYERS": [
    { "NODE": "bafkrei…wipeout_1_0" },
    { "DELTA": "1.1.vgdelta", "TARGET": "FILES/%GameDir%", "BASE_SOURCES": ["Qm…"] },
    { "EXEC": { "Wipeout XL v1.1": { "HOST": "win32", "EXE": "%GameDir%/Wipeout2.exe", "TILE": { "UID": "17260" } } } } ] }
```

**`null` deletes an entry**, as it deletes any address: `{"EXEC": {"Play": null, "Wipeout XL v1.1": {…}}}` is a
version renaming the entry it inherits. Without the `null` the inherited `Play` would stay beside the new label, a
second way to run that row.

A node's **default entry** is its recommended entry (§9.4), else the first game entry its own layers write that
carries a tile (its row's face), else an inherited recommended entry, else the first entry of its composition that
carries a tile, else its first own game entry, else the first game entry of its composition, else (a runner) its first
entry with `GUEST`. A launch of a row runs the entry on the launched tile the same way: the row's own (recommended
first) before anything it inherits — an inherited `RECOMMENDED` picks a card's default row, never another row's entry. A launch that names no entry runs it. A node with an
effective entry without `GUEST` is **runnable**: it can be started (by the CLI, by an author testing), whether or not
it is on the shelf.

## 9.3 Tiles

A **tile** is one presentable game: a `TILE` object on an entry.

| Field | Meaning |
|-------|---------|
| `UID` | the tile's identity, a string. Faces with the same `UID` anywhere (other nodes, other packages) are one tile. |
| `PARENTUID` | the family it belongs to: an expansion's base game (`"803"` → `"802"`). Absent: the tile is a family root. |
| `TITLE` | the title shown |
| `COVER` | `{FILE, SOURCE, SIZE}`: an image in the node's package, by file name and CID. `FILE` is required; `SOURCE` and `SIZE` are stamped at publish. |
| `META` | free-form presentation fields (year, developer, series, …) |

An effective entry carrying a tile is a **face** of its node. A tile is part of its entry, so it is written whole with
it; a row that is not its card's recommended one carries only `{"UID"}`, a reference to the card (§9.4).

- **Families.** Tiles nest by `PARENTUID`. The root of the chain is the family's **`%PackageUID%`**: the install
  folder (`%GameDir%` is normally `C:\<PackageUID>`), and the owner of the instance and of everything the user keeps.
  An expansion therefore shares the base game's install and saves. A family's child tiles sort by title.
- **Presentation.** What the shelf shows for a tile (title, cover, meta) comes from its **recommended row's** tile,
  which carries (or takes) the full tile.
- **Across packages.** Anyone may extend a tile: publish a node that contains an existing variant (a new row), or one
  whose entry carries a tile with a new `UID` and `PARENTUID` set (a new face, such as an editor). Only that node is
  new.
- Tiles never name runners.

## 9.4 Variants and rows

A **variant** is a node with `VARIANT: "<name>"`. It is what the user picks. A variant is not a child of a tile: one
node can present several tiles (a Warcraft III version is a row under both Reign of Chaos and The Frozen Throne).

**Rows = variant nodes × the tiled entries in their composition.** Under each tile, the variants that present it are
its rows; picking a row selects that node and that entry. Every build of a game is a row.

- **One recommended row per card.** The entry of exactly one row of each tile says `"RECOMMENDED": true`: the card's
  default, shown first, and the row its presentation comes from. It carries the full tile (`TITLE`, `COVER`, `META`);
  every other row's tile is `{"UID"}`. The other rows follow by name.
- A validator refuses: a `VARIANT` node with no tiled effective entry; a card with **two** recommended rows; a card
  with several rows and **none** recommended.
- A validator SHOULD warn about a tile no variant presents (its card has no row; a part's own card is exempt, its rows
  are the ends it goes into), and about a recommended row whose tile has no `TITLE` (the card would be named by a CID).
- A variant whose parts do not hold (a part it contains goes `INTO` something it does not contain, chapter 12 §12.5)
  is listed but blocked, with the reason.
- An implementation SHOULD hide a tile none of whose rows has content present locally (it is in the catalog, not the
  library).

**Grafts with an entry.** A graft (chapter 23) that has an entry is a **row**, never an option to tick:

- **with its own `TILE`** it is its own card (Forgotten Empires over The Conquerors, Horn of the Abyss over Heroes
  III): on that card, each row is the graft applied over an end it goes `INTO`, and its entry is what runs;
- **without a tile** it must `TAKE` one (`{"INTO": ["…"], "TAKE": ["EXEC/Play/TILE"]}`): it is a row under that card;
- an entry with neither appears nowhere, and a validator refuses it.

Grafts nest: Forgotten Empires goes `INTO` UserPatch, which goes `INTO` 1.0e, which goes `INTO` The Conquerors'
builds. Launching a graft's row applies the chain beneath it; a graft with an entry is never pre-ticked on another row.
A graft without an entry is a plain mod: a ticked option (`"RECOMMENDED": true` on its `INTO` layer ticks it by
default).

## 9.5 Runner entries

```jsonc
{ "geproton_runner": { "HOST": "linux64", "GUEST": ["win32", "win64"],
  "EXE": "%RunnerMount%/proton", "ARGS": ["waitforexitandrun", "%GameDir%\\%ContentPath:winpath%"],
  "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true,
  "GUEST_ROOTS": { "%GameDir%": "C:\\%PackageUID%", "%Documents%": "C:\\users\\steamuser\\Documents" },
  "DRIVES": { "C:": "%PrefixRoot%/drive_c" } } }
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
workdir = the game entry's WORKDIR, else its runner's WORKDIR, else the directory of its EXE
```

Exactly one process is started. Chains of runners nest (chapter 11).

An implementation MAY run a runner with an override target instead of the content, for tooling (generating a prefix
with `wineboot`). That is a runtime mechanism, not a package field.

Next: [Platforms & runners](10-platforms-and-runners.md).
