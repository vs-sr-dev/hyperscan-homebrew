# SPG290 SPU — first emulated HyperScan audio

This documents the SPG290 sound unit and the work that produced the **first known working HyperScan audio in an emulator**, published as the YouTube video *["First Ever HyperScan Emulated Audio — MAME Said It Was Impossible"](https://www.youtube.com/watch?v=1vh_27eTpfc)*. The emulated output was A/B-compared against a real-hardware capture and matches note-for-note.

The SPU has two distinct modes, both now implemented and verified in a MAME device patch:

1. a **DAC FIFO** (used by homebrew and most retail games), and
2. a **24-channel ADPCM hardware synth** (used by at least one retail title that bypasses the FIFO entirely).

SPU base: `0x08050000`.

---

## The key discovery: the DAC clock is 54 MHz, not 27 MHz

The PPCSDK documents a 27 MHz DAC clock. Real hardware uses **54 MHz** (crystal × 2). This was confirmed by comparing emulated audio against a real-hardware capture — at 27 MHz everything played an octave too low.

- Sample rate = `54 MHz / (P_DAC_SAMPLE_CLK + 1)`.
- `phase = 0x4C7` → `54 MHz / 1224 = 44117 Hz` (CD quality), not 22058 Hz.
- MAME fix: `SPG290_SPU(config, m_spu, 27_MHz_XTAL * 2)` in `spg29x.cpp`.

## DAC FIFO mode

Reverse-engineered from the PPCSDK `MP3Drv.c`:

- Double-buffered ping-pong FIFO: 4096 × 16-bit words (8 KB), 2-byte stride.
- IRQ every 2048-word boundary (half-buffer) on vector 63.
- Data is **unsigned** 16-bit PCM (XOR `0x8000` for signed conversion).
- Stereo interleaved (L, R, L, R).

**Resampling fix:** the naïve integer `rate_ratio` truncation caused audible stutter. Replacing it with a **16.16 fixed-point fractional accumulator** (`m_ring_frac`) gives zero drift. The stream stays at 44100 Hz and MAME handles the final resampling.

## 24-channel ADPCM synth mode

The hard case. *Interstellar Wrestling League* (IWL) uses **no DAC FIFO at all** — zero DMA starts in the log. Its entire audio runs on the hardware channels. This required implementing the synth path in `spg290_spu.cpp`.

Architecture as captured by structured register telemetry:

- 24 channels, each with its own state struct.
- Wave addresses are **22-bit WORD addresses** — `byte_addr = waddr * 2`. (Verified empirically: byte interpretation reads zeros at IWL's addresses; word interpretation hits real ADPCM data.)
- Phase accumulator is **19-bit**, converted to a 16.16 fixed-point step for the 44.1 kHz output stream.
- Decode via the standard `ima_adpcm_state` (`sound/imaadpcm.h`).
- Phase formula: `Phase = sample_rate * 2^19 / 281250` for the 54 MHz chip. E.g. `0xA07D = 41085 → ~22050 Hz`.

### What IWL actually does (and why it's instructive)

Telemetry across boot and an in-match capture (900+ channel-enable events) showed something architecturally surprising: **IWL uses only channel 0**. Music, commentary, SFX and crowd ambience are all software-mixed in CPU code and pumped as pre-mixed ADPCM chunks through a single hardware channel. The SPU is used purely as an **ADPCM decoder + DAC**, not as a multi-voice synth.

- Envelope mode is **manual** (`ChEnvMode = 0`): the CPU writes EDD directly, so no hardware ADSR state machine is needed.
- `MainVol = 0x7F`, fixed — the in-game master volume is *not* done via `MainVol`.
- The per-channel volume field `Pan_Vol[6:0]` is driven directly by the in-game SFX slider (proven by moving the slider and watching the value: `0x4000` at 0 → `0x404b` during play).
- Pan is always centre (`0x40xx`) — the game pans in software.

### The two correctness fixes that mattered

**1. Resume on re-trigger.** IWL's audio engine runs a per-video-frame service loop that, every 16.6 ms, re-writes the same wave address and re-asserts `ChEn = 1`. If `start_channel` does a fresh restart each time (the obvious implementation), you replay the same 16.6 ms of ADPCM every frame — the decoder ramps from `m_signal = 0, m_step = 0` each time, producing a **60 Hz sawtooth buzz**. (The diagnosis came from the user, a musician, identifying it by ear as "dirty monotone saw".)
The fix: on a `ChEn` rising edge, only do a fresh start if the wave address *differs* from the last-started one; otherwise preserve the read position and ADPCM decoder state. The game's per-frame `ChEn = 1` is a *resume*, not a *restart*. A `started_waddr` tracking field (sentinel `0xFFFFFFFF` after reset / natural end) implements this.

**2. No safety cap.** An initial 110000-sample auto-stop cut music at exactly 5 seconds. Removed — the channel terminates only on a `0xFFFF` end-code or a game-written `ChEn = 0`. No runaway occurs without the cap.

**Result:** A/B-verified against the real-hardware IWL recording — pitch, tempo, content all match.

### Known residual

Navigation tick SFX share channel 0 with the music (different wave address), so a tick does a correct fresh-start but then the music "resume" becomes a fresh-start too (the tick overwrote `started_waddr`). The proper fix is **per-wave-address state preservation** — a small `waddr → {adpcm_state, cur_waddr, sample_shift}` map — so each unique address resumes where it left off.

---

## For anyone reproducing this

- Channel 0 is the litmus test: if ch0 makes any noise, IWL SFX will play.
- Start with 8-bit PCM (skip ADPCM) to verify the pipeline end-to-end, then plug in the ADPCM decoder.
- IWL's ADPCM data starts at SDRAM `0x378C24` — easy to dump from the MAME debugger to validate the decoder independently.
- Other retail titles (Ben 10, Marvel, X-Men, Spider-Man) were not yet tested with the synth; IWL was the hardest case (no DAC FIFO at all), so they are likely fine.
- Remaining open question: video decode is slower than audio, so A/V desync in MAME is the video path's fault, not the audio's.

See [`../mame/notes.md`](../mame/notes.md) for the device-file pointers.
