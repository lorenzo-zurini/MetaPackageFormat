# 16 · Run modes & the CLI surface

The format defines data; this chapter defines the *operational* surface a conforming runtime is expected to expose —
where its data lives, the run modes that relocate it, and a headless CLI for resolving/validating/launching without a GUI.
These are recommendations for interoperability and testability, drawn from the reference implementation; an implementation
MAY differ, but SHOULD provide equivalent capabilities.

## 16.1 The data root

A runtime hangs everything it produces off one **data root**: the content backend's repo, the instance lock, the default
library location, and the per-session `TEMP` runtime tree. Rooting everything here means a single delete is a true clean
slate, and relocating the root relocates the whole footprint. The one deliberate exception is per-package `USERDATA`,
which lives with the bundle (chapter 7) so saves travel and survive a data-root wipe. (VidyaGod: `AppPaths::DataRoot`,
default `~/.VidyaGod`.)

## 16.2 The four run modes

A runtime resolves its data root and which resident features are allowed from how it was launched. The reference defines
four modes:

| Mode | Trigger | Data root | Daemon/tray | Start-on-login |
|------|---------|-----------|-------------|----------------|
| **Normal** | default launch (from a desktop entry / app image) | `~/.VidyaGod` (or platform equivalent) | allowed | allowed |
| **Portable** | a `portable.txt` / `VidyaGod_data` beside the executable | that directory | allowed | **not** allowed (greyed, explained) |
| **In-package** | launched *inside* a package bundle (run-this-folder) | the package dir; runtime/userdata = the package dir | **not** allowed (run the game, exit) | not allowed |
| **CLI paths** | explicit path arguments (`--data-dir`, …) | as specified | possible if it reaches a GUI | not allowed |

The point of the modes is that "where does state go?" and "may it stay resident?" are answered once, up front, from the
launch context — so a portable install is genuinely self-contained, an in-package run is ephemeral and exits, and
`--data-dir` gives an isolated second instance for testing. (VidyaGod: `AppPaths::Mode`, `DaemonSupported`,
`AutostartSupported`.)

## 16.3 Per-launch path overrides

Independently of mode, a launch's three relocatable paths can be overridden:

| Override | Effect |
|----------|--------|
| `--data-dir <path>` | the whole data root (repo + lock + TEMP + default library). |
| `--package-dir <path>` | the launch package/bundle path (where the launchable's bundle is read from). |
| `--runtime-dir <path>` | the overlay mount root (`%RuntimePath%`) for this launch. |
| `--userdata-dir <path>` | the durable `USERDATA` location for this launch. |

In-package mode sets package/runtime/userdata all to the package directory, making the bundle a self-contained,
relocatable runnable unit. (VidyaGod: `AppPaths::*Override` consumed by the launch engine.)

## 16.4 The single-instance guard

A runtime SHOULD enforce a single live instance (an exclusive lock on a file under the data root), for two reasons:
two instances would fight over the same `TEMP`/runtime, and a single instance lets the runtime assume any mount still
present under `TEMP` at startup is crash debris to sweep (chapter 13 §13.7). A stale lock from a crashed process must not
block a new instance (use an OS lock that releases on process death, e.g. `flock`). (VidyaGod:
`AcquireSingleInstanceLock`.) Distinct data roots (`--data-dir`) have distinct locks, so an isolated test instance can run
alongside the main one.

## 16.5 Headless CLI

A headless surface makes the format testable and scriptable without a GUI. The reference implementation's verbs (names
are illustrative; the *capabilities* are what matter):

| Verb | Capability |
|------|-----------|
| `--node <id>` | Launch a launchable node by id (full real path: resolve, hydrate-check, mount, execute, tear down). |
| `--resolve-only <id>` | Resolve everything (closure, runner chain, variables, paths) and dump the resolved parameters **without** mounting or launching. The fast, side-effect-light way to verify resolution — including the full runner chain. |
| `--runner <id>` (repeatable) | Pin the runner chain: each occurrence appends a link (innermost→outermost); the native terminal is appended automatically. |
| `--var KEY=VALUE` (repeatable) | Override a `CustomVar` for this launch. |
| `--module ID=on|off` (repeatable) | Override an optional node's toggle for this launch. |
| `--validate-nodes` | Validate the whole graph (chapter 15) and report errors/warnings. |
| `--list-nodes` | List presentable library tiles (grouped launchables) and hydration status. |
| `--import-package <uid>` / `--import-runner <id>` | Hydrate a game / install a runner (fetch build + generate prefix). |
| `--publish <dir>` / `--seed <dir>` | Publish (dehydrate-for-sharing) / re-seed a package's content (chapter 14). |

`--resolve-only` is the most useful for spec conformance testing: it exercises closure resolution, chain resolution and
variable expansion, and emits a dump a test can golden-compare — proving the *decisions* without the side effects of a
real mount/launch. (For cross-namespace *command composition*, which happens at execution, a real `--node` launch or an
equivalent dry path is needed.)

## 16.6 Saved user settings

Per-package user choices persist across launches, keyed by the launchable's `UID`: the preferred runner chain
(`RUNNER_CHAIN`), `CustomVar` values (`VARIABLES`), optional-node toggles (`MODULES`), and UI preferences (e.g. skip the
prelaunch dialog). These are an implementation's own store (the reference keeps them in its global config), not part of a
distributed package — they describe *this user's* preferences, and feed the resolution priority chain (chapter 8 §8.3,
chapter 11 §11.4).

Next: [Worked examples](17-examples.md).
