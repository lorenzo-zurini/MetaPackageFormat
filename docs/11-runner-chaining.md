# 11 · Runner daisy-chaining

This is the chapter that makes "platform-agnostic" literal. A launchable states *what platform its content is*; the
runtime *constructs* the shortest sequence of runners that carries that content to the machine platform, nesting them
into a single executable command. No package authors a chain — chains are derived from runner edges (chapter 10).

## 11.1 The problem

A SNES ROM is `snes` content. There may be no runner that runs `snes` directly on `linux64`. But there might be:

- `snes9x.exe` — a **win32** program that runs `snes` content (edge `snes → win32`), and
- Proton — a **linux64** runtime that runs `win32` programs (edge `win32 → linux64`).

Composing them, `snes` content runs: the ROM inside `snes9x.exe` inside Proton on Linux. Three platforms, abstracted. The
runtime must (a) *find* this route and (b) *execute* it as one nested command with paths translated across the Wine
boundary. Both are mechanical given the platform graph.

## 11.2 The chain

A **runner chain** is an ordered list of runners `[R₁ … Rₙ]`, innermost to outermost:

- `R₁` runs the **content**.
- `R₍ᵢ₊₁₎` runs `Rᵢ`'s command.
- `Rₙ` is the **native terminal**: a runner whose host and guest are both the machine platform. The implementation
  `execve`s **only `Rₙ`**; every inner runner is an argument nested inside it (invariant I4).

For our SNES example on `linux64`:

```
[ snes9x(snes→win32),  proton(win32→linux64),  native(linux64→linux64) ]
   R₁ runs the ROM       R₂ runs snes9x          R₃ runs proton (= the terminal VidyaGod execs)
```

For a plain Windows game: `[ proton(win32→linux64), native ]`. For a native Linux game:
`[ native(linux64→linux64) ]` — the terminal runs the content directly.

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

- **Shortest path.** BFS over runner edges yields the fewest-runner chain. If a native `snes → linux64` runner exists, it
  beats the two-hop `snes → win32 → linux64` route automatically.
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

When an inner runner runs *inside* a boundary runner's guest filesystem (snes9x inside Proton's Wine), concatenation is
not enough — the inner runner's executable and arguments are *files inside the guest fs* and must be expressed in the
boundary's **guest path namespace** (Wine `C:\…` paths), and the inner runner's build must be **mounted into** that fs.
This is §11.6.

The **boundary runner** of a chain is the *outermost* link that creates a guest fs (e.g. Proton). Links *inside* it
(snes9x) are cross-namespace inner links; links *outside* it (the native terminal, any native wrappers) are
same-namespace. The implementation composes inner links into the boundary's content target, then wraps the boundary with
the outer same-namespace links.

## 11.6 Cross-namespace nesting in detail

For a chain `[snes9x, proton, native]` with boundary `proton`:

**1. Mounting.** Each inner runner's build mounts as content inside the boundary's content root, at a reserved
subdirectory `<CONTENT_ROOT>/__runner_<innerNodeId>__/`. So `snes9x.exe` lands at
`pfx/drive_c/<uid>/__runner_snes9x_win__/snes9x.exe`, which Wine sees at `C:\<uid>\__runner_snes9x_win__\snes9x.exe`. The
actual content (the ROM) mounts at the content root as usual. (VidyaGod: the inner-build mount loop in
`vfsmount.cpp::BuildLayerSpec`, gated on the chain having inner links.)

**2. Guest-path translation.** The boundary declares (or derives) a `GUEST_PATH` template mapping a content-root-relative
path to its guest path. For Wine, that's `C:\<uid-part>\%REL%`. The implementation:

- Derives the template from `CONTENT_ROOT` when `GUEST_PATH` is absent: a `CONTENT_ROOT` containing `drive_c/<sub>` →
  template `C:\<sub>\%REL%`, with `/`→`\` conversion on `%REL%`. So **no runner edit is needed** — Proton/Wine/umu get
  cross-namespace translation for free from their existing `CONTENT_ROOT`.
- Translates the ROM's relative path and each inner runner's exe/args into guest paths via the template.

**3. Command composition.** The inner runners' guest command is built innermost-out, then becomes the boundary's content
target:

```
romGuest      = GuestPath(template, content.CONTENTPATH)          # C:\299\Super Metroid….sfc
snes9xGuest   = GuestPath(template, "__runner_snes9x_win__/snes9x.exe")  # C:\299\__runner_snes9x_win__\snes9x.exe
innerArgv     = [ snes9xGuest ] + snes9x.ARGS with %Content% = romGuest  # [snes9xGuest, romGuest]

# the boundary (proton) runs innerArgv: its %ContentPath% is redirected to the inner exe,
# and the inner args trail the boundary's ARGS:
protonArgv    = proton.EXECUTABLE
              + proton.ARGS with %ContentPath% = "__runner_snes9x_win__/snes9x.exe"
              + [ romGuest ]                                       # the trailing inner args
```

The resulting boundary command, before wrapping by the (pass-through) terminal:

```
/…/RUNNER/proton  waitforexitandrun  "C:\299\__runner_snes9x_win__\snes9x.exe"  "C:\299\Super Metroid….sfc"
```

which is exactly "Proton, run snes9x.exe, which loads the ROM." (VidyaGod: `ComposeGuestTarget` in `launchresolver.cpp`;
the boundary's content tokens are redirected and the trailing args appended in `ContainerWrapper::Execute`.)

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
  pick the Windows build of snes9x → its host is `win32` → the next step offers `win32 → ???` (Proton, Wine, …) → and so
  on until a step reaches the machine platform, where the native terminal is offered.
- A hint shows the target (`→ linux64`) and whether the current chain reaches it.
- The chosen chain is persisted (per package) and honored by the resolver on the next launch.

This is what lets a user say "run this SNES game through the *Windows* emulator under Proton" deliberately, even though
the shortest default would use a native emulator. And it is the same mechanism that will expose ARM routes when ARM
runners exist.

## 11.8 Worked example: the SNES daisy chain (verified end-to-end)

Author an embedded Windows-emulator runner in a SNES game's bundle:

```json
// super_metroid_snes9x_win.json  (a runner embedded in the game's bundle)
{ "NODE_ID": "super_metroid_snes9x_win", "ROLE": "runner",
  "PLATFORM": { "HOST": "win32", "GUEST": ["snes"] },
  "EXEC": { "EXECUTABLE": "snes9x.exe", "ARGS": ["%Content%"], "ENV": {}, "REMOVE_ENV": [] },
  "PARENTS": ["super_metroid_snes9x_win_build"] }

// super_metroid_snes9x_win_build.json  (its build: the win32 binary)
{ "NODE_ID": "super_metroid_snes9x_win_build", "ROLE": "content",
  "LAYERS": [ { "TYPE": "VFSFileLayer", "PATH": "snes9x.exe" } ] }
```

Pin the chain `[super_metroid_snes9x_win, ge-proton10-30]` (the terminal is appended automatically). The runtime:

1. Resolves `[super_metroid_snes9x_win, ge-proton10-30, native-passthrough]`.
2. Mounts the ROM at `pfx/drive_c/299/…sfc`, snes9x.exe at `pfx/drive_c/299/__runner_super_metroid_snes9x_win__/snes9x.exe`.
3. Derives Proton's guest template from its `CONTENT_ROOT` → `C:\299\%REL%`.
4. Composes and execs:
   ```
   proton  waitforexitandrun
       "C:\299\__runner_super_metroid_snes9x_win__\snes9x.exe"
       "C:\299\Super Metroid (Japan, USA) (En,Ja).sfc"
   ```
5. Proton starts, Wine launches snes9x.exe, snes9x loads the ROM. `snes → win32 → linux64`, executed.

This is a real, validated launch in the reference implementation; the composed command above is taken from its logs.

## 11.9 Invariants recap

- **I4 — one process:** only the terminal is `execve`d; inner runners are nested arguments, never separately spawned.
- **I5 — always terminated:** a chain that can't reach the machine platform means the content is unrunnable; the launch
  is refused with a diagnostic, not reported as a clean exit.

Next: [Dependency resolution](12-resolution.md).
