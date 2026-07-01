# 18 · Reference implementation map

[VidyaGod](https://github.com/lorenzo-zurini) is the reference implementation (Qt 6 / C++23). This chapter maps spec
chapters to the source that realizes them, so a reader can cross-check behavior against working code or port it. **These
pointers are illustrative, not normative** — the spec defines the format; this table says where one program implements it
(file/function names current as of writing; they drift).

| Spec chapter | Reference source | Key symbols |
|--------------|------------------|-------------|
| 02 Nodes · 04 Indexing | `src/manifestmodel.{h,cpp}` | `Node`, `NodeIndex`, `ParseNode`, `ScanBundleNodes`, `BuildNodeIndex` |
| 03 Roles | `src/manifestmodel.h` | `Node::IsLaunchable/IsRunner/Presentable/GameKey` |
| 04 Repos / sync | `src/packagecatalog.cpp` | `SyncGitRepository`, `SyncRepositories`, `RepositoryDirs` |
| 05 VFS layers | `src/manifestmodel.cpp`, `src/vfsmount.cpp` | `IsVfsLayer`, `LayerType`, `LayerLocator`, `ResolveLayerSource`, `ZipFullyStored`, `BuildLayerSpec` |
| 06 Edit layers | `src/fileedits.cpp`, `src/registrylayer.cpp`, `src/registrywrapper.cpp` | `ProcessFileEdits` (`ConfigWrite`/`FileOverwrite`/`AppendLine`), `ProcessDLLOverrides`, `ApplyRegEdits`, `BuildDefaultData`, `ApplyOverrideRegEdits` |
| 07 Persistence | `src/persistlayer.cpp`, `src/registrylayer.cpp`, `src/launchresolver.cpp` | `DerivePersistence`, `SeedPersistFiles`/`CapturePersistFiles`, `SeedPersistRegistry`/`CapturePersistRegistry`, `CapturePersistRegKeys` |
| 08 Variables / CustomVar | `src/varsubst.cpp`, `src/launchparams.cpp`, `src/launchresolver.cpp` | `StringVariableSubstitution`, `TranslateCustomVarValue`, `ContainerParams::GetVariablesMap`, `ResolveCustomVariables` |
| 09 EXEC | `src/launchresolver.cpp`, `src/containerwrapper.cpp` | `ResolveExecutableDefinition`, `ContainerWrapper::Execute` |
| 10 Platforms / runners | `src/manifestmodel.cpp`, `src/packagecatalog.cpp`, `src/runnerwrapper.cpp` | `MachinePlatform`, `CompatibleRunners`, `RunnerInstalled`, `RunnerWrapper::ExecutableAvailable`/`DefPrefixDir` |
| 11 Daisy-chaining | `src/launchresolver.cpp`, `src/vfsmount.cpp`, `src/containerwrapper.cpp` | `ResolveChainIds`, `ResolveChainTail`, `ResolveRunnerChain`, `RunnerAvailable`, `BoundaryLinkIndex`, `GuestPath`, `ComposeGuestTarget`, `InnerRunnerMountRel`; `BuildLayerSpec` (inner mounts); `Execute` (nested command) |
| 11 Chain UI | `src/prelaunchwindow.cpp` | `RebuildRunnerChain`, `RenderChainCombos`, `onChainStepChanged`; `PackageCatalog::CandidateRunners` |
| 12 Resolution | `src/manifestmodel.cpp` | `ResolveNodeOrder`, `OptionalNodes` |
| 13 Runtime model | `src/vfsmount.cpp`, `src/containerwrapper.cpp`, `src/registrylayer.cpp`, `src/launchresolver.cpp` | `BuildLayerSpec`, `MountVFS`, `MountRunnerBuild`, `InitializeDefPrefix`, `DerivePaths`, `ContainerWrapper::BuildContainerRuntime`/`Cleanup`, `CleanStaleRuntime` |
| 13 Overlay filesystem | `VidyaGodFS` (`vidyagodfs`) | the FUSE union/zip/file mounter driven by the JSON layer spec |
| 14 Content addressing | `src/packagecatalog.cpp`, `src/ipfswrapper.cpp`, `src/launchsources.cpp`, `VidyaGodIPFS` | `PublishPackage`, `SeedDirectory`, `MirrorDehydrated`, `HydrateNode`/`DehydrateNode`, `NodeContentCids`, `EnsureSources`/`MaterializeLayers`; embedded Boxo/IPFS node |
| 15 Validation | `src/manifestmodel.cpp` | `ValidateNodeGraph`, `GatherLaunchContentFiles`, `FindCrossLayerCaseCollisions` |
| 16 Run modes / CLI / paths | `src/main.cpp`, `src/apppaths.{h,cpp}` | argument parsing, `AcquireSingleInstanceLock`, `DumpResolution`; `AppPaths::DataRoot/Mode/*Override` |
| 16 Saved settings | `src/packagecatalog.cpp` | `GetPackageUserSettings`, `SetPackageUserSetting` (`RUNNER_CHAIN`/`VARIABLES`/`MODULES`) |

## Architectural notes (non-normative)

- **Launch engine split.** The launch pipeline is decomposed into single-purpose units: `launchparams` (the resolved
  parameter struct + token map), `launchresolver` (closure/chain/path/variable/persistence resolution), `varsubst`
  (the substitution engine), `vfsmount` (overlay assembly), `registrylayer`/`persistlayer`/`fileedits` (layer
  application), `launchsources` (content materialization), `runnerinstall` (runner build + prefix install), with
  `containerwrapper` as the thin session orchestrator. This mirrors the spec's chapter boundaries.
- **The overlay is a separate project.** `vidyagodfs` (its own repo, a submodule) is the FUSE filesystem that mounts the
  layer spec — overlay + zip-by-offset + file/dir passthrough in one mount, with a watchdog that auto-unmounts if the
  launcher dies. The format depends on the *semantics* (chapter 13 §13.8), not on this particular filesystem.
- **The content backend is a separate project.** `VidyaGodIPFS` (a submodule, Go/Boxo) is an in-process IPFS node
  exposed to the C++ via a small C ABI; it provides storage, content-addressed fetch, and P2P seeding.
- **Testing surface.** `--resolve-only` dumps resolved parameters (including the runner chain) for golden-comparison;
  the resolver's pure decisions (closure ordering, chain BFS, cross-namespace composition, guest-path translation) are
  unit-tested with synthetic graphs. This is the recommended way to conformance-test a *new* implementation against the
  reference: compare resolution dumps for the same package set.

Next: [Conformance](19-conformance.md).
