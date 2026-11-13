<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: PHY](phy.md) | [Index](README.md#index) | [Next: Self-Refresh and MC STATUS →](self-refresh.md)

# Mode Registers

The MC exposes the DRAM mode registers through the CA_CMD command port
(`ddr+0x24`) and the MR_DATA result register (`ddr+0x370`). Command-word layout
is described in [memory-controller.md](memory-controller.md).

## Reading a mode register (MRR)

Only **outside** self-refresh:

```
CA_CMD = 0x10010000
         | ((cs + 1) << 24)      /* cs 0 or 1 -> rank mask 1 or 2 */
         | (ch << 18)            /* JEDEC channel 0 or 1 */
         | mr                    /* 0..255 */
wait MR_DATA bit 31              /* valid */
value = MR_DATA & 0xff
```

Budget 10 µs. Reads address one (rank, channel) pair at a time: with two ranks
there are four pairs to read (cs0/ch0, cs0/ch1, cs1/ch0, cs1/ch1).

**Runtime access.** With Linux running, the same MMIO writes do not complete:
`CA_CMD` (`ddr+0x24`) is not acted upon and `MR_DATA` (`ddr+0x370`) never sets
bit 31. At the **U-Boot prompt** the sequence works and returns `0x8000ffff` for
MR5 on the Micron board. The valid bit (bit 31) appears only on the first read
of `MR_DATA` after the command; a second read returns the same byte with bit 31
clear. `MR_DATA[15:8]` mirrors `[7:0]`.

## Writing a mode register (MRW)

```
cs_sel   = (cs_num == 1) ? 1 : 3
CA_CMD   = 0x10020000 | (cs_sel << 24) | (mr & 0xff)
         /* = 0x11020000 | mr with 1 rank, 0x13020000 | mr with 2 ranks */
```

- The **number** of the MR is in the command; the **data is not**. The MC
  serialises it from the timing bank of the active frequency point
  (`CTL_FREQ[31:30]`).
  - **MR14** data comes from `ddr+0x30c[21:16]` (mask `0xffc0ffff`).
  - **MR12** data comes from `ddr+0x30c[31:24]`.
  - **MR1, MR2, MR3, MR11, MR13, MR22, MR23** are derived by the MC from the
    rest of the bank (`+0x300`, `+0x310`, `+0x314`, `+0x2c8`, timings).
    There is no register holding "MR1 = value"; fill the bank
    ([device-init.md](device-init.md)) before issuing the command.
- Writes carry no channel field: the MC applies them to both JEDEC channels of
  the package.
- The silicon exposes no "write done" bit. Issue writes in sequence with
  ≥1 µs between them (tMRW). Verify by reading back outside self-refresh.

## Writing Vref(DQ) during training (MR14)

Used at **every** step of the write-training sweep, with the ranks in
self-refresh:

```
CTL_TXVREF = (CTL_TXVREF & 0xffc0ffff) | ((vref_dq & 0x3f) << 16)   /* ddr+0x30c */
CA_CMD     = 0x1302000e                                               /* MRW MR14 */
```

(`0x1102000e` with one rank.) The PI does **not** program MR14 by itself:
changing only `ddr+0x30c` without the command trains against the previous Vref.

After the self-refresh exit at the end of the pass, MR14 read from the dies must
equal `CTL_TXVREF[21:16]`; a mismatch means the last command did not reach the
die.

Read training never touches MR12 / Vref(CA).

## Frequency set points (FSP)

JEDEC LPDDR4/4X dies hold two FSP copies of the frequency-dependent MRs
(FSP-0, FSP-1); the MC has four banks. On this SoC:

| fp | FSP |
|---|---|
| 0, 1 | FSP-0 |
| 2, 3 | FSP-1 |

- Device_Init (fp0 active) writes MR13 **once**, before MR14.
- The HW-DFC flow writes MR13 itself (before and after the other MRs) and
  issues the FSP-WR / FSP-OP requests through USER_0 (`0x13000008`,
  `0x13000004`) ([dfc-engine.md](dfc-engine.md)).
- Software does not assemble the MR13 byte: `CA_CMD = 0x1302000d` lets the
  MC serialise MR13 from the bank.
- Do not issue USER_0 `0x13000008` after a DFC "to align" FSPs: it flips
  FSP-WR away from the set that reads observe.

## Read quirks (measured)

### MR13 always reads 0

An MRR of MR13 returns `0x00` **always** on this SoC: after Device_Init, after
every DFC, after an explicit `0x1302000d`. MR13 cannot be used to confirm
FSP-WR / FSP-OP; treat it as write-only. JEDEC behaviour still applies to the
write side; this port simply does not show it.

### The first MRR after the first DFC to fp2 reads 0

After the **first** frequency change to fp2 (fp2 still untrained), the
**first** MRR returns `0x00` whatever the MR, rank and channel. Every following
MRR — same MR on another (rank, channel), or the same one again — returns the
correct value. The DFC to fp1 and the final DFC to an already trained fp2 do not
show it.

| After | MR14 (1st round) | MR13 | MR14 (2nd round) |
|---|---|---|---|
| DFC to fp1 | 19 19 19 19 | 00 00 00 00 | 19 19 19 19 |
| first DFC to fp2 | **00** 19 19 19 | 00 00 00 00 | 19 19 19 19 |
| final DFC to fp2 | 19 19 19 19 | 00 00 00 00 | 19 19 19 19 |

(Order cs0/ch0, cs0/ch1, cs1/ch0, cs1/ch1; values in hex. When MR13 is read
first, it is MR13 that absorbs the zero.)

Rule: after **every** HW-DFC, before any MRR whose value matters, issue one
**dummy MRR** (any readable MR, e.g. MR5 or MR14, any rank/channel), wait for
bit 31 and discard the value. No delay in µs replaces this dummy read. Do not
fail the boot on that first value.

Measured on both the 16 GiB Micron board and the 8 GiB Samsung board: the first
MRR immediately after the first DFC to fp2 reads `0x00`, whereas a second MRR
1 µs later already returns the valid byte (`0xFF` on the Micron board, `0x01`
on the Samsung board), as do reads at 10 µs, 100 µs and 1 ms. Thus the invalid
value affects only the first read; the bus is usable within ~1 µs after it. This
is SoC behaviour, independent of the DRAM vendor.

This is **not** the die's FSP-1 holding a default `0x00`: the following read
already returns the bank value. *Inferred:* it is related to the first read
capture at 2400 MT/s with an untrained slice (or to a leftover valid bit in
MR_DATA); the exact mechanism is unknown and the initialisation does not depend on
it.

### No re-seed needed after a DFC

Re-writing MR13 + MR14 outside self-refresh right after the DFC (`0x1302000d`,
then `0x1302000e`) is optional and has no measurable effect on training results
(A/B tested). It is not needed.

## Do not access mode registers in self-refresh

The MR protocol uses CA_CMD and competes with self-refresh. Read MR5/8/12/14
**before** entering SR (diagnostics) or **after** exiting (MR14 check). The
only command allowed in SR is the training's MR14 write above.

## Which MRs are written when

| When | MRs |
|---|---|
| Device_Init, first batch | MR1, MR2, MR13, MR3, MR22 |
| Device_Init, after ZQCAL | MR12, MR14, MR11, MR23 |
| Every HW-DFC (table flow) | MR13, MR1, MR2, MR3, MR11, MR12, MR14, MR22, MR13 again (MR21 too with 16 GiB) |
| Write training, each step | MR14 |
| After all training, 16 GiB only | MR21 (`CA_CMD = 0x13020095`) |

MR2 (RL/WL) changes with the fp because it is derived from the bank's `+0x300`.

## Values on the reference board (LPDDR4X)

Read back; software does not write these bytes directly (the MC derives
them from the bank):

| MR | After init | After pp2 training | Source |
|---|---|---|---|
| MR5 | `0x01` | `0x01` | Manufacturer (read-only): Samsung |
| MR6 | `0x08` | `0x08` | Revision (read-only) |
| MR8 | `0x52` | `0x52` | Width/density (read-only) |
| MR12 | `0x5d` | `0x5d` | CA Vref from `ddr+0x30c[31:24] = 0x9D` |
| MR14 | `0x19` | `0x1d`–`0x1f` at 2400 | Seed 25, then chosen by write training |

`ddr+0x30c[31:24] = 0x9D` produces MR12 = `0x5d`: the 6-bit Vref value
(`0x1d`) is the same, and the bank's bit 7 shows up as MR12 bit 6, the JEDEC
VR-CA range bit (*inferred*: the MC re-encodes the range flag).

### Values on the 16 GiB board (Micron)

Measured on the 16 GiB board across all four (CS, CH) pairs:

| MR | After init | After pp2 training | Meaning / Source |
|---|---|---|---|
| MR5 | `0xFF` | `0xFF` | Manufacturer (read-only): **Micron** |
| MR6 | `0x07` | `0x07` | Revision ID 1 (read-only) |
| MR7 | `0x00` | `0x00` | Revision ID 2 (read-only) |
| MR8 | `0x58` | `0x58` | `[7:6] = 1` → x8 byte mode; `[5:2] = 6` → 16 Gb per die |
| MR12 | `0x5d` | `0x5d` | CA Vref from `ddr+0x30c[31:24] = 0x9D` |
| MR14 | `0x19` | `0x1b`–`0x1d` at 2400 | Seed 25, then chosen by write training |

The package is a Micron `MT53E4G32D8` (8 × 16 Gb x8 dies, 2 channels × 2 ranks,
128 Gb = 16 GiB) ([configurations.md](configurations.md)). As noted above, if
MR5 is read immediately after the first DFC to fp2 without a prior dummy read,
the first (CS0, CH0) read returns `0x00` while the remaining three pairs return
`0xFF`; never classify the DRAM vendor from an unprimed first MRR after a DFC.

### MR8 decoding

| Bits | Field | 8 GiB board (Samsung) | 16 GiB board (Micron) |
|---|---|---|---|
| `[7:6]` | I/O width: 0 = x16, 1 = x8 | 1 → x8 byte mode | 1 → x8 byte mode |
| `[5:2]` | Density code (JEDEC, per channel): 0 = 2 Gb, 1 = 3 Gb, 2 = 4 Gb, 3 = 6 Gb, 4 = 8 Gb, 5 = 12 Gb, 6 = 16 Gb, 12 = 1 Gb | 4 (8 Gb) | 6 (16 Gb) |
| `[1:0]` | Type | 2 (`0x52`) | 0 (`0x58`) |

`MR8[7:6] != 0` (x8) requires the MC reset and the second Device_Init before
the second fp0 pass ([configurations.md](configurations.md)). Capacity and
geometry derivation is in [address-map.md](address-map.md).

## LPDDR4 (non-X)

Same order and the same ports, with the LPDDR4 IO table. MR14 on LPDDR4 uses
the JEDEC range bit (MR14 bit 6) and a different absolute scale: the
LPDDR4X sweep seed 21..36 does **not** fit the same range. No LPDDR4 board was
available to measure a correct seed.

## Gaps

- Mechanism of the zero on the first MRR after the first DFC to fp2: leftover
  MR_DATA valid bit vs. first DQ capture at an untrained rate. A read of
  `ddr+0x370` before the first CA_CMD after that DFC would discriminate. The
  initialisation does not depend on it.
- Why MR13 reads 0 (port limitation vs. die behaviour).
- How the MC derives MR1/2/3/11/22/23 from the bank fields.
- Meaning of bit 7 in the MR21 command low byte (`0x95` = `0x80 | 21`) and of
  MR21 bit 7 on the die (RFU per JEDEC). The 16 GiB Micron board boots and
  trains cleanly with `0x13020095`, but the bit's internal semantics remain
  unknown.
- LPDDR4 MR14 range and seed.

---

[← Previous: PHY](phy.md) | [Index](README.md#index) | [Next: Self-Refresh and MC STATUS →](self-refresh.md)
