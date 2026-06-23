# MAME reference notes

MAME is the working emulator for the HyperScan and was the reference platform for all of the documentation in this repository. The driver is `hyprscan`. This file collects pointers into the MAME source tree for anyone cross-referencing emulator behaviour, plus the build and launch invocations used.

These references are to the `mame-master` tree as of the work in 2026; line numbers may drift, but the file paths are stable.

---

## Key source files

| File | What it tells you |
|---|---|
| `src/mame/tvgames/spg29x.cpp` | The main HyperScan driver: full memory map, device wiring, the SPU clock (`27_MHz_XTAL * 2`). |
| `src/devices/cpu/score/score.cpp` | The S+Core 7 CPU core. Note `m_icount -= 6;` (~line 210) — a flat 6 ticks per instruction, with a `FIXME` that real per-opcode timing is unknown. This is the hidden constant behind software-timing-loop calibration. |
| `src/devices/machine/spg290_i2c.cpp` | The I²C controller device. `i2c_update()` is ~30 lines and is the authoritative reference for controller-input register behaviour. |
| `src/devices/sound/` (SPU device: `spg290_spu.{h,cpp}`) | The SPU: DAC FIFO + 24-channel ADPCM synth. The audio work in [`../audio/spu-mame-audio.md`](../audio/spu-mame-audio.md) lives here. |

## Memory map highlights

| Region | Address |
|---|---|
| TVE framebuffer address | set at `0x08070000` |
| TVE buffer select | `0x08090020` |
| TVE video mode | `0x08030000` |
| PPU | `0x08010000` |
| SPU | `0x08050000` |
| I²C controllers | `0x08130000` (cached mirror) |
| RFID / card GPIO | `0x08200024`, `0x08200068`, card-detect `0x08200070` |
| HYPER.EXE load address | `0xA00901FC`, entry `0xA0091000` |

The TVE framebuffer is RGB565; `TVE_CONTROL` bit 15 (TVEN) must be set for output.

## Build (native Windows, MSYS2/MinGW64)

```
make SUBTARGET=hyprscan SOURCES=src/mame/tvgames/spg29x.cpp -j$(nproc) NOWERROR=1
```

WSL build dependencies for a Linux ELF build: `libsdl2-dev libsdl2-ttf-dev libpulse-dev libfontconfig-dev qtbase5-dev`.

## Launch with a homebrew binary

```
hyprscan.exe hyprscan -rompath "<roms>;<build>" -quickload "<build>\HYPER.EXE"
```

The user inserts an RFID card image at runtime via *Tab → File Manager → card → select .bin*.

## Toolchain / byte-order caveat

The `score-elf` GCC toolchain (binutils 2.35.2 + GCC 14.3.0 + newlib, built under WSL) emits **big-endian** code; MAME reads little-endian. A byte-swap step is required: reverse-bytes=4, then fix the 16-bit instruction pairs by swapping the halfwords back. Without this, 16-bit S+Core instruction pairs execute in the wrong order. See [`../gcc-score-elf/bugs-and-workarounds.md`](../gcc-score-elf/bugs-and-workarounds.md) for the compiler-side caveats.
