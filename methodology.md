<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[Index](README.md#index) | [Next: Glossary →](glossary.md)

# Methodology

## Clean-room approach

The work follows a clean-room split between two roles.

**Analyst.** Investigates the hardware and existing boot software (including
SpacemiT's public GPL-2.0+ U-Boot BSP sources in `drivers/ddr/spacemit/k1x/`,
static analysis of the training binary, and active hardware probing on the
reference board) and writes the specification. Every fact enters the
specification as either

- behaviour of the **silicon** (a register, a field, an encoding, a wait
  condition, a side effect): *programming X produces Y*; or
- a **hardware requirement** of the initialisation (a step, its order relative
  to other steps, the value the hardware needs, how to verify the result).

A fact is never written as "the other firmware does X", nor as "our code does
X", and software-internal artifacts (function names, control-flow quirks,
compiler bit-packing, stack layouts) are stripped out. Opaque values needed by
the hardware (for example the PI sequencer parameters) are listed as engine
parameters — *reset value X, the hardware needs Y* — without claiming to know
what the bits mean.

**Implementer.** Whoever writes DDR firmware from this specification uses only

- this specification;
- public documents: JEDEC JESD209-4 (LPDDR4) and JESD209-4-1 (LPDDR4X), the
  datasheet of the DRAM part on the board, and the public SpacemiT K1
  datasheet / manuals where available;
- measurements taken on the board.

The implementer does not read listings, decompilations, linker maps, strings or
source code of the proprietary training binary that originally initialised this
SoC, and does not transcribe function names, log strings, stack layouts or code
addresses of another program. Programming the **silicon's registers** described
here is not copying: it is the hardware.

Rules derived from this:

1. If a step is not in the specification, the implementer stops and asks for an
   amendment instead of guessing from memory of other code.
2. New measurements (UART, MMIO reads) are welcome input to the specification;
   source code of another DDR training implementation is not.
3. The specification describes hardware only. It contains no information about
   any particular implementation (its structure, code, timings or results).
4. Running new firmware on the board and comparing the hardware state with the
   reference measurements ([reference-measurements.md](reference-measurements.md))
   does not break the clean room. Comparing a new binary with another binary
   byte by byte would defeat the purpose: new code does **not** have to be
   identical to anything. Equivalence is defined by the **hardware state**
   reached (status registers, trained values within the reference ranges,
   DRAM integrity), not by code.

## Trial and error through FDL

The K1 BootROM has a USB download mode (FDL, *firmware download*). When the
boot media fail, or when the FDL button is held at reset, the BootROM exposes a
fastboot-compatible USB device that accepts a signed first-stage image, places
it in SRAM and runs it. This is the tool that made the reverse engineering
possible without any boot medium and without risk of bricking the board.

The experimental loop:

1. Build a first-stage image (mainline U-Boot SPL plus a DDR image in the
   `ddr_fw.bin` slot) that programs the registers under test and dumps the
   resulting state over the UART.
2. Sign it with the vendor signing tool (the BootROM verifies the image; the
   tool is only executed, never studied).
3. Load and run it through FDL: `fastboot stage <image>` then
   `fastboot continue`.
4. Capture the UART (115200 baud) and compare the observed state with the
   hypothesis.
5. Reset the board by software (SoC watchdog or PMIC soft reset,
   [test-platform.md](test-platform.md)); the BootROM falls back to FDL
   by itself and the next run starts. Runs can alternate between two variants
   (A/B) with nobody at the board.

> **Do not confuse FDL with flashing.** The BootROM's fastboot `flash` command
> reports `OKAY` but the `fsbl` partition it exposes has size 0: the image is
> only downloaded to SRAM, not written to the SD card or eMMC. `continue` then
> runs that SPL in download mode, where it tries to fetch the next stage over
> fastboot (`SPL load error: bootdev=12`) and stops. To change the boot medium
> image, boot Linux from another medium (for the reference board, the eMMC,
> with the SD card removed), insert the SD card while it runs (the SD host has a
> card-detect GPIO and accepts hot-plug) and write the partition with `dd`.

Kinds of experiments that produced the facts in this specification:

- **Register dumps** before and after each step (PMU, MC STATUS, `CTL_FREQ`,
  `PI_CFG`, `PI_STATUS`, PHY status, taps, Vref words, mode registers).
- **Ordering and omission tests**: removing or moving a step and observing
  what the hardware does (for example the DFC engine hanging when its table is
  incomplete).
- **Encoding probes**: changing one field and watching the effect (CS size
  codes, PLL words, command formats).
- **Fault experiments**: deliberately leaving a precondition unmet (PLL
  enables cleared, a read-gate wait without a command, a DFC kick without a
  table) to see how the hardware reports it.
- **A/B comparisons** between two ways of programming a step (for example
  re-writing MR14 after a frequency change, which turned out to change
  nothing).
- **Latency probes** as a frequency meter, because the PLL block has no
  frequency counter (pointer-chasing loads timed by `mtime`).
- **DRAM integrity checks**: pattern tests and address markers across the whole
  8 GiB, and execution of code from DRAM.

## Confidence labels

Used throughout the documents (see also the [README](README.md)):

| Label | Meaning |
|---|---|
| *measured* | Observed on the reference board. |
| *required* | Needed by the hardware: omitting it was observed to break the initialisation, or the hardware was only observed to reach a working state with it. |
| *inferred* | Consistent with all observations and with JEDEC, but not isolated by a dedicated measurement. |
| *opaque* | Value known and required; meaning of the bits unknown. Write it as given. |
| *unknown* | Not established. Preserve hardware values; do not invent. |

## Limits of the method

- The **PI training sequencer's internal FSM** is treated as a black box. Its
  internal edge-search state machine is not documented at the RTL level, but the
  specification provides everything required both to drive the PI sequencer in
  hardware (~20 ms total across passes) and, if desired for validation or
  characterisation, to sweep and centre all four delay/Vref axes directly in
  software via memory-pattern tests (write leveling in `byte+0x24`, DQS read
  gate delay in `byte0+0x64[4:0]`, RX Vref + DQ delays, and TX MR14 + DQ/DM
  delays).
- Power-on-reset values of several PMU registers cannot be observed: by the
  time the DDR image runs, the BootROM and SPL have already touched them.
  Values *at entry* to `ddr_init` are measured and documented instead.
- Two boards were available for active hardware probing: BPI-F3 8 GiB (Samsung
  `K4UCE3Q4AA-MGCL`, LPDDR4X x8, 2 CS) and BPI-F3 16 GiB (Micron
  `MT53E4G32D8`, LPDDR4X 16 Gb x8 dies, 2 CS). On both boards the address map,
  MR5/MR8, the four training passes, all five final rates, the PI and
  independent Rx/Tx/WL windows (including per-lane centring and the per-rank
  CPU address boundary), ODT/IO sweeps and 15-minute memory soaks were measured.
  Other configurations (1 CS, x16, LPDDR4 non-X, Hynix `H9HCNNNFAMMLXR-NEE` x8
  CA-ODT SKU, and other DRAM vendors) are derived from public BSP tables,
  command formats and JEDEC encodings, and are marked as not measured on
  hardware.
- The DDR image was exercised both through FDL (loaded into SRAM) and written to
  the boot medium's first-stage slot (`fsbl`) and booted from it into Linux.
  Only the second path exercises the full production boot; FDL is used for rapid
  iteration without touching the boot medium.

---

[Index](README.md#index) | [Next: Glossary →](glossary.md)
