# Tony Hawk's Underground (Xbox) — static recompilation

A native build of Neversoft's *Tony Hawk's Underground* (Xbox, 2003; EU
disc, title ID `0x4156002D`, XDK 5659) made with the
[DirectXbox-Recomp](https://github.com/phobos665/DirectXbox-Recomp) toolkit.
The game's own code is lifted to C from the XBE and runs natively; Direct3D 8
and DirectSound are replaced at the API boundary.

**No game code or data is in this repository.** You need the files from a
copy of the disc you own.

## Current State

Runs on macOS (Apple Silicon) from boot into Story Mode gameplay: the
Activision and Neversoft movies and the intro (Start skips them), the main
menu, Story Mode and difficulty select, the opening in-engine cutscenes,
Create-a-Skater, and skating New Jersey ("Chapter 1: Hometown Hijinks") with
the HUD, tricks scoring, pedestrians and traffic. No crashes and no
unresolved indirect calls in two-minute runs.

- **60 fps, natively.** The game asks for presentation interval 1 and runs
  one vblank a frame, as it does on the console. In-level it holds 60 with
  about 9 ms of work a frame (120 fps with the cap off on a 120 Hz display).
  The movies are 30 fps content and play at their own rate. Short dips happen
  while a scene loads and, now and then, while the level streams from disc.
- **Sound.** Native DirectSound path: music and dialogue streams and the
  sound effects are all heard (`[AUDIO] health` in the log).
- **Widescreen** (`widescreen = 1`, launcher or `RECOMP_WIDESCREEN=1`). THUG
  has a 16:9 mode of its own, switched by the console's widescreen setting:
  it widens its camera and lays out its HUD for 16:9, so nothing is stretched.
  The main menu's backdrop is a 3D set built for 4:3, and in 16:9 the camera
  sees past its edges to the clear colour (grey-blue bands at the sides);
  the renderer draws what the game asks for, so this is how its own 16:9 mode
  looks.

Not yet checked: other levels and modes, two-player, the create modes beyond
Create-a-Skater, saving, and the sound by ear.

## Layout

```
thug-recomp/
  CMakeLists.txt          the project; pulls in ../xboxrecomp
  src/main.c              entry point, game folder lookup (from the toolkit's template)
  src/recomp_manual.c     the title's own overrides (none needed so far)
  src/recomp/gen/         lifted code (generated, not committed)
  config/seeds/           functions the disassembler cannot find on its own
  scripts/recompile.sh    runs the toolkit's lift pipeline for this game
  game/                   your game files (not committed)
```

## Building

You need the toolkit checked out beside this repository, at
`../xboxrecomp`, at a commit with the fixes this game needs. Until they are
on the toolkit's `main`, that is its `title/thug` branch. Point
`XBOXRECOMP_DIR` elsewhere to use another checkout, both for
`scripts/recompile.sh` and for CMake (`-DXBOXRECOMP_DIR=...`).

The toolkit's own requirements apply: see its `docs/GETTING_STARTED.md`.

1. **Game files.** Extract the disc into `game/`, so that `game/default.xbe`
   exists (`extract-xiso -x "Tony Hawk's Underground (Europe).iso"`).
2. **XDK symbols.** From the toolkit (it builds the XbSymbolDatabase CLI in
   `third_party/XbSymbolDatabase/build`):

   ```bash
   .venv/bin/python -m tools.xdk_symbols ../thug-recomp/game/default.xbe
   ```
3. **Lift.** `scripts/recompile.sh` (about 15 seconds).
4. **Build.**

   ```bash
   cmake -S . -B build/Release -G Ninja -DCMAKE_BUILD_TYPE=Release
   cmake --build build/Release --target thug_recomp thug_launcher --parallel
   ```

## Running

```bash
build/Release/thug_launcher      # settings, then the game
build/Release/thug_recomp        # the game directly
```

F9 shows the frame rate, F10 steps the frame cap, F11 saves a capture of the
frame on screen.

**Saves.** The game keeps its saves on the Xbox's hard disk, which the
runtime keeps in the per-user save folder; test runs belong on a fresh
`RECOMP_SAVE_DIR`.

## What is specific to this game

- **Widescreen declaration.** `recomp_title_widescreen(NATIVE)`: with the
  console flag set the game asks CreateDevice for
  `D3DPRESENTFLAG_WIDESCREEN` and does the rest itself.
- **Seeds:** ten thread routines and indirect-call targets found by running
  it (`tools.seed_from_log`).
- **The disc's `default.xbe`** in the copy this was brought up on was patched
  (the certificate's allowed media widened to all, and at 0x001791B9 the
  `jge` after a call's result made a `jmp`), with the original kept as
  `default.bak`. This project was lifted and run from the patched image. The
  original should behave the same, since the toolkit answers XAPI's media
  checks, but that has not been tried.

The toolkit fixes this game needed (on `title/thug`):

- **More than 512 textures in a frame.** The in-engine cutscenes bind about
  600 distinct textures a frame. The texture cache kept 512 and could evict
  none bound in the current frame, so everything after the 512th sampled
  white; the loading screen, drawn last, came out pure white between scenes.
  The cache now grows for such a frame.
- **Pads opened without enumeration** (from THPS3, `title/thps3`).

## Scripted route

For test runs, `RECOMP_INPUT_SEQ` (milliseconds from the first pad read):
Start every 3 s from 3 s to 24 s reaches Create-a-Skater through the intro and
Story Mode; then `54000:down:150,55500:down:150,57500:a:150` picks DONE, and
the level is playable from about 100 s.
