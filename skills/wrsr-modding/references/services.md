# Waste, technical offices, routes, customs and vehicle classes

Part of wrsr-agents (CC BY 4.0). Everything here comes from WRSR Mail
(TheRealMeowMeow00, GPL-3.0), a mod that builds a new service on the structure
of the waste system. SOVIET64.exe 1.1.1.9. Its `docs/waste-system.md` and
`docs/vehicle-classes.md` hold the function addresses and the evidence; its
`plugins/mail/mail.cpp` holds working signatures: github.com/TheRealMeowMeow00/WRSR_Mail

Marks: **[game]** seen working in the game, **[code]** read in the disassembly
and not tried separately, **[inferred]** deduced, not proven. Keep them apart
when you repeat a fact. Offsets on `game` are from the static at `0x9D4F10`.

## Waste

- `waste_mixed` is a standalone record at `game+0xC3B8`, outside the resource
  vector. It is really a composition: record field `+0x2EC` is its share per
  fraction. The sorted list storages use is at `game+0x12078`, stride `0x10`.
  [code]
- All sources (citizen at home, customer leaving, worker leaving, production)
  end in one function taking `(game, building, list of {resource, amount},
  factor)`. Hooking it sees every waste event, and it runs only while waste is
  enabled in the game settings. [code, game]
- `$WASTE_WORKERS` keeps a resource only if it is `waste_mixed` or in the sorted
  list: an ini cannot name another resource there. Disable tokens:
  `$WASTE_DISABLE`, `$WASTE_PRODUCTION_DISABLE`, `$WASTE_WORKERS_DISABLE`,
  `$WASTE_CUSTOMERS_DISABLE`. [code]
- A type that makes waste and declares no waste storage gets a default bin at
  the end of the parser: class 17, kind 5 (small) or 6 (big). [code] A building
  copies its type's storages when it is built, so a storage added to the type
  does not reach buildings already on a map. [game]
- Landfills and transfer stations are plain `$TYPE_STORAGE` with
  `$STORAGE RESOURCE_TRANSPORT_WASTE`. Decay, pollution and overflow run only on
  class 17 storages, in a tick every 10 s per building. [code]
- Waste does not touch happiness directly: it adds to the pollution map, which is
  copied to housing, and health falls with it. [code]
- Container stands: `$TYPE_TRASH_CONTAINER`, type 105. A building's reachable
  stands are at `building+0x1140`; waste goes there first, then to the building's
  own bin. Their window expects its own kinds of storage: extra storages on a
  stand crashed the window when clicked (null read, picking a container image).
  [game; the cause is inferred]

## Technical office (`$TYPE_GARBAGE_OFFICE`, type 49)

- Its tick walks the vehicles at `office+0xC70`, collects the idle ones by class
  15, 16, 17 and snow ones by skill `0x23`, rebuilds targets at `office+0xF90`
  every 10 to 30 s, then runs one assigner each for snow, water, sewage and
  waste. [code]
- `office+0xF78` is a vector of `{building*, source id}`: the destinations the
  player sets, one row per `$RESOURCE_SOURCE_*` token. Ids: `0x14` water, `0x15`
  sewage, `0x16` gravel, `0x17` steel, `0x18` aluminium, `0x19` plastic, `0x1A`
  bio, `0x1B` toxic, `0x1C` general. The row captions are a hardcoded table.
  [code] A custom service can reuse rows: read them by position, searching by id
  did not work. [game]
- Vehicle fit: one function decides whether a vehicle type fits a building, and
  the purchase list passes through it. Hooking it widened what an office buys
  [game]; limiting it was not checked. A category exists in the purchase window when a listed vehicle has
  that tag. [code]
- Horse carts have class `GENERAL`: an office that buys only class 0 road trucks
  does not list them. [code]

## Routes

- A vehicle: `+0x3B8` / `+0x3C0` cargo vector `{resource, amount}`, `+0x4F0` the
  building it stands in, `+0x4F8` owner, `+0x508` a cooldown the game also
  resets, `+0x528` and `+0xC80` busy markers, `+0x638` / `+0x640` stops, `+0x698`
  current stop. A type: `+0x200` name, `+0x294` category, `+0x85E0` skills
  (stride `0x10`), `+0x8600` class, `+0x8604` capacity. [code]
- A stop has stride `0x198`: `+0x00` load flag, `+0x04` load share, `+0x08` load
  resources, `+0xC8` unload flag, `+0xCC` unload share, `+0xD0` unload resources.
  [code]
- A plugin can build routes with the game's clear, add-stop and start functions.
  A route of 11 stops ran in full, the cargo growing at each stop. [game]
- Route start can refuse, often for several vehicles in one tick, and trips ended
  early with the vehicle back at base and cargo aboard. An unreachable stop is the
  likely reason, not established. [game; cause inferred]
- Before calling a vehicle free, check its stops and current stop, not only the
  busy markers and `+0x508`. Clearing a live route on every tick looks like a
  storm of refused starts. [game]
- The depot button's function clears the route and returns the vehicle to its
  office, keeping the cargo for office types. It can also rescue stalled trucks: a
  watcher that calls it after 90 s without progress is how WRSR Mail handles a
  stall whose root cause it never found. [code; the stall is game, cause open]
- Reaching a building with no road bay: the road search sets a "service" flag
  when the owner's type is `0x49`, `0x4D` or `0x6A`, or `0x31` with class 1, 2 or
  15 to 17. With it, a building with fewer than two bays gets its own search mode.
  Adding class 0 to that list lets covered trucks serve pedestrian-only houses.
  [game] Water and sewage tankers have no skill for this and rely on the flag.
  [code]

## Customs and trade

- Customs houses are type 20: the road ones, plus two hidden air ones,
  `zoll_air_west` and `zoll_air_soviet`, created on map load, listed in
  `game+0x11B08` with pointers at `game+0x11AF8` and `game+0x11B00`. [code]
- Type 31 is the electricity connection (`BUILDINGTYPE_ELETRIC_EXPORT`), not
  customs: an earlier reading had them mixed up.
- A customs house takes a good only if a store lists it. Generic `$STORAGE` stores
  have fixed lists from the game (11 goods for `COVERED`), so a vehicle arrives
  with a new resource and keeps it. Appending the resource to the generic class 0
  store's two vectors (`+0x00` and `+0x18`) on each customs instance made selling
  work by plane, which had failed; road and ship sold too, but ships had not been
  tried before the change. It acts on instances, so a map made earlier should
  benefit as well. [game; old maps code]
- A resource has four prices, see `internals.md`. Selling and the cargo value in a
  vehicle's window use the sell pair; customs pays it as is: 271.89 t for 309,990
  RUB against a price of 1140.39. Dollar sales run, but their per-tonne price was
  not compared. [game] The market table reads snapshots at `game+0x114C0`
  (buy USD), `+0x114D8` (buy RUB), `+0x114F0` (sell USD), `+0x11508` (sell RUB).
  [code]
- A resource nothing produces prices at zero, and the price solver overwrites
  prices on some days. A mod that wants its own price writes it back regularly
  and updates the snapshots too.

## Vehicle classes

`$RESOURCE_TRANSPORT_TYPE` sets `type+0x8600`; storages have the same numbers at
`+0x90`. Can a mod add a class? All [code], none of it tried in game:

- **Number 19 no:** the resource record's table of class blocks runs from `0xCC` at
  `0x20` each and reaches `+0x30C`, the material family, after 18 blocks (0 to 17).
  Block 19 would even end at `0x34C`, past the record's `0x340`.
- **Number 18 no:** an unknown name parses to 18 without complaint, so a typo
  loads silently. But 18 means "no class", the type's default, 29 places compare
  with it, and it has no block of its own.
- **Reusing 9 or 14 is risky:** no vehicle uses them, but storages compare with
  them 73 and 54 times as electricity and heating. Other carriers would stop
  carrying the resource unless handled.
- **A virtual class is the practical answer:** vehicles stay in a real class, a
  list in the plugin's ini says which ones count, and the vehicle-fit hook applies
  it. They then load and travel everywhere as usual: planes, ships, wagons and
  customs carried class 0 mail with no further work. [game for the carrying]

Skills, by parser order: `0x13` groundworks, `0x15` crane, `0x1E` ambulance, `0x1F`
fire truck, `0x20` fire ladder, `0x21` personal, `0x22` forklift, `0x23` snow plow,
`0x24` police car, `0x25` police bus, `0x26` and `0x27` waste containers. [code]

## Not read

Runtime of `$WASTE_EXTRACTION` and `$WASTE_CONSUMPTION`, production waste in full,
the save format of storages, how a building joins `building+0x1140`, the caption of
a building category, the code that pays at customs.
