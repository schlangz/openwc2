# The intro boot sequence

Traced statically from `_main` (flat `0x13e0c`), cross-checked live.

## Call chain

```
_main (0x13e0c)
  -> sub_27D1A (stub) -> sub_4898E
       loads "stars.00" (8.3, numeric extension, NOT .Sxx) into a
       stack buffer via SCOPY@, sets up decompression-related globals
       (word_3408E = 0x20C, etc.), then:
       -> sub_49110
       -> sub_4891B
            - pushes 0x14 (20 decimal) -> sub_28EA0 (stub)
                                        -> sub_632A0
              installs a custom 20Hz PIT timer (see
              cutscene_dispatcher.md) -- this is the SAME function
              that gates the intro's whole cinematic frame rate.
            - loads "arrow.vga" (cursor sprite, for interactive
              playback controls)
            - loads "title.vga" (confirmed by user: the Origin FX
              logo, first screen of the intro)
            - sets word_33A8F = 0x14, byte_33A78 = 0x14 (20 again --
              a second, separate confirmation this is the deliberate
              "intro rate" constant, not just one call's argument)

  -> loop, in _main itself:
       ax = ShowWc2Logo()          [sub_46921 via sub_27C24 stub;
                                     loads "wc2logo.vga", confirmed by
                                     user: the WC2 intro splash]
       if ax == 0: ShowCampaignVga()   [sub_27C1A -> sub_47005;
                                         loads "campaign.vga"]
       elif ax == 1: (alternate branch, sub_27AA6)
       else: skip both
       if <no-advance condition>: goto loop  (replays the WC2 logo)
```

So the logical opening order is: **Origin FX logo (`title.vga`) ->
WC2 splash (`wc2logo.vga`) -> `campaign.vga`** (gated on the WC2 logo
screen's own return value -- most likely "not interrupted/skipped" =
`0` -> proceed).

## `campaign.vga` is a real, intentionally-optional asset

`sub_47005` (the `campaign.vga` loader) explicitly checks file
existence (`sub_24510`) before attempting the load, and on failure
just logs an internal debug message (string at `0x8836`) and jumps
past the load (`jmp loc_47147`) -- **no crash, no visible player-facing
error**. The screen it would have drawn targets a full `320x199`
region (`word_34B5C..word_34B62` = `0,0x13F,0,0xC7`), consistent with
a full-screen splash rather than a UI element.

This file is genuinely absent from at least one real retail DOS
install (confirmed: not present alongside `title.vga`/`wc2logo.vga` in
that install's `GAMEDAT`), and the game plays through cleanly without
it -- the engine was built to tolerate its absence.

## `INCIDENT.S00`: the actual studio-credits + story-setup content

The literal filenames `title.vga`/`wc2logo.vga`/`campaign.vga` above
are all that `_main`'s own static disassembly references directly.
The deeper content -- the scrolling star-field, the "Origin
presents... a Chris Roberts game... directed by Stephen Beeman" credit
text, and the framing story-setup scene (Thrakhath reporting to the
Kilrathi Emperor) -- lives in a *data file*, `GAMEDAT\INCIDENT.S00`,
not as literal strings in `WC2.EXE` itself. See
`incident_s00_format.md` for the full format and scene catalog.

Specifically, one scene block inside `INCIDENT.S00` has a `SHAPFILE`
manifest of exactly `tiger.v00 field.v00 logo.v00 titles.v00` with a
companion symbol table spelling out `tigerscreen starfield starfield2
origin presents ... chris roberts game directed by stephen beeman
logosprite` -- i.e. the real per-word credit-text sprite sheet
(`titles.v00`) and the tiger-logo/starfield assets for that specific
beat of the intro. A separate later scene block (named `Kneel`, and a
related `runit`) uses `straight.v00`+`artdata.v00` and is tagged with
the symbols `Thrakhath`/`emperor` -- plausibly the framing scene
directly following the studio credits.

**Not yet resolved**: the precise call site where `WC2.EXE` opens
`INCIDENT.S00` itself and picks a specific named scene to play first.
`_main`'s own directly-traced chain only gets as far as `campaign.vga`
before this doc's tracing stopped; the transition into `INCIDENT.S00`
scene playback happens somewhere downstream of that (or via a
different call chain, since `campaign.vga` is a single flat `.VGA`
shape file, not the FORM-container `.S00` format).
