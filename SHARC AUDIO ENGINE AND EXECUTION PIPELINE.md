---
title: "Digitakt II OS 1.17 — SHARC Audio Engine & Pipeline Mechanics"
tags:
  - sharc
  - audio-dsp
  - execution-pipeline
  - reverse-engineering
date: 2026-10-05
---

# SHARC Audio Engine & Pipeline Mechanics

## 1. Memory Spaces & Address Mapping

The Analog Devices ADSP-21569 uses multiple addressing spaces:

$$\text{RAM Byte Address} = 2 \times \text{Short-Word Address} + \text{0x28000000}$$

| Memory Region | Address Range | Role in Digitakt II OS 1.17 |
| :--- | :--- | :--- |
| **L1 SRAM** | `0x20000000` .. `0x2001AB9C` | Fast code/data (Block 69: 109,436 bytes). Ends at `0x2001AB9C`. |
| **L2 SRAM (Base)** | `0x28000000` .. `0x28100000` | Global engine memory, lookup tables, and buffers. |
| **L2 Audio Core** | `0x283827CC` .. `0x2839C000` | **Block 93** (Main Audio Engine: 104,500 bytes). |
| **DDR Memory** | `0x80000000` .. `0x8055E8A0` | Frame buffers, command queues, Block 101 dispatch tables. |

---

## 2. The Two-Stage Voice Execution Pipeline (`FUN_1c642a`)

Audio rendering inside `FUN_1c642a` (file offset `0x03BC54`, `SW 0x1C642A`) runs in two distinct, non-overlapping stages:

```
[ColdFire Frame @ 0x2558dc]
            │
            ▼
┌────────────────────────────────────────────────────────┐
│ STAGE 1: Note Trigger / Parameter Staging             │
│ - Fires on note-on / trigger events                    │
│ - Table 0x8055C840 (Block 101 @ File 0x04C588)         │
│ - Entry [2] jumps to STRETCH (SW 0x1C6782, 472B slot)  │
│ - Stock code sets pitch/loop bounds, calls 0x1C4D88    │
│ - Exits via: JUMP 0x1C65FE (db); r14 = 0x3ecf34d7;     │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│ STAGE 2: Audio Synthesis & Decimation Loop            │
│ - Runs EVERY 64 SAMPLES @ 96 kHz for all 32 voices     │
│ - Location: SW 0x1C6AE8                                │
│ - Checks Machine Selector (+0x4c):                     │
│     • If Selector == 2 (STRETCH): executes 0x1C5576    │
│     • If Selector != 2: executes 0x1C4ECF              │
│ - The Zero-Fill Trap: If byte +0x1b8 == 0, clears buf! │
│ - If byte +0x1b8 == 1, 0x1C4F81 interpolates slice!    │
│ - Mixer passes work_buffer to 0xb80000 (decimates 2:1) │
└────────────────────────────────────────────────────────┘
```

---

## 3. The Voice Record Contract (Stride: `0x1D8` Bytes / 118 Words)

Passed in register **`I4`**:

| Word Index | Byte Offset | Field Name | Type / Format | Function |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `+0x000` | `sample_ptr` | `const s16 *` | Pointer to 16-bit signed PCM audio in RAM (`0` if empty) |
| `0x01..0x40` | `+0x04..+0x103` | `work_buffer[64]` | `float[64]` | Audio output work buffer (64 samples @ 96 kHz) |
| **`0x41..0x5F`** | **`+0x104..+0x17F`** | **`scratchpad[31]`** | **`u32[31]`** | **124 bytes of persistent voice-isolated memory** |
| `0x60` | `+0x180` | `prev_sample` | `float` | De-click boundary tracking |
| `0x61` | `+0x184` | `sample_rate` | `u32` | Sample rate (e.g., 48,000) |
| `0x62` | `+0x188` | `sample_len_lo` | `u32` | Total sample length in samples |
| `0x64..0x65` | `+0x190..+0x194` | `loop_start` | `int64` (Q31) | Sample Loop Start bound (Start knob) |
| `0x68..0x69` | `+0x1A0..+0x1A4` | `loop_end` | `int64` (Q31) | Sample Loop End bound (Len knob) |
| `0x6a..0x6b` | `+0x1A8..+0x1AC` | `step` | `int64` (Q31) | Playback step / speed / pitch |
| `0x6c..0x6d` | `+0x1B0..+0x1B4` | `phase` | `int64` (Q31) | Accumulator phase |
| `+0x1b8` | `+0x1B8` | `active_flag` | `u8` | `1` = sounding, `0` = idle |
| `+0x1bb` | `+0x1BB` | `reverse_flag` | `u8` | `0` = forward, `1` = reverse |
| `+0x1bc` | `+0x1BC` | `loop_flag` | `u8` | `0` = one-shot, `1` = infinite loop |

---

## 4. The Machine Dispatch Table (`0x8055C840`)

Located in **Block 101 @ File `0x04C588`** (RAM `0x8055C840`):

```text
Machine [ 0] -> SW 0x1C65BD (OneShot)
Machine [ 1] -> SW 0x1C6715 (Werp)
Machine [ 2] -> SW 0x1C6782 (STRETCH Slot, 472 Bytes Budget)
Machine [ 3] -> SW 0x1C686E (REPITCH Slot)
Machine [ 4] -> SW 0x1C692F (Slice)
Machine [ 5] -> SW 0x1C6992 (MIDI / Secondary)
Machine [ 6] -> SW 0x1C69ED
Machine [ 7] -> SW 0x1C69C6
...
Machine [15] -> SW 0x1C7395
```
