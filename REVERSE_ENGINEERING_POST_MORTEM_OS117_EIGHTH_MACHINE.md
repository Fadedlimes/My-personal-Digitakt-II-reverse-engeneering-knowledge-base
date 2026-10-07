# Digitakt II OS 1.17 — 8th Machine UI Findings & Post-Mortem

**Date:** October 2026  
**Target:** ColdFire MCF54415 (Section 3) & SHARC+ ADSP-21569 (Section 7)

---

## 1. Hardware Traps & Opcode Legality

* **The V04 (Illegal Instruction) Bug:**
  - `machinepatch.py` emitted `2F 7C` (`move.l #imm, (d16, %sp)`).
  - ColdFire MCF54415 (ISA A) is a RISC core that **lacks immediate-to-memory moves**.
  - Executing `2F 7C` triggers **Vector 4 (V04: Illegal Instruction)**.
  - Fix: Immediates must be staged through a data register (`20 3C ... 2F 40 ...`).

* **The V0B (Line-F) Open-Bus Trap:**
  - File offset `0x300820` is alignment padding inside `.rodata` right before font glyphs.
  - Instruction fetches from non-executable `.rodata` fault on the bus, reading floating `0xFFFF`.
  - In ColdFire, opcodes starting with `0xF...` trigger **Vector 11 (V0B: Line-F Exception)**.

---

## 2. Memory Boundaries & Runtime Allocation

* **The 0x310000 BSS Boundary:**
  - Memory past `0x310000` (`0x40310000`+) is assigned to the audio BSS allocator.
  - The runtime zero-fills this region at cold boot, wiping code with `0x0000` (which triggers V04).
* **Safe Caves:**
  - `cave_b` (`0x40315400`) and `0x403117C4` are inside the audio BSS / SRAM copy zone.
  - Cave 2 (`0x40265000`) sits 690 KB below `0x310000`, safe from zero-filling.

---

## 3. The C++ Cold Boot Panic Chain (PP...)

* **The Rank Map Key Drift (`std::out_of_range`):**
  - In the 1.17 profile, `list_source` was pointed to `0x401F5458` (C++ symbol strings).
  - The real machine order array `{0, 1, 2, 3, 6, 4, 5}` lives at `0x401F5058`.
  - Pointing to symbol strings caused `map.at(0)` to fail and throw `std::out_of_range`.

* **The Real `map::insert` Hook Site:**
  - `0x40051ED4` is `__cxa_atexit` registering the destructor `0x40051B34`.
  - The real `map::insert` call is at `0x40051EBE` (`jsr 0x401AAE3C`).

* **The 28-Byte Heap Vector Overflow:**
  - `0x4005262C` (`pea 0x1c`) and `0x40052652` (`lea 28`) hardcode 28 bytes for 7 machines.
  - Supplying 8 machines (32 bytes) overflows 4 bytes into adjacent heap objects.

* **The True Blocker: Table E (`list_filter`) at 0x400522C4:**
  - In `HANDOVER-2026-09-14-eighth-machine.md`, authors assumed Table E was "not blocking"
    because their emulator resumed from a post-boot snapshot (`boot400M.snap`).
  - On hardware cold boot, `FUN_40052210` builds Table E, which is hardcoded for 6 machines.
  - An 8-machine list causes iterator `%a4` to run out of bounds, crashing at `0x400522C4`.

---

## 4. Conclusion & Recommended Path

Adding an 8th machine requires simultaneously expanding Table D, Table E, vector heap
capacity, rank maps, and filter bounds across tightly coupled C++ templates.

**The safest path is replacing an underutilized machine (Machine 2 / STRETCH) in-place.**
In-place swap bypasses all C++ container bounds, runs custom DSP kernels cleanly, and
guarantees high-speed USB recovery stays active.
