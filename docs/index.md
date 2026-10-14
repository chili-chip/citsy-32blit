# citsy-32blit

A [32blit](https://github.com/chili-chip/32blit-sdk) player for the [citsy](https://github.com/chili-chip/citsy) engine — a headless C++ reimplementation of [Bitsy](https://bitsy.org) — running on a **Waveshare RP2350 Plus** with a **1.5" 128×128 RGB OLED**.

citsy stays engine-only. This repository is the handheld host: it opens no windowing code of its own, maps buttons through the 32blit API, blits a 128×128 framebuffer, plays square-wave audio, and boots into a retro launcher (game list, settings, pause overlay).

<div class="grid cards" markdown>

-   :material-chip: **RP2350 handheld**

    ---

    Dual Cortex-M33 / Hazard3 RISC-V, 128×128 RGB565 OLED, chili-chip 32blit SDK.

-   :material-gamepad-variant: **Launcher + player**

    ---

    Carousel of `.bitsy` games, system settings, and a MENU pause overlay.

-   :material-battery-charging: **On-device telemetry**

    ---

    Battery sense on GPIO26 / ADC0, volume and brightness persisted to flash.

</div>

[Hardware integration](hardware-integration.md){ .md-button .md-button--primary }
[System architecture](citsy-32blit-architecture.md){ .md-button }
[GitHub](https://github.com/chili-chip/citsy-32blit){ .md-button }

## Hardware specifications

| Item | Specification |
|---|---|
| **MCU** | [Waveshare RP2350 Plus](https://www.waveshare.com/wiki/RP2350-Plus) — RP2350A, dual ARM Cortex-M33 / Hazard3 RISC-V, up to 150 MHz |
| **Screen** | 1.5" SPI RGB OLED, **128×128**, 16-bit **RGB565** (65K colours). chili-chip HAL driver: `dbi_ssd1351` |
| **SDK** | Custom 32blit fork: [chili-chip/32blit-sdk](https://github.com/chili-chip/32blit-sdk) |
| **Engine** | [citsy](https://github.com/chili-chip/citsy) (headless Bitsy runtime, sibling checkout or FetchContent) |
| **Power & battery** | Onboard LiPo header on the RP2350 Plus. Charge is sampled with Pico SDK `adc_read()` on **GPIO26 / ADC0** through a 1/3 resistor divider |
| **Audio** | 32blit PWM square-wave channels; master volume is `blit::volume` |
| **Storage** | Bundled `.bitsy` assets in flash; extra games from `bitsy-games/` on the CHILICHIP drive; settings in 32blit save slot 0 |

`blit::screen` is **128×128** on device (`DISPLAY_WIDTH` / `DISPLAY_HEIGHT` in the chili-chip VGC board config). All launcher chrome is authored in that coordinate space.

## Quickstart

### Prerequisites

| Tool | Notes |
|---|---|
| CMake 3.20+ | Presets in `CMakePresets.json` |
| **ARM GCC** | `gcc-arm-none-eabi` for the RP2350 firmware |
| Host GCC/Clang | C++20, for the desktop SDL player |
| [Pico SDK](https://github.com/raspberrypi/pico-sdk) | Sibling checkout at `../pico-sdk` |
| [chili-chip 32blit-sdk](https://github.com/chili-chip/32blit-sdk) | Sibling at `../32blit-sdk`, or `32BLIT_DIR` |
| [32blit tools](https://github.com/32blit/32blit-tools) | `pip install 32blit "bitstring<5"` (packs `assets.yml`; bitstring 5 breaks the tools) |
| [citsy](https://github.com/chili-chip/citsy) | Sibling at `../citsy`, or `-DCITSY_DIR=...`. Without one, CMake fetches the commit pinned in `CMakeLists.txt`. A local checkout is used as is and overrides the pin. |
| SDL2, SDL2_image, SDL2_net | Desktop only |

On Ubuntu / Debian:

```bash
sudo apt install git gcc g++ gcc-arm-none-eabi cmake make \
  python3 python3-pip python3-setuptools \
  libsdl2-dev libsdl2-image-dev libsdl2-net-dev unzip

python3 -m pip install --user 32blit "bitstring<5"
```

Recommended layout:

```text
project_root/
├── 32blit-sdk/      # chili-chip/32blit-sdk
├── pico-sdk/
├── pico-extras/     # pulled by the 32blit Pico HAL as needed
├── citsy/           # chili-chip/citsy (optional, pinned via FetchContent otherwise)
└── citsy-32blit/    # this repository
```

### Desktop (Linux)

```bash
cd citsy-32blit
cmake --preset linux
cmake --build --preset linux
./out/build/linux/citsy-32blit.elf
```

The launcher lists bundled **Mossland** and **Sandbox**, plus any `*.bitsy` in `bitsy-games/` (created on first run) or next to the binary. Press **A** to start a game.

### ChiliChip OS (`.blit`)

```bash
cd citsy-32blit
cmake --preset vgc-blit
cmake --build --preset vgc-blit
```

Copy `out/build/vgc-blit/citsy-32blit.blit` to the top of the console's **CHILICHIP** drive and eject; the loader installs it onto the home screen. Then copy `.bitsy` files into `bitsy-games/` on the same drive, eject, and open citsy. CI uploads the `.blit` as the `citsy-blit` artifact and attaches it to releases.

On ChiliChip OS the firmware applies volume and brightness to every game and HOME opens the system menu, so the launcher hides its own Settings row and reads the battery from `blit::get_battery_level()`.

### Standalone device build (RP2350 / VGC)

A standalone `.uf2` replaces ChiliChip OS; use it for SWD debugging.

```bash
cd citsy-32blit
cmake --preset vgc
cmake --build --preset vgc
```

Equivalent without presets:

```bash
cmake -B build.vgc \
  -D32BLIT_DIR=../32blit-sdk \
  -DPICO_SDK_PATH=../pico-sdk \
  -DPICO_BOARD=chilichip_vgc \
  -DPICO_PLATFORM=rp2350-arm-s \
  -DCMAKE_TOOLCHAIN_FILE=../32blit-sdk/pico2.toolchain
cmake --build build.vgc
```

The firmware binary is `out/build/vgc/citsy-32blit.uf2` (and `.elf` for SWD).

### Flashing the `.uf2`

1. Connect the RP2350 Plus over USB-C.
2. Hold **BOOT**, tap **RESET** (or power on while holding BOOT) until the board appears as `RPI-RP2`.
3. Copy the UF2 onto the volume:

```bash
cp out/build/vgc/citsy-32blit.uf2 /media/$USER/RPI-RP2/
```

The board reboots into the launcher. chili-chip VGC units can also enter BOOT by holding **X** and pressing power — see the [VGC notes](https://github.com/chili-chip/32blit-sdk/blob/master/docs/vgc.md) in the SDK.

SWD flash (OpenOCD + CMSIS-DAP) is wired in `.vscode/tasks.json` against `out/build/vgc/citsy-32blit.elf`.

### Controls

| Action | Handheld | Desktop |
|---|---|---|
| Move cursor / walk | D-pad | Arrows / WASD |
| Confirm / interact | A | Z |
| Back / cancel | B | X |
| Settings (desktop only) | X or MENU (launcher) | C or Escape |
| Pause overlay | MENU in-game | Escape or `2` |
| Volume / brightness | HOME system menu (ChiliChip OS) | Left / Right in Settings |

!!! tip "Skip the launcher"
    Pass `--launch_path /path/to/game.bitsy` (desktop) to boot a file immediately, the same way the 32blit console launch path works.
