# retroDE — Environment Setup

Everything you need to go from a stock DE25-Nano to a working retroDE
system. Read this once, do it once.

## What you need

### Hardware

- **Terasic DE25-Nano** — Intel Agilex 5 SoC FPGA development board.
  This is the only supported board. No external USB-Blaster needed —
  the DE25 has JTAG built in over USB.
- **HDMI display** — retroDE outputs 1080p or 720p over HDMI.
- **USB keyboard** — for OSD navigation and cores that use keyboard
  input (ao486, CoCo2).
- **USB gamepad** (optional) — any Linux-compatible USB gamepad works.
  DualShock 2 wired controllers are also supported via the DS2 GPIO
  header on the DE25.
- **microSD card** — for the HPS Linux image (Terasic provides one
  with the board).

### Software (development machine)

- **Intel Quartus Prime Pro 26.1** — required for Agilex 5 synthesis.
  A free Quartus Pro license is included with the DE25-Nano purchase;
  register your board serial on Terasic's site to get your license file.
- **Linux host** — tested on Ubuntu 24.04. Quartus Pro 26.1 runs on
  Linux; Windows should also work but hasn't been tested.
- **ARM cross-compiler** — `aarch64-linux-gnu-gcc` for building HPS
  userspace software. Install on Ubuntu/Debian:
  ```bash
  sudo apt install gcc-aarch64-linux-gnu
  ```
- **Git** — to clone the repos.

## Repository layout

retroDE is organized as one repo per core, plus this umbrella repo.
Clone them as siblings in a single parent directory:

```
FPGA_Projects/              (or wherever you like)
├── retroDE_splash/          ← platform chassis (REQUIRED, build first)
├── retroDE_nes/             ← NES core
├── retroDE_ao486/           ← ao486 (x86 PC) core
├── retroDE_coco2/           ← CoCo2 core
├── retroDE_Atari2600/       ← Atari 2600 core
├── retroDE_gb/              ← Game Boy core
└── retroDE_docs/            ← (optional) your local copy of Terasic docs
```

**`retroDE_splash` is the foundation.** Every other core depends on
the shared platform RTL in `retroDE_splash/rtl/platform/`. Build splash
first; then build whichever cores you want.

### Cloning

```bash
mkdir ~/FPGA_Projects && cd ~/FPGA_Projects

# Required — platform chassis
git clone https://github.com/Thejayman77/retroDE_splash.git

# Then clone whichever cores you want to build:
git clone https://github.com/Thejayman77/retroDE_nes.git
# git clone https://github.com/Thejayman77/retroDE_ao486.git
# git clone https://github.com/Thejayman77/retroDE_coco2.git
# etc.
```

## Board setup

### SD card

The DE25-Nano ships with a microSD card containing Terasic's stock
Linux image (Debian-based, runs on the Cortex-A55 HPS cores). Use
that image as-is — retroDE doesn't require a custom OS image.

> **Note:** If you need to reflash or reimage the SD card, follow the
> instructions in the DE25-Nano User Manual (Section 7, "Linux on HPS").

### File layout on the board

retroDE expects the following layout under `/home/terasic/` on the
running DE25-Nano:

```
/home/terasic/
├── software/
│   └── retrodesd              supervisor daemon (cross-compiled)
├── cores/
│   ├── manifest.cfg           core registry (tells retrodesd what's installed)
│   ├── retroDE_splash.core.rbf
│   ├── retroDE_nes.core.rbf
│   └── ...
├── bios/
│   ├── boot0.rom              ao486 system BIOS
│   ├── boot1_vbe.rom          ao486 VGA BIOS
│   └── ...                    (CoCo2 ships a clean-room BIOS — no sourcing needed)
├── roms/
│   ├── nes/                   NES ROM files (.nes)
│   ├── coco2/                 CoCo2 cartridge ROMs
│   └── ...
├── DOS/                       ao486 hard drive / floppy images
└── saves/                     SRAM + savestate files (auto-created)
```

Create the directories if they don't exist:

```bash
ssh terasic@<board-ip>
mkdir -p ~/software ~/cores ~/bios ~/roms ~/DOS ~/saves
```

### manifest.cfg

`retrodesd` reads `~/cores/manifest.cfg` to know which cores are
installed and how to launch them.

**Format**: INI-style. `[section]` headers define a core ID; each
section is a list of `key=value` lines. Lines starting with `#` are
comments; blank lines are ignored.

#### Section header

`[<core_id>]` — unique core ID (alphanumeric, max 63 chars). Used
internally and saved to `~/retrode.state` so retrodesd remembers the
last-loaded core.

#### Per-core options

| Key | Type | Required | Default | Purpose |
|---|---|---|---|---|
| `name` | string | No | section ID | Display name shown in the OSD menu |
| `rbf` | absolute path | **Yes** | — | Path to the core's `.core.rbf` |
| `backend` | string | **Yes** | — | Session backend type: `splash`, `rom_simple`, `ao486`, `coco2` |
| `config` | absolute path | No | (none) | Per-core persisted config file |
| `rom_dir` | absolute path | No | (none) | ROM browser directory (`rom_simple` backends) |
| `core_id` | uint32 (decimal or `0x...`) | No | `0` (skip check) | Expected `CORE_ID` register value — fails core load if mismatch |
| `min_abi` | uint32 (decimal or `0x...`) | No | `0` (skip check) | Minimum `ABI_VERSION` required |
| `media` | comma-separated list | No | (none) | Supported media types: `ide`, `floppy`, `cdrom`, `rom` (max 8) |
| `extensions` | comma-separated list | No | (none) | File extensions for the ROM browser: `.nes`, `.gb`, `.bin` (max 8) |
| `bios.<n>` | `slot:path` | No | (none) | BIOS file by slot name; `n` = 0..3 (max 4 BIOS entries per core). Slot names (`bios`, `vgabios`, etc.) are backend-specific |

**Limits**: 16 cores total per manifest, 4 BIOS slots per core,
8 media types per core, 8 extensions per core.

**Validation**: cores missing `rbf` or `backend` are skipped at parse
time with a warning. The `core_id` and `min_abi` checks are skipped
when set to `0`; otherwise mismatches against the FPGA's identity
registers will fail the core load.

#### Example

Each core's repo includes a recommended manifest snippet to copy
into your `manifest.cfg`. The example below shows all options on a
single fake entry — use it as a template.

```ini
# retroDE core manifest

[example]
name=Example Core
rbf=/home/terasic/cores/retroDE_example.core.rbf
backend=rom_simple
config=/home/terasic/example.cfg
rom_dir=/home/terasic/roms/example
core_id=0x12345678
min_abi=0x0100
media=rom,floppy
extensions=.rom,.bin
bios.0=bios:/home/terasic/bios/example_boot.rom
bios.1=auxbios:/home/terasic/bios/example_aux.rom
```

### Starting retrodesd

`retrodesd` runs as root (it needs `/dev/mem` access for the FPGA
bridge). A systemd service template is provided in the splash repo:

```bash
# On the DE25-Nano:
sudo cp ~/software/retrodesd.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable retrodesd
sudo systemctl start retrodesd
```

The service template lives at `retroDE_splash/software/retrodesd.service.example`.
Copy it to the board and rename to `retrodesd.service`.

## First-time QSPI flash

The DE25-Nano boots from QSPI flash — at power-on, the FPGA configures
from QSPI and the HPS first-stage bootloader (U-Boot SPL) runs from
the same image. Without a compatible image in QSPI, the HPS won't reach
Linux and runtime core swaps will hang the fabric.

You need to do this **once** when first setting up the board, and
**again after upgrading Quartus** to a different version (the QSPI
image must be built with the same Quartus version you're using to
build cores).

### Build the JIC

The JIC (JTAG Indirect Configuration file) is generated automatically
when you compile splash. After running `./compile_splash.sh`:

```
retroDE_splash/output_files/retroDE_splash_hps.hps.jic
```

This file is pre-configured for the DE25-Nano's QSPI part
(`MT25QU128`, Active Serial x4 mode) by the splash project's
`post_flow.tcl` — no manual JIC settings needed.

### Flash the JIC via Quartus Programmer

1. Connect the DE25-Nano to your dev machine via USB (the JTAG cable
   is built into the board).
2. Launch **Quartus Programmer**:
   ```bash
   quartus_pgmw &
   ```
3. Click **Hardware Setup...** and select your DE25 (typically shows
   as **DE-SoC** or similar).
4. Click **Add File...** and select
   `retroDE_splash/output_files/retroDE_splash_hps.hps.jic`.
5. In the device list that appears, **check the box for "QSPI"** as
   the programming target (the JIC contains both FPGA and HPS-SPL
   sections targeting the QSPI flash).
6. Click **Start**. Programming takes a couple of minutes.
7. When complete, **power-cycle the board**. The new image now lives
   in QSPI and the board will boot it on every power-on.

After this, runtime core swaps via `retrodesd` only need the
`.core.rbf` — the HPS keeps running Linux from SD card and the FPGA
fabric reconfigures cleanly between cores.

## Quartus license

Quartus Prime Pro is a paid tool, but **the DE25-Nano includes a free
license**. To activate:

1. Register your board on Terasic's website with the serial number
   printed on the DE25.
2. Download the license file (`.dat`) from your Terasic account.
3. Point Quartus at the license file via `Tools → License Setup` or
   the `LM_LICENSE_FILE` environment variable.

Without a valid license, synthesis will fail at the start.

## Next steps

Once your environment is set up, see [BUILDING.md](BUILDING.md) for
how to compile cores and deploy them to the board.
