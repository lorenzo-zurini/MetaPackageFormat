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
├── DEFAULTDATA   the composed registry hives, rebuilt every launch
└── HOME          the launched program's private home (§14.5)
```

The scratch root is keyed by user, so two users cannot collide on or pre-create each other's mount points. Owned
state lives outside it, in the instance (chapter 13 §13.5).

## 14.3 Content placement

- **Content root** (`CONTENT_ROOT`, the runner's): where the game's files sit. `""` for native content,
  `pfx/drive_c/%PackageUID%` inside a Proton prefix.
- **Prefix root** (`%PrefixRoot%`): the part of the content root before `drive_c` (`""` for Wine at the root, `pfx` for
  Proton). The registry hive files live there.
- Every placed path (a content target, an edit's file, a kept address) is substituted, then mapped from guest
  coordinates to the runtime layout through the boundary runner's `DRIVES`, or, for a guest path at or below
  `%RuntimePath%` (a native guest), to its path below the runtime (chapter 10 §10.4).

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

**A private home.** The launched program SHOULD NOT be able to write the user's home folder. It gets one of its own:
`HOME` and the XDG base directories (`XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME`, `XDG_CACHE_HOME`) point at
an empty folder in the session tree, made for every launch and removed with it, and the user's real home is read-only
to the program apart from the folders the launch itself exposes there (the instance's kept folders, chapter 13).
These values are the base of the process environment (chapter 07 §7.3), so a runner or package `ENV` that sets or
removes one of them wins. What a program writes in its home is ephemeral, like the writable layer: to keep it, a runner
points the variable into the runtime (`%RuntimePath%/…`) and the package keeps that folder (chapter 13).
The guest's profile anchors (`%UserProfile%`, `%AppData%`, `%Documents%`) are runtime addresses, not the host's home,
so they and the keeps on them are unaffected.

A program also no longer sees what the user configured in their own home (a `~/.drirc`, user fonts, Vulkan layers
installed under `~/.local/share`): a package behaves the same on every machine.

*(VidyaGod: `SandboxLayer::FromParams` and `HomeEnv`. Two things of the real home stay reachable: the GPU driver's
shader caches, bound WRITABLE into the private home where the driver looks for them (otherwise every launch recompiles
every shader; the caches are the driver's, shared with the user's other programs), and `XAUTHORITY`, set to the real
`~/.Xauthority` when the session names no other. The sandbox init drops `CAP_SYS_ADMIN` before it becomes the game, so
the game cannot remount the read-only root. At teardown each file a program left in its home is a warning.)*

**On Windows, the box is the prefix.** A Windows program on a Windows host has no Wine prefix to hold what its
package puts in the profile, the system folders or the registry. The sandbox holds it instead: every launch of an
instance gets its own box, rooted in the session tree (never inside the runtime mount), and

1. before the program starts, the box is emptied, then filled with everything the runtime has outside the game folder,
   each file at the real path the box shows it at (`%AppData%/X` → the box's copy of the user's AppData), and the
   registry the plan composes (§14.4 steps 2–4) is imported into the box's registry;
2. the game folder is not copied: the runtime mount is passed through the box, so the program reads the package's
   files and writes its own into the writable layer and the kept folders, as on Linux;
3. after the program exits, the kept folders and files outside the game folder are copied out of the box into the
   runtime, the kept registry subtrees are exported from the box's registry, and the usual capture (§14.6) stores
   them; then the box is emptied. Every launch starts from what the package and the instance say, never from what an
   earlier run left in the box.

The box drops administrator rights. Its network is not closed: LAN play needs it, and the overlay's own rules apply.

*(VidyaGod: `BoxLaunch::Prepare`/`Harvest`, `BoxPrefix`, Sandboxie-Plus. The runtime is mounted at a free drive
letter, since a program in a box cannot load a DLL from a folder mount. Sandboxie's privacy and security modes need a
supporter certificate and are not used.)*

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
also after a launch that mounted and was then refused. Told to quit while a game runs (a logout, an interrupt, a
service manager's stop), it SHOULD end the game as a user's stop does and run this teardown before it exits; a launch
not yet at its game stops there instead of starting it.

## 14.7 Stale runtimes

A process can crash, or be killed, and leave its scratch tree and mounts behind. Before assembling a runtime, an
implementation MUST look for mounts still present under that launch's scratch root, unmount them (with the same
save-safety gate), and only then clear the stale tree.

At startup it SHOULD clear every scratch tree whose launch is gone, the same way: debris piles up otherwise, and a
scratch root in memory fills it. Whether a launch is gone is a fact about its owner, not a guess from age or from a
guard: scratch roots can be shared by more than one running implementation (two data roots on one machine). So a
launch holds a lock on its scratch tree from assembling the runtime to the end of its teardown, one the operating
system drops when the process dies however it dies, and a tree whose lock nobody holds is debris. A launch whose
scratch tree is locked by another live launch (the same instance started twice) is refused.

*(VidyaGod: `<scratch>.lock` beside the scratch folder, so clearing the folder never removes it: an exclusive `flock`,
taken before anything is cleared or mounted, in `VfsMount::ClaimSession` / `SweepDeadSessions`; the
quit is `QuitSignals` for SIGTERM, SIGINT and SIGHUP on Linux. On Windows sessions are cleared by their instance's next
launch only.)*

## 14.8 What an implementation must provide

An overlay file system that can mount STORE-zip entries without copying, reconstruct delta archives on read, mount
directories and single files, place part of a source at another path (what a `TAKE` lowers to), stack layers by
priority, expose a writable top branch, and pass kept folders through read-write. VidyaGod ships a FUSE file system,
`vidyagodfs`, driven by a JSON layer spec (WinFsp on Windows). Any overlay with the same semantics conforms.

Next: [Packages, CIDs & sharing](15-packages-and-sharing.md).
