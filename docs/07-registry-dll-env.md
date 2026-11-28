# 07 · `REG`, `DLL`, `ENV`

These layers write facts that are not files: registry values, DLL load policy and the process environment. Each is
composed per address like everything else (chapter 05 §5.4, chapter 12): the later writer wins, `null` deletes, and
every address is written whole.

## 7.1 `REG`: registry keys and values

```jsonc
{ "REG": { "HKCU\\Software\\nipkow\\SH2EEsetup":       { "lang": "en", "fullscreen": "dword:00000001" },
           "HKCU\\Software\\nipkow\\SH2EEsetup\\Paths": { "install": "%GameDir%" },
           "HKCU\\Software\\nipkow\\Old":              null },
  "ARCH": ["64"] }
```

The payload maps **key paths** to what the key holds. A key path is written as Windows writes it: a hive (`HKLM`,
`HKCU`, `HKCR`, `HKU`, or their long names), then key names, separated by backslashes (doubled in JSON, as in any
Windows path written there). A `/` inside a key name is part of the name. For each key:

- an **object** is its values, `{name: data}`; the **empty name `""`** is the key's default value (what `regedit` shows
  as `@`); a value written `null` is deleted;
- an **empty object** `{}` creates the key with no values;
- **`null`** deletes the key, with its values and everything beneath it, as written by layers beneath. It does not
  reach into the runner's own default registry.

A key holds values only: a subkey is its own path, and a value whose data is an object or a list is refused. Values
use Wine `.reg` encodings: a bare string (`"en"`), `"dword:0000000a"`, `"hex(b):…"`. Drive them from a variable rendered
at the use site: `"%FULLSCREEN:dword%"`, `"%MASK:qword%"` (chapter 08 §8.4). A path in a value is written from an anchor
(`"%GameDir%"`, `"%ProgramFiles32%\\X\\x.ax"`); the runner substitutes its guest path.

**`KEY`** (optional) is the key the layer's paths are under: `{"KEY": "HKLM\\Software\\Classes", "REG": {"CLSID\\{…}":
{…}}}` writes `HKLM\Software\Classes\CLSID\{…}`. It only shortens a large layer (a codec's thousands of class keys);
the addresses are the full paths. A path `""` under a `KEY` is that key itself.

**`ARCH`** is a list of `"32"` and/or `"64"`: the bitness of the program that reads the keys. The keys are written once
per listed architecture; for `"32"` on a 64-bit prefix the runtime inserts `Wow6432Node` after `Software` so the key
lands where a 32-bit program looks. Never write `Wow6432Node` yourself. `ARCH` is **required**: a `REG` layer without
it is refused, because a 32-bit program's `HKLM\Software` keys written to the 64-bit view are silently never read. A
kept key names its view the same way (`KEEP` … `{"ARCH": ["32"]}`, chapter 13 §13.2).

**Addresses** are `REG/<key path>` for a key and `REG/<key path>#<value>` for a value, compared without case, per
architecture: `REG/HKCU\\Software\\nipkow\\SH2EEsetup#lang`. A key name may contain `#`, a value name never: the
value starts after the **last** `#`. A later layer's write of a value replaces that value; a `{}` or `null` at a key
replaces the key.

The registry exists only for a runner that generates a prefix (chapter 10 §10.7). Under any other runner `REG`
layers have nothing to write to; an implementation SHOULD say so in the launch log rather than drop them silently.

*(VidyaGod: `Fold`'s `FoldReg`; `RegistryWrapper::ApplyRegEdits` writes the composed keys.)*

## 7.2 `DLL`: Wine DLL load policy

```json
{ "DLL": { "d3d8": "n,b", "dinput8": "n,b", "winegstreamer": "" } }
```

A map of DLL name → load order: `"n,b"` (native then builtin), `"b,n"`, `"n"`, `"b"`, or `""` (disabled). `null`
removes a policy set beneath. Names compare without case. The composed map becomes `WINEDLLOVERRIDES` for the launch;
an override the user already has in their environment is kept and placed last, so a person debugging wins over the
package.

- **An empty order means disabled.** It is not an omission to default to `"n,b"`.
- A value MAY carry a multi-DLL spec (`"n,b;dinput8=n,b"`); an editor offering the common orders as a menu MUST let
  such a value through unchanged.
- Only a prefix-generating runner reads it; elsewhere it is reported as unused.

## 7.3 `ENV`: the process environment

```json
{ "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%SteamAppId%", "LD_LIBRARY_PATH": null } }
```

A map of name → value. Values MUST be strings (quote a number); `%TOKEN%`s are expanded at launch. `null` removes
the name: from what layers beneath set, and from the host environment the process would otherwise inherit.

**Composition.** Like any address: the later layer wins a name, a later set undoes an earlier removal and a later removal
undoes an earlier set. A ticked graft's `ENV` composes above the variant's, so a mod loader that needs a variable
declares it.

**Along the runner chain.** Each runner resolves its own node, so each link has its own composed environment. At
launch the host environment is taken (with `HOME` and the XDG base directories pointing at the launch's private home,
where the runtime gives it one: chapter 14 §14.5), then the links are applied from the outermost inwards, so **the
innermost (closest to the content) wins a shared name**, and the game's own `ENV` is applied last, over every runner. A game's `null` removes a
name even if its runner sets it. An `EXEC` entry carries no environment; how a program needs to be
started is a fact of the node, so it is a layer.

*(VidyaGod: `Fold`'s `ENV` fold; `LaunchResolver::ApplyInnerEnv` for runners nested inside the boundary;
`ContainerWrapper::Execute` composes the process environment.)*

Next: [Variables & `WHEN`](08-variables.md).
