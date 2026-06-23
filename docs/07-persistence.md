# 07 · Persistence layers

The runtime is **ephemeral by construction**: it is assembled fresh each launch and wiped afterward (invariant I6).
Persistence layers declare which state escapes that wipe — written to the package's **`USERDATA`** store, which lives
beside the bundle and travels with it (`<bundle>/USERDATA`), surviving even a full wipe of the implementation's data root.

There are four persistence layer types, plus one default behavior.

## 7.1 The default: whole-runtime persistence (`PersistAll`)

If a launchable's resolved closure declares **no** persistence layer at all, the runtime uses **whole-runtime
persistence**: the writable top branch of the overlay *is* the durable `USERDATA` directory. Everything the session
writes — saves, configs, registry, caches — persists, and the next launch resumes on top of it. This is the
zero-configuration default and is correct for most software (it behaves like a normal install with a home directory).

The trade-off: everything persists, including throwaway state. Declaring *any* persistence layer switches the closure to
**selective persistence**: the writable branch becomes *ephemeral* (wiped each session) and **only the explicitly
declared targets** are carried in and out of `USERDATA`. Use selective persistence when you want a clean runtime each
launch but must preserve specific saves/config.

(VidyaGod: `DerivePersistence` sets `PersistAll = (no Persist* declared)`.)

## 7.2 `PersistDir` — a live durable directory

| Field | Meaning |
|-------|---------|
| `TYPE` | `"PersistDir"` |
| `PATH` | a directory path, relative to the runtime root, that should be durable. `%TOKEN%`-expanded. |

The named directory is a **read-write passthrough** to `USERDATA/<PATH>`: the game reads and writes it *live* during the
session (no copy-in/copy-out), and its contents persist. Use it for save directories that the game writes continuously.

```json
{ "TYPE": "PersistDir", "PATH": "drive_c/users/steamuser/Documents/My Games/Skyrim/Saves" }
```

Because a `PersistDir` exposes real durable data *through the live mount*, it imposes save-safety obligations on teardown
(it MUST be unmounted non-lazily and verified gone before any ephemeral wipe — invariant I7, chapter 13).

## 7.3 `PersistFile` — a durable single file

| Field | Meaning |
|-------|---------|
| `TYPE` | `"PersistFile"` |
| `PATH` | a single file path, relative to the runtime root. `%TOKEN%`-expanded. |

A `PersistFile` is **seeded and captured by copy** (not a live passthrough): before the mount, the durable copy at
`USERDATA/<PATH>` (if any) is copied into the ephemeral writable layer so it shadows lower layers; after the session, the
runtime's `<PATH>` is copied back to `USERDATA/<PATH>`. Use it for individual config/save files. Copy semantics avoid
bind-mounting single files and mirror the registry model (§7.4).

```json
{ "TYPE": "PersistFile", "PATH": "drive_c/Game/settings.cfg" }
```

## 7.4 `RegPersist` — the whole Wine registry (Wine-family only)

| Field | Meaning |
|-------|---------|
| `TYPE` | `"RegPersist"` |

Persists the three Wine hive files (`system.reg`, `user.reg`, `userdef.reg`) wholesale to `USERDATA/__REGISTRY__/`. Wine
always rewrites the complete hive file, so persisting whole files is correct. Seeded before mount (shadowing the pristine
prefix) and captured at teardown. No-op for non-Wine runners (no hives) and under `PersistAll`.

```json
{ "TYPE": "RegPersist" }
```

## 7.5 `RegKeyPersist` — a single registry subtree (Wine-family only)

| Field | Meaning |
|-------|---------|
| `TYPE` | `"RegKeyPersist"` |
| `REGPATH` | the registry key whose subtree should persist, e.g. `HKCU\\Software\\Vendor\\App`. `%TOKEN%`-expanded. |

Persists *only* the named key's subtree (not the whole registry) to a partial-hive store at `USERDATA/__REGKEYS__/`,
accumulating across sessions. At launch, the persisted subtree is merged into the default-data hives *after* base
`RegEdit`s, so the user's saved key state wins over package defaults. At teardown, the subtree is extracted from the
session hives and merged into the durable store; a key the session never created is kept as-is rather than dropped.
No-op for non-Wine runners and under `PersistAll`.

```json
{ "TYPE": "RegKeyPersist", "REGPATH": "HKCU\\Software\\id Software\\Quake\\1.0" }
```

(VidyaGod: `SeedPersistRegistry`/`CapturePersistRegistry` and `CapturePersistRegKeys` in `registrylayer.cpp`;
`SeedPersistFiles`/`CapturePersistFiles` in `persistlayer.cpp`.)

## 7.6 Where persisted state lives

```
<bundle>/USERDATA/
├── <PersistDir paths…>            # live RW passthrough trees
├── <PersistFile paths…>          # captured single files
├── __REGISTRY__/{system,user,userdef}.reg   # RegPersist
└── __REGKEYS__/                   # RegKeyPersist partial hives
```

`USERDATA` is the deliberate exception to "everything under the data root": it sits with the package bundle so saves
travel when the bundle is copied, and survive deleting the implementation's data root. Implementations MUST place
persisted state here (or a per-launch override of it — chapter 16), never in the ephemeral runtime tree.

## 7.7 Lifecycle summary

| Phase | `PersistAll` (default) | Selective (any Persist* declared) |
|-------|------------------------|-----------------------------------|
| Writable branch | **is** `USERDATA` (durable) | ephemeral (wiped after) |
| Before mount | — | seed `PersistFile`/`RegPersist`/`RegKeyPersist` from `USERDATA` into the writable layer |
| During session | everything persists live | only `PersistDir` is live-durable |
| At teardown | nothing to capture (already durable) | capture `PersistFile`/`RegPersist`/`RegKeyPersist` back to `USERDATA`, before unmount |
| Save-safety | the union mount is durable-backed → non-lazy verified unmount | `PersistDir`s are durable-backed → non-lazy verified unmount |

Save-safety (invariant I7) is detailed in [chapter 13 §13.6](13-runtime-model.md).

Next: [Variables & CustomVar](08-variables.md).
