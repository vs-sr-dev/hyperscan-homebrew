# hyperscan-homebrew

Technical documentation for homebrew development on the **Mattel HyperScan** (2006).

This repository exists because, to the best of our knowledge, **no other public documentation of this depth exists for this platform**. It is a preservation-research effort: the goal is not to ship a game, but to make the hard-won technical knowledge — compiler bugs, hardware register maps, undocumented driver behaviour — permanently and verifiably available to anyone who comes after.

---

## What is the HyperScan?

The HyperScan is a video game console released by Mattel in 2006. It was a commercial failure: discontinued within a year, sold off at a fraction of its launch price, and remembered today mostly as a curiosity. It is notable for one thing — it was an early mainstream console built around **RFID cards**: physical trading cards the player scanned to bring characters into the game.

Under the hood it is an unusual machine:

- **CPU:** Sunplus S+Core 7 (SPG290 SoC) running at ~108 MHz. The S+Core (sometimes "score7") is a little-known RISC architecture; its GCC backend (`score-elf`) is **unmaintained** and shipped with serious code-generation bugs.
- **Video:** integrated TVE (TV encoder) framebuffer + a separate PPU (picture processing unit) with text layers, sprites and bitmap modes.
- **Audio:** SPG290 SPU — a dual-mode DAC FIFO + 24-channel ADPCM hardware synth.
- **Input:** controllers over an I²C bus.
- **Storage:** **none**. There is no NVRAM and no flash. The only persistence on the entire system is the 92 writable bytes inside each RFID card.

Because the console flopped, there was never a public SDK, a homebrew scene, or a body of reverse-engineering notes. Almost everything in this repository was derived empirically, against the MAME driver for the platform, with cross-validation against real-hardware captures where possible.

---

## Why this documentation exists

Anyone who, in the future, tries to:

- write homebrew for the HyperScan,
- work on the SPG290 / score7 device in MAME,
- or use the `score-elf` GCC toolchain for any other S+Core 7 target,

…will hit the same walls we did. The `score-elf` backend in particular generates broken code in ways that are extremely hard to diagnose if you don't already know they exist. This repository documents **16+ reproducible compiler bugs** with root-cause analysis and verified workarounds, none of which appear in the GCC bug tracker at this level of detail.

The intent is concrete and specific: the CFLAGS fix `-ffixed-r3 -fno-jump-tables -fno-merge-constants` is a directly applicable contribution to *anyone* working on this CPU. Everything else here is in the same spirit.

---

## What's in this repository

| Path | Contents |
|---|---|
| [`gcc-score-elf/bugs-and-workarounds.md`](gcc-score-elf/bugs-and-workarounds.md) | 16+ reproducible bugs in the unmaintained `score-elf` GCC backend, each with root cause and a verified workaround. The single most reusable document here. |
| [`hardware/ppu-registers.md`](hardware/ppu-registers.md) | SPG290 PPU register map: bitmap layers, NPTR per-scanline scroll, hardware alpha blend, plus quirks not documented elsewhere (TVE override, NPTR vs POSX/POSY). |
| [`hardware/rfid-nfc-driver.md`](hardware/rfid-nfc-driver.md) | A from-scratch NFC bit-banging driver for the RFID card reader, implemented via raw GPIO timing analysis with no SDK available. Full 120-byte card read, verified end-to-end. |
| [`hardware/i2c-canonical-driver.md`](hardware/i2c-canonical-driver.md) | The controller-input I²C driver, and a root-cause investigation of an apparent timing-fragility bug that turned out to be a non-canonical driver pattern. Direct references to the MAME source. |
| [`audio/spu-mame-audio.md`](audio/spu-mame-audio.md) | The SPG290 SPU: DAC FIFO + the 24-channel ADPCM synth. This is the work behind the first-ever emulated HyperScan audio in MAME (see below). |
| [`mame/notes.md`](mame/notes.md) | Pointers into the MAME source tree (`spg290_i2c.cpp`, `score.cpp`, the SPU device) for anyone cross-referencing emulator behaviour. |

---

## Verifiable artifacts

- **First emulated HyperScan audio.** The SPU work documented in [`audio/spu-mame-audio.md`](audio/spu-mame-audio.md) produced the first known working audio for this console in an emulator, published as the YouTube video *["First Ever HyperScan Emulated Audio — MAME Said It Was Impossible"](https://www.youtube.com/watch?v=1vh_27eTpfc)*. The emulated output was A/B-compared against a real-hardware capture (Interstellar Wrestling League) — pitch, tempo and content match note-for-note.

---

## Part of the Silicon Relics project

This is one of three documentation repositories produced as part of **Silicon Relics**, a long-running cross-console preservation and homebrew effort focused on obscure and failed vintage platforms:

- **[hyperscan-homebrew](https://github.com/vs-sr-dev/hyperscan-homebrew)** — Mattel HyperScan / S+Core 7 (this repo)
- **[ngpc-homebrew-notes](https://github.com/vs-sr-dev/ngpc-homebrew-notes)** — SNK Neo Geo Pocket Color / TLCS-900H, incl. the first NGPC↔Dreamcast link-cable bridge
- **[pippin-homebrew](https://github.com/vs-sr-dev/pippin-homebrew)** — Bandai Pippin @WORLD / DingusPPC networking

GitHub profile: **[@vs-sr-dev](https://github.com/vs-sr-dev)**

---

## Attribution

This research is a human + LLM collaboration. Stating that plainly is the honest thing to do — and it is also the point, because the interesting result here is what the collaboration itself makes possible.

**This work would not exist without Claude as a technical bridge. The human brings domain knowledge and direction; Claude provides implementation across architectures the human could not otherwise approach.** Neither side reaches these results alone. A domain expert does not, by hand, write a bit-banged NFC driver from scratch, root-cause sixteen GCC backend bugs from disassembly, or implement a 24-channel ADPCM synth in a MAME device. And a language model, on its own, does not know the HyperScan exists, that its emulated audio was considered impossible, or which findings are significant enough to be worth the weeks of digging.

The split, concretely:

- **Human:** decades of encyclopaedic knowledge of the retro-hardware ecosystem; the judgement to recognise when a result is a genuine world-first *because* the field is known; project direction — what to look for, why it matters, where it's worth digging; hardware acquisition and real-hardware captures; empirical testing on MAME and hardware; and the ground-truth calls ("is this waveform right? does this look correct?").
- **Claude (Anthropic) — Opus 4.x family, via Claude Code:** translating that direction into implementation — the homebrew sources, the NFC and I²C drivers, the MAME device patches, the disassembly analysis behind the compiler-bug investigations, and the writing of this documentation. (The work spanned several Claude Code sessions in 2026; the published HyperScan-audio video credits Claude Opus 4.7 specifically.)

Findings were verified empirically; nothing here is asserted from the model's prior knowledge alone.

---

## License

MIT — see [LICENSE](LICENSE). The documentation is original work and may be reused freely with attribution.
