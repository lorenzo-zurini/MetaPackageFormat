# 17 · Worked examples

Complete node sets that exercise the whole spec, each with the behaviour it resolves to. Comments (`//`) are for the
reader; strip them for real JSON. Refs are written as readable handles; in a real tree they are CIDs.

Throughout, the runner library (Proton, the native terminal, emulators) lives in a separate bundle — see
[§17.8](#178-the-runner-library).

Read every graph **bottom-up**: a node's `OVER` is what must be mounted before it; the launchable sits above its own
closure, and grafts above that.

---

## 17.1 A native Linux game — one node

The simplest case: content whose platform *is* the machine platform. Content, tile and entrypoint are ONE node.

```
[1234] My Linux Game/
├── My Linux Game.json
└── mylinuxgame.zip            // STORE zip of the game tree, exe at the root
```

```jsonc
{ "LABEL": "My Linux Game",
  "TILE": { "UID": "1234", "TITLE": "My Linux Game" },
  "LAYERS": [ { "FORM": "zip", "PATH": "mylinuxgame.zip" } ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "linux64", "PATH": "mygame", "ARGS": ["--fullscreen"] } ] }
```

**Resolves to:** chain `[native-passthrough]`; the terminal runs `<runtime>/mygame --fullscreen`.

---

## 17.2 A Windows game under Proton, with a knob and a registry default

```
[7804] Age of Mythology/
├── Age of Mythology.json
├── aom_config.json
├── aom.zip
└── AoM_Cover.jpg
```

```jsonc
// aom_config.json — one meaningful change: a knob AND the registry default that reads it
{ "LABEL": "aom_config",
  "VARS": [ { "KEY": "WIDESCREEN", "DEFAULT": "1", "UI": { "LABEL": "Widescreen UI", "CONTROL": "bool" } } ],
  "REGEDITS": [ { "ARCHITECTURE": ["32"],
                  "HKCU": { "Software": { "Microsoft": { "Microsoft Games": { "Age of Mythology": {
                      "Widescreen": "%WIDESCREEN:dword%" } } } } } } ] }

// Age of Mythology.json — the launchable, OVER its config
{ "LABEL": "Age of Mythology", "OVER": ["aom_config"],
  "TILE": { "UID": "7804", "TITLE": "Age of Mythology",
            "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
            "META": { "UMUID": "266840" } },
  "LAYERS": [ { "FORM": "zip", "PATH": "aom.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%",
                "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/aom.exe",
                     "ARGS": ["xres=%ScreenWidth%", "yres=%ScreenHeight%"] } ] }
```

**Resolves to:** chain `[ge-proton10-30, native-passthrough]`; Proton generates a prefix, content mounts at
`pfx/drive_c/7804`, the command is `proton waitforexitandrun C:\7804\aom.exe xres=1920 yres=1080`. `WIDESCREEN`
resolves (default on → `dword:00000001`) into the default-data hive.

---

## 17.3 A console ROM via a native emulator

```jsonc
{ "LABEL": "Star Voyager",
  "TILE": { "UID": "8500", "TITLE": "Star Voyager" },
  "LAYERS": [ { "FORM": "file", "PATH": "StarVoyager.sfc" } ],
  "ENTRYPOINTS": [ { "HOST": "snes", "PATH": "StarVoyager.sfc" } ] }
```

**Resolves to:** with a native `snes9x` runner (`GUEST: ["snes"], HOST: "linux64"`), the chain is
`[snes9x, native-passthrough]`: `snes9x -fullscreen <runtime>/StarVoyager.sfc`.

---

## 17.4 A cross-platform daisy chain (a console with only a Windows emulator)

The *Vortex* is a hypothetical console whose only emulator is a Windows program.

```jsonc
// Vortex Quest.json
{ "LABEL": "Vortex Quest", "TILE": { "UID": "9001", "TITLE": "Vortex Quest" },
  "LAYERS": [ { "FORM": "file", "PATH": "VortexQuest.vtx" } ],
  "ENTRYPOINTS": [ { "HOST": "vortex", "PATH": "VortexQuest.vtx" } ] }

// vortexemu_win.json — a runner that is itself win32 content: build + entry in one node
{ "LABEL": "vortexemu_win",
  "LAYERS": [ { "FORM": "file", "PATH": "vortexemu.exe" } ],
  "ENTRYPOINTS": [ { "HOST": "win32", "GUEST": ["vortex"], "PATH": "vortexemu.exe", "ARGS": ["%Content%"] } ] }
```

No pin is needed — `vortex → win32 → linux64` is the only route. The runtime resolves
`[vortexemu_win, ge-proton10-30, native-passthrough]` and execs
`proton waitforexitandrun "C:\9001\__runner_vortexemu_win__\vortexemu.exe" "C:\9001\VortexQuest.vtx"`.

---

## 17.5 A multi-variant game (two editions, one card) and an expansion

Two launchables carrying the same `TILE.UID` are one card; the user picks a variant. An expansion is its own
tile, `OVER` the main game — and that is all: nesting is derived from the chain, never declared.

```jsonc
// aoe2_base.json — the pristine game: content only (the canonical)
{ "LABEL": "aoe2_base", "LAYERS": [ { "FORM": "zip", "PATH": "aoe2.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%" } ] }

// Vanilla.json — the MAIN: OVER no other launchable of this UID
{ "LABEL": "Vanilla", "OVER": ["aoe2_base"],
  "TILE": { "UID": "749", "TITLE": "Age of Empires II - The Age of Kings", "COVER": "aok.png" },
  "ENTRYPOINTS": [ { "LABEL": "Vanilla", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/empires2.exe" } ] }

// HD.json — a VARIANT: the same tile OVER the main (an edition), picked from the card
{ "LABEL": "HD", "OVER": ["Vanilla"],
  "TILE": { "UID": "749", "TITLE": "Age of Empires II - The Age of Kings", "COVER": "aok.png" },
  "LAYERS": [ { "FORM": "zip", "PATH": "hd_patch.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%" } ],
  "ENTRYPOINTS": [ { "LABEL": "HD", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/empires2.exe" } ] }

// The Conquerors.json — a CHILD: a different tile OVER the main = an expansion, nested under Age of Kings
{ "LABEL": "The Conquerors", "OVER": ["Vanilla"],
  "TILE": { "UID": "749", "TITLE": "Age of Empires II - The Conquerors", "COVER": "conq.png" },
  "LAYERS": [ { "FORM": "zip", "PATH": "conquerors.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%" } ],
  "ENTRYPOINTS": [ { "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/age2_x1/age2_x1.exe", "RECOMMENDED": true } ] }

// Forgotten Empires.json — a child of the child: OVER The Conquerors, its own tile
{ "LABEL": "Forgotten Empires", "OVER": ["The Conquerors"],
  "TILE": { "UID": "749", "TITLE": "Age of Empires II - Forgotten Empires", "COVER": "fe.png" },
  "LAYERS": [ { "FORM": "zip", "PATH": "fe_patch.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%" } ],
  "ENTRYPOINTS": [ { "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/age2_x1/age2_x1.5.exe" } ] }
```

**Behavior:** one card, named and covered as *Age of Kings* (the main); *Vanilla* and *HD* are its variants; *The
Conquerors* and, under it, *Forgotten Empires* are its children with their own covers; the default launch is the
`RECOMMENDED` entry, The Conquerors. Saves, settings and the content root all key on the one UID, so the expansions
install where the game is. A mod `OVER ["Vanilla"]` sits on this card and is offered when *Vanilla* is the selected
variant; `OVER [["Vanilla", "The Conquerors"]]` when either is (chapter 12: offers follow selection, not the card).
Nothing needs a `PARENTUID`: the edge already says it.

---

## 17.6 Expansions as grafts, with automatic load order

Optional expansions are **grafts**: nobody lists them, each is `OVER` the launchable, each registers itself in the
game's load-order file with an appended line. Ordering among grafts is the instance's precedence.

```jsonc
// GOTY.json — the launchable
{ "LABEL": "GOTY", "TILE": { "UID": "2050", "TITLE": "The Elder Scrolls III: Morrowind" },
  "LAYERS": [ { "FORM": "zip", "PATH": "morrowind.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ],
  "FILEEDITS": [ { "FILE": "Data Files/openmw.cfg", "OVERRIDE": true,
                   "EDITS": [ { "MODE": "AppendLine", "VALUE": "content=Morrowind.esm" } ] } ],
  "ENTRYPOINTS": [ { "LABEL": "GOTY", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/Morrowind.exe" } ] }

// Tribunal.json — a graft: files + its cfg line, one node. TOGGLE absent ⇒ offered, unticked.
{ "LABEL": "Tribunal", "OVER": ["GOTY"],
  "LAYERS": [ { "FORM": "zip", "PATH": "tribunal.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ],
  "FILEEDITS": [ { "FILE": "Data Files/openmw.cfg", "OVERRIDE": true,
                   "EDITS": [ { "MODE": "AppendLine", "VALUE": "content=Tribunal.esm" } ] } ] }

// Bloodmoon.json — another graft; it NEEDS Tribunal (a plain requirement on a graft = must be selected)
{ "LABEL": "Bloodmoon", "OVER": ["GOTY", "Tribunal"],
  "LAYERS": [ { "FORM": "zip", "PATH": "bloodmoon.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ],
  "FILEEDITS": [ { "FILE": "Data Files/openmw.cfg", "OVERRIDE": true,
                   "EDITS": [ { "MODE": "AppendLine", "VALUE": "content=Bloodmoon.esm" } ] } ] }

// Two mutually exclusive texture packs, each a graft with a NOT
{ "LABEL": "Textures HD", "OVER": ["GOTY", { "NOT": "Textures Vanilla+" }], "LAYERS": [ … ] }
{ "LABEL": "Textures Vanilla+", "OVER": ["GOTY"], "LAYERS": [ … ] }
```

**Behavior:** the prelaunch sheet offers Tribunal (tickable) and Bloodmoon ("needs Tribunal", greyed). Tick both →
both mount above GOTY, Tribunal before Bloodmoon (Bloodmoon is `OVER` it) → `openmw.cfg` ends with exactly
`content=Morrowind.esm`, `content=Tribunal.esm`, `content=Bloodmoon.esm`. Untick Tribunal → Bloodmoon is no longer
applicable and unticks with it. Tick *Textures HD* → *Vanilla+* unticks (the `NOT` is symmetric in effect). The load
order is a *consequence* of the graph and the instance, not a feature.

---

## 17.7 Skyrim at scale — versions, a loader, a thousand mods

```jsonc
{ "LABEL": "1.6.640", "TILE": { "UID": "skyrimse", "TITLE": "Skyrim Special Edition" }, "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "win64", "PATH": "…/SkyrimSE.exe" } ] }
{ "LABEL": "1.6.659", "TILE": { "UID": "skyrimse", "TITLE": "Skyrim Special Edition" }, "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "win64", "PATH": "…/SkyrimSE.exe" } ] }

// a launchable graft: the loader, on either version — picked from the card; the group is the version choice
{ "LABEL": "SKSE 2.2.6", "OVER": [["1.6.640", "1.6.659"]], "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "SKSE", "HOST": "win64", "PATH": "…/skse64_loader.exe" } ] }

{ "LABEL": "tex",      "OVER": [["1.6.640", "1.6.659"]], "LAYERS": [ … ] }              // works on two versions
{ "LABEL": "tex-hd",   "OVER": ["tex"], "LAYERS": [ … ] }                               // a graft on a graft
{ "LABEL": "ab-patch", "OVER": ["modA", "modB", "1.6.640"], "LAYERS": [ … ] }           // only when both are on
{ "LABEL": "quest",    "OVER": ["1.6.640", "SKSE 2.2.6", { "NOT": "old-quest" }],
  "LAYERS":    [ { "FORM": "file", "PATH": "quest.esp", "TARGET": "…/Data" } ],
  "FILEEDITS": [ { "FILE": "…/plugins.txt", "EDITS": [ { "MODE": "AppendLine", "VALUE": "*quest.esp" } ] } ] }
```

Pick *SKSE* → choose 1.6.640 → selected `{SKSE, 1.6.640}`. Offered: tex, quest, old-quest; blocked: tex-hd (needs
tex), ab-patch (needs modA, modB). Tick tex → tex-hd offered; tex and tex-hd both provide `rock01.dds` with
different bytes → a conflict is reported; the instance ranks tex-hd above tex, or names a winner for that file.
Tick old-quest → quest unticks. `plugins.txt` is the composed file, one appended line per ticked mod in precedence
order. A hundred such configurations are a hundred instances over one pool of nodes; a mod update is a new CID that
the user re-selects.

---

## 17.8 The runner library

A runner is a node with an entry that has `GUEST`; its build is its own `LAYERS` (or what it is `OVER`).

```jsonc
// native-passthrough — the universal terminal
{ "LABEL": "native-passthrough",
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["linux64"], "PATH": "%Content%", "ARGS": [] } ] }

// GE-Proton 10-30 — Wine-family, generates a prefix; build, knob, keep-set and entry in one node
{ "LABEL": "GE-Proton 10-30",
  "LAYERS": [ { "FORM": "zip", "PATH": "GE-Proton10-30.zip", "TARGET": "", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ],
  "VARS": [ { "KEY": "PROTON_LOG", "DEFAULT": "0",
              "UI": { "LABEL": "Proton logging", "CONTROL": "enum", "CHOICES": [ { "LABEL": "Off", "VALUE": "0" }, { "LABEL": "On", "VALUE": "1" } ] } } ],
  "PERSISTS": [ { "SCOPE": "file", "PATH": "pfx/drive_c/users", "TARGET": "users" },
                { "SCOPE": "registry", "PATH": "HKCU" } ],
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["win32", "win64"],
                     "PATH": "%RunnerMount%/proton",
                     "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
                     "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%", "PROTON_LOG": "%PROTON_LOG%" },
                     "ENV_REMOVE": ["LD_LIBRARY_PATH"],
                     "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true } ] }

// snes9x — a native-Linux emulator
{ "LABEL": "snes9x", "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["snes"], "PATH": "snes9x", "ARGS": ["-fullscreen", "%Content%"] } ] }
```

A *cloned terminal* that wraps every launch in `gamescope` is just another native runner:

```jsonc
{ "LABEL": "native-gamescope",
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["linux64"], "PATH": "gamescope", "ARGS": ["-f", "--"] } ] }
```

Next: [Reference implementation map](18-reference-implementation.md).
