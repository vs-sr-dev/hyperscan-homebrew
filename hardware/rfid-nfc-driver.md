# RFID / NFC card driver — bit-banged from scratch

The HyperScan's defining feature is its RFID cards. The console ships no SDK and no documentation for the card reader, so this driver was implemented **from scratch via GPIO timing analysis**: figuring out the NFC pulse protocol by observation, then bit-banging it directly. The result reads arbitrary bytes from the 120-byte card memory and is verified end-to-end against real card dumps.

This matters beyond the HyperScan because it is a worked example of bringing up an undocumented NFC peripheral with nothing but a GPIO pin and a timer.

---

## Card layout (120 bytes)

| Bytes | Field | Access |
|---|---|---|
| 0–6 | RFID UID | read-only |
| 8–9 | Game ID | read |
| 10–11 | Card ID | read (used to differentiate cards) |
| 12–103 | Save state (92 bytes) | read/write (WRITE-E command) |

The Card ID (bytes 10–11) is the natural key for content differentiation: card A → content A, card B → content B. The 92-byte save area is the **only persistence mechanism on the entire console** — there is no NVRAM and no flash. If you want anything to survive a power cycle, it lives in those 92 bytes.

## Two levels of access

There are two distinct ways to talk to a card, with very different complexity:

### 1. Presence detection (simple)

In MAME, card presence is exposed through a homebrew-extension register at GPIO `0x08200070`. The emulator returns 1 when a memcard image is loaded (the user inserts a card via MAME's *Tab → File Manager → card → select .bin*). This uses MAME's public `exists()` API (`is_loaded()` is protected). This is the easy path if all you need is "is a card present?".

### 2. Full read via NFC bit-banging (the real work)

Reading actual card data requires speaking the NFC protocol over GPIO. The driver:

1. Sends a REQA-style READ command — 9 bytes, `0x01` prefix.
2. Samples 8 response bits per byte via a `pulse_sync` followed by `GPIO_IN` reads.
3. Reassembles the bytes.

GPIO registers involved: `0x08200024`, `0x08200068`, and the card-detect mirror at `0x08200070`.

**Verified end-to-end:** reading `test_card.bin`, bytes 0–11 all match — UID `8C E3 0D EB EA B8 F1`, Game ID `CFB0`, Card ID `8F2D`. Different cards produce different display output and different narrative dispatch keyed on the Card ID.

## Timing constants

The NFC pulse widths are the crux. The protocol distinguishes bits and sync by pulse length:

| Symbol | Constant (MAME) | Approx. width | MAME window |
|---|---|---|---|
| bit 0 | `DELAY_BIT0_N = 25` | ~18 µs | < 36.8 µs |
| bit 1 | `DELAY_BIT1_N = 75` | ~54 µs | 36.8–73.8 µs |
| sync | `DELAY_SYNC_N = 300` | ~217 µs | 73.8–368 µs |

Each constant sits in the middle of its MAME timing window for maximum margin. These widths assume MAME's flat 6-icount-per-instruction model (see [`../gcc-score-elf/bugs-and-workarounds.md`](../gcc-score-elf/bugs-and-workarounds.md), timing note). **On real hardware**, with true per-opcode timing, expect to re-bisect all three.

## Critical implementation constraint — one giant leaf function

The driver is written as a **single, flat leaf function** with every delay loop and every helper inlined and every bit-check expanded as explicit `if/else`. This is not a style choice — it is forced by `score-elf` **Bug #15**: any function that saves the link register on the stack and then makes an internal `jl` call will silently hang or mis-return. A byte-read routine decomposed into the obvious helper-call chain hangs after the first call.

The only reliable shape is "pure leaf, no internal `jl`". The NFC byte-read routine is ~3.5 KB of machine code, 150+ inline pulses, all in one function. Display helpers around it are `#define` macros for the same reason. See the bug doc for the full reproduction and rationale.

---

## Design notes

- The cards work naturally as cross-console "messages in a bottle": optional enrichment carried on a physical token, not a save-game gate.
- Because the console is otherwise stateless, designs that assume passive persistence (companion characters, progress tracking) don't fit — the statelessness is a property of the hardware, not a limitation to work around.
