# 13 · The runtime model

This chapter specifies how the resolved closure, the runner build and the prefix are assembled into **one overlay mount**
that the runner executes against, and how the session is torn down save-safely. This is where layers, persistence,
runners and prefixes meet the filesystem.

## 13.1 One overlay, many layers

The entire runtime is a **single overlay/union filesystem** mounted at the **runtime path** (`%RuntimePath%`). A
Wine-family runner's prefix env (`STEAM_COMPAT_DATA_PATH` / `WINEPREFIX`) points at this one mount; an emulator runs from
it; the content lives inside it at the content root. There is no extraction and no second copy: content is mounted by
reference (zips served zero-copy by offset, dirs/files passed through).

The overlay is built from an ordered list of layers, **lowest priority first**:

```
┌──────────────────────────────────────────────────────────────┐  highest priority (top)
│  OVERRIDE edits          (applied post-mount, COW → writable)  │   ← chapter 6 §6.1
│  WRITABLE layer          (always ephemeral)                    │   ← general COW top; session writes land here
│  persist passthrough dirs (RW, live durable → <instance>/<TARGET>)│ ← chapter 7 (RW for their own subtree)
│  DEFAULT-DATA layer      (base edits: FileEdit/RegEdit defaults)│   ← chapter 6
│  INNER-RUNNER builds      (cross-namespace, @ CONTENT_ROOT/__runner_…__)│ ← chapter 11 §11.6
│  CONTENT layers          (the game's Content nodes, @ CONTENT_ROOT)│   ← chapters 5,12 (closure order within)
│  DEFPREFIX base          (the generated Wine prefix tree)       │   ← §13.4 (only if PREFIX_GENERATE)
└──────────────────────────────────────────────────────────────┘  lowest priority (bottom)
```

The `layers` array is stacked lowest→highest in the order above; the **writable layer** is a distinct top branch and is
**always ephemeral** (a persist passthrough is read-write for *its own* subtree, backed by the durable instance target;
the writable layer is the catch-all scratch for everything else and is wiped after the session — which keeps the
instance's own config out of any game-writable mount); OVERRIDE edits are written into the writable layer *after*
mounting so they win unconditionally. Ordering among
*non-conflicting* layers (e.g. content vs. an inner-runner build in a separate `__runner_…__` subdir) is immaterial.

Notes:

- **DEFPREFIX at the bottom**: the prefix's `drive_c`/`pfx` structure provides the base filesystem the content is laid
  *into*; content layers at `CONTENT_ROOT` (= `pfx/drive_c/<uid>`) sit above it. The runner's **prefix-assembly**
  content ([ch. 10 §10.5](10-platforms-and-runners.md)) belongs to this base and is therefore **prepended** with it,
  never appended: it includes the `system32`/`syswow64` builtin DLLs, and putting those on top silently shadows every
  native override a package ships at the same path.
- **Content layers** are ordered by the resolved closure (chapter 12): later closure node = higher; the launchable's own
  layers highest among content.
- **DEFAULT-DATA above content**: base edits override the package's shipped files…
- **WRITABLE layer above DEFAULT-DATA**: …but the user's live writes shadow the base edits ("factory defaults under live
  user data").
- **OVERRIDE edits last**: forced values win over everything, written post-mount straight into the writable layer.

(VidyaGod: `VfsMount::BuildLayerSpec` produces this exact stack; the separate runner-build mount is `MountRunnerBuild`.)

## 13.2 The session directory tree

Everything ephemeral for one launch lives under a per-session temp root (in the OS temp dir — it is pure scratch and
never needs to persist, so it stays off the data root entirely), so a single delete cleans up:

```
<tmp>/VidyaGod/<PackageUID>/<instance>/
├── RUNTIME       ← the overlay mount point (%RuntimePath%)
├── RUNNER        ← the boundary runner's build mount (%RunnerMount%), if separate
├── WRITELAYER    ← ephemeral writable branch (%WriteLayerPath%), always
├── DEFAULTDATA   ← base-edit layer (%DefaultData%), regenerated each launch
└── DEFPREFIX     ← per-launch generated prefix (%DefPrefixPath%), if not an installed artifact
```

Durable saves live **outside** this tree, in the per-game **instance** directory (chapter 7/16) as named `TARGET`
subdirs, so wiping the temp tree never touches them. The data root, runtime path and instance/userdata path can each be
relocated per launch (chapter 16) for portable/in-package runs.

## 13.3 Content placement & the prefix root

- **Content root** (`CONTENT_ROOT`, runner-declared): the runtime-relative directory content mounts under. `%ProgramPath%`
  = `RuntimePath / CONTENT_ROOT`; the launch target is `%ProgramPath% / ` the launchable's `PATH`.
- **Prefix root**: derived as the part of `CONTENT_ROOT` *before* `drive_c` — `""` for plain Wine (hives at the root),
  `"pfx"` for Proton (`pfx/drive_c/…`). The three registry hive files (`system.reg`, `user.reg`, `userdef.reg`) live at
  `<root>/<prefixRoot>`. The implementation uses the prefix root to find hives when building default-data and capturing
  persistence.

## 13.4 Prefix generation (Wine-family)

A `PREFIX_GENERATE` runner needs a prefix before content can stack on it. The prefix is created **once** and reused:

- **Generation**: run the runner's own launcher with a `wineboot` *override target* (chapter 9 §9.3) and the prefix env
  pointed at the prefix directory being built. This produces a pristine `drive_c` + hives. (VidyaGod:
  `RegistryLayer::InitializeDefPrefix`.)
- **Reuse**: the generated prefix is stored as a read-only artifact alongside the runner (`<runnerBundle>/__DEFPREFIX__/
  default`) and mounted whole as the overlay's base layer every launch. It is **never mutated** — per-session registry
  and `drive_c` writes go to the writable layer above it (invariant I6).
- **Build-shipping runners** generate their prefix at install time (so launches don't pay for `wineboot`); a runner with
  no shipped build generates a throwaway prefix per launch under `TEMP/.../DEFPREFIX`.

A non-Wine runner (`PREFIX_GENERATE: false`) has no prefix and no base layer here.

## 13.5 Default-data layer

Base (`OVERRIDE: false`) edits are materialized into the **DEFAULT-DATA** layer, regenerated each launch and mounted
between content and the writable layer:

- **Base `FileEdit`s** → files written at their runtime-relative paths under `DEFAULTDATA` (any runner type).
- **Base `RegEdit`s** → full `user/system/userdef.reg` built by loading the pristine prefix's hives, applying the edits,
  and writing the result into `DEFAULTDATA` (Wine only; the prefix itself is never touched). Wine COWs the whole hive
  from this layer when it writes the registry.
- **Persisted registry-subtrees** (`DeclarePersist` `SCOPE:registry`) are merged into those hives *after* the base
  edits, so saved user keys win over package defaults.

A package with no base edits gets no (empty) default-data layer. (VidyaGod: `RegistryLayer::BuildDefaultData`.)

## 13.6 Teardown & save-safety (invariant I7)

After the process exits, the session is dismantled in a **save-safe** order:

1. **Capture** declared persistence *while the runtime is still mounted*: persisted hives, files, and
   registry-subtrees → the instance's durable `TARGET`s (chapter 7). (Directory persists are live passthroughs —
   already durable.)
2. **Unmount durable-backed mounts non-lazily and verify**: any mount that exposes a durable instance `TARGET` through
   it (any directory-persist passthrough) is unmounted with a blocking unmount and retried; the implementation confirms
   it is gone from the mount table before proceeding.
3. **Lazy-unmount purely-ephemeral mounts** (content/runner mounts whose backing is all under `TEMP`).
4. **Wipe `TEMP` only if every durable mount detached cleanly.** If any durable-backed mount is still live, the wipe is
   **aborted** and `TEMP` is left in place — because a recursive delete could otherwise traverse a live passthrough into
   real `USERDATA` and destroy saves. Leaving `TEMP` behind is the safe failure mode.

This ordering is mandatory: an implementation MUST NOT delete an ephemeral tree while a mount that reaches durable user
data could still be traversed. (VidyaGod: `ContainerWrapper::Cleanup`; the same gate guards pre-launch stale-runtime
cleanup in `CleanStaleRuntime`.)

## 13.7 Stale-runtime recovery

Because the runtime is a real mount and a process can crash, an implementation MUST handle leftovers: before assembling a
new runtime, detect any mounts still present under this package's `TEMP`, unmount them (lazily, then verify cleared with
the same save-safety gate), and only then clear the stale tree. A single-instance guard (chapter 16) lets the
implementation assume any mount still present at startup is crash debris. (VidyaGod: `CleanStaleRuntime`,
`MountpointsUnder`.)

## 13.8 What an implementation must provide

To realize this chapter, an implementation needs an **overlay filesystem** that can: mount STORE-zip entries zero-copy,
mount directories and single files, stack layers by priority, expose a writable top branch, and support read-write
passthrough sub-mounts for persistence. The reference implementation ships a purpose-built FUSE filesystem
("vidyagodfs") that does exactly this from a JSON layer-spec; any overlay with equivalent semantics conforms. The format
does not mandate FUSE — only the *layering semantics* above.

Next: [Content addressing & distribution](14-content-addressing.md).
