# 17 · Worked examples

Complete, copy-pasteable node sets that exercise the whole spec. Each shows the bundle layout and the resolved behavior.
Comments (`//`) are for the reader; strip them for real JSON.

Throughout, the runner library (Proton, the native terminal, emulators) is assumed to live in a separate bundle — see
[§17.7](#177-the-runner-library).

Read every chain **bottom-up**: a node's `PARENTS` are what must be applied before it, so the launchable is always the
last node of its chain and the highest-priority layer of its runtime.

---

## 17.1 A native Linux game

The simplest case: content whose platform *is* the machine platform. The chain is just the native terminal.

```
[1234] My Linux Game/
├── mylinuxgame.json           // holds the whole chain as a JSON array
└── mylinuxgame.zip            // STORE zip of the game tree, exe at the root
```

```jsonc
[
  // the content
  { "NODE_ID": "mylinuxgame_content", "TYPE": "Content", "FORM": "zip", "PATH": "mylinuxgame.zip" },

  // the tile — carries no content, and is a PARENT of the launchable
  { "NODE_ID": "mylinuxgame", "TYPE": "DeclareLibraryItem", "UID": "1234", "TITLE": "My Linux Game" },

  // the launchable — no GUEST, so it is the terminal link: the thing you run
  { "NODE_ID": "mylinuxgame_game", "TYPE": "DeclareExec",
    "PARENTS": ["mylinuxgame_content", "mylinuxgame"],
    "HOST": "linux64", "PATH": "mygame", "ARGS": ["--fullscreen"] }
]
```

**Resolves to:** chain `[native-passthrough]`; the terminal runs `%Content%` = `<runtime>/mygame --fullscreen`. No
prefix, content at the root, pristine runtime plus whatever the runner's keep-set persists.

---

## 17.2 A Windows game under Proton (with a knob and a registry default)

```
[7804] Age of Mythology/
├── aom.json
├── aom.zip
└── AoM_Cover.jpg
```

```jsonc
[
  { "NODE_ID": "aom_content", "TYPE": "Content", "FORM": "zip",
    "PATH": "aom.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },

  // a user knob (bool), exposed as %WIDESCREEN% (raw "1"/"0") and rendered as a dword at the registry use-site
  { "NODE_ID": "aom_var_widescreen", "TYPE": "CustomVar", "PARENTS": ["aom_content"],
    "KEY": "WIDESCREEN", "DEFAULT": "1",
    "UI": { "LABEL": "Widescreen UI", "CONTROL": "bool" } },

  // a base registry default (overridable by the user once they change it in-game)
  { "NODE_ID": "aom_registry", "TYPE": "RegEdit", "PARENTS": ["aom_var_widescreen"],
    "EDITS": [ { "ARCHITECTURE": ["32"],
                 "HKCU": { "Software": { "Microsoft": { "Microsoft Games": { "Age of Mythology": {
                     "Widescreen": "%WIDESCREEN:dword%" } } } } } } ] },   // → dword:00000001

  { "NODE_ID": "aom", "TYPE": "DeclareLibraryItem", "UID": "7804", "TITLE": "Age of Mythology",
    "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    "META": { "UMUID": "266840" } },

  { "NODE_ID": "aom_game", "TYPE": "DeclareExec", "PARENTS": ["aom_registry", "aom"],
    "HOST": "win32", "PATH": "aom.exe",
    "ARGS": ["xres=%ScreenWidth%", "yres=%ScreenHeight%"] }
]
```

**Resolves to:** chain `[ge-proton10-30, native-passthrough]`; Proton generates a prefix, content mounts at
`pfx/drive_c/7804`, the launch command (terminal forwards Proton) is
`proton waitforexitandrun C:\7804\aom.exe xres=1920 yres=1080`. The `WIDESCREEN` knob resolves (default on →
`dword:00000001`) and is baked into the default-data hive; if the user later changes it in-game, their writable-layer
value shadows the default next launch.

Note `ARGS` is **two array elements**, not one string: each is one argv entry and nothing re-splits them.

---

## 17.3 A console ROM via a native emulator

A console (here SNES) that *does* have a native-Linux emulator — the chain is one bridge hop. (`StarVoyager` is a
hypothetical homebrew ROM.)

```
[8500] Star Voyager/
├── star_voyager.json
└── StarVoyager.sfc
```

```jsonc
[
  { "NODE_ID": "star_voyager_rom", "TYPE": "Content", "FORM": "file", "PATH": "StarVoyager.sfc" },

  { "NODE_ID": "star_voyager", "TYPE": "DeclareLibraryItem", "UID": "8500", "TITLE": "Star Voyager" },

  { "NODE_ID": "star_voyager_game", "TYPE": "DeclareExec",
    "PARENTS": ["star_voyager_rom", "star_voyager"],
    "HOST": "snes", "PATH": "StarVoyager.sfc" }
]
```

**Resolves to:** with a native-Linux `snes9x` runner (`GUEST:["snes"], HOST:"linux64"`) installed, the shortest chain is
`[snes9x, native-passthrough]` and the terminal runs `snes9x -fullscreen <runtime>/StarVoyager.sfc`. One bridge hop, no
prefix. Because a native runner exists, there's no reason to chain onward — contrast §17.4, where one doesn't.

---

## 17.4 A cross-platform daisy chain (a console with only a Windows emulator)

The cross-namespace example (chapter 11). The *Vortex* is a hypothetical console whose **only** emulator, *VortexEmu*, is
a Windows program — so the route to `linux64` runs through win32, and the runtime derives the chain automatically. The
emulator is shipped as a runner; here it's embedded in the game's bundle to also show the embedded-runner shape.

```
[9001] Vortex Quest/
├── vortex_quest.json
├── vortexemu_win.json
├── VortexQuest.vtx               // the game ROM
└── vortexemu.exe                 // the win32 emulator build
```

```jsonc
// vortex_quest.json — Vortex content; declares no runner, just the platform it needs
[
  { "NODE_ID": "vortex_quest_rom", "TYPE": "Content", "FORM": "file", "PATH": "VortexQuest.vtx" },
  { "NODE_ID": "vortex_quest", "TYPE": "DeclareLibraryItem", "UID": "9001", "TITLE": "Vortex Quest" },
  { "NODE_ID": "vortex_quest_game", "TYPE": "DeclareExec",
    "PARENTS": ["vortex_quest_rom", "vortex_quest"],
    "HOST": "vortex", "PATH": "VortexQuest.vtx" }
]

// vortexemu_win.json — an embedded runner: VortexEmu is win32-only
[
  { "NODE_ID": "vortexemu_win_build", "TYPE": "Content", "FORM": "file", "PATH": "vortexemu.exe" },
  { "NODE_ID": "vortexemu_win", "TYPE": "DeclareExec", "PARENTS": ["vortexemu_win_build"],
    "HOST": "win32", "GUEST": ["vortex"],
    "PATH": "vortexemu.exe", "ARGS": ["%Content%"] }
]
```

No pin is needed — `vortex → win32 → linux64` is the only route. The runtime resolves
`[vortexemu_win, ge-proton10-30, native-passthrough]`, mounts `vortexemu.exe` at
`pfx/drive_c/9001/__runner_vortexemu_win__/vortexemu.exe` and the ROM at `pfx/drive_c/9001/VortexQuest.vtx`, derives
Proton's guest template from its `CONTENT_ROOT`, and execs:

```
proton waitforexitandrun "C:\9001\__runner_vortexemu_win__\vortexemu.exe" "C:\9001\VortexQuest.vtx"
```

`vortex → win32 → linux64`, one process. Note the emulator's `PATH` is a *build-relative* `vortexemu.exe` (it runs
inside Wine, not from the host `PATH`), and the runner is "available" because it ships a build (chapter 10 §10.6).

---

## 17.5 A multi-variant game (two editions, one tile)

Two launchables grouped under one library-tile node via a `PARENTS` edge — no `GAME` string. The tile carries the
metadata and no content; each variant `PARENTS` it and hangs its own content chain off itself.

```jsonc
// aoe2.json — the GAME TILE: presentable, carries no content, not launchable itself
{ "NODE_ID": "aoe2", "TYPE": "DeclareLibraryItem", "UID": "1001", "TITLE": "Age of Empires II" }

// aoe2_fe.json — the default variant. The PATCH is a CHILD of the base, which is what orders them.
[
  { "NODE_ID": "aoe2_fe_patch", "TYPE": "Content", "FORM": "zip", "PATH": "fe_patch.zip",
    "PARENTS": ["aoe2_base"] },
  { "NODE_ID": "aoe2_fe", "TYPE": "DeclareExec", "PARENTS": ["aoe2_fe_patch", "aoe2"],
    "HOST": "win32", "PATH": "age2_x1/age2_x1.5.exe",
    "LABEL": "Forgotten Empires", "RECOMMENDED": true }
]

// aoe2_gog.json — another edition of the SAME tile
{ "NODE_ID": "aoe2_gog", "TYPE": "DeclareExec", "PARENTS": ["aoe2_gog_base", "aoe2"],
  "HOST": "win32", "PATH": "empires2.exe", "LABEL": "GOG edition" }
```

**Behavior:** the library shows **one** "Age of Empires II" tile (both variants reach `aoe2` through `PARENTS`); opening
it offers two variants, "Forgotten Empires" (pre-selected, `RECOMMENDED`) and "GOG edition". Each variant inherits the
tile's metadata (field-level composition down the closure) and resolves its own content closure and chain.

Note how the patch overrides the base: **`aoe2_fe_patch` lists `aoe2_base` as a parent**, so it is applied after it. It
is *not* enough to list them in that order on `aoe2_fe` — two parents of one node are unordered (invariant **I9**).

---

## 17.6 A game with optional expansions and automatic load order

`TOGGLE: "off"` content + `AppendLine` = a toggleable expansion that registers itself in the game's load-order file, in
closure order, with no mod-manager construct.

```jsonc
[
  { "NODE_ID": "morrowind_base", "TYPE": "Content", "FORM": "zip",
    "PATH": "morrowind.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },

  // register the base master in the load order (idempotent, ordered)
  { "NODE_ID": "morrowind_base_cfg", "TYPE": "FileEdit", "PARENTS": ["morrowind_base"],
    "FILE": "Data Files/openmw.cfg", "OVERRIDE": true,
    "EDITS": [ { "MODE": "AppendLine", "VALUE": "content=Morrowind.esm" } ] },

  // an OPTIONAL expansion (off by default). Its cfg line is a CHILD of the base's, so it comes after.
  { "NODE_ID": "morrowind_tribunal", "TYPE": "Content", "TOGGLE": "off", "FORM": "zip",
    "PATH": "tribunal.zip", "PARENTS": ["morrowind_base_cfg"],
    "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
  { "NODE_ID": "morrowind_tribunal_cfg", "TYPE": "FileEdit", "PARENTS": ["morrowind_tribunal"],
    "FILE": "Data Files/openmw.cfg", "OVERRIDE": true,
    "EDITS": [ { "MODE": "AppendLine", "VALUE": "content=Tribunal.esm" } ] },

  // another OPTIONAL expansion, after Tribunal
  { "NODE_ID": "morrowind_bloodmoon", "TYPE": "Content", "TOGGLE": "off", "FORM": "zip",
    "PATH": "bloodmoon.zip", "PARENTS": ["morrowind_tribunal_cfg"],
    "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
  { "NODE_ID": "morrowind_bloodmoon_cfg", "TYPE": "FileEdit", "PARENTS": ["morrowind_bloodmoon"],
    "FILE": "Data Files/openmw.cfg", "OVERRIDE": true,
    "EDITS": [ { "MODE": "AppendLine", "VALUE": "content=Bloodmoon.esm" } ] },

  { "NODE_ID": "morrowind", "TYPE": "DeclareLibraryItem",
    "UID": "2050", "TITLE": "The Elder Scrolls III: Morrowind" },

  { "NODE_ID": "morrowind_game", "TYPE": "DeclareExec",
    "PARENTS": ["morrowind_bloodmoon_cfg", "morrowind"],
    "HOST": "win32", "PATH": "Morrowind.exe", "LABEL": "GOTY" }
]
```

**Behavior:** the prelaunch UI shows Tribunal and Bloodmoon as toggles (off by default). Enable both → they enter the
closure in chain order → their `AppendLine`s run in order → `openmw.cfg` ends with exactly:

```
content=Morrowind.esm
content=Tribunal.esm
content=Bloodmoon.esm
```

Disable Tribunal → the node is skipped, and so is everything reachable *only* through it (the hierarchy gate, ch. 12
§12.5) — but `morrowind_bloodmoon` is still reached, because its own chain still leads back to a kept node. Its line
never appears, its files never mount. The load order is a *consequence* of the graph, not a feature.
(Mutually-exclusive expansions would add `EXCLUDE` to make a pick-one set.)

---

## 17.7 The runner library

The shared runners every game routes through. A separate bundle/repo (`VidyaGodRunners`), one bundle per runner. A
runner is just a `DeclareExec` with a non-empty `GUEST`.

```jsonc
// native-passthrough — the universal terminal
{ "NODE_ID": "native-passthrough", "TYPE": "DeclareExec",
  "HOST": "linux64", "GUEST": ["linux64"], "PATH": "%Content%", "ARGS": [] }

// ge-proton10-30 — Wine-family, generates a prefix; build on a content parent
[
  { "NODE_ID": "geproton_build", "TYPE": "Content", "FORM": "zip",
    "PATH": "GE-Proton10-30.zip", "TARGET": "", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },

  { "NODE_ID": "geproton_var_log", "TYPE": "CustomVar", "PARENTS": ["geproton_build"],
    "KEY": "PROTON_LOG", "DEFAULT": "0",
    "UI": { "LABEL": "Proton logging", "CONTROL": "enum",
            "CHOICES": [ { "LABEL": "Off", "VALUE": "0" }, { "LABEL": "On", "VALUE": "1" } ] } },

  { "NODE_ID": "proton_keepset", "TYPE": "Persist", "PARENTS": ["geproton_var_log"],
    "KEEP": ["pfx/drive_c/users", "HKCU"] },

  { "NODE_ID": "ge-proton10-30", "TYPE": "DeclareExec", "PARENTS": ["proton_keepset"],
    "HOST": "linux64", "GUEST": ["win32", "win64"],
    "PATH": "%RunnerMount%/proton",
    "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
    "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%",
             "PROTON_LOG": "%PROTON_LOG%" },
    "ENV_REMOVE": ["LD_LIBRARY_PATH"],
    "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true }
]

// snes9x — a native-Linux emulator (one bridge hop for SNES content)
{ "NODE_ID": "snes9x", "TYPE": "DeclareExec",
  "HOST": "linux64", "GUEST": ["snes"], "PATH": "snes9x", "ARGS": ["-fullscreen", "%Content%"] }
```

A *cloned terminal* that wraps every launch in `gamescope` (chapter 11 §11.3) is just another native runner the user can
select as the terminal step:

```jsonc
{ "NODE_ID": "native-gamescope", "TYPE": "DeclareExec",
  "HOST": "linux64", "GUEST": ["linux64"], "PATH": "gamescope", "ARGS": ["-f", "--"] }
```

Selected as the terminal, it composes `gamescope -f -- <whatever the chain produced>` — wrapping the game, or Proton, or
Proton-wrapping-an-emulator, uniformly.

Next: [Reference implementation map](18-reference-implementation.md).
