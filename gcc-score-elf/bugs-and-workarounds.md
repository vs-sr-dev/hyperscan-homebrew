# GCC `score-elf` — critical code-generation bugs and workarounds

The HyperScan CPU is a Sunplus **S+Core 7** ("score7"). The only freely available C toolchain for it is GCC's `score-elf` backend. That backend is **unmaintained** and emits incorrect code in a number of situations. The bugs below were isolated empirically while developing homebrew against the MAME driver; each affects real, ordinary C code, and several manifest as symptoms so far removed from their cause that they are nearly impossible to diagnose without prior knowledge.

None of these are reported in the GCC bug tracker at this level of detail. They are documented here so that anyone using `score-elf` for any S+Core 7 target — not just the HyperScan — can recognise the symptom and apply the fix.

> **The single most important takeaway:** keep `-ffixed-r3 -fno-jump-tables -fno-merge-constants` in your CFLAGS. These three flags fix the worst three bugs (#14, #9, #10) outright. Everything else below is detail and history.

The toolchain used for all of this was binutils 2.35.2 + GCC 14.3.0 + newlib, built under WSL.

---

## The root cause behind many symptoms: Bug #14 (r3 / link register)

**Bug #14 — GCC allocates r3 (the link register) as a general-purpose scratch register.**

On S+Core 7, `r3` is the link register: `jl` (jump-and-link) writes it, `br r3` returns through it. `score-elf` does **not** reserve r3 as fixed. In any function with enough register pressure, GCC may emit something like `lw r3, [r7]` to use r3 as a temporary. If the function later executes a `jl` (which clobbers r3) or returns via `br r3`, the value is wrong — the function returns to garbage or silently loses state.

- **Smoking gun:** a `hs_puts` leaf with a `font8x8` loop emitted `lw r3, [r7]` to load a font row into r3, then `br r3` at the end returned to the *font byte value* instead of to the caller. Observed as "only the first call works" when `hs_puts` was called in a loop.
- **Workaround (permanent):** add **`-ffixed-r3`** to CFLAGS. This marks r3 reserved so GCC never allocates it as a GPR. Cost: one fewer GPR — negligible on score7. Empirically confirmed to fix the `3D STRESS TEST` → `?D STRESS TEST` glitch.

`-ffixed-r3` very likely subsumes several of the older bugs below (they were probably symptoms of the same root cause), but the original workarounds are cheap and have been left in place for stability rather than removed reactively:

- **#1** (loops with `jl` crash) — plausibly r3 held a live loop value, clobbered each `jl`.
- **#4** (r3 not saved in functions with `jl`) — literally the same bug, different framing.
- **#7** (division via `__divsi3` crashes in goto loops) — `__divsi3` is called via `jl` → clobbers r3.
- **#12** (const-prop through `draw_line` clobbers registers) — plausibly r3 held a propagated constant.
- **#13** (volatile cascade) — volatile shifts register allocation, pushing/pulling r3 into/out of live roles.

Bugs **not** related to r3 (different root cause, still need their own workarounds): #3, #5, #6, #9, #10, #11.

---

## The flag-fixable bugs

**Bug #9 — jump tables on `if/else` chains with 5+ cases silently break.**
At `-Os`, GCC turns a long `if (sel==0)… if (sel==4)…` chain into an indirect-branch jump table. The table base is computed conditionally on one path; the fall-through path reads an uninitialised `[sp+32]` slot and does `br r6` into garbage. Observed: adding a 5th menu entry silently broke selection for *every* menu item.
**Workaround:** `-fno-jump-tables`. Permanent.

**Bug #10 — string-literal CSE/merging across a ternary-in-macro misaligns rodata pointers *and* corrupts nearby `.bss`.**
At `-Os`, GCC does cross-item common-subexpression elimination on patterns like `fb_puts(x, y, (sel==N) ? ">…" : "  …")` when the macro internally reads more characters than the string length. It passes *wrong* base addresses to `romchar` — e.g. loading `"> SIGNAL TEST" + 4` instead of `"> RFID VERIFY"` — so every menu item ends up shifted/garbled.
The dangerous part is the second-order effect: the mis-addressed reads and writes can stomp adjacent `.bss` variables, so the bug can also surface as a **phantom input/state bug** in unrelated code. Rule of thumb: an input/state bug that mysteriously clears after visiting a different menu is very likely ternary-CSE trashing memory in the redraw path.
**Workaround:** `-fno-merge-constants`, *plus* use explicit `if/else` blocks with separate `fb_puts` calls instead of ternary string selection — applied to **every** redraw path, not just visible ones (the bug is latent and activates when a new redraw path is added). Pad related strings to uniform length so an over-reading macro doesn't walk into a neighbouring literal.

---

## The structural bugs (need code-shape workarounds)

**Bug #1 — loops containing function calls (`jl`) crash after 1–2 iterations.**
Any `for`/`while` loop with a `jl` inside produces corrupted results or crashes. *Workaround:* use macros instead of functions, or goto-based state machines. (Likely subsumed by `-ffixed-r3`.)

**Bug #2 — `volatile` loop variables inside goto blocks generate broken code.**
`for (volatile int i = 0; i < N; i++)` inside a goto-label block crashes. *Workaround:* unrolled macro sequences for delays/polling.

**Bug #3 — the 5th argument (passed on the stack) is read at the wrong offset.**
Functions with 5+ arguments read the 5th at the wrong stack offset. Confirmed on `fb_line(x0,y0,x1,y1,color)` where `color` was always white. *Workaround:* max 4 args per function; use globals for extra params.

**Bug #5 — `.sbss` relocation truncated (`R_SCORE_GP15`).**
*Workaround:* `__attribute__((section(".bss")))` on affected globals.

**Bug #6 — XOR-3 address compensation mis-folded when inlined.**
*Workaround:* `__attribute__((noinline, noclone))` on `romchar()`.

**Bug #7 — division emits a `jl` to `__divsi3`, which crashes inside goto loops.**
Any integer division by a non-power-of-2 becomes a library call. *Workaround:* use shifts (`>> N`) or avoid division. (Likely subsumed by `-ffixed-r3`.)

**Bug #8 — inline functions with internal goto labels break inside outer goto blocks.**
Label collision / broken control flow when an inline function uses `goto draw_step/draw_done` and is called from within a goto-based state machine. *Workaround:* macros instead of inline functions for code called from goto blocks.

**Bug #11 — small for-loop copies silently skip an iteration.**
A plain `for (i = 0; i < 8; i++) { arr[i] = src[i]; arr2[i] = src2[i]; }` can write 7 of 8 elements. Observed: the 3D cube copy loop left `cube_sx[4]`/`cube_sy[4]` at their `.bss` zero default, pinning a cube vertex to (0,0). *Workaround:* manually unroll small copy loops with 2+ stores per iteration.

**Bug #12 — const-propagation through `draw_line` clobbers registers with wrong values.**
With `static const int32_t cube_sx[8]` in rodata, GCC const-propagates the values into caller registers and passes literals to back-to-back `draw_line(...)` calls — but makes register-assignment mistakes across the sequence, so some edges get wrong constants (phantom edges shooting off-screen). *Workaround:* `volatile` on the arrays forces real memory loads per call site. (Likely subsumed by `-ffixed-r3`.)

**Bug #13 — the volatile cascade.**
Once one variable needs `volatile`, related state tends to need it too. Adding `volatile` to `cube_sx/sy` broke previously-working I²C timing; the fix required making `btn0/btn1/analog_y/g_fg/g_bg` volatile as well. *Lesson:* be conservative introducing `volatile` — each addition re-lays-out codegen and can break something downstream.

**Compilation explosion — deeply nested macros with `break`/control flow.**
A `FB_LINE` macro with 512 unrolled steps + `break` inside `do{}while(0)` caused 10–20 minute compile times. *Workaround:* remove `break`, use a conditional step instead.

---

## The two subtle late bugs (#15, #16)

These two were isolated by step-wise reproduction and are the hardest to recognise in the wild.

**Bug #15 — a non-leaf function that saves r3 on the stack and makes *any* internal `jl` silently hangs or mis-returns.**

The minimal trigger is:

```asm
funcA:
    sw   r3, [r0, -4]+      ; save r3 (pre-decrement auto-update)
    addi r0, -4             ; reserve extra frame
    jl   funcB              ; ANY internal call
    ...                     ; body
    addi r0, 4
    lw   r3, [r0]+, 4       ; restore r3 (post-increment auto-update)
    j    funcC              ; or br! r3
```

After return, the caller either never resumes (a visual "hang") or resumes with broken state. The `sw r3, [r0, -4]+` + `addi r0, -4` + `jl` combination alone is sufficient — no r12/r13/r14 prologue saves are required to trigger it.

- **Functions that work:** pure leaves with no `sw r3`; tail-call-only wrappers (`fn(){ jl other(); }` compiles to `ldi r4, arg; j other`, no r3 save); a wrapper whose body is only an inline delay loop with no `jl`.
- **Functions that hang:** any small wrapper with a couple of saves and one `jl`; a byte-read routine decomposed into helper calls where any helper has an internal `jl`.
- **Workaround (proven):** make the function a **true leaf**. Inline every delay loop, inline every helper, expand bit-by-bit checks as explicit `if/else`. One function, many loops, zero internal `jl`. GCC then omits the `sw r3` prologue entirely. 150+ inline pulses in a single function works cleanly (~3.5 KB of code for the NFC byte-read routine).
- **Display helpers:** keep them as `#define` macros, not wrapper functions.
- **Do not** try to fix the wrapper shape — every variant tried (remove tail-call, remove nibble-helper, ternary→if/else, remove r12/r13 saves) still triggers the bug if the function has *any* internal `jl` with a saved r3 on the stack. "Pure leaf" is the only reliable shape.

**Bug #16 — data-flow value tracking lies across goto targets and across loop bodies under register pressure.**

`x = K;` for a small constant `K` gets compiled to `sw r<stale>, [x]`, reusing whatever GPR GCC's data-flow analysis *thinks* holds `K`. The tracked value came from one predecessor of the goto target (or from before a high-pressure loop body), but at runtime that path held something else. Likewise `x ^= 1` becomes `xor r, r, r<stale>` where `r<stale>` is supposedly 1 but was clobbered by loop-body temporaries.

Observed cascade in a double-buffer flip: `TVE_BUF_CTRL = 0;` at a goto target compiled to `sw r12, [BUF_CTRL]` with r12 = 1 (left over from a `_done = 1` assignment in the previous `fb_puts`), writing 1 instead of 0; then `phase_c_draw_idx ^= 1;` in the loop body XOR-ed against a register clobbered by 48 `PUT1` macro expansions, so `BUF_CTRL` took invalid values and the TVE read an unset framebuffer → black screen with `.text` noise in the bottom scanlines.

- **Workaround (proven):** force an explicit `ldi` via inline asm **at each use site** — not once at the goto entry, but re-materialised on every path/iteration that consumes the constant:
  ```c
  uint32_t _one;
  asm volatile ("ldi %0, 1" : "=r"(_one));
  x ^= _one;
  ```
- **Key lesson:** local inline-asm is *not* enough. The constant must be re-materialised at the exact point it is **used**, inside the same basic block / iteration as the consumer. A `_one` materialised at function entry and referenced 100 lines later through a loop body *will* be clobbered.
- **How to detect:** if a volatile device register or state variable takes unexpected values, dump the store site. `sw r<N>, [addr]` with no nearby `ldi r<N>, K` is the smoking gun; grep backwards for the most recent `ldi r<N>` — if it's from a distant/unrelated context, it's Bug #16.

---

## Emulator timing note (MAME)

`src/devices/cpu/score/score.cpp` does `m_icount -= 6;` per instruction with a `// FIXME: if available use correct cycles per instructions`. Every instruction costs exactly 6 icount ticks regardless of opcode. This flat cost is the hidden reason cycle-counted software timing loops never matched naïve calculations: a 13-instruction inline loop iteration is 13 × 6 = 78 ticks ≈ 722 ns at 108 MHz.

The NFC delay constants that work in MAME (`DELAY_BIT0_N=25`, `DELAY_BIT1_N=75`, `DELAY_SYNC_N=300`) sit in the middle of MAME's NFC timing windows. **On real hardware** the CPU has true per-opcode timing, so these delays will likely need rescaling — expect to bisect the three constants against a physical unit.

---

## Applying all of this to new code

- Keep **`-ffixed-r3 -fno-jump-tables -fno-merge-constants`** in CFLAGS always (#14, #9, #10).
- Still use: explicit `if/else` for string selection (#10), manual unroll for small copy loops (#11), `__attribute__((noinline, noclone))` on `romchar` (#6), max 4 args per function (#3), `__attribute__((section(".bss")))` on affected globals (#5), and **prefer a macro over a small wrapper that makes internal `jl` calls — especially for display helpers (#15)**.
- The older workarounds for #1/#4/#7/#12/#13 are likely subsumed by `-ffixed-r3` but are cheap to keep; don't remove them reactively. For new code, prefer the simpler pattern, test it on MAME, and if it works, keep it.
