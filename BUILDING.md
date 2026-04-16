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

The build takes roughly 6–7 minutes per core on a modern machine.
Output lands in `output_files/`:

```
output_files/retroDE_splash.core.rbf    ← FPGA fabric bitstream
output_files/retroDE_splash.hps.rbf     ← HPS boot image (splash only)
```

> **Splash is special:** it produces both a `.core.rbf` (FPGA fabric)
> and an `.hps.rbf` (HPS boot image with U-Boot SPL baked in). Other
> cores only produce a `.core.rbf`.

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

Copy the compiled RBF and software to the DE25-Nano over the network:

```bash
# Copy the core RBF
scp output_files/retroDE_splash.core.rbf terasic@<board-ip>:~/cores/

# Copy the HPS RBF (splash only, first-time setup)
scp output_files/retroDE_splash.hps.rbf terasic@<board-ip>:~/cores/

# Copy retrodesd (after cross-compiling)
scp software/retrodesd terasic@<board-ip>:~/software/

# Restart the supervisor to pick up the new binary
ssh terasic@<board-ip> sudo systemctl restart retrodesd
```

For subsequent core updates, just scp the new `.core.rbf` — retrodesd
picks it up on the next core switch via the OSD menu.

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
generate the HPS RBF. This file ships with the repo — if it's missing,
re-clone or restore it from git history.

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
