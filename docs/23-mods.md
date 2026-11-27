# 23 · Mods

> **Status (2026-11-27).** Sections 23.1–23.3 and 23.5–23.9 are **normative and implemented** (engine, validator,
> package editor, CLI). Section 23.4 (merging by units) is **normative design, partly implemented**: the fold's
> addresses merge today; units inside files and archives arrive with chapter 22's descriptions. The user's words that
> settled each point are quoted where they decided it.

Today's mod managers model a mod as a list of files and resolve overlap by priority: the later file wins, whole. That
breaks wherever a mod is an edit rather than a file (binary patches, patches for other mods, edits inside archives,
INI tweaks, a mod made for another game version) and it needs an order between mods, which nobody can know: "that
would imply knowledge of all mods, including future ones". This chapter models a mod as a **part that goes into a
state**, merged with the other parts by units, never stacked by a list.

## 23.1 Ends and parts

**Ends.** A launchable is an END: a node with `VARIANT` (chapter 09). It is a complete state, the game fully built and
ready to run, and every mod is made against one: "there is no mod that applies to a half-installed game". An end
knows all its ancestors through its chain. An end contained by another end is just another, lower end.

**One relation, in two directions.**

| Layer | Direction | Who composes | Order |
|---|---|---|---|
| `NODE X` | downward: "X is part of me" | the author | the author's list (later wins) |
| `INTO [E, …]` | upward: "I am part of E" (E never names me) | the user, at launch | none: parts merge (23.3, 23.4) |

Containment composes upward, one minted node per composition: without `INTO`, modding an end means one node per mod
and per combination of mods. `INTO` attaches any number of parts to an end without minting anything.

**A part** (a mod; "graft" is the same word) is a node whose list **opens** with one or more `INTO` layers:

```json
{ "LABEL": "SkyUI - USSEP Patch v1.2", "LAYERS": [
    { "INTO": ["bafkrei…skyui52"] },
    { "INTO": ["bafkrei…ussep432", "bafkrei…ussep433"] },
    { "ZIP": "SkyUI - USSEP Patch (Nexus 1.2).zip", "TARGET": "FILES/%GameDir%/Data" } ] }
```

- **Each `INTO` layer lists alternatives**: the part goes into any one of them (made for build A or build B).
- **Several `INTO` layers mean all of them**: the part goes into each (a patch between two mods). It is offered, and
  applies, only once every layer has a member that is the end launched or a part applied to it.
- **`INTO` layers open the list.** What a part goes into comes before what it brings; an `INTO` after any other layer is
  refused (16).
- **The hard link.** A part is offered where what it goes into IS the end launched, or a part applied to it, never
  because that node lies somewhere beneath. An end containing an end is another end: "a mod for 1.0 SHOULDN'T reach
  1.5; if it's compatible, say so" (the part's `INTO` lists 1.5 too, or a claim part does: 23.6).
- **What a part goes into is an end or another part**, never a runner ("a package never names a runner CID, EVER, only
  its capability") and never plain content. The validator refuses both (16).
- **Siblings off the launched end's path are discarded.** Running Forgotten Empires (an end built `INTO` UserPatch,
  which goes `INTO` 1.0e) applies none of the other parts that go into 1.0e: a 1.0e mod that works on Forgotten Empires
  says so itself ("the author's problem").
- **An author who needs an order writes containment.** An end needing UserPatch then BuserBatch on 1.0e is
  `{INTO [1.0e], NODE UserPatch, NODE BuserBatch, its own layers}`: the author vouches for what it stacks. A part a node
  contains is part of that node, never offered again.
- **A part with its own tile is an end** (it carries `VARIANT` and a `TILE`): Forgotten Empires, Enderal, a curated mod
  set, a translation, a speedrun setup, a personal card. "A variant, a graft that's a version, a graft with its own
  tile: they're the same thing, a launchable." Making a new tile for any reason is making such a part (23.5).

## 23.2 Conditions: `WHEN` over variables and node CIDs

There is no `ANY` and no `NOT`. Every check is conditional logic in `WHEN` (08 §8.6): a node CID used as an operand
reads `true` when that node is in the composition and `false` when it is not.

```json
{ "INTO": ["bafkrei…game"], "WHEN": "bafkrei…aok_soundtrack == false" }
{ "NODE": "bafkrei…ussep_patch", "WHEN": "%sse_ae% == 1 && bafkrei…ussep432 == true" }
```

- **Node CIDs only.** Content is always wrapped in a node.
- **Never a runner's CID** (refused by the validator).
- **A `WHEN` on an `INTO` layer gates the whole part**: false, the part is not applied at all, never partly. "Never with
  X" is `X == false` there.
- **Exclusive parts** are written as these conditions, on one side or both ("the WHEN conditions are cleaner"); ticking
  either side unticks the other in the launch window and in `grafts tick`.
- **Evaluated before the launch window is shown.** The whole expansion is resolved first: the end, the selected parts
  (the ticks are the input), every `WHEN`. A part's condition reads the selection; a condition inside a node sees what
  lies beneath it and the selected parts, so nothing loops.
- **A CID that names nothing here reads false**, and that is the author's problem: it may name a part in a package that
  need not exist. The validator warns.
- **A condition's CID is a comparison, never a link.** It is not followed by any closure (sharing, hydrating, keeping),
  but it is renamed wherever the node it names is re-minted (`catalog repoint`, a save's cascade). Before a save, an
  editor's draft handle (`draft-` and 12 hex digits) stands in for the CID and is renamed the same way.

## 23.3 Parts are a set

"Unlike the contained nodes, grafts have no order", and none can be chosen. So:

- **The selection is a set.** The instance keeps which parts are ticked (and their options), never an order. There is no
  reordering, in the launch window or anywhere else.
- **A part applies after what it goes into**: one `INTO` step from the end, then two, and so on. Parts at the same depth
  are folded in the order of their CIDs: a reading order, never a preference. Where two of them write the same unit
  differently, that is a conflict (23.4), shown, never settled by that order.
- **The launch window** lists the offered parts after what they go into, then by `LABEL`, then CID (presentation). A
  part needing another (a part of a part, or several `INTO` layers) appears once what it needs is ticked.
- **A fresh instance** ticks the parts `RECOMMENDED` under the launched tile.
- **On the command line:** `launch plan|run --with <part,…>` adds parts to the selection the launch would use;
  `launch plan` reports the parts applied (`applied_parts`) and those chosen but not applied (`dropped_parts`: not
  offered here, a condition false, not installed). `grafts tick` computes a selection after one tick.

## 23.4 Merging by units

Parts that go into the same thing MERGE, per unit. What a unit is, and how two writes to it combine, are declared as
data, never coded: "we need merging primitives for each mutation type, for each archive type... all need to be data".

- **Every mutation type has units**: a content layer's files (compared without case, as the fold's `FILES/`
  addresses are); `ConfigWrite`, `JsonSet`, `XmlSet`, `LineFlag` their keys (already keyed addresses, `file#key`);
  `AppendLine` its line; a binary edit its byte range; `Delete` its path; `REG` a value; `VARS`, `ENV`, `DLL` a name; a
  `DELTA` its edit script in base coordinates (the ranges of its base it keeps, deletes, replaces).
- **Every format description declares its units and nests them** (chapter 22): a zip's or a BSA's members, a plugin's
  records (FormID) and their fields, an INI's keys, a text file's lines. A unit that is itself a described container
  merges by its own units, to any depth ("each format has an atomic unit: for a zip it is a file, for a BSA it is this").
- **Every unit kind names its merge from a closed set of primitives**, versioned like chapter 22's instruction set:
  equal-or-conflict (the default; identical writes are one write), disjoint ranges (byte ranges, delta edit scripts:
  overlap is a conflict), set union (appended lines), commutative bits (`Or`), recurse (a described container). A
  removal against a write is a conflict. A missing primitive is added to the set, never written as code.
- **Addresses are units are `TAKE` selectors.** The same addresses name what a part writes, what two parts conflict
  on, and what a `TAKE` selects, in both directions (`NODE`'s `TAKE` downward, a part's footprint upward), down to an
  archive's members and a plugin's records.
- **A conflict** names its unit, the parts and their values. It is resolved by structure, never by an order: a
  compatibility patch that goes into both (it applies after both, so its write stands), or the user's resolution kept
  in the instance (by default one choice for everything two parts share, or per unit). A published resolution is a part
  that goes into both.
- **The same primitives merge sibling deltas** (three delta mods from one root), so mods and the delta format share one
  merge algebra.

Today: parts fold by the fold's addresses, at different depths later wins, at one depth in CID order. Conflicts are
found per unit and SHOWN (the launch window's add-ons tab, `launch plan`'s `PartConflicts`, `grafts conflicts`): a
file path (compared without case; an archive's members are read from the archive), a keyed edit (`file#key`), a binary
edit's byte range (disjoint ranges merge: Wipeout XL's 200 fps and widescreen patch one exe at different offsets), an
anchored edit's anchor, an appended line (a set: two different lines never conflict), a registry value, `ENV`, `DLL`,
`VARS`. Identical writes are one write, and a part never conflicts with one it goes into. **The user's resolution** is
kept in the instance (`RESOLUTIONS`: {address: the part whose write stands}; the window's "Keep …" buttons, `grafts
resolve`): parts are then folded in an order where each still follows what it goes into and every kept part comes as
late as that allows, all others keeping theirs. A resolution orders whole parts, so one that cannot hold (stale: its
part is not one of the two, or not applied; or two resolutions needing a pair both ways) is shown as an open conflict,
never as settled. Units inside BSAs and plugins (records) arrive with their descriptions (chapter 22).

## 23.5 Tiles, identity and the package

- **Tiles stay in the nodes**, on the launchable they present (chapter 09), as close as possible to the content they
  describe. "In both cases something gets re-minted, might as well not complicate things further and just keep it in
  the nodes." A cover or a recommendation changing re-mints that launchable; nothing else.
- **A new tile is a part**: a node `INTO` an end that carries its own `TILE` (and `VARIANT`), with or without parts of
  its own. Its saves follow its `UID` and `PARENTUID` (chapter 13): in the base game's family it shares that family's
  saves; with a `UID` of its own it has its own.
- **Node identity is functional**: `LABEL`, `COMMENT` and entry labels are read by people and by the editor, and follow
  the naming rules (23.8); they are not cosmetic.
- **The folder is the package** (chapter 15). A part, a tile part, a mod collection lives in whichever package folder
  is its home: the game's, or one of its own (a mod author's, a peer's personal cards). A package has no other record.

## 23.6 Versions and DLC

- **One end per version, for every package.** Every version a game ever had is a row of its own (for Steam, one per
  build date). A mod made for an old build is satisfied by running that build. Rows are built in the library rebuild;
  each one is run at that game's golden pass.
- **Ends are append-only.** An end once offered stays; a new build is a new end, never a rewrite of the old one.
- **DLC is part of the chain**, at the version it appeared, as a `NODE` gated by an option; later rows inherit both. A
  part that needs a DLC (or its absence) says so in a condition naming the DLC's node.
- **Version-independent mods** list every end they go into, or go into a part present on all of them.
- **Compatibility beyond what the author listed** is a claim part: `INTO [another end]` plus the mod's content (or a
  `NODE` of the mod).
- **Re-minting a settled node orphans what names it, by design**: a CID guarantees one state for everyone, and builders
  are deterministic, so a package is minted once, right. `catalog repoint` carries a re-mint across the library.

## 23.7 Closures follow hard links

Sharing (the share sheet's closure, chapter 15 §15.5), hydrating a received package, and keeping a node a version
switch would drop (15.6) all follow **hard links**: `NODE` (contains) and `INTO` (goes into). A mod package shared on
its own brings the packages of the ends it goes into; a mod installed for an end keeps that end. A condition's CIDs are
never followed.

## 23.8 Naming

The rules (USER; the Golden Pass's list) are not a style: they are how a person reads a package and a share sheet.

| What | Rule |
|---|---|
| Package folder | `[<uid>] <Title>` (no version tag: identity is the CIDs) |
| An entry | `<Title> v<version>`, + ` (<Edition>)` where it tells versions apart; never "Play"; real colons |
| A content file | `<Title> (<Edition> - <Store> <version>).zip`, ` - ` for `:` |
| A Steam build | `<Title> - <date> (Depot N, Manifest M).zip` / `.vgdelta` |
| A disc | its Redump name |
| A runner | `<runner>_<ver>_<host>_exec` |
| `VARIANT` | names the platform, never the runner |
| A node file | its `LABEL` (02 §2.5) |
| A mod | `<Mod> v<version>` (its upstream name) |
| A mod built per game version | + ` (for <Game> v<version>)`, or ` (for <Game> <date>)` for Steam builds — only when a folder holds builds of one mod for several game versions |
| A mod's content file | `<Mod> (Nexus <version>).zip` (ModDB, GitHub likewise) |
| A compatibility patch | its upstream name, else `<Mod A> + <Mod B> Patch v<version>` |
| A mod folder that is not a game | `[<game uid>] <Game> - <Author or Mod>` |

## 23.9 Tools

| Command | Does |
|---|---|
| `build link <pkg> <child> <parent> --into [--at N]` | child goes into parent; `--at` picks the `INTO` layer (one past the last: "and also into") |
| `build link … --not` | "never with parent": a condition on the part's `INTO` |
| `catalog refs <cid>` | every node naming it, and how (`NODE`, `INTO`, `WHEN`) |
| `catalog repoint OLD NEW [--map f.json] [--dry]` | swap a CID everywhere, re-minting what names it, instance graft lists following; under the library's write lock |
| `node check <node>` | its refs: contains, goes into (each `INTO` layer), the CIDs its conditions name |
| `grafts tick` / `launch plan --with` | the selection after a tick; a launch with parts added |
| `grafts conflicts` / `grafts resolve --address A --keep P` | where ticked parts write one unit differently; keep one (instance `RESOLUTIONS`) |
| `coding describe / info / mint / verify / cat` | the coding algebra (chapter 22): describe a format, mint a shipped file into content and a record (refused unless bit-exact), verify it back |

`tools/gen6/resolve.py` carries the reference semantics (`into_layers`, `when_cids`, `node_refs`, `hard_links`,
`remap_node`), held plan-for-plan to the engine by `fold_gate.py`.

## 23.10 Scenarios

Each judged against the model. **✅** covered · **📐** covered by an authoring rule · **🔧** the model covers it, the app
or tooling does not yet · **❗** a gap in the model.

### Versions and builds

| # | Scenario | How | |
|---|---|---|---|
| 1 | Linear patch history (WC3 1.21b → 1.29) | a storage chain; an end per version | ✅ |
| 2 | Steam betas / legacy branches (Skyrim 1.5.97 vs AE) | ends whose chains diverge from a common ancestor | ✅ |
| 3 | Downgrading because an update broke mods | pick the older end; its mods are offered there | ✅ |
| 4 | A hotfix that changes only the exe | a new end; mods are not offered until a claim part says they work | 📐 |
| 5 | A build whose depots are incomplete | not offered | ✅ |
| 6 | Windows / macOS / Linux builds | an end per platform | ✅ |
| 7 | Regional / censored editions | ends per edition; region-bound mods go into theirs | ✅ |
| 8 | Store editions with different exes | ends per store; SKSE's GOG build goes into the GOG end | ✅ |
| 9 | Re-releases (Skyrim LE / SE / AE; AoE2 / HD / DE) | separate packages and families; shared assets dedup through decoding (22) | ✅ |
| 10 | Hundreds of builds (Minecraft snapshots) | one chain, one end per build | ✅ |
| 11 | Updates shipped as patcher programs | the patched result is the chain node | ✅ |
| 12 | A save from a newer build loaded in an older one | owned state is per family; separate instances per build are manual | 🔧 |
| 13 | Online-only games whose servers are gone | out of scope (a server emulator would be a part with an entry) | — |

### DLC and expansions

| # | Scenario | How | |
|---|---|---|---|
| 14 | Expansions with their own exe and tile | an entry with a tile, `PARENTUID` = the base game | ✅ |
| 15 | Optional DLC (Creation Club, 'Mech Paks) | in the chain at its version, a `NODE` gated by an option | 📐 |
| 16 | DLC patched later | later chain nodes change it | ✅ |
| 17 | Bundles toggled together | one option gating several nodes | ✅ |
| 18 | Mutually exclusive DLC | an enum option | ✅ |
| 19 | DLC that changes the exe | the exe change gated with the DLC | ✅ |
| 20 | A mod that needs a DLC / needs it absent | a condition naming the DLC's node | ✅ |
| 21 | Dozens of micro-DLC | one option each, grouped by `SECTION` | ✅ |
| 22 | DLC the user does not own | the option's default follows ownership where known | 🔧 |

### Mods: shape and dependencies

| # | Scenario | How | |
|---|---|---|---|
| 23 | Loose-file replacer | a part with a `ZIP` | ✅ |
| 24 | A mod in its own archive (BSA, BA2, pak) | a part with the archive; its members as units need decoding (22) | ✅ / 🔧 units |
| 25 | A mod that needs another (SkyUI needs SKSE) | `INTO` the other part, or a condition naming it | ✅ |
| 26 | Compatibility patch for A + B | two `INTO` layers: offered once both are ticked | ✅ |
| 27 | FOMOD options, "if USSEP is installed use this patch" | the part's options; presence-dependent pieces as small parts or conditions | ✅ |
| 28 | Lite / full versions of one mod | exclusive parts (conditions) or one part with an enum option | ✅ |
| 29 | Mod updates (v1 → v2) | versions are sibling parts; their content may chain for storage | 📐 |
| 30 | A mod pinned to a game build (SKSE plugins) | its `INTO` names that build's end | ✅ |
| 31 | A mod that works on several builds | its `INTO` lists those ends | ✅ |
| 32 | Version-independent mods (texture packs) | `INTO` every end, or a part on all of them | 📐 |
| 33 | Address Library + version-independent plugins | one Address Library part per build; plugins go into those parts | ✅ |
| 34 | Script extender as the launcher | the part's entry is one more way to run the end | ✅ |
| 35 | Mod loaders per game version (Forge, Fabric) | a loader part per end; mods go into loader parts | ✅ |
| 36 | Dependency ranges ("Forge ≥ 47.1") | the `INTO` lists the acceptable parts; ranges are never computed | ✅ |
| 37 | Two mods needing different versions of one library mod | exclusive; the window shows which cannot apply | ✅ |
| 38 | Mutually incompatible mods | a condition | ✅ |
| 39 | "Load after X if X is present" | not declared (it needs knowledge of every mod): structure (`INTO` X) or a merge | ✅ by design |
| 40 | Total conversions (Enderal, Nehrim) | an end `INTO` the base end, with its own tile, often its own package | ✅ |
| 41 | Source ports and their mods | ends containing the original's content; mods go into those ends | ✅ |
| 42 | A base-game mod on a derived end (an SP mod on SA-MP) | not offered (another end); a claim part says otherwise | ✅ |
| 43 | Mod collections (Nexus Collections, Wabbajack) | an end `INTO` the base, containing the mods in its order, with its own tile | ✅ |
| 44 | Cross-game mods (Tale of Two Wastelands) | `INTO [FNV end]` + `NODE`s of Fallout 3's data with `TAKE` | ✅ |
| 45 | ROM hacks and translations per ROM revision | ends per revision; the hack goes into its revision | ✅ |
| 46 | Emulator texture packs, cheats | content at the emulator's anchors; codes as `EDIT`s / options | ✅ |
| 47 | Same version re-uploaded with different bytes | a different CID, a different node | ✅ |

### Mods: edits and conflicts

| # | Scenario | How | |
|---|---|---|---|
| 48 | INI tweaks from several mods | keyed addresses (`file#key`): different keys never meet; the same key differently is a conflict | ✅ |
| 49 | Exe patches (4GB, no-CD, widescreen hex) | binary `EDIT` with `EXPECT`, on the build's end | ✅ |
| 50 | Two exe patches, disjoint bytes (Wipeout 200 fps + widescreen) | disjoint byte ranges merge | 🔧 (byte-range units) |
| 51 | A mod shipped as a delta of a game file | version-bound by nature: it goes into the ends whose file it was made from | ✅ |
| 52 | Three delta mods from one root | merge in base coordinates (23.4) | 🔧 |
| 53 | Plugin record conflicts | records as units need the plugin description (22) | 🔧 |
| 54 | Two mods replace one file, one meant to win | a conflict: resolved by the user or a patch part going into both | ✅ |
| 55 | A mod deleting game files (skip intros) | `EDIT Delete`, refused inside user state | ✅ |
| 56 | ENB and ReShade both as `d3d11.dll` | a conflict; chain-loading renames one with a `TAKE` pair | ✅ |
| 57 | Companion processes (trainers, a server beside the client) | one process per launch: only through a wrapper entry | — (out of scope) |

### Generated content and tools

| # | Scenario | How | |
|---|---|---|---|
| 58 | Nemesis / FNIS / BodySlide / bashed patches | authored once per input set: a part `INTO` every input | ✅ model / 🔧 tooling |
| 59 | LOOT order | `AppendLine` units in a deterministic order; record conflicts once records are units | ✅ / 🔧 |
| 60 | Tools that write into the game folder | the runtime is ephemeral: what a tool writes is captured into a new part | 🔧 |
| 61 | Mod tools as launch entries (xEdit) | a part's entry without a tile | ✅ |

### State the game writes

| # | Scenario | How | |
|---|---|---|---|
| 62 | Saves shared by base game and expansions | owned state belongs to the family | ✅ |
| 63 | Saves that depend on the mod set | nothing links a save to the end + parts it was made with | 🔧 |
| 64 | MCM settings, SKSE co-saves | inside kept folders | ✅ |
| 65 | Save locations that differ per store edition | `KEEP` folds per end | ✅ |
| 66 | Per-machine settings | options with take-backs; `CLOUD: false` | ✅ |
| 67 | Game self-updaters | the runtime is rebuilt every launch | ✅ |
| 68 | First-run generated files | `IF_EXISTS` edits | ✅ |

### Identity, packaging, sharing

| # | Scenario | How | |
|---|---|---|---|
| 69 | The pristine re-minted after publishing | must not happen once settled | 📐 |
| 70 | A bug fixed in build K's storage after mods exist | everything above K changes CID; `catalog repoint` carries it; by design | ✅ |
| 71 | Repacking a game (decoded deltas) | as #70; repacks happen before a package is settled | ✅ |
| 72 | Two people packaging the same game | deterministic builders give both the same CIDs | 📐 |
| 73 | A mod author forbids redistribution | profiles and parts carry CIDs only | ✅ |
| 74 | A peer has another version of a mod | package version switching (15.6) | ✅ |
| 75 | Mods as `.7z` / `.rar` / DEFLATE zips | decoded through descriptions (22) | ✅ / 🔧 |
| 76 | "Mod X also works on build Y", said by someone else | a claim part: `INTO [end Y]` + X | ✅ |
| 77 | Malicious mods | the sandbox (14.5) | ✅ |
| 78 | A cover or a recommendation changes | that launchable re-mints (tiles live in the nodes) | ✅ by design |

### Platforms and runners

| # | Scenario | How | |
|---|---|---|---|
| 79 | Native Linux build vs Proton | ends per platform | ✅ |
| 80 | A mod that needs a particular runner | `RECOMMENDEDRUNNER` on its entry: a soft pin (never a runner CID) | ✅ |
| 81 | Mods needing runtimes (.NET, VC++, Java) | library nodes in the part | ✅ |
| 82 | Steam Deck vs desktop | per-machine variables, `CLOUD: false` | ✅ |

Open: units inside files and archives (23.4), saves linked to their composition (#63), capturing tool output (#60), DLC
defaults from ownership (#22).
