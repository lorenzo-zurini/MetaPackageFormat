# 07 · Persistence

The runtime is **ephemeral by construction**: it is assembled fresh each launch and wiped afterward (invariant I6).
Persistence declares which state escapes that wipe — written to the package's **`USERDATA`** store, which lives beside
the bundle and travels with it (`<bundle>/USERDATA`), surviving even a full wipe of the implementation's data root.

Persistence is **one primitive** — a `Persist` layer — that lives in `LAYERS`, so it composes through the same
dependency-chain hierarchy as everything else (chapter 12). It is **purely additive**: `KEEP` adds durable state, `DROP`
removes it. There is no mode or policy flag — the runtime is pristine by default, and "keep everything" is simply a
`KEEP` of the runtime root.

## 7.1 The `Persist` primitive

```jsonc
"LAYERS": [
  { "TYPE": "Persist", "KEEP": "drive_c/Game/Saves" },           // persist a target (durable)
  { "TYPE": "Persist", "KEEP": "HKCU\\Software\\id Software\\Quake" },
  { "TYPE": "Persist", "DROP": "drive_c/Game/cache" }            // make a path ephemeral (writes discarded)
]
```

| Facet | Meaning |
|-------|---------|
| `KEEP` | persist this target. Subsumes the old `PersistDir`/`PersistFile`/`RegPersist`/`RegKeyPersist` — its kind is **derived** from the target's shape (§7.2). A `KEEP` of the runtime root (`%RuntimePath%`) persists the *whole* runtime. |
| `DROP` | make this runtime path **ephemeral** — its writes go nowhere durable. The exclude axis: carve a throwaway hole out of a kept tree. Paths only. |

A single `Persist` layer typically carries one facet; an author MAY combine `KEEP` and `DROP` on one layer.

> **Design note.** Earlier MPF had four persistence types and an **inverted default**: a closure that declared
> *nothing* persisted the *entire* prefix (saves, configs, registry, **and** caches/system files), with no way to
> exclude. That bloated `USERDATA` and contradicted invariant I6's "ephemeral runtime, durable saves". An interim
> redesign added an explicit `MODE: all|none` policy — but a mode is redundant when the root is itself a valid `KEEP`
> target. The current model drops the mode entirely: persistence is **purely additive** (default pristine; `KEEP`
> adds, `DROP` removes), and `KEEP %RuntimePath%` *is* whole-runtime persistence.

## 7.2 Self-describing `KEEP` targets

The dir/file/registry/whole-runtime *kind* of a `KEEP` is **derived** from the target string — there is no separate type
to pick:

| Target shape | Resolves to | Was |
|--------------|-------------|-----|
| `%RuntimePath%` (the mount root) | the **whole runtime** is durable (the writable branch *is* `USERDATA`) | `MODE:all` / old default |
| `registry` (sentinel) | all prefix hives (`user`/`system`/`userdef.reg`) | `RegPersist` |
| a hive root — `HKCU`, `HKLM`, `HKU`, … | that whole hive's `.reg` file | `RegPersist` (scoped) |
| a deeper registry path — `HKCU\Software\…` | just that key's subtree | `RegKeyPersist` |
| a runtime-relative **directory** | a **live** durable RW passthrough | `PersistDir` |
| a runtime-relative **file** | a copy-in / copy-out durable file | `PersistFile` |

A path target is a **directory** when it ends in `/`, already exists as a directory, or its final component has no
extension; otherwise it is a **single file**. Registry roots map to Wine hive files (`HKCU`→`user.reg`,
`HKLM`/`HKCR`/`HKCC`→`system.reg`, `HKU`→`userdef.reg`); registry KEEPs are **Wine-family only** (no hives elsewhere).

- A **whole-runtime** `KEEP` (`%RuntimePath%`) makes the durable `USERDATA` the overlay's writable branch — everything
  the session writes persists. The maximal, install-like behavior; use it sparingly (it forgoes reproducibility).
- A **directory** `KEEP` is a read-write passthrough to `USERDATA/<rel>`, **unioned** over the lower layers — the prefix
  skeleton stays visible while the game's writes land in (and persist to) `USERDATA`. Live; no copy.
- A **file** `KEEP` is seeded before mount (the durable copy shadows lower layers) and captured after the session.
- A **registry hive** `KEEP` copies the whole `.reg` file in/out of `USERDATA/__REGISTRY__/`.
- A **registry subtree** `KEEP` merges just that key's subtree to a partial-hive store `USERDATA/__REGKEYS__/`,
  accumulating across sessions; a key the session never created is kept as-is rather than dropped.

## 7.3 Runner keep-sets — saves survive with zero per-game work

Because the runtime is pristine by default, *something* must declare where the standard user-state lives. That something
is the **runner**: a runner knows its own platform's layout, with prefix-correct paths, so it ships a default keep-set
that is folded into **every** launch alongside the game's own `Persist` layers. The Wine/Proton runners keep the
user-profile tree and the user hive:

```jsonc
// on the Proton runner node (CONTENT_ROOT "pfx/drive_c/…")
"LAYERS": [ { "TYPE":"Persist", "KEEP":"pfx/drive_c/users" }, { "TYPE":"Persist", "KEEP":"HKCU" } ]
// on the Wine runner node  (CONTENT_ROOT "drive_c/…")
"LAYERS": [ { "TYPE":"Persist", "KEEP":"drive_c/users" },     { "TYPE":"Persist", "KEEP":"HKCU" } ]
```

So by default a typical game saves to `…/users/<user>/Documents`, `Saved Games`, `AppData`, and the registry — **all
kept** by the runner — while the rest of the prefix (system files, caches, the DXVK shader cache) regenerates pristine
each launch. A game adds a `KEEP` only for a **non-standard** save path (e.g. a save folder under the install dir) or a
`DROP` to trim. (`Persist` layers are otherwise the one kind of layer read off a runner *node* directly — its build
comes from the runner's `PARENTS`, but a keep-set is policy, not content.)

## 7.4 Mechanics — the overlay, no new filesystem

`Persist` reuses the single overlay mount (chapter 13). A `KEEP %RuntimePath%` makes the writable branch the durable
`USERDATA`; otherwise the writable branch is an ephemeral scratch layer. Each `KEEP` dir adds a **durable RW
passthrough** unioned at its target; each `KEEP` file/registry is copy-seeded/captured. Each `DROP` adds an **ephemeral
RW shadow** at its target, appended *above* everything (so it wins) — its writes hit a per-launch scratch dir and vanish.
A `DROP` therefore nests correctly inside either a whole-runtime keep or a `KEEP` dir.

Reproducibility and garbage-collection fall out of the pristine default: only `KEEP`s accrete in `USERDATA`, so there is
no unbounded growth, and a launch is reproducible from the content + the keep-set. Persistence being purely additive,
**resolution order is immaterial** — the runner keep-set and the game's `Persist` layers simply union.

(VidyaGod: `DerivePersistence` classifies the targets; `BuildLayerSpec` emits the branch + KEEP/DROP layers;
`persistlayer.cpp`/`registrylayer.cpp` seed/capture the copy-based KEEPs.)

## 7.5 Where persisted state lives

```
<bundle>/USERDATA/
├── <KEEP dir paths…>             # live RW passthrough trees (e.g. drive_c/users)
├── <KEEP file paths…>            # captured single files
├── __REGISTRY__/{user,system,userdef}.reg   # KEEP hive targets (only the kept hives)
└── __REGKEYS__/                  # KEEP registry-subtree partial hives
```

`USERDATA` is the deliberate exception to "everything under the data root": it sits with the package bundle so saves
travel when the bundle is copied, and survive deleting the implementation's data root. Implementations MUST place
persisted state here (or a per-launch override of it — chapter 16), never in the ephemeral runtime tree.

## 7.6 Lifecycle summary

| Phase | `KEEP %RuntimePath%` (whole runtime) | default (selective KEEPs) |
|-------|--------------------------------------|----------------------------|
| Writable branch | **is** `USERDATA` (durable) | ephemeral (wiped after) |
| Before mount | — | seed `KEEP` file/registry from `USERDATA`; union the `KEEP` dirs; stack the `DROP` shadows |
| During session | everything persists live | only `KEEP` dirs are live-durable; `DROP` writes vanish |
| At teardown | capture nothing (already durable) | capture `KEEP` file/registry back to `USERDATA`, before unmount |
| Save-safety | the union mount is durable-backed → non-lazy verified unmount | any `KEEP` dir is durable-backed → non-lazy verified unmount |

Save-safety (invariant I7) is unchanged and detailed in [chapter 13 §13.6](13-runtime-model.md).

## 7.7 Validation (lint)

A validator (chapter 15) checks the structural mistakes the self-describing model still allows: a `Persist` layer with
no `KEEP`/`DROP` (a no-op — **warning**); a `DROP` aimed at the registry (paths only — **warning**); and a `host:`
target (**warning** — reserved, see §7.8).

## 7.8 Out of scope: native `$HOME` (future)

A `host:<path>` target is **reserved but not implemented**. Native (uncontained) applications write to the real `$HOME`,
so there is no overlay to keep their state in. Capturing it will require sandboxing the native process's home into the
overlay (a **bubblewrap** container) so it persists like a Wine prefix — a future addition. Until then, native apps
persist to `$HOME` directly, exactly as they do today, and `Persist` governs only the overlay (Wine/Proton/emulator)
runtimes.

Next: [Variables & CustomVar](08-variables.md).
