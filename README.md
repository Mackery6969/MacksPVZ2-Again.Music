# MacksPVZ2-Again — Music

The FMOD Studio project for [MacksPVZ2-Again](https://github.com/Mackery6969/MacksPVZ2-Again),
a moddable reimplementation of PvZ2 Gardendless in Godot.

See [COPYRIGHT.md](COPYRIGHT.md) first: the audio belongs to PopCap Games,
PopCap Games (Shanghai) and Electronic Arts. It is here for development
reference and is not licensed by this repository.

**This is an unofficial fan work**, not developed, authorised or endorsed by
PopCap Games or Electronic Arts.

## Layout

```
PVZGE.fspro            the Studio project
Metadata/              events, mixer, banks  (the authored work)
Assets/
  Music/               music referenced by events
  SFX/                 effects referenced by events
  PVZGE/               the full extracted library, organised for browsing
    Music/InGame/<World>/
    SFX/Zombies/ Plants/ Shared/ UI/ Board/ ...
```

`Assets/PVZGE/` is the whole recovered library (4187 clips) under the names
recovered from the PVZGE build. Most of it is unreferenced — it is a palette to
pick from, and it costs nothing in a built bank.

`.gitattributes` puts `.mp3`, `.wav`, `.ogg` and `.bank` through Git LFS.

## How the game uses it

The game never names a bank path or a file. It raises a named event, and the
catalog in the main repo (`data/native/audio/events.json`) maps that name to an
FMOD event path. So renaming or restructuring an event here only means editing
that catalog — no game code changes.

Build banks to the main repo's `audio/banks/`. Under Proton the prefix maps
`Z:` to `/`, so that path is reachable from Studio as:

```
Z:\var\home\mackery\pvz-2\audio\banks
```

The master bank **and** its `.strings` bank are both required, or event paths
cannot resolve and every lookup fails silently.

Events added by `tools/fmod_create_events.py` are written **unassigned**, so
they build into no bank until you select them in Studio and assign them to the
Sound bank. The tool leaves bank assignment alone deliberately: guessing at
the relationship name risks a project that will not open.

## Notes on the source material

- **Music is dynamic.** Twelve worlds have four phases (`P1_LP`, then
  `P2_Start`/`P2_LP` and so on), driven by a parameter rather than by swapping
  tracks. Phase values are **0-3**, matching the conditions in this project.
- **`_LP` files are pure loops** — their loop region is the whole file. An
  intro belongs in the matching `_Start` file, outside the loop.
- **Front Lawn has no phases**, only `LP_Tutorial`: a static loop.
- **Egypt ships only `P4_Start`** — there is no `CYS_Start`, `P2_Start` or
  `P3_Start` for it.
- **Zen Garden is a shuffle, not a loop**: 18 short pieces (`zen00`-`zen17`)
  picked at random with a ~5 second gap between them. It wants a shuffled
  multi instrument, not a looping timeline.
