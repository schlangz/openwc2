# Methodology: correlating static addresses with live DOSBox memory

Reusable technique for further work on this binary (or any similarly
Borland-overlay-managed DOS executable).

## Tools used

- **IDA (idalib/headless)** for static disassembly, byte reads,
  regex/byte-pattern search, and xref queries against `WC2.EXE`'s
  resident image plus whatever IDA's own auto-analysis reached.
- **Ghidra**, with the flattened-overlay-loading approach from
  `overlay_format.md`, for anything needing to see inside the 124
  `CODE|OVR` overlay modules -- IDA's own static view doesn't resolve
  overlay call gates (they're `INT 3Fh` placeholders in the file, only
  ever resolved at runtime).
- **DOSBox Staging**, driven via its debugger HTTP bridge, for live
  register/memory reads, breakpoints (execution and memory-write), and
  full CPU instruction-trace logging (`LOGL`).

## The live/static delta problem

Static tools (IDA, Ghidra) address the file using its own declared
segment numbering. The *live*, running-in-DOSBox process is loaded at
some real-mode segment DOS chose at load time -- unknown in advance,
and **not** simply "static value + a positive load offset" for this
binary. Naive approaches that worked for a different, unrelated
project's own DOS binary did not transfer cleanly here:

- **MZ header's own `SS` field** (`0x2776`, unrelocated) does not
  reconcile with the live `SS`/`DS` actually observed (`0x1B6F`,
  confirmed stable across multiple pause points and a full process
  restart) -- a live value *smaller* than the static header value is
  impossible under naive relocation (the load offset is always
  positive). Conclusion: this game sets up its own stack/segments
  explicitly during startup rather than relying on the loader-provided
  one. Do not trust this field as an anchor for this binary.
- **Assuming "current CS at the program's very first instruction
  equals the delta"** also failed -- pausing "as early as possible"
  after launch in practice still lands well into Borland C runtime
  startup code (CPU/FPU detection loops), not literally offset 0 of
  the load module, and the bytes there didn't match the file's own
  start-of-module bytes at all.

## What worked: full memory dump + known-string correlation

1. Pause the DOSBox debugger at any point during actual gameplay (the
   game must be *running*, not at a DOS prompt).
2. `MEMDUMPBIN 0 0 A0000` via the debugger console (`debugger_command`
   tool) -- dumps the full 640KB conventional-memory range to
   `MEMDUMP.BIN` in DOSBox's own **host working directory** (not the
   guest DOS current directory -- for this setup that's
   `D:\Git\dosbox-staging\build\release-windows\Release`, found by
   asking the user directly rather than guessing/searching the whole
   filesystem).
3. Search that dump for several **known resident string literals**
   whose exact static (file) addresses are already known from IDA
   (e.g. `arrow.vga`, `stars.00`, `campaign.vga`, `wc2logo.vga`,
   `title.vga` -- ordinary ASCII, trivial to `bytes.find()` for).
4. For each hit: `delta = static_flat_address - live_flat_address`.

All 5 independent string anchors gave the **exact same delta**,
`0xE070`, in this session. That agreement across 5 unrelated addresses
is what makes the delta trustworthy -- a single lucky match would not
be. Confirmed **deterministic across a full process restart** too (a
second, independent restart and memdump reproduced the identical
delta and the identical live addresses for the same strings).

```
live_flat  = static_flat - 0xE070
```

Convert to segment:offset for arming a DOSBox breakpoint via
paragraph alignment: `segment = live_flat >> 4`, `offset = live_flat
& 0xF` (or any other valid seg:off decomposition of the same flat
address -- real-mode addressing is not unique).

**Caveat, confirmed the hard way**: this delta only holds for
*resident* addresses (the fixed DGROUP/data region and non-overlay
code). It does **not** apply to overlay-swapped code/data -- those
live at whatever segment VROOMM happened to place that specific
module at for *this* load, which changes across overlay swaps within
a single run, not just across restarts. Live return addresses
captured mid-execution while inside overlay code need per-instance
resolution (see below), not this constant.

## Resolving a live overlay-code address to its real module

When a live address (e.g. a return address read off the stack) turns
out to be *inside* overlay code, the resident delta doesn't apply.
What worked instead: read the live bytes at that address directly,
then search for that exact byte sequence inside the *flattened*
overlay image (`flatten_all`'s output, see `overlay_format.md`) rather
than the raw per-module extracts -- the flattened image has already
had cross-module fixups resolved to consistent synthetic addresses, so
a plain byte match is reliable. The manifest CSV's `byte_offset` +
`codesz` columns then identify which module (`seg_index`/`real_seg`)
the match falls in, and the *offset within the module* carries over
identically between the live address and the flattened-buffer address
(only the segment changes for overlay swapping, not in-module
offsets) -- a useful sanity check that the match is correct.

From there, `synthetic_seg` (from the manifest) + the loader script's
own reported delta (printed when it ran, e.g. `0x1000` in this
project's Ghidra session) gives the exact Ghidra address to decompile.

## Full-instruction-trace logging (`LOGL`)

`LOGL <hex_instruction_count>` via `debugger_command` resumes
execution, traces every instruction (with full register state) for
that many instructions, and writes the log to `LOGCPU.TXT` in the same
host working directory as `MEMDUMP.BIN`. Useful for catching a
specific upcoming call's live target when single-stepping by hand
would be too slow (e.g. confirming an indirect `call far word
[ptr]`'s actual resolved destination) -- grep the log for the
instruction of interest and read the destination directly out of the
following line's address field, rather than trying to compute it.

Caveat: a poll/wait loop (`test al,08 ; je $-5`-style) can burn an
entire `LOGL` budget without making forward progress if the awaited
condition never becomes true while the emulator is effectively
stalled for the read -- if a trace comes back still sitting at the
same `EIP`, increase the instruction count substantially rather than
assuming the technique failed.

## Breakpoints: known bridge quirk

Newly-armed breakpoints (both execution `debugger_add_breakpoint` and
memory-write `BPM` via `debugger_command`) sometimes don't reliably
register with this bridge until the user manually pauses the emulator
once (Alt+Pause) after arming. Not fully explained; a cheap, harmless
step to include after arming any new breakpoint before resuming.

## What this did *not* solve

Chasing the write site for `DAT_344f_2fa8`/`DAT_344f_2fac` (the
resource-type handler-pointer table, see `cutscene_dispatcher.md`)
via write-breakpoints failed twice, including after establishing the
correct live address via the memdump-correlation technique above and
confirming (by re-checking the value both before and after a full
intro playthrough) that it never actually gets written during that
window -- i.e. the breakpoints were correctly placed and correctly
never fired, not silently missed. The write, if it happens on this
code path at all, happens on some later trigger than the studio-logo
intro.
