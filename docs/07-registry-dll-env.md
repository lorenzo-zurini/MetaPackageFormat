# 07 · `REG`, `DLL`, `ENV`

These layers write facts that are not files: registry values, DLL load policy and the process environment. Each folds
per address like everything else (chapter 12): the later writer wins, `null` deletes.

## 7.1 `REG`: the registry tree

```jsonc
{ "REG": { "HKCU": { "Software": { "nipkow": { "SH2EEsetup": {
           "lang": "en",
           "fullscreen": "dword:00000001",
           "Paths": { "install": "%GameDir%" } } } } } },
  "ARCH": ["64"] }
```

The payload is a tree hanging off hive names (`HKLM`, `HKCU`, `HKCR`, `HKU`, `HKCC`). Inside it:

- an **object** is a subkey; a **string** is a value; a key may hold both;
- an **empty object** creates the key with no values;
- the name `"@"` is the key's default value;
- **`null` deletes**, from the fold: the value of that name, and a subkey of that name with everything beneath it,
  as written by layers beneath. It does not reach into the runner's own default registry.

Values use Wine `.reg` encodings: a bare string (`"en"`), `"dword:0000000a"`, `"hex(b):…"`. Drive them from a
variable rendered at the use site: `"%FULLSCREEN:dword%"`, `"%MASK:qword%"` (chapter 08 §8.4). A path in a value is
written from an anchor (`"%GameDir%"`, `"%ProgramFiles32%\\X\\x.ax"`); the runner substitutes its guest path.

**`ARCH`** is a list of `"32"` and/or `"64"`: the bitness of the program that reads the keys. The tree is written once
per listed architecture; for `"32"` on a 64-bit prefix the runtime inserts `Wow6432Node` after `Software` so the key
lands where a 32-bit program looks. Never write `Wow6432Node` yourself. No `ARCH` means no redirection: the tree is
written as given.

Addresses are `REG/<hive>/<key…>/<value>`, compared without case, per architecture. A value name and a subkey name
cannot be the same in one key; a tool that flattens the tree for editing MUST refuse to write one over the other.

The registry exists only for a runner that generates a prefix (chapter 10 §10.7). Under any other runner `REG`
layers have nothing to write to; an implementation SHOULD say so in the launch log rather than drop them silently.

*(VidyaGod: `Fold`'s `FoldReg`; `RegistryWrapper::ApplyRegEdits` writes the composed hives.)*

## 7.2 `DLL`: Wine DLL load policy

```json
{ "DLL": { "d3d8": "n,b", "dinput8": "n,b", "winegstreamer": "" } }
```

A map of DLL name → load order: `"n,b"` (native then builtin), `"b,n"`, `"n"`, `"b"`, or `""` (disabled). `null`
removes a policy set beneath. Names compare without case. The folded map becomes `WINEDLLOVERRIDES` for the launch;
an override the user already has in their environment is kept and placed last, so a person debugging wins over the
package.

- **An empty order means disabled.** It is not an omission to default to `"n,b"`.
- A value MAY carry a multi-DLL spec (`"n,b;dinput8=n,b"`); an editor offering the common orders as a menu MUST let
  such a value through unchanged.
- Only a prefix-generating runner reads it; elsewhere it is reported as unused.

## 7.3 `ENV`: the process environment

```json
{ "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%", "LD_LIBRARY_PATH": null } }
```

A map of name → value. Values MUST be strings (quote a number); `%TOKEN%`s are expanded at launch. `null` removes
the name: from what layers beneath set, and from the host environment the process would otherwise inherit.

**The fold.** Like any address: the later layer wins a name, a later set undoes an earlier removal and a later removal
undoes an earlier set. A ticked graft's `ENV` folds above the variant's, so a mod loader that needs a variable
declares it.

**Along the runner chain.** Each runner resolves its own node, so each link has its own folded environment. At
launch the host environment is taken, then the links are applied from the outermost inwards, so **the innermost
(closest to the content) wins a shared name**, and the game's own `ENV` is applied last, over every runner. A game's
`null` removes a name even if its runner sets it. An `EXEC` entry carries no environment; how a program needs to be
started is a fact of the node, so it is a layer.

*(VidyaGod: `Fold`'s `ENV` fold; `LaunchResolver::ApplyInnerEnv` for runners nested inside the boundary;
`ContainerWrapper::Execute` composes the process environment.)*

Next: [Variables & `WHEN`](08-variables.md).
