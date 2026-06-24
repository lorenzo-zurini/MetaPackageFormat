# 19 · Conformance

This chapter is the checklist for a from-scratch MPF implementation. An implementation **conforms** if it satisfies every
**MUST** below. The invariants (chapter 1 §1.5) are restated where they apply. This is also the recommended build order:
each section is runnable once the previous ones are.

## 19.1 Indexing & the graph

- [ ] **MUST** scan library roots two levels deep (root → bundles → top-level `.json` files), parse each file, and treat
      a file with a non-empty string `NODE_ID` as a node; ignore the rest. (ch. 4)
- [ ] **MUST** key nodes by `NODE_ID` globally and resolve duplicates first-seen-wins with a diagnostic (invariant I1).
- [ ] **MUST** apply every field default exactly as in chapter 2 §2.2, and ignore unknown fields.
- [ ] **MUST** record each node's bundle directory and resolve relative `PATH`/`SOURCE.PATH`/`COVER.PATH` against *that
      node's* bundle (not the launchable's). (ch. 4 §4.1)

## 19.2 Resolution

- [ ] **MUST** resolve a launchable's content closure by the two-phase algorithm of chapter 12: BFS enable pass
      (`OPTIONAL`/`DEFAULT` toggles, symmetric `EXCLUDE` first-kept-wins, explicit-choice-beats-default, the hierarchy
      gate) then post-order topological emission (parents first, launch node last).
- [ ] **MUST** tolerate `PARENTS` cycles by breaking the back-edge and completing (invariant I2), and a validator **MUST**
      report them as errors.
- [ ] **MUST** treat closure order + `LAYERS` array order as overlay priority (later = higher).

## 19.3 Platforms & runner chains

- [ ] **MUST** model runners as `GUEST → HOST` edges and resolve a chain by shortest-path BFS from the launchable's
      `PLATFORM.HOST` to the machine platform, considering only *available* runners (PATH-resolvable **or** ships a
      build). (ch. 10, 11)
- [ ] **MUST** always append a native terminal (host = guest = machine), synthesizing a pass-through if none is authored,
      and **MUST** `execve` only the terminal (invariant I4).
- [ ] **MUST** refuse a launch whose platform cannot reach the machine platform, with a diagnostic — never report a clean
      exit for an unrunnable launch (invariant I5).
- [ ] **MUST** honor a pinned/saved chain when it forms a valid path, appending the terminal if omitted.
- [ ] **SHOULD** expose the chain as an editable per-step control with cascading re-resolution (ch. 11 §11.7).
- [ ] **MUST** compose same-namespace links by command concatenation (pass-through forwards; a wrapper tool prepends),
      merging env innermost-wins. (ch. 11 §11.5)
- [ ] To support cross-platform nesting, **MUST**: mount inner-runner builds at `<CONTENT_ROOT>/__runner_<id>__`; derive
      (or read) the boundary's guest-path template (Wine: from `CONTENT_ROOT`); translate the inner runners' exe/args
      into guest paths; redirect the boundary's content target at the inner exe and append the inner guest args.
      (ch. 11 §11.6)

## 19.4 Variables

- [ ] **MUST** implement the substitution engine of chapter 8 §8.1 exactly: paired `%`, optional `:format` suffix,
      unmatched `%` stops, unknown token left in place, single non-recursive pass.
- [ ] **MUST** provide the built-in token table (ch. 8 §8.2) and the use-site render formats (ch. 8 §8.4:
      `dword`/`qword`/`bool`/`winpath`/`upper`/`lower`) at minimum.
- [ ] **MUST** resolve `CustomVar`s to **raw** values by priority override > saved > `DEFAULT` (a `secret`+`POOL` picks
      per launch; `UI` presence ⇒ a user option, absence ⇒ a binding; same-`KEY` override via dependency-chain order),
      and bind `%KEY%` — **before** expanding the layers that reference them.

## 19.5 The runtime

- [ ] **MUST** assemble a single overlay mount in the priority order of chapter 13 §13.1 (prefix base, inner-runner
      builds, content @ content-root, persist passthroughs, default-data, writable, post-mount overrides).
- [ ] **MUST** mount STORE-zip layers zero-copy, dir/file layers by reference, and **MUST** reject DEFLATE-compressed zip
      layers. (ch. 5, 13 §13.8)
- [ ] **MUST** generate a `PREFIX_GENERATE` runner's prefix once (via the runner's own launcher with a `wineboot`
      override), reuse it read-only, and never mutate it (invariant I6).
- [ ] **MUST** build the default-data layer from base (`OVERRIDE:false`) edits and apply `OVERRIDE:true` edits post-mount
      to the writable layer. (ch. 6, 13 §13.5)
- [ ] **MUST** keep source content / prefixes pristine, the assembled runtime ephemeral, and only declared persistence
      durable (invariant I6).

## 19.6 Persistence & save-safety

- [ ] **MUST** default to whole-runtime persistence when no `Persist*` is declared, and selective persistence otherwise.
      (ch. 7 §7.1)
- [ ] **MUST** seed selective persistence before mount and capture it before unmount, to/from `<bundle>/USERDATA` (or its
      per-launch override). (ch. 7)
- [ ] **MUST** tear down save-safely (invariant I7): capture while mounted; unmount durable-backed mounts non-lazily and
      verify them gone; **only then** wipe the ephemeral tree; abort the wipe if any durable mount is still live. (ch. 13
      §13.6)
- [ ] **MUST** recover stale runtimes from a previous crash before assembling a new one, under the same save-safety gate.
      (ch. 13 §13.7)

## 19.7 Content addressing

- [ ] **MUST** treat a present local file as authoritative and consult `SOURCE.CID` only to fetch a missing file
      (invariant I8), refusing a launch when a needed layer is neither present nor fetchable.
- [ ] **SHOULD** support hydrate (fetch the closure's CIDs to local paths) and dehydrate; **SHOULD** support `ipfs` for
      ecosystem interoperability; **MUST** support whatever backends appear in the packages it consumes.
- [ ] **SHOULD** support publish (write CIDs into nodes + seed bytes) and re-seed for distribution. (ch. 14)

## 19.8 Validation

- [ ] **SHOULD** implement the validator of chapter 15: graph integrity (errors), STORE-zip + VFS-path (errors),
      `CONTENTPATH` case-exactness + cross-layer case collisions (errors), and the various warnings (dir layers, runner
      layers, EXCLUDE symmetry, missing host/runner, prefix-without-drive_c).

## 19.9 Operations

- [ ] **SHOULD** expose a single data root with relocation (run modes / `--data-dir`) and place `USERDATA` with the
      bundle. (ch. 16)
- [ ] **SHOULD** enforce a single live instance per data root with a crash-safe lock. (ch. 16 §16.4)
- [ ] **SHOULD** provide a headless resolve/validate/launch surface; **SHOULD** provide a side-effect-light
      "resolve-only" dump for conformance testing. (ch. 16 §16.5)

## 19.10 Conformance testing against the reference

The cheapest high-confidence test: take a corpus of packages, run **resolve-only** on each launchable in both the
reference and your implementation, and diff the resolved output — the closure (ordered node ids), the runner chain
(ordered runner ids + the composed/cross-namespace target), the resolved variables, and the derived paths. Equal
resolutions across a broad corpus is strong evidence of conformance for everything short of the live mount/execute path
(which requires a real launch to exercise — chapter 11 §11.8). Add a few real launches (a native game, a Windows game, a
cross-namespace chain) to cover the runtime.

---

*End of the specification.* Back to the [README](../README.md) · [Glossary](00-glossary.md).
