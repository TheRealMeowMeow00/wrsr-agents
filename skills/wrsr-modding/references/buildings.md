# Buildings

Part of wrsr-agents (CC BY 4.0). Sources: the 488 base-game
`buildings_types/*.ini`, and TesmioLoader's `docs/06-building-mods.md` and
`docs/13-buildings.md` (MaxLegend, GPL-3.0), which have the full walk-through.

A new building needs no reverse engineering: the Workshop format describes one
completely. A loader is only needed when the building refers to something the
base game does not have, such as a new resource or deposit type.

## Cloning a donor

1. **Pick the donor by shape, not looks**: footprint, conveyor inputs and
   outputs, storages, animation. A processing plant needs a donor with conveyor
   inputs as well as outputs.
2. **Collect its files** from `media_soviet/`:

| What | Base-game path | Copy to |
|---|---|---|
| mesh | `buildings/<donor>.nmf` | `<Object>/model.nmf` |
| material | `buildings/<donor>.mtl` | `material.mtl` at the item root |
| emissive material, if it exists | `buildings/<donor>_e.mtl` | `material_e.mtl` at the item root |
| collision box | `buildings_types/<donor>.bbox` | `<Object>/building.bbox` |
| fire points | `buildings_types/<donor>.fire` | `<Object>/building.fire` |
| build-menu icon | `editor/tool_<donor>.png` | `<Object>/imagegui.png`, also `previewimage.png` |
| definition | `buildings_types/<donor>.ini` | `<Object>/building.ini`, then edit |

   Animated parts are referenced in place, not copied: `$ANIMATION_MESH` paths
   are relative to `media_soviet/`.

3. **Write `renderconfig.ini`.** Its keywords carry no `$` except the opening
   `$TYPE_WORKSHOP`, and the body is indented by one space:

```
$TYPE_WORKSHOP
 MODEL model.nmf
 MATERIAL ../material.mtl
 MATERIALEMISSIVE ../material_e.mtl
 LIFE 3000.000000
 EXPLOSION_GROUP 0
 DERBIS_FALLING_FX buildingfall1 1.000000
 DERBIS_FALLED_FX buildingfall2 1.400000
 DERBIS_FALLED_SFX collapse
 DERBIS_NUM 20
 DERBIS_FALLING_FX_MAXTIME 3.000000
 DERBIS_SCALE 1.000000
 DERBIS_MESH buildings/buildingwreck1.nmf buildings/buildingwreck.mtl
 DERBIS_MESH buildings/buildingwreck2.nmf buildings/buildingwreck.mtl
 END
```

   Declare `MATERIALEMISSIVE` whenever the donor has an `_e.mtl` (159 of the
   493 base building materials do). Without it the first frame crashes in
   `C3DDLL64.dll + 0xAC544`. Other keywords in use: `LIGHT`, `LIGHT_RGB`,
   `PLANESHADOW`, `SMOKEPOINTCHANCE`, `MODEL_LOD2`, `EXACTSPECULAR`,
   `REFLECTION`.
   A donor's file can name sibling materials and numbered variants: copy those too,
   and drop lines for files that do not exist.

4. **Edit `building.ini`: change the economy, keep the geometry.** Use
   `$NAME_STR "..."` (a literal) rather than `$NAME <id>` (a language-file id).
5. **Check the material** (below), then load the game.

## building.ini

There is no comment syntax: a `$` anywhere starts a token. `//`, indentation
and the base game's own `-------` separator lines are harmless; a `$` inside
them is not.

### Identity and type

- `$NAME_STR "Text"`, or `$NAME <language id>`.
- Exactly one `$TYPE_*`: `$TYPE_FACTORY`, `$TYPE_LIVING`, `$TYPE_STORAGE`,
  `$TYPE_CARGO_STATION`, `$TYPE_PASSANGER_STATION`, `$TYPE_SHOP`,
  `$TYPE_ENGINE` (power plants), `$TYPE_TRANSFORMATOR`, `$TYPE_MINE_*` and
  more. The engine keys on the `BUILDINGTYPE_*` number listed in
  `scripts/SOVIETInstructions.txt`; a genuinely new `$TYPE_*` needs a code
  patch.

### Labour and production

```
$WORKERS_NEEDED 80
$PROFESORS_NEEDED 4
$CITIZEN_ABLE_SERVE 9
$PRODUCTION furniture 0.012
$CONSUMPTION boards 0.020
$CONSUMPTION fabric 0.008
```

`$PRODUCTION` and `$CONSUMPTION` are per worker-day. Every resource name goes
through the engine's resource lookup, so a name the game does not know
resolves to nothing. That can crash the game while it loads, so buildings naming a
plugin's resource must ship and run with that plugin.

### Storages

```
$STORAGE RESOURCE_TRANSPORT_ELETRIC 5000
$STORAGE_IMPORT RESOURCE_TRANSPORT_COVERED 30
$STORAGE_EXPORT RESOURCE_TRANSPORT_COVERED 20
$STORAGE_IMPORT_SPECIAL RESOURCE_TRANSPORT_OPEN 40 boards
$STORAGE_FUEL RESOURCE_TRANSPORT_OIL 8
```

Transport class, capacity, and for `_SPECIAL` one resource. The capacity is in
game units: the window shows it times the resource's factor for that class, so
300 with a factor of 0.35 reads 105 t. The class must match the resource's own class, or the capacity is zero and the
line reads `0.00 of 0.00 t`. Two inputs of different classes need two
storages. A plain `$STORAGE` holds only the game's fixed list of goods for that
class, so a resource a plugin added needs `$STORAGE_SPECIAL`, one line each.

| # | `RESOURCE_TRANSPORT_` | # | `RESOURCE_TRANSPORT_` |
|---|---|---|---|
| 0 | `COVERED` | 9 | `ELETRIC` |
| 1 | `OPEN` | 10 | `VEHICLES` |
| 2 | `GRAVEL` | 11 | `GENERAL` |
| 3 | `OIL` | 12 | `NUCLEAR1` |
| 4 | `CEMENT` | 13 | `NUCLEAR2` |
| 5 | `COOLER` | 14 | `HEATING` |
| 6 | `LIVESTOCK` | 15 | `WATER` |
| 7 | `PASSANGER` | 16 | `SEWAGE` |
| 8 | `CONCRETE` | 17 | `WASTE` |

The misspellings are the game's own.

`$STORAGE_DEMAND_*` is the shops' family; which goods each category stocks is
decided in the engine's code, not in the file. A `$TYPE_SHOP` building sells
whatever sits in any of its storages to citizens who want it, including a
`$STORAGE_SPECIAL` holding a single good; the pub sells alcohol that way.

### Electricity

Power is class 9 and moves over wires, so it never belongs in a
`$CONSUMPTION` line. Either the draw scales with staffing:

```
$ELETRIC_CONSUMPTION_LIVING_WORKER_FACTOR 1.5
$ELETRIC_CONSUMPTION_LIGHTING_WORKER_FACTOR 1.25
```

or it is flat: `$CONSUMPTION_PER_SECOND eletric 0.027`. Not both.

Without power a building keeps producing at `$ELETRIC_WITHOUT_WORKING_FACTOR`
(0.4 or 0.7 on most base buildings; 0.0 stops production). A clone inherits
the donor's value, which looks like "the electricity line is broken".

### Geometry: copy verbatim

`$CONNECTION_*` (roads, pedestrians, rail, conveyors, bulk, pipes, wires),
`$CONNECTIONS_ROAD_DEAD_SQUARE`, `$VEHICLE_STATION`, `$VEHICLE_PARKING`,
`$VEHICLE_PARKING_PERSONAL`, `$TEXT_CAPTION`, `$PARTICLE`,
`$WORKER_RENDERING_AREA`: all are measured against the mesh.

`$PARTICLE factory_*` is not only smoke: the parser adds pollution per line. Read
in code on 1.1.1.9:

| Particle | big | medium | small |
|---|---|---|---|
| `factory_*_black` | not decoded | 7 | 4 |
| `factory_*_gray` | 6 | 4.2 | 2.4 |
| `factory_*_white` | not decoded | 2.1 | 1.2 |

For a clean building remove them and any `$POLLUTION_*`. That no pollution remains
on the pollution map was not yet checked in game.

### Construction

```
$COST_WORK SOVIET_CONSTRUCTION_GROUNDWORKS 0.0
$COST_WORK_BUILDING_NODE brickShape1
$COST_RESOURCE_AUTO ground_asphalt 1.0
```

`$COST_WORK` opens a phase, `$COST_RESOURCE_AUTO` says what it consumes, and
`$COST_WORK_BUILDING_NODE` names a node inside the `.nmf` shown during that
phase. Keep the donor's block.

### `$RESOURCE_VISUALIZATION`

It takes a storage index, counting `$STORAGE*` lines from zero in file order.
Reordering storages moves the piles to the wrong yard. Open-class goods are
drawn as a grid of whole units, bulk goods as a single heap.

### Odds and ends

`$STYLE_FLAG` (the era filter in the build menu), `$AMBIENT_SFX`,
`$WORKING_SFX`, `$MENU_SFX`, `$POLLUTION_SMALL` / `_MEDIUM`,
`$HEATING_DISABLE`, `$ATTRACTIVE_SCORE`, `$QUALITY_OF_LIVING`,
`$CIVIL_BUILDING`, `$MOVEABLE_DOOR`, `$UNDERGROUND_MESH`.

`$VEHICLE_LOADING_FACTOR` and `$VEHICLE_UNLOADING_FACTOR` speed up a warehouse's
stops. `$WORKING_VEHICLES_NEEDED` sets how many vehicles an office runs.
`$RESOURCE_SOURCE_*` lines give a technical office one destination row each, see
`services.md`.

Turning a car plant into a factory: change `$TYPE_PRODUCTION_LINE` to
`$TYPE_FACTORY` and remove `$SUBTYPE_*`, `$PROFESORS_NEEDED`,
`$ELETRIC_CONSUMPTION_LIVING_WORKER_FACTOR`, `$VEHICLE_PARKING`, `$PARKING_NOT_BLOCK*`
and any `$RESOURCE_VISUALIZATION` block. Written this way in WRSR Mail, not yet
seen in game.

`$ENGINE_SPEED` belongs to conveyor and cableway engine buildings
(`bulk_engine*.ini`); it is not a vehicle setting.

## Materials (.mtl)

```
$SUBMATERIAL _Material__517
$TEXTURE 0 buildings/factorycommon/3mtv5.dds
$TEXTURE 1 buildings/blankspecular.dds
$TEXTURE 2 buildings/blankbump.dds
$DIFFUSECOLOR 0.96 0.96 0.96 1.0
$SPECULARCOLOR 1.0 1.0 1.0 1.0
$AMBIENTCOLOR 0.96 0.96 0.96 1.0
$SPECULARPOWER 2.0
$END
```

- Texture slots: 0 diffuse, 1 specular, 2 bump. The base game's placeholders
  are `buildings/blankspecular.dds` and `buildings/blankbump.dds`.
- `$SUBMATERIAL` names must match the names in the `.nmf` header (0x40 bytes
  each, starting at offset 0x14). A mismatch renders untextured, with no error.
- `$TEXTURE` paths are relative to `media_soviet/`. `$TEXTURE_MTL` paths are
  relative to the `.mtl`'s own folder, so they stop resolving once the file is
  copied into an item: rewrite them as `$TEXTURE`. 130 of the 493 base
  building materials use `$TEXTURE_MTL`.
- No comment syntax here either. A `$TEXTURE` before the first `$SUBMATERIAL`
  crashes in `C3DDLL64.dll + 0x9D0AE`. Unknown tokens are ignored silently.

## When it fails

| Symptom | Cause |
|---|---|
| `ResourceGet - not found <English word>`, then a crash in the parser | a `$TOKEN` inside a comment |
| the first frame crashes in `C3DDLL64.dll + 0xAC544` | a mesh with nothing loaded: a missing `MATERIALEMISSIVE`, or an `.nmf` path that does not resolve |
| a storage shows `0.00 of 0.00 t` | its class does not match the resource's class |
| the building produces without power | the donor's `$ELETRIC_WITHOUT_WORKING_FACTOR` |
| part of the mesh is untextured | a `$SUBMATERIAL` name mismatch, or a copied `$TEXTURE_MTL` path |
| pollution around a building with no `$POLLUTION_*` | its `$PARTICLE factory_*` lines |
| a vehicle arrives and keeps its cargo | the good is not on the store's list: see Storages |

The game's own log says `Failed to open ...building.ini` when
`$OBJECT_BUILDING` names a folder that is not there.

## Generating buildings from config

TesmioLoader's `buildings` plugin writes a whole Workshop item from one
section of `plugins/buildings.ini` (donor, object name, display name, and
`line = $TOKEN ...` replacements) before the game reads its first file. See
its `docs/13-buildings.md`.
