<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Methodology](methodology.md) | [Index](README.md#index) | [Next: System Overview →](system-overview.md)

# Glossary

| Term | Meaning |
|---|---|
| **APMU / PMU** | Application power-management unit at `0xd4282800`. Holds the DDR PLLs, clock mux, frequency-change kicks and the MC/PHY reset. Called `pmu` in these documents. |
| **MPMU** | Main PMU at `0xd4050000`; only used here for the SoC watchdog clocks ([test-platform.md](test-platform.md)). |
| **MC** | DDR memory controller at `ddr = 0xc0000000`. Contains the command ports, timing banks, address map, the PI training sequencer and the DFC table. |
| **PHY** | DDR PHY at `ddr + 0x40000`. Delay taps, analog (Vref, ODT, drive), training status and result ("harvest") registers. |
| **DATAPHY0 / DATAPHY1** | The two data blocks of the PHY. Each contains two sub-PHYs (A and B), each sub-PHY one byte lane. |
| **CAPHY** | Command/address block of the PHY. |
| **COMMON** | Per-slice analog block of the PHY (Rx Vref, ODT, drive strength, pre/postamble). |
| **OTHER** | Global PHY block at `phy + 0x10000` (contains `PHY_INIT`). |
| **TRAINING** | Global PHY block at `phy + 0x18000` (training status and harvest). |
| **Slice** | 16 KiB PHY register window holding the settings of one frequency point: `phy + (fp << 14)`. |
| **VCO family** | Which of the two PLL1 VCO configurations is programmed at boot, selected by the requested final data rate: 2400 (for final rates 1066/1200/2400, giving fp0 = 1200 and fp2 = 2400) or 2666 (for final rates 1333/2666, giving fp0 ≈ 1333 and fp2 = 2666). fp1 is 1066 in both families. |
| **fp / pp / boot_pp** | Frequency point 0..3. The MC holds one timing bank and the PHY one slice per fp. `boot_pp` is the fp currently being trained. |
| **Bank (MC)** | One of four copies of the MC timing/IO registers, one per fp, selected through `CTL_FREQ[31:30]`. Not to be confused with DRAM banks. |
| **CS / rank** | Chip select. The K1 supports up to two (CS0, CS1). |
| **CS mask / `cs_sel`** | Two-bit rank selector in command words: bit 0 = CS0, bit 1 = CS1; `cs_sel = 1` with one rank, `3` with two. |
| **JEDEC channel (CH)** | An LPDDR4 package has two independent 16-bit channels (CH0, CH1). The K1 PHY combines them into one 32-bit bus. |
| **x8 / byte mode** | LPDDR4 dies with an 8-bit interface ("byte-mode"); two of them form one 16-bit JEDEC channel. Reported in MR8[7:6]. |
| **x16** | Standard LPDDR4 die organisation, one die per 16-bit channel. |
| **MR / MRR / MRW** | DRAM mode register / mode-register read / mode-register write. |
| **FSP** | Frequency set point (LPDDR4): each die holds two copies of the frequency-dependent MRs (FSP-0, FSP-1). MR13 selects which one is written (FSP-WR) and which one is operating (FSP-OP). |
| **MPC** | JEDEC multi-purpose command (used here for ZQ calibration). |
| **ZQCAL** | DRAM output-driver impedance calibration (MPC START, then LATCH). |
| **SR** | Self-refresh. Training runs with both ranks in SR. |
| **DFI** | Interface between MC and PHY. `init_start` triggers PHY initialisation. |
| **PI** | The MC's training sequencer ("PI engine"). It runs write leveling, gate, read and write training on command and writes taps and margins into the PHY. |
| **WL** | Write leveling: aligns DQS with CK at the DRAM. |
| **RGATE / gate** | DQS read gate: the window during which the PHY listens to DQS on reads. |
| **RD / Rx training** | Read eye centering (DQ delays at a given PHY Rx Vref). |
| **WR / Tx training** | Write eye centering (DQ/DM delays at a given DRAM Vref(DQ)). |
| **Vref(DQ)** | DRAM-side reference voltage for written data, set with MR14. |
| **Vref(CA)** | DRAM-side reference voltage for command/address, set with MR12. |
| **Rx Vref** | PHY-side reference voltage for read data, set in the PHY COMMON block. |
| **Harvest** | PHY registers holding per-DQ margins latched by the last PI read or write operation. |
| **Max-min** | Vref selection rule: for every Vref step take the minimum margin over all DQs and ranks; pick the step with the largest minimum. |
| **Tap** | One unit of a PHY delay line. |
| **UI** | Unit interval: one data bit time (`1 / data rate`). |
| **SW-FC** | Software frequency change: PLL/mux change with DRAM not yet initialised (kick bit 26 of `pmu+0x04`). |
| **HW-DFC / DFC** | Hardware dynamic frequency change: a table-driven engine in the MC/PMU that parks the DRAM, switches the clock, reprograms the dies and resumes (kick bit 22 of `pmu+0x04`). |
| **DFC table** | Command list consumed by the HW-DFC engine, filled through the window at `0xF0000000`. |
| **Device_Init** | The cold initialisation of MC + PHY analog + DRAM dies at fp0, before the first training pass. |
| **FDL** | BootROM USB firmware-download mode (fastboot protocol). |
| **FSBL** | First-stage boot loader image as loaded by the BootROM (here: mainline SPL + DDR firmware, signed). |
| **SPL** | U-Boot Secondary Program Loader; on the K1 it is the first stage, running in SRAM in M-mode. |
| **`ddr_fw.bin`** | The DDR firmware image that mainline U-Boot embeds in the SPL image and calls. |
| **TPKS** | The BootROM's image-signature verification (name as printed by the BootROM). |
| **P1** | SpacemiT P1 PMIC on I2C bus 8, address `0x41`. |

---

[← Previous: Methodology](methodology.md) | [Index](README.md#index) | [Next: System Overview →](system-overview.md)
