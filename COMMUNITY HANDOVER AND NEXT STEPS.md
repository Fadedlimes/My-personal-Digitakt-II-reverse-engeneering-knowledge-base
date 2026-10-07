---
title: "Digitakt II OS 1.17 — Community Handover & Next Steps"
tags:
  - handover
  - roadmap
  - task-list
date: 2026-10-06
---

# Community Handover & Next Steps

## 1. Current State Summary
1. **Hardware Safety**: Device is unbricked, stable, running stock OS 1.17, and high-speed USB (`hw:1,0,0`) OS upgrade is active.
2. **DSP Engine Success (Section 7)**:
   * Cloudy is injected into **Slot 11** (`SW 0x1C6DD6`, File `0x03CFAC`).
   * The Stage 2 DSP hang is completely resolved (runs via sinc interpolator `0x1C4ECF`).
3. **ColdFire UI Crash Resolved (Section 3)**:
   * **The Problem**: Expanding the UI to 8 machines using the tail padding at `0x403117C4` caused an `EXCEPTION DS0082 V04` at `P40311A64`.
   * **The Root Cause**: `0x40311A64` lies beyond the active memory-mapped execution page (which ends at `0x403048BD` in OS 1.17). The hardware MMU blocked the instruction fetch.

## 2. The Path Forward: "The Internal Text Cave"
Do not overwrite any factory machines. The user requires the Slice and Manual Slice engines to remain intact. We will safely add Cloudy as the 8th machine by moving the ColdFire hooks into an active, mapped execution page.

### Next Agent Action Plan:
1. **Verify the Internal Cave**:
   * Inspect the 254-byte zero-run at `0x40300820` (File Offset `0x300820`) in `section3_unpacked.bin`.
   * Confirm it contains zeros in both the static binary and the `boot400M.snap` runtime memory.
2. **Relocate Section 3 Hooks**:
   * Repoint the C++ map `RANK_SHIM` and the machine `TRAMPOLINE` to `0x40300820` (instead of `0x403117C4`).
   * Because `0x40300820` is safely inside the active `.text` boundary, the CPU will execute the 8th machine logic without MMU page faults.
3. **Build & Flash**:
   * Patch Section 3 and Section 7.
   * Run offline gate checks (LDR lengths, RISC opcodes).
   * Repack to `Digitakt_II_OS1.17_cloudy.syx` and instruct the user to flash.
