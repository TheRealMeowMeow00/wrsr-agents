# Vehicles

Part of wrsr-agents (CC BY 4.0). Measured on the base game's 594 vehicle
`script.ini` files and about 1 400 from subscribed items, September 2026.
Meanings marked *inferred* come from base-game values, not from code.

## Where they are

`media_soviet/vehicles/` (road, 408), `trains/` (136), `airplanes/` (21),
`ships/` (21), `helicopters/` (8): one folder per vehicle, each with
`script.ini`, `main.nmf`, a material and preview images. The Workshop layout
is in `workshop.md`.

## script.ini

`$TYPE` is one of `VEHICLETYPE_ROAD`, `VEHICLETYPE_RAIL_LOCOMOTIVE`,
`VEHICLETYPE_RAIL_VAGON`, `VEHICLETYPE_SHIP`, `VEHICLETYPE_AIRPLANE`,
`VEHICLETYPE_HELICOPTER`.

The most used tokens, with counts over about 2 000 files:

| Token | Uses | Meaning |
|---|---|---|
| `$NAME_STR "..."` / `$NAME <id>` | 1426 / 541 | display name, literal or language id |
| `$DESCRIPTION` / `$DESCRIPTION_STR` | 1041 / 41 | |
| `$COUNTRY <id>` / `$COUNTRY_STR` | 1864 / 99 | country of origin (*inferred*: the ids are language ids) |
| `$AVAILABLE <from> <to>` | 1945 | the years it can be bought |
| `$COST_RUB` / `$COST_USD` | 1496 / 462 | price |
| `$MOVEMENT_SPEED` | 1900 | top speed, km/h |
| `$MOVEMENT_POWER_KW` | 1788 | power in kW; 0 on unpowered cars |
| `$MOVEMENT_EMPTY_WEIGHT` | 1958 | empty mass, tonnes (*inferred*) |
| `$MOVEMENT_CONSPUMPTION` | 665 | fuel consumption; the spelling is the game's |
| `$MOVEMENT_HILL_SPEED`, `$MOVEMENT_HILL_ACCELERATION` | 130 / 112 | behaviour on grades |
| `$RESOURCE_TRANSPORT_TYPE` | 1619 | a `RESOURCE_TRANSPORT_*` class (table in `buildings.md`) |
| `$RESOURCE_CAPACITY` | 1663 | capacity in that class's units: passengers, tonnes |
| `$RESOURCE_ALLOW_ONLY` | 51 | restricts the cargo to one resource |
| `$RESOURCE_VISUALIZATION`, `_JOINT` | 801 / 256 | cargo drawn on the vehicle |
| `$LOD1`, `$LODDISTANCE1`, `$LOD2`, `$LODDISTANCE2` | about 550 / 350 | levels of detail |
| `$LIGHT_FORWARD`, `$LIGHT_BACKWARD`, `$LIGHT_BLINKER` | 3722 / 3287 / 469 | light points |
| `$SOUND_PARAMS` | 3875 | sounds |
| `$PARTICLE_MOVEMENT` | 1424 | exhaust and similar |
| `$FAMILY` | 637 | grouping in the purchase list (*inferred*) |
| `$JOINT` | 305 | articulated parts |
| `$SKILL_*` | | service roles: `_AMBULANCE`, `_FIRETRUCK`, `_POLICECAR`, `_SNOWPLOW`, `_CONSTRUCTION_CRANE` and others |
| `$PURCHASE_EXCLUDE` | 83 | kept out of the purchase list (*inferred*) |

Rail only:

| Token | Meaning |
|---|---|
| `$TRAINGROUP_LOCOMOTIVE`, `_VAGON`, `_TRAINSET`, `_TRAM`, `_METRO` | the group a unit belongs to |
| `$TRAINSET`, `$TRAINSET_REVERSE` | a fixed multiple-unit composition |
| `$TRAIN_FORWARD_AXIS_DISTANCE`, `$TRAIN_BACKWARD_AXIS_DISTANCE` | where the car pivots on the track, in metres from each end. When absent both default to 3 m, which is wrong for long cars: their bogies swing off the rails in tight curves |
| `$TRAIN_BBOX_FORWARD_CORRECTION`, `$TRAIN_BBOX_BACKWARD_CORRECTION` | corrections to the car's extent (*inferred*) |

Base-game files sometimes write a decimal comma (`32,5`); how the parser reads
it has not been verified here.

## The speed a train actually reaches

The lower of the slowest locomotive's `$MOVEMENT_SPEED` in the consist and the
track's limit; wagons do not appear to limit it. Track limits are hardcoded in
the exe: concrete 150 km/h, wood 70, bridge and tunnel tiers 120 to 140. No
ini raises them; the railspeed plugin (SosseTurner, GPL-3.0) does.

The base game's rail physics is simplified: constant acceleration, braking at
a third of it, gravity only uphill. RailPhysics (GPL-3.0) replaces it. Engine
details: `internals.md`, section Rail.
