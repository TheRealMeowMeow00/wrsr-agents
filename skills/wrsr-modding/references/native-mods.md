# Native plugins

Part of wrsr-agents (CC BY 4.0). Sources: TesmioLoader (MaxLegend, GPL-3.0)
`docs/01-architecture.md`, `docs/09-plugins.md` and `src/tesmio_api.h`;
Republic Mod Loader (Ultimate-Universe, GPL-3.0) `README.md` and
`Source/docs/`; TLMM (TheRealMeowMeow00, GPL-3.0) `README.md`; WRSR Mail
(TheRealMeowMeow00, GPL-3.0) for adding a resource. Versions as of November 2026.

## What a loader is

The game has no plugin interface. A loader starts `SOVIET64.exe` suspended,
gets its own runtime DLL into the process, loads the enabled plugin DLLs and
lets the game run. From then on a plugin is ordinary native code in the game's
address space: there is no sandbox, and a plugin can corrupt the process as
easily as the loader can. Both loaders below work this way; they differ in how
they find plugins and in what services they offer. A plugin written against
TesmioLoader's API runs on both.

| | TesmioLoader | Republic Mod Loader (RML) |
|---|---|---|
| Source | `github.com/MaxLegend/TesmioLoader` | `github.com/Ultimate-Universe/WRSR-RepublicModLoader` |
| Install | a release unpacked into `<game>/tesmioloader/`; TLMM installs and updates it | Workshop item 3787969749 |
| Start | `tesmioloader/build/tesmiolauncher.exe` | `RepublicModLoader.exe` from that item |
| Plugins | `tesmioloader/build/plugins/<name>.dll` plus `<name>.ini` | DLLs in a `plugin/` or `plugins/` folder of enabled Workshop items (root or one level down), `workshop_wip` projects, standalone DLLs |
| Native API | Tesmio API 4, accepts 3 and 4 | RML API 1, plus a compatibility host for Tesmio API 3/4 |
| Game build | refuses to inject into a build it does not know | tested on 1.1.1.9; keeps a symbol cache per exe SHA-256 |
| Extras | VFS file redirection, save manifest, crash handler, plugin signing | playlists, per-session Workshop filtering, hook-chain tracking, diagnostic logging |

A third one, `soviet-mod-loader` (PresFox), is a prototype: `mods/<id>/mod.ini`
plus `bin/*.dll`, with an optional `SovietModInitialize` export. Some mods ship
external patchers instead, which write into the running game from another
process; they need no loader and get none of a loader's checks.

## TesmioLoader in practice

- `tesmioloader/build/` holds `tesmiolauncher.exe` (run this, not the game),
  `tesmioloader.ini` (master switches, plus a `[plugins]` section of
  `name = 0|1` where an absent key means on), `plugins/*.dll` with one `.ini`
  each, and `tesmioloader.log`.
- The launcher reads the exe's version and stamp and refuses builds it does
  not know. `--find` prints what it resolved and exits. `version_check = 0`
  is for people porting addresses, not for playing.
- VFS: a read of `media_soviet/<path>` is served from the loader's VFS root
  when the same relative path exists there. The root is `tesmioloader/vfs/`
  when that folder exists, otherwise `tesmioloader/build/vfs/`; plugins get it
  as `TsmHost::vfsRoot`. Absolute paths, which the game uses for Workshop
  content, are not redirected.
- Save manifest: `tesmioloader.save.ini` beside each save's `stats.ini` lists
  the mod content in effect, and a save whose mods are missing warns before it
  loads.
- Crash handler: logs `module + rva`, registers and return addresses. Read the
  earliest `=== CRASH` or `faulted` line, not the last.

## Writing a TesmioLoader-API plugin

```
plugins/<name>/<name>.cpp     the folder name decides the DLL name
plugins/<name>/<name>.ini     optional, copied beside the DLL
```

```cpp
#include "../../src/tesmio_plugin.h"   // header-only helpers vendored from TesmioLoader

extern "C" __declspec(dllexport) unsigned TsmPluginApiVersion(void) { return TSM_API_VERSION; }

extern "C" __declspec(dllexport) int TsmPluginInit(const TsmHost* host, TsmPluginInfo* info)
{
    TsmBind(host);
    info->name = "myplugin"; info->version = "1.0";
    if (!H->configInt("plugins\\myplugin.ini", "myplugin", "enabled", 1))
        return 1;                                   // declined, nothing hooked
    return 0;
}

// optional; runs after every plugin's Init
extern "C" __declspec(dllexport) int TsmPluginStart(void) { return 0; }
```

- `TsmPluginApiVersion` must do nothing but return the constant. It is called
  before `Init`, so an incompatible plugin is refused before it reads a moved
  field.
- Two phases: `Init` reads config and `provide`s services; `Start` `consume`s
  services and installs hooks. Every `Init` runs before any `Start`, so the
  (alphabetical) load order does not matter.
- Return non-zero only if nothing was hooked: a plugin with a live hook can
  never be unloaded.
- `TsmHost`, API 4: `exeModule`, `exeBase` (add RVAs to it), `exeSize`,
  `engineModule` (`C3DDLL64.dll`), `baseDir`, `pluginDir`, `log`,
  `findIatSlot`, `patchIat`, `installInlineHook` (compares the expected bytes,
  refuses under 14 stolen bytes), `allocNear` (executable memory within
  +-2 GB), `readablePtr`, `faultFilter` (the filter for a `__try` around reads
  of game structures), `configInt`, `configString`, `provide`, `consume`, and
  `vfsRoot` (appended in API 4; check `structSize` before reading it).
- Build `/MT` (MSVC) or `-static` (llvm-mingw clang): each DLL has its own
  CRT, so nothing may be freed across the boundary.
- One settings ini per plugin, beside the DLL, section `[<name>]`, UTF-8
  without BOM, read through the ANSI profile API: keep settings ASCII.
- `tesmio_plugin.h` is compiled into your DLL, which makes the DLL a GPL-3.0
  work.

## Adding a resource from a plugin

The method comes from TesmioLoader's `resources` plugin, and it is GPL-3.0.

1. Hook the resource lookup. Publish on its first call with the base list present,
   before the parsers resolve names: `$RESOURCE_ALLOW_ONLY` and building tokens need
   the record to exist by then.
2. Clone a template record into the vector's **free capacity**, set the name and the
   caption id, and move `end` only after the records are complete. Never relocate the
   vector: other code and plugins hold pointers into it. Short of room, add nothing
   and say so in the log.
3. A map load clears the vector with `end = begin`: treat a changed `end` as the
   signal to look again, and drop every cached `Resource*` then.
4. Serve the caption through a `GetString` import swap, on private ids above the
   game's. To pick a language, WRSR Mail looks at the game's own text for a known id;
   that was not yet run on a non-Russian game.
5. Give it an icon at `resources/<name>.png`, and a price: see `services.md`.

Other plugins may hook the same lookup (Space Race 3814051864 ships a `resources.dll`
at that address) or fill the same capacity. Test alone as well: free slots with only
your plugin are what a stranger gets. A real need does not come with it: the game's
citizen demands run through shops on four literal goods, so a new consumption has to
be driven by your plugin.

`TsmPluginInit` must return 0 once a hook is installed: a non-zero return unloads the
DLL and crashes the game.

## RML-native plugins

```cpp
#define RML_PLUGIN_BUILD
#include "rml_api.h"

RML_EXPORT int RML_CALL RmlPluginQuery(uint32_t host_api_version, RmlPluginDescriptor* out);
RML_EXPORT int RML_CALL RmlPluginLoad(const RmlHostApi* host);
```

`RmlPluginQuery` fills the descriptor (id, name, version, author) and declares
the API range it accepts. The host offers `log`, `get_game_info` (version, exe
SHA-256, image base), `get_symbol` by semantic id (for example
`wrsr.vehicle.task_advance`, with a confidence), and `provide_service` /
`consume_service`. RML API 1 has no hook API: a native plugin installs its
own hooks and is responsible for them. Plugins stay loaded for the life of the
process. Details: RML's `Source/docs/PLUGIN_SDK.md`.

A Tesmio-API plugin under RML is hosted by a clean-room compatibility layer
that chains inline hooks when several plugins hook the same target: a later
hook must match the current bytes and steal at least 14 bytes, and no more
than the first owner did. Test a plugin on both loaders; file and config
paths are where they are most likely to differ.

The runtime log is `rml/logs/rml-runtime.log` in the RML item's folder, crash reports
in `rml/crash-archives`.

## Shipping a plugin on the Workshop

- Put the DLL and its ini in `plugins/` at the item root: RML finds it in
  place, and TLMM copies it into TesmioLoader's plugin folder.
- Files for TesmioLoader's VFS go in `vfs/`.
- Additions to another plugin's ini, such as a resource declaration, go in a
  `<target>.ini.merge` fragment. TLMM merges it and never overwrites a key the
  user already has.
- Ship the source or link to it: GPL-3.0 requires it for anything built on
  these loaders' headers.
- Say in the description which loaders and game build it was tested on, and
  whether it touches saves.

## Linux

Plugins are Windows DLLs and run unchanged inside the game's Proton prefix.
What Linux needs is Steam starting the game through the loader: TLMM puts a
wrapper in the launch options that swaps the target for
`tesmiolauncher.exe --nogui`. Steam's default target for this game is
`SETUPAPPLICATION SOVIET.exe`, not `SOVIET64.exe`.
