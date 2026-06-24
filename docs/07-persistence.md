# 07 · Persistence

The runtime is **ephemeral by construction**: it is assembled fresh each launch and wiped afterward (invariant I6).
Persistence declares which state escapes that wipe — written to the package's **`USERDATA`** store, which lives beside
the bundle and travels with it (`<bundle>/USERDATA`), surviving even a full wipe of the implementation's data root.

Persistence is **one primitive** — a `Persist` layer with three facets — that lives in `LAYERS`, so it composes through
the same dependency-chain hierarchy as everything else (chapter 12).

## 7.1 The `Persist` primitive

```jsonc
"LAYERS": [
  { "TYPE": "Persist", "MODE": "all" },                          // base policy (default "none")
  { "TYPE": "Persist", "KEEP": "drive_c/Game/Saves" },           // persist a target (durable)
  { "TYPE": "Persist", "KEEP": "HKCU\\Software\\id Software\\Quake" },
  { "TYPE": "Persist", "DROP": "drive_c/Game/cache" }            // make a path ephemeral (writes discarded)
]
```

| Facet | Meaning |
|-------|---------|
| `MODE` | the base policy: `none` (a **pristine** runtime — only `KEEP` targets persist) or `all` (the **whole** runtime is durable). **Default `none`.** Resolves **last-wins** along the dependency chain — the most-specific node wins (§7.5). |
| `KEEP` | persist this target. Subsumes the old `PersistDir`/`PersistFile`/`RegPersist`/`RegKeyPersist` — its kind is **derived** from the target's shape (§7.2). |
| `DROP` | make this runtime path **ephemeral** — its writes go nowhere durable. The exclude axis: carve a throwaway hole out of `MODE:all` (or out of an enclosing `KEEP`). Paths only. |

A single `Persist` layer typically carries one facet; an author MAY combine them (a `MODE` and several `KEEP`/`DROP`).

> **Design note.** Earlier MPF had four persistence types and an **inverted default**: a closure that declared *nothing*
> persisted the *entire* prefix (saves, configs, registry, **and** caches/system files), with no way to exclude. That
> bloated `USERDATA` and contradicted invariant I6's "ephemeral runtime, durable saves". The current model inverts the
> default to **pristine** and folds the four types into one self-describing `KEEP`, plus a `DROP` exclude axis.

## 7.2 Self-describing `KEEP` targets

The dir/file/registry *kind* of a `KEEP` is **derived** from the target string — there is no separate type to pick:

| Target shape | Resolves to | Was |
|--------------|-------------|-----|
| `registry` (sentinel) | all prefix hives (`user`/`system`/`userdef.reg`) | `RegPersist` |
| a hive root — `HKCU`, `HKLM`, `HKU`, … | that whole hive's `.reg` file | `RegPersist` (scoped) |
| a deeper registry path — `HKCU\Software\…` | just that key's subtree | `RegKeyPersist` |
| a runtime-relative **directory** | a **live** durable RW passthrough | `PersistDir` |
| a runtime-relative **file** | a copy-in / copy-out durable file | `PersistFile` |

A path target is a **directory** when it ends in `/`, already exists as a directory, or its final component has no
extension; otherwise it is a **single file**. Registry roots map to Wine hive files (`HKCU`→`user.reg`,
`HKLM`/`HKCR`/`HKCC`→`system.reg`, `HKU`→`userdef.reg`); registry KEEPs are **Wine-family only** (no hives elsewhere).

- A **directory** `KEEP` is a read-write passthrough to `USERDATA/<rel>`, **unioned** over the lower layers — the prefix
  skeleton stays visible while the game's writes land in (and persist to) `USERDATA`. Live; no copy.
- A **file** `KEEP` is seeded before mount (the durable copy shadows lower layers) and captured after the session.
- A **registry hive** `KEEP` copies the whole `.reg` file in/out of `USERDATA/__REGISTRY__/`.
- A **registry subtree** `KEEP` merges just that key's subtree to a partial-hive store `USERDATA/__REGKEYS__/`,
  accumulating across sessions; a key the session never created is kept as-is rather than dropped.

## 7.3 Runner keep-sets — saves survive with zero per-game work

Because the default is pristine, *something* must declare where the standard user-state lives. That something is the
**runner**: a runner knows its own platform's layout, with prefix-correct paths, so it ships a default keep-set that is
folded into **every** launch ahead of the game's own `Persist` layers. The Wine/Proton runners keep the user-profile
tree and the user hive:

```jsonc
// on the Proton runner node (CONTENT_ROOT "pfx/drive_c/…")
"LAYERS": [ { "TYPE":"Persist", "KEEP":"pfx/drive_c/users" }, { "TYPE":"Persist", "KEEP":"HKCU" } ]
// on the Wine runner node  (CONTENT_ROOT "drive_c/…")
"LAYERS": [ { "TYPE":"Persist", "KEEP":"drive_c/users" },     { "TYPE":"Persist", "KEEP":"HKCU" } ]
```

So under `MODE:none` a typical game saves to `…/users/<user>/Documents`, `Saved Games`, `AppData`, and the registry —
**all kept** by the runner — while the rest of the prefix (system files, caches, the DXVK shader cache) regenerates
pristine each launch. A game adds a `KEEP` only for a **non-standard** save path (e.g. a save folder under the install
dir) or a `DROP` to trim. (`Persist` layers are otherwise the one kind of layer read off a runner *node* directly —
their build comes from the runner's `PARENTS`, but a keep-set is policy, not content.)

## 7.4 Mechanics — the overlay, no new filesystem

`Persist` reuses the single overlay mount (chapter 13). `MODE` picks the writable branch — `all` → the durable
`USERDATA`; `none` → an ephemeral scratch layer. Each `KEEP` dir adds a **durable RW passthrough** unioned at its target;
each `KEEP` file/registry is copy-seeded/captured. Each `DROP` adds an **ephemeral RW shadow** at its target, appended
*above* everything (so it wins) — its writes hit a per-launch scratch dir and vanish. `DROP` therefore nests correctly
under both modes: a hole in the durable `MODE:all` runtime, or in an enclosing `KEEP` dir under `MODE:none`.

Reproducibility and garbage-collection fall out: `MODE:none` ⇒ a clean prefix every launch (only `KEEP`s accrete) — no
unbounded `USERDATA` growth, and a launch is reproducible from the content + the keep-set.

(VidyaGod: `DerivePersistence` classifies the targets; `BuildLayerSpec` emits the branch + KEEP/DROP layers;
`persistlayer.cpp`/`registrylayer.cpp` seed/capture the copy-based KEEPs.)

## 7.5 Override via the dependency chain

`MODE` resolves like any chained value: the runner keep-set is scanned first, then the game's closure (ancestors first,
the launchable last), and the **last** `MODE` wins — so a game can force `MODE:all` over the runner's implicit `none`,
and a more-specific node overrides a parent. `KEEP`/`DROP` are **unioned** (order-independent, additive). A game that
depends on a shared package inherits that package's keeps and adds its own.

## 7.6 Where persisted state lives

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

## 7.7 Lifecycle summary

| Phase | `MODE:all` | `MODE:none` (default) |
|-------|-----------|------------------------|
| Writable branch | **is** `USERDATA` (durable) | ephemeral (wiped after) |
| Before mount | — | seed `KEEP` file/registry from `USERDATA`; union the `KEEP` dirs; stack the `DROP` shadows |
| During session | everything persists live | only `KEEP` dirs are live-durable; `DROP` writes vanish |
| At teardown | capture nothing (already durable) | capture `KEEP` file/registry back to `USERDATA`, before unmount |
| Save-safety | the union mount is durable-backed → non-lazy verified unmount | any `KEEP` dir is durable-backed → non-lazy verified unmount |

Save-safety (invariant I7) is unchanged and detailed in [chapter 13 §13.6](13-runtime-model.md).

## 7.8 Validation (lint)

A validator (chapter 15) checks the structural mistakes the self-describing model still allows: a `MODE` that is neither
`all` nor `none` (**error**); a `Persist` layer with no `MODE`/`KEEP`/`DROP` (a no-op — **warning**); a `DROP` aimed at
the registry (paths only — **warning**); and a `host:` target (**warning** — reserved, see §7.9).

## 7.9 Out of scope: native `$HOME` (future)

A `host:<path>` target is **reserved but not implemented**. Native (uncontained) applications write to the real `$HOME`,
so there is no overlay to keep their state in. Capturing it will require sandboxing the native process's home into the
overlay (a **bubblewrap** container) so it persists like a Wine prefix — a future addition. Until then, native apps
persist to `$HOME` directly, exactly as they do today, and `Persist` governs only the overlay (Wine/Proton/emulator)
runtimes.

Next: [Variables & CustomVar](08-variables.md).
