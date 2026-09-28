# Reverse engineering

Part of wrsr-agents (CC BY 4.0). Sources: TesmioLoader (MaxLegend, GPL-3.0)
`docs/03-reverse-engineering.md`, `docs/01-architecture.md` and
`docs/07-pitfalls.md`; railspeed `railspeed.cpp` (SosseTurner, GPL-3.0);
RailPhysics `docs/04-reverse-engineering.md` (TheRealMeowMeow00, GPL-3.0).

## Know which file you are looking at

```
python -c "import struct;d=open(r'SOVIET64.exe','rb').read();print(hex(struct.unpack_from('<I',d,struct.unpack_from('<I',d,0x3c)[0]+8)[0]))"
```

`0x6A3EB6AD` is v1.1.1.9, `0x69C4098C` is v1.1.1.7. The printed version comes
from the main menu's format string `v%d.%d.%d.%d (64 bit DX11.1 - GPU: %ls)`;
the four numbers are immediates stored just before the call that uses it. A
Ghidra project or a list of addresses is a claim about one specific file:
re-derive when bytes stop matching.

## Techniques, cheapest first

1. **PE structure** (instant): imports, exports, sections, and `.pdata`, whose
   `RUNTIME_FUNCTION` records give every function's exact start and end.
   `C3DDLL64.dll`'s exports are its symbol file: demangle them with `undname`,
   or list them with `llvm-objdump -p` (llvm-mingw).
2. **String cross-references** (the workhorse): find a distinctive string in
   `.rdata`, scan `.text` for RIP-relative `lea` to it (`REX.W 8D /r` with
   mod=00, rm=101), map each hit to its function through `.pdata`. UI labels
   are language ids stored as immediates, so a label's id finds the function
   that draws it. This fails on code that touches no string.
3. **Runtime observation**: hook an import and log its arguments and results;
   hook a vtable slot for virtual calls; guard pages find who reads a buffer.
   Usually faster, and more trustworthy, than reading disassembly.
4. **Ghidra** for decisions (comparison chains, switches, field arithmetic),
   not for finding code or learning behaviour. Import once, then run scripts
   headless with `-process SOVIET64.exe -noanalysis`. The first decompile of a
   big function takes minutes, and some are too large for the decompiler and
   must be read as disassembly.
5. **Shaders**: `media_soviet/shaders_d3d11/*.inix` hold a technique name and
   its vertex and pixel DXBC blobs, each blob's length at its own +0x18.
   `D3DDisassemble` from `d3dcompiler_47.dll` (on every Windows) prints them
   with full reflection. Read a shader before assuming it needs a patch.
6. **Replaying a builder**: data written by straight-line code, such as the
   resource table, can be rebuilt exactly by replaying the stores against the
   frame pointer. Anchor the frame pointer, model the helper calls that clear
   parts of the buffer, follow every commit idiom, then rebuild each instance
   from your field list and diff every byte.

Sanity-check the decompiler on a function whose behaviour you already know.
Ghidra names registers, not values: one variable can hold two different
constants. Check floats and addresses against the disassembly, which a patch
needs anyway for instruction lengths and encodings.

## Porting to a new build

What the 1.1.1.7 to 1.1.1.9 update did at the byte level, although its notes
called it content-only:

- `.data` did not move: globals, the game object and every structure offset
  survived.
- `.rdata` float literals moved by uniform shifts within clusters (-0x18,
  -0x20): find a constant by its value, never by its address.
- `.text` shifted by different amounts at different positions (+0, -0x10,
  +0x70, +0xA0, +0xD0, +0x120, +0x1E0): no single delta works.
- A constant can change value, not only address: TesmioLoader's day-length
  constant went from 60.0 to 63.0.

The method that worked: signatures that wildcard only displacement and jump
fields, candidates filtered against `.pdata` function starts, ties broken by
what each candidate's body reads.

## Signatures

- Wildcard displacements and jump targets; keep opcodes, registers and
  compared immediates.
- Require exactly one match. Zero or several is a failure, never a guess.
- Then check meaning, not just bytes: follow the displacement and confirm the
  constant it reaches holds the expected value. railspeed does this, and its
  three signatures find the infrastructure speed-limit lookup on both 1.1.1.7
  and 1.1.1.9.

## Patching

1. Record the original bytes and compare before writing; refuse on a mismatch.
2. Code caves and replacement constants must sit within +-2 GB of the exe
   (rel32 and RIP-relative reach): walk allocation granularity outward from the
   module base with hinted `VirtualAlloc`.
3. Inline hook: `FF 25 00000000` plus an 8-byte address is 14 bytes, ending on
   an instruction boundary. Never steal a RIP-relative instruction. Prologues of
   the form `mov [rsp+8],rbx / push / sub rsp,imm32` are 13 bytes followed by
   the stack-cookie load: count the callers and patch the `call rel32`
   instead.
4. To change one site's constant, repoint that instruction's disp32 at your
   own float; never write into the shared literal pool.
5. `FlushInstructionCache` after writing code.
6. Import swaps: patch both modules' import tables. When another plugin may
   own a slot, read the slot at call time instead of caching the pointer.

## Reading a crash

- `module + rva` locates it. The earliest crash or caught fault in the log is
  the cause, even twenty seconds and a whole world load earlier.
- A near-null write address is `field + index * stride` off a null base:
  solve it and you have the statement.
- A crash inside a graphics driver with no return address into
  `SOVIET64.exe` on the stack means the damage was done elsewhere, earlier;
  for example a D3D11 `Map` from the wrong thread.

## Where current addresses are

Do not copy addresses out of prose. For v1.1.1.9:

- TesmioLoader's plugin sources (`plugins/*/*.cpp`) `#define` each RVA with the
  previous build's value in a comment. Its `docs/02-findings.md` quotes
  v1.1.1.7 addresses and records how each thing was found.
- railspeed's `railspeed.cpp`: signatures for the speed-limit lookup.
- RailPhysics's `plugins/railphysics/railphysics.cpp`: signatures and offsets
  for the rail update, verified in game on 1.1.1.9.
- RML keeps a per-exe symbol cache and serves semantic symbols to its plugins.
