# 19 · Reference implementation map

[VidyaGod](https://github.com/lorenzo-zurini/VidyaGod) is the reference implementation (Qt 6, C++23, Linux and
Windows). This chapter maps chapters to the source that implements them, for cross-checking and porting. The pointers
are illustrative, not normative, and names drift.

| Chapter | Source | Key symbols |
|---------|--------|-------------|
| 02 Nodes | `src/cid.cpp`, `src/nodegraph*.cpp`, `src/manifestmodel.cpp` | `Cid::Canonical`, `Cid::OfNode`; `NodeGraph::GatherWorkingTree`, `FreezeToIndex`; `ParseNode`; `NodeIndex::Duplicates` (one home) |
| 03 Layers | `src/nodelower.cpp` | `CheckNode` / `CheckLayer` (the vocabulary), `LowerNode`, `LowerPlan`, `LowerEntry` |
| 04 Content | `src/manifestmodel.cpp`, `src/vfsmount.cpp`, `external/VidyaGodFS` | `LayerLocator`, `IsRunnerBuildLayer`, `ZipFullyStored`, `BuildLayerSpec`; `vgdelta` for deltas |
| 05 `NODE` layers · 12 Resolution | `src/fold.cpp` | `Fold::Resolve` (phases, expansion, occurrences, the fold), `TakeView`, `DecidingMentions`, `BuildGraftIndex`, `OfferedGrafts`, `ApplyGrafts`, `Unsatisfied`. Reference semantics: `tools/gen6/resolve.py`, held to the C++ by `tools/gen6/fold_gate.py` |
| 06 `EDIT` | `src/fileedits.cpp`, `src/binarypatch.cpp` | `ProcessFileEdits` (`ConfigWrite`, `JsonSet`, `Overwrite`, `AppendLine`), `ProcessBinaryPatches`, `BinaryPatch::ApplyOne`; `vglobby --mem-patch` for `APPLY: "memory"` |
| 07 `REG` `DLL` `ENV` | `src/fold.cpp`, `src/registrywrapper.cpp`, `src/registrylayer.cpp`, `src/containerwrapper.cpp` | `FoldReg`, `ApplyRegEdits`, `BuildDefaultData`; the environment composed in `ContainerWrapper::Execute` |
| 08 Variables | `src/varsubst.cpp`, `src/launchparams.cpp`, `src/fold.cpp` | `StringVariableSubstitution`, `RenderValue`, `EvaluateCondition`, `EvaluateInteger`; `ContainerParams::GetVariablesMap`; `Fold::ResolveVars` |
| 09 `EXEC`, tiles | `src/fold.cpp`, `src/manifestmodel.cpp`, `src/launchresolver_exec.cpp` | the per-field `EXEC` fold (`Merge`); `DeriveFacts` (faces, the tile table, families); `ResolveExecutableDefinition` |
| 10 Runners | `src/manifestmodel.cpp`, `src/packagecatalog.cpp`, `src/runnerinstall.cpp` | `ServesPlatform`, `MachinePlatform`; `RunnerInstalled`, `HasUsableChain`, `CandidateRunners`; `RunnerBuildPresent` |
| 11 Chaining | `src/launchresolver_chain.cpp`, `src/vfsmount.cpp`, `src/prelaunchwindow.cpp` | `ResolveChainIds`, `ResolveChainTail` (installed first), `FindBridge`, `ComposeGuestTarget`, `ApplyInnerEnv`, `InnerRunnerMountRel`; the chain combos |
| 13 Ownership | `src/nodelower.cpp`, `src/launchresolver_persist.cpp`, `src/persistlayer.cpp`, `src/registrylayer.cpp` | `UserOwned`, take-backs in `LowerPlan`; `DerivePersistence`; `CapturePersistFiles`, `CapturePersistRegKeys` |
| 14 Runtime | `src/vfsmount.cpp`, `src/containerwrapper.cpp`, `src/sandboxlayer*.cpp`, `external/VidyaGodFS` | `BuildLayerSpec`, `BuildContainerRuntime`, `Execute`, `Cleanup`, `CleanStaleRuntime`; bubblewrap / Sandboxie; `vidyagodfs` |
| 15 Packages & sharing | `src/packagecatalog*.cpp`, `src/appmodel.cpp`, `src/ipfswrapper.cpp`, `external/VidyaGodIPFS` | `BuildCatalogIndex`, `PublishLibrary` (rows with `needs`), `SharedWith`, `AppModel::pushSharesTo`, `SeedNodeContent`, `PlanReceivedFetches`, `PruneStaleReceived`, `AdoptReceivedPackage`; versions: `AvailableUpdates`, `StageUpdate`, `ApplyStagedUpdate`, `ContentFile::Path` (bumped names, `src/contentfile.cpp`); the embedded Boxo node and its one rolling fetch queue |
| 16 Validation | `src/manifestmodel.cpp`, `src/cli/cliaudit.cpp` | `ValidateNodeGraph`, `FindCrossLayerCaseCollisions`; `--audit-packages` |
| 17 CLI, instances | `src/main.cpp`, `src/cli/*.cpp`, `src/apppaths.cpp`, `src/instancestore.cpp` | argument parsing, `AcquireSingleInstanceLock`, `DumpResolution`; `AppPaths::DataRoot` and the overrides; `InstanceStore` |
| Editor | `src/pkgdoc.cpp`, `src/pkgcanvas.cpp`, `src/pkggraph.cpp`, `src/pkgactions.cpp` | the package document (handles, undo, save = re-mint with the library cascade), the canvas, the field tables |

## Architectural notes (non-normative)

- **The launch engine** is split into single-purpose units: `fold` (resolution), `nodelower` (the plan in the
  engine's vocabulary), `launchresolver*` (chain, paths, variables, ownership), `vfsmount` (overlay assembly),
  `registrylayer` / `persistlayer` / `fileedits` / `binarypatch` (application), with `containerwrapper` as the session
  orchestrator.
- **The overlay is its own project**: `vidyagodfs` (a submodule) mounts the JSON layer spec: union, zip by offset,
  deltas, part-of-a-source mounts, passthroughs, with a watchdog that unmounts if the launcher dies.
- **The content backend is its own project**: `VidyaGodIPFS` (a submodule, Go/Boxo), an in-process IPFS node behind a
  small C ABI: storage, fetch, seeding, friends.
- **Testing.** Unit tests on synthetic graphs; `tools/launch_matrix.py` (a fixture covering every launch feature, with
  plan goldens from `--resolve-only` and a real `--node` run whose probe reports what it saw); `tools/mutate.sh`
  (every test must fail when the code it guards is broken). Comparing `--resolve-only` dumps for the same packages is
  the recommended way to test a new implementation against the reference.

Next: [Conformance](20-conformance.md).
