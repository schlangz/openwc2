# Overlay format: Borland VROOMM / FBOV

`WC2.EXE` is a Borland C++-compiled DOS executable (confirmed via
runtime helper names present in the binary: `SCOPY@`, `LXLSH@`,
`_harderr`, `_setcbrk`, `_ctrlbrk`) using Borland's VROOMM overlay
manager. Its overlay table format is **byte-for-byte identical** to
another, unrelated Origin/EA-era DOS title's own VROOMM usage --
confirmed directly by running that project's existing `fbov_extract`
tool against `WC2.EXE` with zero code changes and getting a clean
extraction (240 total segment-table entries, 124 real `CODE|OVR`
overlay modules, matching structure throughout).

This means VROOMM/FBOV is Borland's own general-purpose runtime
technology, not something specific to any one game -- any DOS
executable showing the same Borland runtime fingerprints is a
reasonable candidate for the same extraction approach.

## File layout

Standard MZ header, followed by the resident load module, followed by
an `FBOV` header (magic `FB`/`OV`, i.e. bytes `46 42` `4F 56`) directly
after the load module (paragraph-aligned):

```
mz_hdr_t   { magic, cblp, cp, crlc, cparhdr, minalloc, maxalloc,
             ss, sp, csum, ip, cs, lfarlc, ovno }   (28 bytes)
...
[load module: resident code/data]
...
[FBOV header, at (loaded_size + 15) & ~15]
  bofh_t   { id[2]="FB","OV", size:u32, stofs:u32, nsegs:i32 }
[segment table, at file offset `stofs` -- absolute from file start,
 NOT relative to the FBOV header]
  boseg_t[nsegs] { seg:u16, maxoff:u16, flags:u16, minoff:u16 }
    flags: bit0=CODE(1), bit1=OVR(2), bit2=DATA(4)
[per-module stub headers, at start_ofs + seg*16, one per OVR entry]
  bosh_t   { code[2]=CD,3F (INT 3Fh placeholder), saveret:u16,
             fileofs:i32, codesz:u16, fixupsz:u16, jmpcnt:u16,
             link:u16, bufseg:u16, retrycnt:u16, next:u16,
             ems_page:u16, ems_ofs:u16, user[6] }   (32 bytes)
[actual module code+fixup bytes, at fbovofs + 16 + bosh_t.fileofs]
```

`start_ofs = mz_hdr.cparhdr * 16` (start of the resident load module).

## WC2.EXE's own header values

```
start_ofs  = 0x1A00
loaded_sz  = 0x291E0 (168416 bytes)
fbovofs    = 0x291E0  (FBOV header immediately follows the load module)
FBOV size  = 195264
FBOV stofs = 0x18640  (absolute file offset, not fbovofs-relative)
FBOV nsegs = 240
```

Segment table flag breakdown (240 entries): 95 resident `CODE`-only
(flags=`0x1`), **124 `CODE|OVR`** (flags=`0x3`, the real overlay
modules), 14 tiny `DATA`-only (flags=`0x4`, all a handful of bytes --
not a hidden string table, checked directly), 7 with flags=`0`
(padding/unused).

## Overlay call gates

Each `FBOV_OVR` segment-table entry has a fixed, resident 32-byte
`bosh_t` stub at `start_ofs + seg*16`. In its *unresolved* state (the
target overlay not currently swapped in) the stub's first 2 bytes are
`CD 3F` (`INT 3Fh`) -- a soft interrupt VROOMM's overlay manager
handles by inspecting the bytes immediately following the `INT 3F`
opcode (the rest of the `bosh_t` struct) to know which module to load
and where to patch the resolved jump target. Once resolved, those same
2 bytes get overwritten with `EA <off:u16> <seg:u16>` (`JMP FAR`) --
confirmed live (see `docs/methodology.md`).

`bosh_t.fileofs` gives a direct, seg-table-independent way to locate
any overlay module's real bytes in the file:
`code_ofs = fbovofs + 16 + bosh_t.fileofs`, then read `codesz +
fixupsz` bytes from there.

## Cross-module fixups

Each module's own fixup table (`fixupsz` bytes immediately following
its `codesz` code bytes) is a flat array of `u16` in-module offsets.
At each such offset, the 2 bytes there encode `(seg_table_index << 3)
| flags` (bit0 = `FIXUP_FUNREF`) -- i.e. "patch this location to point
at segment-table entry N's real (or synthetic) segment value."
Confirmed **zero `FUNREF`-flagged fixups exist** across the real
`9295` plain-segment fixups checked when flattening (1 stray hit noted
separately, negligible) -- every cross-module call target resolves
cleanly this way, no unresolved-offset gap to work around.

## Flattening for static analysis

Because overlay modules physically share/reuse memory slots at
runtime (their real `seg` values overlap by design), loading them at
their file-declared `seg` values into a disassembler would corrupt
data through overlap. The existing `flatten_all` tool solves this
generically:

1. Find the highest real `seg` value used anywhere in the table.
2. Assign every `FBOV_OVR` entry a synthetic, non-overlapping base
   above that, sequential by segment-table index, sized by each
   module's own `codesz` (paragraph-rounded). Non-`OVR` entries keep
   their real `seg` unchanged (resident, already valid,
   non-overlapping).
3. For each `OVR` module, re-derive its fixups from the *original* raw
   bytes (not a previously-patched copy) and rewrite every resolved
   target to point at the synthetic address instead of the real one.
4. Emit one flat, non-overlapping binary blob plus a manifest CSV
   (`seg_index, real_seg, synthetic_seg, codesz, byte_offset,
   funref_count`) mapping each module.

Running this against `WC2.EXE` (again, zero code changes needed):

```
max_real_seg    = 0x2776
synthetic_start = 0x3776
synthetic_end   = 0x64C6
total           = 185600 bytes, 124 OVR modules
```

## Loading into Ghidra

A small Ghidra script (`GhidraScript` subclass, run via Script
Manager) reads the flattened `.bin` + manifest directly off disk
(round-tripping ~185KB of module bytes through MCP tool-call
parameters was tried once and is far too context-expensive to be
practical) and, per module: scans for a free synthetic-segment range
in *this* Ghidra project (avoiding collision with anything already
loaded), creates an initialized memory block there, copies the
module's bytes in, disassembles from the block's start, and creates a
named function (`OVR_<index>_seg<realseg>`) at that address.

For `WC2.EXE` this loaded all 124 modules cleanly; the script picked
base segment `0x4776` (delta `0x1000` from `flatten_all`'s own
`0x3776`, to avoid colliding with the resident image already loaded at
lower segments).

**Caveat, confirmed directly**: the script's own per-module
`disassemble()` call only walks code reachable in a straight line from
each module's *own* entry point. Functions reached only via indirect
jump/call tables inside a module (e.g. a `switch` dispatcher's case
handlers) are **not** automatically disassembled or function-bounded
by this pass -- Ghidra's own `create_function`-equivalent action
reliably fails on such addresses even when the expected bytes are
readable there, because there's no pre-existing code unit for it to
wrap. A full project-wide "Auto Analyze" re-run after loading the
overlay blocks recovers *some* of these (real xrefs/functions do
appear afterward for reachable code), but not all -- some addresses
still need a manual Disassemble-then-Create-Function pass in the
Ghidra UI directly (confirmed reliable where the scripted/MCP path
wasn't).
