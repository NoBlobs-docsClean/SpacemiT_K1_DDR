<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Memory Controller](memory-controller.md) | [Index](README.md#index) | [Next: PHY →](phy.md)

# Device_Init: MC Programming, JEDEC Power-Up, ZQ and Mode Registers

Device_Init brings the controller, the PHY analog and the DRAM dies up **at
frequency point 0** (≈1200 MT/s). It runs once before the first training pass,
and once more after the MC reset required by x8 byte-mode dies
([configurations.md](configurations.md)).

Device_Init is not training: it does not trigger the PI, does not enter the
training self-refresh and does not write taps.

Precondition: PLLs locked, SW-FC to fp0 done, MC clock enabled
([clocks-and-pmu.md](clocks-and-pmu.md)).

## Order

1. MC globals.
2. The four timing banks (fp3, fp2, fp1, fp0), each preceded by its
   `CTL_FREQ` bank select. On the second Device_Init for x8 dies, the x8
   overlays go here, after the timings of each bank.
3. `ddr+0x308` read-modify-write; select fp0 in `CTL_FREQ` and `PI_CFG`.
4. Refresh registers.
5. PHY analog of all four slices ([phy.md](phy.md)).
6. DFI `init_start` and wait for the PHY.
7. JEDEC power-up of the DRAM, sequenced by the MC.
8. First batch of mode-register writes.
9. ZQ calibration, per rank.
10. Second batch of mode-register writes.

`cs_sel = 1` if `cs_num == 1`, otherwise `3`.

## 1. MC globals (frequency-independent)

Reset values are not established; write complete words.

| Offset | Value | Notes |
|---|---|---|
| `+0x44` | `0x00040300` | CTL0 |
| `+0x48` | `0x00000001` | *opaque* |
| `+0x50` | `0x000000ff` | *opaque* |
| `+0x58` | `0x3fd53fd5` | *opaque* |
| `+0x64` | `0x100d0803` | *opaque* |
| `+0x80` | `0x00000000` | |
| `+0x180` | `0x00010200` | *opaque* |
| `+0xa00` | `0x00000000` | |
| `+0xac0` | `0x00000000` | |
| `+0xacc` | `0xffffffff` | |
| `+0x200` | `0x00100001` | CS0 present, default size 4 GiB |
| `+0x204` | `0x00000000` | CS0 start high |
| `+0x208` | `0x00100001` | CS1 default, written even with one rank (fixed later by the map adjustment) |
| `+0x20c` | `0x00000001` | CS1 start = 4 GiB |
| `+0x220` | `0x05030732` | CS0 geometry: 8 banks, COL_10, ROW_17 |
| `+0x224` | `0x05030732` | CS1 geometry |
| `+0x2c0` | `0x14008000` | *opaque* |
| `+0x2c4` | `0x000000b8` | *opaque* |
| `+0x2c8` | `0x0000ffff` | bit 17 = VOH / PU-cal select, see below |
| `+0x2cc` | `0x00000000` | Auto power-save off |

`+0x2c8`: read-modify-write, clear bit 17 and write `rx_pu_cal << 17`. Both
LPDDR4 and LPDDR4X use VOH = 0.6·VDDQ → bit 17 = 0, the word stays
`0x0000ffff`.

The address-map defaults are what the first training pass sees; the real
capacity is programmed later ([address-map.md](address-map.md)).

## 2. The four timing banks

Select each bank with `CTL_FREQ` (`ddr+0x304`) **before** writing its words.
Both nibbles of `[31:28]` are set to the fp:

| fp | `CTL_FREQ` while filling |
|---|---|
| 3 | `0xF0800400` |
| 2 | `0xA0800400` |
| 1 | `0x50800400` |
| 0 | `0x00800400` |

All four banks are filled, fp3 included, also on LPDDR4X.

### IO / termination words (same in every bank)

Resistance codes used by the 3-bit fields: 1 = 240 Ω, 2 = 120 Ω, 3 = 80 Ω,
4 = 60 Ω, 5 = 48 Ω, 6 = 40 Ω.

**`ddr+0x30c` (`DRAM_CONFIG_4` / TXVREF):**

| Bits | Field | LPDDR4X | LPDDR4 |
|---|---|---|---|
| `[31:24]` | Vref(CA), sent as MR12 | `0x9D` | `0xB2` |
| `[23:16]` | Vref(DQ), sent as MR14 (6 useful bits `[21:16]`; bit 22 = MR14 range bit on LPDDR4) | `0x19` | `0xA7` |
| `[15:12]` | SoC-side ODT (resistance code) | 4 (60 Ω) | 3 (80 Ω) |
| `[11:0]` | Fixed control field (*opaque*, must be `0x314`) | `0x314` | `0x314` |

Composed 32-bit value: `(ca_vref << 24) | (tx_vref << 16) | (soc_odt << 12) | 0x314`.
- LPDDR4X → `0x9d194314` (or `0x19194314` on the Hynix x8 2-CS SKU where CA Vref = `0x19`).
- LPDDR4 → `0xb2a73314`.

**`ddr+0x310` (CS0) and `ddr+0x314` (CS1) (`DRAM_CONFIG_5_CS0/CS1`), identical:**

| Bits | Field | LPDDR4X | LPDDR4 |
|---|---|---|---|
| `[30:28]` | Device CA ODT resistance code | 0 (or 2 = 120 Ω on Hynix x8 2-CS SKU, [configurations.md](configurations.md)) | 0 |
| `[23]` | Device CA ODT enable | 0 (or 1 on Hynix x8 2-CS SKU) | 0 |
| `[22:20]` | Device DQ ODT for writes ("Tx ODT") | 3 (80 Ω) | 2 (120 Ω) |
| `[19:16]` | Device pull-down output drive for reads ("Rx drive") | 4 (60 Ω) | 4 (60 Ω) |
| `[15:0]` | Reserved / zero | `0x0000` | `0x0000` |

Composed 32-bit value: `(ca_odt << 28) | (ca_odt_en << 23) | (tx_odt << 20) | (rx_drv << 16)`.
- LPDDR4X (80 Ω DQ ODT, 60 Ω drive, CA ODT disabled) → `0x00340000`.
- LPDDR4X Hynix `H9HCNNNFAMMLXR-NEE` (CA ODT code 2 enabled) → `0x20b40000`.
- LPDDR4 (120 Ω DQ ODT, 60 Ω drive) → `0x00240000`.

*Inferred:* the MC derives MR11 (CA ODT and DQ ODT) and MR3 (pull-down drive)
from these fields when it serialises those mode registers.

**Measured sensitivity of these IO fields at 2666 MT/s (8 GiB Samsung and
16 GiB Micron boards):** both boards use the standard LPDDR4X setting above
(`0x9d194314` / `0x00340000`, 80 Ω DRAM DQ ODT). Sweeping the fields at
2666 MT/s shows:

- **Device DQ ODT ("Tx ODT", `+0x310/314[22:20]`):** on the 16 GiB Micron board,
  48, 60, 80 and 120 Ω all train with `PHY_STATUS = 0` and identical minimum
  margins; **40 Ω** sets bit 3 of `PHY_STATUS` (write DQ training error) at fp2,
  and **240 Ω** lowers the Tx minimum margins in every pass. On the 8 GiB
  Samsung board, **48 Ω also fails** the fp2 write training at 2666 MT/s
  (`PHY_STATUS = 0x8`), so the safe band across both boards is **60–120 Ω** (48–120 Ω
  on the Micron board).
- **Other IO parameters:** varying SoC DQ drive (`COMMON+0x4[7:0]`), DRAM output
  drive (`+0x310/314[19:16]`), PHY DQ ODT (`COMMON+0x4[15:8]`), SoC ODT
  (`+0x30c[15:12]`) and PHY Rx Vref (`VREF0/1[7:0]`) produces no improvement in
  the minimum margins at 2666 MT/s. The standard LPDDR4X IO configuration is
  therefore adequate for both Samsung and Micron SKUs without vendor-specific
  tables.

### Timing words per bank (LPDDR4X)

JEDEC names are *inferred* from the role of each word (counts in clock cycles of
the fp); the boot needs the **values**, not the names. Offsets are from `ddr`
(do not confuse `ddr+0x3b4` with `pmu+0x3b4`).

| Offset | fp3 (2133) | fp2 (2400) | fp1 (1066) | fp0 (1200) | Role (*inferred*) |
|---|---|---|---|---|---|
| `+0x300` | `0x00000c18` | `0x00000c18` | `0x0000080e` | `0x0000080e` | RL/WL (MR2 source) |
| `+0x30c` | IO | IO | IO | IO | Vref / SoC ODT (above) |
| `+0x310` | IO | IO | IO | IO | DRAM ODT / drive, CS0 |
| `+0x314` | IO | IO | IO | IO | DRAM ODT / drive, CS1 |
| `+0x34c` | `0x0007204a` | `0x0007204a` | `0x00030848` | `0x00030848` | |
| `+0x38c` | `0x00360020` | `0x00430000` | `0x00280018` | `0x00280018` | tRAS/tRP… |
| `+0x390` | `0x042a0020` | `0x05350028` | `0x03200018` | `0x03200018` | tRCD/tRC |
| `+0x394` | `0x8096012b` | `0x80a80151` | `0x807000e0` | `0x805400a8` | tRFC/tREFI |
| `+0x398` | `0x01330133` | `0x017f017f` | `0x00e600e6` | `0x00e600e6` | |
| `+0x39c` | `0x00101010` | `0x00141414` | `0x000c0c0c` | `0x000c0c0c` | tRRD/tCCD |
| `+0x3a0` | `0x06100808` | `0x07140a0a` | `0x050c0606` | `0x050c0606` | tWTR/tRTP |
| `+0x3a4` | `0x00000003` | `0x00000003` | `0x00000003` | `0x00000003` | |
| `+0x3a8` | `0x0000020f` | `0x00000213` | `0x0000020c` | `0x0000020c` | |
| `+0x3ac` | `0x2044142d` | `0x36541838` | `0x18330f22` | `0x18330f22` | tFAW/tRAS |
| `+0x3b0` | `0x17140814` | `0x1c180a18` | `0x110f080f` | `0x110f080f` | |
| `+0x3b4` | `0x08000b00` | `0x08000e00` | `0x08000800` | `0x08000800` | changes in x8 byte mode |
| `+0x3b8` | `0x00000800` | `0x00000e00` | `0x00000600` | `0x00000600` | |
| `+0x3bc` | `0x02020404` | `0x02020404` | `0x02020404` | `0x02020404` | |
| `+0x3c0` | `0x00000004` | `0x10000004` | `0x00000003` | `0x00000002` | |
| `+0x3c4` | `0x00000004` | `0x00000004` | `0x00000003` | `0x00000003` | |
| `+0x3d8` | `0x00008190` | `0x0000d94e` | `0x00008190` | `0x00008190` | |
| `+0x3fc` | `0x000d0065` | `0x000c005e` | `0x000c005e` | `0x000c005e` | |
| `+0x13e4` | `0x13000802` | `0x13000802` | `0x0a000402` | `0x0a000402` | DFI/PHY timing; changes in x8 byte mode |
| `+0x13ec` | `0x00000480` | `0x00000450` | `0x00000480` | `0x00000480` | PI_RD_CFG default; training writes `0x5070` |

These are x16 values. For x8 dies, `+0x13e4` and `+0x3b4` are overridden in
each bank during the second Device_Init ([configurations.md](configurations.md)).
Programming the x16 values on x8 dies is enough for the first training pass to
complete with `PHY_STATUS = 0` (*measured*).

LPDDR4 (non-X) additionally writes `+0x400` (WDQS) in every bank
([configurations.md](configurations.md)).

2666 MT/s uses **the same** fp2 bank as 2400; only the PLL1 VCO changes. There
is no bank, RL/WL or MR2 setting for anything above 2666.

## 3. Close the bank programming

```
ddr+0x308  = (ddr+0x308 & 0xF00FFFFF) | (0x10 << 20)    /* opaque field [27:20] */
CTL_FREQ   = 0x00800400                                  /* select fp0 */
PI_CFG    |= (0 << 2)                                    /* fp0 */
```

## 4. Refresh (global, identical for every fp)

After selecting fp0:

```
ddr+0x380 = 0x0030d400
ddr+0x384 = 0x0004e200
ddr+0x388 = 0x0c800000
```

## 5. PHY analog

All four slices, both sub-PHYs. See [phy.md](phy.md), section
"Analog initialisation".

## 6. DFI `init_start`

```
ddr+0x13d0 = 0x10000001 | (cs_sel << 24)     /* PHY init_start */
wait (ddr+0x13fc & 0x80000000) != 0           /* PI_STATUS bit 31 */
ddr+0x13d0 = 0x10000100 | (cs_sel << 24)     /* clear init_start */
```

This also completes the PHY impedance calibration armed by the analog step
(there is no separate done bit for it). Budget 10 ms.

## 7. JEDEC power-up

The **MC** performs RESET_n, CKE and the tINIT1..tINIT5 timing of JEDEC. The
software only starts it and waits:

```
ddr+0x20 = 0x10000001 | (cs_sel << 24)        /* USER_0: DRAM init */
wait (ddr+0x08 & status_check) == status_check
```

`status_check = 0x01` (1 rank) or `0x11` (2 ranks): bits 0 and 4 of STATUS are
"init done" per rank. Budget 10 ms. There is no separate start bit.

## 8–10. Mode registers and ZQ

Mode-register writes name the MR in the command; the data comes from the bank
of the active fp (fp0 here). Write format and details in
[mode-registers.md](mode-registers.md).

```
MRW MR1, MR2, MR13, MR3, MR22            /* 0x10020000 | (cs_sel << 24) | mr */
ZQCAL per rank                           /* through USER_0, not CA_CMD */
MRW MR12, MR14, MR11, MR23
```

No completion bit exists for mode-register writes: issue them in sequence with
≥1 µs between writes (tMRW).

**ZQCAL** (MPC through USER_0):

```
ddr+0x20 = 0x11002000      /* CS0 START */
wait 2 µs                  /* tZQCAL = 1 µs */
ddr+0x20 = 0x11001000      /* CS0 LATCH */
wait 1 µs                  /* tZQLAT >= max(30 ns, 8 nCK) */
if cs_num == 2:
    ddr+0x20 = 0x12002000  /* CS1 START */
    wait 2 µs
    ddr+0x20 = 0x12001000  /* CS1 LATCH */
    wait 1 µs
```

The silicon exposes no ZQCAL-done bit in STATUS. The waits are the JEDEC
minimums with margin. The DFC command flows contain no ZQCAL command, so ZQ
calibration only happens here.

MR21 for 16 GiB configurations is written **after all training passes**, not
here ([configurations.md](configurations.md)).

## Registers Device_Init does not touch

- `ddr+0x148`: written only by the HW-DFC preparation (`0x80ac0000`).
- `ddr+0x28` (MISC): not written; reads 0 afterwards.
- `phy+0x10028` (`PHY_INIT`): not written; reads `0x100` afterwards.

## How to verify Device_Init

- `ddr+0x08 & 0x11 == 0x11` (2 ranks): dies ready, not in training SR. The
  typical value is `0x33` (bits 0, 1, 4, 5).
- `phy+0x10028` reads `0x100`.
- `CTL_FREQ == 0x00800400`, `PI_CFG[3:2] == 0`.
- `PI_STATUS == 0x80008000` (idle).
- PHY `COMMON+0xc` reads `0x36e4d800` (LPDDR4X, reference board).
- Out of self-refresh, MR5 / MR8 / MR12 / MR14 are readable; on the reference
  board `0x01 / 0x52 / 0x5d / 0x19`.

## Gaps

- Exact JEDEC meaning of each timing word; the table is a set of values
  measured to work, not a formula. Deriving values for other speed bins or DRAM vendors
  is not possible yet. This blocks DRAM parts with different timings; it could
  be closed by perturbing one word at a time and correlating the effect with
  JEDEC timings (through FDL).
- Meaning of the globals marked *opaque*, of `ddr+0x308[27:20]` and of
  `ddr+0x30c[11:0] = 0x314`.
- How the MC builds MR1/MR2/MR3/MR11/MR22/MR23 from the bank (no register
  exposes "MR1 = value"); mapping of bank fields to MR bits is inferred only
  for MR12, MR14 (direct) and RL/WL (`+0x300`).
- No completion indication exists for ZQCAL or MR writes; the waits are
  JEDEC-based.

---

[← Previous: Memory Controller](memory-controller.md) | [Index](README.md#index) | [Next: PHY →](phy.md)
