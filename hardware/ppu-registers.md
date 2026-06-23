# SPG290 PPU — register map and scrolling model

The HyperScan's SPG290 SoC has two independent display paths: the **TVE** (TV encoder framebuffer) and the **PPU** (picture processing unit). This document covers the PPU bitmap mode as derived empirically while building a working dual-layer parallax demo, including several behaviours that are not documented in the PPCSDK and not implemented in MAME.

The findings here were validated on a working PPU stress test: two independent bitmap layers with hardware alpha blending and smooth parallax scrolling at different speeds.

---

## Display path basics

- **PPU base:** `0x08010000`. Three text layers + sprites + bitmap modes.
- **TVE must be disabled for the PPU to be visible.** Set `TVE_BUF_CTRL = 3`. The TVE draws *on top of* the PPU, so leaving it enabled hides the PPU output entirely. This is the first thing to get right.

## Layer control

- **PPU enable:** control bit 12 (`0x1000`).
- **Per-layer enable:** control bit 3 (`0x08`).
- **Per-layer blend enable:** control bit 8 (`0x100`); blend amount in a separate register, range 0–63.
- **Pixel formats:**
  - **RGB565** — opaque bitmap.
  - **ARGB1555** — per-pixel transparency (bit 15 = 1 → transparent). Useful for a text/overlay layer on top of an opaque background layer.

## Scrolling — use NPTR, not POSX/POSY

This is the key non-obvious finding:

- **POSX/POSY only offset the layer's position on screen; they do not scroll the layer's content.** Discovered during implementation — using them for scrolling does not do what you expect.
- **True scrolling is done via the NPTR table.** NPTR is a **per-scanline offset table** in PPU RAM. For bitmap mode, the pixel fetched is at:

  ```
  buf_start + NPTR[line] * 2 + x * 2
  ```

  Manipulating NPTR per line gives true horizontal *and* vertical scroll, and (in principle) raster effects.

- **H-offset table exists in PPU RAM but is NOT implemented in MAME.** No raster/wave effects are possible in the emulator — they would require either a MAME patch or real hardware.

## Working parallax demo — implementation notes

- Buffers were **1280×480** (double width) so the content tiles seamlessly.
- Scroll wrap at **512 px** (= 2× triangle-wave period = 16× stripe period) to make the wrap invisible.
- Tiling rule: the wrap point must be an exact multiple of the pattern period or you get a visible seam.
- During animation, **zero CPU rendering** is needed — only the NPTR table is updated per frame. The blend and scroll are entirely in hardware.
- **Frame pacing:** use a *non-volatile* skip counter (`ppu_skip & 0x1FF`). Volatile delay loops inside goto blocks trigger a `score-elf` codegen bug (see [`../gcc-score-elf/bugs-and-workarounds.md`](../gcc-score-elf/bugs-and-workarounds.md), Bug #2/#6).

## Video-mod feasibility (hardware note)

For anyone considering a video output mod:

- The SPG290 TVE is fully integrated — there are **no external RGB pins**, so a clean analog RGB tap is not feasible.
- The LCD interface (pins 129–148, `TFT_GPIO0–19`) outputs **16-bit digital RGB** and is mostly unused on the HyperScan PCB. A digital capture there (e.g. an FPGA snooping the LCD bus → HDMI) is the realistic mod path.

---

## Quick register reference

| Function | Address / bit |
|---|---|
| PPU base | `0x08010000` |
| PPU enable | control bit 12 (`0x1000`) |
| Layer enable | control bit 3 (`0x08`) |
| Layer blend enable | control bit 8 (`0x100`) |
| Blend amount | 0–63 |
| TVE disable (to show PPU) | `TVE_BUF_CTRL = 3` |
| Per-scanline scroll | NPTR table in PPU RAM |
| Pixel fetch (bitmap) | `buf_start + NPTR[line]*2 + x*2` |

For TVE framebuffer addresses and the rest of the memory map, see [`../mame/notes.md`](../mame/notes.md).
