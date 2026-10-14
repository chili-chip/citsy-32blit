# citsy-32blit

A [32blit](https://github.com/chili-chip/32blit-sdk) player for the [citsy](https://github.com/chili-chip/citsy) engine — a headless C++ reimplementation of [Bitsy](https://bitsy.org).

This repository is generated from the [chili-chip/game-template](https://github.com/chili-chip/game-template). It is the Phase 4 host: citsy stays engine-only; this project opens a window (or drives the handheld), maps buttons, blits the 128×128 framebuffer, and plays square-wave audio.

The player boots into a 128×128 retro launcher (game carousel, settings, pause overlay) designed for a Waveshare RP2350 Plus + 128×128 16-bit SPI OLED, using the chili-chip 32blit SDK.

Documentation is in [`docs/`](docs/index.md) and published next to the web player at [chili-chip.github.io/citsy-32blit/docs](https://chili-chip.github.io/citsy-32blit/docs/). The Emscripten game stays at the Pages root; MkDocs is nested under `/docs` by `.github/workflows/build.yml`.

Preview locally with:

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

## Requirements

- CMake 3.20+
- C++20 compiler
- [32blit SDK](https://github.com/chili-chip/32blit-sdk) (sibling checkout at `../32blit-sdk`, or set `32BLIT_DIR`)
- [32blit tools](https://github.com/32blit/32blit-tools): `pip install 32blit "bitstring<5"` (bitstring 5 breaks the tools)
- SDL2, SDL2_image, SDL2_net (desktop)
- A [citsy](https://github.com/chili-chip/citsy) checkout (sibling at `../citsy`, or `-DCITSY_DIR=...`). Without one, CMake fetches the pinned commit from `CMakeLists.txt`. A local checkout is used as is and overrides the pin.

## Desktop build

From this directory, with `32blit-sdk` and `citsy` as siblings:

```bash
cmake --preset linux
cmake --build --preset linux
./out/build/linux/citsy-32blit.elf
```

Equivalent without presets:

```bash
cmake -B build -D32BLIT_DIR=../32blit-sdk
cmake --build build
./build/citsy-32blit.elf
```

The launcher lists bundled games (`Mossland`, `Sandbox`) plus any `.bitsy` files in a `bitsy-games/` folder next to the binary (created on first run) or next to the binary itself. A game is started only when you press **A**, or when a launch path is passed (see below).

## Loading `.bitsy` files

Games are plain-text Bitsy exports. This player loads them in three ways:

1. **Bundled assets** — `assets/mossland.bitsy` and `assets/sandbox.bitsy` are packed into the binary by the 32blit asset tool (`assets.yml`, `type: raw/binary`). They are registered with `File::add_buffer_file` so they behave like real files.
2. **Filesystem** — any `*.bitsy` in the `bitsy-games/` folder, or at the top of the storage, appears in the launcher, sorted by file name. On the console this is the **CHILICHIP** USB drive; on desktop it is the binary directory (`SDL_GetBasePath`). The launcher creates `bitsy-games/` on first run. Files show as `DRIVE` in the list, bundled games as `ROM`.
3. **Launch path** — skip the menu and boot a file the same way the 32blit console does:

```bash
./out/build/linux/citsy-32blit.elf --launch_path /path/to/game.bitsy
```

Author games at [bitsy.org](https://bitsy.org), export `.bitsy`, and drop the file in.

### Controls

| Action | 32blit | Desktop keyboard |
|---|---|---|
| Move cursor / walk | D-pad | Arrows / WASD |
| Confirm / interact | A | Z |
| Back / cancel | B | X |
| Settings (desktop only) | X or MENU (from launcher) | C or Escape |
| Pause overlay (in game) | MENU | Escape or `2` |
| System menu (ChiliChip OS) | HOME (hold to quit to home) | — |
| Volume / brightness | HOME system menu on the console; Left / Right in Settings on desktop | Left / Right |

Pause overlay items: **Resume**, **Restart**, **Exit** (back to the launcher). MENU or B on the overlay resumes.

On the console, ChiliChip OS owns volume, brightness and the battery reading (`blit::get_system_settings()`, API 0.3+), so the launcher has no Settings row there and changes nothing on the hardware. On desktop, Settings (volume and a brightness veil) are written to 32blit save slot 0 when you leave the screen.

## ChiliChip OS (VGC Zero)

citsy installs on ChiliChip OS like any other game, then plays the `.bitsy` files you put on the console.

1. Build the installable `.blit` (or download `citsy-blit` from the CI run or the release):

   ```bash
   cmake --preset vgc-blit
   cmake --build --preset vgc-blit
   ```

   This writes `out/build/vgc-blit/citsy-32blit.blit`.
2. Plug the console in while the home screen is showing. A drive named **CHILICHIP** appears.
3. Copy `citsy-32blit.blit` to the top of the drive and eject it. The console installs citsy and it appears on the home screen.
4. Copy your `.bitsy` files into the `bitsy-games` folder on the drive (create it if citsy has not run yet), then eject. The OS installer only picks up `.blit` files, so `.bitsy` files stay where you put them.
5. Open citsy from the home screen, pick a game and press **A**.

The drive is read-only to the computer while a game (citsy included) is running, so copy files while the home screen is showing. Build with a 32blit SDK at least as new as the console's loader expects; "Needs a system update" on install means the loader is older than the SDK.

## Standalone VGC Zero build

A standalone `.uf2` replaces ChiliChip OS on the console. It is handy for debugging over SWD; flash `blit-loader.uf2` (or `vgc-os-factory.uf2`) afterwards to get the home screen back.

```bash
cmake --preset vgc
cmake --build --preset vgc
```

Or:

```bash
cmake -B build.vgc \
  -D32BLIT_DIR=../32blit-sdk \
  -DPICO_SDK_PATH=../pico-sdk \
  -DPICO_BOARD=chilichip_vgc \
  -DPICO_PLATFORM=rp2350-arm-s \
  -DCMAKE_TOOLCHAIN_FILE=../32blit-sdk/pico2.toolchain
cmake --build build.vgc
```

See the [VGC notes in the 32blit SDK](https://github.com/chili-chip/32blit-sdk/blob/master/docs/vgc.md).

Optional compile definitions:

- `CITSY_BATTERY_ADC_PIN` / `CITSY_BATTERY_ADC_CHANNEL` — override the battery sense pin (default GPIO26 / ADC0)
- `CITSY_BACKLIGHT_PIN` — PWM brightness pin; if unset, device brightness is SSD1351 `CONTRAST_MASTER` (OLED has no backlight). Desktop still uses a software veil.

## How it talks to citsy

```
32blit init/update/render  →  launcher state machine  →  CitsyBlitHost  →  citsy::Engine  →  .bitsy text
```

States: `MainMenu` → `Settings` / `GameRunning` → `PauseOverlay`. Each frame the host reports `delta_time_ms()` and button state. The engine writes a 128×128 colour-index buffer (and two square-wave channels). `CitsyBlitHost::draw()` palettes that buffer and integer-scales it into the framebuffer (1× on the 128×128 OLED).

The core library never links 32blit. See [citsy Host API](https://teriyakigod.github.io/citsy/host/).
