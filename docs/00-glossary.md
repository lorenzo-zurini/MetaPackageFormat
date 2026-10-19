# 00 · Glossary

Terms are defined once here and used with exactly these meanings. Capitalised JSON keys (`LAYERS`, `TAKE`) are field
names; lower-case words in `code font` are derived readings.

**Node** — the unit of the format: one JSON file, `{LABEL, VARIANT?, RECOMMENDED?, SECTION?, COMMENT?, POS?,
LAYERS}`. Its identity is its **CID** (chapter 02). `LABEL` is cosmetic and may repeat. A node has no type; what it
is follows from its layers.

**CID** — a content identifier: CIDv1, raw codec, sha2-256 of the node's canonical bytes (chapter 02 §2.3), or of a
content file's bytes (chapter 15). Every reference in the format is a CID string.

**Layer** — one entry of `LAYERS`: an object with exactly one **type key** (`ZIP` `FILE` `DELTA` `DIR` `NODE` `EDIT`
`REG` `VARS` `ENV` `DLL` `EXEC` `KEEP` `ANY` `NOT`) plus that type's own fields, and optionally `WHEN`, `COMMENT`,
`LABEL` and `SECTION` (chapter 03).

**Address** — the name of something a layer writes: `FILES/<path>`, `FILES/<file>#<key>`, `REG/<hive>/<key…>/<value>`,
`VARS/<KEY>`, `ENV/<NAME>`, `DLL/<name>`, `EXEC/<label>` (chapter 03 §3.3). The fold is per address.

**Anchor** — a `%Name%` at the start of a path that a runner places: `%GameDir%`, `%Documents%`, `%SysDir32%`, …
(chapter 03 §3.4, chapter 10 §10.4). Packages write anchors; runners say where they are.

**Containment** — a `NODE` layer: "this node is made of that one, here". The contained node's layers expand in place
of the `NODE` layer, selected by `TAKE` and placed by `TARGET` (chapter 05).

**Occurrence** — one expansion of a node: (node CID, what it is taken through, where it is placed). The same
occurrence reached twice is one occurrence (chapter 05 §5.4).

**Fold** — resolving a node: expand its list, then for each address the latest writer wins; `null` deletes; an edit
applies to what is beneath it (chapter 12).

**Plan** — what a resolution produces: the ordered content and edits, and the folded registry, DLLs, environment,
variables, entries and keeps.

**Variant** — a node carrying `VARIANT: "<name>"`: listed on the shelf under that name (chapter 09 §9.4).

**Entry** — one object of an `EXEC` layer: a way to run (no `GUEST`) or a runner (with `GUEST`) (chapter 09).

**Tile / face** — a `TILE` object on an entry: one presentable game (`UID`, `PARENTUID`, `TITLE`, `COVER`, `META`).
An effective entry carrying a tile is a **face**. Tiles merge by `UID` across nodes and packages (chapter 09 §9.3).

**Row** — what the shelf lists: a variant × one of the tiled entries in its fold.

**Family** — the tiles under one root `UID` (the base game and its expansions, nested by `PARENTUID`).
`%PackageUID%` is the family's root UID; owned state belongs to the family.

**Graft** — a node whose list begins with an `ANY` layer: an option applied on top of a resolution that contains a
member of that `ANY` (a mod, a fix, a loader). It enters a launch only when ticked (chapter 12 §12.5).

**Instance** — the user's loadout for one family: chosen variant, ticked grafts and their order, variable values,
owned state. Local data, never a node (chapter 17 §17.3).

**Ownership** — whose an address is: the package's (recomputed every launch) or the user's (kept across launches).
Everything is the package's unless a `KEEP` layer gives it to the user (chapter 13).

**Runner** — a node whose effective entry has a non-empty `GUEST`: it runs content of the guest platforms while
itself running on its `HOST` platform (chapter 10).

**Platform token** — an opaque string naming an execution environment (`linux64`, `win32`, `snes`, `custom`).

**Machine platform** — the platform of the host the implementation is running on.

**Runner chain** — the runners from an entry's platform to the machine's, innermost first, always ending in a native
terminal (chapter 11).

**Native terminal** — the last link of every chain: a runner with `HOST` = `GUEST` = the machine platform.

**Boundary** — the outermost runner in a chain that creates a guest file system (`CONTENT_ROOT` or a prefix). It owns
the mount; runners inside it are nested in its guest (chapter 11 §11.5).

**Prefix** — a Wine-family runner's generated Windows file system and registry (chapter 14 §14.4).

**Content root** — a runner entry's `CONTENT_ROOT`: where the game's files sit in the runtime.

**Runtime** — the overlay mount assembled for one launch at `%RuntimePath%`, plus the session's scratch space. It is
rebuilt every launch and removed after (chapter 14).

**Package** — a folder of node files and the content files they name, published together as one folder CID
(chapter 15).

**Library** — the local set of packages: a root holding named collections of package folders.

**Catalog** — what friends have shared: their package folders' node files, browsable before anything is downloaded.

**Hydrate** — fetch the content a resolution needs. **Publish** — freeze nodes, add their content and announce the
package folder.

**Working tree** — the editable node files on disk, carrying a stored `CID` handle that references point at before a
freeze turns them into real CIDs (chapter 02 §2.5).

**Reference implementation** — VidyaGod (chapter 19).
