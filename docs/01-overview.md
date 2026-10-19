# 01 · Overview & design model

## 1.1 The single idea: a node is an ordered list of layers

Everything in MPF is a **node**: a JSON file whose `LAYERS` list says, bottom to top, what the node is made of and
what it writes. A layer is one typed entry:

```jsonc
{ "LABEL": "Warcraft III 1.27a",
  "LAYERS": [
    { "NODE":  "bafkrei…wc3_1_26a" },                                  // the version before, contained
    { "DELTA": "delta_from__wc3_1_26a.vgdelta", "SOURCE": "Qm…",
      "TARGET": "FILES/%GameDir%" },                                    // what changed, as a binary delta
    { "REG":   { "HKCU": { "Software": { "Blizzard Entertainment": { "Warcraft III": {
                 "Gameplay": { "netgameport": "dword:000017e0" } } } } } } },
    { "KEEP":  { "REG/HKCU/Software/Blizzard Entertainment/Warcraft III": true } },
    { "EXEC":  [ { "LABEL": "Play", "HOST": "win32", "EXE": "%GameDir%/Frozen Throne.exe" } ] } ] }
```

There are no node types. A game, a patch, a library, a runner, a mod and a configuration option are all nodes; what
a node *is* follows from its layers: it has a way to run (`EXEC` without `GUEST`), it is a runner (`EXEC` with
`GUEST`), it is a graft (its list begins with `ANY`), it is on the shelf (`VARIANT`).

**Parents are layers.** A node's ancestors (the version it patches, the libraries it uses) are `NODE` layers at the
position the node chooses, and a `NODE` layer can take part of a node (`TAKE`) and place it (`TARGET`). The
direction is always newer → older: a library never lists its users, a version never lists its mods.

## 1.2 Addresses and the fold

Everything a layer writes has an **address**: a file (`FILES/%GameDir%/Data/x.dat`), a key inside a config file
(`FILES/%GameDir%/game.ini#Resolution=`), a registry value, a variable, an environment variable, a DLL policy, an
entry. Resolving a node walks its list, expands every `NODE` layer in place, and folds: per address, **the later
writer wins**, `null` deletes, and an edit applies to the value beneath it. There is one rule for every kind of
fact; a registry value, an environment variable and a file are overridden the same way.

## 1.3 What a complete package is

A **package** is a folder of node files and the content files they name, published as one folder CID (chapter 15).
A typical game package holds:

- the **pristine** node: the game's files exactly as released, its entry and its tile;
- **versions**: nodes that contain the previous version and add a delta or a patch;
- **variants**: the nodes the user can pick, marked `VARIANT`;
- **grafts**: optional changes the user can tick, whose list begins with `ANY`;
- package-local **libraries** and, for a game with its own engine, the engine as a `custom` runner (chapter 10 §10.6).

Shared runners and libraries (Proton, DirectX redistributables, dgVoodoo) live in their own packages. A game never
names its runner: the runner chain is found by capability.

## 1.4 The lifecycle of a launch

1. **Pick a row**: a variant × one of its tiled entries (chapter 09).
2. **Resolve** the variant's node with the instance's variable values and ticked grafts (chapter 12): phase 1 reads
   the variables `WHEN` may use, phase 2 expands, phase 3 folds and resolves the variables fully.
3. **Find the runner chain** from the entry's platform to the machine's, preferring installed runners (chapter 11).
   Each runner resolves its own node the same way.
4. **Hydrate** what is missing: fetch each content layer's bytes by CID (chapter 15).
5. **Assemble the runtime**: one overlay mount of the runner's layers, the game's content, the edits and the user's
   owned state (chapters 13 and 14).
6. **Run** the entry: one process, the native terminal, nesting every runner and the content.
7. **Tear down**: capture the user's owned state, unmount, remove the runtime.

## 1.5 Things that fall out of the model

- **Mod load order.** An `AppendLine` edit in each mod writes one line of a load-order file; the ticked mods' lines
  appear in graft order. No mod-manager construct.
- **Optional expansions.** An expansion is a node containing the base game with an entry carrying its own tile
  (`PARENTUID` = the base game). Both tiles share one family, one install folder and one set of owned state.
- **Multi-edition games.** Several variants contain one pristine and differ by a delta or an option.
- **Cross-platform execution.** A console ROM, a Windows emulator and Proton compose by capability.
- **Peer-to-peer distribution.** Every node and every content file is addressed by CID; a friend can serve them.
- **Portable saves.** What the user owns is captured into the instance by name, independent of the runner layout.
- **Reproducible runtimes.** A launch is a pure function of the nodes, the instance and the runner chain.

When a feature is proposed, the first question is: *which arrangement of layers already expresses this?* The format
grows only when the answer is none.

## 1.6 Core invariants

A conforming implementation MUST preserve these. Later chapters refer to them.

- **I1 — CID identity.** A node's identity is the CID of its canonical bytes and nothing else. `LABEL` is cosmetic
  and may repeat. Every reference is a CID string (chapter 02).
- **I2 — Containment is the only structure.** A node is exactly its list. What a node is made of is what its `NODE`
  layers contain, where they are; nothing outside a node changes what it resolves to except the instance (variable
  values, ticked grafts) and the runner chain. A cycle is reported and its back edge skipped (chapter 12).
- **I3 — One fold.** Per address, the latest writer in expansion order wins; `null` deletes; an edit applies to what
  is beneath it. A node never rises above the node that contains it (chapter 12).
- **I4 — Single overlay, one process.** The runtime is one overlay mount. A launch executes one host process, the
  native terminal, which nests every runner and the content (chapters 11 and 14).
- **I5 — Chains always end at the machine.** Every runner chain ends at the machine platform through a native
  terminal. Content with no route to the machine is refused, never "launched" (chapter 11).
- **I6 — Pristine sources, ephemeral runtime, owned state.** Source content is never modified; the runtime is rebuilt
  each launch and removed after; only what the user owns (`KEEP`) survives, in the instance (chapters 13 and 14).
- **I7 — Save safety.** Teardown never destroys owned state: capture first, unmount durable-backed mounts and verify
  them gone before deleting any runtime tree (chapter 14 §14.6).
- **I8 — Present beats fetch.** A content file present at its path is authoritative; its `SOURCE` CID is used only to
  obtain a missing file (chapter 15 §15.2).
- **I9 — Nothing names what grows.** References point from newer to older. A package never names its runner, a
  library never names its users, a pristine never names its grafts. Grouping, offering and chaining are derived.

## 1.7 Relationship to the reference implementation

VidyaGod implements all of the above. Notes like *(VidyaGod: `Fold::Resolve` in `fold.cpp`)* point at the code so a
reader can cross-check. They are a convenience, not part of the format. If VidyaGod and this document disagree, one of
them has a bug: the spec decides what the format is, the implementation decides what currently runs.

Next: [Nodes](02-nodes.md).
