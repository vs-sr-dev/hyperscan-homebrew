# I²C controller-input driver — and a timing-fragility post-mortem

Controller input on the HyperScan goes over an I²C bus. This document covers the working driver and, more usefully, the root-cause investigation of an apparent "timing fragility" bug that cost half a debugging session — because the lesson generalises: the bug was neither in MAME nor in GCC, but in using a non-canonical driver pattern.

---

## Use the canonical driver

For all controller input, read all five registers of both controllers per call (10 I²C transactions total). The PPCSDK's `HS_Controller_Read` does exactly this, and its source comment explains why:

> *"responses seem to hang if you don't actually read all of the I2C data ports"*

This is a genuine hardware quirk. Reading the full register set per call is not wasteful — it is required for the controllers to respond reliably.

- **I²C base:** `0x08130000` (cached mirror). MAME does not map the `0x88130000` uncached mirror; on real hardware either works.
- **Ready bit:** `MODE_CTRL & 0x08` (ACK8) — **not** `INT_STATUS & 0x01`. The SDK's `INT_STATUS` poll requires IRQ-enable, which MAME doesn't auto-set; the ACK8 bit is the correct one to poll.
- **Init:** set `I2C_SLAVE_ADDR = 0x54`, configure `TFT_GPIO_OUTPUTEN`/`PULLUP`, do a flush read. Call once after `tve_init()`.

## Button / axis mapping

| Register | Field | Bits |
|---|---|---|
| reg 0 (Buttons A) | Blue / Start / Select / LShoulder / RShoulder | bit 0 / 1 / 2 / 3 / 4 |
| reg 1 (Buttons B) | Green / Red / Yellow | bit 7 / 6 / 5 |
| reg 2 (Analog Y) | down < 0x40, centre = 0x7F, up > 0xC0 | (PORT_REVERSE in MAME) |

## The post-mortem: timing fragility was a symptom, not the disease

Early homebrew used a short-circuit polling macro (`I2C_POLL_Y` etc.) that read 1–2 bytes per call with a sentinel like `btn0 == 0xFF`. This depended on MAME's I²C scheduler advancing far enough *during* the poll, which in turn depended on the surrounding memory-access pattern GCC happened to emit. It worked by luck — until adding `volatile` to some unrelated cube arrays (a fix for a different bug) shifted codegen enough that the timing collapsed.

Half a session went into chasing workarounds: invisible `fb_putchar` timing pads, interleaving reads, `volatile` cascades. **None were stable, because all of them were fighting the wrong problem.**

The actual fix was to switch to the canonical 10-transaction driver. As a side effect, those 10 transactions give MAME ~60 µs of virtual time per call — plenty for its scheduler to fire the I²C completion timer regardless of how GCC lays out the caller. The fragility vanished.

**The generalisable lesson:** when emulated-timing behaviour seems to depend on unrelated code changes, suspect that you're relying on an accidental timing property of a non-canonical access pattern. The fix is usually to do the *full, documented* hardware handshake, not to add pads around a shortcut.

### Obsolete workarounds — do not re-apply

Invisible `fb_putchar` pads, doubling pad counts, `volatile` on `g_fg`/`g_bg`, interleaving I²C reads between polls, kludges to skip the redraw. These were all symptom-chasing and are superseded by the canonical driver.

---

## MAME reference

MAME's SPG290 I²C device is `src/devices/machine/spg290_i2c.cpp` — `i2c_update()` is ~30 lines and is the authoritative answer for what any register does in the emulator. See [`../mame/notes.md`](../mame/notes.md).
