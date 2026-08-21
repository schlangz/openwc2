# The cutscene engine: resource dispatch, tag interpreter, and timing

This covers the actual runtime machinery that plays cutscene content
-- as opposed to `incident_s00_format.md`, which covers the *data*
format it consumes.

## Layer 1: generic resource-type dispatcher (`sub_21C1A`, resident)

`sub_21C1A` (called directly by, among others, the `campaign.vga`
loader `sub_47005`) is a generic resource-loader dispatcher, not a
per-format parser:

```c
undefined4 FUN_21c1_000a(uint param_1, int param_2, uint param_3,
                          uint param_4, uint param_5, int param_6)
{
    ...
    iVar2 = FUN_21d7_0005(local_1a, param_6);   // reads a small header
    if (iVar2 != 0) {
        uVar3 = local_14 & 0xc0;                 // 2-bit type code
        if ((local_14 & 0xff3f) == 0) uVar3 = 0xc0;
        if (uVar3 == 0x80)      local_6 = (*DAT_344f_2fac)(0x21d7, ...);
        else if (uVar3 < 0x81) {
            if (uVar3 == 0)      local_6 = FUN_2191_0002(...);
            else if (uVar3 == 0x40) local_6 = (*DAT_344f_2fa8)(0x21d7, ...);
        } else if (uVar3 == 0xc0) {
            // inline: allocate a buffer (FUN_268a_00b1), then
            // FUN_21ee_0008(...) to actually load into it
        }
    }
    ...
}
```

**Dispatch is by a 2-bit numeric type code read from the file's own
header, not by comparing `FORM`/`SCNE`/`SHAP`-style ASCII tags.**
Confirmed exhaustively: zero occurrences of any of those tag strings
or their raw 4-byte values (as string literals, as flattened-overlay
byte scans, or via Ghidra's own string engine before/after a full
project re-analysis) exist anywhere in `WC2.EXE`'s code. The container
format's tag names are for humans/tools (the original scene editor,
and this documentation) -- the engine itself never string-compares
against them at the resource-dispatch layer.

`DAT_344f_2fa8`/`DAT_344f_2fac` are two adjacent far-pointer slots (a
small pluggable handler table, type `0x40` and `0x80` respectively) in
a large resident data region (`344f:0000`-`344f:325f`). Both read as
all-zero in the static file -- populated at runtime. **Not resolved**:
which init routine writes them. Multiple static-tracing and live
write-breakpoint attempts (see `methodology.md`) failed to catch the
write, including confirming the table is *still* zero all the way
through a full playthrough of the opening studio-logo intro -- meaning
`title.vga`/`wc2logo.vga` load via type `0` or `0xC0`, never touching
this table, and `campaign.vga` never reaches dispatch at all (it fails
its file-existence check first and skips out). The table is plausibly
only populated on first use of a resource category the intro itself
never needs.

## Layer 2: the FORM-tag interpreter (`OVR_137_seg183F`, overlay)

This is the real chunk-tag walker for the `.S00` scene-script format
-- found by live-tracing an actual intro playthrough (see
`methodology.md`), not by static search (the tag-string search that
came up empty at Layer 1 turned out to be the wrong layer to look at;
this one genuinely does read/copy the raw 4-byte tag, just switches on
a *derived* classifier byte from it rather than the tag itself
directly).

Confirmed live: during actual intro playback, the literal ASCII bytes
`SCNE` were captured sitting on the current stack frame
(`[BP+0x30]`), proving the raw tag really is read and copied through
this pipeline.

Full decompile, function `FUN_6308_04b0` (Ghidra address `6308:04b0`
after manual disassemble+create-function in the Ghidra UI -- the
scripted/MCP path couldn't reach this address, see
`overlay_format.md`'s caveat):

```c
uint FUN_6308_04b0(undefined2 param_1, undefined4 param_2)
{
    ...
    func_0x000106cd();                  // low-level char/byte read
    uVar27 = func_0x0000fc21();
    local_2 = (uint)uVar27;
    if (local_2 != 4) return local_2;   // validates a 4-byte tag was read
    ...
    uVar8 = (int)*(char *)0x43c0 - 0x3b;  // classifier byte -> 0..8 index
    if (8 < uVar8) return uVar8;          // out of range -> unhandled
    switch (uVar8) {
      case 0: ...   // walks a linked list of records (insert/match/
                     // relink via FUN_6745_019c) -- plausibly SYMB
                     // (symbol-table) handling
      case 1: ...   // shorter handler, several fixed literal calls
      case 3: ...   // pure state-check/classification, no I/O
      case 4: ...   // by far the largest case: calls into a SEPARATE
                     // overlay module (ovr_module_148_seg1880),
                     // several global-flag writes, processes what
                     // looks like 4 sequential 31-byte sub-records
                     // (0xa204/0xa223/0xa240/0xa25f) -- plausibly the
                     // top-level SCNEFORM/CSCPFORM "run this scene"
                     // handler
      case 5: return uVar8;   // trivial passthrough
      case 6: ...   // loops slots 0x30..0x36 (exactly 7 slots,
                     // matching the __slot0..__slot7 symbols
                     // cataloged in every INCIDENT.S00 scene) -- the
                     // per-object slot-array processor
      case 7:        // *** this is the live-traced path ***
        uVar8 = (*(code *)*(undefined2 *)0xdf16)();  // indirect call
                     // gate -> lands in the decompressor overlay
                     // (see Layer 3)
        pcVar1 = (char *)((int)param_2 + 2);
        *pcVar1 = *pcVar1 + '\x01';       // increments a counter field
                     // in the caller's own context struct
        return uVar8;
      case 8: lVar28 = FUN_6ea4_0460(puVar12); ...
    }
    ...
}
```

This is the **9-case switch dispatch** matching the 9 real chunk tags
seen in the format (`FORM`/`SCNE`/`CSCP`/`SHAP`/`FILM`/`SPRT`/`SYMB`/
`SEQU`/`PLNE`). The exact classifier-byte-to-tag mapping (`char -
0x3b`, cases 0-8) isn't byte-for-byte proven against each tag name --
that would need tracing `func_0x000106cd`/`func_0x0000fc21` (the
char-read primitives) precisely. What *is* proven: this is a real,
general chunk-tag switch with 9 substantial, distinct handlers,
confirmed reachable with real cross-references (3 real call sites from
`ovr_module_182_seg18E8`, per Ghidra's own analysis) and directly tied
by live evidence to the actual `SCNE` tag during real playback.

## Layer 3: the decompressor (`OVR_142_seg1858`, overlay)

Reached via the indirect call gate in case 7 above. The module's own
entry point (`ovr_module_142_seg1858`, Ghidra `6745:0000`) is a
low-level token/character decoder for an LZ-style compression scheme:
reads one byte, classifies it by range (high-bit-set -> strip and
treat as an extended code; control-range 0x00-0x1F -> special/
terminator handling, including one specific `'T'` followed by `'H'`
2-byte escape sequence; otherwise -> two table lookups keyed by the
byte value) to produce a `(length, repeat-count)`-style pair, which
gets written into fields of a passed-in decoder-state struct. The
actual bulk-copy loop this feeds (`repe movsw`/`rcl cx,1`/`repe
movsb`, a classic odd-byte-aware block copy) was observed directly
live, mid-execution, during intro playback.

The caller (traced live from the stack, in a different function
within this same module, around offset `0x1900`) builds the call by
pulling fields at offsets `+0x06`/`+0x08`/`+0x0A`/`+0x0E` out of a
passed-in structure pointer (`ES:BX`) -- consistent with a generic
"decompress resource described by this descriptor struct" call
signature, reused across whatever asset type reaches this path.

## The 20Hz cinematic timer

Separately from the chunk-tag/resource-loading machinery above,
`sub_4891B` (the same resident function that loads `title.vga`/
`wc2logo.vga`/`arrow.vga` -- see `intro_sequence.md`) installs a
**custom, PIT-hardware-driven 20Hz timer** before doing those loads:

```
sub_4891B: push 0x14 (20 decimal)
  -> sub_28EA0 (stub) -> sub_632A0
       allocates a buffer (size = rate*16), relocates/installs a
       custom INT 08h handler, then:
       -> sub_634E0 (also reachable independently for teardown:
                       restores INT 08h/09h/15h vectors, frees the
                       buffer)
       -> [via sub_29050 stub] -> sub_637D0
            normalizes the requested rate (rounds to nearest multiple
            of 60), then:
            -> sub_2009D(desired_hz)
                 dx:ax = 0x0012:34DEh = 1,193,182  -- the REAL Intel
                 8253/8254 PIT base clock, split across two 16-bit
                 mov-immediate instructions (why a plain 32-bit
                 immediate search for 1193182 finds nothing -- it's
                 never stored as one operand)
                 divisor = 1193182 / desired_hz
                 out 43h, 0x36     ; PIT cmd: ch0, mode3, lobyte/hibyte
                 out 40h, divisor_lo
                 out 40h, divisor_hi
                 (requests <= 18Hz just use the default divisor 0,
                  i.e. the standard 65536 -> 18.2Hz BIOS rate, instead
                  of reprogramming)
```

`sub_4891B` also separately sets `word_33A8F = 0x14` and
`byte_33A78 = 0x14` (20 again) right around the same point -- 20 is a
deliberate, pervasively-set "intro/cinematic rate" constant, not a
one-off argument to a single call.

**This same 20Hz constant appears independently in the *Kilrathi
Saga* Win32 reconstruction** (`wc2-re`, a from-scratch decompiled
reconstruction of the different, later Win32 `WC2.EXE` build):
`SetCinematicFrameTiming(20.0f)`, fired when speech/lip-sync audio
starts. Two independently reverse-engineered codebases -- one live DOS
disassembly, one Win32 decompile-reconstruction -- landing on the
identical rate is strong convergent evidence this is a real,
deliberate design constant carried across both versions of the
engine, not a coincidence of either analysis.

**Relationship between the timer and the dispatcher above**: proven at
the call-chain/global-state level (the 20Hz rate gets installed as
ambient state by the same function that kicks off intro asset loading,
before any scene content is processed, and stays active through
whatever follows) -- **not** proven at the instruction level (i.e. no
specific "read the tick counter" instruction has been located inside
the Layer 2 dispatcher or its handlers). The custom `INT 08h` ISR
body itself (installed at `CS:0x370` relative to its installer, per
static disassembly) was never located/decompiled. Confirming the exact
read site would need either tracing that ISR directly or a live
capture watching its tick counter increment alongside dispatcher
activity.
