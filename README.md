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
  has a 16:9 mode of its own, switched by the console's widescreen setting,
  which widens its camera. Its menus and HUD stay laid out for 4:3 in that
  mode, so they are kept at 4:3 in the middle (`CENTRE_2D`); full-screen
  fades and tints span the picture. The main menu's 3D backdrop is a set
  built for 4:3: in 16:9 it reaches the right edge and leaves the left one
  in the game's clear colour.
- **Colour.** Skaters draw with fog off through a final combiner that
  weights fog per channel; the toolkit used to read the fog register as
  fully fogged there, which tinted every skater purple. Fixed on
  `title/thug`.

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
- **Two images are accepted** (`release/thug.json`): the European disc's
  `default.xbe` and a common hard-drive copy of it (the certificate's allowed
  media widened, and the `jge` at 0x001791B9 that follows XAPI's disc check
  made a `jmp`). Both reach gameplay.

The toolkit fixes this game needed (on `title/thug`):

- **More than 512 textures in a frame.** The in-engine cutscenes bind about
  600 distinct textures a frame. The texture cache kept 512 and could evict
  none bound in the current frame, so everything after the 512th sampled
  white; the loading screen, drawn last, came out pure white between scenes.
  The cache now grows for such a frame.
- **The disc's security sector.** THUG's certificate asks XAPI to check the
  security sector as well as the security page. The toolkit refused that read,
  so the disc's own `default.xbe` rebooted into itself and quit. It is now
  answered with a sector made for the running title.
- **Fog off means no fog.** The combiners read the fog register's alpha from
  the vertex stage even with fog disabled, where the NV2A gives 1. THUG's
  skater shaders weight fog per channel and draw with fog off, so every
  skater came out magenta and purple.
- **A shadow-map test clamps its depth.** The skater's shadow is projected
  onto the ground around them and tested against a depth map drawn from the
  shadow's point of view. Ground beyond that view's far limit compared
  unclamped, failed the test, and came out as dark bands and triangles; it
  now compares as the far limit and is lit, as on the hardware.
- **A menu transition's offscreen pass gets the cleared depth.** During a
  menu change THUG draws the new menu into a 640x480 texture with the
  device's depth and fades it over the old screen. The host keeps a separate
  depth for that texture (the device depth is supersampled, the texture is
  not), which never saw the title's clear, so the 3D skater and city panel
  failed their depth test: half the screen faded to black and an old logo
  smeared across it. The stand-in now gets the device depth's clear.
- **A colour ramp spans the picture.** With 2D kept at 4:3, a full-screen
  quad through a texture at most 8 texels across one way (the pause tint's
  256x4) is a whole-screen pass, like a fade.
- **Pads opened without enumeration** (from THPS3, `title/thps3`).

## Releases

A version tag (`git tag v1.0 && git push origin v1.0`) runs
`.github/workflows/release.yml`, which builds self-building releases for
Linux x86-64, macOS arm64, Windows x86-64 and Android arm64 and publishes them
on the tag's GitHub release. None of them holds game code: the launcher asks
for the player's own disc or folder and builds the game on their machine
(the toolkit's `docs/technical/self-build-release.md`). The recipe is
`release/thug.json`. Run the workflow by hand for a dry run that publishes
nothing. The Android job signs with the `ANDROID_SIGNING_KEY` and
`ANDROID_SIGNING_PASSWORD` repository secrets; a dry run with `sign` off uses
a throwaway key.

## Scripted route

For test runs, `RECOMP_INPUT_SEQ` (milliseconds from the first pad read):
Start every 3 s from 3 s to 24 s reaches Create-a-Skater through the intro and
Story Mode; then `54000:down:150,55500:down:150,57500:a:150` picks DONE, and
the level is playable from about 100 s.
