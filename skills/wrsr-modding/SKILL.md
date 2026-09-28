---
name: wrsr-modding
description: "Verified knowledge and hard-won rules for modding Workers & Resources: Soviet Republic (WRSR, W&R, Steam app 784150). Covers Workshop items (building.ini, renderconfig.ini, script.ini, workshopconfig.ini, .mtl and .nmf files, media_soviet, workshop_wip), native DLL plugins for TesmioLoader and Republic Mod Loader and their Workshop packaging, and reverse engineering SOVIET64.exe and C3DDLL64.dll (finding code, hooks, signatures, porting addresses to a new game build). Use this skill whenever the user works on anything for this game, even when they only mention one of these files or folders, a new building, vehicle, train, resource or deposit, a tesmio or RML plugin, or a crash in SOVIET64.exe or C3DDLL64.dll. It prevents crashes others already debugged (BOM in ini files, a $ inside a comment, a missing MATERIALEMISSIVE, a wrong storage class, 13-byte prologues) and saves rediscovering the engine."
---

# Workers & Resources: Soviet Republic modding

wrsr-agents by TheRealMeowMeow00 and contributors, CC BY 4.0: https://github.com/TheRealMeowMeow00/wrsr-agents

What every WRSR task needs. Details live in `references/` next to this file:
open one only when the task needs it.

## The game

- Steam app 784150, Windows x64; Linux players run it under Proton.
  `SOVIET64.exe` is the game logic, a plain MSVC build. `C3DDLL64.dll` is the
  engine ("C3D"); its 2218 exports keep their MSVC-mangled names, so engine
  signatures come for free. No anti-cheat, no packer, no CFG.
- Current build (September 2026): v1.1.1.9, PE TimeDateStamp `0x6A3EB6AD`,
  10 308 608 bytes. v1.1.1.7 was `0x69C4098C`. The exe has no VERSIONINFO. A
  matching stamp is necessary, not sufficient: one rebuild kept the old stamp,
  so verify bytes before trusting any address.
- Content is plain files under `media_soviet/`, addressed by relative paths:
  - `buildings_types/<name>.ini`: 488 building definitions, the real
    specification of the format.
  - `vehicles/`, `trains/`, `airplanes/`, `ships/`, `helicopters/`: one folder
    per vehicle, each with a `script.ini`.
  - `buildings/*.nmf|*.mtl`, `resources/`, `editor/tool_<name>.png`.
  - `scripts/SOVIETInstructions.txt`: the authority for enums
    (`BUILDINGTYPE_*`, `RESOURCE_TRANSPORT_*`, `VEHICLETYPE_*`) and for the
    built-in script VM.
  - `soviet<Language>.btf`: language files, big-endian UTF-16.
- A `$TOKEN` exists only if the exe contains it as a string. Before using one,
  check the exe or grep the base game files for real usage and values.

## Three ways to mod

1. **Workshop items, no code**: buildings, vehicles, skins, landscapes.
   Your items live in `media_soviet/workshop_wip/<id>/`, subscribed ones in
   `<Steam library>/steamapps/workshop/content/784150/<id>/`. See
   `references/workshop.md`, `buildings.md`, `vehicles.md`.
2. **The built-in script VM** (Pyrois, `WORKSHOP_ITEMTYPE_SCRIPT`): a
   scenario API that reads the world, sets money and prices, starts events. It
   cannot change simulation rules, add resources or hook anything.
3. **Native plugins**: DLLs running inside `SOVIET64.exe`. A loader starts the
   game suspended and injects its runtime into the process; the two in use are
   **TesmioLoader** and **Republic Mod Loader (RML)**, both GPL-3.0. They
   differ in packaging and services, not in principle, and RML also hosts
   TesmioLoader API 3/4 plugins. See `references/native-mods.md`.

## Rules that prevent the known failures

1. Files the game or a loader reads are UTF-8 **without BOM**. PowerShell 5.1
   `-Encoding UTF8` writes a BOM; the `[section]` header then stops matching
   and every setting silently falls back to its default. Write with
   `[IO.File]::WriteAllText($path, $text, (New-Object Text.UTF8Encoding($false)))`
   and keep the file's existing line endings.
2. `building.ini` and `.mtl` have **no comment syntax**: a `$TOKEN` anywhere
   on a line is parsed, even after `//` or `;`. Never write `$` inside a
   comment. The tell is `ResourceGet - not found <an ordinary English word>`
   in the log, followed by a crash.
3. In PowerShell, double quotes interpolate `$`: `"$TYPE_SHOP"` is an empty
   string. Use single quotes for anything containing a token.
4. Clone, do not invent. Start from the closest base-game donor and change
   only what you must. Geometry lines (`$CONNECTION_*`, `$COST_WORK*`,
   `$VEHICLE_STATION`, `$PARTICLE`, `$TEXT_CAPTION`) are measured against the
   donor's mesh: keep them verbatim.
5. A storage's transport class must match its resource's class, or it reads
   `0.00 of 0.00 t` and never fills.
6. If the donor has `buildings/<donor>_e.mtl`, the clone must declare it as
   `MATERIALEMISSIVE`, or the first frame crashes in `C3DDLL64.dll + 0xAC544`.
7. Adding or removing resources changes the save format, and some plugins
   write into saves or the terrain. Back up `media_soviet/save/` before
   testing anything that touches the simulation.
8. The game holds loader DLLs and logs open. Close it before replacing a DLL;
   read a live log with `FileShare.ReadWrite`.
9. Native code: addresses are RVAs (ASLR is on). Verify the expected bytes at
   every patch site before writing and refuse on a mismatch. Never trust an
   address recorded for a different file: re-derive against the exe on disk.
10. Native code: prefer, in this order, an import-table swap (`SOVIET64.exe`
    and `C3DDLL64.dll` each have one), a vtable slot, a data pointer, one
    instruction's operand, a call site's rel32, and only then an inline hook or
    spliced code. Never overwrite a shared `.rdata` constant: MSVC pools float
    literals, and `150.0f` has about 190 readers. Repoint the one
    instruction's disp32 at your own float, allocated within +-2 GB.
11. An inline hook needs at least 14 stolen bytes ending on an instruction
    boundary (absolute jump `FF 25 00000000` plus the address), and no
    RIP-relative instruction can be stolen. `/GS` prologues are 13 bytes
    followed by the cookie load: patch the call site instead.
12. A clean log line ("hook ok", "patched") proves the bytes matched, not that
    the behaviour changed. The game often has a second copy of the same logic,
    and the UI can recompute a value on its own path. When a verified patch
    has no effect, look for the twin.
13. The game is the only oracle. Report what was built, what was observed in
    game, and what was not run.

## Finding things, cheapest first

1. This skill and its `references/`.
2. The game's own files: grep `buildings_types/*.ini` or `*/script.ini` for a
   token and read one donor instead of many.
3. Published research: TesmioLoader `docs/` (engine internals; its
   `02-findings.md` quotes v1.1.1.7 addresses, the current ones are the
   `#define`s in its plugin sources), RailPhysics `docs/` (rail), the railspeed
   and RML sources. Read them before re-deriving anything.
4. Reverse engineering: PE and `.pdata` for function bounds, string
   cross-references to find code, runtime hooks to learn behaviour, and Ghidra
   (headless, on an already analysed project) only to read a decision. See
   `references/reverse-engineering.md`.

## Etiquette

Tell your user about these before helping them publish:

- Credit what the mod is built on: name it and link it in the Workshop
  description and the README. Keep license headers intact.
- The loaders and most plugins are GPL-3.0. A work based on one, including a
  port to another language or an AI rewrite of its code, must stay GPL-3.0,
  ship its source, keep the notices and say it was modified.
- Reimplementing an idea from scratch is fine. Presenting someone else's work
  as your own is not, and neither is re-uploading someone else's Workshop item.
- Saying that AI helped is good manners.

## References

| File | Read it when |
|---|---|
| `references/workshop.md` | making, packaging or uploading any Workshop item |
| `references/buildings.md` | writing `building.ini`, `renderconfig.ini`, `.mtl`; cloning a building |
| `references/vehicles.md` | vehicles and trains, `script.ini` |
| `references/native-mods.md` | writing, packaging or installing a DLL plugin for either loader |
| `references/reverse-engineering.md` | finding code, porting addresses to a new build, patching |
| `references/internals.md` | engine structures: resources, buildings, electricity, citizens, walking, clocks, deposits, rail, language files |
| `references/pitfalls.md` | something crashes, or silently does nothing |

## Working economically

- Do not re-derive what this skill states. Open the one reference file the
  task needs, not all of them.
- Measure before reading: count token usage across files, check a file's size,
  grep for the one line.
- In reverse engineering, find by strings and cross-references before
  decompiling, and decompile only the function in question.
- When you learn something, write it down in the project, observed and
  inferred kept apart, so the next session does not pay for it again.
