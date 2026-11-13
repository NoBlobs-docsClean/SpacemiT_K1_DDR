<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: The Complete `ddr_init` Sequence](boot-sequence.md) | [Index](README.md#index) | [Next: Acceptance Criteria and Reference Measurements →](reference-measurements.md)

# Configurations: Byte Mode, Rank Count, Capacity, Rates, LPDDR4

The measured hardware configurations are **LPDDR4X, 2 ranks, x8 byte-mode dies,
8 GiB** (BPI-F3, Samsung `K4UCE3Q4AA-MGCL`) and **16 GiB** (BPI-F3, Micron
`MT53E4G32D8`). Final rates measured on both boards: **1066, 1200, 1333, 2400
and 2666 MT/s**; 1333 is fp0 with the 2666 VCO. Everything else in this file is
derived from public BSP tables, command formats, encodings and JEDEC, and is
marked as such.

## x8 byte mode (measured)

`MR8[7:6] != 0` ⇒ x8 dies. After the first fp0 pass, out of SR:

1. **MC reset**: `pmu+0xb0` bit 21 low ≈100 µs, high ≈100 µs. The PLLs stay
   locked; do not repeat the PLL sequence. The PHY returns to its post-reset
   state (`PHY_INIT` reads `0x100`, taps `0xf`).
2. **Device_Init again**, with the x8 overlays written in every bank **after**
   its timings and **before** fp0 is selected in `CTL_FREQ`.

Overlays (select the bank with `CTL_FREQ`, then write both words):

| fp | `CTL_FREQ` | `ddr+0x13e4` | `ddr+0x3b4` |
|---|---|---|---|
| 3 | `0xF0800400` | `0x19000A02` | `0x08001400` |
| 2 | `0xA0800400` | `0x15000802` | `0x08001000` |
| 1 | `0x50800400` | `0x0c000402` | `0x08000A00` |
| 0 | `0x00800400` | `0x0c000402` | `0x08000A00` |

The x16 values these replace are in the timing table of
[device-init.md](device-init.md) (fp2: `0x13000802` / `0x08000e00`, …).

Effect (measured): the second fp0 pass gate drops from ≈105–118 to ≈32–40 taps.

## x16 dies

`MR8[7:6] == 0`: **no** MC reset, **no** overlays, **no** second Device_Init.
The second fp0 pass still exists (after the map adjustment and the DFC table).
*Not measured on hardware.*

## One rank (typical 4 GiB SKUs) — not measured

- Self-refresh: `mode = 1`, mask `0x04` (bit 2 only); commands
  `0x11000040` / `0x11000080`.
- `cs_sel = 1`: USER_0 init `0x11000001`, MRW `0x11020000 | mr`, ZQCAL only for
  CS0, MR14 write `0x1102000e`.
- PI commands for `cs = 1` only.
- DFC flows: the `0x13` rank byte of USER_0 / CA_CMD / PI entries becomes
  `0x11` (*inferred*).
- Map: a 4 GiB single rank is typically x16 16 Gb dies × 2 channels →
  `area_mb = 4096`, code `0x10`, ROW_17. MR8 expected `[7:6] = 0`,
  `[5:2] = 6`.
- CS1 is still written (start = `area_mb`, same code) or clears its
  present bit ([address-map.md](address-map.md)).
- No x8 reset, no fp3.

## 16 GiB (2 ranks × 8 GiB, x8 16 Gb dies) — measured

Confirmed and characterised end to end on a 16 GiB BPI-F3 booting from the SD
card:

- Size code `0x11` (8 GiB per rank); geometry ROW_18, COL_10, BANK_8:
  `ddr+0x200` / `+0x208 = 0x00110001`, `ddr+0x20c = 2`,
  `ddr+0x220` / `+0x224 = 0x05030832`.
- DRAM: **Micron `MT53E4G32D8`** (128 Gb package: 8 × 16 Gb x8 dies, 2 channels ×
  2 ranks, 200-ball TFBGA; ordering P/N `MT53E4G32D8CY-046 WT:C`).
  MR5 = `0xFF` (Micron), MR6 = `0x07`, MR7 = `0x00`, MR8 = `0x58`
  (`[7:6] = 1` x8, `[5:2] = 6` 16 Gb).
- After all training passes: `CA_CMD = 0x13020095` (MR21).
- MR21 is also inserted in both DFC flows (entry flow after index `0x18`,
  shifting the rest; exit flow after `0x18b`) ([dfc-engine.md](dfc-engine.md)).
- The standard LPDDR4X IO configuration (80 Ω DRAM DQ ODT, 60 Ω DRAM output
  drive, 60 Ω SoC ODT, 40/40 Ω SoC DQ drive, CA Vref `0x9D`, Tx Vref seed
  `0x19`, Rx Vref `0x55`) is selected by device type (LPDDR4X vs LPDDR4) rather
  than vendor ID (`MR5`), so both the Micron 16 GiB board and the Samsung 8 GiB
  board use the same baseline IO settings; the Hynix x8 CA ODT special case does
  not apply (`MR5 != 0x06`).
- All four training passes complete with `PHY_STATUS = 0`: fp0 (1200 MT/s)
  twice, fp1 (1066 MT/s), fp2 (2400 or 2666 MT/s). With a 1066 MT/s final rate
  (2400 VCO), RX minimum margins are 25/25/27/11 of 32 and TX minimum margins
  are 23/23/27/10 of 48; WL taps are 19–24, RX DQ0 16–19, TX DQ0 33–37, and
  DQS gate 29–34 after the x8 PHY re-init. MR14 after fp2 is `0x1b`–`0x1d`,
  always equal to `CTL_TXVREF[21:16]`. Full tables and per-Vref margin curves
  are in
  [reference-measurements.md](reference-measurements.md#16-gib-micron-board-measured-1066-mts-final).
- Independent software RX/TX sweeps land within 0–1 code of the PI on this
  board (RX centres 17/17/17, TX 35/35/36). The TX Vref curve is a flat plateau
  at 1200/1066 MT/s, so the chosen MR14 varies across boots due to plateau
  tie-breaking. At fp2 a single byte-wide RX centre is too optimistic and requires
  **per-lane centring** to pass multi-pattern confirmation with 0 errors — see
  [eye-training.md](eye-training.md#on-the-16-gib-micron-board-measured-1066-final).
- All five final rates (1066, 1200, 1333, 2400, 2666 MT/s) boot cleanly and were
  benchmarked from Linux: 4-process aggregate 3.45/4.04/4.28/7.66/7.87 GB/s
  write, 3.23/3.86/4.64/6.95/7.26 GB/s read, and
  389.7/358.4/335.8/266.3/251.8 ns single-core random latency. A 15-minute
  4-process memory soak at 2666 MT/s (≈6.4 TB of traffic) completed with 0
  errors. See
  [reference-measurements.md](reference-measurements.md#16-gib-micron-board-measured).
- **DRAM DQ ODT ("Tx ODT", `ddr+0x310/314[22:20]`) sweep across all five final
  rates:** on the 16 GiB Micron board, 48, 60, 80 and 120 Ω all complete with
  `PHY_STATUS = 0` and identical minimum margins at 1066, 1200, 1333, 2400 and
  2666 MT/s (25/23, 27/27, 11/10 on the 2400-VCO family; 22/21, 27/27, 9/8 on
  the 2666-VCO family); only the Tx Vref chosen inside the plateau shifts with
  ODT. **40 Ω** sets bit 3 of `PHY_STATUS` (Write DQ Training error) at fp2 at
  every final rate and lowers the fp2 Tx margin to 9 on the 2400-VCO family.
  **240 Ω** degrades Tx minimum margins across all passes at 2666 MT/s
  (19/26/7 instead of 21/27/8) and is not recommended.
- **8 GiB vs 16 GiB Tx ODT at 2666 MT/s:** on the 8 GiB Samsung board, **48 Ω
  also fails** the fp2 write training at 2666 MT/s (`PHY_STATUS = 0x8`,
  reproduced twice), just like 40 Ω, whereas 60, 80 and 120 Ω pass (`240 Ω`
  passes with degraded Tx margin 7). At 1066 MT/s both boards accept 48–120 Ω
  and reject 40 Ω. Thus the safe Tx ODT band at 2666 MT/s is **60–120 Ω** on
  the 8 GiB Samsung board and **48–120 Ω** on the 16 GiB Micron board; the 80 Ω
  default is safe on both.
- **Other IO fields at 2666 MT/s (swept one at a time on the 16 GiB board):**
  varying SoC DQ drive (`COMMON+0x4[7:0]` at 48/48 Ω or 60/60 Ω), DRAM output
  drive (`ddr+0x310/314[19:16]` at 40 or 80 Ω), PHY DQ ODT (`COMMON+0x4[15:8]`
  at 40 or 80 Ω), SoC ODT (`ddr+0x30c[15:12]` at 40 or 80 Ω), and PHY Rx Vref
  (`VREF0/1[7:0]` at `0x44` or `0x66`) keeps `PHY_STATUS = 0` and the same
  minimum margins (22/21, 27/27, 9/8); only 40 Ω PHY DQ ODT adds +1 RX code at
  fp2 (10 vs 9) while TX stays at 8.
- **Static Vref(CA) (`ddr+0x30c[31:24]`, base `0x9D`) at 2666 MT/s:** `0x8D` and
  `0xAD` boot and pass a 120 s memory soak with 0 errors on both boards.
  **`0xBD` passes on the 16 GiB Micron board but fails on the 8 GiB Samsung
  board** at fp1/1066 (`PHY_STATUS = 0x8`, all Tx margins 0) and hangs at fp2
  ([ca-training.md](ca-training.md)).
- **Rank boundary in CPU address space:** ranks are contiguous, not
  fine-interleaved. On the final adjusted map, CS0 occupies `0–9.75 GiB` and CS1
  starts at **CPU address 10 GiB** (`0x2_8000_0000`, DRAM offset 8 GiB) on the
  16 GiB board, whereas CS1 starts at **CPU address 6 GiB** (`0x1_8000_0000`,
  DRAM offset 4 GiB) on the 8 GiB board (as well as on the unadjusted 8 GiB
  default map during the first fp0 pass)
  ([address-map.md](address-map.md#rank-boundary-in-cpu-address-space-measured)).

## Final rates (measured)

| Final rate | fp | Clock | Notes |
|---|---|---|---|
| 1066 | 1 | PLL2 / 2 | Highest eye margin |
| 1200 | 0 | PLL1 / 2, 2400 VCO | |
| 1333 | 0 | PLL1 / 2, 2666 VCO | Same `fp0` bank and DFC entry as 1200; not a `data_rate` table entry |
| 2400 | 2 | PLL1 / 1, 2400 VCO | |
| 2666 | 2 | PLL1 / 1, 2666 VCO | SoC maximum |
| 2133 | 3 on LPDDR4; fp0 on LPDDR4X (fallback) | PLL2 / 1 (fp3) | LPDDR4X has no fp3; the final DFC returns to fp0 |

## 2666 MT/s (measured)

Only the PLL1 VCO changes (`0x203c5555` / `0x50dd6538`,
[clocks-and-pmu.md](clocks-and-pmu.md)). **Nothing** changes in the MC
banks, IO table, RL/WL or DFC table; fp2 is the same slice and bank. fp0 then
runs at ≈1333 MT/s — the same VCO + fp0 is the 1333 final-rate configuration
below. Training margins at fp2 are 8–9 instead of 10 (11 % shorter
UI). 2666 is the SoC ceiling.

## 1333 MT/s (measured)

There is **no 1333 entry** in the `data_rate` table; a plain request for 1333 is
treated as 1200 (the ABI has no way to distinguish it). The hardware reaches
≈1333 by programming PLL1 with the **2666 VCO** and finishing at **fp0**
(clksel `0x3b50`, `CTL_FREQ 0x00800400`). No MC bank, IO table, RL/WL or
DFC-table change; only the PLL1 VCO differs from the 1200 configuration.
Measured: RX window 20/32, TX 17/48; Linux 4.20 GB/s write / 3.23 GB/s read
(1 process), 337 ns; 4 processes 4.30/4.55 GB/s. An image that wants 1333 must
special-case it.

## Rates above 2666 MT/s — not supported

No PLL1 setting, no MC bank, no RL/WL or MR2 value and no DFC flow exists for
2700–3200 MT/s. The DRAM's speed bin (4266 MT/s for the reference part) does not
lift the K1 PHY limit. Any other `data_rate` is treated as 1200; the only
exception is a 1333 request handled by an image that selects the 2666 VCO + fp0
(above).

## 2133 MT/s

LPDDR4 only (fp3 = PLL2/1, clksel `0x3b00`). On LPDDR4X there is no fp3: the
request falls back to 1200 (measured: the final DFC from fp2 back to fp0
completes).

## LPDDR4 (non-X) — derived, not measured

Needs a **separate image**: the ABI does not pass the memory type
and nothing in the silicon reveals it ([boot-interface.md](boot-interface.md)).

Differences in Device_Init:

| Item | LPDDR4X | LPDDR4 |
|---|---|---|
| CA Vref (`ddr+0x30c[31:24]`) | `0x9D` | `0xB2` |
| Tx Vref seed (`ddr+0x30c[23:16]`) | `0x19` | `0xA7` |
| SoC ODT (`ddr+0x30c[15:12]`) | 60 Ω (4) | 80 Ω (3) |
| DRAM Tx ODT (`+0x310/314[22:20]`) | 80 Ω (3) | 120 Ω (2) |
| DRAM Rx drive (`+0x310/314[19:16]`) | 60 Ω (4) | 60 Ω (4) |
| PHY Tx drive p/n (`COMMON+0xc[15:8]`) | 40/40 Ω (`0xD8`) | 48/60 Ω (`0xB0`) |
| PHY Rx ODT | 60 Ω | 120 Ω |
| PHY Rx Vref (`COMMON+0x4`) | `0x55` | `0x33` |
| VOH (`ddr+0x2c8[17]`) | 0.6 V | 0.6 V |
| PHY supply (`pmu+0x398`) | `[11:10] = 3` | `[11:10] = 2`, `[13:12] = 3` (LDO ≈0.88 V) |
| WDQS (`ddr+0x400`, per bank) | not written | fp3 `0x80001e06`, fp2 `0x80001b04`, fp1 `0x80001500`, fp0 `0x80001500` |
| Trained points | fp0, fp1, fp2 | fp0, fp1, **fp3**, fp2 |

WDQS encoding (*inferred*): `bit 31 | (n << 8) | m`, e.g. fp3
`0x80000000 | (0x1E << 8) | 6`.

Orchestration:

```
PLL, SW-FC fp0
Device_Init(fp0)
train(pp0)                 cold, read MR8
[MC reset + Device_Init if x8]
adjust map
fill DFC table
train(pp0)
DFC(1); train(pp1)         1066
DFC(3); train(pp3)         2133 — after pp1, before pp2
DFC(2); train(pp2)
DFC(final)
```

Training differences:

- MR14 on LPDDR4 uses the JEDEC range bit (bit 6) and a different absolute
  scale; the 4X sweep 21..36 must **not** be copied. The correct LPDDR4 sweep is
  unknown.
- CA training may be needed ([ca-training.md](ca-training.md)).
- Acceptance criteria are not established (no board). `PHY_STATUS = 0`, gate
  PASS and a usable DRAM still apply.

## Tx ODT on other LPDDR4X SKUs

Default 80 Ω for LPDDR4X, 120 Ω for LPDDR4. An EEPROM may specify another value
for another 4X SKU, but the 4-argument ABI does not pass it: an image for the
mainline slot can only tie the Tx ODT 1:1 to the type it is built for.

## Hynix x8, 2-rank LPDDR4X SKU (`H9HCNNNFAMMLXR-NEE`, 8 GiB) — derived from BSP, not measured

In addition to the x8 overlays, the public BSP defines a CA drive / CA ODT
configuration table (combinations `0..6`) and selects combination `2` for the
8 GiB Hynix `H9HCNNNFAMMLXR-NEE` (x8, 2-CS) LPDDR4X part:

- PHY `COMMON+0x8` (`C0+0x8` and `C1+0x8` in all four slices): bits `[15:8]` =
  `(ca_pdrv << 5) | (ca_ndrv << 2) = (6 << 5) | (6 << 2) = 0xD8` (40 Ω / 40 Ω
  SoC CA pull-up/pull-down drive).
- `ddr+0x310` and `ddr+0x314` in all four banks: set bits `[30:28] = 2` (120 Ω
  DRAM CA ODT) and bit `23 = 1` (CA ODT enable), giving `0x20b40000` (with
  80 Ω DQ ODT and 60 Ω drive).
- `ddr+0x30c` in all four banks: set bits `[31:24] = 0x19` (CA Vref = `0x19`
  instead of `0x9D`), giving `0x19194314`.

For completeness, the 7 combinations defined in the BSP for `(ca_pdrv, ca_ndrv,
ca_odt, ca_vref)` are:

| Combination | `ca_pdrv` / `ca_ndrv` (`COMMON+0x8[15:8]`) | `ca_odt` (`+0x310/314[30:28]`, bit 23 = 1) | `ca_vref` (`+0x30c[31:24]`) |
|---|---|---|---|
| 0 | 6 / 6 (`0xD8`, 40/40 Ω) | 4 (60 Ω) | `0x52` |
| 1 | 6 / 6 (`0xD8`, 40/40 Ω) | 3 (80 Ω) | `0x2d` |
| 2 (Hynix x8 2-CS) | 6 / 6 (`0xD8`, 40/40 Ω) | 2 (120 Ω) | `0x19` |
| 3 | 6 / 6 (`0xD8`, 40/40 Ω) | 1 (240 Ω) | `0x15` |
| 4 | 5 / 5 (`0xB4`, 48/48 Ω) | 3 (80 Ω) | `0x20` |
| 5 | 5 / 5 (`0xB4`, 48/48 Ω) | 2 (120 Ω) | `0x13` |
| 6 | 5 / 5 (`0xB4`, 48/48 Ω) | 1 (240 Ω) | `0x10` |

The reference board is Samsung (`K4UCE3Q4AA-MGCL`): **do not** apply this there.

## Gaps

These are the gaps with the highest practical impact: each one blocks a class
of boards.

| Gap | Blocks | How it could be closed |
|---|---|---|
| LPDDR4 (non-X): correct MR14 sweep (range bit 6, absolute values), acceptance ranges, whether CA training is needed | Any LPDDR4 board | Bring-up on an LPDDR4 K1 board through FDL, using the IO table and the fp3 ordering above |
| One rank / x16 dies (4 GiB): SR mask `0x04`, map, DFC flows with rank byte `0x11` | 4 GiB SKUs | Repeat the measurements of [reference-measurements.md](reference-measurements.md) on a one-rank board |
| 16 GiB (x8 16 Gb dies): meaning of bit 7 in the MR21 command (`0x13020095`) | Meaning of MR21 bit 7 | Size code `0x11`, ROW_18, MR5/MR8, MR21 insertion in both DFC flows, and all four training passes are confirmed on the 16 GiB Micron board; only the semantics of bit 7 in the MR21 command (`0x80 \| 21`) remain open |
| DRAM vendors other than Samsung and Micron, including the Hynix x8 CA-ODT variant | Vendor-specific IO tuning | Both the 8 GiB Samsung (`K4UCE3Q4AA-MGCL`) and 16 GiB Micron (`MT53E4G32D8`) boards are characterised end to end and share the standard LPDDR4X IO settings (with 60–120 Ω Tx ODT safe on both at 2666 MT/s). The Hynix CA-ODT SKU and other vendors remain untested on hardware |
| A SKU that needs a different Tx ODT or the Hynix CA ODT cannot be identified through the current ABI | One binary for all boards | Needs an SPL change or a dedicated binary per SKU ([boot-interface.md](boot-interface.md)) |

---

[← Previous: The Complete `ddr_init` Sequence](boot-sequence.md) | [Index](README.md#index) | [Next: Acceptance Criteria and Reference Measurements →](reference-measurements.md)
