# hyperscan-homebrew (archived)

> **About the "compiler bugs":** they were found with a GCC 14.3.0 that had the long-removed S+core backend patched back in out of tree (the toolchain script from [LiraNuna/hyperscan-emulator](https://github.com/LiraNuna/hyperscan-emulator), as of April 2026). They describe that patch at that time, not the S+Core CPU and not any maintained toolchain, and that patch has received fixes since. If someone sent you here about S+Core compiler bugs, this is not the place to look.

> **Obsolete, and archived.** This was one of the first repositories on this account, published in June 2026. It stays online, read-only, as a record of where the work started. It is not a reference for HyperScan or S+Core development: please don't cite it as one.

## Why it is obsolete

- **The compiler notes describe one toolchain, not the CPU.** They were written while building with an unmaintained `score-elf` GCC and no SDK. Once a real SDK was in use, those problems stopped mattering. The workarounds are specific to that setup and should not be read as a list of S+Core pitfalls.
- **The "to our knowledge" claims are dated.** They described what we had found at the time, in particular the state of HyperScan audio in emulators in June 2026. They have not been kept up to date since.
- **The newer work does not depend on it.** The projects below were built with an SDK and against MAME, and this repository was never updated to match.

## Where the HyperScan work is now

- **[flick](https://github.com/vs-sr-dev/flick)**: an emulator for the Mattel HyperScan. It plays the retail games with sound, RFID cards and two players, and every step is checked against MAME's instruction traces.
- **[hs-lba](https://github.com/vs-sr-dev/hs-lba)**: *Little Big Adventure* ported to the HyperScan.

For in-depth knowledge of the hardware and S+Core development tools from the console's homebrew scene, see **[scoresdk](https://github.com/scoresdk)**.

## What is left here

The files are unchanged from the original publication: `gcc-score-elf/`, `hardware/`, `audio/` and `mame/`. The original README is kept as [ORIGINAL-README.md](ORIGINAL-README.md).

Thanks to the people from the HyperScan homebrew scene who pointed out that this repository had outlived its use.

## License

MIT, see [LICENSE](LICENSE).
