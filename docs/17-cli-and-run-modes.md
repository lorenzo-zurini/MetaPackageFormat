# 17 · Run modes, instances & the CLI

The format defines data. This chapter describes the operational surface a runtime is expected to expose: where its
data lives, instances, and a headless command line for resolving, validating and launching without a GUI. These are
recommendations drawn from the reference implementation; an implementation MAY differ but SHOULD offer the same
capabilities.

## 17.1 The data root

Everything a runtime keeps hangs off one **data root**: the library and catalog, the content backend's repository,
the instance lock and the instances (`USERDATA`). One root means one place to back up and one place to wipe. The
per-launch session scratch lives in the system's temporary directory instead (chapter 14 §14.2): it never needs to
persist. *(VidyaGod: `AppPaths::DataRoot`, default `~/.VidyaGod`.)*

## 17.2 Run modes

How the runtime was started decides where its state goes and whether it may stay resident:

| Mode | Trigger | Data root | Tray / background node | Start on login |
|------|---------|-----------|------------------------|----------------|
| **Normal** | the installed application | `~/.VidyaGod` (or the platform's equivalent) | allowed | allowed |
| **Portable** | a `portable.txt` or `VidyaGod_data` beside the executable | that folder | allowed | not allowed (explained in the UI) |
| **In-package** | started inside a package folder | the package folder: runtime and userdata too | not allowed: run the game and exit | not allowed |
| **CLI paths** | explicit path arguments (`--data-dir`, …) | as given | if it reaches a GUI | not allowed |

`--data-dir` gives an isolated second runtime for testing beside the user's own.

## 17.3 Instances

An **instance** is one family's loadout and owned state (chapter 12 §12.6, chapter 13 §13.5):

```
<data root>/USERDATA/<PackageUID>/<instance name>/
├── instance.json     VARIABLES, GRAFTS (the ticked grafts in order), RUNNER_CHAIN, SELECTED_VARIANT,
│                     SKIP_LAUNCH_DIALOG, LASTRUN
└── <NAME>…, REGKEYS/ the owned state
```

- Instances are discovered by scanning the family's folder; there is no registry of them. A family with none gets
  `DefaultInstance` on its first launch. The pre-selected instance is the one that ran last.
- A user may keep several instances of one game (two save sets, two mod loadouts). Deleting the folder deletes the
  instance.
- `instance.json` is never inside a mount the game can see: every game-writable mount is a named child of the
  instance folder.
- The `<PackageUID>` segment comes from package data and MUST be sanitised to one safe path segment.

## 17.4 Path overrides and the single-instance guard

| Override | Effect |
|----------|--------|
| `--data-dir <path>` | the whole data root |
| `--package-dir <path>` | where the launched package is read from |
| `--runtime-dir <path>` | the overlay mount point for this launch |
| `--userdata-dir <path>` | the owned-state folder for this launch, bypassing instances (one implicit instance) |

In-package mode sets the package, runtime and userdata paths to the package folder: a self-contained, relocatable
runnable unit.

A runtime SHOULD allow one live process per data root (an exclusive OS lock that is released when the process dies),
so two processes never fight over one runtime, and any mount present at startup can be treated as crash debris
(chapter 14 §14.7). Distinct data roots have distinct locks.

## 17.5 Headless CLI

The reference implementation's verbs. The capabilities matter, not the names. An unknown argument does not print
help; it starts the GUI.

**Launching and resolving**

| Verb | Capability |
|------|-----------|
| `--node <cid>` | launch a node: resolve, check what is present, mount, run, capture, tear down |
| `--tile <uid> [--variant <name>]` | launch by what the shelf shows: a tile, and optionally a variant by its `VARIANT` name |
| `--resolve-only <cid>` | resolve everything (the plan, the runner chain, variables, paths, the environment) and write it as JSON, without mounting or launching. Takes the node as its argument. |
| `--entrypoint <label>` | run this entry of the row |
| `--graft <cid>` (repeatable), `--no-grafts` | the ticked grafts, in order, for this launch; or none |
| `--runner <cid>` (repeatable) | pin the runner chain, innermost first; the terminal is appended |
| `--var KEY=VALUE` (repeatable) | a per-launch variable value |
| `--instance <name>` | an existing instance (an unknown name is refused) |
| `--offline` | do not start the content network (resolve sweeps) |

**Checking**

| Verb | Capability |
|------|-----------|
| `--validate-nodes` | validate the library (chapter 16) |
| `--audit-packages` | resolve and lower every launchable as a launch would, without mounting, and report what would fail |
| `--list-nodes` | list the shelf's tiles and what is present |

**Content and sharing**

| Verb | Capability |
|------|-----------|
| `--cid <file>…` | the CID a file's bytes get, without seeding anything |
| `--hydrate <cid>` | fetch a node's content |
| `--import-package`, `--import-runner` | download a game, install a runner |
| `--publish`, `--seed <dir>`, `--mint <dir>` | publish the library, re-seed content, freeze a working tree (chapter 15) |
| `--friend-code`, `--friend-add`, `--friend-ls` | the friend list sharing goes through |

`--resolve-only` is the most useful for conformance testing: it exercises resolution, the chain and variables, and
its output can be compared with a golden. But a green sweep proves only what its path touches: a resolve says nothing
about the mount, the play gate, capture or hydration. Test each surface. (VidyaGod's launch matrix does both: plan
goldens from `--resolve-only`, and a real `--node` run whose probe reports what the game saw.)

Every launch, from any front end, ends with the teardown of chapter 14 §14.6, including capture of owned state.

Next: [Worked examples](18-examples.md).
