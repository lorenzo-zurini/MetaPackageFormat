# 17 · Worked examples

Complete, copy-pasteable node sets that exercise the whole spec. Each shows the bundle layout and the resolved behavior.
Comments (`//`) are for the reader; strip them for real JSON.

Throughout, the runner library (Proton, the native terminal, emulators) is assumed to live in a separate bundle — see
[§17.7](#177-the-runner-library).

---

## 17.1 A native Linux game

The simplest case: content whose platform *is* the machine platform. The chain is just the native terminal.

```
[1234] My Linux Game/
├── mylinuxgame.json
├── mylinuxgame_base.json
└── mylinuxgame.zip            // STORE zip of the game tree, exe at the root
```

```jsonc
// mylinuxgame.json — single-variant game: tile + exec on one node
{ "NODE_ID": "mylinuxgame",
  "PARENTS": ["mylinuxgame_base"],
  "LAYERS": [
    { "TYPE": "DeclareLibraryItem", "UID": "1234", "TITLE": "My Linux Game" },
    { "TYPE": "DeclareExec", "PLATFORM": "linux64", "CONTENTPATH": "mygame", "EXEARGS": "--fullscreen" } ] }

// mylinuxgame_base.json
{ "NODE_ID": "mylinuxgame_base",
  "LAYERS": [ { "TYPE": "VFSZipLayer", "PATH": "mylinuxgame.zip" } ] }
```

**Resolves to:** chain `[native-passthrough]`; the terminal runs `%Content%` = `<runtime>/mygame --fullscreen`. No
prefix, content at the root, whole-runtime persistence (no `Persist*` declared).

---

## 17.2 A Windows game under Proton (with a knob and a registry default)

```
[7804] Age of Mythology/
├── aom.json
├── aom_base.json
├── aom.zip
└── AoM_Cover.jpg
```

```jsonc
// aom.json
{ "NODE_ID": "aom",
  "PARENTS": ["aom_base"],
  "LAYERS": [
    { "TYPE": "DeclareLibraryItem", "UID": "7804", "TITLE": "Age of Mythology",
      "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
      "UMUID": "266840" },
    { "TYPE": "DeclareExec", "PLATFORM": "win32",
      "CONTENTPATH": "aom.exe", "EXEARGS": "xres=%ScreenWidth% yres=%ScreenHeight%" } ] }

// aom_base.json
{ "NODE_ID": "aom_base",
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "aom.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },

    // a user knob (bool), exposed as %WIDESCREEN% (raw "1"/"0") and rendered as a dword at the registry use-site
    { "TYPE": "CustomVar", "KEY": "WIDESCREEN", "DEFAULT": "1",
      "UI": { "LABEL": "Widescreen UI", "CONTROL": "bool" } },

    // a base registry default (overridable by the user once they change it in-game)
    { "TYPE": "RegEdit", "ARCHITECTURE": "32", "OVERRIDE": false,
      "REGPATH": "HKCU\\Software\\Microsoft\\Microsoft Games\\Age of Mythology",
      "KEYVALUES": { "Widescreen": "%WIDESCREEN:dword%" } }   // → dword:00000001
  ] }
```

**Resolves to:** chain `[ge-proton10-30, native-passthrough]`; Proton generates a prefix, content mounts at
`pfx/drive_c/7804`, the launch command (terminal forwards Proton) is
`proton waitforexitandrun C:\7804\aom.exe xres=1920 yres=1080`. The `WIDESCREEN` knob resolves (default on → `dword:
00000001`) and is baked into the default-data hive; if the user later changes it in-game, their writable-layer value
shadows the default next launch.

---

## 17.3 A console ROM via a native emulator

A console (here SNES) that *does* have a native-Linux emulator — the chain is one bridge hop. (`StarVoyager` is a
hypothetical homebrew ROM.)

```
[8500] Star Voyager/
├── star_voyager.json
├── star_voyager_base.json
└── StarVoyager.sfc
```

```jsonc
// star_voyager.json
{ "NODE_ID": "star_voyager",
  "PARENTS": ["star_voyager_base"],
  "LAYERS": [
    { "TYPE": "DeclareLibraryItem", "UID": "8500", "TITLE": "Star Voyager" },
    { "TYPE": "DeclareExec", "PLATFORM": "snes", "CONTENTPATH": "StarVoyager.sfc" } ] }

// star_voyager_base.json
{ "NODE_ID": "star_voyager_base",
  "LAYERS": [ { "TYPE": "VFSFileLayer", "PATH": "StarVoyager.sfc" } ] }
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
├── vortex_quest_base.json
├── vortexemu_win.json
├── vortexemu_win_build.json
├── VortexQuest.vtx               // the game ROM
└── vortexemu.exe                 // the win32 emulator build
```

```jsonc
// vortex_quest.json — Vortex content; declares no runner, just its platform
{ "NODE_ID": "vortex_quest",
  "PARENTS": ["vortex_quest_base"],
  "LAYERS": [
    { "TYPE": "DeclareLibraryItem", "UID": "9001", "TITLE": "Vortex Quest" },
    { "TYPE": "DeclareExec", "PLATFORM": "vortex", "CONTENTPATH": "VortexQuest.vtx" } ] }

// vortex_quest_base.json
{ "NODE_ID": "vortex_quest_base",
  "LAYERS": [ { "TYPE": "VFSFileLayer", "PATH": "VortexQuest.vtx" } ] }

// vortexemu_win.json — an embedded runner: VortexEmu is win32-only
{ "NODE_ID": "vortexemu_win",
  "PARENTS": ["vortexemu_win_build"],
  "LAYERS": [ { "TYPE": "DeclareRunner", "HOST": "win32", "GUEST": ["vortex"],
                "EXECUTABLE": "vortexemu.exe", "ARGS": ["%Content%"], "ENV": {}, "REMOVE_ENV": [] } ] }

// vortexemu_win_build.json
{ "NODE_ID": "vortexemu_win_build",
  "LAYERS": [ { "TYPE": "VFSFileLayer", "PATH": "vortexemu.exe" } ] }
```

No pin is needed — `vortex → win32 → linux64` is the only route. The runtime resolves
`[vortexemu_win, ge-proton10-30, native-passthrough]`, mounts `vortexemu.exe` at
`pfx/drive_c/9001/__runner_vortexemu_win__/vortexemu.exe` and the ROM at `pfx/drive_c/9001/VortexQuest.vtx`, derives
Proton's guest template from its `CONTENT_ROOT`, and execs:

```
proton waitforexitandrun "C:\9001\__runner_vortexemu_win__\vortexemu.exe" "C:\9001\VortexQuest.vtx"
```

`vortex → win32 → linux64`, one process. Note the emulator's `EXECUTABLE` is a *build-relative* `vortexemu.exe` (it runs
inside Wine, not from the host `PATH`), and the runner is "available" because it ships a build (chapter 10 §10.6).

---

## 17.5 A multi-variant game (two editions, one tile)

Two launchable (`DeclareExec`) nodes grouped under one library-tile (`DeclareLibraryItem`) node via a `PARENTS` edge —
no `GAME` string. The tile node carries the metadata and no content; each variant `PARENTS` it and hangs its own content
chain off itself. (A *single*-variant game collapses to one node with both layers — see §17.1/§17.2.)

```jsonc
// aoe2.json — the GAME TILE: presentable, carries no content, not launchable itself
{ "NODE_ID": "aoe2",
  "LAYERS": [ { "TYPE": "DeclareLibraryItem", "UID": "1001", "TITLE": "Age of Empires II" } ] }

// aoe2_fe.json — the default variant; PARENTS the tile to group under it
{ "NODE_ID": "aoe2_fe",
  "PARENTS": ["aoe2", "aoe2_base", "aoe2_fe_patch"],   // patch listed later ⇒ higher priority (overrides base)
  "LAYERS": [ { "TYPE": "DeclareExec", "PLATFORM": "win32", "CONTENTPATH": "age2_x1/age2_x1.5.exe",
                "LABEL": "Forgotten Empires", "RECOMMENDED": true } ] }

// aoe2_gog.json — another edition of the SAME tile
{ "NODE_ID": "aoe2_gog",
  "PARENTS": ["aoe2", "aoe2_gog_base"],
  "LAYERS": [ { "TYPE": "DeclareExec", "PLATFORM": "win32", "CONTENTPATH": "empires2.exe",
                "LABEL": "GOG edition" } ] }
```

**Behavior:** the library shows **one** "Age of Empires II" tile (the two variants share `aoe2` as a `PARENTS` ancestor);
opening it offers two variants, "Forgotten Empires" (pre-selected, `RECOMMENDED`) and "GOG edition". Each variant inherits
the tile's metadata (field-level composition down the closure) and resolves its own content closure and chain.

---

## 17.6 A game with an optional mod and automatic load order

Optional content + `AppendLine` = a toggleable mod that registers itself in the game's load-order file, in closure order,
with no mod-manager construct.

```jsonc
// morrowind.json — single-variant game: tile + exec on one node
{ "NODE_ID": "morrowind",
  "PARENTS": ["morrowind_base", "morrowind_tribunal", "morrowind_bloodmoon"],
  "LAYERS": [
    { "TYPE": "DeclareLibraryItem", "UID": "2050", "TITLE": "The Elder Scrolls III: Morrowind" },
    { "TYPE": "DeclareExec", "PLATFORM": "win32", "CONTENTPATH": "Morrowind.exe", "LABEL": "GOTY" } ] }

// morrowind_base.json
{ "NODE_ID": "morrowind_base",
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "morrowind.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    // register the base master in the load order (idempotent, ordered)
    { "TYPE": "FileEdit", "MODE": "AppendLine", "FILE": "Data Files/openmw.cfg", "VALUE": "content=Morrowind.esm" }
  ] }

// morrowind_tribunal.json — an OPTIONAL expansion (off by default)
{ "NODE_ID": "morrowind_tribunal", "OPTIONAL": true, "DEFAULT": false,
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "tribunal.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    { "TYPE": "FileEdit", "MODE": "AppendLine", "FILE": "Data Files/openmw.cfg", "VALUE": "content=Tribunal.esm" }
  ] }

// morrowind_bloodmoon.json — another OPTIONAL expansion
{ "NODE_ID": "morrowind_bloodmoon", "OPTIONAL": true, "DEFAULT": false,
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "bloodmoon.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    { "TYPE": "FileEdit", "MODE": "AppendLine", "FILE": "Data Files/openmw.cfg", "VALUE": "content=Bloodmoon.esm" }
  ] }
```

**Behavior:** the prelaunch UI shows Tribunal and Bloodmoon as toggles (off by default). Enable both → they enter the
closure after the base → their `AppendLine`s run in order → `openmw.cfg` ends with exactly:

```
content=Morrowind.esm
content=Tribunal.esm
content=Bloodmoon.esm
```

Disable Tribunal → its node never enters the closure → its line never appears, its files never mount. The load order is a
*consequence* of the graph, not a feature. (Mutually-exclusive mods would add `EXCLUDE` to make a pick-one set.)

---

## 17.7 The runner library

The shared runners every game routes through. A separate bundle/repo (`VidyaGodRunners`), one bundle per runner.

```jsonc
// native-passthrough — the universal terminal
{ "NODE_ID": "native-passthrough",
  "LAYERS": [ { "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["linux64"],
                "EXECUTABLE": "%Content%", "ARGS": [], "ENV": {}, "REMOVE_ENV": [] } ] }

// ge-proton10-30 — Wine-family, generates a prefix; build on a content parent
{ "NODE_ID": "ge-proton10-30",
  "PARENTS": ["geproton_build"],
  "LAYERS": [ { "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["win32", "win64"],
                "EXECUTABLE": "%RunnerMount%/proton",
                "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
                "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%" },
                "REMOVE_ENV": ["LD_LIBRARY_PATH"],
                "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true } ] }

// geproton_build — the Proton tree (+ a couple of runner knobs)
{ "NODE_ID": "geproton_build",
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "GE-Proton10-30.zip", "TARGET": "",
      "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    { "TYPE": "CustomVar", "KEY": "PROTON_LOG", "DEFAULT": "0",
      "UI": { "LABEL": "Proton logging", "CONTROL": "enum",
              "CHOICES": [ { "LABEL": "Off", "VALUE": "0" }, { "LABEL": "On", "VALUE": "1" } ] } }
  ] }

// snes9x — a native-Linux emulator (one bridge hop for SNES content)
{ "NODE_ID": "snes9x",
  "LAYERS": [ { "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["snes"],
                "EXECUTABLE": "snes9x", "ARGS": ["-fullscreen", "%Content%"], "ENV": {}, "REMOVE_ENV": [] } ] }
```

A *cloned terminal* that wraps every launch in `gamescope` (chapter 11 §11.3) is just another native runner the user can
select as the terminal step:

```jsonc
{ "NODE_ID": "native-gamescope",
  "LAYERS": [ { "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["linux64"],
                "EXECUTABLE": "gamescope", "ARGS": ["-f", "--"], "ENV": {}, "REMOVE_ENV": [] } ] }
```

Selected as the terminal, it composes `gamescope -f -- <whatever the chain produced>` — wrapping the game, or Proton, or
Proton-wrapping-an-emulator, uniformly.

Next: [Reference implementation map](18-reference-implementation.md).
