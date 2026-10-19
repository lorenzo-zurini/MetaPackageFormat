# 18 · Worked examples

Packages from the smallest to the most composed. §18.1, §18.3 and §18.7 are taken from the reference library;
the others are simplified from real packages (offsets, keys and file names are illustrative). Node CIDs are shortened
(`bafkrei…name`); in real files they are full CIDs.

## 18.1 A console ROM: one node, one runner

```jsonc
// [299] Super Metroid — the whole package
{ "LABEL": "super_metroid_base", "VARIANT": "super_metroid_base", "RECOMMENDED": ["299"],
  "LAYERS": [
    { "FILE": "Super Metroid (Japan, USA) (En,Ja).sfc", "SOURCE": "QmRVRg1s…", "SIZE": 3145728 },
    { "EXEC": [ { "LABEL": "Play", "HOST": "snes", "EXE": "Super Metroid (Japan, USA) (En,Ja).sfc", "ARGS": [],
                  "TILE": { "UID": "299", "TITLE": "Super Metroid",
                            "COVER": { "FILE": "Super_Metroid.png", "SOURCE": "QmTr9QpV…", "SIZE": 953516 },
                            "META": { "DEVELOPER": "Nintendo", "SERIES": "Metroid", "RELEASEDATE": "1994-04-18" } } } ] } ] }

// snes9x, in the runner library: a system command, so no build
{ "LABEL": "snes9x_exec", "LAYERS": [
    { "EXEC": [ { "LABEL": "snes9x_exec", "HOST": "linux64", "GUEST": ["snes"], "EXE": "snes9x",
                  "ARGS": ["-fullscreen", "%Content%"] } ] } ] }
```

Chain: `[snes9x, native]`. The ROM lands at the runtime root; `%Content%` is its host path.

## 18.2 A Windows game under Proton, with an option and a registry value

```jsonc
{ "LABEL": "Silent Hill 2 Enhanced", "VARIANT": "Enhanced", "RECOMMENDED": ["10972"],
  "LAYERS": [
    { "NODE": "bafkrei…sh2_pristine" },                                       // the game and its Play entry + tile
    { "ZIP": "SH2EE.zip", "SOURCE": "Qm…", "TARGET": "FILES/%GameDir%" },     // the enhancement files, above it
    { "VARS": { "FULLSCREEN": { "DEFAULT": "1",
                  "UI": { "LABEL": "Fullscreen", "CONTROL": "bool", "SECTION": "Display" } } } },
    { "REG": { "HKCU": { "Software": { "nipkow": { "SH2EEsetup": {
               "lang": "en", "fullscreen": "%FULLSCREEN:dword%", "install": "%GameDir%" } } } } },
      "ARCH": ["64"] },
    { "KEEP": { "FILES/%GameDir%/savedata/": { "NAME": "Saves" } } } ] }
```

Chain: `[proton, native]`. `%GameDir%` is `C:\10972` to the game and `pfx/drive_c/10972` in the runtime. The
registry is composed into the launch's hives; the saves folder is the user's.

## 18.3 A game with its own engine, two builds of it, and a translation

Cave Story's package holds the original Japanese release, the English translation, and the doukutsu-rs engine in
Linux and Windows builds:

```jsonc
// the original release: a Windows game
{ "LABEL": "Doukutsu Monogatari 1.0.0.6", "LAYERS": [
    { "ZIP": "Doukutsu Monogatari 1.0.0.6.zip", "SOURCE": "QmXLSRno…", "TARGET": "FILES/%GameDir%" },
    { "KEEP": { "FILES/%GameDir%/Profile.dat": true, "FILES/%GameDir%/Config.dat": true } },
    { "EXEC": [ { "LABEL": "Play", "HOST": "win32", "EXE": "%GameDir%/Doukutsu.exe",
                  "TILE": { "UID": "8333", "TITLE": "Cave Story" } } ] } ] }

// the English translation: the original, plus the 166 files the translation changes
{ "LABEL": "Cave Story - Aeon Genesis Translation 1.03", "VARIANT": "English (Aeon Genesis 1.03)",
  "RECOMMENDED": ["8333"], "LAYERS": [
    { "NODE": "bafkrei…doukutsu_1_0_0_6" },
    { "ZIP": "Cave Story - Aeon Genesis Translation 1.03.zip", "SOURCE": "QmeGfexn…", "TARGET": "FILES/%GameDir%" } ] }

// the engine, as the package's own runner (Linux build; the Windows build is the same with HOST "win64")
{ "LABEL": "doukutsu_rs_1_0_0_linux64_exec", "LAYERS": [
    { "ZIP": "doukutsu-rs 1.0.0 - Linux x64.zip", "SOURCE": "QmRS8s48…", "TARGET": "FILES/__doukutsu-rs" },
    { "ENV": { "CAVESTORY_DATA_DIR": "%RuntimePath%/game/cavestory/data", "XDG_DATA_HOME": "%TempPath%/doukutsu-rs" } },
    { "EXEC": [ { "LABEL": "doukutsu_rs_1_0_0_linux64_exec", "HOST": "linux64", "GUEST": ["custom"],
                  "EXE": "%RunnerMount%/__doukutsu-rs/doukutsu-rs", "ARGS": [],
                  "CONTENT_ROOT": "game/cavestory",
                  "GUEST_ROOTS": { "%GameDir%": "C:\\cavestory" }, "DRIVES": { "C:": "game" } } ] } ] }

// the engine variant: the translation's data folder, nothing else, run by the package's engine
{ "LABEL": "doukutsu-rs", "VARIANT": "doukutsu-rs (open-source engine)", "LAYERS": [
    { "NODE": "bafkrei…aeon_genesis", "TAKE": ["FILES/%GameDir%/data"], "TARGET": "FILES/%GameDir%" },
    { "KEEP": { "FILES/%GameDir%/user/": true } },
    { "EXEC": [ { "LABEL": "Play", "HOST": "custom", "ARGS": [], "TILE": { "UID": "8333" } } ] } ] }
```

- The English variant runs the original Windows executable under Proton (`win32`).
- The engine variant's entry is `custom`, so only an engine in this package can run it (chapter 10 §10.6). With the
  Linux build installed the chain is `[doukutsu-rs linux64, native]`; with only the Windows build and Proton
  installed it is `[doukutsu-rs win64, proton, native]` (chapter 11 §11.4).
- The engine variant takes only `data` from the translation: the Windows executables never mount.

## 18.4 A version history in deltas

```jsonc
{ "LABEL": "wc3_1_26a", "LAYERS": [ { "NODE": "bafkrei…wc3_1_24e" },
    { "DELTA": "delta_from__wc3_1_24e.vgdelta", "SOURCE": "Qm…", "TARGET": "FILES/%GameDir%" } ] }

{ "LABEL": "wc3_1_27a", "VARIANT": "1.27a", "LAYERS": [ { "NODE": "bafkrei…wc3_1_26a" },
    { "DELTA": "delta_from__wc3_1_26a.vgdelta", "SOURCE": "Qm…", "TARGET": "FILES/%GameDir%" },
    { "EXEC": [ { "LABEL": "Frozen Throne", "EXE": "%GameDir%/Frozen Throne.exe" } ] } ] }
```

Each version contains the one before and rebuilds the game folder from it with a delta. The pristine at the bottom
declares both entries and both tiles (Reign of Chaos `802`, The Frozen Throne `803` with `PARENTUID 802`); a version
changes only the field it must (here, the TFT executable). Every version from 1.21b is a row under both tiles.
Getting version *N* fetches the base and the deltas on its path.

## 18.5 Grafts: a soundtrack, a fix and a frame-rate mode

```jsonc
{ "LABEL": "Wipeout XL FLAC soundtrack", "RECOMMENDED": ["17260"], "SECTION": "Soundtrack", "LAYERS": [
    { "ANY": ["bafkrei…woxl_sp", "bafkrei…woxl_mp"] },
    { "NODE": "bafkrei…woxl_flac_files", "TARGET": "FILES/%GameDir%/music" },
    { "EDIT": [ { "MODE": "ConfigWrite", "KEY": "MusicFormat=", "VALUE": "flac" } ],
      "TARGET": "FILES/%GameDir%/wipeout.ini" } ] }

{ "LABEL": "Wipeout XL 200 fps", "LAYERS": [
    { "ANY": ["bafkrei…woxl_sp", "bafkrei…woxl_mp"] },
    { "VARS": { "fps": { "DEFAULT": "120",
                  "UI": { "LABEL": "Frame rate", "CONTROL": "enum", "CHOICES": [
                          { "LABEL": "60", "VALUE": "60" }, { "LABEL": "120", "VALUE": "120" }, { "LABEL": "240", "VALUE": "240" } ] } } } },
    { "EDIT": [ { "MODE": "Poke", "OFFSET": "0x4a2c10", "VALUE": "%fps:u16le%", "EXPECT": "1e00",
                  "COMMENT": "the frame limiter's target" } ],
      "TARGET": "FILES/%GameDir%/Wipeout2.exe" } ] }
```

Both are offered on the single-player and multiplayer variants; the soundtrack is pre-ticked. Ticked, a graft folds
above the variant; its option appears in the launch window; its binary op patches the executable the variant left.

## 18.6 A daisy chain

The Vortex console of chapter 11:

```jsonc
// the Windows-only emulator, as a runner
{ "LABEL": "vortexemu_win", "LAYERS": [
    { "FILE": "vortexemu.exe", "SOURCE": "Qm…" },
    { "EXEC": [ { "LABEL": "vortexemu", "HOST": "win32", "GUEST": ["vortex"],
                  "EXE": "%RunnerMount%/vortexemu.exe", "ARGS": ["%Content%"] } ] } ] }

// the game
{ "LABEL": "Vortex Quest", "VARIANT": "Vortex Quest", "LAYERS": [
    { "FILE": "VortexQuest.vtx", "SOURCE": "Qm…", "TARGET": "FILES/%GameDir%" },
    { "EXEC": [ { "LABEL": "Play", "HOST": "vortex", "EXE": "%GameDir%/VortexQuest.vtx",
                  "TILE": { "UID": "9001", "TITLE": "Vortex Quest" } } ] } ] }
```

Chain: `[vortexemu, proton, native]`. Proton is the boundary; vortexemu's build mounts at
`pfx/drive_c/9001/__runner_<id>__`, and the command is
`proton waitforexitandrun "C:\9001\__runner_<id>__\vortexemu.exe" "C:\9001\VortexQuest.vtx"` (chapter 11 §11.6).

## 18.7 The runner library: a Proton build assembled from shared libraries

```jsonc
{ "LABEL": "geproton_7_55", "LAYERS": [
    { "NODE": "bafkrei…geproton_7_55_files" },
    { "NODE": "bafkrei…dxvk_2_3",          "TARGET": "FILES/files/lib64/wine/dxvk" },
    { "NODE": "bafkrei…vkd3d_proton_2_9",  "TARGET": "FILES/files/lib64/wine/vkd3d-proton" },
    { "NODE": "bafkrei…wine_mono_7_4",     "TARGET": "FILES/files/share/wine/mono" },
    { "DIR": "%DefaultPfxDir%",   "TARGET": "FILES/pfx" },
    { "DIR": "%WineSys32Dir%",    "TARGET": "FILES/pfx/drive_c/windows/system32" },
    { "DIR": "%WineSysWow64Dir%", "TARGET": "FILES/pfx/drive_c/windows/syswow64" },
    { "EDIT": [ { "MODE": "Overwrite", "VALUE": "%PROTON_VERSION%" } ], "TARGET": "FILES/version" },
    { "VARS": { "PROTON_VERSION": { "DEFAULT": "GE-Proton7-55" } } },
    { "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%", "LD_LIBRARY_PATH": null } },
    { "EXEC": [ { "LABEL": "geproton_7_55_runner_exec", "HOST": "linux64", "GUEST": ["win32", "win64"],
                  "EXE": "%RunnerMount%/proton", "ARGS": ["waitforexitandrun", "%GameDir%\\%ContentPath%"],
                  "CONTENT_ROOT": "pfx/drive_c/%PackageUID%", "PREFIX_GENERATE": true,
                  "GUEST_ROOTS": { "%GameDir%": "C:\\%PackageUID%", "%UserProfile%": "C:\\users\\steamuser",
                                   "%Documents%": "C:\\users\\steamuser\\Documents",
                                   "%SysDir32%": "C:\\windows\\syswow64", "%SysDir64%": "C:\\windows\\system32" },
                  "DRIVES": { "C:": "%PrefixRoot%/drive_c" } } ] } ] }
```

The Proton tree and each library are separate nodes in separate packages, contained and placed where Proton expects
them. Another Proton version contains the same DXVK node: one copy on disk, one CID on the network. The `DIR` layers
assemble the prefix at launch (chapter 10 §10.9).

Next: [Reference implementation map](19-reference-implementation.md).
