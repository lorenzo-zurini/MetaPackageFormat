# 14 · The runtime model

This chapter specifies how a plan, the runner's build and the user's owned state become **one overlay mount** that the
runner executes against, and how the session is taken down without risking the user's data.

## 14.1 One overlay

The runtime is a **single overlay (union) file system** mounted at `%RuntimePath%` (I4). A Wine-family runner's
prefix variables point at it, an emulator runs from it, and the content lives in it. Nothing is extracted or copied:
zips are served by offset, directories and files are passed through, deltas are reconstructed on read.

The layers, lowest priority first:

```
┌──────────────────────────────────────────────────────────────────────┐  highest
│  WRITABLE layer        ephemeral; every session write lands here        │
│  kept folders          read-write, live, backed by the instance (ch. 13)│
│  DEFAULT-DATA          the composed registry hives (§14.4)              │
│  inner runner builds   at <CONTENT_ROOT>/__runner_<id>__ (ch. 11 §11.6) │
│  content layers        the plan's content, in fold order                │
│  runner assembly       the boundary runner's runtime-sourced layers     │
│                        (its prefix, its system DLL folders)             │
└──────────────────────────────────────────────────────────────────────┘  lowest
```

- **Runner assembly at the bottom.** A runner's prefix and builtin DLL folders are the base system the content is laid
  into; a package's own DLLs at the same paths must win over them (chapter 10 §10.5).
- **Content in fold order** (chapter 12): later wins.
- **Kept folders** expose the instance's folders read-write at their addresses, over the package's files there.
- **The writable layer is always ephemeral.** It catches every write outside a kept folder and is removed after the
  session, which keeps the instance's own files out of anything the game can write.

After mounting, the plan's **edits** run in fold order against the mounted files, writing into the writable layer
(chapter 06): text edits, then binary patches. Owned files are put in place before them (chapter 13), so an edit on
an owned file sees the user's copy and leaves it alone.

*(VidyaGod: `VfsMount::BuildLayerSpec` builds this stack for `vidyagodfs`; the runner build mount is separate.)*

## 14.2 The session tree

Everything ephemeral for one launch lives in one per-user scratch root, so one delete removes it:

```
<tmp>/VidyaGod-<user id>/<PackageUID>/<instance>/
├── RUNTIME       the overlay mount point             %RuntimePath%
├── RUNNER        the boundary runner's build mount    %RunnerMount% (unless unified)
├── WRITELAYER    the ephemeral writable branch
└── DEFAULTDATA   the composed registry hives, rebuilt every launch
```

The scratch root is keyed by user, so two users cannot collide on or pre-create each other's mount points. Owned
state lives outside it, in the instance (chapter 13 §13.5).

## 14.3 Content placement

- **Content root** (`CONTENT_ROOT`, the runner's): where the game's files sit. `""` for native content,
  `pfx/drive_c/%PackageUID%` inside a Proton prefix.
- **Prefix root** (`%PrefixRoot%`): the part of the content root before `drive_c` (`""` for Wine at the root, `pfx` for
  Proton). The registry hive files live there.
- Every placed path (a content target, an edit's file, a kept address) is substituted, then mapped from guest
  coordinates to the runtime layout through the boundary runner's `DRIVES` (chapter 10 §10.4).

## 14.4 The registry

For a prefix-generating runner, the **DEFAULT-DATA** layer holds the registry hives (`system.reg`, `user.reg`,
`userdef.reg`), composed each launch:

1. load the runner's pristine hives (never modified);
2. apply the plan's `REG` values (and the runner's), per architecture;
3. merge the user's kept registry subtrees (chapter 13), so the user's state wins over package defaults;
4. re-apply the values the package took back inside them.

Wine writes the registry copy-on-write into the writable layer; what the user keeps is captured from there after the
run. A runner without a prefix has no hives and no DEFAULT-DATA layer.

## 14.5 The sandbox

The launched program SHOULD run isolated from the host: its file system view limited to the runtime, the mounts it
needs and the devices a game uses. VidyaGod requires a sandbox (bubblewrap on Linux, Sandboxie on Windows) and refuses
to launch without one, naming what is missing. Multiplayer LAN access goes through the sandbox's own network
namespace. The format does not depend on a particular sandbox.

## 14.6 Teardown and save safety (I7)

After the process exits:

1. **Capture** owned state while the runtime is still mounted: kept files, pattern matches and registry subtrees go to
   the instance (kept folders are already durable). A failed capture is an error the user sees.
2. **Unmount durable-backed mounts non-lazily and verify**: any mount that exposes an instance folder is unmounted
   with a blocking unmount, retried, and confirmed gone from the mount table.
3. **Unmount purely ephemeral mounts** (content and runner mounts backed only by scratch).
4. **Remove the scratch tree only if every durable-backed mount detached.** If one is still live, the delete is
   abandoned and the scratch is left: a recursive delete could otherwise walk through a live mount into the user's
   saves. Leaving scratch behind is the safe failure.

An implementation MUST NOT delete an ephemeral tree while a mount that reaches owned data could still be traversed.
It MUST run this teardown after every launch it started, from every front end (a GUI and a command line alike), and
also after a launch that mounted and was then refused.

## 14.7 Stale runtimes

A process can crash and leave mounts behind. Before assembling a runtime, an implementation MUST look for mounts still
present under that launch's scratch root, unmount them (with the same save-safety gate), and only then clear the stale
tree. A single-instance guard (chapter 17 §17.4) lets it treat any such mount at startup as debris.

## 14.8 What an implementation must provide

An overlay file system that can mount STORE-zip entries without copying, reconstruct delta archives on read, mount
directories and single files, place part of a source at another path (what a `TAKE` lowers to), stack layers by
priority, expose a writable top branch, and pass kept folders through read-write. VidyaGod ships a FUSE file system,
`vidyagodfs`, driven by a JSON layer spec (WinFsp on Windows). Any overlay with the same semantics conforms.

Next: [Packages, CIDs & sharing](15-packages-and-sharing.md).
