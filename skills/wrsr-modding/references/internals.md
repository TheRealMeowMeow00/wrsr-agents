# Engine internals

Part of wrsr-agents (CC BY 4.0). Most of this is summarised from TesmioLoader's
`docs/02-findings.md` and the per-plugin docs (MaxLegend, GPL-3.0), which
carry the evidence behind every line; the Rail section is from RailPhysics
(TheRealMeowMeow00, GPL-3.0), verified in game on 1.1.1.9. Prices and the
storage kinds were corrected from WRSR Mail (TheRealMeowMeow00, GPL-3.0). Waste,
offices, routes and customs are in `services.md`.

Everything here is **data**: structure offsets, globals in `.data`, values of
constants. None of it moved between 1.1.1.7 and 1.1.1.9. Code addresses are
deliberately left out because they move with every build; see
`reverse-engineering.md`, "Where current addresses are". RVAs are relative to
`SOVIET64.exe`'s image base.

## Globals

| RVA | What |
|---|---|
| `0x9D4F10` | the **game object**, a static in `.data` (not a pointer to one). Most offsets below are relative to it |
| `0x9941F0` | a qword **pointer** to the world object: `*(BYTE**)(base + 0x9941F0)`, then `+0xED8` the `C3D_TERRAIN`, `+0xF00` / `+0xF08` the two deposit-map textures, `+0xF10` farmap, `+0xF18` emissive map |
| `0x9D4EE0` | the `C3D_TIMER` the whole simulation steps on |
| `0x9E75B8` to `0x9E75C0` | `vector<Person*>`, every live citizen |
| `0x9E9C3C` | save format version of the loaded world (124 for saves this build writes) |
| `0x9D5F10` | the terrain editor object |

TesmioLoader's findings call both of the first two "the game object". They are
different: resources, buildings and the calendar hang off the static at
`0x9D4F10`; the terrain and the deposit maps hang off the object that
`0x9941F0` points to.

## The game object (`0x9D4F10`)

| Offset | What |
|---|---|
| `+0x580`, `+0x588` | balances, roubles and dollars |
| `+0x590` / `+0x594` | day of the year (0 to 364) / the year |
| `+0x59C` | a float timer that accumulates seconds and advances the date when it reaches the day length. Not a date: treating it as one ran a daily job every 2 s. Use `+0x590` for "once a day" |
| `+0x5BC` | "Education simulation", > 0 means Complex |
| `+0x5C8` | "Unsatisfied citizens reaction": 0 small, 1 reasonable, 2 great |
| `+0x5D4` | pollution enabled |
| `+0xC2B0` | the resource vector `{begin, end, capacity}` |
| `+0xC2C8` to `+0xC488` | 57 cached `Resource*`; `+0xC300`, `+0xC310`, `+0xC318`, `+0xC320` are food, meat, clothes, electronics |
| `+0x10330` | `vector<wstring*>`, unused city names |
| `+0x10348` | `vector<NamePoint*>`, every city: a name, a position, its buildings |
| `+0x10BD0` | loans, `0x28`-byte records |
| `+0x114C0`, `+0x114D8`, `+0x114F0`, `+0x11508` | snapshots the market table reads: buy USD, buy RUB, sell USD, sell RUB |
| `+0x11AF8`, `+0x11B00` | the hidden air customs houses, `zoll_air_west` and `zoll_air_soviet` |
| `+0x11B08` to `+0x11B10` | `vector<Building*>`, every building |
| `+0x11B20` | building types, stride `0xBE8` |
| `+0x11F88`, `+0x11FA0` | queues of buildings whose walking / parking connections need rebuilding |
| `+0x120A8` | pollution grid, 12-byte cells (count `+0x120C0`, width and height `+0x120B8` / `+0x120BC`) |
| `+0x127E8` | vehicle tag table, stride `0x44`: name, caption id at `+0x40` |
| `+0x12840` | vehicle types, stride `0x9F08` |

## Resources

- A `std::vector` of 832-byte (`0x340`) records at `game+0xC2B0`: 57 base
  records, capacity 63. A map load clears it (`end = begin`) rather than
  freeing it, so code that caches record pointers must re-verify them after
  every load.
- Index order is the field order of the `Resources` struct in
  `SOVIETInstructions.txt`: `workers eletric vehicles trains heat gravel
  rawgravel plants steel aluminium prefabpanels bricks wood oil chemicals coal
  rawcoal iron rawiron bauxite rawbauxite bitumen boards uranium yellowcake uf6
  nuclearfuel nuclearfuelburned fuel fabric alcohol cement alumina food clothes
  meat livestock asphalt concrete ecomponents mcomponents plastics eletronics
  explosives water usagewater fertiliser_liquid waste_gravel waste_steel
  waste_aluminium waste_plastic waste_bio fertiliser waste_burnable waste_toxic
  waste_other waste_ash`. `waste_mixed` and `service_material` appear in that
  struct but are standalone objects, not in the vector.
- The resolver is a linear `strcmp` scan (four standalone records, then the
  vector). It is called speculatively on every token the ini parsers meet, so
  misses are routine, and every miss logs `ResourceGet - not found <name>`.
  Never call it per frame for a name that may not exist.

| Record offset | What |
|---|---|
| `+0x00` | name, inline, NUL-terminated (bytes after the NUL are garbage) |
| `+0x40` | language id of the caption |
| `+0x44` | price kind: -1 workers, -2 electricity, -3 vehicles, -4 trains, -5 heat, 0 raw, 1 manufactured, 2 to 4 consumer goods |
| `+0x48` | icon texture, loaded from `resources/<name>.png` (48x48 RGBA), the only asset found by name |
| `+0x58` / `+0x5C` | buy price, USD / RUB |
| `+0x60` / `+0x64` | sell price, USD / RUB; customs pays this as is, and a vehicle window's cargo value reads it |
| `+0x78` / `+0x7C` | base price (`$Economy_BaseRUB` / `_BaseUSD`) |
| `+0x88` / `+0x8C` | sell / buy multipliers (0.95 / 1.05) |
| `+0xCC` | 18 transport-class blocks (0 to 17), `0x20` each, ending where `+0x30C` begins; `+0x00` of a block is the capacity factor that makes a storage of that class hold this resource. On screen a storage shows its ini capacity times this factor, and a vehicle's tonnage varies by resource the same way |
| `+0x30C` | material family (which sorted waste it becomes); -1 for immaterial ones |
| `+0x318` | the vehicle-load mesh (the only mesh an open-class resource has) |
| `+0x320` to `+0x338` | the four pile-stage meshes |

- Meshes come from literal paths in the engine's resource table, so a record
  cloned from a template draws the template's meshes whatever the files are
  called. `eletronics` is a safe template for a covered good: it has no cargo
  mesh.
- Prices are recomputed from the production chains of the building types; a
  resource no building produces prices at zero regardless of its base price.
  Base prices also drift by a random walk.
- A save stores the economy per resource name, but the resource count is part
  of the save format.

## Buildings

| Building offset | What |
|---|---|
| `+0x318` | the type descriptor; `+0x360` in it is the `BUILDINGTYPE_*` number (2 living, 3 shop, 6 factory, 7 mine, 8 field, 17 power plant, 18 substation, 19 transformer, 20 customs, 30 unknown, 31 electricity connection, 49 technical office, 105 container stand). The build menu's categories are this number; `+0x364` is the subtype (`0x1F` on air customs) |
| `+0x368` | deposit (mine) type |
| `+0x604` | >= 1.0 when the building is finished |
| `+0x970` to `+0x978` | storages, stride `0xE0` |
| `+0xA10` to `+0xA18` | wire connections, stride `0x60`: `+0x00` type (7 high voltage, 8 low), `+0x28` the line |
| `+0xBC0` to `+0xBC8` | the building's citizens |
| `+0xBD8` to `+0xBE0` | a shop's customers |
| `+0xCA8` to `+0xCB0` | walking connections, stride `0xF0`: `+0x08` the building at the other end, `+0x98` path length |
| `+0xCC0` | the same for personal cars |
| `+0xDF8` | quality of source (mines): written once at construction, which is why deposits never run out |
| `+0xE90` | a mine's deposit sample points, 28 bytes each |
| `+0xEA8` | non-zero when the building is going away |
| `+0x10E0`, `+0x10E8`, `+0x10F8` | pointers to wattage, voltage, breaker capacity |
| `+0x1130` | set when the grid solver did not reach this node |

A storage: `+0x00` / `+0x08` its slot vector (16-byte slots: `+0x00` the
`Resource*`, `+0x08` content, `+0x0C` quality), `+0x18` a parallel settings
vector, `+0x84` / `+0x85` import / export flags, `+0x8C` capacity, `+0x90`
transport class, `+0x88` which `$STORAGE*` token made it (-1 plain, 0 basic,
1 medium, 2 advanced and mediumadvanced, 3 hotel, 5 small bin, 6 big bin, 8 and 9
special, 10 carplant, 50 fuel).

Behaviour worth knowing:

- Electricity (class 9) and heat (class 14) are ordinary resources in ordinary
  storages. The transfer code checks building type in one place only, to keep
  power plants and importers from being receivers.
- For electricity a slot's quality *is* the node's voltage. Quality rises only
  inside a transfer, and a transfer only happens into a shortfall, so a full
  storage with quality 0 stays dark.
- A shop's sale loop is generic: any storage slot whose resource matches a
  customer's demand is sold from, whatever `$STORAGE*` token made it.
- The `building.ini` parser has its own copy of the storage logic; the
  storage builder it resembles is not what runs for `building.ini`.
- A plain `$STORAGE` of a class has a fixed list of goods from the game (11 for
  `COVERED`). Unloading finds a place through that list, so a vehicle with
  anything else keeps its cargo. `$STORAGE_SPECIAL` holds the one good it names.
- Shops and three similar building kinds restock on their own from any store
  nearby, through every storage of the building: a storage for a new resource gets
  filled that way unless it is hidden from those ticks.
- Pollution comes from the `$POLLUTION_*` tokens and from `$PARTICLE factory_*`
  lines, which the parser adds up.

## Citizens

A person is a `0x750`-byte object.

| Offset | What |
|---|---|
| `+0x20` | the building the person is in |
| `+0xA4` | personal speed multiplier, 0.9 to 1.1, fixed at birth |
| `+0xC8` | non-zero suppresses service demands (foreign workers, tourists) |
| `+0xD8` | eleven status floats: happiness, food, health, soviet, alcohol, culture, sport, religion, clothing, electronic, crime |
| `+0x110` | demand count |
| `+0x118` | demands, stride `0x80`, capacity 7: `+0x00` still wanted, `+0x04` total, `+0x08` kind (1 and 2 are the kinds shops serve), `+0x10` the `Resource*` |
| `+0x4F0` | unsatisfied-demand count |
| `+0x71C` | 0 citizen, 1 soviet tourist, 2 western tourist |

- The four shop goods are literals in the daily planner, not a table.
- The age has not been located: `+0x70` is a state timer and `+0x65C` the walk
  animation phase.
- The statistics window's happiness (23 reasons) and health (12) ledgers
  enumerate everything needs can affect. A need also has other faces: per
  inhabitant rates, counters the building window prints, and real mechanics.

## Walking

- The connection lists are the whole truth: job, shop and service code only
  asks whether a pair is listed; nothing re-checks the distance.
- Base limits: 480 m on foot, 2500 m by personal car. The walking limit is
  written out separately in several functions (a batch builder, a queued
  builder, and the building window's overlay, which recomputes paths on its own);
  the rebuild radii after a road change (530 and 2600) are a different thing.
- Connections are stored in the save; the loader regenerates them only for
  old saves.

## Clocks

- Everything per frame integrates from the `C3D_TIMER` at `0x9D4EE0` through
  the engine's `C3D_TIMER::Power`, `PowerTime` and `PowerKmh` (vehicle speeds).
  The game's speed buttons multiply its frame divisor (0.35 normal, 0.05 fast,
  0.01 fastest), so scaling that timer slows everything, movement included.
- The calendar is `game+0x590` / `+0x594`, advanced by a day tick when the timer
  at `+0x59C` passes 60.0 on 1.1.1.9 (read in the exe; TesmioLoader records a
  day-length constant of 63, which may be another one). For a daily job, compare
  the date: a clock derived from the engine timer ran 1.7 to 2.6 times fast.
  Many systems run once per calendar day and from nowhere else: pollution
  decay, loans (terms counted in days), fires, random events, expiring
  notifications, price recomputes (days 5, 10, 15, 20, 25 and each month
  change). Slowing only the calendar or only the timer desynchronises the two
  halves.
- A technical office keeps ticking on a paused game, so a wall-clock timeout in a
  building tick counts the pause. Seen on 1.1.1.9; other building kinds not checked.
- The base game's day/night cycle spans thirteen calendar days.
- Snow season depends on the day of the year and the map's climate (0 to 3).

## Deposits

- Two textures per map, `resourcemap.dds` and `resourcemap2.dds`, 1024x1024
  BGRA, a 128-byte DDS header, no mipmaps. Each colour component is one
  deposit type; the byte is the richness.
- `resourcemap`: component 0 oil, 1 iron, 2 coal, 3 unused (all 0 or all 255
  depending on the map). `resourcemap2`: 0 uranium, 1 bauxite, 2 data on some
  maps with no minimap button, 3 free.
- Gravel lives in component 2 of the terrain's own material mask
  (`terrain+0x158`), which the editor's Rocks tab paints; component 3 of that
  mask is the oasis brush.
- Deposit type numbers: 1 iron, 2 coal, 3 gravel, 4 wood, 6 uranium,
  7 bauxite, 8 water, 9 surface water; oil is probably 0. Building type 7 is a
  mine, 92 a water well.
- Search radii: iron, coal, uranium 210; bauxite 50; gravel 30; oil 130;
  wood 250; water 170; surface water 25. An unknown type gets radius 0 and the
  window prints -2147483648 %.
- Texture access (`TextureAccessOpen` / `Close`) is a D3D11 `Map` / `Unmap`
  on the immediate context that copies the whole map both ways: render thread
  only, as rarely as possible. `Close` leaves a dangling pointer at
  `tex+0x158`.

## Rail

From RailPhysics, verified in game on 1.1.1.9.

| Vehicle offset | What |
|---|---|
| `+0x398` | on a wagon: the vehicle pulling it; 0 on the lead unit |
| `+0x3A0` | the wagon vector (begin, end at +8) |
| `+0x4C4` | byte: car coupled back to front |
| `+0x5DF` | emergency flag (forces the limit to 0) |
| `+0x5FC` | fuel empty |
| `+0x680` | route objects |
| `+0x6B8` | route segments, in travel order |
| `+0x700` | current route index |
| `+0x798` | current track segment |
| `+0x7A0` | byte: 0 travelling node0 to node1, 1 the other way |
| `+0x7A4` | float: metres into the current segment, from its entry end in either direction |
| `+0xD24` | speed, km/h |
| `+0xD28` | effective target speed, km/h, recomputed every frame |
| `+0xD30` | slope: pitch / 0.12, clamped to +-1 |
| `+0xD39` / `+0xD3C` | will stop at the upcoming station / another stop follows |
| `+0xD60` | the `C3D_NODE` the car is drawn with |
| `+0x1708` | the vehicle type |
| `+0x1794` | non-zero: wagon excluded |

| Type offset | What |
|---|---|
| `+0x294` | category: 3 wagon, 4 locomotive, 5 service |
| `+0x7A78` | length |
| `+0x7A7C` / `+0x7A80` | front / back extent |
| `+0x7AF8` / `+0x7AFC` | pivot distances (`$TRAIN_FORWARD/BACKWARD_AXIS_DISTANCE`) |
| `+0x8600` | transport class (7 passengers) |
| `+0x8624` | top speed |
| `+0x8678` | power, kW |
| `+0x867C` | weight, t |

A track segment: `+0x08` its polyline (stride `0x18`), `+0x11C` its length in
metres, `+0x120` / `+0x124` / `+0x12C` the class ids handed to the
infrastructure speed-limit lookup.

- Vanilla physics runs on one scalar, the "divisor": acceleration is
  100 / divisor km/h per second (in effect `3.61 * sqrt(P / (5 m))`,
  constant at any speed), braking a third of that, and grade drag only uphill.
- The limit a train obeys is the minimum of the consist's top speed and the
  infrastructure limit; the emergency flag and one more check force it to 0.
- Every rail vehicle is placed through
  `C3D_NODE::CreateFromPositionRotationScale` with rotation (pitch, yaw, 0):
  the roll component is unused by the game.

## Language files

`media_soviet/soviet<Language>.btf`, twenty-one plus `sovietComments.btf`.
Big-endian throughout: `u32 count`, `u32 file size`, `u32 payload length` in
UTF-16 code units, then `count` entries of `{u32 id, u32 offset, u16 length}`
(offset and length in code units), then the UTF-16BE payload, each string
followed by a NUL.

- Ids run from 300 to 580231. A mod's private ids must sit above the game's
  highest (TesmioLoader uses 1 000 000 and up).
- The lookup is a linear scan and the first match wins. An empty string counts
  as missing and falls back to `sovietEnglish.btf`. Negative ids are runtime
  custom strings.

## Textures, meshes, UI

- `C3DAPI_D3D11_TEXTURE` vtable slots: 2 `Load2DFromFile`, 16
  `TextureAccessOpen`, 18 `TextureAccessClose`, 19
  `TextureAccessInitTempResource`, 20 `TextureAccesGetTexel`, 23
  `TextureAccesSetTexel`, 36 `SaveToDDS`.
- `.nmf` header: submaterial names, `0x40` bytes each, from offset `0x14`.
- Building info panels are built by per-panel functions that track a running
  Y; the window's `+0x250` holds the panel's bottom edge, so a post-hook can
  draw one more row there and write the new bottom back.
- `C3D_FONTMANAGER::PrintLeftUnicode` is variadic: a hook cannot forward its
  `va_list`, so prefer patching the call site that feeds it.
- Terrain editor tools are identified by name strings, their icons found as
  `editor/tool_<name>.png`, and the active tool pointer is the tool's
  descriptor.

## Script VM

Pyrois, a bytecode VM (Michal Kucharik, 2017). `SOVIETInstructions.txt` lists
261 opcodes, 20 structs and 69 struct methods. Its `Resources` struct is in
effect a header for the engine's resource order, and its `Building` struct
names 72 fields. Script state is saved (`runningScripts.bin`).
