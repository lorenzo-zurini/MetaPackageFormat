# 20 · Conformance

The checklist for a from-scratch implementation. An implementation **conforms** if it satisfies every **MUST**
below. It is also a build order: each section runs once the ones before it do.

## 20.1 Nodes and the index

- [ ] **MUST** read every `.json` file in a package folder that is a node object (it has a `LAYERS` list), and key it
      by the CID of its canonical bytes (I1, chapter 02 §2.3); a working tree's stored `CID` handles are resolved by
      freezing.
- [ ] **MUST** compute canonical bytes exactly: `CID` and `POS` dropped, keys sorted by code point, no insignificant
      whitespace, UTF-8; CIDv1, raw codec, sha2-256, base32.
- [ ] **MUST** refuse unknown node fields and malformed layers (chapter 16 §16.1); a node that does not freeze or
      parse affects only itself and what references it.
- [ ] **MUST** keep one home per node: read a node's content from its home package, treat a friend's catalog copy of
      an installed package as a view of it, and report a node found in two installed packages as an error.

## 20.2 Resolution

- [ ] **MUST** resolve in three phases (chapter 12 §12.1): `WHEN` variables from the lists, expansion, fold.
- [ ] **MUST** expand `NODE` layers in place through `TAKE` (left-stripped selections, renaming pairs, fact namespaces)
      and `TARGET` (composed placement; anchored paths stay put) (chapter 05).
- [ ] **MUST** treat a repeated occurrence as one, at its first position, moving it only for a later mention in the
      same list; a node never leaves its container (chapter 05 §5.4, I3).
- [ ] **MUST** fold per address, later wins, `null` deletes; edits and deltas apply to what is beneath them; entries
      fold per label and field in first-declared order (chapter 12 §12.2).
- [ ] **MUST** evaluate `ANY` at its position and `NOT` against the final expansion; block a variant whose checks fail.
- [ ] **MUST** offer grafts through the index of their leading `ANY`, apply ticked grafts in instance order judging
      each against the row with the ones before it, tick `RECOMMENDED` grafts on a fresh instance, and never offer a
      graft the row contains (chapter 12 §12.5).
- [ ] **MUST** complete resolution through a cycle, reporting it.

## 20.3 Variables

- [ ] **MUST** implement substitution exactly (chapter 08 §8.1): paired `%`, an optional `:format`, an unmatched `%`
      stops, an unknown token stays.
- [ ] **MUST** provide the built-in tokens and every runner anchor (chapter 08 §8.2), and the render formats of
      chapter 08 §8.4.
- [ ] **MUST** resolve variables by override > instance value > `DEFAULT`, with declaration `WHEN` gating the value,
      `EVAL`, and a fixpoint over all variables, before applying anything that references them.
- [ ] **MUST** evaluate the `WHEN` grammar as specified, fail-open at launch, with a malformed condition a validator
      error.

## 20.4 Platforms and runner chains

- [ ] **MUST** treat runners as `GUEST → HOST` edges and find the shortest chain to the machine platform, over
      installed runners first and all available runners only when installed ones cannot carry the launch, always
      ending in a native terminal (synthesised if none) (chapter 11 §11.4).
- [ ] **MUST** scope `custom` to engines whose home is the launch's home package, at every place a runner is matched
      (chapter 10 §10.6).
- [ ] **MUST** refuse a launch with no route to the machine (I5) and execute only the terminal (I4).
- [ ] **MUST** honour a valid pinned chain.
- [ ] **MUST** place anchored paths through the boundary runner's `GUEST_ROOTS` and `DRIVES` (chapter 10 §10.4).
- [ ] **MUST** compose outer links by concatenation and inner links inside the boundary's guest: builds at
      `<CONTENT_ROOT>/__runner_<id>__`, guest paths from the boundary's `%GameDir%`, an inner runner's
      `%RunnerMount%` its own build folder, its `ENV` merged under the game's (chapter 11 §11.6).
- [ ] **SHOULD** show the chain as per-step choices that re-resolve downstream.

## 20.5 Content, edits and the runtime

- [ ] **MUST** mount `ZIP` (STORE only), `FILE`, `DELTA` (base = the nearest content beneath at its target) and `DIR`
      layers as one overlay in fold order (chapters 04 and 14).
- [ ] **MUST** apply present-beats-fetch (I8) and refuse a launch needing unavailable content.
- [ ] **MUST** place the boundary runner's runtime-sourced layers beneath the content, and its build at
      `%RunnerMount%` (or unified).
- [ ] **MUST** apply every `EDIT` op to the file as composed beneath it, after the mount, in fold order, and report
      failed ops (chapter 06).
- [ ] **MUST** compose the registry per launch from pristine hives, `REG` layers, kept subtrees and take-backs
      (chapter 14 §14.4) for prefix runners.
- [ ] **MUST** build the process environment as chapter 07 §7.3 says.

## 20.6 Ownership and save safety

- [ ] **MUST** treat everything as package-owned except what `KEEP` gives the user, most specific address winning, with
      `false` taking back (chapter 13).
- [ ] **MUST** keep folders live, copy files in and out, store pattern matches and registry subtrees, keyed by `NAME`
      in the family's instance; refuse reserved and colliding names.
- [ ] **MUST** apply an edit on a user-owned address only while the user has no saved copy, and re-apply taken-back
      addresses after restoring the user's state.
- [ ] **MUST** capture before unmounting, unmount durable-backed mounts non-lazily and verify, and delete scratch only
      when every durable mount is gone (I7), after every launch from every front end.
- [ ] **MUST** recover stale runtimes before mounting.

## 20.7 Packages and sharing

- [ ] **MUST** fetch content by CID only when missing and verify fetched bytes.
- [ ] **MUST** follow `NODE`, `SOURCE` and `TILE.COVER.SOURCE` when hydrating, never `ANY` or `NOT`.
- [ ] **SHOULD** publish one share folder (node files) and one pin folder (plus content) per package, reproducibly,
      with the packages each one needs, and share them closed over what they need.
- [ ] **MUST NOT** fetch a node into a package other than its home.
- [ ] **MUST** change an installed package only when the user chooses a friend's version, keep every node something
      installed still contains, and give colliding content files their bumped names (chapter 15 §15.6).
- [ ] **MUST** treat received node files and paths as untrusted: sanitise, bound, keep file names inside the package.

## 20.8 Validation

- [ ] **MUST** report every error of chapter 16; **SHOULD** report the warnings.

## 20.9 Testing against the reference

The most direct test of a new implementation is to resolve the same packages in both and compare: VidyaGod's
`--resolve-only` writes the plan, the chain, the variables and the environment as JSON, and its launch-matrix fixture
(`tests/fixtures/launchmatrix`) exercises every feature with goldens for the plans and for what a real run sees.

Next: [Appendix: platform tokens](21-platform-tokens.md).
