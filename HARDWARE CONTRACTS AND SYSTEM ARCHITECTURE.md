---
title: "Digitakt II OS 1.17 — Hardware Contracts & System Architecture"
tags:
  - elektron
  - digitakt-ii
  - reverse-engineering
  - sharc-dsp
  - coldfire
date: 2026-10-05
---

# Digitakt II OS 1.17: Hardware Contracts & System Architecture

## 1. Dual-Core Processing Topology

The Elektron Digitakt II operates on a split asymmetric dual-core architecture:

```
┌──────────────────────────────────────────────┐
│        NXP ColdFire MCF54415 (Section 3)     │
│  - Linux / Main OS & UI Engine               │
│  - Sequencer & Pattern Memory                │
│  - Front Panel Encoders, OLED Display        │
└──────────────────────┬───────────────────────┘
                       │ High-Speed SPI / FlexBus
                       ▼ (321,016-byte boot stream DMA)
┌──────────────────────────────────────────────┐
│      Analog Devices SHARC+ ADSP-21569        │
│                (Section 7)                   │
│  - 32-Voice Audio Synthesis Engine           │
│  - 96 kHz Internal Work Buffers              │
│  - 2:1 Polyphase Decimator (96 kHz -> 48 kHz)│
│  - Hardware Interpolators & Effects          │
└──────────────────────────────────────────────┘
```

---

## 2. Firmware Container Invariants (The Golden Rules)

Firmware container: `Digitakt_II_OS1.17.syx`, unpacked and repacked via `./elektron-firmware-tool`.

### The Bit-for-Bit Matching Law
The ColdFire SPI bootloader uses **hardcoded DMA transfer counters** configured in hardware registers during early startup. It feeds the DSP executable stream directly to the SHARC SPI slave port at boot:

| Firmware Section | Physical Binary | Required File Length | Boot Failure Mode on Deviation |
| :--- | :--- | :--- | :--- |
| **Section 3** | `section_3_MAIN_OS.bin` | **`3,275,616 bytes`** | Boot ROM halt / CRC rejection |
| **Section 7** | `section_7_BLOB.bin` | **`321,016 bytes`** | **SPI DMA Desync $\rightarrow$ DSP never boots $\rightarrow$ OS freeze** |

> [!CAUTION]
> **Never append blocks or alter Section 7 file size.**
> Appending even a single 2 KB LDR block (size becomes 323,048 bytes) desyncs the SPI DMA counter. The DSP will fail to issue the "DSP Ready" handshake interrupt, freezing the ColdFire CPU on the startup screen.

---

## 3. Recovery Paths & Transport Mechanics

### Path A: High-Speed USB OS Upgrade (Fast Path — ~20 Seconds)
When the unit boots and the UI is functional:
1. Connect via USB.
2. Navigate to **`[SETTINGS]` $\rightarrow$ `SYSTEM` $\rightarrow$ `OS UPGRADE`**.
3. Stream the container:
   ```bash
   amidi -p <usb_port> -s Digitakt_II_OS1.17.syx -i 5
   ```
4. Transfer finishes in $\approx 20\text{ seconds}$ at full USB MIDI speed.

### Path B: Early Startup Bootstrap Menu (Fallback — 8–12 Minutes)
If the DSP is soft-bricked and the UI fails to initialize:
1. Power off. Hold **`[FUNC]`** while powering on until the white bootstrap menu appears.
2. Press **Trig 3** (`EMPTY RESET`), then **Trig 4** (`MIDI UPGRADE`).
3. Screen displays `WAITING FOR MIDI`.
4. **Hardware Constraint**: Bootstrap recovery operates strictly over **5-pin DIN MIDI** at standard 31.25 kbaud:
   ```bash
   amidi -p <din_port> -s Digitakt_II_OS1.17.syx -i 5
   ```
5. Takes **8 to 12 minutes** to stream the 1.4 MB image over DIN.

---

## 4. The Community Testing Invariant (Rule 4)

> **"Every synth feature gets stress-tested and its performance checked against the factory machines' numbers before it counts as done."**

### Hard Performance Ceilings:
* **Audio Block Period**: 64 samples @ 96 kHz = **$666.67\,\mu\text{s}$**.
* **Voice Ceiling**: Across 32 voices, each voice engine has an upper bound of **$\approx 20\,\mu\text{s}$** ($\approx 10,000$ SHARC cycles on a 500 MHz core).
* **Target Budget**: Any custom machine must run in $\le 4,000$ cycles per voice ($\approx 25\%$ total DSP load across 32 voices) to match stock resampler numbers.
```
