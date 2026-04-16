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
installed and how to launch them. Example:

```ini
# retroDE core manifest

[splash]
name=Splash (Main Menu)
rbf=/home/terasic/cores/retroDE_splash.core.rbf
backend=splash

[nes]
name=NES
rbf=/home/terasic/cores/retroDE_nes.core.rbf
backend=rom_simple
extensions=.nes
media=rom

[ao486]
name=ao486 (x86 PC)
rbf=/home/terasic/cores/retroDE_ao486.core.rbf
backend=ao486
config=/home/terasic/ao486.cfg
bios.0=bios:/home/terasic/bios/boot0.rom
bios.1=vgabios:/home/terasic/bios/boot1_vbe.rom
media=ide,floppy,cdrom

[coco2]
name=CoCo2
rbf=/home/terasic/cores/retroDE_coco2.core.rbf
backend=coco2
extensions=.rom,.ccc,.bin
media=rom
```

Add or remove sections as you install cores. The `backend` field tells
`retrodesd` which session backend handles the core's lifecycle.

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
