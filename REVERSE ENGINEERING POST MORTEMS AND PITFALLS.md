---
title: "Digitakt II OS 1.17 — Reverse Engineering Post-Mortems & Pitfalls"
tags:
  - post-mortem
  - bugs
  - lessons-learned
  - sharc-visa
date: 2026-10-05
---

# Reverse Engineering Post-Mortems & Pitfalls

## 1. Post-Mortem 1: The 49.82 Hz Drone
* **Symptom**: Machine 2 (Cloudy) produced audio, but output a constant 50 Hz square/saw buzz (shifting up an octave to 96 Hz when speed doubled).
* **Root Cause**: `cloudy_window.s` set `r11 = 1` (`Arg 2 = Loop (1 = Continuous Drone)`) and called `0x1C4D88` using the stock 20.07 ms time-stretch slice bounds (`R6` and `R7`).
* **The Math**:
  $$\text{Frequency} = \frac{1}{20.07\,\text{ms}} = 49.82\,\text{Hz}$$
* **Fix**: Do not loop a static slice. Modulate `R6` and `R7` with stochastic PRNG spray, scale grain lengths dynamically, and respect the `PLAY` knob's loop/one-shot flag.

---

## 2. Post-Mortem 2: The OS Load Freeze (Soft Brick)
* **Symptom**: Device froze permanently during boot screen loading bar.
* **Root Cause**: An extra 2,016-byte LDR block was appended to `section_7_BLOB.bin`, expanding it to 323,048 bytes.
* **Hardware Failure Mechanism**: The ColdFire SPI DMA engine transfers **exactly 321,016 bytes** to the SHARC slave port. Because the file size changed:
  1. The SPI DMA controller desynced.
  2. The SHARC boot ROM never signaled completion.
  3. ColdFire froze waiting for the DSP handshake.
* **Fix**: Never append blocks to Section 7. Section 7 must be **exactly 321,016 bytes** bit-for-bit.

---

## 3. Post-Mortem 3: The Audio Engine / Sequencer Freeze
* **Symptom**: ColdFire UI booted cleanly and menus worked, but pressing Play on the sequencer did nothing, and all machines (including OneShot) were silent.
* **Root Cause**: Two locations outside the STRETCH slot were modified:
  1. **File `0x03C9E2` (`SW 0x1C6AF1`)**: We assumed this was an isolated call to `0x1C5576`. Disassembly proved it is a Type 7a instruction in the **shared 32-voice loop**. Overwriting it crashed the DSP on Voice 0.
  2. **File `0x03D16E` (`SW 0x1C6EB7`)**: Assumed to be dead dummy code. Disassembly proved it contains active wavetable synthesis routines.
* **Failure Mechanism**: When the SHARC crashed on Voice 0, it stopped sending periodic DSPI2 frame pacing interrupts (vector 191) to the ColdFire. The sequencer transport refused to advance without DSP timing ticks.
* **Fix**: **Never touch code outside the STRETCH slot.** The only voice-isolated safe zone in Section 7 is `SW 0x1C6782` to `SW 0x1C686E` (File `0x03C304` to `0x03C4DC`, 472 bytes).

---

## 4. Toolchain (`selas`) Quirks & Syntax Hazards

1. **The Stack Pointer Collapse Bug**:
   ```sharc
   i7 = modify (i7, 4);   // BUG: Compiles to "0f 17 00 00 00 00" (I7 = 0) -> Instant crash!
   ```
   * *Fix*: Avoid stack frames entirely, or use register transfers:
     ```sharc
     r1 = 4; r2 = i7; r2 = r2 + r1; i7 = r2;
     ```
2. **DAG Memory Syntax**:
   ```sharc
   r1 = dm(0x41, r4);     // BUG: "expected I-register in memory operands: 0X41, R4"
   ```
   * *Fix*: Memory accesses strictly require DAG index registers (`I0`..`I15`):
     ```sharc
     i4 = r4;
     r1 = dm(0x41, i4);    // VALID
     ```
3. **Register-Only Comparisons**:
   ```sharc
   comp(r0, 0);           // BUG: "unknown register: 0"
   ```
   * *Fix*: SHARC ALU comparisons require two data registers:
     ```sharc
     r3 = 0; comp(r0, r3); // VALID
     ```
4. **Register-Only Bit Shifts**:
   ```sharc
   r2 = lshift r1 by 13;  // BUG: "unknown register: 13" in certain selas modes
   ```
   * *Fix*: Put shift count into a register:
     ```sharc
     r3 = 13; r2 = lshift r1 by r3; // VALID
     ```
```

---

