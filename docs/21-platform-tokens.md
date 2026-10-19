# 21 · Appendix: platform tokens

Platform tokens are opaque strings compared for equality (chapter 10 §10.1). Any author may use any token; this
appendix is the **canonical list** and the rules for naming a new one, so that two authors never spell one platform
two ways.

## 21.1 What a token names

A platform token names a **contract between content and the programs that can run it**: "this is content of
format X; anything that runs X may run it." It names neither an engine nor a game. `snes` works because a SNES ROM
runs on snes9x, bsnes or Mesen without knowing which; a package that says `HOST "snes"` never names one.

Every engine is one of two kinds:

| Kind | Examples | Token |
|------|----------|-------|
| **An engine for a family of games**: its data format is a platform, like a console's | Doom source ports, GemRB, ScummVM, Build ports, Quake ports, DOSBox, Java runtimes | a canonical token named after the **format** (`doom`, `infinity`), served by a shared runner in its own package |
| **An engine for one game**: it is part of that game | doukutsu-rs, VCMI, OpenTricky, Gardens of Kadesh, a static recompilation | `custom`, with the engine in the game's own package (chapter 10 §10.6) |

**The test:** could a second game, or a second engine, ever use this token? If yes, it is a platform and gets a
token. If no, it is `custom`. An engine *designed* for a family gets the family token from its first game on, even
when only one of its games is packaged, so nothing has to be re-minted when the second arrives.

## 21.2 Naming rules

1. Lower case ASCII letters and digits, words joined by `_`.
2. Name the **data format or ecosystem**, never an engine binary: `doom`, not `gzdoom`; `infinity`, not `gemrb`.
3. **Refine with a suffix** only when runners genuinely differ in what they can run: `java_8`, `java_17`; `doom`,
   `doom_boom`, `doom_zdoom`. Content declares the **narrowest** token it needs; a runner lists in `GUEST` every
   token it can serve. A vanilla Doom WAD (`doom`) then runs on any Doom port, while a ZDoom-only mod
   (`doom_zdoom`) finds only the ports that list it:

   ```
   GZDoom          GUEST ["doom", "doom_boom", "doom_zdoom", "heretic", "hexen", "strife"]
   PrBoom+         GUEST ["doom", "doom_boom"]
   Chocolate Doom  GUEST ["doom"]
   ```

4. Consoles and computers use their established system names (No-Intro / libretro usage), shortened to one word.
5. Do not invent a token for one game: that is what `custom` is for.

## 21.3 The canonical list

**Machine platforms** (the host a native terminal runs on):

| Token | Platform |
|-------|----------|
| `linux64` | Linux, x86-64 |
| `win32` | Windows, 32-bit x86 programs |
| `win64` | Windows, 64-bit x86 programs |
| `macos_arm64`, `macos64` | macOS on Apple silicon / x86-64 (reserved) |
| `linux_arm64` | Linux on 64-bit ARM (reserved) |

**Runtimes and computers:**

| Token | Content |
|-------|---------|
| `dos` | MS-DOS programs (DOSBox and its forks) |
| `java_8`, `java_17`, `java_21`, `java_25` | Java bytecode needing at least that runtime version |

**Consoles** (examples; the full set follows rule 4): `nes`, `snes`, `n64`, `gb`, `gbc`, `gba`, `nds`, `gamecube`,
`wii`, `megadrive`, `mastersystem`, `saturn`, `dreamcast`, `psx`, `ps2`, `psp`, `xbox`.

**Engine families:**

| Token | Content | Runners (examples) |
|-------|---------|--------------------|
| `infinity` | Infinity Engine game data (Baldur's Gate, Icewind Dale, Planescape: Torment) | GemRB |
| `doom`, `doom_boom`, `doom_zdoom` | Doom-engine IWADs and PWADs, by the features they need | GZDoom, PrBoom+, Chocolate Doom |
| `heretic`, `hexen`, `strife` | the other id Tech 1 games | GZDoom, Chocolate Heretic/Hexen/Strife |
| `quake`, `quake2`, `quake3` | Quake-engine game data | vkQuake, Yamagi Quake II, ioquake3 |
| `build` | Build-engine game data (Duke Nukem 3D, Blood, Shadow Warrior) | Raze, EDuke32 |
| `scummvm` | games ScummVM runs (it detects the engine itself) | ScummVM |

A token not listed here is not wrong. If it names a family others will use, add it to this list.

## 21.4 `custom`

`custom` is the one reserved token. Content with `HOST "custom"` is run only by a runner with `GUEST ["custom"]` in
the **same package** (chapter 10 §10.6). There is no list of `custom` engines and never needs to be.

Next: back to the [README](../README.md).
