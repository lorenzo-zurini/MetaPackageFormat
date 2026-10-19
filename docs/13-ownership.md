# 13 · Ownership: `KEEP`

The runtime is rebuilt every launch and removed after (I6). **Everything in it belongs to the package**, recomputed
from the nodes each time, **unless a `KEEP` layer gives it to the user**. What the user owns is captured after a run
into the instance and put back before the next one.

```json
{ "KEEP": { "FILES/%Documents%/My Games/Tonic Trouble/": { "NAME": "Tonic Trouble", "CLOUD": true },
            "FILES/%Documents%/My Games/Tonic Trouble/ubi.ini#Language=": false,
            "FILES/%GameDir%/Profile.dat": true,
            "REG/HKCU/Software/Blizzard Entertainment/Warcraft III": true } }
```

## 13.1 The `KEEP` layer

A map of address → `true`, `false` or options.

| Value | Meaning |
|-------|---------|
| `true` | the address (a file, a folder, a registry key, and everything beneath it) is the user's |
| `{ "NAME": …, "CLOUD": … }` | the same, with options: `NAME` is the folder name it is stored under in the instance (one path segment; default: the address's last segment); `CLOUD` (default `true`) marks it eligible for cloud sync, `false` for machine-specific state (a shader cache) |
| `false` | **takes back**: the address is the package's again, even inside something the user owns |

- Addresses are `FILES/…` or `REG/…` only. A variable's value is never kept: what the user picks for an option lives
  in the instance already. A validator refuses any other namespace.
- **The most specific address wins** (compared without case): a kept folder with one file inside taken back gives the
  user the folder and the package the file.
- `KEEP` **folds** like any address: a library can keep its config, a game can take one of its files back, a ticked
  graft can keep or take back while ticked. Runners may keep things too, though VidyaGod's runners keep nothing: it
  is each author's decision.
- Owned state belongs to the **family** (`%PackageUID%`): an expansion and its base game read and write the same
  saves. Things that must stay separate (two tiles' CD-key files) are separate addresses.

## 13.2 What each kind of address does

| Address | Becomes |
|---------|---------|
| a `FILES` **folder** (a trailing `/`, or a last segment without an extension) | a **live** folder: the user's folder in the instance is mounted read-write at that place, over the package's files there. Writes land in the instance as they happen. |
| a `FILES` **file** (a last segment with an extension) | a **copied** file: the user's saved copy is put in place before the mount (it shadows the package's), and captured after the session |
| a `FILES` **pattern** (`*` or `?` in the last segment) | a stored folder (`NAME`, default the pattern with wildcards as `_`) holding every matching file under its own name. Matching ignores case. Capture stores each match the session left, and drops a stored one only when this session put it there and the game deleted it, after a listing with no error. Restore puts every stored match back. |
| a `REG` **key** | its subtree, stored in the instance and merged into the composed hives before the run, captured after. It must name a key below a hive: a whole hive cannot be kept. |

A pattern names files in one folder: wildcards only in the last segment, and never on a folder (`…/*/`). A bare
`FILES` (the whole runtime) is an authoring aid for discovering what a game writes, never something to ship; an
implementation MUST require a `NAME` for it and SHOULD warn.

Capture records the whole owned file, folder or key as the session left it. It cannot tell the user from the program:
whatever a game or its runner changed under an owned address is kept. Scope `KEEP` to what should persist.

## 13.3 Package defaults inside owned addresses

- **An `EDIT` on a user-owned address is the package's default.** It applies while the user has no saved copy of
  the file; once the user has one, the user's copy wins and the edit does not touch it.
- **A taken-back address is applied every launch**, after the user's state is restored: a `ConfigWrite` key inside a
  kept config file, a registry value inside a kept key. Ownership of the registry is judged per value
  (`REG/<key>/<value>`), so a key can be the user's except for one value an option controls.
- A package update reaches only what the user does not own. A new default for an owned file reaches users who have no
  saved copy yet.

## 13.4 Options in owned files

An option chosen in the launch window that writes into a user-owned file (a `ConfigWrite` or `JsonSet` whose `VALUE`
or `WHEN` names the option) would apply on the first launch only, because afterwards the user's copy wins. Its key
must be taken back:

```json
{ "KEEP": { "FILES/%LocalAppData%/Game/settings.json": true,
            "FILES/%LocalAppData%/Game/settings.json#/video/targetfps": false } }
```

A validator SHOULD warn about an option written into an owned file without such a take-back.

## 13.5 Where owned state lives

```
<data root>/USERDATA/<PackageUID>/<instance>/
├── instance.json        the instance's configuration: never inside a mount the game can see
├── <NAME>/              a kept folder or pattern store (live, or a capture of matches)
├── <NAME>               a kept file
└── REGKEYS/             kept registry subtrees
```

- A `NAME` is one safe path segment. `instance.json`, `registry` and `regkeys` are reserved (case aside) and refused.
- Two keeps with one `NAME` and different addresses are refused (case aside, since instances travel to
  case-insensitive file systems); the second is skipped with an error. The same address kept twice is harmless.
- An implementation MUST keep the instance's own files out of any mount exposed to the game, and MUST never place owned
  state in the runtime tree.

## 13.6 Lifecycle

| Phase | Folder | File | Registry key |
|-------|--------|------|--------------|
| before the mount | mount the instance's folder read-write at the address | put the saved copy in place | merge the saved subtree into the hives; re-apply taken-back values |
| during the run | writes are durable as they happen | writes go to the runtime | writes go to the runtime hives |
| after the run | nothing to capture | capture the file | capture the subtree |
| teardown | verified unmount before any delete (I7) | | |

Capture happens before anything is unmounted (chapter 14 §14.6), and a failed capture is an error the user sees.

## 13.7 Cloud saves (forward-looking)

`CLOUD` marks what may sync between a user's machines. Machine-specific state (a shader cache, a configuration tuned
to one GPU) sets `CLOUD: false`. The flag is part of the format now; the sync itself is future work.

*(VidyaGod: `NodeLower::UserOwned` and `LowerPlan` (defaults and take-backs); `LaunchResolver::DerivePersistence`;
`PersistLayer` and `RegistryLayer` capture and restore.)*

Next: [The runtime model](14-runtime-model.md).
