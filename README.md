# retroDE
Old school games, new school silicon.  
An open-source FPGA retro platform for the Terasic DE25-Nano (Intel Agilex 5).

---

## What is retroDE?

retroDE is an open-source FPGA retro gaming platform built for the **Terasic DE25-Nano (Intel Agilex 5)**. 
Multiple classic systems run under a unified framework as dedicated FPGA hardware cores — not software 
emulation on a general-purpose CPU.

Real hardware logic, rebuilt in modern silicon.

## Supported Hardware

| Board | FPGA | Status |
|---|---|---|
| Terasic DE25-Nano | Intel Agilex 5 | ✅ Supported |

## Cores

| Core | System | Status |
|---|---|---|
| NES | Nintendo Entertainment System | ✅ Working |
| Atari 2600 | Atari 2600 | ✅ Working |
| CoCo2 | TRS-80 Color Computer 2 | ✅ Working |
| ao486 | x86 PC (486-era) | ✅ Working |

> All cores are in active development. Working does not mean finished.

## Project Status

**retroDE is running on real hardware today.**

Cores are working. The framework, documentation, and release structure are being cleaned up and 
staged for first public release. The project is moving fast.

## Getting Started

1. **[SETUP.md](SETUP.md)** — one-time environment setup: hardware,
   Quartus install, board prep, repo layout, SD card file structure.
2. **[BUILDING.md](BUILDING.md)** — compile cores from source and
   deploy to the board.

`retroDE_splash` is the platform chassis — clone and build it first.
Every core depends on its shared RTL.

## License

GPL v3 — see [LICENSE](LICENSE) for details.

## BIOS Notes

The CoCo2 core ships with a clean-room BIOS — original work, not derived from any Tandy or 
Microsoft ROM. No external ROM sourcing required. It just works.
