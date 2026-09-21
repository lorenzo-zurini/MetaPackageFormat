# 07 · Persistence

The runtime is **ephemeral by construction**: it is assembled fresh each launch and wiped afterward (invariant I6).
Persistence declares which state escapes that wipe — promoted to a **named durable target** under the launch
**instance** (chapter 16), a directory that survives the runtime wipe and holds the player's saves, configs, and
registry state for that game.

Persistence is **one primitive** — a `DeclarePersist` node — so it composes through the same dependency-chain
hierarchy as everything else (chapter 12). It is **purely additive**: each node promotes one runtime location to one
durable target; there is no exclude axis and no policy flag. The runtime is **pristine by default** — only what a
`DeclarePersist` names survives.

## 7.1 The `DeclarePersist` primitive

```jsonc
{ "LABEL": "quake_saves", "TYPE": "DeclarePersist", "PARENTS": ["quake_content"],
  "SCOPE": "file",                       // "file" (default) or "registry"
  "PATH":  "drive_c/Game/Saves",         // the runtime source to persist
  "TARGET": "Saves",                     // the durable subdir name under the instance
  "CLOUD": true }                        // include in Cloud Saves (default true)
```

| Facet | Meaning |
|-------|---------|
| `SCOPE` | `file` (a runtime path) or `registry` (a Windows/Wine registry key). Default `file`. |
| `PATH` | the runtime source to persist. For `file`, a runtime-root-relative path; for `registry`, a key like `HKCU\Software\…`. **Empty `PATH` persists the whole scope** (the entire runtime, or all hives) — an *authoring aid* that warns, never something to ship. |
| `TARGET` | the durable subdir name (one path segment) under the instance the state maps to. Defaults to `PATH`'s last segment; **required** when `PATH` is empty (there is no leaf to default from). |
| `CLOUD` | whether this target is eligible for Cloud Saves (chapter 16). Default `true`; set `false` for machine-specific state (a shader cache, a GPU-tuned config) that must not sync across machines. |

**One node = one persist.** A game persisting three locations declares three `DeclarePersist` nodes. This is the
deliberate replacement for the earlier plural `KEEP`/`DROP` arrays — a keep is a node, so it composes, carries its own
`WHEN`, and reads in the graph like every other layer.

Because every persist maps to a **named** `TARGET`, the durable store is a set of named sibling directories that never
includes the instance's own config file — so a sandboxed game can never read or tamper the instance metadata beside its
saves. (This is why mapping a `TARGET` is mandatory rather than mirroring the raw `PATH`.)

> **Design note.** MPF's persistence has been through three shapes. (1) Four persistence types with an **inverted
> default**: a closure that declared nothing persisted the *entire* prefix, with no way to exclude — bloated storage,
> contradicted I6. (2) One additive `Persist` node with plural `KEEP`/`DROP` arrays and a self-describing
> `KEEP %RuntimePath%` for whole-runtime persistence — better, but the whole-runtime keep pointed the writable branch
> straight at the durable store, dragging the instance config into a game-writable mount, and `DROP` was redundant once
> the default was pristine. (3) The **current** model: one node = one persist, each mapped to a *named* durable target,
> no `DROP`, no whole-runtime flag, plus a `CLOUD` attribute for Cloud Saves. Simpler, safer, and forward-compatible.

## 7.2 What each persist resolves to

The dir/file/registry *kind* is derived from `SCOPE` + `PATH`'s shape — there is no separate type to pick:

| `SCOPE` | `PATH` shape | Resolves to |
|---------|--------------|-------------|
| `file` | a **directory** (trailing `/`, or a final component with no extension) | a **live** durable RW passthrough |
| `file` | a **file** (a final component with an extension) | a copy-in / copy-out durable file |
| `file` | **empty** | the **whole runtime**, TARGET-mapped (authoring aid — warns) |
| `registry` | a key — `HKCU`, `HKCU\Software\…`, … | that key's subtree (a bare hive root is just the broadest key) |
| `registry` | **empty** | all prefix hives (`user`/`system`/`userdef.reg`) (authoring aid — warns) |

Registry roots map to Wine hive files (`HKCU`→`user.reg`, `HKLM`/`HKCR`/`HKCC`→`system.reg`, `HKU`→`userdef.reg`);
registry persistence is **Wine-family only** (no hives elsewhere).

- A **directory** persist is a read-write passthrough to `<instance>/<TARGET>`, **unioned** over the lower layers — the
  prefix skeleton stays visible while the game's writes land in (and persist to) the durable target. Live; no copy.
- A **file** persist is seeded before mount (the durable copy shadows lower layers) and captured after the session.
- A **registry subtree** persist merges just that key's subtree to a partial-hive store, accumulating across sessions;
  a key the session never created is kept as-is rather than dropped.
- A **whole-scope** persist (empty `PATH`) is an authoring aid for *discovering* what to keep — it persists everything
  so you can inspect what changed, then narrow to specific dirs/files/keys before shipping. Implementations warn on it.

## 7.3 Runner keep-sets — saves survive with zero per-game work

Because the runtime is pristine by default, *something* must declare where the standard user-state lives. That something
is the **runner**: a runner knows its own platform's layout, with prefix-correct paths, so it ships default
`DeclarePersist` nodes that are folded into **every** launch alongside the game's own persists. The Wine/Proton runners
keep the user-profile tree and the user hive:

```jsonc
// for the Proton runner (CONTENT_ROOT "pfx/drive_c/…")
{ "LABEL": "proton_keep_users", "TYPE": "DeclarePersist", "SCOPE": "file", "PATH": "pfx/drive_c/users", "TARGET": "users" }
{ "LABEL": "proton_keep_hkcu",  "TYPE": "DeclarePersist", "SCOPE": "registry", "PATH": "HKCU" }
```

So by default a typical game saves to `…/users/<user>/Documents`, `Saved Games`, `AppData`, and the registry — **all
kept** by the runner — while the rest of the prefix (system files, caches, the DXVK shader cache) regenerates pristine
each launch. A game adds a `DeclarePersist` only for a **non-standard** save path (e.g. a save folder under the install
dir). A runner is an ordinary package: its keep-set is just `DeclarePersist` nodes on its chain, with no special
casing — a runner has every option a game does, and should carry a persist only when there is a real reason to.

## 7.4 Mechanics — the overlay, no new filesystem

`DeclarePersist` reuses the single overlay mount (chapter 13). The writable branch is **always** an ephemeral scratch
layer; persisted state is never the writable branch itself (that is what keeps the instance config out of a
game-writable mount). Each directory persist adds a **durable RW passthrough** unioned at its `PATH`, backed by
`<instance>/<TARGET>`; each file/registry persist is copy-seeded before mount and captured after. An empty-`PATH`
directory persist passes its `TARGET` dir through at the runtime **root** (target `""`) — a durable RW layer over the
ephemeral write branch, not a replacement for it.

Reproducibility and garbage-collection fall out of the pristine default: only named targets accrete durable state, so
there is no unbounded growth, and a launch is reproducible from the content + the keep-set. Persistence being purely
additive, **resolution order is immaterial** — the runner keep-set and the game's persists simply union.

(VidyaGod: `DerivePersistence` classifies the persists; `BuildLayerSpec` emits the ephemeral write branch + the durable
passthrough layers; `persistlayer.cpp`/`registrylayer.cpp` seed/capture the copy-based persists.)

## 7.5 Where persisted state lives

```
<instance>/                      # the per-game launch instance (chapter 16)
├── instance.json                # instance config — NEVER inside a game-writable mount
├── <TARGET dir…>                # live RW passthrough trees (e.g. "users", "Saves")
├── <TARGET file…>               # captured single files
├── REGISTRY/{user,system,userdef}.reg   # whole-hive persists (only the kept hives)
└── REGKEYS/                     # registry-subtree partial hives
```

Each `DeclarePersist` maps to a **named** `TARGET` directly under the instance, a sibling of `instance.json` — never the
instance root itself. Implementations MUST place persisted state here (or a per-launch override of it — chapter 16),
never in the ephemeral runtime tree, and MUST keep the instance's own metadata out of any mount exposed to the game.

## 7.6 Cloud Saves (forward-looking)

`CLOUD` marks which targets are eligible to sync across a player's machines. Machine-specific state — a shader cache, a
resolution/GPU config tuned to one box — sets `CLOUD: false` so it stays local while real saves and settings sync. The
attribute is declared now and honored by the sync layer when it lands; a target with `CLOUD: true` (the default) is a
cloud-syncable save.

## 7.7 Lifecycle summary

| Phase | directory persist | file / registry persist |
|-------|-------------------|--------------------------|
| Writable branch | ephemeral (wiped after) | ephemeral (wiped after) |
| Before mount | union the durable `<instance>/<TARGET>` dir at `PATH` | seed the file/hive from `<instance>/<TARGET>` (or `REGISTRY`/`REGKEYS`) |
| During session | writes are live-durable to the target | writes hit the ephemeral runtime |
| At teardown | nothing to capture (already durable) | capture the file/hive back to the durable store, before unmount |
| Save-safety | durable-backed → non-lazy verified unmount | durable-backed → non-lazy verified unmount |

Save-safety (invariant I7) is unchanged and detailed in [chapter 13 §13.6](13-runtime-model.md).

## 7.8 Validation (lint)

A validator (chapter 15) checks the structural mistakes the model still allows: a `DeclarePersist` with an unknown
`SCOPE` (**warning**); and an **empty `PATH`** (a whole-scope catch-all — **warning**, since nothing should ship one).
A whole-scope `file` persist with no `TARGET` is refused outright (there is no leaf to name the durable dir).

Next: [Variables & CustomVar](08-variables.md).
