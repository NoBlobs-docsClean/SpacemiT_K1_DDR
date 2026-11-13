<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Boot Interface](boot-interface.md) | [Index](README.md#index) | [Next: Clocks, PLLs and the PMU →](clocks-and-pmu.md)

# MMIO Map and Register Index

All registers are 32-bit, little-endian. Addresses in this file use the K1
values (`ddr = 0xc0000000`); portable code should derive everything from the
`ddr_base` argument (`phy = ddr + 0x40000`) and never hard-code `0xc004…` /
`0xc005…` literals.

## Address windows

| Window | Address | Role |
|---|---|---|
| DRAM, low window | `0x0000_0000` – `0x7fff_ffff` | First 2 GiB of DRAM |
| DRAM, high window | `0x1_0000_0000` – … | Rest of DRAM (up to `0x2_7fff_ffff` with 8 GiB) |
| SRAM: DDR info block | `0xc080_0000` (1536 B) | Written by some first stages **after** `ddr_init`; not a contract ([boot-interface.md](boot-interface.md)) |
| SRAM: DDR firmware | `0xc082_d000` – `0xc083_7fff` | Where the SPL copies and runs `ddr_fw.bin` |
| SRAM: BootROM data | from ≈`0xc083_8000` | Belongs to the BootROM (storage API pointer at `0xc083_8498`) |
| SRAM: SPL BSS | `0xc083_fc00` (1 KiB) | Mainline SPL BSS |
| **MC** | `0xc000_0000` (`ddr`) | Memory controller |
| **PHY** | `0xc004_0000` (`ddr + 0x40000`) | DDR PHY |
| **APMU / PMU** | `0xd428_2800` | DDR PLLs, clock mux, frequency-change kicks, MC/PHY reset |
| Boot flag | `0xd428_2d10` | Boot-mode flag (`sys` printed by the BootROM; the SPL writes its boot mode `0x55a`…`0x55e` here) |
| Boot strap | `0xd428_2c20` | `[10:9]` = boot medium strap (0 eMMC, 1 SPI, 2 NAND, 3 SD) |
| MPMU | `0xd405_0000` | `+0x200` WDTPCR (watchdog clocks/reset) |
| SoC watchdog | `0xd408_0000` | Used only for software reset ([test-platform.md](test-platform.md)) |
| Watchdog start | `0xd405_1020` | bit 4 starts the counter |
| `mtime` | `0xe400_bff8` | 64-bit machine timer, 24 MHz |
| **DFC table window** | `0xf000_0000` | Window through which the DFC table and the PHY analog markers are filled ([dfc-engine.md](dfc-engine.md)). On the reference board it aliases the MC. |
| PMIC P1 | I2C bus 8, address `0x41` | Board reset ([test-platform.md](test-platform.md)) |

## MC register index (`ddr + offset`)

"Per bank" registers exist four times, one copy per frequency point; the copy
being accessed is selected with `CTL_FREQ[31:30]` ([memory-controller.md](memory-controller.md)).

| Offset | Name | Role | Doc |
|---|---|---|---|
| `0x08` | STATUS | Per-rank init-done, CKE (*inferred*) and self-refresh bits | [self-refresh.md](self-refresh.md) |
| `0x20` | USER_0 | Command port: DRAM init, ZQCAL, self-refresh, FSP requests | [memory-controller.md](memory-controller.md) |
| `0x24` | CA_CMD (USER_1) | Command port: mode-register read and write | [mode-registers.md](mode-registers.md) |
| `0x28` | MISC | Command port: CA on (`0x13010000`) / off (`0x13008000`) | [memory-controller.md](memory-controller.md) |
| `0x44` | CTL0 | Global control; Device_Init writes `0x00040300` | [device-init.md](device-init.md) |
| `0x48`, `0x50`, `0x58`, `0x64` | — | Globals written by Device_Init (*opaque*) | [device-init.md](device-init.md) |
| `0x54`, `0x60`, `0x14c` | — | Globals not written by Device_Init but captured in the DFC snapshot (*opaque*) | [dfc-engine.md](dfc-engine.md) |
| `0x70` | TBL_INDEX | DFC table port: writing the index commits the entry | [dfc-engine.md](dfc-engine.md) |
| `0x74` | TBL_DATA | DFC table port: entry data | [dfc-engine.md](dfc-engine.md) |
| `0x78` | TBL_ADDR | DFC table port: entry target offset + flags | [dfc-engine.md](dfc-engine.md) |
| `0x80` | — | Global, `0` | [device-init.md](device-init.md) |
| `0x148` | — | Written `0x80ac0000` before every HW-DFC (*opaque*) | [dfc-engine.md](dfc-engine.md) |
| `0x180` | — | Global, `0x00010200` (*opaque*) | [device-init.md](device-init.md) |
| `0x200` | CS0 map | Present bit + size code | [address-map.md](address-map.md) |
| `0x204` | CS0 start high | 0 (CS0 starts at 0) | [address-map.md](address-map.md) |
| `0x208` | CS1 map | Present bit + size code + start (low bits) | [address-map.md](address-map.md) |
| `0x20c` | CS1 start high | Upper bits of the CS1 start | [address-map.md](address-map.md) |
| `0x220` / `0x224` | CS0 / CS1 geometry | Rows, columns, banks | [address-map.md](address-map.md) |
| `0x2c0`, `0x2c4` | — | Globals (*opaque*) | [device-init.md](device-init.md) |
| `0x2c8` | — | bit 17 = VOH / PU-cal select; rest `0xffff` | [device-init.md](device-init.md) |
| `0x2cc` | — | Auto power-save, 0 = off | [device-init.md](device-init.md) |
| `0x300` | DRAM_CFG1 | Per bank: RL/WL (MR2 source) | [device-init.md](device-init.md) |
| `0x304` | CTL_FREQ | `[31:30]` active fp / bank select, `[29:28]` frequency select | [memory-controller.md](memory-controller.md) |
| `0x308` | — | Field `[27:20]` set to `0x10` after the banks are filled (*opaque*) | [device-init.md](device-init.md) |
| `0x30c` | TXVREF / DRAM_CFG | Per bank: CA Vref (MR12), DQ Vref (MR14), SoC ODT | [device-init.md](device-init.md) |
| `0x310` / `0x314` | DRAM_CFG5 | Per bank: DRAM Tx ODT and drive (CS0 / CS1) | [device-init.md](device-init.md) |
| `0x34c` | — | Per bank timing (*opaque*) | [device-init.md](device-init.md) |
| `0x370` | MR_DATA | Mode-register read result; bit 31 = valid | [mode-registers.md](mode-registers.md) |
| `0x380` – `0x388` | — | Refresh parameters (global) | [device-init.md](device-init.md) |
| `0x38c` – `0x3c4`, `0x3d8`, `0x3fc` | — | Per bank timings | [device-init.md](device-init.md) |
| `0x400` | WDQS | Per bank, LPDDR4 only | [configurations.md](configurations.md) |
| `0xa00`, `0xac0` | — | Globals, `0` | [device-init.md](device-init.md) |
| `0xacc` | — | Global, `0xffffffff` | [device-init.md](device-init.md) |
| `0x13d0` | PI_CMD | PI training commands; also DFI `init_start` | [pi-training-engine.md](pi-training-engine.md) |
| `0x13e0` | PI_CFG | `[3:2]` = active fp (written by init/DFC, never by training) | [pi-training-engine.md](pi-training-engine.md) |
| `0x13e4` | DFI timing | Per bank; changes in x8 byte mode | [device-init.md](device-init.md) |
| `0x13e8` | PI_WL_CFG | Write-leveling engine parameters | [pi-training-engine.md](pi-training-engine.md) |
| `0x13ec` | PI_RD_CFG | Read-training engine parameters (also in the per-bank list) | [pi-training-engine.md](pi-training-engine.md) |
| `0x13f0` / `0x13f4` | PI_CA0 / PI_CA1 | CA-training engine parameters | [ca-training.md](ca-training.md) |
| `0x13f8` | PI_WR_CFG | Write-training engine parameters | [pi-training-engine.md](pi-training-engine.md) |
| `0x13fc` | PI_STATUS | Sticky done bits; bit 31 also "PHY init done" | [pi-training-engine.md](pi-training-engine.md) |

## PHY register index (`phy + offset`)

### Per-frequency slices

`slice(fp) = phy + (fp << 14)`:

| fp | Slice base |
|---|---|
| 0 | `phy + 0x0000` |
| 1 | `phy + 0x4000` |
| 2 | `phy + 0x8000` |
| 3 | `phy + 0xc000` |

Inside a slice:

| Region | Offset | Content |
|---|---|---|
| DATAPHY0 A | `+0x0000` | Byte 0 (CS1 copy at `+0x0100`) |
| DATAPHY0 B | `+0x0200` | Byte 2 (CS1 copy at `+0x0300`) |
| DATAPHY1 A | `+0x1000` | Byte 1 (CS1 copy at `+0x1100`) |
| DATAPHY1 B | `+0x1200` | Byte 3 (CS1 copy at `+0x1300`) |
| CAPHY | `+0x2000` | CA delays (`0xf` while CA training is off) |
| COMMON A | `+0x3000` | Analog of sub-PHY A: `+0x4` = VREF0 |
| COMMON B | `+0x3200` | Analog of sub-PHY B: `+0x4` = VREF1 |

Pitfall: COMMON is relative to the **slice**, not to DATAPHY1.
`slice + 0x1000 + 0x3004` lands in the next frequency's slice.

### Global blocks

| Address | Name | Role |
|---|---|---|
| `phy + 0x10000` | OTHER | bit 0 set by the PHY init |
| `phy + 0x10024` | — | Written 0 by the PHY init |
| `phy + 0x10028` | PHY_INIT | Training FSM arm: write 512 at the start of every pass |
| `phy + 0x10100` / `+0x10104` / `+0x10108` | — | Analog marker registers, filled through the DFC window |
| `phy + 0x18000` | PHY_STATUS | Training error bits (1 WL, 2 read leveling, 3 write DQ, 4 read gate) |
| `phy + 0x18004` | CA_STAT | `0x01008870` by default |
| `phy + 0x18200` | Harvest RX | 8 words, per-DQ read margins |
| `phy + 0x18220` | Harvest RX DM | 1 word |
| `phy + 0x18300` | Harvest TX | 8 words, per-DQ write margins |
| `phy + 0x18320` | Harvest TX DM | 1 word |

The TRAINING block is **global**, not per slice. Details in [phy.md](phy.md).

## PMU register index (`pmu = 0xd4282800`)

| Offset | Name | Role | Doc |
|---|---|---|---|
| `0x04` | CC_AP | Frequency-change kicks: bit 26 SW-FC, bit 22 HW-DFC (self-clearing busy) | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x98` | AP_IMR | bit 4 masks the DCLK frequency-change interrupt | [dfc-engine.md](dfc-engine.md) |
| `0xa0` | AP_ISR | Interrupt status bits 1, 4, 15 (W1C *inferred*) | [dfc-engine.md](dfc-engine.md) |
| `0xb0` | `MC_HW_SLP_TYPE` | PHY clock enable, DCLK bypass, MC/PHY reset (bit 21), DFC mode and table number | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0xe8` | `MCK_CTRL` | `[1:0]` = 3 enables the MC clock | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x180` | `DFC_AP` | Runtime AP-triggered DFC control (level `0..7`, request, IRQ mask/clear) | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x188` | `DFC_STATUS` | Runtime DFC status (in-progress bit 0, current/target levels) | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x190`–`0x1ac` | `DFC_LEVEL_0..7` | 8 runtime DVFS level descriptors | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x398` | `PHY_LDO` | PHY supply mode, LPDDR4 vs LPDDR4X | [phy.md](phy.md) |
| `0x39c` | PLL1_CTRL1 | PLL1 VCO word (2400 vs 2666) | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x3a0` | PLL1_CTRL2 | PLL1 analog word | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x3a4` | (PHY bias) | `[15:8] = 0x0F` written by the PHY init | [phy.md](phy.md) |
| `0x3a8` | PLL2_CTRL1 | PLL2 word | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x3ac` | PLL2_CTRL2 | PLL2 analog word | [clocks-and-pmu.md](clocks-and-pmu.md) |
| `0x3b4` | CLKSEL | DDR clock mux + PLL enables + lock status | [clocks-and-pmu.md](clocks-and-pmu.md) |

## Gaps

- Reset (power-on) values of most MC globals and of the PMU registers are not
  established; only the values at entry to `ddr_init` are known
  ([clocks-and-pmu.md](clocks-and-pmu.md)).
- The meaning of the MC globals marked *opaque* is unknown.
- Whether the `0xf000_0000` window is always an alias of the MC or a separate
  RAM on other boards/revisions is unknown; filling the table through the
  window works in both cases ([dfc-engine.md](dfc-engine.md)).

---

[← Previous: Boot Interface](boot-interface.md) | [Index](README.md#index) | [Next: Clocks, PLLs and the PMU →](clocks-and-pmu.md)
