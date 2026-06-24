# 11 · Runner daisy-chaining

This is the chapter that makes "platform-agnostic" literal. A launchable states *what platform its content is*; the
runtime *constructs* the shortest sequence of runners that carries that content to the machine platform, nesting them
into a single executable command. No package authors a chain — chains are derived from runner edges (chapter 10).

## 11.1 The problem

Throughout this chapter we use a **hypothetical** console, the *Vortex* (platform token `vortex`), to make the
cross-platform case concrete. Suppose the only emulator that exists for the Vortex, *VortexEmu*, is a **Windows** program
— there is **no** native-Linux Vortex emulator. (This is the realistic shape of the problem: a platform whose only
tooling lives on a *different* platform. If a native emulator existed, you'd just use it — chaining matters precisely when
it doesn't.)

So to run Vortex content on `linux64` we have:

- `vortexemu.exe` — a **win32** program that runs `vortex` content (edge `vortex → win32`), and
- Proton — a **linux64** runtime that runs `win32` programs (edge `win32 → linux64`).

Composing them, `vortex` content runs: the game inside `vortexemu.exe` inside Proton on Linux. Three platforms,
abstracted. The runtime must (a) *find* this route and (b) *execute* it as one nested command with paths translated across
the Wine boundary. Both are mechanical given the platform graph — and because there is no shorter route, the runtime finds
this one **automatically**, with nothing pinned by hand.

## 11.2 The chain

A **runner chain** is an ordered list of runners `[R₁ … Rₙ]`, innermost to outermost:

- `R₁` runs the **content**.
- `R₍ᵢ₊₁₎` runs `Rᵢ`'s command.
- `Rₙ` is the **native terminal**: a runner whose host and guest are both the machine platform. The implementation
  `execve`s **only `Rₙ`**; every inner runner is an argument nested inside it (invariant I4).

For our Vortex example on `linux64`:

```
[ vortexemu(vortex→win32),  proton(win32→linux64),  native(linux64→linux64) ]
   R₁ runs the game           R₂ runs vortexemu        R₃ runs proton (= the terminal the runtime execs)
```

For a plain Windows game: `[ proton(win32→linux64), native ]`. For a native Linux game:
`[ native(linux64→linux64) ]` — the terminal runs the content directly. For content whose platform *does* have a native
runner (e.g. a console with a native-Linux emulator), the chain is a single bridge hop plus the terminal — daisy-chaining
across a foreign platform only happens when there is no shorter route.

## 11.3 The native terminal is always appended

Every chain ends with a native terminal (host = guest = machine). This is a deliberate design choice with three payoffs:

1. **Uniform execution primitive.** The terminal is the single `execve`. Whatever is inside — a game, Proton, Proton
   wrapping an emulator — the terminal runs it the same way.
2. **One place to wrap everything.** Because *every* launch goes through the terminal, replacing or cloning the terminal
   runner (e.g. a variant that prepends `gamescope` or `mangohud`, or sets an env var) wraps **all** launches uniformly.
   This is why the native runner is an explicit, authorable node and not a hidden built-in.
3. **It costs nothing in the common case.** A pass-through terminal (empty/`%Content%` executable) simply forwards the
   inner command, so `[proton, native]` produces byte-identical execution to running Proton directly.

If no native-terminal runner is authored in the graph, the implementation MUST synthesize a pass-through terminal so
chains still complete (back-compat / safety). An authored terminal (or a user's clone of it) takes precedence.

## 11.4 Resolving the chain (BFS)

Given the launchable's content platform `start` and the machine platform `goal`, build the chain:

```
function ResolveChain(graph, start, goal, launch):
    runners = [ r in graph.runners
                if RunnerAvailable(r) ]                 # PATH-resolvable OR ships a build (ch.10 §10.6)
    sort runners better-first by:                       # tie-break, applied at each BFS step
        RECOMMENDED, then package-local (same bundle as `launch`), then NODE_ID

    # 1. Honor a pinned chain if one is saved/requested and forms a valid path to `goal`.
    if pinnedChain validates as a connected path start→…→goal over `runners`:
        return ensureTerminal(pinnedChain)

    # 2. Otherwise BFS the platform graph for the shortest bridge start→goal.
    bridge = BFS over runner edges from `start` to `goal`      # empty if start == goal (native content)
    if bridge is unreachable: return []                        # unrunnable ⇒ refuse the launch (invariant I5)

    # 3. Append the native terminal (authored best, else synthesized pass-through).
    return bridge + [ pickNativeTerminal(runners, goal) or SYNTH_TERMINAL ]
```

Key points:

- **Shortest path.** BFS over runner edges yields the fewest-runner chain — if a platform can reach the machine both
  directly (a native runner) and via an intermediate platform, the direct one-hop wins automatically.
- **At least one edge.** Even when `start == goal` (native content), the chain has the terminal (one self-loop edge): the
  terminal runs the content. So native content still routes through the (cloneable) native runner.
- **Determinism / tie-breaks.** Runners are pre-sorted *better-first* (RECOMMENDED > package-local > id) so that when
  several runners could bridge the same step, the best one is chosen and the result is stable.
- **Availability.** Only runners that can actually run on this machine participate (chapter 10 §10.6) — including
  build-shipping runners whose exe isn't on `PATH`.
- **Pinning & user choice.** A user (or a saved setting) MAY pin a specific chain; the resolver honors it if it still
  forms a valid path, appending the terminal if omitted. This is how a user picks, say, the *Windows* build of an
  emulator over the native one (§11.7).

(VidyaGod: `ResolveChainIds` / `ResolveChainTail` / `ResolveRunnerChain` in `launchresolver.cpp`; per-step tie-break and
BFS as above. The chain is persisted per package as `RUNNER_CHAIN`.)

## 11.5 Composing the command: nesting

The runtime executes **one** process (the terminal), built by nesting from the inside out. Two kinds of link compose
differently, decided by whether a link is a **namespace boundary** (it introduces a guest filesystem — a non-empty
`CONTENT_ROOT` / a generated prefix):

### Same-namespace links (native wrappers, the terminal)

Links that run in the **host** namespace (host paths, no prefix). They compose by **command concatenation**:

```
wrappedArgv = [ wrapper.EXECUTABLE ] + wrapper.ARGS + innerProgram + innerArgs
```

A pass-through wrapper (empty `EXECUTABLE`, or one referencing `%Content%`) contributes nothing and **forwards the inner
argv unchanged**. A real wrapper tool (e.g. `gamescope`, `mangohud`) prepends its executable and args, then the inner
command. Env from every same-namespace link is merged into the single process environment (innermost wins on conflict;
all `REMOVE_ENV` applied).

This is what makes the native terminal free in the common case, and what makes "clone the native runner to add
`gamescope`" work: `gamescope -- proton waitforexitandrun C:\…\game.exe`.

### Cross-namespace links (an inner runner inside a boundary's guest fs)

When an inner runner runs *inside* a boundary runner's guest filesystem (VortexEmu inside Proton's Wine), concatenation is
not enough — the inner runner's executable and arguments are *files inside the guest fs* and must be expressed in the
boundary's **guest path namespace** (Wine `C:\…` paths), and the inner runner's build must be **mounted into** that fs.
This is §11.6.

The **boundary runner** of a chain is the *outermost* link that creates a guest fs (e.g. Proton). Links *inside* it
(VortexEmu) are cross-namespace inner links; links *outside* it (the native terminal, any native wrappers) are
same-namespace. The implementation composes inner links into the boundary's content target, then wraps the boundary with
the outer same-namespace links.

## 11.6 Cross-namespace nesting in detail

For a chain `[vortexemu, proton, native]` with boundary `proton`, running the Vortex game with `UID` `9001` and ROM
`VortexQuest.vtx`:

**1. Mounting.** Each inner runner's build mounts as content inside the boundary's content root, at a reserved
subdirectory `<CONTENT_ROOT>/__runner_<innerNodeId>__/`. So `vortexemu.exe` lands at
`pfx/drive_c/9001/__runner_vortexemu_win__/vortexemu.exe`, which Wine sees at
`C:\9001\__runner_vortexemu_win__\vortexemu.exe`. The actual content (the ROM) mounts at the content root as usual.
(VidyaGod: the inner-build mount loop in `vfsmount.cpp::BuildLayerSpec`, gated on the chain having inner links.)

**2. Guest-path translation.** The boundary declares (or derives) a `GUEST_PATH` template mapping a content-root-relative
path to its guest path. For Wine, that's `C:\<uid-part>\%REL%`. The implementation:

- Derives the template from `CONTENT_ROOT` when `GUEST_PATH` is absent: a `CONTENT_ROOT` containing `drive_c/<sub>` →
  template `C:\<sub>\%REL%`, with `/`→`\` conversion on `%REL%`. So **no runner edit is needed** — Proton/Wine/umu get
  cross-namespace translation for free from their existing `CONTENT_ROOT`.
- Translates the ROM's relative path and each inner runner's exe/args into guest paths via the template.

**3. Command composition.** The inner runners' guest command is built innermost-out, then becomes the boundary's content
target:

```
romGuest        = GuestPath(template, content.CONTENTPATH)              # C:\9001\VortexQuest.vtx
emuGuest        = GuestPath(template, "__runner_vortexemu_win__/vortexemu.exe")  # C:\9001\__runner_vortexemu_win__\vortexemu.exe
innerArgv       = [ emuGuest ] + vortexemu.ARGS with %Content% = romGuest        # [emuGuest, romGuest]

# the boundary (proton) runs innerArgv: its %ContentPath% is redirected to the inner exe,
# and the inner args trail the boundary's ARGS:
protonArgv      = proton.EXECUTABLE
                + proton.ARGS with %ContentPath% = "__runner_vortexemu_win__/vortexemu.exe"
                + [ romGuest ]                                         # the trailing inner args
```

The resulting boundary command, before wrapping by the (pass-through) terminal:

```
/…/RUNNER/proton  waitforexitandrun  "C:\9001\__runner_vortexemu_win__\vortexemu.exe"  "C:\9001\VortexQuest.vtx"
```

which is exactly "Proton, run vortexemu.exe, which loads the ROM." (VidyaGod: `ComposeGuestTarget` in
`launchresolver.cpp`; the boundary's content tokens are redirected and the trailing args appended in
`ContainerWrapper::Execute`.)

**Env across the boundary.** Same-namespace links share the one host process environment. A cross-namespace inner
runner's environment is *guest* environment (inside Wine) and is carried by the boundary runner's own mechanism (its
`ENV`), not merged into the host process env. (Inner-runner guest env injection is an area the format leaves to the
boundary's runtime; most emulators need none.)

## 11.7 Exposing the chain to the user

The chain is not just internal: an implementation SHOULD surface it as an **editable, per-step control** so a user can
deviate from the default. The model (VidyaGod's prelaunch UI):

- One control per chain step. Step *i* shows `"<input platform> → ???"`, where the choices are the installed runners that
  *consume* that input platform (`GUEST ∋ input`), regardless of their host.
- Choosing a runner fixes its host = the next step's input. The downstream steps **re-resolve** from there (a cascade):
  pick VortexEmu at step 1 → its host is `win32` → the next step offers `win32 → ???` (Proton, Wine, …) → and so on until
  a step reaches the machine platform, where the native terminal is offered.
- A hint shows the target (`→ linux64`) and whether the current chain reaches it.
- The chosen chain is persisted (per package) and honored by the resolver on the next launch.

This is what lets a user deviate deliberately — pick *Wine* instead of *Proton* for the win32 step, or swap the native
terminal for a clone that wraps the launch in `gamescope` (§11.3). And it is the same mechanism that will expose ARM
routes when ARM runners exist.

## 11.8 Worked example: the Vortex daisy chain

The Windows-only VortexEmu emulator, shipped as a runner (its build on a content parent):

```json
// vortexemu_win.json  (a runner — VortexEmu is a win32 program)
{ "NODE_ID": "vortexemu_win", "ROLE": "runner",
  "PLATFORM": { "HOST": "win32", "GUEST": ["vortex"] },
  "EXEC": { "EXECUTABLE": "vortexemu.exe", "ARGS": ["%Content%"], "ENV": {}, "REMOVE_ENV": [] },
  "PARENTS": ["vortexemu_win_build"] }

// vortexemu_win_build.json  (its build: the win32 binary)
{ "NODE_ID": "vortexemu_win_build", "ROLE": "content",
  "LAYERS": [ { "TYPE": "VFSFileLayer", "PATH": "vortexemu.exe" } ] }
```

Launch a Vortex game (`PLATFORM.HOST: "vortex"`, `UID: "9001"`, `CONTENTPATH: "VortexQuest.vtx"`). Because the only route
from `vortex` to `linux64` runs through win32, the runtime resolves the chain **automatically** — nothing is pinned:

1. Resolves `[vortexemu_win, ge-proton10-30, native-passthrough]` (shortest, and only, path).
2. Mounts the ROM at `pfx/drive_c/9001/VortexQuest.vtx`, `vortexemu.exe` at
   `pfx/drive_c/9001/__runner_vortexemu_win__/vortexemu.exe`.
3. Derives Proton's guest template from its `CONTENT_ROOT` → `C:\9001\%REL%`.
4. Composes and execs:
   ```
   proton  waitforexitandrun
       "C:\9001\__runner_vortexemu_win__\vortexemu.exe"
       "C:\9001\VortexQuest.vtx"
   ```
5. Proton starts, Wine launches vortexemu.exe, VortexEmu loads the ROM. `vortex → win32 → linux64`, executed.

> Note the inner runner's `EXECUTABLE` is a *build-relative* `vortexemu.exe` (it runs inside Wine, not from the host
> `PATH`), and the runner is "available" because it ships a build (chapter 10 §10.6).

The reference implementation has validated this cross-namespace execution path end-to-end (a real win32 emulator running
console content under Proton, with the composed command and guest-path translation exactly as above); the worked example
here is the same mechanism on the hypothetical Vortex platform. (VidyaGod: `ResolveChainIds` + `ComposeGuestTarget` +
`ContainerWrapper::Execute`; the cross-namespace mount in `vfsmount.cpp::BuildLayerSpec`.)

## 11.9 Invariants recap

- **I4 — one process:** only the terminal is `execve`d; inner runners are nested arguments, never separately spawned.
- **I5 — always terminated:** a chain that can't reach the machine platform means the content is unrunnable; the launch
  is refused with a diagnostic, not reported as a clean exit.

Next: [Dependency resolution](12-resolution.md).
