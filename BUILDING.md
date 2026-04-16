# retroDE — Building and Deploying Cores

How to compile a core from source and get it running on your DE25-Nano.

## Prerequisites

Make sure you've completed [SETUP.md](SETUP.md) first — Quartus
installed, license activated, repos cloned, board file layout in place.

## Build order

**Always build `retroDE_splash` first.** Every core depends on the
shared platform RTL in `retroDE_splash/rtl/platform/`. After splash,
build cores in any order.

## Compiling a core

Each core repo has a `compile_<core>.sh` script that runs the full
Quartus flow: synthesis → fitter → timing analysis → assembler → RBF
generation.

```bash
cd ~/FPGA_Projects/retroDE_splash
./compile_splash.sh

cd ~/FPGA_Projects/retroDE_nes
./compile_nes.sh
```

Build times vary widely — anywhere from **~6 minutes for smaller
cores (NES, Atari 2600, Game Boy) up to ~30 minutes for ao486** on
a modern machine. CPU core count, clock speed, and Quartus's mood
all factor in. Splash itself sits near the lower end since it has
no game logic.

### Output files

`output_files/` ends up with several artifacts. The only one you
deploy at runtime is the `.core.rbf`:

| File | Purpose |
|---|---|
| **`<project>.core.rbf`** | **FPGA fabric bitstream — this is what you scp to the board for runtime core swaps.** |
| `<project>.sof` | Quartus SRAM Object File — direct JTAG load for debugging |
| `<project>.hps.rbf` | HPS boot image with U-Boot SPL baked in. Not used in the normal workflow; only relevant if rebuilding the HPS-first boot path |
| `<project>_hps.sof` / `<project>_hps_auto.sof` | Intermediate SOFs with the HPS bootloader merged. Inputs to JIC/RBF generation; you don't deploy these |
| `<project>_hps.core.rbf` | Identical to `<project>.core.rbf` (different generation path, same bytes) |
| `<project>_hps.hps.jic` | **One-time QSPI flash image** — see [SETUP.md](SETUP.md#first-time-qspi-flash) for the initial board flash procedure |

> **Always deploy `<project>.core.rbf`.** The other files are
> intermediates or for the one-time QSPI flash described in SETUP.md.

### Clean build

To wipe all intermediate files and start fresh:

```bash
./compile_splash.sh clean
```

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `QUARTUS_ROOTDIR` | (auto-detect from PATH) | Path to your Quartus install |
| `UBOOT_HEX` | `software/u-boot/spl/u-boot-spl-dtb.hex` | U-Boot SPL hex for HPS RBF generation (splash only) |

## Building HPS software

The userspace tools (`retrodesd`, `coremgr`, etc.) are cross-compiled
on your development machine and copied to the board.

```bash
cd ~/FPGA_Projects/retroDE_splash/software

# Cross-compile retrodesd
aarch64-linux-gnu-gcc -O2 -static -o retrodesd retrodesd.c \
    osd_draw.c osd_input.c input_thread.c ds2_poll_thread.c \
    fabric_mgr.c save_helpers.c splash_backend.c \
    rom_simple_backend.c \
    ao486_backend.c ao486_ide_service.c ao486_fdd_service.c \
    ao486_cdrom_service.c ao486_cmos.c ao486_bios_load.c \
    ao486_osd.c ao486_keyboard.c \
    coco2_backend.c coco2_bios_load.c coco2_cart_load.c \
    coco2_keyboard.c coco2_osd.c \
    a2600_adapter.c nes_adapter.c gb_adapter.c \
    -lpthread
```

> This is a static build — no shared libraries needed on the board.
> The exact file list may change as cores are added. Check the repo
> for the current build command.

## Deploying to the board

> **First time on a new board?** You must flash the QSPI once
> before any of this works — see
> [SETUP.md → First-time QSPI flash](SETUP.md#first-time-qspi-flash).

Runtime deploys only ever copy the `.core.rbf` and (if rebuilt) the
`retrodesd` binary:

```bash
# Copy the core RBF
scp output_files/retroDE_splash.core.rbf terasic@<board-ip>:~/cores/

# Copy retrodesd (only if you rebuilt it)
scp software/retrodesd terasic@<board-ip>:~/software/

# Restart the supervisor to pick up a new retrodesd binary
ssh terasic@<board-ip> sudo systemctl restart retrodesd
```

For routine core updates, just scp the new `.core.rbf` — retrodesd
loads it on the next core switch via the OSD menu. No restart needed.

## Workflow summary

```
Dev machine                          DE25-Nano
──────────────                       ──────────
1. Edit RTL / software               
2. ./compile_<core>.sh               
3. scp .core.rbf ──────────────────→ ~/cores/
4. scp retrodesd (if changed) ─────→ ~/software/
                                     5. retrodesd loads core via OSD
                                     6. Play / test / debug
```

## Troubleshooting

### Synthesis fails immediately
Check your Quartus license. Agilex 5 requires Quartus Prime Pro;
the free license from Terasic must be activated. See SETUP.md.

### RBF generation skipped ("u-boot hex not found")
The splash build needs `software/u-boot/spl/u-boot-spl-dtb.hex` to
generate the HPS-merged outputs (JIC + HPS RBF). This file ships with
the splash repo — if it's missing, re-clone or restore it from git
history. Without it the `.core.rbf` still builds, but the JIC won't
generate.

### Core loads but the system hangs / no boot
The QSPI flash on the DE25 needs a compatible HPS-first image programmed
once. If you're seeing the FPGA configure but the HPS never reaches
Linux (no SD card activity, no console output), the QSPI image is
probably stale or missing. See SETUP.md → First-time QSPI flash.

This also applies after a Quartus version upgrade — Quartus changes
between versions can require reflashing the QSPI with a freshly-built
JIC from the current Quartus version.

### retrodesd won't start
- Needs `sudo` (or run via systemd as root) — requires `/dev/mem`
  access for the FPGA bridge.
- Check `~/cores/manifest.cfg` exists and lists at least one core.
- Check the splash `.core.rbf` exists at the path in the manifest.

### No video output
- Verify HDMI cable is connected before powering on.
- Check that the splash core RBF was loaded successfully
  (`dmesg | grep fpga` on the HPS).
- The ADV7513 HDMI transmitter needs a valid EDID handshake — some
  monitors are slow to respond. Power-cycle the monitor if needed.
