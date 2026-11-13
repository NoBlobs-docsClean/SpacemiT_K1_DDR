<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# SpacemiT K1 — LPDDR4/4X Boot Initialization (Clean-Room Reverse Engineering)

This directory documents a **clean-room reverse engineering of the SpacemiT K1
SoC with respect to RAM initialization at boot**: everything that has to happen
between "the BootROM has loaded the first stage into SRAM" and "the first stage
can use DRAM": DDR PLLs and clock muxes, memory-controller (MC) programming, PHY
analog setup, JEDEC power-up of the LPDDR4X dies, mode registers, address map,
the hardware frequency-change engine (DFC), and the per-frequency training of
the data eye (write leveling, DQS gate, read and write Vref/delay centering).

The work was initially based on the **Banana Pi BPI-F3 with 8 GB of RAM**,
populated with a **Samsung K4UCE3Q4AA-MGCL** LPDDR4X package (two ranks,
byte-mode x8 dies, 8 Gb per die; the board EEPROM identifies it as
`k1-x_deb1`), and subsequently expanded and verified on a **Banana Pi BPI-F3
with 16 GB of RAM**, populated with a **Micron MT53E4G32D8** LPDDR4X package
(two ranks, byte-mode x8 dies, 16 Gb per die). All reference measurements in
these documents come from those two boards unless stated otherwise.

## Target

The specification is written for a DDR initialization image designed to be
**loaded by mainline U-Boot**. Mainline U-Boot's K1 SPL embeds a file called
`ddr_fw.bin` with binman, copies it to SRAM at `0xc082d000` and calls it as

```c
int ddr_init(u64 ddr_base, u32 cs_num, u32 data_rate, void (*puts)(const char *));
```

The only image that existed for this slot was a proprietary vendor binary.
These documents describe what the hardware requires so that an independent
image with the same ABI can be written from them alone.
[boot-interface.md](boot-interface.md) describes the slot and the calling
contract.

## How the information was obtained

The specification was produced under a strict **clean-room split** between an
**Analyst** (who investigated the existing boot software — including SpacemiT's
public GPL-2.0+ U-Boot BSP sources in `drivers/ddr/spacemit/k1x/` and the
training binary — and validated every hardware mechanism on silicon) and an
**Implementer** (who writes independent code from this hardware specification
alone).

Active hardware verification was performed **on real hardware using the board's
FDL function**: the BootROM's USB firmware-download mode, which accepts a
first-stage image over USB (`fastboot stage` / `fastboot continue`) and runs it
from SRAM. Each hypothesis about a register, an encoding, a wait condition or an
ordering rule was turned into a small experiment, loaded through FDL, and
checked against UART output, register dumps and DRAM integrity tests. Software
resets (SoC watchdog or PMIC) brought the board back to FDL without anyone
touching it, so many runs could be chained.

From the SD-card experiments onward, the first-stage image was also written
directly to the boot medium's first-stage slot (`fsbl`) and booted from it,
which exercises the production boot path end to end into Linux. Note that the
BootROM's fastboot `flash` command does **not** write the storage medium, and
recovering a non-booting SD card image uses the eMMC boot path with SD hot-plug;
see [test-platform.md](test-platform.md#loading-without-fdl-writing-the-boot-medium).
The method, the clean-room rules and the confidence labels are described in
[methodology.md](methodology.md); the experimental setup is in
[test-platform.md](test-platform.md).

## Scope

In scope:

- the silicon contract: MMIO registers, encodings, wait conditions, ordering
  rules and required values for the PMU (DDR part), MC, PHY, PI training engine,
  self-refresh, mode registers and the hardware DFC engine;
- the boot contract: what the DDR firmware must do, in which order, and what
  must be true before each step;
- the training algorithm that the hardware requires to center the eye, treating
  the PI sequencer as a black box;
- reference measurements on the BPI-F3: what the hardware produces when it is
  programmed as described, usable as acceptance ranges;
- for every topic, what is still unknown.

Out of scope:

- the internal edge-search algorithm of the PI sequencer (the firmware
  *triggers* it; the Rx/Tx/WL centring it performs can also be reproduced
  without it by a software memory-pattern sweep, see
  [eye-training.md](eye-training.md#independent-verification-of-the-eye-memory-pattern-sweep));
- the BootROM, the secure-boot signature format and the SPL itself;
- any description of the vendor program's code (see the clean-room rules).

## Index

Suggested reading order for a first pass. Afterwards, use each file as a
reference.

| # | File | Topic |
|---|---|---|
| 01 | [methodology.md](methodology.md) | Clean-room method, FDL-based experimentation, confidence labels |
| 02 | [glossary.md](glossary.md) | Terms and abbreviations used throughout |
| 03 | [system-overview.md](system-overview.md) | DDR subsystem topology, byte lanes, frequency points, reference board and DRAM part |
| 04 | [boot-interface.md](boot-interface.md) | Contract with mainline U-Boot SPL: load address, ABI, arguments, execution environment |
| 05 | [mmio-map.md](mmio-map.md) | All address windows and a register index for MC, PHY and PMU |
| 06 | [clocks-and-pmu.md](clocks-and-pmu.md) | PLL1/PLL2, clock mux, enable/lock, SW-FC, MC reset, 2400/2666 MT/s and 1333 (fp0 with the 2666 VCO) |
| 07 | [memory-controller.md](memory-controller.md) | MC registers, command ports (USER_0, CA_CMD, MISC), STATUS, timing banks |
| 08 | [device-init.md](device-init.md) | Device_Init: MC values, timing tables, DFI init, JEDEC power-up, ZQ, MR order |
| 09 | [phy.md](phy.md) | PHY slices, per-byte tap registers, gate code, analog COMMON block, status, harvest |
| 10 | [mode-registers.md](mode-registers.md) | MRR/MRW protocol, MR14 path, FSP mapping, read quirks, die values |
| 11 | [self-refresh.md](self-refresh.md) | Self-refresh protocol and the MC STATUS bits |
| 12 | [address-map.md](address-map.md) | CS size codes, CS1 start, geometry, derivation from MR8, CPU address map |
| 13 | [dfc-engine.md](dfc-engine.md) | Hardware DFC: table window, entry format, command flows, snapshot, kick |
| 14 | [pi-training-engine.md](pi-training-engine.md) | PI sequencer: commands, done bits, PI_STATUS semantics, CFG parameters |
| 15 | [training-pass.md](training-pass.md) | One training pass: preconditions, mandatory order, outputs, errors, stage-by-stage specification |
| 16 | [write-leveling.md](write-leveling.md) | Write-leveling stage |
| 17 | [read-gate.md](read-gate.md) | DQS read-gate stage and timeout detection |
| 18 | [eye-training.md](eye-training.md) | Read (PHY Vref) and write (MR14) sweeps, max-min Vref selection |
| 19 | [ca-training.md](ca-training.md) | CA training engine (present, unused by the LPDDR4X boot) |
| 20 | [boot-sequence.md](boot-sequence.md) | Complete `ddr_init` orchestration and why each pass exists |
| 21 | [configurations.md](configurations.md) | x8 byte mode, x16, 1 CS, 16 GiB, final rates 1066/1200/1333/2400/2666 MT/s, LPDDR4 (non-X), other SKUs |
| 22 | [reference-measurements.md](reference-measurements.md) | Acceptance criteria and measured values on the BPI-F3 |
| 23 | [test-platform.md](test-platform.md) | FDL loading, software resets, BootROM eMMC layout, latency meter |

## Conventions

- All registers are 32-bit, little-endian, accessed as aligned words.
- `ddr` is the MC base. On the K1 it is `0xc0000000`; it is passed by the SPL
  as the first argument.
- `phy = ddr + 0x40000` (PHY base).
- `pmu = 0xd4282800` (APMU, the application power-management unit).
- `fp` (frequency point, also `pp` or `boot_pp` in a training context) is the
  index 0..3 of one of the four MC timing banks / PHY slices.
- `slice(fp) = phy + (fp << 14)`: the 16 KiB PHY register slice of one
  frequency point.
- `cs_num` is the number of populated ranks (1 or 2). Chip selects are written
  CS0 and CS1. Command fields that select ranks use a CS mask
  (bit 0 = CS0, bit 1 = CS1); see [memory-controller.md](memory-controller.md).
- Data rates are in MT/s (the PHY/DRAM data rate, not the clock frequency).
- Register addresses are always given as `base + offset`; `ddr+0x3b4` and
  `pmu+0x3b4` are different registers.

### Confidence labels

Every fact in these documents falls into one of these classes. When a statement
carries no label it is **measured** or **required**.

| Label | Meaning |
|---|---|
| *measured* | Observed on the silicon of the reference board (register read-back, UART, DRAM tests). |
| *required* | A value or step the hardware needs: leaving it out was observed to break the initialisation, or the hardware was only observed to reach a working state with it. |
| *inferred* | Deduced from consistency (masks, encodings, JEDEC, centering behaviour) but not independently measured. |
| *opaque* | The value is known and needed, the meaning of its bits is not. Write it exactly; do not reinterpret it. |
| *unknown* | Not established. Do not invent values; preserve what the hardware holds. |

## Coverage at a glance

| Configuration | How well it is characterised |
|---|---|
| LPDDR4X, 2 CS, x8 dies, 8 GiB (reference board) | Whole sequence measured on hardware, from a cold controller to a fully mapped DRAM; all 8 GiB pass pattern tests |
| Same board at 2400 and 2666 MT/s | Measured; 2666 (SoC maximum) has smaller training margins, as expected from the shorter UI |
| Same board at 1066 and 1200 MT/s (final fp1 / fp0) | Measured: boot, training and benchmarks at each rate |
| Same board at ≈1333 MT/s (fp0 with the 2666 VCO) | Measured; an image must special-case 1333 (PLL1 at 2666 VCO with `fp0` as final point) |
| Same board with fp0 as the final point (what a 2133 MT/s request maps to on LPDDR4X) | Measured: the DFC from fp2 back to fp0 completes |
| 1 CS / x16 (4 GiB SKUs) | Derived from public BSP tables and command formats; not measured |
| 16 GiB (2 CS × 8 GiB) | Fully measured on a Micron `MT53E4G32D8` board: MR5/MR8, address map, all four training passes, all five final rates, PI and independent Rx/Tx/WL windows, per-lane and per-rank boundaries, ODT/IO sweeps, and a 15-minute memory soak |
| 16 GiB at all final rates (1066/1200/1333/2400/2666) | Measured: boot, training margins and Linux throughput/latency at each rate |
| Tx ODT and other IO fields | Measured: on the 16 GiB Micron board, 48–120 Ω DRAM DQ ODT are equivalent, 40 Ω fails fp2 write training (`PHY_STATUS` bit 3), and 240 Ω degrades Tx margins; varying SoC DQ drive (`COMMON+0x4[7:0]`), DRAM output drive (`ddr+0x310/314[19:16]`), PHY DQ ODT (`COMMON+0x4[15:8]`), SoC ODT (`ddr+0x30c[15:12]`), and PHY Rx Vref (`VREF0/1[7:0]`) gives no margin gain at 2666. On the 8 GiB Samsung board, 48 Ω **fails** at 2666 (60–120 Ω safe) — a board/die difference |
| 8 GiB vs 16 GiB parity | All tests from the 16 GiB board were repeated on the 8 GiB board: rank boundary, WL/Rx/Tx per rank, per-lane Rx/Tx centring, ODT/IO sweeps, Vref(CA) sensitivity, and 15-minute soak. SoC-level behaviours are identical; board/die-dependent thresholds differ |
| Rank layout (CPU addresses belonging to CS0/CS1) | Measured on both boards: ranks are contiguous (not fine-interleaved). On the final map, CS1 starts at CPU address **10 GiB** (`0x2_8000_0000`, DRAM offset 8 GiB) on the 16 GiB board and **6 GiB** (`0x1_8000_0000`, DRAM offset 4 GiB) on the 8 GiB board |
| LPDDR4 (non-X) | IO table and sequence derived from public BSP tables; no reference board, MR14 sweep range unknown |
| Training without the PI sequencer | Demonstrated for Rx, Tx, WL and DQS gate: software memory-pattern sweeps reproduce the PI sampling points to 0–1 code on both ranks, with per-lane centring required at fp2 (~20 s total vs ~20 ms for the PI sequencer) |

What is still unknown is documented where it belongs: every topic file ends
with a "Gaps" section for that topic. The gaps with the most practical impact
(other memory types, rank counts and capacities) are in
[configurations.md](configurations.md).
