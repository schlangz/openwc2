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

**Correction after tracing further**: the initial hypothesis here was
that mouth state (and cutscene state generally) only advances once per
*rendered* frame, with no independent fixed-rate update -- i.e. that
the whole engine needed decoupling from the render loop. Tracing the
actual frame-present script opcode (`case 0x7c` in `RunCutsceneScript`,
`screens.c`) disproves that: it already does a correct real-time
busy-wait (`while (g_nInputClock < g_nNextCutsceneFrameClock)
PumpWindowMessages(0);`, and `PumpWindowMessages` is confirmed to be
`g_nInputClock`'s own refresh site, called on every iteration of that
wait) before presenting each frame and re-arming the next frame's
deadline. That part of the architecture is sound -- a full "decouple
everything from the render loop" rewrite is **not** the right fix and
was not implemented.

Two real, separate, much more surgical bugs were found and fixed
instead:

### Bug 1: `SetCinematicFrameTiming(70.0f)` is a symptom patch for a real problem, not an isolated bug

`AnimateCutsceneSpeakerMouth` drives the mouth from the caption *text*,
at its own per-letter pace, entirely independent of how long the
actual speech *audio* clip runs. On this Kilrathi Saga release the
audio is real recorded voice acting (unlike the DOS original's
synthesized speech), so nothing guarantees the text-driven mouth
timing still matches the recorded clip's real length. If the audio
finishes first, the mouth keeps moving with no sound playing --
visibly broken. `ServiceSoundSystem`'s 20-tick grace period followed
by `SetCinematicFrameTiming(70.0f)` (`sound.c`) is the original
engine's own fix for exactly that: once audio's confirmed stopped,
spike the frame rate to rush through whatever text-driven animation is
still pending. Real motivation, bad implementation -- it also speeds
up every other sprite/plane/sequence on screen for that window, not
just the mouth, and the magic `70` appears nowhere else in the engine.

**Fix applied**: perform the same reset `AnimateCutsceneSpeakerMouth`
already does on its own once it notices speech is inactive, directly,
at the exact point `ServiceSoundSystem` decides speech has truly
ended -- clear the speech-active/text-advance flags and the speech
cursor, and force the current speaker sprite's frame back to neutral
(`11`) immediately, instead of either the original rate-spike hack or
just leaving the text to finish at its own (now unmatched) pace.

### Bug 2: cutscene script opcode `0x8f`'s fps-to-tick-delay divisor is off by one

Found while re-examining what "cutscenes running at different speeds"
independently of the mouth-specific issue could mean. `RunCutsceneScript`
(`screens.c`) has two script opcodes that set the per-scene frame rate:
`0x8d` takes an already-computed tick delay straight from the script
(no arithmetic), and `0x8f` takes a requested fps `value` and computes
`g_nCutsceneFrameDelay_00499c8c = 0x3b / value` -- **59**, not **60**.
Since `g_nInputClock` ticks in exact 1/60s units (confirmed above),
converting fps to a tick delay is `60/value`; with integer division,
`59/value` truncates to a *smaller* (faster) delay than `60/value` for
nearly every requested rate -- e.g. a scene asking for 20fps gets
`59/20=2` ticks/frame (~30fps, 50% too fast) instead of the correct
`60/20=3` (20fps, matching the 20Hz rate used everywhere else in the
engine). Different requested rates truncate by different amounts,
which is exactly consistent with "some cutscenes noticeably too fast,
others closer to correct" rather than one uniform speed error.

**Fix applied**: `0x3b` -> `0x3c` (59 -> 60).

### Status

Both fixes are implemented, syntax-verified against the project's real
compiler flags, and pushed to a branch on this project's `wc2-re` fork
for in-game testing before any upstream PR -- see that repo's own
commit history on branch `fix/cutscene-speech-complete-framerate-spike`
for the exact diffs and full reasoning in each commit message.

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
