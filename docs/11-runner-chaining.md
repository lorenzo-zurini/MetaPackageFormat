# 11 · Runner chaining

An entry states what platform its content is. The runtime **constructs** the shortest chain of runners that carries
that content to the machine platform and nests them into one command. No package authors a chain.

## 11.1 The problem

Take a hypothetical console, the *Vortex* (`vortex`), whose only emulator, *VortexEmu*, is a Windows program. On
`linux64` there is:

- `vortexemu.exe`, a `win32` program that runs `vortex` content (edge `vortex → win32`);
- Proton, a `linux64` program that runs `win32` programs (edge `win32 → linux64`).

Composed, Vortex content runs: the ROM in VortexEmu in Proton on Linux. The runtime must find this route and run it
as one nested command, translating paths across the Wine boundary. Both are mechanical given the platform graph.

## 11.2 The chain

A **runner chain** is a list of runners `[R₁ … Rₙ]`, innermost first:

- `R₁` runs the content;
- `Rᵢ₊₁` runs `Rᵢ`'s command;
- `Rₙ` is the **native terminal** (host = guest = the machine platform). Only `Rₙ` is executed; every inner runner
  is an argument nested inside it (I4).

| Content | Chain on `linux64` |
|---------|--------------------|
| a Vortex ROM | `[vortexemu (vortex→win32), proton (win32→linux64), native]` |
| a Windows game | `[proton, native]` |
| a native Linux game | `[native]` |
| a game with its own Linux engine | `[engine (custom→linux64), native]` |
| a game with only a Windows engine | `[engine (custom→win64), proton, native]` |

The chain starts from the platform of the **entry that runs** (the selected row's entry, or a ticked graft's), never
from a node-level default: a node with a native entry and a Windows entry resolves a chain for each.

## 11.3 The native terminal

Every chain ends in a native terminal. This gives one execution primitive (whatever is inside, the terminal runs it
the same way) and one place to wrap every launch (a terminal that prepends `gamescope` or sets an environment
variable wraps everything). A pass-through terminal (empty or `%Content%` `EXE`) forwards the inner command
unchanged, so `[proton, native]` runs exactly what Proton alone would.

If no native terminal is available in the library, the implementation MUST synthesize a pass-through one. An
available authored terminal is preferred.

## 11.4 Finding the chain

```
function ResolveChain(library, launch, entry, pinned):
    start  = entry.HOST
    goal   = machine platform
    all    = available runners (chapter 10 §10.7), sorted better-first:
               RECOMMENDED for the launched tile or its family,
               then in the launch's own package folder,
               then serving fewer GUEST platforms,
               then by LABEL
    chain  = ChainOver(installed(all))          # installed runners first
    if chain is empty:
        chain = ChainOver(all)                  # else the chain to download
    return chain                                # empty: unrunnable — refuse (I5)

function ChainOver(runners):
    if pinned forms a path start → … → goal over runners, each link serving the platform before it:
        return pinned + [terminal] unless pinned already ends in one
    bridge = breadth-first search over runner edges from start to goal, runners tried better-first
    if goal unreachable: return []
    return bridge + [terminal]                  # an installed terminal, else any available one, else synthesized
```

- **Installed first.** A longer chain of installed runners beats a shorter one that still has to be downloaded:
  with a Windows engine and Proton installed, a game with both a Windows and a Linux build of its engine runs on the
  Windows one rather than waiting for the Linux one. Only when nothing installed can carry the launch is the chain
  the one a download fetches. A play gate ("can this run here?") MUST agree with this: it asks whether installed
  runners carry the launch.
- **Shortest.** Breadth-first search gives the fewest runners: a direct native runner beats a route through another
  platform.
- **Always at least the terminal.** Native content runs through the terminal alone.
- **Specialist first.** Among otherwise equal runners, the one serving fewer platforms wins: a dedicated SNES
  emulator before a multi-system one that also lists `snes`. A library can carry a multi-system emulator as an
  alternative without it becoming every console's default; the user picks it per launch or per instance.
- **Deterministic.** Runners are tried better-first, so the result is stable.
- **`custom` is package-scoped** (chapter 10 §10.6) at every step, including pins.
- **Pins.** The user (or a saved setting, VidyaGod's instance `RUNNER_CHAIN`, or the CLI's `--runner`) MAY pin a
  chain. It is honoured when it still forms a valid path over the runner set being searched; otherwise the default
  is used and the implementation SHOULD say that the pin was not honoured.

## 11.5 Composing the command

One process is executed, built from the inside out. Links compose in two ways, depending on whether a link creates a
guest file system (a `CONTENT_ROOT` or a generated prefix). The **boundary** is the outermost link that does. The
boundary owns the runtime mount, its `GUEST_ROOTS` and `DRIVES` are the anchors in effect, and links are classified
around it:

- **Outer links** (after the boundary: the terminal, native wrappers) run in the host namespace and compose by
  concatenation: `[wrapper.EXE] + wrapper.ARGS + innerProgram + innerArgs`. A pass-through contributes nothing.
  `gamescope -- proton waitforexitandrun C:\…\game.exe` is a wrapper around Proton.
- **Inner links** (before the boundary: an emulator inside Proton) run inside the boundary's guest and need
  their paths expressed in it (§11.6).

**Environment.** The host environment, then each outer link's, the boundary's and each inner link's, outermost first,
so the innermost wins a shared name; then the game's own `ENV`, which wins over every runner (chapter 07 §7.3).

## 11.6 Nesting across a namespace

For `[vortexemu, proton, native]`, boundary `proton`, a Vortex game with family UID `9001` and the ROM
`VortexQuest.vtx`:

**Mounting.** Each inner runner's build is mounted inside the boundary's content root at a reserved folder,
`<CONTENT_ROOT>/__runner_<id>__/`. Under Proton, `vortexemu.exe` appears to Wine at
`C:\9001\__runner_<id>__\vortexemu.exe`. The content mounts at the content root as usual.

**Guest paths.** The boundary's guest template is its `%GameDir%` guest path followed by the relative path:
`C:\9001\%REL%`, with `/` turned into `\` in `%REL%`. A runtime-relative path becomes a guest path through it.

**An inner runner's `%RunnerMount%`** is its own build folder: `""` inside its `EXE` (so `%RunnerMount%/vortexemu.exe`
is the build-relative `vortexemu.exe`), and the folder's guest path inside its `ARGS` and `ENV`. A Windows engine
written `%RunnerMount%/x.exe`, as it must be to run directly on Windows, nests correctly under Proton.

**Composition**, innermost out:

```
romGuest   = GuestPath("VortexQuest.vtx")                               # C:\9001\VortexQuest.vtx
emuGuest   = GuestPath("__runner_<id>__/vortexemu.exe")                 # C:\9001\__runner_<id>__\vortexemu.exe
innerArgv  = [emuGuest] + vortexemu.ARGS with %Content% = romGuest

boundary   = proton.EXE
           + proton.ARGS with %ContentPath% = "__runner_<id>__/vortexemu.exe"
           + innerArgv[1:]                                              # the inner arguments trail
```

giving, before the pass-through terminal:

```
/…/RUNNER/proton  waitforexitandrun  "C:\9001\__runner_<id>__\vortexemu.exe"  "C:\9001\VortexQuest.vtx"
```

**Environment.** An inner runner's `ENV` reaches the process (Wine passes its environment to Windows programs),
expanded with its own `%RunnerMount%` and merged as in §11.5.

*(VidyaGod: `LaunchResolver::ResolveChainIds`, `ComposeGuestTarget`, `ApplyInnerEnv`; the inner-build mounts in
`vfsmount.cpp`.)*

## 11.7 Showing the chain to the user

An implementation SHOULD show the chain as one editable control per step:

- step *i* reads `<input platform> → ?`; its choices are the installed runners whose `GUEST` contains that platform
  (package-scoped for `custom`), whatever their host;
- choosing a runner fixes the next step's input (its host), and the steps after it are re-resolved;
- a hint shows whether the chain reaches the machine platform;
- the chosen chain is saved in the instance and honoured next launch (§11.4).

This lets a user deliberately pick Wine over Proton, a Windows build of an emulator over a native one, or a terminal
that wraps the launch in `gamescope`.

## 11.8 Invariants recap

- **I4, one process.** Only the terminal is executed; inner runners are nested arguments.
- **I5, always terminated.** No route to the machine means the launch is refused with a diagnostic, never reported as
  a clean exit.

Next: [Resolution](12-resolution.md).
