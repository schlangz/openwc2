# `GAMEDAT\INCIDENT.S00`: the scene-script container format

`INCIDENT.S00` (190086 bytes) is **not** an intro-only file -- it's a
shared, FORM-chunked scene-script database covering cutscene content
across the entire campaign (launch-deck sequences, a murder-mystery
subplot, the poker minigame, funeral/court-martial scenes, major story
beats, briefing-room scenes, and the lose-ending) as well as the
opening studio-credits/story-setup sequence. Extraction was done via
targeted byte-pattern scanning (tag positions + printable-run
heuristics), not a full formal chunk parser -- offsets and
`SHAPFILE`/`SYMB` text below are as-extracted and may be imprecise
where a value overruns the heuristic's read window (noted inline
where relevant).

**Tag names below are provisional.** Cross-checking against `wc2-re`
(see `wc2re_cross_reference.md`) confirmed `FORM`/`SCNE`/`PLNE`/
`SEQU`/`SPRT` directly, but found the real Win32-side tag set also
includes `DATA`/`SCRP`/`HOTR`/`HTXT`/`CMAP`/`PAL` instead of this
doc's guessed `CSCP`/`SHAP`/`FILM`/`SYMB`. `SCRP` in particular is
very likely what this doc's heuristic parser misread as `CSCP`. Treat
`SHAPFILE`/`FILMFILE`/`SYMB` chunk names throughout this doc as
unconfirmed until `INCIDENT.S00` gets a proper, non-heuristic parse.

**File order is not playback order.** The first ~15 scene blocks in
the raw file are almost entirely reusable `doit`-named launch-deck
templates plus the `LoseFinale` ending -- not the intro. The real
studio-credits scenes sit roughly 90KB into the file. Whatever decides
*which* named scene to play *when* is external to this file (most
likely `CAMPAIGN.S00`/`SERIES.S00`, going by naming) -- this file is a
shared template library indexed by scene name, the same way
Privateer's own `PROG` mission bytecode picks dialogue lines rather
than playing a file top to bottom.

## Chunk shape

Observed structure (IFF-family, but a custom Origin dialect -- not
literal EA/Amiga IFF):

```
FORM <len> SCNEFORM
  SCRPOFST <script offset table, binary>
  SYMB <len> "<scene name>"
  FORM <len> CSCPFORM
    [optional: FORM TEXT / SYMB "<dialogue line 1>...<line N>"]
    SHAPFILE <len> "<file1> <file2> ..."     -- space-separated list
    FORM <len> FILMFILE
      SYMB <len> "<film file, usually series.s00>"
      FORM <len> SPRTFORM
        SCRPOFST <sprite/animation offset table>
        SYMB <len> "<per-object symbol names: __slot0..__slot7, plus
                     named objects like 'backdrop', 'ship', 'stars'>"
        FORM PLNEFORM ...
        FORM SEQUFORM ...
```

A large, shared verb/opcode dictionary precedes many scene blocks (a
real scene-scripting vocabulary, not per-scene data): `removeall
showslot hideslot delmediumslot setupmediumslot mediumshot
setupuniform setupbackdrop setupbackground initshardrun talking
settalker vidit printit closeup restoredefaultfont removeperson
addplanet removeplanet narrating doshow sidewaysthrusters removestars
whoa`. This is architecturally the same kind of thing as Privateer's
own `MissionTrigger_ExecuteOpcodes` opcode dispatch -- a small,
named-verb scripting surface for cutscene direction, just at the
file-format level here rather than confirmed as native bytecode (see
`cutscene_dispatcher.md` for the actual runtime side of this).

## The credits/story-setup scenes

The scene most directly tied to the opening studio-credits sequence:

```
SHAPFILE: tiger.v00 field.v00 logo.v00 titles.v00
FILMFILE: series.s00
SYMB (object names): __slot0..__slot7, tigerscreen, starfield,
  starfield2, origin, presents, asprite, chris, roberts, game,
  directed, by, stephen, beeman, logosprite
```

The `SYMB` list literally spells out the credit text: *"origin
presents ... a [chris roberts] game ... directed by stephen beeman"*
-- each word is its own named sprite object. `titles.v00` is
specifically this credit-text sprite sheet; `tiger.v00` backs the
`tigerscreen` object (a tiger/Kilrathi logo animation); `field.v00`
backs `starfield`/`starfield2`; `logo.v00` backs `logosprite`.

The following story-setup scene (named `Kneel`, appearing 3 times with
slightly different `SHAPFILE` sets, and a related `runit` scene at the
very end of the file) is tagged with the symbols `Thrakhath`/
`emperor` and uses `straight.v00`+`artdata.v00` -- plausibly the
framing scene of Thrakhath reporting to the Kilrathi Emperor that
opens the story proper, right after the credits.

## Full scene catalog (115 `SCNEFORM` blocks, file order)

Grouped by distinct scene name (many are near-identical reusable
templates -- repeat counts noted). "..." shapfile entries are
heuristic-parser truncations of a longer list, not full content.

| Scene name(s) | Typical `SHAPFILE` | Notes |
|---|---|---|
| `doit` (many, ~40 occurrences) | `launch.v00 laungame.v11 laungame.v12 mcspark.v00 medium.v00 wingman.v00 objects.vga ...`, or `landgame.v0*`, `repbay.v0*`, `doors.v00`, `field.v00`, `midgame.v0*`, `hitdemo.v00`, `nivl.v00`, `powerdn.v00`, `wormhole.v01`, `love.v00`, `ship.v*` | Reusable launch-deck / repair-bay / generic scene template, reused across many mission contexts |
| `LoseFinale` (x3) | `despair.v00` | The lose-ending cutscene |
| `WHAMMO`, `_TRAITOR_TALKS_`, `_TECH_IN_DOORWAY_`, `_TECH_TALKS_`, `_TECH_WALKS_UP_`, `_TECH_SEES_SCREEN_`, `_GUNSHOT_`, `_TECH_FALLS_`, `_DEAD_TECH_` | `murder.v00` (all) | A full murder-mystery subplot sequence, one scene per beat |
| `WARPSHOT` (x6) | `objects.vga wormhole.v01 planets.v00 field.v00`, or `ship.v21 ship.v18`, or `poker.v00` | Jump/warp visual beats |
| `_DEALING_CARDS_DEALT_`, `_CALL_AND_RAISE_`, `_COUNTING_CHIPS_` | `poker.v00` | The poker minigame cutscene |
| `_HOBBES_GETS_CARDS_`, `_SPARKS_WORKING_` | `sparks.v00`, `field.v00 artdata.v00` | Poker-adjacent beats (Hobbes is a named WC2 character) |
| `_DISMISS_`, `thefuneral`, `_TRAITOR_STANDING_` | `funer.v00`, `murder.v00`, `pcship.v04 pilotanm.vga cockpit.vga` | Court-martial / funeral subplot |
| `LockedOnJazz` (x2) | `ship.v04 pcship.v04`, or `tiger.v00 field.v00 logo.v00 titles.v00` | Second occurrence reuses the credits shapfile set -- see caveat below |
| `_THREE_FOR_JAZZ_` | `stride_2.v00 kilhal_1.v00 kilhal_2.v00` | |
| `Thrak_Walk`, `Kneel` (x3) | `artdata.v00`, or `straight.v00 artdata.v00` | Story-setup / Emperor scene, see above |
| `Terran_Grasp` | `wingman.v00 ship.v23 objects.vga` | |
| `_SPIRIT_DEATH_`, `_SHADOW_DEATH_` | `artdata.v00 objects.vga`, `ship.v09 ship.v23` | Ship-loss story beats |
| `Olympus` | `objects.vga wormhole.v01 planets.v00 field.v00` | |
| `_EXPLOSION_ON_THE_FLIGHT_DECK_` | `objects.vga ship.v21 ship.v18` | |
| `hello`, `_PILOTS_LEAVE_BRIEF_`, `_PILOTS_ENTER_BRIEF_`, `_PILOTS_SITTING_` | `briefing.v00`, `pilotanm.vga ship.v10` | Briefing-room scenes |
| `_EJECT_DEATH_`, `_EJECT_RESCUE_`, `_EST_NIVEN_` | `pilotanm.vga ship.v04`, `nivl.v00`, `pcship.v04 pilotanm.vga cockpit.vga` | Ejection/rescue beats |
| `SETUP_SCALE_FACTOR` | `tiger.v00 field.v00 logo.v00 titles.v00` | Shares the credits shapfile set -- see caveat below |
| `_JAZZ_PULLS_GUN_` | `doors.v00` | |
| `_kiss_` (x3) | `love.v00`, or the `doit`-style launch shapfile set | |
| `runthething` (x5) | `launch.v00 laungame.v11 ...` (launch-deck set) | |
| `_BRIEFING_BACKGROUND_`, `_JAZZ_RESCUE_` | `jazzresc.v00`, `launch.v00 planets.v00 helmet.v00 artdata.v00 laungame.v*` | |
| `dumdedum` | `midgame.v03` | |
| `runit` | `straight.v00 artdata.v00` | Final scene in the file, same shapfile set as `Kneel` |

**Caveat on `LockedOnJazz`/`SETUP_SCALE_FACTOR` sharing the credits
`SHAPFILE` set**: this doesn't necessarily mean the WC2-logo/tiger/
starfield assets play again during a `LockedOnJazz` combat moment --
`SETUP_SCALE_FACTOR` in particular reads like a generic
calibration/init scene name, so a shared-asset-list coincidence
(reusing the same 4 files for an unrelated purpose, or the heuristic
parser attributing the wrong `SHAPFILE` block to the wrong `SYMB` name
due to nearby chunk boundaries) is equally plausible. Not
disambiguated.
