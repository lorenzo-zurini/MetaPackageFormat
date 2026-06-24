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
// mylinuxgame.json
{ "NODE_ID": "mylinuxgame", "ROLE": "launchable",
  "UID": "1234", "GAME": "mylinuxgame", "LABEL": "Linux", "RECOMMENDED": true,
  "META": { "TITLE": "My Linux Game" },
  "PLATFORM": { "HOST": "linux64" },
  "EXEC": { "CONTENTPATH": "mygame", "EXEARGS": "--fullscreen" },
  "PARENTS": ["mylinuxgame_base"] }

// mylinuxgame_base.json
{ "NODE_ID": "mylinuxgame_base", "ROLE": "content",
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
{ "NODE_ID": "aom", "ROLE": "launchable",
  "UID": "7804", "GAME": "aom", "LABEL": "Original Release", "RECOMMENDED": true,
  "META": { "TITLE": "Age of Mythology",
            "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
            "UMUID": "266840" },
  "PLATFORM": { "HOST": "win32" },
  "EXEC": { "CONTENTPATH": "aom.exe", "EXEARGS": "xres=%ScreenWidth% yres=%ScreenHeight%" },
  "PARENTS": ["aom_base"] }

// aom_base.json
{ "NODE_ID": "aom_base", "ROLE": "content",
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
{ "NODE_ID": "star_voyager", "ROLE": "launchable",
  "UID": "8500", "GAME": "star_voyager", "LABEL": "Original", "RECOMMENDED": true,
  "META": { "TITLE": "Star Voyager" },
  "PLATFORM": { "HOST": "snes" },
  "EXEC": { "CONTENTPATH": "StarVoyager.sfc" },
  "PARENTS": ["star_voyager_base"] }

// star_voyager_base.json
{ "NODE_ID": "star_voyager_base", "ROLE": "content",
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
{ "NODE_ID": "vortex_quest", "ROLE": "launchable",
  "UID": "9001", "GAME": "vortex_quest", "LABEL": "Original", "RECOMMENDED": true,
  "META": { "TITLE": "Vortex Quest" },
  "PLATFORM": { "HOST": "vortex" },
  "EXEC": { "CONTENTPATH": "VortexQuest.vtx" },
  "PARENTS": ["vortex_quest_base"] }

// vortex_quest_base.json
{ "NODE_ID": "vortex_quest_base", "ROLE": "content",
  "LAYERS": [ { "TYPE": "VFSFileLayer", "PATH": "VortexQuest.vtx" } ] }

// vortexemu_win.json — an embedded runner: VortexEmu is win32-only
{ "NODE_ID": "vortexemu_win", "ROLE": "runner",
  "PLATFORM": { "HOST": "win32", "GUEST": ["vortex"] },
  "EXEC": { "EXECUTABLE": "vortexemu.exe", "ARGS": ["%Content%"], "ENV": {}, "REMOVE_ENV": [] },
  "PARENTS": ["vortexemu_win_build"] }

// vortexemu_win_build.json
{ "NODE_ID": "vortexemu_win_build", "ROLE": "content",
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

Two launchables sharing a `GAME` group under one library tile.

```jsonc
// aoe2.json — the default variant
{ "NODE_ID": "aoe2", "ROLE": "launchable",
  "UID": "1001", "GAME": "aoe2", "LABEL": "Forgotten Empires", "RECOMMENDED": true,
  "META": { "TITLE": "Age of Empires II" },
  "PLATFORM": { "HOST": "win32" },
  "EXEC": { "CONTENTPATH": "age2_x1/age2_x1.5.exe" },
  "PARENTS": ["aoe2_base", "aoe2_fe_patch"] }      // patch listed later ⇒ higher priority (overrides base)

// aoe2_gog.json — another edition of the SAME tile
{ "NODE_ID": "aoe2_gog", "ROLE": "launchable",
  "UID": "1002", "GAME": "aoe2", "LABEL": "GOG edition",
  "META": { "TITLE": "Age of Empires II" },
  "PLATFORM": { "HOST": "win32" },
  "EXEC": { "CONTENTPATH": "empires2.exe" },
  "PARENTS": ["aoe2_gog_base"] }
```

**Behavior:** the library shows **one** "Age of Empires II" tile (grouped by `GAME: "aoe2"`); opening it offers two
variants, "Forgotten Empires" (pre-selected, `RECOMMENDED`) and "GOG edition". Each resolves its own closure and chain.

---

## 17.6 A game with an optional mod and automatic load order

Optional content + `AppendLine` = a toggleable mod that registers itself in the game's load-order file, in closure order,
with no mod-manager construct.

```jsonc
// morrowind.json
{ "NODE_ID": "morrowind", "ROLE": "launchable",
  "UID": "2050", "GAME": "morrowind", "LABEL": "GOTY", "RECOMMENDED": true,
  "META": { "TITLE": "The Elder Scrolls III: Morrowind" },
  "PLATFORM": { "HOST": "win32" },
  "EXEC": { "CONTENTPATH": "Morrowind.exe" },
  "PARENTS": ["morrowind_base", "morrowind_tribunal", "morrowind_bloodmoon"] }

// morrowind_base.json
{ "NODE_ID": "morrowind_base", "ROLE": "content",
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "morrowind.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    // register the base master in the load order (idempotent, ordered)
    { "TYPE": "FileEdit", "MODE": "AppendLine", "FILE": "Data Files/openmw.cfg", "VALUE": "content=Morrowind.esm" }
  ] }

// morrowind_tribunal.json — an OPTIONAL expansion (off by default)
{ "NODE_ID": "morrowind_tribunal", "ROLE": "content", "OPTIONAL": true, "DEFAULT": false,
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "tribunal.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    { "TYPE": "FileEdit", "MODE": "AppendLine", "FILE": "Data Files/openmw.cfg", "VALUE": "content=Tribunal.esm" }
  ] }

// morrowind_bloodmoon.json — another OPTIONAL expansion
{ "NODE_ID": "morrowind_bloodmoon", "ROLE": "content", "OPTIONAL": true, "DEFAULT": false,
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
{ "NODE_ID": "native-passthrough", "ROLE": "runner",
  "PLATFORM": { "HOST": "linux64", "GUEST": ["linux64"] },
  "EXEC": { "EXECUTABLE": "%Content%", "ARGS": [], "ENV": {}, "REMOVE_ENV": [] } }

// ge-proton10-30 — Wine-family, generates a prefix; build on a content parent
{ "NODE_ID": "ge-proton10-30", "ROLE": "runner",
  "PLATFORM": { "HOST": "linux64", "GUEST": ["win32", "win64"] },
  "EXEC": { "EXECUTABLE": "%RunnerMount%/proton",
            "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
            "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%" },
            "REMOVE_ENV": ["LD_LIBRARY_PATH"],
            "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true },
  "PARENTS": ["geproton_build"] }

// geproton_build — the Proton tree (+ a couple of runner knobs)
{ "NODE_ID": "geproton_build", "ROLE": "content",
  "LAYERS": [
    { "TYPE": "VFSZipLayer", "PATH": "GE-Proton10-30.zip", "TARGET": "",
      "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    { "TYPE": "CustomVar", "KEY": "PROTON_LOG", "DEFAULT": "0",
      "UI": { "LABEL": "Proton logging", "CONTROL": "enum",
              "CHOICES": [ { "LABEL": "Off", "VALUE": "0" }, { "LABEL": "On", "VALUE": "1" } ] } }
  ] }

// snes9x — a native-Linux emulator (one bridge hop for SNES content)
{ "NODE_ID": "snes9x", "ROLE": "runner",
  "PLATFORM": { "HOST": "linux64", "GUEST": ["snes"] },
  "EXEC": { "EXECUTABLE": "snes9x", "ARGS": ["-fullscreen", "%Content%"], "ENV": {}, "REMOVE_ENV": [] } }
```

A *cloned terminal* that wraps every launch in `gamescope` (chapter 11 §11.3) is just another native runner the user can
select as the terminal step:

```jsonc
{ "NODE_ID": "native-gamescope", "ROLE": "runner",
  "PLATFORM": { "HOST": "linux64", "GUEST": ["linux64"] },
  "EXEC": { "EXECUTABLE": "gamescope", "ARGS": ["-f", "--"], "ENV": {}, "REMOVE_ENV": [] } }
```

Selected as the terminal, it composes `gamescope -f -- <whatever the chain produced>` — wrapping the game, or Proton, or
Proton-wrapping-an-emulator, uniformly.

Next: [Reference implementation map](18-reference-implementation.md).
