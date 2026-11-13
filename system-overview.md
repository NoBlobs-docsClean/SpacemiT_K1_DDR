<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Glossary](glossary.md) | [Index](README.md#index) | [Next: Boot Interface →](boot-interface.md)

# System Overview

## DDR subsystem topology

From the point of view of the boot firmware the K1 exposes **one 32-bit memory
channel**: one memory controller (MC), one PHY with four byte lanes, and up to
two chip selects (ranks).

```
   CPU (SPL, M-mode, SRAM)                     APMU  0xd4282800
          |                                    PLL1 / PLL2, clock mux,
          | 32-bit MMIO                        SW-FC & HW-DFC kicks,
          v                                    MC/PHY reset
  +---------------------+        +---------------------------------+
  | MC   0xc0000000     |  DFI   | PHY   0xc0040000                |
  |  command ports      |------->|  4 slices (one per fp) x 16 KiB |
  |  4 timing banks     |        |   DATAPHY0 A/B, DATAPHY1 A/B    |
  |  address map        |        |   CAPHY, COMMON (analog)        |
  |  PI training engine |        |  OTHER    (PHY_INIT)            |
  |  DFC table          |        |  TRAINING (status, harvest)     |
  +---------------------+        +----------------+----------------+
                                                  |
                       DQ[31:0] DM[3:0] DQS[3:0]  |  CA, CK, CS[1:0]
                                                  v
                          LPDDR4/4X package: 2 JEDEC channels x 16 bit,
                          1 or 2 ranks, x16 or x8 (byte-mode) dies
```

The two 16-bit JEDEC channels of the package (CH0, CH1) are presented to the MC
as one 32-bit bus. Mode-register reads address a (CS, CH) pair; mode-register
writes issued during initialisation go to both channels at once
([mode-registers.md](mode-registers.md)).

### Byte lanes

The PHY data path is split into two DATAPHY blocks, each with two sub-PHYs
(A and B). Each sub-PHY drives one byte lane.

| Byte | Block | Sub-PHY | DQ lanes | Offset inside a slice |
|---|---|---|---|---|
| 0 | DATAPHY0 | A | DQ[7:0] | `+0x0000` |
| 1 | DATAPHY1 | A | DQ[15:8] | `+0x1000` |
| 2 | DATAPHY0 | B | DQ[23:16] | `+0x0200` |
| 3 | DATAPHY1 | B | DQ[31:24] | `+0x1200` |

The CS1 copy of each byte's registers sits `+0x100` above the CS0 copy. All four
bytes are trained independently; results of sub-PHY A must not be copied to
sub-PHY B — each lane has its own skew. Register details are in
[phy.md](phy.md).

### Frequency points

The MC keeps **four** timing banks and the PHY **four** register slices, one per
frequency point (fp). Switching frequency selects a bank and a slice; it does not
reprogram them. Each fp is trained separately, so its slice keeps the taps of
its own rate.

| fp | Clock source | Data rate | Notes |
|---|---|---|---|
| 0 | PLL1 / 2 | 1200 MT/s (≈1333 when PLL1 runs the 2666 VCO) | Cold-boot point; Device_Init runs here; final point for 1200 (2400 VCO) and 1333 (2666 VCO) |
| 1 | PLL2 / 2 | 1066 MT/s | |
| 2 | PLL1 / 1 | 2400 MT/s, or 2666 MT/s with the 2666 VCO | Final point when 2400/2666 MT/s is requested |
| 3 | PLL2 / 1 | 2133 MT/s | LPDDR4 (non-X) only |
| (sel 4) | external clock | — | Only used by the CA-training path |

2666 MT/s has no slice or MC bank of its own: it is fp2 with PLL1 retuned
([clocks-and-pmu.md](clocks-and-pmu.md)).

### SoC limits

- Memory types: LPDDR4 and LPDDR4X.
- Maximum data rate: **2666 MT/s**. The public K1 datasheet qualifies the
  32-bit LPDDR4/4X interface at 2666 Mbps; there is no PLL setting, MC bank or
  PHY slice for anything faster. The DRAM speed bin does not change this.
- Maximum capacity: **16 GiB** (2 ranks × 8 GiB).
- No bank/rank interleave or address hash is used by this initialisation; CS1 is
  contiguous after CS0 ([address-map.md](address-map.md)).

## Reference board

| Item | Value |
|---|---|
| Board | Banana Pi BPI-F3, 8 GB variant |
| Board EEPROM identity | `k1-x_deb1` (SpacemiT reference design) |
| DRAM package | Samsung **K4UCE3Q4AA-MGCL**, LPDDR4X |
| Ranks | 2 (both in the same package) |
| Die organisation | x8 byte mode, 8 Gb per die (MR8 = `0x52`) |
| Speed bin of the die | 4266 MT/s (the K1 PHY stops at 2666) |
| Total capacity | 8192 MB |
| Boot data rate | 2400 MT/s (SPL default when the EEPROM is not read); final rates 1066, 1200, 1333, 2400 and 2666 MT/s measured (1333 = fp0 with 2666 VCO) |

Mode registers read from all four (CS, CH) pairs of this board (all identical):

| MR | Value | Meaning |
|---|---|---|
| MR5 | `0x01` | Manufacturer ID: Samsung |
| MR6 | `0x08` | Revision |
| MR8 | `0x52` | [7:6] = 1 → x8; [5:2] = 4 → 8 Gb density code |
| MR12 | `0x5d` | Vref(CA), set by the init; training does not change it |
| MR14 | `0x19` | Vref(DQ) after init (25); write training then selects ~`0x1d`–`0x1f` at 2400 |

### Capacity arithmetic

With x8 dies two dies form one 16-bit JEDEC channel, so each (CS, CH) pair holds
2 × 8 Gb = 2048 MB. Two channels and two ranks give 2 × 2 × 2048 MB = 8192 MB.
The 32-bit bus is made of four x8 dies per rank. The general derivation from
MR8 is in [address-map.md](address-map.md).

Typical other SKUs: 4 GiB boards usually have a single rank of x16 dies (not
available for measurement); 16 GiB is the controller maximum (2 ranks × 8 GiB)
and was measured and trained on a second BPI-F3 populated with a **Micron
`MT53E4G32D8`** package (8 × 16 Gb x8 dies, MR5 = `0xFF`, MR8 = `0x58`,
8192 MB per rank, size code `0x11`, ROW_18; see
[configurations.md](configurations.md)).

## Who does what at boot

| Stage | Responsibility |
|---|---|
| BootROM | Loads and verifies the first stage (from eMMC/SD/SPI NOR, or over USB in FDL mode). Leaves PLL1 locked at its own setting. |
| SPL (mainline U-Boot) | Reads the board EEPROM (rank count, data rate), copies `ddr_fw.bin` to SRAM at `0xc082d000`, calls `ddr_init`, writes `0xaa55aa55` to address 0 and reads it back. |
| DDR image (subject of this specification) | Everything else: PLLs, clock mux, MC and PHY programming, JEDEC power-up, mode registers, address map, DFC table, every training pass, the final frequency change. |
| SPL afterwards | Loads OpenSBI + U-Boot proper into DRAM. |

The DDR image runs entirely from SRAM: DRAM is not usable for code, stack or
buffers until it returns. The boot sequence is detailed in
[boot-sequence.md](boot-sequence.md).

## Gaps

- The internal structure of the PHY (number of delay-line stages, tap size in
  picoseconds) is not documented; the tap-to-UI scale in
  [phy.md](phy.md) is inferred.
- How the SoC interconnect maps DRAM above 2 GiB to the CPU (fixed decode vs.
  programmable) is only observed, not documented
  ([address-map.md](address-map.md)).
- Two 2-rank x8 LPDDR4X boards (8 GiB Samsung `K4UCE3Q4AA-MGCL` and 16 GiB
  Micron `MT53E4G32D8`) were measured; 1-rank, x16 and LPDDR4 (non-X) boards
  remain untested on hardware.

---

[← Previous: Glossary](glossary.md) | [Index](README.md#index) | [Next: Boot Interface →](boot-interface.md)
