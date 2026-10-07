---
title: "Digitakt II OS 1.17 — Cloudy Granular Engine Specification"
tags:
  - cloudy
  - granular-synthesis
  - specification
  - dsp-algorithm
date: 2026-10-05
---

# Cloudy Granular Engine Specification

## 1. Engine Concept & UI Bindings

Inspired by the Mutable Instruments Clouds texture synthesizer, adapted to Digitakt II Track 2:

```
[ColdFire UI: Section 3]
  Track 2 Machine Title: "Cloudy"  (Offset 0x24288A: b"Cloudy\0\0")
  Track 2 Abbreviation : "CLDY"    (Offset 0x248DFB: b"CLDY\0")
  Parameter Display    : "RATE"    (Offsets 0x2273C8, 0x2273CD)
```

### Encoder Controls on the SRC Page:
* **START (Knob E)**: Grain Playhead Scrub. Moves the central sampling window across the loaded audio RAM buffer.
* **LEN (Knob F)**: Grain Duration / Size. Scaled dynamically from micro-grains (~25 ms) to sustained clouds (~300 ms).
* **RATE (Knob G)**: Pitch Step & Freeze.
  * Setting $\le 1$: **FREEZE Mode** ($F8 = 0.0\text{f}$). Playhead halts, capturing the current grain texture in place.
  * Settings $> 1$: Continuous variable playback speed and pitch transposition ($F8 = \text{RATE} \times 0.03125\text{f}$).
* **PLAY (Knob B)**: Mode and Direction (`DM(-8, I6)`):
  * **Forward One-Shot**: Spawns single stochastic grains on sequencer trigs.
  * **Continuous Loop Cloud**: Sustains an ambient, overlapping granular stream.
  * **Reverse Modes**: Inverts grain playback direction.

---

## 2. DSP Algorithmic Components

### A. Stochastic Position Spray (32-bit PRNG)
To eliminate metallic comb-filtering, every grain trigger adds stochastic scatter around the playhead position using a 4-instruction Linear Congruential Generator (LCG):
$$x_{n+1} = (x_n \times \text{0x41C64E6D} + 12345) \pmod{2^{32}}$$
$$\text{spray} = x_{n+1} \ \& \ \text{0x07FF} \quad (0 \dots 2047\,\text{samples} \approx 0 \dots 21.3\,\text{ms})$$
$$\text{Grain Start} = \text{Playhead} + \text{spray}$$

### B. Dynamic Grain Boundary Scaling
Instead of a fixed 20 ms slice, grain end boundaries are dynamically calculated:
$$\text{Grain End} = \text{Grain Start} + (\text{RATE} \ll 8)$$

### C. 6-Tap Polyphase Sinc Interpolator (`0x1C4F81`)
Playback is dispatched through Elektron's native polyphase resampler:
* 6-tap sinc FIR reconstruction.
* Zero digital aliasing under pitch transposition.
* Automatic de-clicking and sign-crossing zero-mute flags (`+0x17d`).
* Single-instruction execution overhead on the SHARC core.

---

## 3. Verified Memory Slot Map

```text
[Section 7 Blob (321,016 bytes exact)]
0x000000 ─────────────────────────────────────────────── Base
         ...
0x03C304 ─────────────────────────────────────────────── SW 0x1C6782 (STRETCH Slot Start, 472 Bytes Budget)
         [0x03C304..0x03C3ED]: Reserved / Dispatch Entry
0x03C3EE ─────────────────────────────────────────────── SW 0x1C67F7 (Verified Window, 206 Bytes Budget)
         [0x03C3EE..0x03C4BC]: CLOUDY Granular Injection Window
0x03C4BC ─────────────────────────────────────────────── SW 0x1C685E (Stock 0x1C4D88 Call Vector)
         [0x03C4BC..0x03C4DB]: CALL 0x1C4D88 & Exit Tail (jump 0x1C65FE)
0x03C4DC ─────────────────────────────────────────────── SW 0x1C686E (REPITCH Machine Start)
         ...
0x04E5F8 ─────────────────────────────────────────────── End of File (Offset 321,016 Exact)
```
## 4. Assembler Toolchain & Syntax Rules (`selas`)

Custom SHARC assembly injected into Section 7 must adhere to these verified toolchain constraints:
* **No Stack Pointer Modifies**: Do not use `i7 = modify(i7, n);` (compiles to `0f 17 00 00 00 00`, collapsing I7 to 0 and crashing the DSP). Use direct register arithmetic instead (`r2 = i7; r2 = r2 + r1; i7 = r2;`).
* **Strict DAG Memory Operands**: Memory accesses must use DAG index registers (`I0`..`I15`).
* **Register-Only Shifts & Comparisons**: Shifts and ALU comparisons require explicit data registers (`r3 = 13; r2 = lshift r1 by r3;`).

---

## 5. DSP Cycle Budget & Performance Ceiling (Rule 4)

* **Hardware Period**: 64 samples @ 96 kHz = **666.67 µs** per audio block.
* **Per-Voice Ceiling**: Across 32 active voices, each voice engine has a hard performance budget of **≤ 20 µs** (~10,000 SHARC cycles on a 500 MHz core).
* **Granular Engine Target**: The stochastic PRNG spray, dynamic slice scaling, and polyphase dispatch must execute in **≤ 4,000 cycles per voice** (~25% total DSP load) to match stock resampler numbers.

## 6. Community Machine Qualification & UX Quality Checklist (Machine 2 / STRETCH In-Place Swap)

Every custom machine replacing Machine 2 (STRETCH) must pass this 10-point user experience and sequencer qualification suite before release:

1. [ ] **Machine Box Alignment**: The short text abbreviation on the SRC page (`CLDY`) is <= 4 characters, ensuring the OLED machine select box renders at the factory width.
2. [ ] **MIDI CC Inheritance**: All 8 SRC page encoders respond accurately to external MIDI CC messages mapped to the Machine 2 parameter mirror indices.
3. [ ] **LFO Destination Routing**: The replaced parameter (`RATE`, formerly BARS) appears properly labeled in the Track LFO destination menu and modulates cleanly at audio rate without DSP clicks.
4. [ ] **Default Value Reset ([FUNC] + [NO])**: Pressing Clear Page on the SRC page resets all encoders to defined Cloudy defaults (`RATE` = 32 / 1.0x pitch, `STRT` = 0, `LEN` = 64).
5. [ ] **Parameter Randomization ([PAGE] + [YES])**: Page randomization produces musically useful granular variations without generating out-of-range DSP lockups or NaN audio mute traps.
6. [ ] **[FUNC] + Encoder Snapping**: Pressing [FUNC] while turning Knob G (`RATE`) engages integer semitone snapping; turning Knobs E/F engages musical division jumps.
7. [ ] **Macro Modulation (Velocity / Breath / Aftertouch / Mod Wheel)**: Mod matrix routing to `RATE`, `STRT`, and `LEN` scales smoothly without comb-filtering.
8. [ ] **Copy & Paste Page ([PAGE] + [REC] / [STOP])**: Parameter pages copy and paste between tracks and patterns without parameter drift or freezing.
9. [ ] **Sound Pool & Sound Locks**: Sounds saved using the Cloudy machine load cleanly from the +Drive Sound Pool and can be p-locked across sequencer steps.
10. [ ] **Stock Machine Coexistence**: All other factory engines (**OneShot, Werp, Repitch, Slice, and Manual Slice**) operate simultaneously across other tracks with 100% stock fidelity and timing.
