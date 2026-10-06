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
