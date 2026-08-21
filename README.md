# openwc2

Reverse-engineering notes for the DOS build of *Wing Commander II*
(`WC2.EXE`, retail/Kilrathi-Saga-included DOS install) and, where
relevant, cross-references against the *Kilrathi Saga* Win32
reconstruction ([`wc2-re`](https://github.com/neuromancer/wc2-re)).

No game data or copyrighted assets are included. This repo holds
analysis notes, addresses, and byte-level format documentation only.

## Status

Everything here comes from one focused investigation into the intro
cutscene pipeline: how `WC2.EXE` loads and plays its opening sequence
(Origin FX logo -> WC2 logo -> studio credits -> story-setup scene),
and the general-purpose engine underneath it that also drives every
other in-game cutscene. See `docs/` for the full write-up.

## Docs

- [`docs/overlay_format.md`](docs/overlay_format.md) -- `WC2.EXE`'s
  Borland VROOMM/FBOV overlay format, confirmed byte-for-byte
  compatible with existing extraction tooling built for a different
  Origin/EA DOS title.
- [`docs/intro_sequence.md`](docs/intro_sequence.md) -- the real,
  traced boot chain from `_main` through the opening logo screens.
- [`docs/incident_s00_format.md`](docs/incident_s00_format.md) -- the
  `.S00` scene-script container format (`FORM`/`SCNE`/`SHAPFILE`/
  `FILMFILE`/`SYMB` chunks), plus a full catalog of the 115 named
  scenes found in `INCIDENT.S00`.
- [`docs/cutscene_dispatcher.md`](docs/cutscene_dispatcher.md) -- the
  actual engine that plays cutscenes: the resource-type dispatcher,
  the FORM-tag switch/interpreter, and the custom 20Hz PIT timer that
  paces intro playback.
- [`docs/methodology.md`](docs/methodology.md) -- the live-tracing
  technique used to correlate static (file) addresses with live DOSBox
  memory addresses, reusable for further work on this binary.

## Tooling

The overlay-extraction tooling referenced throughout (`fbov_extract`,
`flatten_all`, the Ghidra overlay-loader script) lives in a sibling
project, not duplicated here -- see `docs/overlay_format.md` for the
exact reuse notes.
