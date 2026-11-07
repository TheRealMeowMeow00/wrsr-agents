# Pitfalls

Part of wrsr-agents (CC BY 4.0). Every entry cost somebody at least one
debugging session. Most come from TesmioLoader's `docs/07-pitfalls.md`
(MaxLegend, GPL-3.0), which tells each story in full; the rest from
RailPhysics, railspeed and WRSR Mail. The rules in `SKILL.md` are not repeated
here.

## Files and text

- A setting that seems ignored: check the file's first bytes for a BOM
  (`EF BB BF`).
- Loader settings go through the ANSI profile API: keep them ASCII.
- Private language ids must sit above the game's highest id (580231). A hook
  that claims a range of ids it does not fully own replaces real labels
  several menus away, silently.
- An empty string is a silent failure: a name that resolves to an id that
  resolves to nothing shows as blank, with no error anywhere.

## Assets

- `$TEXTURE_MTL` copied into an item no longer resolves; `$SUBMATERIAL` names
  that do not match the mesh render untextured. Neither logs anything.
- `$RESOURCE_VISUALIZATION` counts storages: reordering them moves the piles.
- A clone keeps the donor's `$ELETRIC_WITHOUT_WORKING_FACTOR` and runs without
  power.
- `$ENGINE_SPEED` is a conveyor/cableway engine setting, not a vehicle's top
  speed (`$MOVEMENT_SPEED`).
- Long rail cars without `$TRAIN_*_AXIS_DISTANCE` pivot 3 m from their ends and
  leave the rails in tight curves.
- A resource whose cargo meshes are missing crashes later, on a worker thread,
  in `C3DDLL64.dll + 0xAC544`: check the files exist before asking the engine
  to load them.

## Native code

- **A new resource is missing from every fixed list.** Customs and generic stores
  list their goods in the game's own code: a vehicle arrives and keeps the cargo,
  with nothing in the log. Shops restock it through generic code from any store
  nearby, so it can appear where nobody sent it.
- **Check the build stamp before trusting a single address.** An update in the
  middle of a session silently invalidates everything derived before it.
- **Hook both import tables.** `C3DDLL64.dll` opens files and resolves strings
  through its own imports. The list of file openers (`fopen`, `fopen_s`,
  `_wfopen`, `_wfopen_s`, `fread`, `CreateFileA`, `CreateFileW`,
  `CreateFile2`) has grown one entry at a time; assume it is still incomplete.
- **Virtual calls are invisible to import hooks.**
- **A cached import is not the live import.** If another plugin may swap a
  slot, read the slot when you call through it.
- **One `VirtualQuery` is not a range check.** `.data` regions split as pages
  are first written, so a check over a large struct that passed at startup
  fails later with every byte still readable. Walk the regions, and only ask
  about what you actually dereference.
- **A heuristic find needs an invariant the lookalike cannot fake**: "do the
  pointers point where real ones point", not "do the fields look plausible".
- **An `__except` that logs "disabled" must disable.** Otherwise the same fault
  fires on every call, forever.
- **Catching a fault inside the engine is not surviving it.** A half-built
  engine object crashes later, somewhere else, on another thread.
- **A failed lookup once a frame is a frame-rate leak**: the resolver logs every
  miss. Cache the answer; latch the miss after one warning.
- **State that outlives the world**: a map load, a return to the menu or the
  terrain editor rebuild what the game owns. Re-verify cached pointers against
  the game's structure, not against the event that last changed them.
- **A new resource wakes dead code.** Branches the base game's fixed set of
  goods never reached, such as a fifth bucket after four cached records, have
  never run and may hold never-constructed vectors.
- **A name table sized for today reads out of bounds tomorrow** once another
  config adds values.
- **`TextureAccessOpen` is a GPU `Map`**, not a lock: render thread only.
- **A dangling pointer that returns the right answer** (after
  `TextureAccessClose`) turns a missing bracket into a bug that only happens
  "before the player painted".
- **Crash handlers**: bound the stack walk with `VirtualQuery` on `RSP`, guard
  against re-entry. A fault inside your own DLL may not be logged at all; its
  absence from the log is itself evidence.

## Simulation semantics

- **Two clocks.** Per-frame systems integrate from the simulation timer;
  others (pollution decay, loans, events, price recomputes) run once per
  calendar day. Scaling one desynchronises the other; scaling the shared timer
  slows movement too.
- **Second copies are normal here.** The `building.ini` parser has its own
  storage logic; the walking limit is written out in several functions; UI
  overlays recompute values instead of reading the simulation's.
- **A displayed value can have its own code path.** When the simulation obeys a
  patch and the window does not, find the window's copy.
- **A resource has separate buy and sell prices.** Writing only the buy pair leaves
  selling and the cargo value on the game's own calculation, and the market table
  reads snapshots of its own.
- **A window can count on storages being the kinds it knows.** Extra storages on a
  container stand crashed its window; a layout value set inside skipped code breaks
  every row after it.
- **Buildings on an old map do not get a type's new storages.** They copied the list
  when they were built. Test on a new map, or add to the instances.
- **A need has four faces**: the citizen's demand, a per-inhabitant rate, a
  counter the building window prints, and a real mechanic. Suppressing one
  leaves the others.
- **When you fake a value, enumerate every consumer of it**, not just the one
  you are aiming at.
- **A constant that looks like the limit can be a search radius.** Follow the
  write into the structure being filled before patching.
- **Search for a value in every form**: `.rdata` literal, RIP-relative read,
  and a bare `imm32` in an instruction.

## Process

- Write down what was observed and what was inferred differently. A guess
  written in the voice of a finding once cost a whole feature for months.
- Two pointers with the same value are not two references to the same object.
- After two clean logs and no effect, stop reasoning and hook the function to
  log its arguments and results.
- The earliest crash line is the cause, even if the process died twenty
  seconds later.
- The game holds DLLs and logs open: close it before a rebuild, read logs with
  `FileShare.ReadWrite`.
- If a DLL stays locked after closing, look for a windowless `SOVIET64.exe` left
  behind, for example by closing before the main menu. It holds the files, and it
  burns CPU and GPU, which also looks like a frame-rate problem in a second instance.
- Adding resources breaks save compatibility both ways; test on copies.
