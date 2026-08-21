# Cross-check against `wc2-re` (the Kilrathi Saga Win32 reconstruction)

[`wc2-re`](https://github.com/neuromancer/wc2-re) is a from-scratch,
independently reverse-engineered reconstruction of the *Kilrathi
Saga*'s Win32 `WC2.EXE` (a different, later build than the DOS
executable this repo otherwise documents -- see its own README: no
VROOMM overlays, uses `timeSetEvent`/`timeGetTime` instead of the PIT,
DirectDraw instead of raw VGA). Despite being a different executable
built with a different toolchain for a different OS, it shares the
same underlying game-data formats and a great deal of engine logic
with the DOS build documented elsewhere in this repo -- useful as an
independent, symbol-rich cross-check on everything found by static/
live analysis of the DOS binary alone.

## Confirmed: the FORM-container tag set

`wc2-re`'s `src/disk.c` does literal 32-bit integer comparisons
against packed 4-byte tag values -- the exact same technique
inferred (but not directly provable, since the DOS compiler lowered
it into a jump table) for the DOS dispatcher in
`cutscene_dispatcher.md`. Every tag found there, decoded:

| Packed hex | ASCII | Context |
|---|---|---|
| `0x4d524f46` | `FORM` | `ReadNextSceneForm`'s own container-tag check |
| `0x41544144` | `DATA` | `DecodeCutsceneObjectResource`, holds literal data entries + symbols |
| `0x50524353` | `SCRP` | `DecodeCutsceneObjectResource`, holds scripts + more symbols |
| `0x454e4353` | `SCNE` | `CreateCutsceneResourceInstance` -> allocates a `CutsceneScene`, calls `ExecuteCutsceneScene` |
| `0x454e4c50` | `PLNE` | -> allocates a `CutscenePlane`, calls `UpdateCutscenePlaneObject` |
| `0x55514553` | `SEQU` | -> allocates a `CutsceneSequence`, calls `ExecuteCutsceneSequence` |
| `0x54525053` | `SPRT` | -> allocates a `SceneFlicObject` sprite, calls `UpdateCutsceneSpriteObject` |
| `0x52544f48` | `HOTR` | `WaitForSceneAdvance`, decodes into a hotspot table |
| `0x54585448` | `HTXT` | same function, decodes into a text table |
| `0x50414d43` | `CMAP` | seen in disk.c, not traced further |
| `0x204c4150` | `PAL ` | palette chunk (trailing space -- 3-char tag, space-padded) |

**This directly corrects `incident_s00_format.md`**, which extracted
tag names from `INCIDENT.S00` via a heuristic byte-scan, not a real
parser, and guessed `CSCP`/`SHAP`/`FILM`/`SYMB` for some chunks. Of
those, only the *concept* of a `SYMB`-like name table survives (as
`DATA`'s symbol-entries here); `SCRP` is almost certainly what the
heuristic parser misread as `CSCP` (very similar byte shape); no
`SHAP`/`FILM`/`SYMB` 4-byte tag exists in `wc2-re`'s own comparisons
at all. `INCIDENT.S00` should get a proper parse against this
confirmed tag set rather than trusting the original heuristic
extraction.

## Corrected: `ovr_module_142_seg1858` is the mouth animator, not a decompressor

Originally documented in `cutscene_dispatcher.md` as "the
decompressor," based on DOS-only analysis of a function that reads a
byte, classifies it by range, and does table lookups keyed by the
byte value. Cross-checking against `wc2-re`'s
`AnimateCutsceneSpeakerMouth(SceneFlicObject *sprite)`
(`src/screens.c:486`) makes clear what it actually is: **per-character
speaker mouth-shape animation**, not compression.

The match is exact, not just similar-shaped. `wc2-re`'s digraph
special case:

```c
} else if (character == 'T' &&
           toupper(*g_pszCutsceneSpeechCursor_00499eb0) == 'H') {
    frame = 8;
    duration = 3;
    g_pszCutsceneSpeechCursor_00499eb0++;
}
```

is byte-for-byte the same constants as the DOS function's own special
case (`frame = '\b'` = 8, `duration = '\x03'` = 3, gated on the
current character being `'T'` and the next being `'H'`). The DOS
function's `local_3 + -0x6aec` / `local_3 + -0x6a6c` pointer arithmetic
(Ghidra's rendering of `table_base[character]`) is the same mechanism
as `wc2-re`'s `g_acCutsceneMouthFrames_00499db0[character]` /
`g_acCutsceneMouthDurations_00499e30[character]` lookups.

### The real per-letter mouth tables

From `wc2-re/src/globals.c`. Both are `signed char[128]`, indexed by
`toupper()`'d character code (so the lowercase half, indices 96-127,
is dead/unreachable padding -- confirmed all-zero in both tables).
Indices below `' '` (space) are also never read via the normal
per-character path (control characters short-circuit to `frame = -1`
before the table lookup happens at all), despite having non-trivial
leftover values in the table data itself.

**Meaningfully used range** (`' '` through `'Z'`):

| Char | Frame | Duration | | Char | Frame | Duration |
|---|---|---|---|---|---|---|
| `' '`-`'/'` | -1 (no shape) | 1 | | | | |
| `0` | 3 | 1 | | `A` | 1 | 1 |
| `1` | 5 | 1 | | `B` | 0 | 1 |
| `2` | 5 | 1 | | `C` | 5 | 1 |
| `3` | 1 | 1 | | `D` | 5 | 2 |
| `4` | 5 | 1 | | `E` | 1 | 1 |
| `5` | 9 | 1 | | `F` | 9 | 1 |
| `6` | 5 | 1 | | `G` | 5 | 1 |
| `7` | 5 | 1 | | `H` | 3 | 1 |
| `8` | 1 | 1 | | `I` | 2 | 1 |
| `9` | 2 | 1 | | `J` | 5 | 1 |
| | | | | `K` | 5 | 1 |
| | | | | `L` | 8 | 2 |
| | | | | `M` | 0 | 1 |
| | | | | `N` | 5 | 1 |
| | | | | `O` | 3 | 1 |
| | | | | `P` | 0 | 1 |
| | | | | `Q` | 6 | 1 |
| | | | | `R` | 5 | 1 |
| | | | | `S` | 5 | 1 |
| | | | | `T` | 5 (2 if followed by `H`, then frame 8/dur 3) | 1 |
| | | | | `U` | 4 | 1 |
| | | | | `V` | 9 | 1 |
| | | | | `W` | 6 | 2 |
| | | | | `X` | 5 | 2 |
| | | | | `Y` | 5 | 1 |
| | | | | `Z` | 5 | 1 |

`':'`-`'?'` and `'['`-onward are unused/default (-1 or trivial
fallback values), not real letter data.

**So: no fixed "mouth fps" exists to find.** Hold time for a given
mouth shape is `speechSpeed(default 1) * duration[char]` ticks -- 1
tick for almost every letter, 2 for `D`/`L`/`W`/`X`, 3 for the `TH`
digraph. On Win32 that "tick" is a fixed 1/60s unit of real elapsed
time (see the next section) -- `wc2-re`'s own SDL port additionally
clamps this to a minimum (`WC2_CUTSCENE_MOUTH_MIN_TICKS = 3`,
`include/wc1.h`) -- explicitly commented there as *not* present in the
original, added only so the SDL port's frame pacing doesn't make
already-short holds imperceptible. That comment is itself a clue to
the bug below -- the SDL port's author independently rediscovered that
the original durations are too short to reliably render, without (as
far as this analysis found) tracing all the way back to why.

## Why Kilrathi Saga's cutscenes desync

Traced `g_nInputClock_005c84a8` -- the clock `AnimateCutsceneSpeakerMouth`
compares `waitStart`/`waitTicks` against -- to its actual update site
(`winmain.c`, run once per frame):

```c
g_nInputClock_005c84a8 = GetTickCount();
g_nInputClock_005c84a8 -= g_dwGameClockStart_005d12b8;
g_nInputClock_005c84a8 *= 60;
g_nInputClock_005c84a8 /= 1000;
```

This is **real, continuously-resynced wall-clock time, expressed in
1/60-second units** -- not milliseconds, not a frame counter. Confirmed
independently: `WC2_CUTSCENE_MOUTH_MIN_TICKS = 3` (above) is exactly
`3/60s = 50ms`, matching the engine's own 20fps (`50ms`) cinematic
frame period -- the SDL port's clamp is, in its own units, "never hold
for less than one 20fps frame."

**The bug**: the mouth-duration table is authored in this same 1/60s
unit, but almost every letter has `duration = 1` -- a **16.7ms** hold
at `speechSpeed = 1`, well *under* one 20fps frame period (50ms).
Since mouth state only advances when something actually calls
`AnimateCutsceneSpeakerMouth` again -- which happens once per rendered
frame, not on an independent fixed-rate update -- any requested hold
shorter than the current frame interval is meaningless: by the time
the next check happens, more real time has already elapsed than was
asked for, so the shape always advances "as fast as frames get drawn,"
never at its own intended rate. The comparison against `g_nInputClock`
is correctly wall-clock-based in principle; what's missing is
decoupling *how often that comparison gets evaluated* from *how often
a frame gets rendered*.

That would just mean "mouth flips once per frame, always" if the frame
rate were rock-steady -- annoying, but not "extreme desync." It becomes
that once combined with something already found and documented in this
project's own DOS-side timer analysis, that carries over structurally
here too: Win32's frame pacing is not hardware-clocked the way DOS's
PIT-driven interrupt is. It runs through `timeSetEvent`/`timeGetTime`
plus a busy-wait `Sleep(0)` throttle loop (`ThrottleFrameAndDrawFps`,
`screen.c`) -- inherently jittery, worse on modern fast/multi-core
hardware than in 1996 -- and, concretely, the *Kilrathi Saga* source
has a confirmed real bug where `SetCinematicFrameTiming(70.0f)` fires
the instant a line of speech finishes (`sound.c`, `ServiceSoundSystem`),
spiking the cinematic rate from 20fps to 70fps for a window right at
the exact moment mouth state needs to settle back to neutral. Audio
keeps playing at its own real-time-accurate pace throughout (driven by
the sound hardware/API, unaffected by any of this) -- visual mouth
state, being coupled to however fast frames happen to render rather
than to the clock it's nominally compared against, drifts against it.
Over a whole cutscene with many speech lines, each one hitting this at
its end, that compounds into exactly the kind of pervasive, worsening
desync reported for this game.

DOS structurally cannot exhibit this: the frame-advance interrupt and
the tick source used for duration-counting are the *same* hardware PIT
interrupt (see `cutscene_dispatcher.md`'s timer section), so "requested
hold shorter than one frame" cannot happen there -- the frame period
*is* the tick, by construction, not merely convention.

### A concrete fix for `wc2-re`

Not tested against the actual codebase -- an architectural
recommendation from this analysis, not a submitted/verified patch.

1. **Decouple cutscene-object state advancement (`AnimateCutsceneSpeakerMouth`
   and the general `waitTicks`/`waitStart` consumption in `screens.c`)
   from the render/frame loop.** Run it on its own fixed-rate update --
   ideally every real 1/60s tick (matching `g_nInputClock`'s own native
   unit exactly), independent of how often a frame actually gets drawn
   to screen. This is the standard "decouple simulation rate from
   render rate" fix, and it directly restores the property the DOS
   binary gets for free from having one shared hardware clock: state
   advances at the rate the data was authored for, rendering just
   displays whatever the current state is whenever it gets around to
   drawing a frame. This is the real fix -- everything below is
   secondary/defense-in-depth.
2. **Fix the confirmed `SetCinematicFrameTiming(70.0f)` bug directly**
   (`sound.c`, fires when `g_nSpeechCompletionDelay_004a265c > 20`
   after a speech sound stops). Whatever the intent was (snapping back
   to a faster non-speaking rate), 70fps is not a value used anywhere
   else in the engine and produces a jarring rate spike at exactly the
   worst moment. Likely candidates: it should match whatever rate was
   active *before* speech started (probably the default/menu rate, not
   a magic 70), or this call should simply be removed if its only
   effect is this spike.
3. **Defense in depth, once (1) is fixed**: `ThrottleFrameAndDrawFps`'s
   busy-wait (`while (timeGetTime() < deadline) Sleep(0)`) could be
   replaced with a higher-resolution wait (`timeBeginPeriod(1)` +
   `Sleep` for the bulk of the interval, spin only the remainder) for
   more consistent real frame pacing generally -- worthwhile on its own
   merits, but not the root cause here, and not sufficient by itself
   without (1).

## The Win32-side resource loader (`FetchDiskPacketRetrying`, `disk.c`)

Not fully cross-referenced against the DOS `sub_21C1A`/`DAT_344f_2fa8`
type-dispatch table yet (see `cutscene_dispatcher.md`'s Layer 1 open
question), but this is the clear Win32-side counterpart -- same file
(`disk.c`) also documents the `FORM` chunk-walking logic directly
(`OpenDiskDataFile`/`FetchDiskPacketRetrying`/`PromptInsertNumberedDisk`
per that file's own header comment). Worth a closer pass if the
DOS-side type-table write site ever needs to be understood without a
live capture -- this reconstruction may already have the equivalent
logic named and readable.
